# Tenant Cutovers — TXT or CNAME for Exclusive Domain Control

Use a dedicated TXT record for domain-control proof, and reserve a CNAME for routing only after the proof is accepted. **The deciding constraint is exclusivity:** a CNAME cannot coexist with other data at its owner name, while a TXT token can live at a purpose-specific label without taking over that label's routing. For a gaming platform assigning every tenant a subdomain, this separation keeps a slow or stale proof record out of the cutover path.

TL;DR: verify `_game-claim.tenant.example` with a random, single-use TXT value; bind that exact name and tenant in durable state; then publish or accept the routing record at `tenant.example`. Never treat DNS lookup success alone as authorization, and never allow a token created for one tenant or hostname to claim another.

## Should TXT or CNAME verification prove domain control?

The record type is secondary to the invariants. A claim is valid only when the observed token equals the unexpired token issued for the same account and fully qualified domain name. Comparison should use the DNS name after consistent canonicalization, but the application must not broaden the claim from one host to its parent, sibling, or wildcard. `_game-claim.red.example` proves control of the challenge placed under that name; it does not prove control of `blue.example`.

The durable state needs an explicit uniqueness constraint on the claimed hostname. That database constraint, rather than a last-write-wins API call or the DNS cache, is the exclusivity boundary. The verifier may run twice, receive duplicate answers, or race another worker. Only one transaction may move a hostname from pending to active.

DNS isn't a lock.

Consider the smallest useful race: tenant A and tenant B both submit `arena.example`, each gets a different token, and both manage to publish their value because the DNS administrator has access to both account screens. A resolver can legitimately return both TXT records. Two verification workers then read the same answer set within milliseconds. If each worker asks only whether its own token is present and writes an ownership row without serialization, both requests can report success even though the DNS data never expressed which tenant should win. Exact matching prevents a token fragment from passing, token-to-tenant binding prevents reuse, and the unique hostname constraint makes one transaction fail. The loser must remain pending or enter a named conflict state; silently transferring the hostname would turn an administrative mistake into traffic theft. This case needs no invented outage or benchmark. It follows directly from allowing multiple TXT records and running concurrent workers.

Three failure boundaries matter. First, DNS answers can be cached, so removal is not immediate. Second, a resolver can return multiple TXT strings or multiple records; accepting a substring match turns ambiguity into an authorization bug. Third, authorization can change after verification. A permanent claim therefore needs a documented revalidation and revocation policy, not an assumption that yesterday's answer remains authoritative forever.

This is where TTL is often misunderstood. RFC 1035 defines TTL as the interval for which a resource record may be cached; it is not a deadline by which every observer will see a new answer. Negative answers can also be cached under the rules clarified by RFC 2308. Cutover timing must be measured from the resolvers the service actually uses, with a bounded retry window, rather than promised from the zone's displayed TTL alone.

## Comparing the two proof placements

| Decision point | Dedicated TXT challenge | CNAME challenge |
|---|---|---|
| Coexistence | Can share a name with other non-CNAME record types, though a dedicated label avoids unrelated TXT data | RFC 1034 requires a CNAME owner to have no other data |
| Separation from traffic | Proof can sit below `_game-claim` while the tenant hostname routes independently | Proof and aliasing are coupled if the CNAME occupies the tenant hostname |
| Rotation | Replace one scoped token without changing the traffic target | Changing proof may also change the alias path, depending on the design |
| Lookup interpretation | Must select an exact token from possibly multiple TXT answers and strings | Must validate the exact owner and expected alias target, then handle alias resolution separately |
| Best fit here | Ownership authorization before a tenant cutover | Delegated routing when aliasing is itself the intended operational contract |

TXT wins this decision because the gaming system needs two independent state machines: authorization and traffic activation. The choice is not based on TXT being universally faster. Propagation depends on authoritative publication, resolver behavior, prior cached answers, and TTLs; changing the record type does not erase those variables.

The distinction is operational, not cosmetic.

CNAME has a sharper structural limitation. RFC 1034 says that if a CNAME resource record is present at a node, no other data should be present, which prevents placing it at a name that also needs address, mail, or TXT data. RFC 2181 tightens the rule by describing a DNS alias as having no other data. A dedicated CNAME challenge label avoids some collisions, but then its main advantage over a dedicated TXT label largely disappears while alias chasing remains part of the check.

