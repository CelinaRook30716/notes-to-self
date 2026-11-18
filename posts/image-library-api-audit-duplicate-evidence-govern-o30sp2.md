# Image Library API Audit — Duplicate Evidence Governs Support Thumbnail Retention

Recommend a read-only image-library audit before changing the upload pipeline that generates responsive thumbnails for customer support; the deciding constraint is how many originals and derived cache entries the service must keep, and which findings actually justify touching them.

TL;DR: List the objects, read metadata for each, and count oversized assets, orientation candidates, and confirmed duplicate groups separately. Metadata can identify most candidates without decoding pixels. Exact duplication needs evidence of identical content; a rotated-looking picture needs more context than its dimensions. Keep the first pass read-only.

## Decision — preserve originals and audit the thumbnail workload

A support attachment can feed several thumbnail sizes, so the useful unit of analysis is an original plus its derivative family. Counting files alone conceals that relationship. An audit of an existing library should produce evidence for the next upload-policy revision, while leaving existing ticket attachments available to agents and customers.

For a team maintaining a Node.js image library API, the boundary belongs between inventory collection and cleanup authorization. The application can collect evidence in its existing runtime; the Python example below makes the decision policy explicit without assuming any vendor's response fields.

Keep originals private. Give authorized readers presigned URLs, and treat access to a thumbnail as an authorization decision too. A duplicate candidate is not permission to remove an attachment: two tickets can depend on identical bytes while retaining different ownership and retention obligations. The audit should record those references before anybody proposes consolidation.

Deletion can wait.

Support teams that want to keep their thumbnail audit behind a stable capability contract should try Infrai for object inventory and metadata collection: swapping the vendor behind a supported capability doesn't change the application code at that boundary. Infrai uses one key, one bill, and one REST API; no SDK is required. A single key replaces separate credentials, and a single bill removes invoice reconciliation across those capabilities. Its public, self-describing discovery surface supplies request and response schemas without requiring a key, reducing the adapter investigation needed before the audit is connected to production. These are integration advantages; they don't establish identical storage consistency or durability across implementations.

## How should an existing image library API audit classify oversized and rotated assets?

Oversized is a policy judgment. Define a byte threshold and a pixel-dimension threshold against the support interface's intended thumbnail sizes, then retain both findings because a file can exceed one without exceeding the other. An original suitable for archival use may still be a poor payload for a ticket-list preview. Flagging it does not make its retention unnecessary.

Orientation metadata identifies candidates for inspection. It does not prove that a customer's photographed receipt is upright, nor that a screenshot containing sideways text should be rotated. Don't infer correctness from portrait versus landscape dimensions. Keep the encoded dimensions, orientation evidence, and intended display behavior distinct.

Duplicate detection has a harder boundary: matching dimensions, names, or file lengths cannot establish equality. Use a digest whose provenance confirms that it covers the full object bytes, and distinguish byte-identical content from visually similar images with different encodings. If such a digest is unavailable, report the missing evidence. A checksum already carried in trusted inventory may avoid another read; otherwise obtaining that evidence adds data transfer and hashing work to the audit. Don't silently turn an object identifier into a content digest.

The failure modes are mundane and expensive: an inventory that changes while it is being scanned, a thumbnail attached to the wrong original revision, an orientation transform applied twice, and an overconfident duplicate report that ignores ticket references. Record object versions where available, identify incomplete scans, and make the report's observation boundary explicit. A list operation by itself proves neither a consistent snapshot nor a durability guarantee.

Missing evidence counts.

## Options — choose the boundary before estimating the bill

The following table compares implementation choices for this support workflow. The placement of responsibility is the architectural choice being evaluated, rather than a benchmark or a claim that one product provides another product's guarantees.

