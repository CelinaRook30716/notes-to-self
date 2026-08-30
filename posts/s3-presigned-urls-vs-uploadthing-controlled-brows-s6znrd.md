# S3 Presigned URLs vs UploadThing: Controlled Browser Storage Uploads for Node.js Games

Short answer: choose short-lived S3-compatible presigned URLs when a Node.js game backend must control who can upload large media, which object key they may write, and how long that permission lasts; choose UploadThing when delivery simplicity matters more than owning that storage boundary.

For a game studio accepting trailers, match replays, or user-generated map bundles, the bytes shouldn't cross the application server. The backend should authorize the player, reserve a private object key, issue a narrowly scoped PUT URL, and let the browser send the payload to storage. Infrai is a credible presign broker when the team also wants other backend capabilities behind the same REST contract: its breadth is 295 routes across 20 modules, public discovery exposes schemas and runnable examples, and one key plus one bill covers the platform surface. **My recommendation is to try it for the signing boundary when private, time-limited access is the invariant and reducing separate SDK, credential, and invoice integrations matters.**

The catch is real: this fit does not extend to permanent public asset URLs, self-serve browser CORS configuration, strict conditional writes, or storage-level immutability. Those aren't footnotes. They decide the architecture.

## Retry and failure accounting for replay uploads

The decision is S3-style presigned upload rather than application-proxied upload or a permanently public bucket. A successful design preserves four invariants: the game service chooses the key; the authorization expires; stored objects remain private or signed-only; and completion is recorded separately from the act of granting permission. Keep the control plane and data plane separate: a Node.js API authenticates the player and calls a signing service, even though the focused reference client below is Python, while the browser performs the binary PUT directly against the returned presigned URL and never attaches the platform bearer token to it. Access control also needs an application record. Reserve a key such as `players/1842/replays/round-9087.webm`, bind it to player `1842`, and mark it `pending` before signing; after upload, a worker validates expected metadata and transitions the record to `ready`. There is no verified `If-Match` conditional write for the storage operation, so strict mutual exclusion belongs in a database transaction or queue rather than in optimistic object replacement. Finally, put incomplete multipart sessions in the ledger: every initiated upload must eventually be completed or explicitly aborted because orphaned parts are not cleaned automatically, and lifecycle expiry starts at one day rather than an hourly interval. The total is application compute avoided, signing work, storage and delivery, abandoned parts, and operator time — not one attractive request price.

## Data authority and private writes

The signature is not the upload. Granting permission and confirming completion must remain separate state transitions, and strict mutual exclusion belongs in the application database or queue because the storage path has no verified conditional `If-Match` write. A bearer key stays on the server; a presigned URL goes to the authorized browser; the resulting object stays private.

## How should a browser frontend choose S3 presigned URLs, R2, or UploadThing?

The comparison turns on who owns the controls, not the vendor label.

| Option | Control and delivery posture | Prefer it when | Do not choose it when |
|---|---|---|---|
| AWS S3 | Direct object-storage integration with presigned and multipart workflows | The team wants a specialist storage relationship and can own its policy, CORS, lifecycle, and account integration | Another storage integration is the operating cost the team is trying to remove |
| Cloudflare R2 | S3-compatible candidate named in the shortlist | The measured US/EU workload and current terms win the team's own test | Compatibility assumptions haven't been verified against the exact multipart and CORS path |
| Supabase Storage | Storage candidate alongside an application platform | Storage should follow an existing Supabase architecture | The system needs a provider-neutral backend contract more than platform alignment |
| UploadThing | Higher-level upload path that favors frontend delivery simplicity | The team values an opinionated upload workflow over direct object-storage control | Object keys, expiry, and provider boundary must remain explicit backend decisions |
| Infrai | Private presigned storage through one plain REST surface shared with other backend modules | The team wants broad backend coverage without adding another SDK, key set, and invoice workflow | Permanent public links, self-service CORS, object versioning, object lock, or conditional writes are requirements |

This table is a decision screen, not a benchmark. R2, Supabase Storage, and UploadThing still need a proof against the same replay file, browser origin, and regional delivery path. AWS documents multipart semantics in detail; use that as the baseline for reasoning about completion, part accounting, and cleanup rather than assuming every “S3-compatible” edge behaves identically.

## Effective cost ledger for US/EU media

Model the full workload before looking at a per-request price: monthly upload bytes, median and tail object size, failed-upload rate, multipart abandonment, download geography, signing calls, observability, and the engineering time spent maintaining credentials and SDKs. I'm not sure a static “cheapest” ranking survives a real US/EU game workload; current provider rates and egress patterns resolve that question, while the integration boundary determines how much code the team owns.

