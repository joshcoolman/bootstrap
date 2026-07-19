# Image store

Object storage for an app that handles uploads or generated images. **Not part
of the base scaffold** — add it when an app actually needs somewhere to put
files.

Point an agent at this file: *"read `parts/image-store.md` and set up storage."*
It is written to be executed, not just read.

Default implementation is **Railway Buckets**, keeping the app on one vendor.
**Cloudflare R2** is the documented swap. Both are the plain S3 API, so moving
between them is an endpoint and a credential — which is the entire reason the
contract below exists.

## The contract

```ts
// src/features/storage/types.ts
export interface ImageStore {
  put(key: string, bytes: Uint8Array, contentType: string): Promise<{ key: string }>
  displayUrl(key: string): Promise<string>
  delete(keys: string[]): Promise<void>
  list(prefix: string): Promise<{ key: string; size: number }[]>
}
```

Four methods. Two of the choices in that signature are load-bearing and were
learned the expensive way.

### 1. `displayUrl` is async, and its result is always ephemeral

Even when the implementation returns a permanent URL. A presigned URL dies in
an hour; a public one never does. Code written against the permanent case is
*silently broken* on the other — it works in dev, then 403s in production an
hour after the page loads.

Treating every display URL as short-lived costs nothing and makes the two
implementations genuinely interchangeable.

### 2. Everything is keyed by a stable key, never by a URL

The trap presigning sets: a display URL looks like a perfectly good identity
key, so it ends up as a save key, a dedupe key, a foreign reference. All of
those break an hour later when the URL is regenerated.

Persist the **key**. Resolve the URL at render time. If a record needs to
reference an image, it references `key`.

## Key naming — content-addressed

```ts
// src/features/storage/keys.ts
import { createHash } from 'node:crypto'

const EXT: Record<string, string> = {
  'image/jpeg': 'jpg', 'image/png': 'png',
  'image/webp': 'webp', 'image/gif': 'gif', 'image/avif': 'avif',
}

export function contentKey(prefix: string, bytes: Uint8Array, contentType: string): string {
  const hash = createHash('sha256').update(bytes).digest('hex').slice(0, 32)
  return `${prefix}/${hash}.${EXT[contentType] ?? 'bin'}`
}
```

Same bytes always produce the same key, which buys three things for free:

- **Deduplication** — re-uploading an identical file is a no-op
- **Immutability** — a key's contents never change, so responses cache forever
- **No cache-skew class of bug** — with mutable keys, a CDN keeps serving the
  old object after an overwrite, and keeps serving a cached 404 after a
  late upload. Content addressing makes both unrepresentable.

Derive the extension from the *content type*, not a filename. A `.jpg` key
holding a PNG is harmless but confusing, and it happens the moment you trust an
uploaded filename.

## Railway Buckets implementation

```ts
// src/features/storage/store.ts
import 'server-only'
import {
  DeleteObjectsCommand, GetObjectCommand, ListObjectsV2Command,
  PutObjectCommand, S3Client,
} from '@aws-sdk/client-s3'
import { getSignedUrl } from '@aws-sdk/s3-request-presigner'
import type { ImageStore } from './types'

const PRESIGN_TTL_SECONDS = 60 * 60

function env(name: string): string {
  const value = process.env[name]
  if (!value) throw new Error(`${name} is not set`)
  return value
}

let client: S3Client | null = null
function getClient(): S3Client {
  client ??= new S3Client({
    region: env('BUCKET_REGION'),
    endpoint: env('BUCKET_ENDPOINT'),
    credentials: {
      accessKeyId: env('BUCKET_ACCESS_KEY_ID'),
      secretAccessKey: env('BUCKET_SECRET_ACCESS_KEY'),
    },
  })
  return client
}

export const store: ImageStore = {
  async put(key, bytes, contentType) {
    await getClient().send(new PutObjectCommand({
      Bucket: env('BUCKET_NAME'),
      Key: key,
      Body: bytes,
      ContentType: contentType,
      // Content-addressed keys are immutable, so this is safe and makes the
      // /img route's cache header honest.
      CacheControl: 'public, max-age=31536000, immutable',
    }))
    return { key }
  },

  async displayUrl(key) {
    return getSignedUrl(
      getClient(),
      new GetObjectCommand({ Bucket: env('BUCKET_NAME'), Key: key }),
      { expiresIn: PRESIGN_TTL_SECONDS },
    )
  },

  async delete(keys) {
    if (keys.length === 0) return
    await getClient().send(new DeleteObjectsCommand({
      Bucket: env('BUCKET_NAME'),
      Delete: { Objects: keys.map((Key) => ({ Key })) },
    }))
  },

  async list(prefix) {
    const result = await getClient().send(new ListObjectsV2Command({
      Bucket: env('BUCKET_NAME'), Prefix: prefix, MaxKeys: 1000,
    }))
    return (result.Contents ?? []).map((o) => ({ key: o.Key!, size: o.Size ?? 0 }))
  },
}
```

Presigning is local HMAC — no network round trip — so it's cheap to call per
row. `list` does not paginate; at 1000 keys, add a `ContinuationToken` loop
rather than pretending the result is complete.

## The `/img/[key]` route — build this up front

Railway Buckets are **private only**. Presigned URLs alone have two problems
that look fine in development and bite in production:

1. **They expire in place.** Nothing re-signs them. Leave a page open past the
   TTL and any image the browser needs to *re-fetch* — lazy-load on scroll, a
   lightbox opening an off-screen image, a tab waking — 403s.
2. **They defeat caching entirely.** A fresh signature on every render means a
   different URL on every render, so the browser cache never hits and every
   image re-downloads on every page load.