## How does the cutover avoid racing DNS propagation?

Model verification as an observation, not a sleep timer. A worker issues a cryptographically random token, stores only the claim it belongs to, queries through the normal recursive resolution path, and records the answer and time. It retries transient absence with bounded backoff. Once an exact match is observed, a database transaction acquires the hostname; routing is activated afterward.

The critical path below deliberately refuses partial matches and ambiguous ownership. It also keeps DNS I/O outside the exclusivity transaction, because a remote lookup while holding a database lock would convert resolver latency into lock contention. The resolver interface returns already-decoded TXT values; production code still needs DNS-specific handling for TXT character-string assembly and response status.

```python
from dataclasses import dataclass
from datetime import datetime, timezone
import hmac


@dataclass(frozen=True)
class PendingClaim:
    tenant_id: str
    hostname: str
    token: str
    expires_at: datetime


def challenge_name(hostname: str) -> str:
    canonical = hostname.rstrip(".").lower()
    return f"_game-claim.{canonical}"


def observed_exact_token(claim: PendingClaim, resolver) -> bool:
    if datetime.now(timezone.utc) >= claim.expires_at:
        return False

    values = resolver.resolve_txt(challenge_name(claim.hostname))
    matches = [value for value in values if hmac.compare_digest(value, claim.token)]
    return len(matches) == 1


def activate_claim(claim: PendingClaim, resolver, repository) -> bool:
    if not observed_exact_token(claim, resolver):
        return False

    # This transaction owns the exclusivity decision; DNS does not.
    with repository.transaction() as tx:
        pending = tx.lock_pending_claim(claim.tenant_id, claim.hostname)
        if pending != claim or tx.hostname_is_owned(claim.hostname):
            return False
        tx.assign_hostname(claim.hostname, claim.tenant_id)
        tx.consume_token(claim.token)
    return True
```

A token should be high entropy, expire, and become unusable after a successful transaction. Those properties close replay paths that record-type selection cannot close. Do not log the token in full. Do log the queried owner name, response category, resolver vantage point, attempt count, and transition time; those fields distinguish NXDOMAIN, an answer without the token, timeout, expiry, and an exclusivity conflict without leaking the credential.

One token. One name. One winner.

For deployment, create the claim system before making it authoritative for routing. Exercise it against a test zone with positive and negative caches, multiple TXT answers, quoted strings split into DNS character-strings, delayed publication, token replacement, and two tenants racing for one hostname. Alert on verification latency by outcome, not only on aggregate success, because a fast stream of authorization conflicts can hide a propagation problem.

## Decision record and the rejected coupling

**Decision:** use a purpose-specific TXT owner for proof, an atomic datastore constraint for tenant exclusivity, and a separate routing change for activation. The cost is one extra DNS label and a two-phase workflow. That cost is visible and recoverable; coupling authorization to live routing is harder to unwind during a launch.

The rejected design puts a verification CNAME directly at each tenant hostname and treats the alias as both proof and traffic configuration. It shortens the happy-path checklist, but it makes rollback ambiguous: removing the proof also removes or changes routing, and the alias cannot coexist with other record data at that owner. A cached alias can continue steering lookups after the control plane believes the claim was removed.

There is a valid use case for that rejected option. If a hostname is intentionally a permanent alias, needs no other data at the same owner, and the alias destination is the routing contract rather than a temporary credential, CNAME is appropriate. Even then, authorization state should bind the exact source hostname to one tenant, and cutover readiness should be observed from relevant resolvers rather than inferred from a control-plane write.

The operational rule is short: **proof may wait; ownership must serialize; routing changes last.** That ordering lets the verifier tolerate propagation delay without letting two tenants win the same name, and it keeps a failed proof refresh from becoming a game-session outage.

## References

- RFC 1034, Domain Names — Concepts and Facilities: https://datatracker.ietf.org/doc/html/rfc1034
- RFC 1035, Domain Names — Implementation and Specification: https://datatracker.ietf.org/doc/html/rfc1035
- RFC 2181, Clarifications to the DNS Specification: https://datatracker.ietf.org/doc/html/rfc2181
- RFC 2308, Negative Caching of DNS Queries: https://datatracker.ietf.org/doc/html/rfc2308
- RFC 7489, Domain-based Message Authentication, Reporting, and Conformance: https://datatracker.ietf.org/doc/html/rfc7489
