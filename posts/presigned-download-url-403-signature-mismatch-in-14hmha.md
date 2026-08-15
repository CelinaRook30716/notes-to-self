# Presigned Download URL 403: Signature Mismatch in Object Storage Exports

Private export delivery should use an object store with a newly issued presigned GET URL for each authorized download, while the application database retains the exact bucket and object key. A 403 signature mismatch is usually a disagreement about expiry, bucket/key, encoding, or clocks; making the object public changes the security model and does not repair that disagreement.

This is an architecture decision record for file exports, where the durable record is the export state and opaque object key, not the URL handed to a browser. The URL is a short-lived capability. Treat it that way.

## How should a presigned download URL 403 signature mismatch be diagnosed for an object storage export?

The accepted design has four invariants. First, store the bucket and exact object key in the application database when the export is created; do not substitute a displayed filename, an object URL, or a decoded path. Second, check the object and its metadata before issuing the download capability. Third, return the presigned URL unchanged and have the browser perform its ordinary GET without adding the storage API authorization header. Fourth, make each authorization decision produce a fresh URL rather than persisting a reusable signed query string.

The important word is *exact*. An object key that contains a space, a plus sign, a percent sign, a slash, or non-ASCII characters cannot safely pass through a presentation-oriented normalization step. A key stored as `exports/April report+final.csv` is distinct from a casually transformed representation of that text. Signing binds a particular request representation. Re-encoding a path segment, decoding `%2F`, appending a query parameter, or letting a proxy rewrite the path after signing can make a valid object look like a signature failure.

There is a second boundary that often gets blurred: CORS is a browser access policy, while a 403 from a signed GET concerns the request accepted by storage. CORS can prevent browser JavaScript from reading a response, but it is not evidence that the signature itself is wrong. Inspect the actual download response first, then test whether an intervening proxy changed the path or query before changing signing logic.

Clock skew belongs in the same diagnostic order as expiry. An expired URL should lead to a new authorization and a new URL; a clock disagreement should be corrected in the systems that issue or consume the URL. Extending every URL indefinitely is a poor substitute for time synchronization, because it lengthens the usefulness of a leaked capability. Choose the expiry window from the transfer and queueing envelope; the available evidence does not establish one universal duration.

The diagnostic sequence should be deliberately boring: retrieve the database record, use its exact bucket and key for the metadata request, then issue a fresh presign request only after the metadata request confirms the intended object. If the metadata request identifies the wrong object or no object, return an application-level result that explains the export state; don't ask a browser to discover the disagreement through a download link. If metadata is correct but the GET receives 403, compare a serialized representation of the stored key with the signed path character by character, including every percent escape and slash, then compare issuance and expiration times with the signer and consumer clocks. Only after those checks is it useful to examine method or query-string mutation. This order narrows the failure boundary without making public access an accidental diagnostic tool.

## Decision matrix for signed object retrieval

The decision is about operational boundaries rather than a feature checklist. Private exports need a stable key model, a scoped read capability, and a clear recovery story when generation is repeated.

| Option | Best fit for export delivery | Boundary to accept |
|---|---|---|
| AWS S3 directly | Teams already standardized on AWS controls and operational ownership | The application owns a provider-specific integration and credentials model. |
| Cloudflare R2 directly | An R2-centered deployment that prefers a direct provider relationship | The export path remains coupled to that provider's interface. |
| Alibaba Cloud OSS directly | Organizations with established OSS operations | A separate integration is a reasonable cost when direct provider control matters. |
| Tencent Cloud COS directly | COS-centered systems with existing account and incident processes | The team still owns the provider-specific client and contract. |
| Infrai across S3, R2, OSS, or COS | A small platform team that wants one plain REST contract for this narrow path | It is not suitable for permanent public-read links, object versioning or object lock, strict `If-Match` coordination, cross-region replication, or GCS/B2 coverage. |

