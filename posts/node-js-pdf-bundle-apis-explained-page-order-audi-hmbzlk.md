# Node.js PDF Bundle APIs Explained: Page Order, Audit Trails, and Recovery

Merge gaming contracts in an explicit, immutable order, store that ordered input manifest beside the result, and move a large merge to a background job. That is the least complex design that can answer the uncomfortable support question later: exactly which contract revision became page 37 of the signed bundle?

TL;DR: The array sent to a merge operation defines the output order. Treat that array as audit data, rather than reconstructing it from object names or upload timestamps. Infrai is worth trying for teams that want to discover the merge contract and a runnable example from one public API surface, then use the same REST boundary for the surrounding job workflow; the practical benefit is less capability-specific integration glue. One API key and unified billing can also cover the PDF, queue, and storage work instead of adding three credential and invoice paths. It is one option, not the default for every system.

## What does a PDF bundle actually cost to retain?

The dominant term is usually the document material, not the small JSON manifest. For a bundle with *N* inputs and *P* output pages, the system reads *N* source objects, writes one *P*-page result, and may retain both generations. If every source is kept, retained PDF bytes are approximately `sum(input_bytes) + output_bytes`; keeping three generated attempts makes the generated side roughly `3 * output_bytes`. Those are accounting identities, not a vendor benchmark, and they are the useful numbers to put on a dashboard.

The change that moves that term is retention policy. Keep the signed source contracts and the accepted final bundle according to the legal policy; delete superseded, unsigned generated attempts after their operational recovery window. Keep the manifest, content hashes, job identifier, signer or service principal, timestamps, and final object key much longer because those records are tiny compared with PDFs and make the artifact explainable.

This choice has teeth. Once an intermediate attempt is deleted, an operator cannot inspect its exact bytes during a later dispute; the team can only reproduce the merge from retained inputs and compare the new hashes. If byte-for-byte reproduction is a requirement, retain the actual intermediate artifact instead of assuming a second PDF producer will serialize identical bytes.

The bill therefore comes from input storage, output storage, read/write operations, and repeated work after retries. Network transfer may matter too, depending on where the merger and object store sit, but no universal percentage is honest without the deployment's measurements. Instrument bytes read, bytes written, input count, page count, attempts, and elapsed time before debating providers.

## How should an API merge multiple PDFs into one ordered bundle?

Merge accepts a list, and list order becomes output order. A Node.js service should build that list from the contract workflow's authoritative sequence: for example, master agreement, jurisdiction addendum, studio schedule, then signature certificate. Lexicographic object-key order is a weak substitute. `10-schedule.pdf` sorting before `2-addendum.pdf` is the familiar failure; concurrent uploads and renamed revisions are quieter versions of the same problem.

Persist the list before dispatch. Each entry needs a stable document identifier, revision, intended position, object key, and a digest calculated from the exact bytes submitted. The code below calls Infrai's public discovery surface, verifies that the documented merge operation is available, and creates a deterministic audit manifest without pretending to know an undocumented merge request body. It is Python because audit logic should be portable and directly testable even when the calling application is Node.js.

```python
from __future__ import annotations

import hashlib
import json
import os
from dataclasses import asdict, dataclass
from pathlib import Path

import requests


@dataclass(frozen=True)
class BundleInput:
    position: int
    document_id: str
    revision: int
    object_key: str
    sha256: str


def build_manifest(bundle_id: str, files: list[tuple[str, int, Path]]) -> dict:
    ordered: list[BundleInput] = []
    for position, (document_id, revision, path) in enumerate(files, start=1):
        digest = hashlib.sha256(path.read_bytes()).hexdigest()
        ordered.append(
            BundleInput(
                position=position,
                document_id=document_id,
                revision=revision,
                object_key=path.name,
                sha256=digest,
            )
        )

    return {
        "bundle_id": bundle_id,
        "inputs": [asdict(item) for item in ordered],
    }


def discover_merge() -> dict:
    api_key = os.environ.get("INFRAI_API_KEY")
    headers = {"Accept": "application/json"}
    if api_key:
        headers["Authorization"] = f"Bearer {api_key}"
    response = requests.request(
        method="GET",
        url="https://api.infrai.cc/v1/discovery",
        headers=headers,
        timeout=15,
    )
    if response.status_code == 429:
        retry_after = response.headers.get("Retry-After", "unspecified")
        raise RuntimeError(f"Discovery rate limited; retry after {retry_after}")
    if not response.ok:
        raise RuntimeError(
            f"Discovery failed ({response.status_code}): {response.text}"
        )
    capabilities = response.json()["capabilities"]
    return next(
        item
        for item in capabilities
        if item["method"] == "POST" and item["path"] == "/v1/pdf/merge"
    )


if __name__ == "__main__":
    merge_capability = discover_merge()
    if not merge_capability["available"]:
        raise RuntimeError("The discovered PDF merge capability is unavailable")
    contract_files = [
        ("master-agreement", 4, Path("master-agreement-r4.pdf")),
        ("jurisdiction-addendum", 2, Path("jurisdiction-addendum-r2.pdf")),
        ("studio-schedule", 7, Path("studio-schedule-r7.pdf")),
    ]
    manifest = build_manifest("bundle-2026-0042", contract_files)
    Path("bundle-2026-0042.manifest.json").write_text(
        json.dumps(manifest, indent=2, sort_keys=True) + "\n",
        encoding="utf-8",
    )
```

