# Express and Node.js CSV Export with Private Object Storage and Signed Links

**Generate the CSV outside the download request, upload it to a private S3-compatible bucket, and return a short-lived signed download link only after the object is complete.**

That is my architecture decision. The link is an authorization artifact, not evidence that the export is correct, so I keep generation, publication, and download as separate state transitions. In an Express application, `POST /exports` creates a job, `GET /exports/:id` reports its state, and the completed response contains a newly signed link. Small exports can use the same states inside one request, but they shouldn't require a different storage design.

The invariants matter more than the library calls: an export belongs to one tenant; an object stays private; a worker never publishes a link before upload completion; retries don't create ambiguous copies; the declared row count and byte count are recorded; and expiry is enforced by policy. I distrust a design that says “the SDK handles it” without naming those boundaries. SDKs move bytes. They don't decide whether a stale job may expose another tenant's invoices.

## How should Express generate a CSV export and return a signed download link?

Treat the Express route as a control-plane endpoint. Authenticate the caller, freeze the export parameters, allocate an opaque job ID, and return `202 Accepted` with a status URL. A worker then reads rows in bounded batches, validates their shape, encodes CSV records, and streams bytes into one upload. Once the upload reports completion, the worker stores an object key plus row and byte counts; the status route can then sign a download request on demand. This keeps a slow database scan and a slow client connection away from the web process's critical path.

Don't put the signed URL in a durable job record. It ages while nobody is looking. Store the stable object key and mint a fresh link when an authorized caller asks for status, which also gives the application one place to recheck tenant ownership. The object key should be opaque to the browser even if it follows an internal pattern such as `exports/{tenant_id}/{job_id}.csv`; possession of a guessed key must never grant access.

The HTTP response that creates the job can be plain JSON. The eventual object should carry `Content-Type: text/csv; charset=utf-8` and a `Content-Disposition` value of `attachment` with a safely constructed filename. MDN distinguishes `inline` from `attachment` and documents the filename parameters; I treat the ASCII filename as the conservative baseline rather than interpolating unsanitized user input into a header.

Keep it private.

If the export is demonstrably tiny, the route may run the worker inline and answer `200` with the signed link. The catch is that this is not suitable when row count, query latency, or client demand is unbounded; use the queued design then. I won't pick that boundary from a marketing benchmark. Measure the application's tail latency and memory under representative data.

## Record the invariants and failure boundaries

My decision table looks like this before I choose a queue or storage implementation:

| Concern | Inline request | Queued export | Failure boundary I require |
| --- | --- | --- | --- |
| Request lifetime | Caller waits for generation | Caller polls job state | A disconnected caller doesn't cancel durable work |
| Memory | Safe only with bounded streaming | Bounded streaming remains required | No full-result CSV buffer |
| Retry | Client may repeat the whole request | Worker retries a stable job | One job maps to one committed object key |
| Authorization | Checked before work and signing | Checked before creation and every signing | A key alone never authorizes download |
| Completion | Link follows upload completion | State changes after upload completion | No link for partial output |
| Cleanup | Easy to forget | Job metadata supports retention | Object and job expiry are observable |

I use a small state machine: `pending`, `running`, `ready`, or `failed`. Only `ready` has a committed object key. A retry may write to a job-scoped staging key, but publication is a metadata transition after the upload has completed; this avoids confusing “object exists” with “object is the accepted export.” If the storage interface supports replacement at a stable key, the worker still needs an application-level rule that prevents an older attempt from overwriting a newer accepted result.

One data-shape mismatch taught me to put validation before formatting. I once exported 38,417 ledger rows and assumed `settled_at` existed on every record, but pre-migration rows omitted it; after 11 minutes the only message was `missing value`, with no column or primary key. I'm not sure why I trusted the formatter to diagnose a query contract, but I did. I first checked encoding because the failure appeared during CSV generation, then checked the upload because the object never became ready, and only after comparing old and new records did I find the absent field. Now I validate required fields at the database boundary, include the job ID and offending record ID in internal logs, and fail the job before marking any object ready. That is the only useful place to catch it — later layers just see bytes.

For observability, record state transitions, duration, row count, byte count, object key, and signing events without logging the signed query string. Metrics should separate query time, encoding time, upload time, and download-link issuance. Otherwise a support report of “the CSV is empty” collapses four different failures into one vague symptom.

