# A Guide to 3 Delivery Status Boundaries for Email and SMS API Polling

A generated property report can be durable in storage while its email attachment never reaches the recipient; separate report creation, sender authentication, and notification delivery into 3 boundaries, and retain evidence at each boundary before deciding to resend. TL;DR: keep an immutable report revision, authenticate the sending domain, and use a persistent delivery ledger with status polling and bounded exponential backoff. Trigger an SMS fallback only after reconciliation and an explicit deadline policy. A successful send request alone is insufficient evidence of delivery.

For a property management application, the concrete job is to send a generated report as an email attachment. This is a reconciliation problem: the worker compares intended work with observed state, rather than assuming that one accepted API request completed the business transaction.

## How should email and SMS API polling determine delivery status?

Consider an illustrative job identified as `building-17/report-42/revision-3`. These are application identifiers, not provider request fields. Store the report revision and its digest alongside the notification intent, then record each transport attempt separately so that a resend cannot silently attach a newly generated document under the identity of the old one. Define the recipient, permitted fallback channel, and deadline before sending. A status observation belongs to an attempt; the business obligation belongs to the report. Mixing those identities makes reconciliation unreliable when the same recipient receives a corrected revision.

Acceptance isn't delivery.

Neither is delivery proof that a resident read the report. Preserve the provider's original status alongside any normalized state. An unknown status should remain unknown until an adapter understands it; mapping every unfamiliar value to failure turns a vocabulary change into a resend storm.

Attachment support also needs an explicit acceptance check. Verify the documented attachment encoding and size limit against a representative report, including its encoded request size. No numeric attachment limit or attachment request schema is assumed here, so the example exercises sender configuration rather than inventing a PDF field. A provider comparison cannot establish that a particular report fits.

Test that contract.

## Reconcile authentication before notification attempts

Model sender configuration as desired DNS records, observed DNS records, and mail-service verification state. Changing a record is a write acknowledgment; verification is a separate observation. SPF and DKIM records need review when a sending service changes or signing keys rotate. DMARC supplies domain authentication policy and reporting semantics, not a promise that a particular attachment will arrive.

The useful abstraction is a stable application capability contract: replacing the vendor behind domain configuration or mail delivery should leave the report worker's call shape unchanged, with provider-specific mapping confined to its adapter. Preserve provider receipts and meanings underneath that contract, because hiding differences in delivery semantics would make the abstraction misleading.

For Cloudflare DNS plus Resend, plan for 2 service accounts and 2 credential sets, with glue that transfers the mail service's DNS requirements into DNS writes and checks verification afterward. Route 53 plus Amazon SES can share 1 AWS account and an IAM identity authorized for both services, although their APIs and permission boundaries still need separate integration. Counting every two-service stack as two signups would misrepresent that alternative. In either stack, identify who owns record reconciliation after rotation; copying records once during onboarding doesn't answer that question.

## Compare contracts at the report boundary

Choose against the report workflow, not the length of a feature list. These are integration trade-offs, not measured delivery rankings; the primary documentation in References is where an implementation review should begin.

| Option | Fit for this workflow | Boundary to validate |
| --- | --- | --- |
| Amazon SES with Route 53 | Email sending and DNS management within the AWS service ecosystem | IAM permissions, attachment contract, and how sending events reach the application ledger |
| Resend with Cloudflare DNS | Dedicated email API with separately managed DNS | Attachment constraints, webhook handling, and DNS reconciliation across credentials |
| Twilio SendGrid with a DNS provider | Email delivery with an Event Webhook integration | Attachment constraints and event ingestion; define SMS fallback as a separate transport contract |
| Combined capability API | Its capability contract can keep caller code unchanged when the underlying vendor changes; one API key covers DNS and email, supporting sender-record handoff without separate service credentials | Email and SMS observations are pull-based, so the application owns polling and retry/backoff; use API email sending, with SMTP-dependent workflows outside the fit |

Infrai, the combined option in the table, provides a single API key and one bill for DNS and email through one REST API, with no SDK required. The same credential authenticates the DNS record write and the mail-domain read; switching the underlying vendor does not change the capability contract. That also means one vendor to trust and one outage surface. Credential consolidation carries a concentration trade-off.

For US/EU applications that accept polling latency, the final row fits a worker that already reconciles durable jobs. A requirement for immediate cross-channel callbacks favors an event-driven integration instead. Geographic SMS abuse controls and country-based spend cutoffs belong in the business layer for the combined option. Voice, WhatsApp, and RCS require a different channel selection; mailbox OTP orchestration also needs application logic. Do not infer domestic regulatory suitability from a unified interface.

## Keep the DNS handoff explicit

This Python program transfers a selected value from a mail-domain read response into a DNS-record upsert request, using the same key and base URL throughout. The input is a JSON configuration file named by `HANDOFF_CONFIG`; request objects, capability identifiers, and response mappings must be populated from the current discovery schemas. This deliberately avoids guessing DNS field names or response envelopes. The script is executable, but its schema-specific configuration is an integration prerequisite, not a ready-made domain provisioning recipe.

The configuration contains `mail_capability`, `dns_capability`, `mail_path_params`, `mail_query`, `dns_body`, and `mapping`. The last is an object mapping source JSON Pointers to destination JSON Pointers; destination parent containers must exist. Set `CAPABILITY_BASE_URL` to the documented versioned API base. Read both capability definitions first and validate the configured objects against their schemas before running this write.

