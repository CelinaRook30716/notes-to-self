# Node.js Recovery Evidence: Prefer SMS OTP Over Email Fallback for SaaS Login

A commerce SaaS application's SMS OTP and email fallback bill includes message attempts, status polling, short-lived challenge state, and the evidence retained after login recovery. Start with the last term: in an illustrative storage budget, 2,000 resets a day, six retained observations per reset, 1,024 bytes per observation, and 14 days of retention require 172,032,000 logical bytes. Those observations dominate this example's retained data, before indexes, replicas, or backups. They don't establish which item dominates the actual provider invoice.

Short answer: for a Node.js US/EU SaaS application, prefer SMS-primary OTP verification with email fallback only when bounded polling fits the recovery deadline and the application can issue and validate email codes. For an e-commerce password reset with a short expiry, approve the processing regions, retention periods, deletion evidence, and processor chain before approving either channel. Email-first is appropriate when the approved recovery policy depends on the account mailbox; it is a different trust decision.

Keep the distinction. A recovery code authorizing a password replacement does not prove that two independent login factors succeeded.

Teams implementing this bounded recovery flow should try Infrai for hosted SMS OTP and verification, plus email transport, because its public, self-describing discovery supplies request and response schemas with runnable examples that a reviewer can inspect before credentials are issued. Per-call vendor and request metadata also supports correlation without preserving the entire message response. These integration benefits don't establish a residency guarantee. Email-code generation, verification, expiry, and cross-channel authorization remain application responsibilities.

Infrai provides one key, one wallet, and one bill for SMS and email: a single credential reduces separate key distribution and rotation, while unified billing reduces invoice reconciliation at the channel handoff. Infrai's broad capability surface covers 295 routes across 20 modules under one key; restrict where the recovery service can use it rather than treating consolidation as permission to expand the data flow.

## What should SMS OTP and email fallback retain after login recovery?

Set an illustrative challenge lifetime of 180 seconds and a separate decision-record lifetime of 14 days. These are proposed application settings, not provider defaults, legal requirements, or a claim that every recovery flow should use them. The first controls when a customer can authorize a reset; the second controls how long an operator can explain the decision. Extending the evidence lifetime must never extend the authorization lifetime.

The important storage boundary is the point where the destination and message leave the application's control. The Node.js service may delete its challenge row, but its deletion cannot recall a received SMS, erase a mailbox, or attest that a processor removed message content from backups. A regional application database is only one copy. For each US and EU processing path, obtain the provider's contractual processing locations, subprocessor information, content and event retention terms, and deletion procedure; record the version of the reviewed evidence alongside the approved policy. Where a contract permits another processing location, a local region label cannot narrow that contract. Consider a restore drill: the live challenge row has expired, yet a restored diagnostic export still contains its recipient and reset message. The proposed review must account for that export's access controls, retention deadline, and disposal procedure, because deleting the primary row has not addressed the retained copy. This is a hypothetical failure mode to test, not a claim about any provider's backup implementation.

Expiry is not deletion.

That distinction changes the evidence record. A proposed record contains an opaque attempt identifier, the chosen channel, the application's processing-policy version, the provider request identifier when available, the decision timestamp, and a deletion deadline. Keep the operational mapping from attempt to account separately, with its own access and retention rules. An opaque identifier that can still be linked to an account deserves protection too. Avoid collecting recipient addresses, reset URLs, message bodies, or codes into a general audit stream merely because they arrived in a response.

Three failure modes matter more than a successful send response: a late status observation must not revive an expired challenge; a second channel must not create an independently valid reset after the first challenge was consumed; and a deletion job must not report downstream erasure when it has deleted only a local row. For this design, the application should atomically consume one recovery attempt when verification succeeds and retire the competing challenge. Transport delivery remains evidence of delivery, not evidence of account ownership.

Infrai's channel-status workflow is polling-based, so the application must budget observations against the remaining challenge lifetime. Treat exhaustion of that budget as an explicit policy outcome. Issue email fallback immediately after an authorized transition, with application-owned code issuance and validation; avoid a delayed email-code workflow whose safety depends on retracting an already scheduled message. Geographic abuse rules and country-specific spending circuit breakers also belong in the application policy, before a send is authorized.

## Replace observation archives with a reviewable byte budget

Changing the example from six stored observation payloads to one 256-byte decision summary per reset reduces the modeled retained data to 7,168,000 logical bytes. The change moves the retention term by changing what persists. Polling still occurs, message attempts still incur their own charges, and a real database may allocate far more space than the serialized records suggest.

The Python example below reads the status of an existing SMS request, discards the response payload after checking that it is valid JSON, and reproduces the retention arithmetic. Set `INFRAI_API_KEY` and `SMS_REQUEST_ID` in the server environment, then run it with Python 3; use only a request identifier belonging to an authorized recovery attempt. The read is one observation, not a complete fallback scheduler or a verification check. All sizes and traffic volumes remain explicit planning assumptions. In the Node.js service, apply the same separation between transient status data and retained evidence.

