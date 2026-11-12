# Password Reset Email: Implementing Custom HTML Template Retention With Suppression in 2026

A marketplace backend should check password reset email suppression before rendering a custom HTML template, while keeping seller report attachments on a separate retention policy. Start with two delivery records and bounded attachment retention. The bill has several components: retained report bytes, report generation, storage operations, outbound transfer, message submission, and operational evidence. Count those quantities before arguing about which one dominates.

**Short answer:** keep the report attachment immutable for a defined delivery window, keep reset credentials out of that archive, and check suppression before each eligible submission. A password reset request needs a fresh, short-lived recovery flow; a report retry needs the same report bytes. Treating both as an indefinitely replayable email blob creates a retention problem and an authentication problem.

## 1. What are you actually paying to keep?

Use an explicit planning example, not an alleged production measurement: suppose a marketplace generates 12,000 seller reports per day, each occupying 2 MiB, and retains each report for 30 days. At steady state, excluding replicas, versions, and metadata, that is 720,000 MiB of report bodies. A seven-day policy reduces the retained quantity to 168,000 MiB. The difference is 552,000 MiB of payload; it is not a claim about a bill, because storage rates and other terms have not entered the calculation.

Count the bytes.

Report payload is the dominant retained-byte term in this example if the associated metadata is smaller than those bodies, but its dominance in total expenditure remains unproven until generation, transfer, submission, and operations are measured. Shorter retention moves the storage term. It doesn't reduce attachment transfer for the same number of sends, and regenerating a deleted report can add computation and reads. Choose the deletion window only after deciding how long a failed delivery may be retried and whether regeneration can reproduce the original snapshot.

Run this Python example to make the proposed change reviewable. These are scenario inputs, not service limits.

```python
reports_per_day = 12_000
report_mib = 2
retention_days = (30, 7)
retained_mib = {
    days: reports_per_day * report_mib * days
    for days in retention_days
}
for days, amount in retained_mib.items():
    print(f"{days} days: {amount:,} MiB")
print(f"Payload removed: {retained_mib[30] - retained_mib[7]:,} MiB")
```

The acceptance criterion is agreement between this model and the retention policy's scope: count report bodies here, account separately for duplicate versions and delivery logs, and establish whether deletion also reaches backups under their own schedules. A deletion marker in application metadata is insufficient evidence that every retained copy has disappeared.

## 2. Separate recovery credentials from report snapshots

The reader looking for a password reset backend example usually needs three things at once: a request handler, a custom HTML link, and a way to understand why an apparently successful request didn't yield an inbox message. For a Next.js API route backed by Node.js, place these responsibilities behind explicit interfaces; the Python examples below specify the boundary behavior without pretending to be framework-specific route code.

For this design, the public reset handler returns the same acknowledgment for known and unknown accounts. An internal service generates a random credential, retains a verifier with an expiry and consumption state, and builds a recovery URL from a configured origin. Follow the password recovery guidance in References for token generation, expiry, single use, consistent responses, and request controls. Don't allow a request's host header to choose the origin. Uniform response text alone doesn't address timing differences or abuse.

Report delivery follows a different contract. Persist the seller, report snapshot identifier, content digest, retention deadline, and delivery state before allowing a worker to submit the attachment. Use an immutable report object so that a retry cannot silently attach a newer settlement report under an older delivery record. That is a proposed application invariant; it doesn't assume any particular object store's consistency or durability guarantees.

| Artifact | Proposed retention boundary | What the boundary gives up |
| --- | --- | --- |
| Generated seller report | Seven days in this planning example | Original bytes cannot be replayed after deletion |
| Reset credential verifier | Until configured expiry or consumption | An expired request requires a new recovery flow |
| Pending reset message containing a credential | Only while that request remains eligible | Old messages cannot be replayed indefinitely |
| Delivery evidence | A separately approved operational window | Older incidents eventually lose detailed attribution |

The awkward trade-off is regeneration.

If a report reads mutable marketplace data, generating it again can produce a different result; either preserve the necessary snapshot under an explicit policy or label the result as a newly generated report. Don't call it a retry of the original attachment.

## 3. How should a backend check password reset email suppression before rendering custom HTML?

A suppression check is an application policy decision, not evidence of inbox delivery. Model an explicit state for an address that cannot currently receive transactional mail, distinguish it from a marketing subscription choice, and define who can correct that state and on what evidence. A temporary failure to read suppression state should leave the job pending under this design. It shouldn't become an invented permission to send.

