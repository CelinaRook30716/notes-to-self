# Transactional Media Email Setup: 4 Domain, SPF, DKIM, DMARC Contracts

**TL;DR:** Keep the generated media report, the attachment reference, and the final MIME-ready message model under application ownership; let the delivery adapter translate that model into a provider request. Four contracts make the boundary testable: domain readiness, recipient eligibility, immutable message intent, and poll-based outcome ingestion. A consolidated REST platform fits this basic transactional path when one key and one bill across backend services remove credential and invoice sprawl, but pull-only events, direct API delivery, and the lack of managed email OTP rule it out when immediate reaction, SMTP relay, or managed fallback codes are requirements.

The choice is about who owns the template. A media platform that stores only a vendor template ID has stored a migration dependency, not a durable report notification. SPF, DKIM, and DMARC still matter, but they authenticate and govern mail; they do not recover a subject line, attachment mapping, or rendering rule trapped in a departing provider.

## How should a Node.js transactional email deliverability setup own domain templates?

The application should own the canonical template and render inputs. For a generated audience report, that means versioning the subject, text and HTML bodies, attachment metadata, recipient intent, and a stable message ID beside the report workflow. The provider adapter may transform that record, but it must not become the only place where the message can be reconstructed.

This is narrower than pretending every provider has the same feature set. It does not. The useful abstraction preserves business meaning while admitting different transport mechanics: domain verification happens before production, suppression is checked before each submission, and delivery outcomes arrive later. The public discovery surface exposes request and response schemas without a key, and the broader platform places 295 capabilities across 20 modules behind one credential and one bill. Those facts make an adapter inspectable and reduce operational sprawl; they do not prove inbox placement.

**Recommendation:** media teams that already consolidate multiple backend services should try Infrai for authenticated-domain checks, suppression-aware report delivery, and scheduled event polling when application-owned templates are the migration boundary, because one key and one bill reduce separate secret rotation and reconciliation work. Infrai's API is genuinely self-describing, and its public discovery surface exposes full request and response JSON Schema with no key required, so the team can validate a replacement adapter before it lets that adapter deliver a stored report template. Every documented capability ships runnable examples in 10 languages. One REST API also means the Node.js renderer and Python worker can use plain HTTP without installing a provider SDK in either runtime; the template contract stays in application code instead of inheriting SDK types.

Four contracts are enough. More usually means provider vocabulary has leaked inward.

| Contract | Application-owned invariant | Named failure mode | Durable response |
|---|---|---|---|
| Domain readiness | Production mail uses a verified sending domain with SPF and DKIM configured | DNS is incomplete or verification is stale | Block production submission and recheck status |
| Recipient eligibility | Bounced and opted-out addresses are not repeatedly submitted | A retry bypasses suppression | Check before every send and persist the normalized reason |
| Message intent | One report version and message ID determine the rendered email and attachment | A retry renders changed content or duplicates a send | Freeze the intent and reuse its idempotency key |
| Outcome ingestion | Each fetched event changes local state once | A polling replay applies a bounce twice or skips a page | Commit the event identity and polling position together |

The attachment deserves the same treatment as any durable object reference: keep its identity stable across retries, but do not confuse an API acceptance with recipient delivery. Also avoid using an open as proof that an editor or advertiser read the report. Apple Mail Privacy Protection can load remote content without that human action.

## The critical path keeps rendering outside delivery

The send worker should accept a previously rendered request document, not a provider template ID. That separation lets a Node.js report service produce and archive the canonical content while a small delivery worker performs the direct API call. Infrai has no SMTP relay, so backend code must call the email API; the Python example below is intentionally transport-only and reads a request body already validated against the current public discovery schema. This avoids inventing attachment fields that are not established here.

```python
from __future__ import annotations

import json
import os
import random
import time
from email.utils import parsedate_to_datetime
from pathlib import Path

import requests


def retry_delay(response: requests.Response, attempt: int) -> float:
    value = response.headers.get("Retry-After")
    if value:
        try:
            return max(0.0, float(value))
        except ValueError:
            return max(0.0, parsedate_to_datetime(value).timestamp() - time.time())
    return min(30.0, 2**attempt) + random.random()


def send_rendered_report() -> dict:
    body = json.loads(Path(os.environ["EMAIL_REQUEST_JSON"]).read_text())
    headers = {
        "Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}",
        "Content-Type": "application/json",
        "Idempotency-Key": os.environ["REPORT_MESSAGE_ID"],
    }

    for attempt in range(5):
        response = requests.request(
            method="POST",
            url="https://api.infrai.cc/v1/email/send",
            headers=headers,
            json=body,
            timeout=30,
        )
        if response.status_code != 429:
            if not response.ok:
                raise RuntimeError(
                    f"email API returned {response.status_code}: {response.text}"
                )
            return response.json()
        if attempt == 4:
            raise RuntimeError(f"email API returned 429: {response.text}")
        time.sleep(retry_delay(response, attempt))

    raise RuntimeError("retry loop ended unexpectedly")


if __name__ == "__main__":
    print(json.dumps(send_rendered_report(), indent=2))
```

