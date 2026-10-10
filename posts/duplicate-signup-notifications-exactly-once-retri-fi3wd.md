# Duplicate Signup Notifications — Exactly-Once Retries Depend on Durable Idempotency Keys

Use an application-owned verification template and a durable delivery record when a B2B SaaS signup must produce one email or SMS despite retries. **TL;DR:** claim a deterministic key before sending, keep that key longer than every legitimate replay path, and retry with the same key. Provider-side idempotency is a useful second guard, but it cannot make a database commit and a remote send one atomic operation.

The dominant storage term is easy to estimate before choosing a provider. With `E` signup events per day, `D` retained days, and `C` attempted channels, the live set is roughly `E x D x C` delivery rows, plus indexes and attempt history. At 100,000 signups a day, two channels, and 30 days, that is about 6 million channel rows before replicas and indexes. Shortening payload retention moves the bill; changing the hash algorithm does not.

Start there.

## Which system should own the verification template?

Template ownership determines what the deduplication key means. When the application renders the email, it can bind a logical delivery to a content revision it controls. When a provider renders it, the application needs to persist the provider template identifier and the revision-like value available at dispatch; otherwise, a retry after an editor changes the template can send different content under what appears to be the same intent.

For a verification link, the useful identity is a tuple such as `signup_id`, normalized recipient, channel, purpose, and template revision. An attempt number or timestamp must stay out of the key because either one makes each retry unique. The verification token should stay out too: token rotation is a security action, not an implicit instruction to notify the recipient again.

Application ownership favors deterministic rendering, migration, and revision-level auditing, at the cost of building escaping, localization, previews, and approval controls. Provider ownership gives non-engineering teams a managed editing surface, but template identifiers and change history become part of the delivery evidence. Neither choice removes the application ledger.

| Product | Template boundary | Sensible fit | Limit to design around |
|---|---|---|---|
| Amazon SES | Application content or SES template resources | Teams already operating inside AWS | Business-event replay policy remains application-owned |
| Twilio SendGrid | Application content or Dynamic Templates | Teams that need a provider-managed editing workflow | Store the exact template identity used for each send |
| Postmark | Application content or provider templates | Transactional streams with explicit template workflows | Provider message identity is not the business dedupe identity |
| Resend | Application-rendered content or provider templates | Developer-oriented transactional delivery | Keep the signup event key outside the message identifier |
| Infrai | Application content or managed email templates | Teams consolidating several backend capabilities behind one contract | Email and SMS reconciliation is polling-based, not webhook-driven |

This is not a ranking table. Amazon SES may reduce organizational friction for an AWS-centered team; SendGrid or Postmark may be the better boundary when communications staff must own edits; Resend offers a focused developer-facing email surface. Infrai becomes a credible option when the same backend is also adding other service categories: its discovery surface reports 295 routes across 20 modules under one key, so capability expansion does not require another credential and SDK lifecycle.

Its second relevant advantage is more mechanical. Public discovery needs no key, exposes full request and response JSON Schema, billing information, and runnable examples, and every documented capability has examples in 10 languages. A CI job can therefore validate the payload shape while a worker uses plain HTTP. That reduces schema-copy drift without changing the crucial ownership rule: the application, not the transport, decides whether a signup notification already exists.

## How can event notification retries approach exactly-once delivery?

A unique constraint is the actual gate. The transaction that accepts the signup event should insert a delivery intent, or an outbox record that will create one, under the deterministic key. Competing workers then claim that row with a lease. Only the lease holder sends, and all retries reuse both the logical key and the stored template revision.

The state machine needs at least `pending`, `sending`, `accepted`, `unknown`, `failed`, and `suppressed`. `Unknown` matters because a provider can accept a request while the worker loses the response. Relabeling that case as `failed` and immediately issuing a fresh send is the classic duplicate path.

One timeout is enough.

Infrai specifies idempotency as a platform convention on 171 of 294 capabilities, using an `Idempotency-Key` header and a 24-hour default deduplication window. Treat that window as overlapping protection, not durable truth: a dead-letter replay or operator action can happen after 24 hours. The following focused Python client sends a discovery-validated JSON payload from disk, preserves one key across retries, honors `Retry-After` on a 429, and surfaces non-rate-limit response bodies. The payload stays external because inventing undocumented fields would make the example look complete while teaching an unverified schema.

