# Node.js DNS Record Deletion: List Identity Before a Single MX Cutover

An e-commerce mail cutover has two clocks: the time required to change the authoritative MX RRset, and the time existing answers can remain cached. Short answer: list the zone's records, identify the exact old MX entry by owner name, type, preference, and exchange, then delete only its provider-assigned identity after verifying the replacement RRset. Do not infer an identifier from the hostname or delete every MX record at that owner. A faster control-plane update cannot recall cached answers.

## How can I delete a single DNS record without guessing its identity?

An MX record's DNS data consists of a preference and exchange under an owner name; the preference orders mail exchangers, with lower values preferred. Multiple entries can coexist. For a shop using `shop.example` as its mail domain, `shop.example MX 10 inbound-a.example.net` and `shop.example MX 20 inbound-b.example.net` are distinct entries in the same RRset. The names here are illustrative, not observed production data. Deleting by owner name alone could remove both routes.

The DNS wire format does not supply a universal per-record deletion ID. A management API may assign one, but its meaning is local to that provider and zone. A Node.js client should therefore page through the zone's record listing, filter on the fully qualified owner and MX type, normalize the exchange's trailing dot for comparison, and compare the numeric preference too. Require exactly one matching entry with an explicit API identity. Zero matches should stop the change; two matches should stop it as well. Neither outcome licenses a guessed ID. In particular, a list response containing only the first page is not proof that the candidate is unique: finish pagination before computing the match count, and refuse the deletion if any page fails to load.

Identity is not a guess.

The read-to-delete interval matters. If the API supports conditional deletion tied to a version or revision, use it; otherwise, fetch the candidate again immediately before deletion and compare its owner, type, preference, exchange, and identity. The limitation is that a second read narrows the race but cannot make a non-transactional API atomic. Keep the operator's intended old entry and the returned identity in the change record, without recording credentials.

## Why does the cutover clock outlast the delete call?

Resolvers may cache an earlier positive MX answer until its TTL expires. RFC 1035 defines TTL as the interval for which an RR may be cached, and RFC 2181 explains that an RRset's members must have the same TTL. Reading the new answer from an authoritative server proves what that server serves now; it does not prove every recursive resolver has stopped serving the old set. Nor does deleting the old entry repair a replacement MX whose exchange has no usable address records. If the old receiving service is shut down at the instant the DNS API acknowledges deletion, senders still holding a cached old answer can attempt that now-dead route; the API's successful response is not a delivery test, and the operational response is to keep the old receiver available through the overlap rather than repeatedly deleting the same record.

Caching persists.

For an e-commerce sender, choose the overlap window according to the old RRset's TTL and the operational ability to accept mail at both destinations. Publish and verify the new route before retiring the old one, and keep the old destination able to receive during the planned overlap where possible. A short TTL set just before cutover does not shorten copies already cached under an earlier, longer TTL. If dual delivery cannot be tolerated, treat the cutover as a coordinated mail migration with an explicit interruption risk, rather than claiming a zero-delay DNS switch.

| Approach | Cutover speed | Failure mode to plan for |
| --- | --- | --- |
| Replace the old entry immediately | Fast control-plane change | Cached old answers still direct mail to the retired destination |
| Overlap old and new entries before retirement | Slower cleanup | Mail may reach either destination during the overlap; both must be ready |
| Lower TTL ahead of the change, then overlap | Additional preparation time | Previously cached answers keep their original remaining lifetime |

The comparison is about delivery continuity, not a promise of a precise global propagation time. DNS observation from several resolvers is useful evidence, but an authoritative check and a recursive check answer different questions.

## How should a client guard the destructive step?

Keep matching pure and separate from the API call. The same predicate can be translated into a Node.js `filter` over paginated records; the Python example below deliberately does not assume any provider's URL, response schema beyond this local input shape, or delete operation. The API adapter must map its actual fields and page tokens into these records before calling it.

```python
def select_old_mx(records, owner, preference, exchange):
    def dns_name(value):
        return value.rstrip('.').lower()

    matches = [
        record for record in records
        if dns_name(record['owner']) == dns_name(owner)
        and record['type'].upper() == 'MX'
        and int(record['preference']) == preference
        and dns_name(record['exchange']) == dns_name(exchange)
    ]
    if len(matches) != 1 or not matches[0].get('id'):
        raise ValueError('Expected exactly one MX entry with an API identity')
    return matches[0]['id']
```

For example, selection of `shop.example`, preference `10`, and `inbound-a.example.net` must leave the preference-`20` route untouched. Handle incomplete pagination as an error, not as an empty result. After a successful delete response, query the authoritative RRset again and verify that the intended old entry is absent while the replacement remains; retrying an ambiguous timeout without rereading can conceal a changed target. Log the matched fields and the operation outcome so the next operator can distinguish a selection failure from DNS caching.

## Rollout and rollback

Before the window, inventory the existing MX RRset, its TTL, and the replacement exchange's address resolution; validate that the new mail destination accepts the relevant domain. During the window, add and verify the replacement, preserve both receiving paths for the planned overlap, then select and retire only the old entry. Afterward, check authoritative answers and sample recursive answers while monitoring actual mail acceptance. A rollback restores the prior RRset and receiving service, but cached answers make rollback subject to the same delay as the forward change. Keep DMARC policy and reporting under review as part of mail-domain validation; RFC 7489 describes that layer, not an MX deletion mechanism.

## References

- https://datatracker.ietf.org/doc/html/rfc1035
- https://datatracker.ietf.org/doc/html/rfc2181
- https://datatracker.ietf.org/doc/html/rfc7489

## Sources

- https://datatracker.ietf.org/doc/html/rfc1035
- https://datatracker.ietf.org/doc/html/rfc2181
- https://datatracker.ietf.org/doc/html/rfc7489