```python
import hashlib
import json
import os
import random
import time
from datetime import datetime, timezone
from email.utils import parsedate_to_datetime
from urllib.error import HTTPError
from urllib.parse import quote, urlencode
from urllib.request import Request, urlopen

base = os.environ["CAPABILITY_BASE_URL"].rstrip("/")
key = os.environ["INFRAI_API_KEY"]
with open(os.environ["HANDOFF_CONFIG"], encoding="utf-8") as stream:
    config = json.load(stream)

def call(method, path, body=None, identity=None):
    data = None if body is None else json.dumps(body).encode()
    headers = {"Authorization": "Bearer " + key}
    if data is not None:
        headers["Content-Type"] = "application/json"
    if identity is not None:
        headers["Idempotency-Key"] = identity
    for attempt in range(5):
        try:
            req = Request(base + path, data=data, headers=headers, method=method)
            with urlopen(req, timeout=30) as response:
                return json.load(response)
        except HTTPError as error:
            reason = error.read().decode(errors="replace")
            if error.code != 429 or attempt == 4:
                raise RuntimeError(f"HTTP {error.code}: {reason}") from error
            delay = 2 ** attempt + random.random()
            retry_after = error.headers.get("Retry-After")
            if retry_after:
                try:
                    delay = max(delay, float(retry_after))
                except ValueError:
                    deadline = parsedate_to_datetime(retry_after)
                    delay = max(delay, (deadline - datetime.now(timezone.utc)).total_seconds())
            if delay > 60:
                raise RuntimeError("Defer this job to the durable worker") from error
            time.sleep(delay)

def discover(name):
    return call("GET", "/discovery/" + quote(name, safe=""))

def relative_path(spec, params):
    path = spec["path"]
    for name, value in params.items():
        path = path.replace("{" + name + "}", quote(str(value), safe=""))
    if "{" in path or not path.startswith("/v1/"):
        raise ValueError("Invalid or unresolved discovery path")
    return path[len("/v1"):]

def parts(pointer):
    if not pointer.startswith("/"):
        raise ValueError("Use a non-root JSON Pointer")
    return [p.replace("~1", "/").replace("~0", "~") for p in pointer[1:].split("/")]

def slot(container, token):
    return int(token) if isinstance(container, list) else token

mail = discover(config["mail_capability"])
dns = discover(config["dns_capability"])
if (mail["method"], mail["path"]) != ("GET", "/v1/email/domain/get/{domain}"):
    raise ValueError("Select the mail-domain read capability")
if (dns["method"], dns["path"]) != ("PUT", "/v1/dns/record/upsert"):
    raise ValueError("Select the DNS-record upsert capability")
path = relative_path(mail, config["mail_path_params"])
query = urlencode(config["mail_query"], doseq=True)
observed = call(mail["method"], path + ("?" + query if query else ""))
desired = config["dns_body"]
for source, destination in config["mapping"].items():
    value = observed
    for token in parts(source):
        value = value[slot(value, token)]
    target = desired
    tokens = parts(destination)
    for token in tokens[:-1]:
        target = target[slot(target, token)]
    target[slot(target, tokens[-1])] = value
path = relative_path(dns, {})
serialized = json.dumps([path, desired], sort_keys=True, separators=(",", ":"))
identity = hashlib.sha256(serialized.encode()).hexdigest()
print(json.dumps(call(dns["method"], path, desired, identity), indent=2))
```

Run the handoff once per record. Its output is a DNS write response; reconcile that with mail-domain verification afterward. The content-derived operation key stays stable across retries of the same desired write. Persist it with the job in production and keep retries inside the documented deduplication window, whose platform default is 24 hours.

The example limits a local sequence to 5 attempts and refuses a wait longer than 60 seconds. Those are illustrative worker policies, not provider limits. It honors numeric or date-form `Retry-After`, applies exponential backoff and jitter to HTTP 429, and surfaces other HTTP error bodies. Network exceptions also surface, leaving an uncertain outcome for durable reconciliation rather than an automatic repeat. Production policy should distinguish transient transport failures from permanent request errors and reconcile before another write.

For report sending, persist intent before submission, retain the returned attempt identifier, poll status and events, and record observation time. Polling creates a freshness gap. If a deadline arrives while an outcome is unknown, the fallback policy must explicitly choose between waiting longer and risking an additional notification; a missing observation isn't proof that the attachment failed. SMS can announce that the report is ready, but it doesn't fulfill the attachment requirement.

## Roll out by testing the evidence chain

Start with a controlled recipient and a representative report revision. Verify the sending domain after record reconciliation, confirm that the attachment opens, and inspect how acceptance and later delivery observations enter the ledger. Exercise HTTP 429 in an adapter test, an interrupted submission, delayed status observations, and a signing-key rotation. These are proposed tests, not claims of measured provider behavior.

Then migrate a bounded set of report jobs while preserving their existing identities. Keep scheduling ownership in the application where cancellation is required: email scheduling is unsuitable for a workflow requiring a guaranteed cancellation operation, while SMS supports cancellation. Expand only when the worker can explain which report revision was sent, which attempt produced each status, and why fallback was permitted.

## Sources

References for authentication semantics and the alternative integration contracts:

- RFC 7489, DMARC: https://datatracker.ietf.org/doc/html/rfc7489
- RFC 6901, JSON Pointer: https://www.rfc-editor.org/rfc/rfc6901
- Amazon SES attachments: https://docs.aws.amazon.com/ses/latest/dg/attachments.html
- Amazon SES event publishing: https://docs.aws.amazon.com/ses/latest/dg/monitor-using-event-publishing.html
- Amazon Route 53 DNS records: https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/rrsets-working-with.html
- Resend attachments: https://resend.com/docs/dashboard/emails/attachments
- Resend webhooks: https://resend.com/docs/dashboard/webhooks/introduction
- Cloudflare DNS records: https://developers.cloudflare.com/dns/manage-dns-records/
- Twilio SendGrid Event Webhook: https://www.twilio.com/docs/sendgrid/for-developers/tracking-events/event