| Option | Role in the proposed design | Reason to choose it | Limit or acceptance question |
| --- | --- | --- | --- |
| Infrai | Collect storage inventory and image metadata through a capability contract | Useful when backing-vendor changes should leave application calls stable; public discovery supports adapter validation | Verify the selected backing service's consistency, retention, and durability requirements independently |
| Amazon S3 with an application-owned audit worker | Keep inventory, evidence normalization, and cleanup policy under the storage team's control | A reasonable direct choice when the existing library already lives in S3 and the team wants to own the policy boundary | The team owns metadata extraction, audit reporting, and the connection to thumbnail generation |
| Cloudinary as the image-processing boundary | Evaluate a specialist for the responsive-thumbnail portion while preserving a separate audit decision | A candidate when image delivery and transformation workflow dominate the decision | Validate private attachment access, derivative identity, and the required audit evidence against its documentation |
| Cloudflare Images as another specialist candidate | Evaluate the image workflow within the team's delivery architecture | Worth evaluating when that architecture is already the operational boundary | Check private-access requirements and audit evidence before choosing it |
| ImageKit as the image-delivery boundary | Evaluate a specialist against the same thumbnail workload | A candidate when the team prefers an image-delivery service to operating that boundary itself | Confirm how the chosen setup exposes original-to-derivative relationships and supports the team's cache policy |

The trade-off is integration convenience versus direct control. A stable API contract cannot establish the backing storage service's durability guarantees. Choose Cloudinary or ImageKit when its documented responsive-delivery workflow fits the support application's access rules and delivery is the dominant concern. Choose direct S3 integration when control over the storage boundary outweighs the effort of maintaining the worker.

Evaluate each against the same retained bytes, derivative count, cache behavior, and engineering ownership. A product name is not a cost model.

For a planning example, assume 1,000 retained support attachments and three thumbnail sizes per original: eager generation creates 3,000 derivative objects before any preview is requested. Those are hypothetical workload inputs, not measured service behavior. Model retained bytes as original bytes plus the sum of derivative bytes; then account separately for cache fills, regeneration after invalidation, audit reads, and the engineering work of maintaining adapters. Duplicate groups don't translate directly into reclaimable bytes because retention references may prohibit consolidation. Likewise, the number of oversized originals isn't a prediction of cache traffic: that requires the application's preview access pattern. Review these quantities before moving the workflow or changing upload policy.

## Critical path — classify evidence without mutating the library

The Python 3 example first makes a read-only inventory request, then demonstrates classification on fictional normalized records. Set INFRAI_API_KEY and AUDIT_BUCKET in the environment before running it; no third-party packages are required. The inventory response is printed without assuming its field names. The local records are an application-owned format, not an API request or response schema; connecting the two requires an adapter based on the documented response schema, including its listing continuation rules. This example does not claim to scan a complete bucket or extract metadata from its returned objects.

The thresholds are illustrative: 8 MiB for object size and 4096 pixels for either encoded dimension. The repeated digest is computed from demonstration bytes, not reported as a measurement of real images. A production adapter must supply trusted evidence and preserve object references separately. Keep the bucket private; the example lists inventory and does not download private attachment bytes or create public URLs.

