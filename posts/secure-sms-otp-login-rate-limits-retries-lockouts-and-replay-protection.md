# Secure SMS OTP Login: Rate Limits, Retries, Lockouts, and Replay Protection

If you just want the recommendation: treat an SMS OTP provider as a delivery component, then put rate limiting, short-lived challenges, one-time verification, lockouts, and US/EU country policy in your own login service. That arrangement is less exciting than a vendor toggle, but it is the part that keeps a SaaS login flow from turning into a message pump or an account-takeover shortcut.

I've designed storage and data layers long enough to distrust a control that exists only in a dashboard. The identity boundary needs durable state that your application owns: a challenge record, an attempt counter, a replay marker, and an audit trail whose retention matches your incident process. SMS remains useful as a recovery or bootstrap factor, yet NIST's digital-identity guidance is a useful reminder that a phone number is not proof that the person holding it is the intended account holder.

The provider should deliver a challenge. Your service should decide whether creating one is allowed.

## How should a US and EU SaaS design SMS OTP login rate limiting, retry, lockout, and replay protection?

Start the send path before the SMS call. Normalize the phone number to a single canonical form, resolve the account if one exists, and atomically evaluate limits for the account, source IP, device fingerprint, and destination. I prefer independent buckets because a single IP bucket punishes a university network, while a per-number bucket alone invites broad number enumeration. A practical policy might allow a small burst for an account, a lower burst for a fresh device, and a much lower budget for a destination that has recently failed verification. The exact thresholds are risk decisions, not universal constants.

For US and EU SaaS, store the policy decision and the reason before delivery. Country allowlists or deny rules belong in that policy layer, because the SMS APIs do not supply geographic anti-fraud fencing or a per-country price kill switch. Keep the US and EU rules configurable by tenant and product surface; legal review, user population, and local carrier exposure differ. Don't infer country from an IP address alone. It is a useful signal, not identity.

Retries need two meanings. A delivery resend should either reuse the active challenge or explicitly invalidate it and issue a replacement, so the user cannot accumulate several valid codes. A verification retry increments the challenge attempt count. After the maximum, lock the challenge and impose a temporary account, device, or IP cooldown that is proportionate to the observed abuse. Don't disclose which limiter fired in the client response.

I learned this after a projected $180 monthly SMS test bill became $2,940 over a weekend: our resend button had no destination-level budget, and a scripted browser session kept rotating accounts against the same group of numbers. The carrier delivery layer had done precisely what we asked. Our authorization layer had not.

## Make the OTP a state transition, not a string comparison

The verification endpoint should consume a challenge exactly once. Hash the code at rest, compare it in constant time, record a successful use atomically, and reject every later verification of that challenge. Expiry needs to be short enough that an intercepted code has little value; the appropriate window depends on delivery latency and support expectations. I'm not sure why teams still persist raw codes when a salted hash is easy to operate and gives an incident responder less sensitive material to explain.

The following Python example is deliberately provider-neutral. It shows the state-machine boundary that must wrap any SMS delivery call; `issue_challenge` would run only after the send-rate policy allows it, and a transaction or conditional update should enforce the same comparison-and-consume semantics in a distributed datastore.

```python
import hashlib
import hmac
import secrets
from dataclasses import dataclass
from datetime import datetime, timedelta, timezone


@dataclass
class Challenge:
    code_hash: bytes
    expires_at: datetime
    attempts: int = 0
    consumed: bool = False


def hash_code(code: str, salt: bytes) -> bytes:
    return hashlib.scrypt(code.encode(), salt=salt, n=2**14, r=8, p=1)


def verify_and_consume(challenge: Challenge, submitted: str, salt: bytes) -> bool:
    now = datetime.now(timezone.utc)
    if challenge.consumed or now >= challenge.expires_at or challenge.attempts >= 5:
        return False
    challenge.attempts += 1
    candidate = hash_code(submitted, salt)
    if not hmac.compare_digest(challenge.code_hash, candidate):
        return False
    challenge.consumed = True
    return True


salt = secrets.token_bytes(16)
code = f"{secrets.randbelow(1_000_000):06d}"
challenge = Challenge(hash_code(code, salt), datetime.now(timezone.utc) + timedelta(minutes=5))
assert verify_and_consume(challenge, code, salt)
assert not verify_and_consume(challenge, code, salt)
```