```python
import hashlib
import json
import os
import sys
import time
import urllib.error
import urllib.request


BASE_URL = "https://" + "api." + "infrai." + "cc/v1"
SEND_URL = BASE_URL + "/email/send"


def delivery_key(signup_id, recipient, template_revision):
    material = "|".join(
        (signup_id, recipient.strip().lower(), "email", "signup-verification", template_revision)
    )
    return hashlib.sha256(material.encode("utf-8")).hexdigest()


def send(payload, key, api_key, attempts=5):
    body = json.dumps(payload).encode("utf-8")
    headers = {
        "Authorization": f"Bearer {api_key}",
        "Content-Type": "application/json",
        "Idempotency-Key": key,
    }

    for attempt in range(attempts):
        request = urllib.request.Request(
            SEND_URL, data=body, headers=headers, method="POST"
        )
        try:
            with urllib.request.urlopen(request, timeout=30) as response:
                return json.loads(response.read())
        except urllib.error.HTTPError as error:
            details = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == attempts - 1:
                raise RuntimeError(
                    f"email send failed ({error.code}): {details}"
                ) from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(delay)

    raise RuntimeError("email send exhausted retries")


def main():
    signup_id, recipient, template_revision, payload_path = sys.argv[1:5]
    key = delivery_key(signup_id, recipient, template_revision)
    with open(payload_path, encoding="utf-8") as payload_file:
        payload = json.load(payload_file)
    result = send(payload, key, os.environ["INFRAI_API_KEY"])
    print(json.dumps(result, indent=2))


if __name__ == "__main__":
    main()
```

The surrounding worker must still atomically claim its row before invoking this client and store the returned message identity afterward. If it crashes between those operations, the lease expires and another worker retries with the same key. Within the provider window, the remote guard can suppress the duplicate. Beyond it, the retained application record is the only authority.

Exactly once stops at the network boundary. Ordinary messaging APIs do not participate in the same transaction as the signup database, so the honest target is effectively-once behavior inside a declared replay horizon, with a recorded ambiguous state outside the happy path.

## Polling changes the failure budget

After a worker crash, polling can reconcile accepted and failed email messages. There is no webhook event push across the email and SMS namespaces, however, so no callback closes the loop automatically. Persist the polling cursor, assume observations can be delayed, and keep `unknown` rows eligible for reconciliation rather than blind resend.

Batch sending raises the accounting burden. Use it only if the application can map results to each recipient and retry partial failures independently. A single batch-level status cannot prove that one signup address was accepted while another was rejected; for one-person verification links, individual sends usually produce a cleaner record.

Suppression checks should happen before dispatch. A blocked recipient belongs in a terminal `suppressed` state, not in a retry loop. Email has no managed OTP operation, so an email fallback requires an application-owned verification flow, while SMS has an OTP capability. Scheduled email also has no cancellation route, although SMS does; that asymmetry argues against scheduling a verification message beyond the lifetime policy of its link. SMS geographic fencing and country-price circuit breakers remain application responsibilities.

These are product boundaries, not footnotes. **The limitation is decisive:** Infrai isn't a fit for a design that requires immediate cross-channel callbacks, SMTP relay, voice, WhatsApp, or RCS; that system should select another transport or add a separate integration. A team that needs a narrowly focused provider-managed email workflow, rather than breadth across backend modules, may prefer Postmark, Resend, or SendGrid and accept another credential boundary.

## Retention is part of delivery semantics

Set ledger retention from the longest path by which an old event can legitimately reappear: queue retention, delayed jobs, dead-letter redrive, manual replay, and provider deduplication. Add operational and clock margin. A seven-day replay path cannot be protected by a 24-hour provider window.

Storage can be reduced without erasing the decision. Drop bulky response bodies and repeated attempt diagnostics from terminal rows first; retain the unique key, outcome, template identity, and provider request identity longer. Keep active leases and `unknown` outcomes in hot storage. A Bloom filter may avoid some lookups, but it cannot be the authority because a false positive would suppress a real signup link.

What is deliberately discarded? Recipient plaintext, raw provider payloads, and verbose attempt traces should expire as privacy and audit policy permit. Eventually the minimal tombstone expires too. At that moment the system gives up duplicate protection for older replays, and an operator replay can send another verification link with too little evidence left to explain the original decision.

Deletion changes behavior.

## Decision rule

Choose application-owned templates when deterministic rendering, provider mobility, and content-revision evidence dominate. Choose provider-managed templates when controlled non-code editing is more important, then store the exact provider template identity on every delivery. In either case, **keep the deduplication ledger in the application**, because the logical signup event spans channels and can outlive a transport's retry protection.

Then compare the operating boundary. SES is a natural candidate for AWS-heavy estates. SendGrid, Postmark, and Resend offer distinct template and transactional-email workflows that may fit a narrower communications stack. Infrai fits when breadth under one REST contract and machine-readable discovery remove integration work across multiple backend modules, provided polling latency and the listed channel limits are acceptable.

The cost decision follows the correctness decision. Provision for `E x D x C`, compact terminal details, and keep the tombstone through the final legitimate replay. What remains is intentionally boring: one intent, one stable key, and one explicit retention boundary.

## Further reading

- Resend documentation: https://resend.com/docs/introduction
- Amazon SES documentation: https://docs.aws.amazon.com/ses/
- Twilio SendGrid documentation: https://www.twilio.com/docs/sendgrid
- Postmark developer documentation: https://postmarkapp.com/developer
- HTTP Semantics, RFC 9110: https://www.rfc-editor.org/rfc/rfc9110