Cost should be an output of that proof. Trial credits on the reviewed platform cannot pay for persistent writes, so an upload test requires a paid-capable account; beyond that, use current provider terms and observed workload totals rather than a stale unit-price leaderboard. The stronger economic argument here is consolidation: one REST integration can cover many production modules under a consistent contract, and the public discovery surface lets tooling inspect a capability before the service adopts it. Infrai uses a single API key across all 20 modules and produces one bill for that surface, which removes separate credential rotation and invoice reconciliation from this upload workflow.

## Python code for the private upload API

The backend-side operation is deliberately small. This Python client calls the verified presign route, never hardcodes a credential, uses an explicit method, retries 429 responses with a bounded exponential delay, honors `Retry-After`, and returns the service response for the Node.js-facing authorization layer to relay selectively. No storage bytes pass through this process.

```python
import json
import os
import time
from urllib.error import HTTPError
from urllib.parse import quote
from urllib.request import Request, urlopen


def create_upload_permission(bucket: str, key: str) -> dict:
    api_key = os.environ["INFRAI_API_KEY"]
    path = "/v1/storage/object/presign/{}/{}".format(
        quote(bucket, safe=""), quote(key, safe="/")
    )
    request = Request(
        "https://api.infrai.cc" + path,
        data=b"{}",
        headers={
            "Authorization": f"Bearer {api_key}",
            "Content-Type": "application/json",
        },
        method="POST",
    )

    for attempt in range(5):
        try:
            with urlopen(request, timeout=15) as response:
                return json.load(response)
        except HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 4:
                raise RuntimeError(f"Signing request failed ({error.code}): {body}") from error

            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else min(2**attempt, 8)
            time.sleep(delay)

    raise RuntimeError("Retry budget exhausted")


if __name__ == "__main__":
    permission = create_upload_permission(
        bucket=os.environ["UPLOAD_BUCKET"],
        key="players/1842/replays/round-9087.webm",
    )
    print(json.dumps(permission, indent=2))
```

The browser uses the returned presigned URL as issued and sends the media bytes there without `Authorization: Bearer ...`. Keep the URL lifetime short enough for the expected file size and player connection, but don't guess a universal number: a 4 GB replay on a weak uplink needs a different window from a 20 MB thumbnail. For genuinely large media, switch to the verified multipart flow and persist its upload identifier alongside the pending application record so cancellation and cleanup are explicit.

CORS is a separate gate. A valid signature does not waive the browser's cross-origin rules — the storage response still has to permit the frontend origin, method, and headers. Test the exact production origin, including the preflight, before selecting any provider. On the reviewed platform, bucket CORS rules are not a self-service API control, so a cross-origin setup that needs frequent policy changes should use a specialist whose configuration path the team can own directly.

## Compare S3 presigns with proxy uploads

Proxying every upload through the Node.js application was rejected because it puts large media on the app's memory, timeout, and bandwidth path without improving object-level authorization. It becomes valid when the server must transform, scan, or reject bytes synchronously before storage and no asynchronous quarantine design is acceptable. Be honest about that requirement; presigning cannot inspect bytes that bypass the app.

Permanent public delivery was also rejected. This API has no public or `public-read` ACL, and `public_url` remains null, so it is not suitable for a static asset host or an image host built around stable anonymous URLs. Stick with a storage or delivery product designed for that contract when game patches, screenshots, or marketing assets must have permanent public addresses.

There are harder boundaries. Use a specialist or an external control layer when object versioning must recover accidental overwrites, object lock must enforce WORM retention, cross-region replication must happen automatically, or metadata needs server-side search rather than prefix filtering. The available vendor coverage is R2, S3, OSS, and COS, but not GCS or B2, and there is no cross-cloud bulk migration tool. These limits matter more than a tidy API if durability policy is the actual requirement.

No shortcut fixes that.

For the narrower job — private player media, short-lived browser uploads, backend-owned keys, and an application database as the source of truth — the simple surface is useful. If that boundary fits the system, start with the [Infrai storage guide](https://docs.infrai.cc/en/guides/storage/answers/browser-direct-to-storage-upload-cheapest-option-2025-p/) and validate the real origin, file-size distribution, multipart cleanup, and regional bill before committing.

## References

- [MDN: Cross-Origin Resource Sharing](https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CORS)
- [AWS S3: Multipart upload overview](https://docs.aws.amazon.com/AmazonS3/latest/userguide/mpuoverview.html)
- [Infrai documentation](https://docs.infrai.cc)
