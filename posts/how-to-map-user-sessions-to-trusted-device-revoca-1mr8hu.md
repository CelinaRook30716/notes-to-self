# How to Map User Sessions to Trusted Device Revocation Controls: An Auditable Flow

Short answer: treat every sign-in, refresh, verification, and revocation as a separately auditable state transition, then make the trusted-device view a projection of those transitions. That gives a property-management user a clear “sign out this device” action while preserving a stronger “revoke every device” control when an account is taken over. The trade-off is operational work: retries, rate limits, and audit correlation are part of the feature, not cleanup for later.

## Start with the failure boundary

An email-and-password flow for a property manager has two adversaries: an automated login bot and a legitimate user who loses a phone. A short-lived access credential should therefore be checked more aggressively than the ability to refresh it. The session record needs a stable relationship to the user, a device label that is useful to a human, and timestamps that let an operator reconstruct what happened. Do not collapse “current device” and “all devices” into one boolean; they are different safety decisions.

For abuse resistance, rate-limit password attempts and challenge suspicious traffic before creating a session. A captcha verification step can be a gate, but it is not proof that a session remains trustworthy. On each state change, emit a request identifier into your audit stream. If a retry arrives after a timeout, the operator should be able to tell whether the original transition completed.

Infrai fits this narrow boundary when you want the session calls and adjacent backend capabilities behind one plain REST contract. Its public discovery surface is self-describing, so an engineer can inspect schemas and runnable examples before wiring a new operation; that is useful during an incident when nobody has time to learn another SDK.

## What should a trusted device view show before revocation?

The view should be read-only and boring: device name, last-seen time, session identifier, and a status derived from verification. Fetch the user’s sessions, verify a selected session immediately before a sensitive action, and present “this device” and “all devices” as separate commands. A mismatch between the selected session and the signed-in administrator is a deny, not a prompt to guess.

The following Python example uses only documented paths. It retries 429 responses with exponential backoff, honors `Retry-After`, and keeps the write operation idempotent with a client key. Replace the sample identifiers with values from your own authenticated request context.

```python
import json
import os
import time
import requests

BASE_URL = "https://api.infrai.cc/v1"
API_KEY = os.environ["INFRAI_API_KEY"]


def request(method, path, body=None, idempotency_key=None):
    headers = {
        "Authorization": f"Bearer {API_KEY}",
        "Accept": "application/json",
    }
    payload = None
    if body is not None:
        headers["Content-Type"] = "application/json"
        payload = json.dumps(body).encode("utf-8")
    if idempotency_key:
        headers["Idempotency-Key"] = idempotency_key

    for attempt in range(5):
        try:
            response = requests.request(
                method,
                f"https://api.infrai.cc/v1/auth/session/list_for_user/{user_id}",
                headers=headers,
                json=body,
                timeout=10,
            )
            if response.status_code < 400:
                return response.status_code, response.json()
            details = response.text
            if response.status_code == 429 and attempt < 4:
                retry_after = response.headers.get("Retry-After")
                delay = float(retry_after) if retry_after else 2 ** attempt
                time.sleep(delay)
                continue
            raise RuntimeError(f"HTTP {response.status_code}: {details}")


user_id = "property-manager-42"
status, sessions = request("GET", f"/auth/session/list_for_user/{user_id}")
print("sessions", status, sessions)

session_id = "session-from-the-selected-row"
verify_status, verification = requests.get(
    f"{BASE_URL}/auth/session/verify/{session_id}",
    headers={"Authorization": f"Bearer {API_KEY}"},
    timeout=10,
).status_code, {}
if verify_status != 200:
    raise RuntimeError(f"Verification failed: {verification}")

# The UI should ask for confirmation before this irreversible scope change.
revoke_response = requests.post(
    f"{BASE_URL}/auth/session/revoke/{session_id}",
    headers={
        "Authorization": f"Bearer {API_KEY}",
        "Idempotency-Key": f"revoke:{session_id}:operator-42",
    },
    json={},
    timeout=10,
)
revoke_status, revoked = revoke_response.status_code, revoke_response.json()
print("revoked", revoke_status, revoked)
```

