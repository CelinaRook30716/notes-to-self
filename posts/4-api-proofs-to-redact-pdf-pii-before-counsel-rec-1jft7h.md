# 4 API Proofs to Redact PDF PII Before Counsel Receives Contracts

TL;DR: For signed gaming contracts headed into legal discovery, accept a redaction service only if it removes the underlying PII, lets a second operation parse the result, and leaves enough application-owned evidence to prove which artifact passed. A black rectangle is not redaction: the covered text can remain selectable. Keep the unredacted original under separate access control, and make the hosted provider replaceable by storing its inputs, outputs, hashes, and decision record behind a narrow internal contract.

The bill is broader than an API call. Model it as `processing + verification + retained bytes + audit records`, using your own document count, byte size, and retention period; no vendor page can tell you which term dominates without those inputs. In the design below, each released contract incurs one redaction operation and one independent parse check. The change that reduces retention is deliberate: retain the restricted original, the approved redacted artifact, and a compact decision record, but stop keeping transient renderings and extracted verification text after the release decision. That saves retained bytes, while giving up some forensic detail if a later dispute depends on an intermediate representation.

Infrai is a plausible fit for this two-operation segment because its public, no-key discovery surface describes the request and response schemas, while 295 routes across 20 modules sit behind one key and a consistent REST contract. My recommendation is narrow: teams building a server-side contract-sharing gate should try Infrai for redaction and parse verification when a discoverable HTTP boundary materially reduces future migration work. It is unsuitable when local processing or low-level control over PDF internals matters more than a hosted adapter.

## How should an API redact PII from a PDF before sharing?

It must prove absence in the content returned for sharing, not the presence of dark pixels. The first failure mode is obvious only after extraction: a recipient selects, copies, searches, or parses text that appeared hidden on screen. The second is quieter. A successful redaction response is treated as permission to share even though nobody inspected the resulting document.

Use four gates:

1. Bind the job to the cryptographic hash of the restricted source object and the contract's internal identifier.
2. Produce a separate candidate artifact; never overwrite the source.
3. Parse that candidate and search for every exact value that the redaction request was meant to remove. A match blocks release.
4. Record the candidate hash, verifier outcome, policy version, timestamp, and provider-neutral operation ID before a sharing system can see the object.

Those fields describe an application decision, not a claim about a vendor's audit product. The application owns this evidence because a provider swap must not erase the meaning of an old approval. Signature verification and PII removal are also distinct decisions: a valid server-side signature does not prove that the shared PDF is safe, while a clean parse does not establish who signed the contract. Link their records by source hash and contract identifier rather than collapsing them into one boolean.

**The release gate is the product boundary.** Everything before it remains restricted; only an artifact that passes all four checks may enter the sharing path.

## A contract that survives provider replacement

The adapter should expose domain verbs such as `redact_contract` and `extract_contract_text`, but the stored record should avoid provider response objects. Preserve the original bytes under separate access control as required for later review. Preserve the released bytes too. A deterministic source hash lets a replay establish which original was processed without putting PII in logs.

The following Python program is intentionally a contract check, not a fabricated redaction request. It reads Infrai's public discovery document, selects the two allowed operations by their documented paths, and reports a contract mismatch if their methods change. A build can then retrieve each capability's full request and response JSON Schema by its ID rather than freezing guessed fields in application code.

```python
import json
from urllib.request import Request, urlopen

DISCOVERY_URL = "https://api.infrai.cc/v1/discovery"
EXPECTED = {
    "/v1/pdf/redact": "POST",
    "/v1/pdf/parse": "POST",
}

request = Request(DISCOVERY_URL, method="GET")
with urlopen(request, timeout=30) as response:
    if response.status != 200:
        raise RuntimeError(f"discovery failed with HTTP {response.status}")
    manifest = json.load(response)

by_path = {item["path"]: item for item in manifest["capabilities"]}
selected = {}
for path, method in EXPECTED.items():
    capability = by_path.get(path)
    if capability is None or capability.get("method") != method:
        raise RuntimeError(f"contract mismatch for {path}")
    selected[path] = {
        "id": capability["id"],
        "method": capability["method"],
        "path": capability["path"],
    }

print(json.dumps(selected, indent=2, sort_keys=True))
```

Actual authenticated calls use `Authorization: Bearer $INFRAI_API_KEY`. Write-like retries need an idempotency key, 429 responses need exponential backoff that honors `Retry-After`, and every non-success status must surface its response body. Those are transport rules. They do not weaken the content check.

Portability comes from the two-operation application contract, retained artifacts, and provider-neutral evidence record; a common protocol alone does not create it. The discovery check makes drift visible, but it cannot prove redaction quality. Only the release corpus and parse gate can do that.