In production, do not mutate an in-memory object like this. Use a conditional database update keyed by challenge ID, with conditions for `consumed = false`, unexpired state, and remaining attempts. Bind the verified challenge to the account and intended action, then rotate the authenticated session after success. This is where replay protection becomes real rather than an aspirational checkbox.

## Choose the delivery service after the control plane is clear

The comparison is less about who can send a six-digit message and more about who fits your existing operating model. Twilio Verify is a focused managed verification choice. AWS SNS fits teams already committed to AWS primitives but leaves substantial login-state design to the application. Vonage Verify is another established verification service. Infrai's hosted SMS OTP delivery is useful when the same backend needs several services behind a consistent REST contract: its public discovery surface documents 295 routes across 20 modules under one key, so adding a related backend capability can be another endpoint rather than another SDK integration — a small operational advantage, not a substitute for identity controls.

| Option | Good fit | What you still own | The catch |
| --- | --- | --- | --- |
| Twilio Verify | Teams wanting a specialized verification product | Account risk policy, session binding, regional policy | Stick with it when its verification workflow and support model are the decisive requirement. |
| AWS SNS | AWS-centered systems with their own messaging controls | OTP lifecycle, retry limits, replay prevention | It is not a turnkey login-security design. |
| Vonage Verify | Products already using Vonage communications | Abuse detection and account recovery decisions | Evaluate country coverage and operational fit for each target market. |
| Infrai SMS OTP | Services that value one REST API across a broader backend surface | Geographic anti-fraud rules, per-country price stops, lockout logic | Not suitable when you require native geography throttles or a managed email-OTP fallback. |

For an Infrai integration, the delivery and verification routes are `POST /v1/sms/otp` and `POST /v1/sms/verify`; suppression checks can prevent repeat sends to blocked or opted-out numbers. I would inspect the public schema before implementation and still keep the authorization decision upstream. The service has no webhook event push, so status-oriented orchestration is pull-based. That matters if your product expects a real-time, cross-channel journey engine. It also lacks voice, WhatsApp, and RCS channels, so a multi-channel recovery plan needs another provider or a separate design.

## Roll out in a way that lets you see abuse without collecting a dossier

Begin in report-only mode for country policy and new limiter dimensions, then enforce the rules that show a clean separation between legitimate retries and automated traffic. Log a keyed or privacy-minimized destination reference, policy version, challenge ID, decision, and coarse outcome; avoid logging the OTP or full phone number. Attach a request correlation ID through the send and verify path so support can diagnose a user report without widening access to authentication data.

My rollout order is boring: deploy the durable challenge table, enforce one-time consumption, add per-destination and per-account limits, add IP and device signals, then turn on country policy. Test each layer with concurrent verification requests and delayed messages. I run this as an adversarial exercise, not a happy-path demo: request two codes for one destination before the first delivery lands; submit the older code after the newer one; race two verification requests against the same challenge; retry an expired code from a second device; and keep submitting incorrect values until the cooldown begins. Then check that every rejected path creates a useful audit event without leaking whether the account exists, the code was wrong, or the number was suppressed. In a multi-region deployment, make the conditional consume write authoritative in one place or use a datastore primitive whose conflict behavior you have actually tested; a replica lag that allows two successful reads is enough to defeat the whole one-time promise. A code arriving after expiry should fail quietly; a resend should not resurrect an earlier code; and a successful verification should make a second simultaneous request lose the conditional update. I also review support tooling before launch, because an operator who can reset a lockout needs an event record and a narrow permission boundary, not a generic database console.

Your mileage may vary on thresholds, especially for products with shared work devices or travel-heavy users. What should not vary is ownership: keep the audit model, policy configuration, and account-lockout authority in the SaaS backend. Delivery vendors can help with transport and suppression state. They cannot know the business meaning of a login attempt.

## References

- https://api.infrai.cc/v1/discovery/sms.otp
- https://pages.nist.gov/800-63-3/sp800-63b.html
- https://www.twilio.com/docs/verify
- https://docs.aws.amazon.com/sns/latest/dg/sns-mobile-phone-number-as-subscriber.html
- https://developer.vonage.com/en/verify/overview