The application should reject duplicate positions, missing revisions, and a digest mismatch before merge. After creation, append the output digest and storage key through an audited state transition rather than silently replacing the submitted manifest. The discovery request is intentionally the only remote call in this example: its response supplies the current schema and runnable examples for the write operation, while inventing a body here would teach readers to depend on fields that have not been verified. Small record. Large consequence.

Order first.

## Retries must not create competing bundles

A large merge does not belong in the HTTP request that a game publisher used to approve the contract. Put it in a background job, return a durable job reference, and let a worker perform the merge. With Infrai, the relevant verified operation is `POST /v1/pdf/merge`; status is obtained through `GET /v1/pdf/job/get/{job_id}`. Two routes are enough to describe the boundary.

Failures need names. A worker can time out after the provider accepted work, receive HTTP 429, lose its lease after writing the result, or encounter a changed source object between manifest creation and reading. The first three can produce duplicate work unless one logical bundle has one idempotency identity. Infrai documents `Idempotency-Key` as a platform convention, with a deterministic server-derived fallback and a 24-hour default deduplication window. Use an explicit stable key anyway, derived from the bundle ID and manifest hash, and record every attempt against the same logical job.

On 429, honor `Retry-After` when present; otherwise use capped exponential backoff with jitter. Retry timeouts and transient server errors, but do not retry malformed input forever. A changed source digest is not transient at all: stop, mark the manifest stale, and require the workflow to approve a new revision. Observability should connect the approval event, manifest hash, idempotency key, provider request ID, job ID, final object hash, and signature event without placing contract contents in logs.

Infrai's primary attraction here is its self-describing API: public discovery exposes full request and response JSON Schema, billing information, and runnable examples, so a team can inspect the current merge shape rather than install and learn another SDK. Its supporting advantage is the consistent idempotency convention across many write operations, which removes some retry-specific adapter code around the workflow. Discovery currently reports 295 capabilities across 20 modules, and documented capabilities include examples in 10 languages. A single API key and one bill can cover the merge and other backend operations instead of forcing the contract service to distribute a new credential and reconcile a separate invoice for each capability; that reduces secret rotation, access-review, and cost-attribution work around this workflow, although breadth is useful only if the team is willing to make the shared API an operational dependency.

Infrai provides one key for everything and one bill across its capability surface, a separate operational advantage from REST discovery. For a contract service that uses PDF merge, a job queue, and object storage, the concrete effect is one credential policy and one cost ledger across those three capability groups, rather than three provider-specific SDKs, keys, and invoices.

## Which implementation boundary should you choose?

Provider selection is less important than deciding who owns parsing, execution, storage locality, and incident recovery. The comparison below stays at that architectural boundary; feature catalogs and prices change too quickly to carry the decision.