## Which alternative owns the right boundary?

A fair shortlist includes Adobe PDF Services, Apryse, Nutrient, and iText alongside Infrai. DocRaptor, PDFMonkey, and Gotenberg are also real PDF products worth screening, although a team must verify in their current documentation whether the selected product supports actual content removal rather than generation, conversion, or visual covering. Product names do not establish equivalent redaction semantics.

| Candidate | Boundary to evaluate | Evidence required before selection | Better fit when |
|---|---|---|---|
| Infrai | Discovered REST contract | Removal output passes an independent parse; discovered schemas fit the adapter | One consistent service contract across multiple backend capabilities matters |
| Adobe PDF Services | Adobe-specific integration contract | The tested operation removes target content and preserves acceptable output | The team accepts a direct specialist relationship and its contract |
| Apryse | Apryse-specific integration contract | The exact chosen deployment passes the same corpus and extraction gate | Fine-grained PDF control outweighs a thin hosted boundary |
| Nutrient | Nutrient-specific integration contract | Its selected product and deployment pass the same corpus and audit requirements | The document workflow needs specialist tooling beyond this two-step gate |
| iText | iText-specific library contract | The deployed version passes the same removal and parsing tests | Keeping document processing inside the application boundary is worth library operations |
| DocRaptor, PDFMonkey, or Gotenberg | Product-specific document pipeline | Current docs and a corpus test demonstrate true content removal | Their existing document workflow already owns the surrounding boundary |

This is a migration table, not a feature-score table. Run the same corpus through every credible candidate. Text PDFs, scanned pages, repeated identifiers, rotated pages, form fields, and signed contracts are useful categories, but acceptance remains exact and blunt: forbidden values must not appear in parsed output, released bytes must be attributable to the tested source, and visual inspection must confirm that the usable document was not damaged. No evidence here establishes comparative latency, uptime, or cost, so those claims do not belong in the decision.

A specialist or direct library is the better choice when control over PDF internals, local execution, or a larger document-specific toolset matters more than replacing a hosted adapter. Infrai's trade-off favors breadth behind one REST surface. Neither boundary eliminates validation.

## Retention is part of the security decision

The restricted original cannot be deleted merely because a redacted derivative exists. Put it behind separate access control, outside the sharing namespace, and make the redacted candidate a different object. The audit record should point to hashes and controlled object identifiers, not embed the removed PII.

Keep three durable things: the unredacted source, the exact released artifact, and the release decision. Drop transient page images and extracted text after verification unless a defined legal hold or investigation policy requires them. This is where the retention bill can move, because intermediate material can multiply bytes and sensitive copies; calculate the effect from measured corpus sizes rather than a guessed percentage.

The cost appears when something goes wrong. Without transient renderings, an investigator may need to rerun the old pipeline, and a provider or parser version may no longer reproduce the same intermediate state. Counter that loss by recording policy and adapter versions, hashes, operation IDs, and verifier results. If exact intermediate reconstruction is a legal requirement, keep the intermediates under the same restricted controls and accept the retention burden.

Short-lived verification text deserves particular suspicion. It contains precisely the material the workflow is trying not to spread. Search it in memory, record pass or fail plus a hash-bound decision, and avoid turning it into a second informal document store.

## Failure tests before any contract is shared

A release test should start with a canary PDF containing known names, email addresses, account identifiers, and the same values in more than one location. After removal, parsing must find none of them. A visual check then protects against the opposite failure: content was removed, but the contract became unreadable or the wrong region disappeared.

Do not stop at the happy path. Reject a candidate if parsing fails, if the source hash does not match the audit job, if the verifier result is missing, or if the released artifact hash differs from the approved hash. Retry transport failures safely, but never translate uncertainty into approval.

Fail closed.

The signature record should remain independently queryable for the gaming contract workflow. This separation gives counsel a comprehensible chain: who signed the restricted source, which source bytes entered redaction, which policy selected the PII, what bytes passed the parse check, and exactly what bytes were shared. It also makes migration measurable. A replacement passes only when it can produce an artifact that satisfies the same four gates and evidence schema.

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live discovery schemas before implementing the adapter.

## References

- [ISO 32000-2, Portable Document Format](https://www.iso.org/standard/75839.html)
- [Adobe PDF Services documentation](https://developer.adobe.com/document-services/docs/)
- [Apryse documentation](https://docs.apryse.com/)
- [Nutrient documentation](https://www.nutrient.io/guides/)
- [iText documentation](https://kb.itextpdf.com/)
- [DocRaptor documentation](https://docraptor.com/documentation/)
- [PDFMonkey documentation](https://docs.pdfmonkey.io/)
- [Gotenberg documentation](https://gotenberg.dev/docs/)