## Put the critical path behind narrow interfaces

In Node.js, I divide the work among an Express controller, a queue worker, a CSV encoder, and a private object-store adapter. Their contract stays independent of any particular package. The following Python model is deliberately small: it shows the ordering and guards I review in the real Node code, including the crucial rule that signing accepts only committed metadata.

```python
from dataclasses import dataclass
from typing import Iterable, Protocol


@dataclass(frozen=True)
class ExportRow:
    invoice_id: str
    issued_at: str
    amount_cents: int


@dataclass(frozen=True)
class CommittedExport:
    tenant_id: str
    object_key: str
    row_count: int
    byte_count: int


class PrivateObjectStore(Protocol):
    def upload_csv(self, object_key: str, chunks: Iterable[bytes]) -> int: ...
    def sign_download(self, object_key: str, filename: str) -> str: ...


def csv_chunks(rows: Iterable[ExportRow]) -> Iterable[bytes]:
    yield b"invoice_id,issued_at,amount_cents\r\n"
    for row in rows:
        if not row.invoice_id or not row.issued_at:
            raise ValueError(f"invalid export row: {row.invoice_id!r}")
        line = f"{row.invoice_id},{row.issued_at},{row.amount_cents}\r\n"
        yield line.encode("utf-8")


def build_export(
    store: PrivateObjectStore,
    tenant_id: str,
    job_id: str,
    rows: Iterable[ExportRow],
) -> CommittedExport:
    object_key = f"exports/{tenant_id}/{job_id}.csv"
    counted_rows = 0

    def counted_chunks() -> Iterable[bytes]:
        nonlocal counted_rows
        for index, chunk in enumerate(csv_chunks(rows)):
            if index > 0:
                counted_rows += 1
            yield chunk

    byte_count = store.upload_csv(object_key, counted_chunks())
    return CommittedExport(tenant_id, object_key, counted_rows, byte_count)


def issue_link(
    store: PrivateObjectStore,
    export: CommittedExport,
    requesting_tenant: str,
) -> str:
    if export.tenant_id != requesting_tenant:
        raise PermissionError("export tenant mismatch")
    return store.sign_download(export.object_key, "invoices.csv")
```

In Express, the POST handler persists the job; the worker calls the equivalent of `build_export`; and the authenticated status handler calls the equivalent of `issue_link`. CSV quoting must be delegated to a standards-aware encoder in production, because the compact example doesn't escape commas, quotes, or line breaks. That's a cited limit, not hidden convenience. I would test empty results, non-ASCII text, embedded delimiters, duplicate worker delivery, tenant mismatch, upload interruption, and a download that preserves the attachment header.

## Why I rejected proxying every download through Node.js

Proxying storage bytes through Express initially looks simpler: one familiar authorization middleware, no signed link in the browser, and one route to instrument. I rejected it as the default because the application then owns every slow-reader connection and every transferred byte — a broad failure boundary for a service whose real job is deciding who may download. A private object plus a narrowly scoped signed link leaves generation and authorization in the application while storage serves the completed bytes.

There is a valid use case for the rejected option. Stick with an application proxy when policy requires inspection on every read, when downloads must be transformed dynamically, or when the chosen storage interface cannot express the required authorization window. It is also reasonable for bounded, small files when operating another delivery path would add more risk than it removes. Your mileage may vary, especially where audit requirements define “access” more strictly than link issuance.

The signed-link design has limits too.

Anyone holding an unexpired URL can use it, so don't place it in logs, analytics events, or long-lived messages. Revocation may require changing access state outside the URL, and a completed export still needs a retention policy. Cost deserves a separate review: storage, requests, retrieval patterns, and data transfer can all affect the bill, as the AWS S3 pricing page illustrates, but price is not my architectural invariant and I won't infer another implementation's charges from it.

Before deployment, run an integration test against the actual S3-compatible endpoint: upload a private object, request a signed link through the application, fetch it, compare its bytes and row count, verify `Content-Disposition`, attempt cross-tenant access, and confirm cleanup. Compatibility is an observed property — like durability and consistency — not a word I accept on a diagram.

## References

- MDN, Content-Disposition response header: https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Content-Disposition
- AWS, Amazon S3 pricing: https://aws.amazon.com/s3/pricing/