The runnable example below exercises that decision and escapes the reset URL for a custom HTML attribute. It deliberately makes no network request and persists no credential. In a deployed service, generate the actual credential securely, store its verifier, and supply the URL only to the authorized delivery path. The sample token is a placeholder and must never be used as an authentication secret.

```python
from html import escape
from urllib.parse import urlencode


def prepare_reset(recipient, suppression, token, origin):
    state = suppression.get(recipient, "unknown")
    if state == "unknown":
        return {"state": "pending_policy_check"}
    if state != "eligible":
        return {"state": "suppressed"}
    url = origin + "/reset-password?" + urlencode({"token": token})
    return {
        "state": "ready",
        "to": recipient,
        "subject": "Reset your marketplace password",
        "text": f"Use this link to reset your password: {url}",
        "html": (
            '<p>A password reset was requested for your account.</p>'
            f'<p><a href="{escape(url, quote=True)}">Reset password</a></p>'
            '<p>If you did not request this, ignore this message.</p>'
        ),
    }


policy = {
    "seller@example.test": "eligible",
    "blocked@example.test": "blocked",
}
origin = "https://marketplace.example.test"
for recipient in (
    "seller@example.test",
    "blocked@example.test",
    "unchecked@example.test",
):
    result = prepare_reset(recipient, policy, "example-only", origin)
    print(recipient, result["state"])
```

Expect ready, suppressed, and pending_policy_check respectively. That distinction is the judgment criterion: a blocked address must not reach submission, and an unavailable policy lookup must not look like an affirmative eligibility decision. Never return these internal distinctions from the public recovery endpoint, where account enumeration is a separate concern.

The suppression result can change between preparation and submission.

Recheck as close to submission as the architecture permits and document that race; a local lookup cannot make an external send atomic with a database update. Report attachment assembly should occur after the same eligibility gate, avoiding unnecessary reads of seller data for jobs that cannot proceed.

## 4. Deploy with evidence of each delivery boundary

Don't compress queued, submitted, accepted, and delivered into one success flag. Record the application request identifier, report snapshot identifier where applicable, submission attempt identifier, and subsequent outcome evidence. Keep raw reset URLs and tokens out of logs. A transport acknowledgment proves only the boundary that issued it; a delivery event still doesn't establish that a person opened the message or completed recovery.

DKIM supplies a domain-level signature mechanism whose verification concerns message integrity and the signing domain, as specified in RFC 6376. It doesn't promise inbox placement. Assess sender authentication, suppression outcomes, bounce evidence, and the receiving system's response separately; a custom HTML template cannot compensate for missing operational evidence. Also measure the encoded message size against documented sender and recipient limits before submitting a report attachment. The 2 MiB planning input above isn't a universal email limit.

For deployment, exercise a blocked recipient, an unavailable policy lookup, an expired reset credential, and a worker crash after external acceptance but before the local result commits. That last failure mode creates an ambiguous submission: a retry can duplicate the email, while abandoning the attempt can lose delivery. An application job identifier helps correlate attempts but cannot, by itself, force an external transport to deduplicate them. Select an explicit reconciliation policy and evaluate any transport idempotency boundary before relying on it.

Test token consumption at the storage boundary as well: concurrent redemption must not turn a single-use credential into two successful password changes. Keep reset links usable until explicit credential submission; don't consume them merely because a URL was fetched. These are application acceptance criteria, and they belong in deployment checks alongside the retention worker's ability to respect a report that is still eligible for retry.

The useful metrics are the age of the oldest pending job, suppression outcomes, ambiguous submissions, expired recovery attempts, and report deletions that precede a permitted retry. Establish thresholds from the marketplace's delivery objective. Operational evidence needs an owner, too; otherwise the team trades a small metadata store for repeated manual investigations without ever recording that decision.

The retention policy deliberately stops keeping original report bodies after the chosen window and stops keeping usable recovery material after expiry or consumption. When an old report delivery is disputed, that choice costs exact replay unless another authorized snapshot remains; when a reset expires, it costs a fresh request. Keep enough bounded evidence to explain those outcomes. The goal is a reliable delivery contract with an honest limit on what can still be recovered.

## References

- RFC 6376, DomainKeys Identified Mail: https://datatracker.ietf.org/doc/html/rfc6376
- OWASP Forgot Password Cheat Sheet, recovery request and token controls: https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html
- Python standard library, HTML escaping: https://docs.python.org/3/library/html.html
- Python standard library, URL encoding: https://docs.python.org/3/library/urllib.parse.html

## Further reading

For the signature boundary discussed above, read RFC 6376, linked in References.

For recovery token lifecycle and request handling, read the OWASP Forgot Password Cheat Sheet, also linked in References.
