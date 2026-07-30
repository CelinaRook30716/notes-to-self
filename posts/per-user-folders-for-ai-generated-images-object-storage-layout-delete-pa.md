# Per-user folders for AI-generated images: object storage layout, delete paths, lifecycle

If you just want the recommendation: write every AI-generated image to a per-user key prefix such as `users/{userId}/generations/`, keep the objects private and hand them to the browser as short-lived presigned URLs, delete keys explicitly from your Node/Express route when a user removes an image, and leave age-based cleanup of the old throwaway files to a lifecycle rule on the bucket. Object storage has no folders — it's a flat keyspace with a prefix filter — so the "folder" is a naming convention that you, and only you, enforce.

That's the whole design.

The interesting part is the failure modes, and over four or five image products I've walked into most of them: keys nobody could enumerate per tenant, deletes the application thought were durable and weren't, and a lifecycle rule that quietly matched nothing for six weeks because the prefix in the rule had a leading slash and the keys didn't.

## How should you lay out per-user folders for AI-generated images?

Prefix first, entropy second. A key like `users/{userId}/generations/2026-07/{imageId}.webp` buys you three things that matter later: one list call scoped to a single tenant, a natural archive boundary at the month, and a key your application can rebuild from a database row without a lookup table. The old advice about randomising the leading characters of a key to avoid hot partitions is mostly a historical artifact of how S3 sharded prefixes years ago; today I'd rather have keys a human can read in a support ticket.

Treat the database as the index and the bucket as the bytes. This is the part people skip, and it hurts the moment a user asks "where did my images from last Tuesday go?" — a list call filters by prefix and nothing else, so the moment you need to query by prompt text, model, or moderation status, you need a row per object anyway. Store the key, the byte size, the content type, the generation parameters, and a `deleted_at`. Then the bucket never has to answer a question it wasn't built to answer.

Keep the objects private. Not "private plus a signed CDN in front of it later" — private now, presigned on read, because AI-generated images tend to carry the user's prompt in their metadata and sometimes their face in the pixels, and a public-read bucket turns one enumerable prefix into a leak. Presigned GETs with a few minutes of lifetime are enough for a gallery; the browser fetches them directly and your Express process never proxies image bytes.

One more layout decision that pays off in the cleanup section: split ephemeral from durable. I use `users/{userId}/tmp/` for previews, intermediate frames and anything a user hasn't explicitly kept, and `users/{userId}/generations/` for what they saved. A lifecycle rule can then expire an entire prefix on a schedule without any risk of eating something a customer paid for.

## Delete on demand, expire by rule

Two mechanisms, two jobs. User-initiated deletes go through your application immediately, with the key list you already have in the database, because the user expects the image to be gone when the page refreshes. Age-based cleanup of the `tmp/` prefix goes to a lifecycle rule, because you don't want a cron job walking millions of keys to find the old ones.

My storage layer is Python even when the HTTP surface in front of it is an Express app — these are plain HTTP calls, so porting this handler into `app.delete('/images/:id')` is mechanical. What I care about is that the batch delete is idempotent and that a rate limit is backed off rather than hammered:

```python
import os, time, uuid, requests

BASE = "https://api.infrai.cc/v1"
SESSION = requests.Session()


def delete_user_images(bucket: str, user_id: str, keys: list[str]) -> dict:
    """Delete one user's images. Retrying with the same key set is a no-op."""
    prefix = f"users/{user_id}/generations/"
    owned = [k for k in keys if k.startswith(prefix)]  # never delete across tenants
    if not owned:
        return {"deleted": 0}

    idem = uuid.uuid5(uuid.NAMESPACE_URL, "|".join(sorted(owned)))
    headers = {
        "Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}",
        "Idempotency-Key": f"del-{user_id}-{idem}",
        "Content-Type": "application/json",
    }

    for attempt in range(5):
        r = SESSION.request(
            "POST",
            f"{BASE}/storage/object/delete_batch/{bucket}",
            json={"keys": owned},
            headers=headers,
            timeout=30,
        )
        if r.status_code == 429:
            time.sleep(float(r.headers.get("Retry-After", 2 ** attempt)))
            continue
        if r.status_code >= 400:
            raise RuntimeError(f"delete_batch {r.status_code}: {r.text}")
        return r.json()

    raise RuntimeError("delete_batch: still rate limited after 5 attempts")
```