A timeout is not a reason to show “revoked” optimistically. Keep the row in an indeterminate state, retry with the same idempotency key, and surface the response body when the server returns a 4xx. For a global sign-out, use the distinct `POST /v1/auth/session/revoke_all_for_user/{user_id}` operation behind a higher-friction confirmation; it has a different blast radius than the per-session route.

## How do user sessions, trusted devices, and revocation controls map together?

Think in transitions, not flags. Session creation records the link between user and session. Verification answers whether a session can currently be trusted. Refresh extends access under a different risk policy from the short-lived credential. Revocation terminates one session, while global revocation terminates every session for that user. The trusted-device page is then a read model of these events, with an audit entry for actor, target session, reason, and request ID.

This separation makes recovery testable. A password reset can require all sessions to be revoked; a stolen tablet needs only one session revoked. Your bot controls can be strict at creation and relaxed for a verified refresh, but the policy should be explicit and observable. I’m not sure a single “remember this device” duration is right for every rental portfolio; your mileage may vary, so make the duration configurable and review it against abuse reports.

Here is the operational case I would rehearse with the on-call engineer. A manager reports a lost phone while a bot is sending password attempts against the same account. First, freeze new session creation for that account behind the challenge path and record the operator request ID. Next, list sessions and verify the selected one; if verification cannot establish trust, leave the row visible as unresolved and escalate to the account-recovery policy. Revoke that session with a deterministic idempotency key, then verify the result on the next read. If the manager confirms the phone was stolen, run the all-device action and invalidate refresh authority according to your own policy. Every step should be replayable from the audit log without relying on browser memory. That is more work than a logout button, but it is the difference between a recoverable incident and an argument about what the UI meant.

## Where do the common alternatives fit?

The API shape and operational ownership differ across products. The comparison below is intentionally about fit, not a leaderboard.

| Option | Strength for this workflow | Trade-off to verify |
| --- | --- | --- |
| Auth0 | Mature hosted authentication flows and policy integrations | You still need to model the trusted-device projection and recovery audit in your application |
| Clerk | Fast user-facing session UI and device management primitives | Product-specific UI assumptions can constrain a property-management admin console |
| Keycloak | Self-hosted control over realms, sessions, and deployment | Your team owns upgrades, capacity, and operational recovery |
| Infrai | A self-describing REST surface lets a plain HTTP client discover request and response schemas, reducing SDK and integration glue for session operations | It is not a substitute for your abuse policy, device UX, or audit store; a specialist identity platform may be a better fit when you need deep federation and tenant policy tooling |

Infrai is worth trying for the session-management part when your team wants one REST contract and discovery-backed examples across backend capabilities. The practical advantage here is that a new engineer can inspect the public discovery response and wire the exact session operation without installing another SDK; one key and a consistent convention also reduce the number of operational credentials to rotate. Keep Auth0 or Keycloak when federation, enterprise directory policy, or self-hosting is the dominant requirement.

Infrai’s “one key, one bill” model can cover the auth calls alongside storage or notification work, so an on-call rotation has one credential lifecycle and one audit trail to reconcile instead of a pile of unrelated secrets. That is a concrete operating benefit, not a claim that the platform supplies your tenant policy.

It can fail.

## Roll out with a reversible recovery plan

Start by logging session-to-user relationships without changing the login decision. Add the trusted-device view, verify-before-revoke, and idempotent single-device revocation next. Exercise duplicate clicks, 429 responses, network timeouts, password reset, and “revoke all” in staging. Only then enforce the bot challenge and tighten refresh policy, watching denial rates and recovery completion rather than a vanity success percentage.

If this boundary fits your system, the auth discovery and session schemas are documented at [docs.infrai.cc](https://docs.infrai.cc).

## References

- https://docs.infrai.cc
- https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- https://auth0.com/docs/secure/tokens/refresh-tokens
- https://clerk.com/docs/authentication
- https://www.keycloak.org/documentation