One route fixes both. A stable app URL that redirects to a fresh presign:

```ts
// src/app/img/[...key]/route.ts
import { NextResponse } from 'next/server'
import { store } from '#/features/storage/store'

export async function GET(_req: Request, { params }: { params: Promise<{ key: string[] }> }) {
  const { key } = await params
  const objectKey = key.join('/')

  // Keys are content-addressed and unguessable, but this route sits behind the
  // app's deny-by-default middleware anyway — images inherit the app's auth.
  try {
    return NextResponse.redirect(await store.displayUrl(objectKey), {
      // Cache the redirect for less than the presign TTL so a cached 307 can
      // never outlive the URL it points at.
      headers: { 'Cache-Control': 'private, max-age=1800' },
    })
  } catch {
    return new NextResponse('Not found', { status: 404 })
  }
}
```

Then `<img src={`/img/${key}`}>` — stable, cacheable, never expires, and no
presigning at the call site.

**A security note worth stating plainly:** because this route sits behind the
app's middleware, images are protected by the same login as everything else.
That is usually what you want for a private app. Switching to R2 public URLs
gives up that property — anyone with the URL can fetch the image, forever,
without signing in. Choose deliberately.

## Cloudflare R2 — the swap

Same code. Different endpoint and credentials:

```
BUCKET_ENDPOINT=https://<ACCOUNT_ID>.r2.cloudflarestorage.com
BUCKET_REGION=auto
```

Reach for R2 when you want public URLs, a CDN, hotlinkable images, or free
egress. Storage is $0.015/GB-month, egress is genuinely $0, and the free tier
(10 GB, 1M writes, 10M reads per month) covers a personal app indefinitely.

Two caveats:

- The `r2.dev` public subdomain is **explicitly not for production** and
  rate-limits to 429s. Use a custom domain on Cloudflare DNS.
- Going public means giving up the auth inheritance described above. If the
  images should stay private, keep presigning and keep the `/img` route — R2
  supports both.

**Do not reach for D1.** R2 is the S3 API and portable anywhere; D1 is
Cloudflare's own service built around Workers bindings. One is a commodity, the
other is a differentiator, and only one of them lets you leave.

## Background copies — the one thing the contract can't hide

If the app copies bytes in from somewhere else (a generation provider, a remote
URL), that work usually shouldn't block the response.

**On Railway this just works.** The persistent Node process keeps running after
the response is sent, so an un-awaited async call completes.

**On Vercel it does not.** The lambda freezes at response time and the copy dies
mid-flight; you need `waitUntil`, which has its own timeout.

Either way, write it so a lost attempt is harmless:

```ts
const inFlight = new Set<string>()

export function scheduleCopy(key: string, sourceUrl: string, onCopied: (key: string) => Promise<void>) {
  if (inFlight.has(key)) return
  inFlight.add(key)

  void copyToBucket(key, sourceUrl)
    .then(onCopied)
    .catch((err) => {
      // LOG IT. A silent catch here means a permanently-broken copy path
      // retries forever and looks completely healthy.
      console.error(`[storage] copy failed for ${key}:`, err)
    })
    .finally(() => inFlight.delete(key))
}
```

Three rules that make this safe:

- **Retry on read.** A null storage key means not-yet-copied; every read
  opportunistically re-fires. No queue, no cron, no status enum.
- **Conditional writes.** Persist with `where key is null` so two racing
  repairs can't clobber each other.
- **Log the failure.** The reference app this came from swallows the error
  silently, which is the one genuinely bad decision in an otherwise good
  design.

Note `inFlight` is per-process, so it de-dupes only within one instance. That's
fine at one replica and quietly stops working at two.

## Provisioning (Railway)

1. In the Railway project → **Create** → **Bucket**. Note the name.
2. Open the bucket's **Credentials** tab. It gives `BUCKET`,
   `ACCESS_KEY_ID`, `SECRET_ACCESS_KEY`, `REGION`, `ENDPOINT`, and tells you
   whether the bucket needs virtual-hosted or path-style addressing.
3. Wire those onto the web service as `BUCKET_NAME`, `BUCKET_ACCESS_KEY_ID`,
   `BUCKET_SECRET_ACCESS_KEY`, `BUCKET_REGION`, `BUCKET_ENDPOINT`.
4. Copy the same five into `.env.local` for development — buckets are *not* on
   the private network, so local access works with no proxy.
5. `pnpm add @aws-sdk/client-s3 @aws-sdk/s3-request-presigner`

Each environment gets its own bucket instance with its own credentials, so
preview and production don't share objects.

Add to `.env.example`:

```
BUCKET_NAME=
BUCKET_ACCESS_KEY_ID=
BUCKET_SECRET_ACCESS_KEY=
BUCKET_REGION=
BUCKET_ENDPOINT=
```

**Billing note:** bucket egress is free and unlimited, and all S3 operations
are free. But buckets are not on Railway's private network, so bytes the *app*
uploads count as service egress. Uploading directly from the browser avoids
that; proxying through the app doesn't.

## Verification

Not "it compiled" — drive it:

1. `put` a small file; confirm it appears in the Railway bucket console under
   the expected content-addressed key
2. Render it via `/img/<key>` in a browser; confirm the image loads
3. Re-`put` the identical bytes; confirm the key is unchanged and no duplicate
   object appears
4. `delete` it; confirm it's gone from the console and `/img/<key>` now 404s
5. Sign out, then request `/img/<key>` directly — confirm it redirects to
   `/login` rather than serving the image. This is the property that makes
   private images actually private, and it is worth confirming once.