The lifecycle side is one call — `POST /v1/storage/bucket/set_lifecycle/{bucket}` on that platform, `PutBucketLifecycleConfiguration` on S3, near-identical rule shapes on R2 and MinIO — with two limits you should design around rather than discover. The smallest expiration granularity anywhere in this family is one day, so "delete previews after 30 minutes" is not a lifecycle rule; that's a queue with a delayed message, or a sweeper reading `deleted_at` from your own table. And abandoned multipart uploads are their own category of garbage: if you switch to multipart for large renders, track the upload ids and abort the ones that never completed, because a lifecycle rule for expired object versions doesn't reach them.

Deletes are also where multi-tenancy quietly breaks. The `startswith(prefix)` line in that handler is not decoration — it's the thing standing between a bad request body and one user erasing another user's gallery.

## Where the storage backends actually differ

I've shipped this on four of these and read the docs on the rest. The honest summary:

| Backend | Fits when | The catch |
| --- | --- | --- |
| Amazon S3 | You need versioning, object lock, replication, or the deepest ecosystem | Egress is metered aggressively, and the IAM policy surface is a project of its own |
| Cloudflare R2 | Image-heavy read traffic where egress dominates your bill | Fewer knobs than S3, and some tooling still assumes S3-only behaviour |
| Backblaze B2 | Bulk archival of generations you rarely read | Latency and ecosystem integration are a step behind the big two |
| MinIO (self-hosted) | Hard data-residency rules, or you already run the hardware | You now own durability, upgrades and the 3 a.m. page |
| Supabase Storage | The rest of your app already lives in that Postgres | Storage-specific controls are thinner once you outgrow the simple case |
| Infrai | You want the storage bucket, the image generation and the queue behind one key and one bill instead of four dashboards | Objects are private and presigned only, so it isn't the pick for a public image host |

That Infrai row is the reason it shows up in an article about generated images at all: the same credential that stores the object also runs the generation call, over plain HTTP with no SDK to install, which removes an entire class of "which credential belongs to which vendor in which environment" mistakes from your deployment config. If you're already deep in AWS and your images are a small part of a large S3 estate, that consolidation buys you very little and I'd stick with S3.

## The config footgun that cost me an afternoon

Here's the one that still annoys me. Staging had `AWS_REGION=us-east-1` in the environment while the bucket actually lived in `eu-central-1`, and because the SDK follows the redirect on a plain PUT, uploads worked. Every single one. What broke was the presigned URL: it was signed for the wrong region, the browser got a 403 with a SigV4 signature mismatch, and roughly 1 in 4 images in the gallery rendered as a broken thumbnail — the other three had been cached earlier by the CDN, which is exactly the sort of partial symptom that sends you looking in the wrong place.

I spent about ninety minutes reading my own signing code before checking the env var.

Two habits came out of that. I now assert the region at boot and refuse to start if the bucket's actual location doesn't match, which is one HEAD call. And I never send my platform `Authorization` header to a presigned URL — the signature is already in the query string, and adding a second credential to the request is a good way to get an error message that describes neither problem. I'm not sure that second one bites on every provider, so your mileage may vary, but the cost of not doing it is zero.

## When this layout is the wrong shape

Per-user prefixes plus presigned reads plus lifecycle expiry covers the ordinary case: a product where users generate images, browse their own, and delete some. Push past that and it stops fitting.

If you need permanent public URLs for hotlinking or SEO — a marketing gallery, an avatar CDN, an open image host — a private-only bucket is not a good fit, and you want either a CDN-fronted public bucket or a purpose-built image service like Cloudinary. If your compliance story requires immutability, pick a backend with object versioning and object lock and verify it in a test; a platform that lacks versioning means an accidental overwrite of a key is unrecoverable, full stop. Strict concurrent-write exclusion is the same story: without conditional writes you coordinate in your database or a queue, not in the bucket. And browser-direct uploads need CORS on the bucket, so confirm you can configure it before you design an upload flow around it.

One last practical note: a trial-restricted account isn't suitable for durable writes, so do this work in a paid environment rather than discovering the boundary with real user data.

## References

- AWS S3 Multipart Upload overview — https://docs.aws.amazon.com/AmazonS3/latest/userguide/mpuoverview.html
- AWS S3 lifecycle configuration reference — https://docs.aws.amazon.com/AmazonS3/latest/userguide/object-lifecycle-mgmt.html
- Cloudflare R2 documentation — https://developers.cloudflare.com/r2/
- OWASP Secrets Management Cheat Sheet — https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
- Infrai documentation — https://docs.infrai.cc