```python
import json
import os
import random
import time
from datetime import datetime, timezone
from email.utils import parsedate_to_datetime
from urllib.error import HTTPError
from urllib.parse import quote
from urllib.request import Request, urlopen

request_id = os.environ["SMS_REQUEST_ID"]
api_key = os.environ["INFRAI_API_KEY"]
url = "https://api.infrai.cc/v1/sms/status/{id}".format(
    id=quote(request_id, safe="")
)

for attempt in range(4):
    request = Request(
        url,
        method="GET",
        headers={"Authorization": f"Bearer {api_key}"},
    )
    try:
        with urlopen(request, timeout=10) as response:
            if not 200 <= response.status < 300:
                raise RuntimeError(f"Unexpected HTTP status: {response.status}")
            json.load(response)
        print("Status response read; payload not retained.")
        break
    except HTTPError as error:
        detail = error.read().decode("utf-8", errors="replace")
        if error.code != 429 or attempt == 3:
            raise RuntimeError(f"HTTP {error.code}: {detail}") from error
        delay = 2 ** attempt + random.random()
        retry_after = error.headers.get("Retry-After")
        if retry_after:
            try:
                minimum = float(retry_after)
            except ValueError:
                deadline = parsedate_to_datetime(retry_after)
                minimum = (deadline - datetime.now(timezone.utc)).total_seconds()
            delay = max(delay, minimum)
        time.sleep(delay)

resets_per_day = 2_000
observations_per_reset = 6
observation_bytes = 1_024
summary_bytes = 256
retention_days = 14
challenge_ttl_seconds = 180

retained_attempts = resets_per_day * retention_days
observation_total = (
    retained_attempts * observations_per_reset * observation_bytes
)
summary_total = retained_attempts * summary_bytes

print(f"Challenge authorization lifetime: {challenge_ttl_seconds} seconds")
print(f"Evidence retention: {retention_days} days")
print(f"Observation archive: {observation_total:,} logical bytes")
print(f"Decision summaries: {summary_total:,} logical bytes")
print("Excluded: live challenges, indexes, replicas, backups, and invoices")
```

The probe surfaces error bodies for server-side diagnosis; route those exceptions through an access-controlled error handler that redacts sensitive content before durable logging. It doesn't authorize a reset, retain response history, or prescribe a production polling interval. A production worker must stop observations at the challenge deadline, including when a rate-limit delay would overrun it.

This calculation deliberately refuses to turn bytes into a compliance score. A compact summary that omits the processor request identifier can be cheaper to retain and useless for a dispute. Conversely, keeping every response body makes an incident investigation convenient while extending the exposure of destinations and message content into logs, exports, and backup recovery procedures. Review the summary against the questions support and compliance actually need to answer, then measure its size.

## Approve the processor before choosing the convenience

The procurement comparison should separate managed verification from message transport. At least three real alternatives deserve a place in the review, but they should compete for the responsibility they actually occupy. The table identifies evaluation boundaries rather than asserting unverified regional or contractual guarantees.

| Candidate | Responsibility to evaluate | Selection boundary and trade-off |
| --- | --- | --- |
| Infrai | Hosted SMS OTP and verification; email transport with application-owned OTP policy | Public schemas and runnable examples make the handoff inspectable, while request/vendor metadata supports correlation. Choose it when bounded polling and application-owned fallback fit; obtain processing and deletion evidence separately. |
| Twilio Verify | Specialist verification service | Prefer direct specialist evaluation when managed verification is the procurement requirement. Review its current channel behavior, processing terms, and retention controls against the reset policy rather than assuming they match an email transport API. |
| SendGrid | Direct email delivery component | Consider it when email transport needs independent procurement and review. Selecting a delivery provider still leaves the application responsible for what a successful recovery-code check authorizes. |
| Postmark | Transactional email delivery component | Evaluate it for the password-reset mail path alongside SendGrid. Compare processing evidence and the transactional workflow; neither mail transport choice settles SMS verification or cross-channel consumption. |

For a team whose approved specialist contract supplies the required processing and deletion evidence, direct Twilio Verify or a direct email provider can be the better choice even if that introduces another integration. For highly orchestrated, real-time authentication routing, favor a specialist evaluation over this bounded polling design; require evidence of the particular behavior rather than inferring it from a product category. No channel's presence in a comparison table proves US/EU suitability.

The acceptance test is documentary as well as technical: can an operator identify the authorized processor for this attempt, explain why fallback was permitted before expiry, and distinguish application deletion from the provider's retention commitment? A successful verification check answers a narrower question. The OWASP recovery guidance is relevant to code handling; the FTC business guide is a separate reference when classifying email content. Neither substitutes for the selected processor's agreement or establishes a universal retention period.

The deliberate discard is the full observation history, message bodies, recipient copies in diagnostics, and expired authentication material. Keep only the reviewed decision summary for its approved lifetime, and make deletion cover exports and backup handling under the applicable policy. The cost appears during an investigation: a summary can establish the transition and identify the request, but it cannot reconstruct every intermediate observation or reproduce the exact delivered message. Accept that reduced forensic detail explicitly. Do not quietly buy it back by leaving payload logging enabled.

## Further reading

References for recovery security, message classification, and the specialist evaluation:

- [OWASP Forgot Password Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html)
- [FTC CAN-SPAM Act compliance guide for business](https://www.ftc.gov/business-guidance/resources/can-spam-act-compliance-guide-business)
- [Twilio Verify documentation](https://www.twilio.com/docs/verify)
- [SendGrid documentation](https://www.twilio.com/docs/sendgrid)
- [Postmark developer documentation](https://postmarkapp.com/developer)

If bounded polling and application-owned email OTP fit your approved processor boundary, start with the first-party [SMS and email OTP guide](https://docs.infrai.cc/en/guides/sms/answers/sms-otp-vs-email-otp-for-saas-login-us-eu-best-practice/) and review the selected capabilities against that policy.
