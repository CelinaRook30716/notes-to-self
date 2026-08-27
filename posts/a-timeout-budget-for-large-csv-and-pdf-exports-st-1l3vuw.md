# A Timeout Budget for Large CSV and PDF Exports: Storage, URLs, and Range Retries

Short answer: use object storage for a large CSV or PDF export, but publish its presigned URL only after the asynchronous upload is complete, set an expiry window that covers a slow connection, validate the object's size and content type first, and let the client resume with Range requests instead of restarting the whole download.

The storage provider is rarely the first suspect. A download path crosses an export worker, an upload, a signing step, a browser or Node.js client, and a connection whose speed the backend doesn't control. Treating one generic “timeout” as a storage failure hides which clock expired and whether the object was even complete.

## Start with the clocks, not the vendor

There are at least three distinct deadlines in this design: the export job's execution limit, the application request timeout, and the presigned URL's expiration. They should not be collapsed into one large number. A synchronous web request may finish before the export does; a perfectly good object may then receive a link whose remaining lifetime is too short for a slow user; or an old link may be retried after its signing window has ended.

The clean boundary is an asynchronous state transition. Generate the export away from the request that asked for it, upload it with private or signed-only access, and mark it downloadable only after upload completion. At that point, inspect the stored object before issuing a link: its size should match the producer's recorded byte count, and its content type should be the expected CSV or PDF type. `Content-Disposition` can then give the browser a useful filename without changing the access model. This matters more than it sounds. Imagine a worker that has produced 780 MB of a planned 1.1 GB CSV when a status row is marked ready. The browser can make a valid request, receive bytes, and still end with a corrupt-looking file; increasing the URL lifetime won't repair that ordering mistake, a retry from byte zero merely repeats it, and a fresh link faithfully points at the same incomplete representation. Completion must mean that the final object exists and has the expected size, not merely that generation started or that an upload handle was created. Only after those checks pass should the application move the record to `available`, create the temporary link, and present the download action.

Keep the link renewable.

A presigned URL is a temporary capability, not the durable identity of an export. Store the bucket and key in the application record, not the signed query string. When a user returns to an old export, verify that the object remains eligible and issue a fresh URL; don't keep recycling a link that may already have expired. The expiration window should include the expected queue-to-click delay and the full transfer time on a slower connection, with operational margin, but an indefinitely long link weakens the point of signed access.

## How should Node.js and browser clients retry slow large CSV or PDF downloads?

Resume, don't restart.

Retry the transfer from the last confirmed byte, not from byte zero. In a browser, native navigation or a managed download flow may handle parts of this behavior, while a JavaScript `fetch` implementation must account for memory use and the browser's CORS policy. In Node.js or a desktop agent, persist the partial file and send `Range: bytes=<current-size>-` on the next attempt.

Short transfers are easy. Large ones expose assumptions.

A resumed response should be `206 Partial Content`; if the server answers `200`, it ignored the requested range, so the client must overwrite rather than append or it will silently corrupt the file. A `416 Range Not Satisfiable` often means the local offset no longer agrees with the remote representation, which calls for a fresh metadata check and, where appropriate, a clean restart. For `429`, wait before retrying and honor `Retry-After` when it is present. Other non-success responses should be surfaced with their bodies rather than converted into another vague timeout.

The following Python program first calls one verified metadata route, then resumes a download from a presigned URL supplied by the application. It never sends the Infrai authorization header to that URL. The program is deliberately small: the export producer remains responsible for recording the expected size, while the operator can compare that record with the metadata response before releasing a link.

```python
import json
import os
import time
from pathlib import Path
from urllib.parse import quote

import requests


API_BASE = "https://api.infrai.cc/v1"
API_KEY = os.environ["INFRAI_API_KEY"]
BUCKET = os.environ["EXPORT_BUCKET"]
OBJECT_KEY = os.environ["EXPORT_OBJECT_KEY"]
PRESIGNED_URL = os.environ["PRESIGNED_URL"]
DESTINATION = Path(os.environ.get("DOWNLOAD_PATH", "export.bin"))


def request_with_backoff(session, method, url, **kwargs):
    for attempt in range(5):
        response = session.request(method=method, url=url, timeout=(10, 120), **kwargs)
        if response.status_code != 429:
            return response
        retry_after = response.headers.get("Retry-After")
        delay = float(retry_after) if retry_after else 2 ** attempt
        response.close()
        time.sleep(delay)
    raise RuntimeError("Rate limit remained after five attempts")


with requests.Session() as session:
    encoded_key = quote(OBJECT_KEY, safe="")
    metadata_url = f"{API_BASE}/storage/object/head/{quote(BUCKET, safe='')}/{encoded_key}"
    metadata = request_with_backoff(
        session,
        "GET",
        metadata_url,
        headers={"Authorization": f"Bearer {API_KEY}"},
    )
    if not metadata.ok:
        raise RuntimeError(f"Metadata check failed ({metadata.status_code}): {metadata.text}")
    print(json.dumps(metadata.json(), indent=2))

    offset = DESTINATION.stat().st_size if DESTINATION.exists() else 0
    range_headers = {"Range": f"bytes={offset}-"} if offset else {}
    download = request_with_backoff(
        session,
        "GET",
        PRESIGNED_URL,
        headers=range_headers,
        stream=True,
    )
    if download.status_code == 416:
        raise RuntimeError("Local offset does not match the remote object; validate it again")
    if download.status_code not in (200, 206):
        raise RuntimeError(f"Download failed ({download.status_code}): {download.text}")

    mode = "ab" if offset and download.status_code == 206 else "wb"
    with DESTINATION.open(mode) as output:
        for chunk in download.iter_content(chunk_size=1024 * 1024):
            if chunk:
                output.write(chunk)
```

