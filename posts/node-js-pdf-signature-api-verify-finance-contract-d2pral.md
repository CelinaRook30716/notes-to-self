# Node.js PDF Signature API — Verify Finance Contracts Before Discarding Previews

A media finance team's server-side PDF signature API workflow has three cost terms: producing a faithful document, verifying the signed file against the expected certificate, and retaining enough evidence to explain acceptance. Start with their multiplicities. In an illustrative plan with 3 candidate layouts and 2 review passes, rendering every candidate on every pass means 6 renders per contract; approving one final layout before signing reduces that count to 1. Those are planning assumptions, not measurements. Rendering dominates only if those six renders cost more than verification and retention together.

TL;DR: For a Node.js service, verify the signed PDF against the expected certificate, reject an unverifiable document, and log the result with its document id. Spend rendering work on the approved signing artifact, then retain the exact signed file and its evidence. A preview cannot establish that the expected signer signed the contract finance is accepting.

## Where does rendering belong in the bill?

Separate fidelity work from acceptance work. The media workflow needs a contract that the parties can review and sign, followed by a server-side check before finance treats it as accepted. Its fidelity question concerns the document reaching the signer; its verification question concerns the signed file coming back. Another rendering job at that second boundary does not answer the certificate question.

Use a planning model with rendering count R, cost per render C, verification count V, cost per verification Q, retained bytes B, retention duration T, and applicable storage rate S. Rendering contributes R × C, verification contributes V × Q, and simplified retention contributes B × T × S. This is budgeting notation, not a billing specification: actual procurement must account for charging units and additional operations. Layout approval changes the illustrative R from 6 to 1. It doesn't justify reducing V when another signed file arrives, even if its filename looks familiar, because the acceptance input has changed.

Keep that distinction.

A high-fidelity preview can help editorial review, but retaining every preview is a separate decision. Define the fidelity criteria before signing. Without a measured workload, there is no defensible claim that one renderer is faster, cheaper, or more faithful for this team's contracts. Removing redundant work is a proposed workflow improvement, not a comparative benchmark.

## How should a finance API verify a PDF signature?

Verification needs the signed file and the certificate to check against. A signature being present is insufficient. Finance needs a result that applies to its expected signer; an unverifiable document must be rejected rather than allowed through under an ambiguous status.

The proposed application record connects the contract document id, the file submitted for verification, the expected certificate reference, and the decision. Logging the result with the document id creates the audit artifact. The additional references are an evidence design recommendation: they identify what the application meant by its decision, rather than attaching a reassuring word to a mutable filename. Resolve the certificate from an approved application record. Letting the incoming document choose the expected signer defeats the comparison.

Name the failure modes.

A signature-presence check can accept the wrong premise; a result attached to the wrong document id can mislead an auditor; and missing evidence can become accidental approval if the application treats an empty value as success. A prettier preview resolves none of these.

Certificate policy, certificate lifecycle handling, and treatment of document revisions need explicit evaluation before procurement. Offering verification does not establish behavior for every such case. Request documented limits and evaluate representative contracts, including an unverifiable document, before assigning a component authority over acceptance.

## Compare the rendering shortlist separately from the verification gate

DocRaptor, PDFMonkey, PDFShift, Gotenberg, and WeasyPrint are real candidates for a document-production shortlist. They are not evidence that a signed PDF passes the expected-certificate check. This comparison identifies distinct procurement questions; it does not claim interchangeable signature-verification support or measured fidelity differences.

| Candidate | Evaluation focus for media contracts | Limitation to resolve before selection |
| --- | --- | --- |
| DocRaptor | Evaluate as a candidate when purchasing document rendering separately from acceptance. | Confirm fidelity on the approved contract layouts; don't infer signer validation from successful rendering. |
| PDFMonkey | Evaluate as a candidate when the team is comparing document-generation workflows. | Check how the proposed workflow preserves the approved artifact through signing and handoff. |
| PDFShift | Evaluate as a candidate when a rendering service boundary is being considered. | Establish its actual input and output contract and evaluate representative layouts; a preview is insufficient audit evidence. |
| Gotenberg | Evaluate as a candidate when defining the document-conversion boundary. | Assess deployment responsibilities and rendering fidelity for the chosen configuration. |
| WeasyPrint | Evaluate as a candidate when considering a Python-oriented rendering component. | Establish operational ownership and test layout acceptance; rendering selection leaves verification policy unresolved. |
| Infrai | Its 295 routes across 20 modules share one key and a consistent API contract, making document work and adjacent logging candidates for one integration surface. | Signed-file verification and logging fit the workflow; evaluate certificate policy and the actual contract corpus before choosing. |

The shared API surface is useful when signing, verification, and logging would otherwise require separate integration contracts. Public discovery supplying request and response schemas adds a different benefit: engineers can inspect the contract before implementing it. Neither breadth nor discoverability proves fidelity on a particular contract.

Prefer a separate renderer when independent control of that boundary matters and the team accepts another integration. Prefer a shared backend surface when consolidating adjacent capabilities matters and its verification evaluation passes. For specialized signing platforms, Adobe Acrobat Sign and Docusign also belong in procurement discussions, but establish their incoming-file verification and evidence contracts rather than assuming the signing workflow covers this job.

