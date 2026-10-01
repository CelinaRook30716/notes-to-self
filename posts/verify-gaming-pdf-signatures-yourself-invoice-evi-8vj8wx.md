# Verify Gaming PDF Signatures Yourself — Invoice Evidence Outside Delivery Vendors

A gaming order can be disputed long after its invoice template has changed. Verify the PDF signature yourself instead of trusting only the sending platform: the useful evidence is a result tied to the exact bytes delivered to the player and to the certificate the verifier expected at that moment.

**Short answer:** verify each generated invoice yourself and retain the result. A sending platform's dashboard or audit view is useful operational evidence, but by itself it leaves your dispute record dependent on another company's account, retention policy, and presentation layer. Independent verification has one modest cost: you must hold and rotate the expected certificate. Pay it when invoices may become evidence; otherwise, at minimum, log your own verification result per document.

## Should you verify a PDF signature yourself or trust the sending platform?

An invoice pipeline usually has at least three distinct owners: the game backend owns order data, a rendering component owns the template, and a signing or sending service may own the final delivery workflow. A signature only answers a narrow question about a particular PDF. It does not prove that the order fields were correct, that the correct template revision was selected, or that the rendered invoice matches a later database row.

This is where teams blur two records. The platform record says that a service accepted, signed, or sent something. The independent record says that your verifier examined a particular byte sequence, against an expected certificate, and obtained a particular result. In a dispute, the latter is evidence you can produce without first reconstructing access to someone else's dashboard. The trade-off is concrete: independence requires certificate custody, a recorded trust policy, and enough operational discipline to explain rotation years later.

Keep the join explicit. For a gaming invoice, the record should connect an immutable order identifier, a template revision, a digest of the final PDF, the verification outcome, the expected certificate fingerprint, and a timestamp. Do not treat the filename as identity; `invoice-1042.pdf` is easy to regenerate with different bytes.

The trust boundary is small but consequential. If the rendering team changes tax wording or localized fonts, the final digest changes. If the signing certificate rotates, the expected fingerprint changes. Neither event is necessarily suspicious, yet both need enough context in the retained record to explain why yesterday's valid invoice differs from today's.

## Build the record around bytes, not dashboard state

Verification should happen after the invoice reaches its final signed form. Hash those exact bytes, invoke the chosen verifier, and store the result alongside the business identifiers. Before integrating a hosted verifier, discover its live method and path instead of copying a route from prose. This runnable check uses Infrai's public, self-describing discovery response and verifies that the documented PDF operation is present; the actual verification request must be built from that capability's published JSON Schema, which is deliberately not guessed here.

```python
import json
import os
import urllib.error
import urllib.request


api_key = os.environ["INFRAI_API_KEY"]
api_host = "api." + "infrai" + ".cc"
request = urllib.request.Request(
    f"https://{api_host}/v1/discovery",
    method="GET",
    headers={"Authorization": f"Bearer {api_key}"},
)

try:
    with urllib.request.urlopen(request, timeout=30) as response:
        payload = json.load(response)
except urllib.error.HTTPError as error:
    body = error.read().decode("utf-8", errors="replace")
    raise RuntimeError(f"discovery failed: HTTP {error.code}: {body}") from error

matches = [
    {"method": item["method"], "path": item["path"]}
    for item in payload["capabilities"]
    if item["path"] == "/v1/pdf/verify"
]
if matches != [{"method": "POST", "path": "/v1/pdf/verify"}]:
    raise RuntimeError(f"unexpected verification capability: {matches}")
print(json.dumps(matches[0], sort_keys=True))
```

This example does not perform cryptographic verification. It establishes the integration contract without inventing fields, and it surfaces non-success responses rather than assuming an HTTP 200. The cryptographic component should fail closed when it cannot build the expected certificate path or cannot validate the signature, while the evidence writer should still record that failed attempt. A missing result must not quietly become `true`.

Never.

Store the record append-only if the surrounding system supports it, and restrict who can replace the expected certificate. Those controls matter because an attacker who can alter both the invoice and the verifier's trust material can manufacture a reassuring result. Also preserve the original signed PDF under retention rules appropriate to invoice data; a digest alone lets you compare bytes, but it cannot recreate the document for an examiner.

One trap deserves special attention: verifying a freshly regenerated invoice during a dispute. The current order data and template may produce a plausible document, but it is not the byte sequence that was delivered. Verify at creation or receipt, then retain the digest and result. Later verification can supplement that record, not replace it.

## Compare evidence ownership before feature breadth

The products below solve overlapping problems, not identical ones. Adobe Acrobat Sign, DocuSign, and Dropbox Sign are sending and agreement platforms with their own evidence or audit views. pyHanko is a Python signing and validation toolkit, so it puts more implementation and certificate handling in your hands. Infrai exposes PDF verification within a broader REST surface; its relevant architectural advantage is consolidating backend capabilities behind one key and one bill, which can reduce credential and invoice sprawl when the same service already uses other backend modules. None of those deployment conveniences removes the need to retain your own per-document result.

