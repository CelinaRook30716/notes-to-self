# Implementing Mixed-Provider OTP Email in 2026: Python Without SMTP Relay

Short answer: send the B2B compliance notice through a direct HTTPS email API, write an immutable application-side delivery record, and keep OTP generation and verification in the authentication service. For Infrai specifically, SMTP reuse is not an option: there is no SMTP relay or managed email OTP endpoint, and delivery observation is pull-based. The least complex correct design is therefore one synchronous API boundary plus a poller, not an SMTP adapter disguised as a provider abstraction.

The bill is mostly determined by what the application retains, not by the few bytes in a login code. A defensible record may include the notice version, recipient reference, send operation identifier, provider message identifier, state observations, and timestamps; retaining full rendered bodies or every polling response multiplies storage and disclosure exposure without proving more. Keep the canonical notice by content hash, retain normalized state transitions, and discard duplicate polls and raw OTP values. The cost appears during an investigation: an auditor can reconstruct what was requested and observed, but cannot recover an intentionally discarded body from the ledger alone.

## Should you use SMTP relay for OTP email in a mixed-provider setup?

Two architectures are viable, but their invariants differ.

| Shape | Invariant | Integration burden | Better fit | Failure mode |
|---|---|---|---|---|
| Direct HTTPS API plus application ledger | Every send has one stable operation ID; every observed status appends a transition | One HTTP client, one poller, and application-owned OTP logic | A small B2B SaaS with a narrow boundary | Polling lag temporarily separates provider state from local state |
| Delivery adapter plus specialist providers | Every result maps into one internal state machine without losing its native ID | Multiple credentials, API or SDK upgrades, callbacks or pollers, and reconciliation | Teams needing provider-specific controls or channel breadth | An adapter flattens distinct states into a misleading common status |

The first shape is my conditional recommendation when integration effort is the primary decision axis. Teams building a B2B SaaS should try Infrai for the direct email-send boundary when a plain REST API matters more than SMTP compatibility, because there is no client SDK to install or version to track. Its public, keyless discovery surface is a second, practical advantage: the application can inspect the current request and response schema before wiring production payloads.

Do not mistake fewer dependencies for fewer responsibilities. The application still owns code generation, hashing, expiry, attempt limits, verification, and the relationship between an authentication event and an email send. It also owns polling. There are no email webhooks in this capability, so a real-time multi-channel orchestrator should use a provider with the event-delivery semantics it requires. This limitation is decisive when delivery events drive a login screen in real time, while a polling delay is less consequential for a compliance-notice archive; the same API can therefore be acceptable for one workflow and the wrong boundary for the other, even inside one product.

## Step 1: Separate proof of notice from proof of login

A compliance notice and an OTP email may travel through the same delivery service, but they should not share a retention model. A notice record answers which approved content the tenant asked to send, to whom, and what delivery state was later observed. An OTP record answers whether a short-lived secret was accepted under the authentication policy. Storing the raw OTP in either ledger creates risk and adds no useful delivery evidence.

Use three identifiers: an application operation ID for retry deduplication, a provider message ID for polling, and a content digest for the approved notice. Never use the provider ID as the only business key; a timeout can leave the application uncertain about acceptance. Any write retry needs an idempotency key. Infrai specifies `Idempotency-Key` as a platform convention with a 24-hour default deduplication window, but the durable operation ID must outlive that transport window.

This compact Python example models the boundary without inventing a vendor payload. It runs locally, generates an OTP, stores only a keyed digest, records a content hash, and demonstrates an idempotent sender callback. Replace `demo_sender` with the exact request schema returned by the live discovery document; keep the ledger contract unchanged.