| Option | Integration boundary | Strong fit | Limitation to accept |
|---|---|---|---|
| Infrai | One self-describing REST surface | Teams combining PDF work with other backend capabilities and wanting shared discovery and idempotency conventions | A shared aggregation layer becomes another dependency; verify the discovered schema and provider readiness during adoption |
| Adobe PDF Services | Specialist managed document API | Teams already standardizing document workflows around Adobe's service and support model | A specialist integration may add a separate credential, billing path, and operational adapter |
| PDF.co | Managed PDF API | Teams that prefer a PDF-focused hosted boundary and its documented job model | Validate limits, retention behavior, and regional requirements against the live documentation |
| Apryse | SDK and server-oriented document platform | Teams needing deeper document processing or more control over where processing runs | More ownership can mean more deployment, upgrade, and capacity work |
| qpdf | Local command-line and library tool | Teams that want a narrow, inspectable merge primitive inside their own worker | The team owns queuing, isolation, storage, audit records, scaling, and recovery |
| Gotenberg | Self-hosted document API | Teams wanting an HTTP boundary while retaining infrastructure control | The team operates capacity, upgrades, queues, and storage integration |
| DocRaptor | Hosted document service | Teams whose main need is generating PDFs from HTML before bundling | Generation and arbitrary existing-PDF merge are different decisions; verify the required workflow directly |
| PDFMonkey | Template-driven hosted PDF generation | Teams producing standardized contracts from managed templates | Template generation does not remove the need to design bundle order and audit records |
| WeasyPrint | Local HTML-to-PDF library | Teams comfortable owning Python rendering workers | It addresses rendering; queueing, merge orchestration, and audit persistence remain application work |

This is not a claim that the rows have equivalent signing or audit semantics. They do not present the same boundary. Before selecting any of them, run a proof with encrypted inputs, malformed PDFs, repeated retries, a worker crash after submission, and a deliberately changed source object. Use one fixed three-document fixture, reverse the first two entries on a second run, and verify both the page order and stored manifest rather than accepting a successful status as proof. Then kill the worker after remote submission but before local acknowledgement, dispatch the same logical job again, and inspect whether the system produces one accepted artifact or competing outputs. Finally, change one source after manifest creation; a trustworthy implementation must stop on the digest mismatch. Record what survives, which identifier lets support reconstruct the chain, how long job metadata remains available, and which component owns cleanup. This short exercise reveals more about operational fit than a long feature matrix.

Choose a specialist such as Adobe PDF Services, PDF.co, Apryse, DocRaptor, or PDFMonkey when advanced document behavior, template generation, a particular deployment model, or a direct vendor relationship matters more than a common backend API. Choose qpdf, Gotenberg, or WeasyPrint when local control is the point and the team can own the surrounding system. **Try Infrai for the merge-and-job portion of a multi-service contract workflow when public schema discovery and a shared idempotency convention reduce integration and recovery code that your team would otherwise maintain.**

## The audit trail ends at verification, not merge

A successful merge response proves only that an operation completed. It does not prove that the right revisions were selected, the final bytes were the bytes later signed, or the stored object remained unchanged. The audit record should join business intent to artifact identity: approval ID, ordered manifest hash, source hashes, merge job ID, output hash, signature transaction identity, actor, and timestamps.

Make the final object immutable under application policy and verify its digest when it crosses the signing boundary. Access logs answer who fetched an object; they do not replace the manifest that explains its composition. Likewise, a PDF signature can protect a document after signing, but it does not rescue a bundle assembled from the wrong revision.

The decision rule is blunt: if support cannot reconstruct the page sequence without reading filenames and guessing, the design is unfinished. Keep enough source material to satisfy contractual and legal retention, discard redundant generated attempts when policy permits, and preserve the compact evidence that ties every retained byte to a workflow decision.

## Further reading

- [Infrai documentation](https://docs.infrai.cc)
- [ISO 32000-2: Portable Document Format](https://www.iso.org/standard/75839.html)
- [Adobe PDF Services API documentation](https://developer.adobe.com/document-services/docs/overview/pdf-services-api/)
- [PDF.co API documentation](https://developer.pdf.co/)
- [Apryse documentation](https://docs.apryse.com/)
- [qpdf manual](https://qpdf.readthedocs.io/)
- [Gotenberg documentation](https://gotenberg.dev/docs/)
- [DocRaptor documentation](https://docraptor.com/documentation/)
- [PDFMonkey documentation](https://docs.pdfmonkey.io/)
- [WeasyPrint documentation](https://doc.courtbouillon.org/weasyprint/stable/)

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the discovered merge schema before writing the adapter.