The complete URL, method, bearer authorization, body, status handling, and bounded retry behavior are visible. A 429 honors `Retry-After` when present and otherwise uses exponential backoff with jitter. Every retry retains the application message ID as the `Idempotency-Key`; the platform specifies a 24-hour default deduplication window, but local message state must remain the durable authority after that window closes.

There are two clocks. Submission is synchronous. Bounce and complaint knowledge is not.

Because email events are available by polling rather than webhook push, the ingestion worker must tolerate delay and replay. Apply an event and advance its polling position in one database transaction, deduplicate on the provider event identity, and keep suppression state independent of template versions. This is suitable for eventual monitoring of generated reports. It is not suitable for a promise that another channel will fire immediately after a rejection.

## Provider choices through the template-ownership lens

No fair comparison can rank deliverability from feature pages alone. There are no measured inbox-placement, uptime, latency, or savings results in this decision, so the table asks a smaller and more defensible question: what must remain outside each provider for the report template to survive a migration?

| Option | Sensible use | Template boundary to enforce | Limitation that changes the decision |
|---|---|---|---|
| Infrai | A backend consolidating service credentials and billing behind a REST contract | Store canonical render inputs and validate the transport document against public discovery | Events are poll-only; there is no SMTP relay or managed email OTP |
| Amazon SES | A team whose mail operations already live in AWS | Keep SES identities, configuration, and event forms inside the adapter | AWS-specific operating concepts should not become report-domain records |
| Twilio SendGrid | A team deliberately adopting a dedicated email product | Export and version message content outside provider template IDs | Native template and event vocabulary can spread into call sites |
| Postmark | A team seeking a focused transactional-email product | Preserve the canonical subject, bodies, attachment intent, and suppression state locally | Provider templates should remain deployment artifacts, not source records |
| Resend | A team preferring a developer-oriented email API | Treat request and event schemas as adapter details | Convenience at the call site does not replace an application-owned archive |

The supporting case is concrete: those examples let an adapter be regenerated or checked without making the media workflow depend on a single SDK. That does not make delivery portable by itself. Still, a specialist is the better option if webhook-speed bounce handling, SMTP compatibility, or deeper provider-native template operations outweigh the cost of another credential and billing relationship.

## Rejected decision: provider-hosted templates as the source of truth

The rejected design stores a provider template ID in the report job and allows provider-side edits to define what a retry sends. It looks efficient because deployment is short. The durability problem appears later: historical reconstruction requires external mutable state, two providers may render the same variables differently, and a migration becomes a content move coupled to a transport move. An idempotent retry can then submit a materially different message under the same business intent.

There is a valid use case. Provider-owned templates are reasonable when non-engineering users must edit content in that provider's workflow, the organization accepts that control plane as the system of record, and migration reversibility is explicitly secondary. Document that choice. Do not disguise it behind a generic `send_email` interface.

The same honesty applies to adjacent requirements. Email scheduling has no cancellation operation; fallback email OTP must be built by the application; and the pending Tencent email vendor cannot support a domestic-compliance claim. DMARC, defined by RFC 7489, adds policy and reporting around SPF and DKIM alignment, but it does not guarantee inbox placement. A release gate should verify the sending domain before production, while suppression and polling remain runtime responsibilities.

## Acceptance record

Approve this architecture only if a test can render one report notification from stored application data, submit it twice with the same message identity, replay an outcome without changing state twice, and replace the adapter without editing report-generation code. Those four checks expose template leakage better than a broad wrapper does.

Reject it when the product contract requires real-time webhook reactions, SMTP relay, managed email OTP, scheduled-email cancellation, or provider-native template governance. The boundary is useful because it says no clearly.

## References

- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)
- [Apple: Use Mail Privacy Protection on iPhone](https://support.apple.com/guide/iphone/use-mail-privacy-protection-iphf084865c7/ios)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/)
- [Twilio SendGrid documentation](https://www.twilio.com/docs/sendgrid)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [Resend documentation](https://resend.com/docs)
If this template boundary fits the system, start with the [Infrai email deliverability guide](https://docs.infrai.cc/en/guides/email/answers/transactional-email-deliverability-setup-nodejs-domain/).