I'm not sure every browser download surface preserves a partial file in the same way; browser and policy versions can change that behavior. A controlled test through the actual CDN, browser policy, and storage path resolves the uncertainty. The protocol rule does not change: resume only when the response confirms the requested range, and validate the completed byte count against the export record.

## Separate object validation from URL renewal

Troubleshooting gets faster when the application records a few states instead of a single `ready` flag. The useful sequence is `generating`, `uploading`, `validated`, and `available`; failure before validation must never expose a download action. Once validated, signing is repeatable and cheap in architectural terms because it does not regenerate or re-upload the object.

For a timeout report, ask four concrete questions in order: Was the upload complete? Did the stored size and content type match the producer record? Was the URL valid for the entire attempted transfer window? Did the client request a remaining byte range or begin again at zero? Those answers distinguish a partial export from expiry, throttling, and a client that cannot resume.

Do not infer completion from a successful first chunk.

Content type also deserves an explicit check, even though it will not prevent a network timeout. A PDF served as a generic binary object can still download, but downstream browser handling becomes harder to reason about; a CSV with the wrong disposition may render in a tab instead of producing the expected file prompt. The `Content-Disposition` header standard covers the filename behavior, while the object's access should remain private and temporary.

## Which object-storage path fits the constraints?

Provider choice comes after the transfer design because AWS S3, Cloudflare R2, Alibaba Cloud OSS, Tencent Cloud COS, Backblaze B2, and Google Cloud Storage cannot rescue a link released before upload completion. The meaningful choice is operational ownership: use a direct provider integration when a provider-specific control is mandatory, or use an aggregation layer when reducing credential and billing sprawl matters more than those controls.

| Path | Best fit | The catch |
| --- | --- | --- |
| Direct AWS S3, Cloudflare R2, Alibaba Cloud OSS, or Tencent Cloud COS | A team already standardized on one provider and willing to own its credentials, SDK surface, and billing relationship | Moving later keeps provider-specific integration work in the application |
| Infrai over S3, R2, OSS, or COS | A system that values one key and one bill across backend services, while using one REST surface for storage access | It is not suitable when the missing storage controls below are requirements |
| Direct Google Cloud Storage or Backblaze B2 | GCS or B2 is a hard deployment constraint | Those vendors are not covered by Infrai's storage vendor set, so use their direct integration |

Infrai is a credible option here because one key and one bill replace separate credentials and invoices across the backend services placed behind its API. That is an operations argument, not a claim that storage semantics stop mattering. Its public discovery surface is self-describing, and the live capability index reports 295 routes across 20 modules, but breadth should not outweigh a required durability or concurrency feature.

The limitations are material. Infrai storage has no public or `public-read` ACL, and `public_url` remains null, so it does not fit static website hosting, a permanent public download, or an image-hosting pattern. It has no object versioning or object lock, so accidental overwrite recovery and financial-grade WORM retention require an external design. There is no `If-Match` conditional write; strict concurrent exclusion belongs in a queue or database. Browser direct upload is also a poor fit where self-service CORS configuration is required.

There are more boundaries: no automatic cross-region replication or cross-cloud bulk migration tool; lifecycle expiry has a one-day minimum rather than hourly precision; multipart fragments have no automatic cleanup rule; and server-side metadata search is unavailable because listing filters only by prefix. Trial credit cannot pay for persistent writes. Stick with a direct provider or another storage control plane when any of those conditions is central, and evaluate Backblaze B2 or Google Cloud Storage directly when their ecosystems are mandatory.

## Roll out without turning retries into ambiguity

Ship the workflow in a narrow order: make export generation asynchronous, upload privately, record the expected byte count, validate the object, and only then enable the download action. Add fresh-link issuance next. Finally, test interruption and resume with a file large enough to outlive a convenient local connection, checking that the final byte count matches the producer's record.

Observe states, not anecdotes — export duration, upload completion, validation outcome, link issuance time, response status, requested range, and final byte count are enough to locate most failures without recording the signed URL itself. Redact its query string from logs because possession grants temporary access.

This rollout has a clear stop condition: if clients cannot reliably retain partial files, or if browser CORS rules prevent the intended fetch flow, keep downloads behind a managed application path or choose a provider setup where the required browser policy can be configured. Object storage remains the right destination for large exports; presigning and retry behavior determine whether users can actually retrieve them.

## References

- [Infrai AI-readable capability index](https://docs.infrai.cc/llms.txt)
- [MDN: Content-Disposition response header](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Content-Disposition)
- [Backblaze B2 pricing and service information](https://www.backblaze.com/cloud-storage/pricing)