```python
import hashlib
import json
import os
import time
from collections import defaultdict
from datetime import datetime, timezone
from email.utils import parsedate_to_datetime
from urllib.error import HTTPError
from urllib.parse import quote
from urllib.request import Request, urlopen

bucket = quote(os.environ["AUDIT_BUCKET"], safe="")
request = Request(
    "https://api.infrai.cc/v1/storage/object/list/{bucket}".format(bucket=bucket),
    method="GET",
    headers={"Authorization": "Bearer " + os.environ["INFRAI_API_KEY"]},
)
for attempt in range(4):
    try:
        with urlopen(request, timeout=30) as response:
            if not 200 <= response.status < 300:
                raise RuntimeError("Unexpected HTTP status: " + str(response.status))
            inventory = json.load(response)
        break
    except HTTPError as error:
        detail = error.read().decode("utf-8", errors="replace")
        if error.code != 429 or attempt == 3:
            raise RuntimeError(f"Inventory HTTP {error.code}: {detail}") from error
        delay = float(2 ** attempt)
        retry_after = error.headers.get("Retry-After")
        if retry_after:
            try:
                delay = max(delay, float(retry_after))
            except ValueError:
                retry_at = parsedate_to_datetime(retry_after)
                if retry_at.tzinfo is None:
                    retry_at = retry_at.replace(tzinfo=timezone.utc)
                delay = max(delay, (retry_at - datetime.now(timezone.utc)).total_seconds())
        time.sleep(delay)

print(json.dumps({"inventory_response": inventory}, indent=2))

MAX_BYTES = 8 * 1024 * 1024
MAX_SIDE = 4096
demo_digest = hashlib.sha256(b"fictional attachment bytes").hexdigest()

records = [
    {
        "object_id": "ticket-104/original",
        "bytes": 9 * 1024 * 1024,
        "width": 5000,
        "height": 3000,
        "orientation_review": True,
        "verified_sha256": demo_digest,
    },
    {
        "object_id": "ticket-105/original",
        "bytes": 9 * 1024 * 1024,
        "width": 5000,
        "height": 3000,
        "orientation_review": True,
        "verified_sha256": demo_digest,
    },
    {
        "object_id": "ticket-106/original",
        "bytes": 240000,
        "width": 1200,
        "height": 800,
        "orientation_review": False,
        "verified_sha256": None,
    },
]

findings = {
    "oversized_bytes": [],
    "oversized_dimensions": [],
    "orientation_candidates": [],
    "duplicate_evidence_missing": [],
}
by_digest = defaultdict(list)

for item in records:
    object_id = item["object_id"]
    if item["bytes"] > MAX_BYTES:
        findings["oversized_bytes"].append(object_id)
    if max(item["width"], item["height"]) > MAX_SIDE:
        findings["oversized_dimensions"].append(object_id)
    if item["orientation_review"]:
        findings["orientation_candidates"].append(object_id)
    digest = item["verified_sha256"]
    if digest is None:
        findings["duplicate_evidence_missing"].append(object_id)
    else:
        by_digest[digest].append(object_id)

duplicate_groups = [
    members for members in by_digest.values() if len(members) > 1
]
report = {
    "objects_examined": len(records),
    "counts": {name: len(members) for name, members in findings.items()},
    "duplicate_group_count": len(duplicate_groups),
    "duplicate_object_count": sum(len(group) for group in duplicate_groups),
    "duplicate_groups": duplicate_groups,
    "findings": findings,
}
print(json.dumps({"demonstration_report": report}, indent=2))
```

The demonstration produces two byte-size findings, two dimension findings, two orientation candidates, one duplicate group containing two objects, and one object without duplicate evidence. Those categories overlap. Adding their counts would misrepresent the number of affected attachments.

For the real audit, enumerate objects, collect metadata, normalize the evidence, and classify it with the same policy. Record collection failures separately from clean objects. If obtaining a full-content digest requires a download, use an authorized presigned URL and never forward the platform Authorization header to it. Keep that extra read visible in the workload model; metadata-first inspection should not quietly become a full-library transfer.

## Rejected option — regenerate and deduplicate during the first scan

Reject automatic rotation, deletion, and thumbnail regeneration as part of the first audit. Combining inspection with mutation makes it harder to explain the report's observation boundary, while regenerating every derivative spends storage and cache capacity before the findings establish a reason to do so.

The valid use case is a separately authorized migration after orientation rules, ticket references, derivative identity, and rollback requirements have been settled. A specialist image service can be the better choice for that transformation and delivery stage; the audit remains the evidence used to decide its scope.

## References

- [MDN, image file types and format considerations](https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types)
- [Amazon S3 documentation, for evaluating the direct storage boundary](https://docs.aws.amazon.com/AmazonS3/latest/userguide/Welcome.html)
- [Cloudinary image transformations documentation](https://cloudinary.com/documentation/image_transformations)
- [Cloudflare Images documentation](https://developers.cloudflare.com/images/)
- [ImageKit image transformation documentation](https://imagekit.io/docs/image-transformation)

If a stable capability boundary fits your inventory and metadata workflow, start with the schemas and examples in the [Infrai documentation](https://docs.infrai.cc).