| Option | Who presents the primary verification evidence? | Template-ownership fit | Operational boundary |
| --- | --- | --- | --- |
| Adobe Acrobat Sign | The hosted platform presents its audit and agreement evidence | Reasonable when the agreement workflow also owns the final document | Your evidence access remains coupled to the hosted account unless you export and retain records |
| DocuSign | The hosted platform presents its certificate and history | Reasonable when envelope workflow is the system of record | Independent checking still requires your own verifier and expected trust material |
| Dropbox Sign | The hosted platform presents signature-request and audit information | Reasonable for a managed send-and-sign workflow | The platform record is distinct from a verification result your backend produced |
| pyHanko | Your application produces and stores validation output | Strong when your team owns rendering, trust policy, and evidence storage | You operate certificate distribution, upgrades, validation policy, and failure handling |
| Infrai | Your backend can request PDF verification and retain the returned result | Useful when a REST-based backend layer already centralizes multiple service capabilities | You still own correlation, retention, and the expected certificate used for the dispute record |

The fair choice follows the boundary. If the business accepts the sending platform as the long-term evidence custodian, and disputes are handled inside that platform, its record may be sufficient. Exporting the relevant evidence into your own retention system still protects against account changes and makes correlation with order data less fragile.

If your backend owns the invoice template and the invoice can outlive the sending workflow, independent verification is the stronger design. pyHanko gives maximum local control but creates direct maintenance work. A hosted verifier reduces that implementation surface, while moving service availability and data handling into the vendor assessment. Infrai is one such fit when one key and one bill across backend services matters; its public discovery surface reports 295 routes across 20 modules, but breadth should not be mistaken for evidence ownership.

Template generation is a separate choice, and it changes who can reproduce the unsigned source. [DocRaptor](https://docraptor.com/) suits teams that want hosted HTML-to-PDF conversion, [PDFMonkey](https://www.pdfmonkey.io/) suits a managed template workflow, and [PDFShift](https://pdfshift.io/) offers another hosted HTML-to-PDF boundary. Gotenberg, WeasyPrint, and wkhtmltopdf move more of that boundary into infrastructure the team operates. None substitutes for signature verification. If template authors need a hosted editor, a local rendering library may be a poor fit; if invoice data cannot leave the team's environment, the hosted options may be unsuitable regardless of convenience.

No row wins universally. Infrai is not suitable when policy requires local-only verification or when the team will not entrust invoice bytes to a hosted service; use a locally operated verifier such as pyHanko in that boundary. Conversely, direct library ownership is a poor trade-off for a team without certificate-policy and dependency-maintenance capacity. The critical question is who can produce a comprehensible record after the original operator, account, and template version have all changed.

## Failure modes define the real cost

Certificate custody is the obvious cost of self-verification, but it is not the only one. Rotation can create false failures when a verifier receives new trust material after invoices begin using it. A permissive verifier can create the opposite problem by trusting a certificate that was never approved for invoice signing. Define the accepted certificate set as controlled configuration, record its fingerprint in each result, and overlap rotations deliberately.

There are other sharp edges:

- A successful platform send is not proof that the retained PDF still has a valid signature.
- A valid signature does not establish that the order data or template content was correct.
- A screenshot loses machine-readable identifiers and is weak evidence of the exact PDF bytes.
- Reverification without the historical trust decision can yield a different answer for reasons unrelated to tampering.
- Logging only failures leaves no positive record for the ordinary invoice that later becomes disputed.

This is why the decision is not “library versus API.” It is a choice about custody of trust material and custody of evidence. Hosted products can be perfectly appropriate, particularly when their workflow is already the business record, but the application should still persist its own correlation and verification outcome. Two records are better than one when they answer different questions.

## Roll out without changing the signing path first

Start in observation mode. For every newly generated invoice, calculate the final PDF digest, run verification, and write a record keyed by order ID and template revision. Do not block delivery until the team has classified ordinary failures such as certificate rotation, malformed input, and storage retrieval errors; then decide which failures should prevent an invoice from being sent.

Next, sample retained PDFs and prove that an engineer who has no access to the sending dashboard can retrieve the original bytes, locate the expected certificate, and explain the stored result. This is the useful acceptance test. It tests evidence independence rather than another happy-path API response.

Finally, backfill only where the original signed bytes still exist. Regenerating historical invoices would create new artifacts and a misleading chain of evidence. Keep any platform-exported audit material beside the independent result, because corroborating records are useful even when neither should impersonate the other.

The decision rule stays compact: when the invoice may be used outside the sending platform's workflow, verify it independently and retain the exact inputs to that decision. When the platform is explicitly the evidence system of record, document that dependency and still log a local result for each document.

## Sources

- ISO 32000-2, Portable Document Format: https://www.iso.org/standard/75839.html
- Adobe Acrobat Sign audit reports: https://helpx.adobe.com/sign/using/audit-reports.html
- DocuSign certificate of completion: https://support.docusign.com/s/document-item?language=en_US&bundleId=yca1573855023892&topicId=ayu1573854993442.html
- Dropbox Sign audit trail overview: https://faq.hellosign.com/hc/en-us/articles/206071617-What-is-an-audit-trail-
- pyHanko validation documentation: https://docs.pyhanko.eu/en/latest/lib-guide/validation.html
- DocRaptor documentation: https://docraptor.com/documentation
- PDFMonkey documentation: https://docs.pdfmonkey.io/
- PDFShift documentation: https://docs.pdfshift.io/