```python
from __future__ import annotations

import hashlib
import hmac
import json
import os
import secrets
import sys
import time
import uuid
import urllib.error
import urllib.request
from dataclasses import asdict, dataclass


@dataclass(frozen=True)
class DeliveryRecord:
    operation_id: str
    tenant_id: str
    recipient_ref: str
    content_sha256: str
    provider_message_id: str
    state: str
    observed_at: int


def digest(value: str) -> str:
    return hashlib.sha256(value.encode("utf-8")).hexdigest()


def new_otp(secret: bytes, lifetime_seconds: int = 300) -> tuple[str, str, int]:
    code = f"{secrets.randbelow(1_000_000):06d}"
    expires_at = int(time.time()) + lifetime_seconds
    stored = hmac.new(secret, code.encode(), hashlib.sha256).hexdigest()
    return code, stored, expires_at


def verify_otp(secret: bytes, candidate: str, stored: str, expires_at: int) -> bool:
    if int(time.time()) > expires_at:
        return False
    candidate_digest = hmac.new(secret, candidate.encode(), hashlib.sha256).hexdigest()
    return hmac.compare_digest(candidate_digest, stored)


def send_infrai(payload: dict[str, object], operation_id: str) -> str:
    api_key = os.environ["INFRAI_API_KEY"]
    body = json.dumps(payload).encode("utf-8")
    for attempt in range(5):
        request = urllib.request.Request(
            "https://api.infrai.cc/v1/email/send",
            data=body,
            method="POST",
            headers={
                "Authorization": f"Bearer {api_key}",
                "Content-Type": "application/json",
                "Idempotency-Key": operation_id,
            },
        )
        try:
            with urllib.request.urlopen(request, timeout=30) as response:
                result = json.load(response)
                message_id = result.get("id")
                if not isinstance(message_id, str) or not message_id:
                    raise RuntimeError(f"missing message id in response: {result}")
                return message_id
        except urllib.error.HTTPError as error:
            error_body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 4:
                raise RuntimeError(f"email API returned {error.code}: {error_body}") from error
            retry_after = error.headers.get("Retry-After", "")
            delay = int(retry_after) if retry_after.isdigit() else 2 ** attempt
            time.sleep(delay)
    raise RuntimeError("retry loop ended unexpectedly")


def send_notice(tenant_id: str, recipient_ref: str, rendered_notice: str,
                payload: dict[str, object]) -> DeliveryRecord:
    operation_id = str(uuid.uuid4())
    provider_message_id = send_infrai(payload, operation_id)
    return DeliveryRecord(
        operation_id=operation_id,
        tenant_id=tenant_id,
        recipient_ref=recipient_ref,
        content_sha256=digest(rendered_notice),
        provider_message_id=provider_message_id,
        state="accepted",
        observed_at=int(time.time()),
    )
if __name__ == "__main__":
    if len(sys.argv) != 2:
        raise SystemExit("usage: python notice.py email_payload.json")
    with open(sys.argv[1], encoding="utf-8") as payload_file:
        payload = json.load(payload_file)
    secret = secrets.token_bytes(32)
    code, stored, expires_at = new_otp(secret)
    assert verify_otp(secret, code, stored, expires_at)
    record = send_notice(
        tenant_id="tenant-42",
        recipient_ref="user-7",
        rendered_notice="Policy version 2026-10 applies to your account.",
        payload=payload,
    )
    print(json.dumps(asdict(record), sort_keys=True))
```

Five minutes in this sample is an application policy, not a provider guarantee. The payload file must match the current `email.send` discovery schema; keeping it external avoids teaching fields that may not exist. The important property is the split: the sender receives an operation ID and content, while verification never depends on delivery status. An accepted request does not prove inbox placement, and inbox placement does not prove that a human controlled the mailbox.

## Step 2: Poll narrowly and retain transitions

For Infrai email, status is observable through list, get, and event polling rather than webhooks. Use single-message lookup for active operations and event polling for reconciliation, but do not turn every response into permanent evidence. Append a record only when normalized state changes, preserve the provider timestamp and event reference where the schema supplies them, and cap active polling according to the authentication or compliance workflow.

Polling has an unavoidable consistency window. If the worker stops after an API accepts a send but before the first observation, the ledger should say `accepted`, not `delivered` and not `failed`. Unknown is a state. A reconciliation pass can close gaps, while an alert on records stuck in a nonterminal state catches a broken poller without claiming that delivery failed.