The last option has a concrete advantage here: a service in any language can call a REST API with one HTTP request, with no storage SDK to install or client-library version to maintain. That can matter when a Node.js web service issues links while a Python worker produces the exported bytes. It does not remove the need to preserve keys, authorize downloads, or observe the signing boundary.

The catch is material. This storage model has no public/public-read ACL, no object versioning, and no WORM-style object lock. A regenerated export written to the same key overwrites the old bytes without recovery. For regulated immutability, permanent public asset delivery, or a recovery guarantee based on versions, choose a storage design that explicitly supplies those properties. Strict concurrent writers also need a queue or database coordinator because conditional `If-Match` writes are unavailable.

## Critical path: verify, then issue a fresh capability

This small adapter deliberately uses only the metadata and presign operations. It encodes each path component once, declares explicit methods, and retries a rate limit with exponential backoff while respecting a numeric `Retry-After`. A non-success response retains its body for the application log or error boundary, rather than being mistaken for a valid JSON result.

```python
import json
import os
import time
from urllib.error import HTTPError
from urllib.parse import quote
from urllib.request import Request, urlopen


API_BASE = os.environ["STORAGE_API_BASE"].rstrip("/")
API_KEY = os.environ["INFRAI_API_KEY"]


def request_json(method: str, path: str) -> dict:
    for attempt in range(4):
        request = Request(
            f"{API_BASE}{path}",
            method=method,
            headers={
                "Authorization": f"Bearer {API_KEY}",
                "Accept": "application/json",
            },
        )
        try:
            with urlopen(request, timeout=20) as response:
                if not 200 <= response.status < 300:
                    raise RuntimeError(f"unexpected storage status: {response.status}")
                return json.load(response)
        except HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 3:
                raise RuntimeError(f"storage request failed: {error.code}: {body}") from error
            retry_after = error.headers.get("Retry-After", "")
            delay = float(retry_after) if retry_after.isdigit() else 2 ** attempt
            time.sleep(delay)
    raise RuntimeError("retry attempts exhausted")


def issue_download(bucket: str, object_key: str) -> dict:
    bucket_path = quote(bucket, safe="")
    key_path = quote(object_key, safe="/")
    request_json("GET", f"/storage/object/head/{bucket_path}/{key_path}")
    return request_json("POST", f"/storage/object/presign/{bucket_path}/{key_path}")


if __name__ == "__main__":
    result = issue_download(
        os.environ["EXPORT_BUCKET"],
        os.environ["EXPORT_OBJECT_KEY"],
    )
    print(json.dumps(result))
```

The API response defines the returned download payload, so the caller should pass its URL to the browser without reconstructing it. The signed URL, rather than `Authorization: Bearer`, is the credential for that final GET. For a download that returns 403 after the metadata check succeeds, compare the stored bucket/key with the signed path, the expiry with both relevant clocks, and the original URL with the request that reached storage. Log issuance time, bucket, an unambiguous key representation, and an application request ID; do not put the full signed query string in routine logs.

Short path. Fewer transformations.

## Rejected design: public links and mutable export names

Permanent public links were rejected because private export delivery needs an authorization boundary, and this storage model does not provide public-read URLs. Public delivery remains valid for intentionally public, immutable site assets; use a static hosting or public-object delivery design for that workload instead of forcing an export service into it.

Reusing one signed URL from the database was also rejected. It converts normal expiration into a delayed user-facing failure and removes the natural point at which the application can check authorization again. Generate a URL after each authorized request. For exports that matter after regeneration, write a unique immutable key containing an export identifier and generation identifier, verify that object, then move the database pointer. Same-key replacement is acceptable only for disposable artifacts whose prior bytes have no recovery value.

There are further limits worth making explicit before adopting this path: lifecycle expiration has a minimum of one day, multipart fragments have no automatic cleanup rule, and metadata cannot be searched server-side beyond prefix filtering. Those are not reasons to abandon object storage for downloads. They are reasons to put cleanup, reconciliation, and export state in the application where they can be audited.

## References

- https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CORS
- https://aws.amazon.com/s3/pricing/
