# Next.js API Route for Private PDF Export Signed Download URLs

Private PDF export delivery is an authorization problem before it is a storage problem. **Short answer:** have a Next.js API route authorize an opaque export identifier, read the record's fixed US or EU placement, and mint a short-lived signed download URL only for that recorded object; the browser should never select a bucket, region, or object key.

That division matters because a signed URL is a bearer capability. It can be appropriate for a large immutable PDF, where application servers should not carry every byte, but it does not replace the tenant check that created it. A region field is similarly not a preference supplied by the client. It is a placement constraint stored with the export and enforced by the service that issues the capability.

Small boundary. Large consequence.

## How should a Next.js API route create a private PDF signed download URL?

Treat the route as a policy decision point. It authenticates the caller, scopes the export lookup to that caller's tenant, checks that the export is eligible for delivery, then selects the signing client from the export record. The API contract can return JSON for a client that opens the link, or issue a redirect when the client needs no metadata. In both cases, response caching deserves an explicit rule: MDN defines `private` as a response directive for private caches and `no-store` as a directive not to store the response. For a route that returns a credential-bearing URL, `Cache-Control: private, no-store` is the conservative default.

The object key must come from durable application metadata. Do not accept `key`, `bucket`, `region`, or a filename as request parameters and transform them into a storage read. A readable filename can be output metadata; a storage key should be an internal value that is neither guessed nor disclosed to an unauthorized caller. This also prevents an otherwise ordinary retry, bookmark, or copied request from becoming a cross-tenant object selector.

Here is the server-side shape in Python. The Next.js handler supplies its own session and deployment configuration, while this narrow interface makes the authority boundary testable without tying it to one storage implementation.

```python
from dataclasses import dataclass
from typing import Literal, Protocol

Region = Literal["us", "eu"]


@dataclass(frozen=True)
class ExportRecord:
    export_id: str
    tenant_id: str
    region: Region
    object_key: str
    state: Literal["ready", "pending", "expired"]


class ExportRepository(Protocol):
    def get_for_tenant(self, export_id: str, tenant_id: str) -> ExportRecord | None: ...


class ReadCapabilitySigner(Protocol):
    def sign_read(self, object_key: str, ttl_seconds: int) -> str: ...


def issue_download(
    export_id: str,
    tenant_id: str,
    exports: ExportRepository,
    signers: dict[Region, ReadCapabilitySigner],
) -> dict[str, object]:
    record = exports.get_for_tenant(export_id, tenant_id)
    headers = {"Cache-Control": "private, no-store"}
    if record is None or record.state != "ready":
        return {"status": 404, "headers": headers}

    url = signers[record.region].sign_read(record.object_key, ttl_seconds=120)
    return {"status": 200, "headers": headers, "json": {"url": url}}
```

The `404` is intentionally shared by an absent record, a foreign record, and a non-deliverable record. A distinct public response for each condition turns the route into an existence oracle. Keep the internal reason, request identifier, tenant identifier, and selected region in protected logs; leave signed query parameters out of those logs because the URL itself is a temporary credential.

## The constraint is placement, not a regional switch

US and EU labels mean little unless they bind to a concrete data model. Store an immutable residency or placement value when the export is created, and use the same value for object creation, object lookup, signing-client selection, retention policy, and audit events. The database record is the source of truth for this mapping. A deployment environment may contain one configured client per region, but runtime routing should derive from the record, never from a browser hint.

This design is deliberately less flexible than a query parameter. Good. Flexibility at this boundary can create a duplicate object in a second location, sign a capability against a mismatched endpoint, or produce an audit trail that describes the user request instead of the data's actual location. For a migration, create new records with an explicit placement first, backfill only after verifying the destination object and its retention policy, and switch reads after that verification. Copying bytes alone does not establish the new authority record.

There is a timing boundary as well. The worker should finish writing the object before it publishes the export as ready; an authorized route then has a meaningful state to evaluate. If a storage system's read-after-write behavior, retention semantics, or signature-expiry behavior is unclear, confirm it against that system's documentation and an integration test. I'm not sure a single 120-second lifetime suits every mobile network or PDF size. Measure issuance-to-first-request latency and choose the smallest configured lifetime that meets the service's real delivery requirement.

## Failure modes worth designing and testing first

The happy-path signer call hides most of the consequential failure modes. An export job may be delivered twice; a lifecycle policy may remove bytes while metadata still says ready; an edge cache may retain a route response longer than intended; or a deployment may map the EU signer to the wrong regional configuration. Consider duplicate job delivery: if each attempt chooses a fresh key, both object writes can succeed and the database can still end with a single ready row, leaving an unreferenced PDF behind. If the attempts instead share an export identifier but publication occurs before the selected write is known complete, the route may sign a record whose intended object is not yet the durable one. The useful invariant is not "the worker ran"; it is that one tenant-scoped export record names one authoritative object, and that the ready transition follows the completed write for that object. That invariant gives retry behavior, cleanup, audit work, and route authorization the same thing to reason about. None of these failures is repaired by making the URL lifetime longer. They need invariants, telemetry, and tests that exercise the boundary between the database and the object store.

| Failure mode | What to assert | Design response |
| --- | --- | --- |
| Duplicate export work | One logical export has one authoritative record and key | Make creation idempotent on the export identifier |
| Metadata published too early | A ready record has a completed object write | Publish the ready state after the write completes |
| Caller-controlled placement | Changing a request field never changes storage selection | Resolve placement only from tenant-scoped metadata |
| Cached capability response | A second session cannot receive an earlier route response | Send `private, no-store` and test through the deployed cache path |
| Lifecycle drift | Retained metadata refers to retained bytes | Reconcile retention rules and export state on a schedule |

Test with two tenants using the same displayed filename, with records in both regions, and with pending, expired, and deleted-export metadata states. Assert that rejected responses omit the object key and that no accepted path signs a key belonging to another tenant. Record authorization denials, successful issuances, signer latency, selected region, and reconciliation discrepancies as distinct measurements. Collapsing them into one success rate makes a routing defect look like an ordinary download fluctuation.

## Choosing a delivery boundary and rolling it out

Signed URLs are useful when a client may receive a short delegated read capability for an immutable object. Application-mediated streaming is the better boundary when every byte needs application-layer inspection or a policy change must stop an already active transfer. The catch is that streaming makes the application responsible for bandwidth, long-lived connections, and concurrency during downloads. Separate US and EU locations are appropriate when placement must be explicit and auditable; a single location is simpler to operate when the governing data policy permits it. Neither choice can be made from a storage-rate headline alone, because request, retrieval, egress, retention, and operational work all affect the outcome.

Roll this out in a narrow sequence: add immutable placement and opaque object-key fields to new exports, place the signer behind the existing authorization check, test cache behavior at the real edge, and enable the path for a controlled cohort. Compare ready records with durable objects before migrating older exports. Keep a reversible application-level routing decision until those records reconcile; do not make a regional URL shape the application contract.

The final test is plain: an authorized caller receives a temporary capability for the object recorded in its assigned location, while every other caller learns nothing useful about that export. The route decides access. Placement decides where the bytes are found.

## References

- https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Cache-Control
- https://www.backblaze.com/cloud-storage/pricing