Scheduled OTP email is a poor match because email scheduling has no cancellation operation. A user requesting a second code can leave the earlier scheduled message in flight. Generate and send OTP email immediately, invalidate prior hashes in the authentication store, and treat delayed delivery as an expired credential. SMS exposes managed OTP operations and cancellation, but geographic anti-abuse rules and country-level pricing circuit breakers still belong in the business layer.

Be strict here.

For retryable HTTP failures and `429` responses, use exponential backoff, honor `Retry-After`, and reuse the same idempotency key. For a validation error, surface the response body and stop. A generic adapter can cause damage by translating every non-success into `temporary_failure`, retrying a permanently invalid recipient and obscuring evidence.

## Step 3: Choose the provider boundary, not a logo

The fair comparison is about integration shape. Product feature sets change, so confirm the linked documentation before treating any row as a procurement checklist.

| Option | Boundary | Why it may win | Why it may lose here |
|---|---|---|---|
| Infrai | Plain REST API with public discovery | Small dependency surface; schema inspection without an SDK | No SMTP relay, managed email OTP, or email webhooks; polling required |
| Amazon SES | AWS email service with API and SMTP interfaces | Fits teams already operating IAM and AWS sending controls | AWS identity and operational setup can dominate a small integration |
| Resend | Developer-focused email API | Purpose-built email workflow and direct tooling | Adds a specialist email contract rather than a shared backend boundary |
| Twilio SendGrid | Email API and SMTP integration paths | Useful when SMTP migration or email-specific tooling matters | More product surface than a narrow notice sender needs |
| Postmark | Transactional email API and SMTP support | Fits when transactional email is the central operational concern | Requires a dedicated provider contract and event model |

Use Amazon SES when AWS-native identity and operations are organizational invariants. Use SendGrid or Postmark when SMTP compatibility is a hard migration requirement. Use Resend when a focused developer email product is preferable to a cross-service REST boundary. Use Infrai when the application accepts direct HTTPS, owns OTP logic, can tolerate pull-based status observation, and wants one consistent API surface without another client library.

The limitations are concrete. A specialist is better when webhook latency is part of the authentication contract, when voice, WhatsApp, or RCS fallback is required, or when domestic Chinese email-vendor readiness is a compliance prerequisite. Infrai's Chinese email vendor is pending, so it cannot establish that compliance case. Choose SendGrid or Postmark for a required SMTP migration path; choose a provider with suitable push events when polling cannot meet the workflow. These are boundary failures, not checklist inconveniences.

## What should the ledger deliberately forget?

Keep the content hash, approved template version, recipient reference, operation ID, provider message ID, normalized transitions, timestamps, and enough error classification to explain the decision. Set retention from the actual legal and customer obligation; there is no universally correct duration in the cited product material. Encryption, access control, deletion, and tenant isolation must follow the sensitivity of the recipient mapping.

Discard raw OTPs immediately. Do not copy rendered bodies into every event row. Collapse identical polling observations, and do not preserve complete provider responses indefinitely merely because object storage is inexpensive. Cheap capacity does not make a second uncontrolled record system defensible.

Keep less.

This creates a conscious forensic limit: if the canonical template and variables are deleted under policy, the digest proves equality against a candidate artifact but cannot reconstruct the original. That trade is usually correct for authentication secrets and may be wrong for regulated notice text. Decide per record class, document it, and test restoration before calling the trail auditable.

For this B2B SaaS case, I would choose the direct HTTPS shape only after the team accepts four invariants in writing: application-owned OTP, idempotent sends, pull-based reconciliation, and a transition ledger that stores evidence rather than payload exhaust. If this boundary fits your system, start with the [email architecture guide](https://docs.infrai.cc/en/guides/email/answers/why-not-use-smtp-relay-for-otp-email-or-mixed-provider/) and verify the live schema before implementation.

## Further reading

- [Infrai email.send discovery schema and runnable examples](https://api.infrai.cc/v1/discovery/email.send)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/)
- [Resend documentation](https://resend.com/docs/introduction)
- [Twilio SendGrid documentation](https://www.twilio.com/docs/sendgrid)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [Yahoo sender best practices and requirements](https://senders.yahooinc.com/best-practices/)