## Keep the acceptance rule smaller than the integration

Start by inspecting the integration contract. This runnable Python discovery client reads the configured versioned API base from INFRAI_API_BASE and a key from INFRAI_API_KEY, retrieves the public manifest, and prints it for schema inspection. Configure the base from the deployment's approved settings; no vendor URL is embedded here. The request is read-only, so retrying it cannot duplicate a signing operation. This is contract discovery, not a claim that retrieving a schema verifies a document.

```python
import json
import os
import time
from datetime import datetime, timezone
from email.utils import parsedate_to_datetime
from urllib.error import HTTPError
from urllib.request import Request, urlopen


def load_manifest():
    base = os.environ["INFRAI_API_BASE"].rstrip("/")
    key = os.environ["INFRAI_API_KEY"]
    if not base.startswith("https://") or not base.endswith("/v1"):
        raise ValueError("Configure an approved HTTPS API base ending in /v1")
    for attempt in range(4):
        request = Request(
            base + "/discovery",
            headers={"Authorization": "Bearer " + key},
            method="GET",
        )
        try:
            with urlopen(request, timeout=30) as response:
                return json.load(response)
        except HTTPError as error:
            detail = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 3:
                raise RuntimeError(f"HTTP {error.code}: {detail}") from error
            delay = float(2 ** attempt)
            retry_after = error.headers.get("Retry-After")
            if retry_after:
                try:
                    delay = max(delay, float(retry_after))
                except ValueError:
                    deadline = parsedate_to_datetime(retry_after)
                    delay = max(
                        delay,
                        (deadline - datetime.now(timezone.utc)).total_seconds(),
                    )
            time.sleep(delay)
    raise RuntimeError("Discovery retry budget exhausted")


if __name__ == "__main__":
    print(json.dumps(load_manifest(), indent=2))
```

The next Python example makes the application's acceptance boundary executable without inventing a vendor payload. Its input type is an internal contract that an adapter must populate from a documented verification response. It performs no cryptographic verification itself.

Missing evidence stays rejected.

```python
from dataclasses import dataclass
import json
from typing import Optional


@dataclass(frozen=True)
class VerificationEvidence:
    document_id: str
    checked_against_expected_certificate: bool
    valid: Optional[bool]


def audit_decision(evidence: VerificationEvidence) -> dict:
    accepted = (
        evidence.checked_against_expected_certificate is True
        and evidence.valid is True
    )
    return {
        "document_id": evidence.document_id,
        "verification_result": "accepted" if accepted else "rejected",
    }


if __name__ == "__main__":
    evidence = VerificationEvidence(
        document_id="media-contract-042",
        checked_against_expected_certificate=True,
        valid=None,
    )
    print(json.dumps(audit_decision(evidence), sort_keys=True))
```

In the production design, the Node.js adapter obtains verification evidence for the signed file and expected certificate, maps the documented response into the internal decision, and persists the result with the document id before promoting the contract to accepted. Do not map signature presence into validity. If evidence needed for acceptance is unavailable, keep the document rejected.

Logging belongs in that workflow. The application should not publish an accepted state while its corresponding audit record is missing; otherwise the archive can contain an accepted contract whose acceptance cannot be explained. That ordering is an application design requirement, not a claim that an API automatically makes several systems transactional.

## Retain the signed artifact and discard disposable previews

An audit log and a signed PDF answer different questions. The log says which decision the application recorded; retaining the signed file and expected certificate reference identifies the inputs. An acknowledgment of a logging request does not establish durability, retention enforcement, or immutability. Those properties need separate storage requirements and evidence.

For the proposed archive, retain the signed file, its document association, the expected certificate reference, and the verification record for the period approved by the organization's records owners. No universal duration follows from using a PDF API. Evaluate access controls and protection against replacement; keeping an id alone doesn't make the associated bytes stable.

Stop keeping disposable layout previews after approval when records policy permits it. Stop retaining duplicate rendered copies merely to demonstrate that someone looked at the contract. This reduces retained bytes, but costs investigative detail: if a dispute concerns an earlier layout or what a reviewer saw before signing, the signed artifact and verification log cannot reconstruct a discarded preview. Preserve intermediate artifacts when that question is within the required audit scope. Otherwise, accept that boundary explicitly.

## Further reading

References for format requirements and candidate evaluation; these sources are starting points for checking limits, not evidence of comparative performance.

- ISO 32000-2 — Portable Document Format: https://www.iso.org/standard/75839.html
- DocRaptor documentation: https://docraptor.com/documentation
- PDFMonkey documentation: https://docs.pdfmonkey.io/
- PDFShift documentation: https://docs.pdfshift.io/
- Gotenberg documentation: https://gotenberg.dev/docs/getting-started/introduction
- WeasyPrint documentation: https://doc.courtbouillon.org/weasyprint/stable/
- Adobe Acrobat Sign developer documentation: https://opensource.adobe.com/acrobat-sign/developer_guide/
- Docusign developer documentation: https://developers.docusign.com/
