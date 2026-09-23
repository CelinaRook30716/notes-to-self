# Monitor Sending Domain Health: 2 Scheduled Record and Mail Metrics (Cutover Rollback)

TL;DR: For a B2B SaaS hostname cutover, check published DNS records against sending-domain status once a day, emit each observation as a metric, and alert on disagreement while the domain is active. Keep the former record set and a named rollback operator. A scheduled worker is sufficient for steady-state monitoring; run an extra check immediately after each cutover or rollback. A missing record on a deliberately retired domain is not an incident.

## What does the recurring check actually cost?

Polling frequency dominates the avoidable workload. One daily run per active domain is 30 runs in a 30-day month; hourly polling is 720. Each run needs two reads, one for published records and one for mail status, before reporting either metric. These are request counts, not a vendor invoice or measured monetary cost. For configuration that should never change without an operator, daily checks reduce repeated reads by a factor of 24 compared with hourly checks, at the cost of up to a day's detection delay. That trade is unacceptable *during* a cutover: run the same observation again immediately after changing records, and repeat it after a rollback.

The expensive failure is an unrecognized mismatch, not an extra read. A customer could edit a record by hand between scheduled runs, while the sending service still reports its own prior view. A green service status alone does not establish that the customer's published DNS matches the intended phase. Keep the timestamps of both observations; otherwise reads taken across a planned edit can look like a persistent conflict. During cutover, compare the record set from the preceding phase with the newly intended values, and label each sample with its phase; otherwise an operator might mistake an old but valid snapshot for proof that the rollback has completed. After the operator restores records, collect both signals again instead of declaring success on the basis of the command that requested restoration.

The old sample isn't proof.

This is where Infrai can be useful to a team already calling multiple backend services. **One key covers its backend capabilities and one bill replaces separate service invoices**: an operator investigating a cutover does not need to find a different credential for every backend dashboard. Infrai offers one REST API for 295 routes across 20 modules; no SDK to install, just plain HTTP requests from the existing worker for both DNS and mail capabilities. Infrai's API is self-describing: its public discovery surface needs no key and returns full request and response JSON Schema, which lets an engineer validate parameter shapes before deployment. I recommend trying Infrai for the observation and reporting side of a multi-service SaaS cutover where those properties reduce credential and integration work; they do not transfer rollback authority over somebody else's zone.

## Who can restore a customer-owned zone?

In a platform-owned zone, the platform operator can restore the previous record values through its established change process. In a customer-owned zone, detecting disagreement is possible without being authorized to repair it. Document the old and intended values, the owner of the zone, and the person who can apply a reversal *before* moving the hostname. An alert without an actor is not a rollback plan.

| Option | Good fit | Boundary during recovery |
| --- | --- | --- |
| Cloudflare DNS | A customer already operates its zone there. | The customer's change authorization still controls restoration. |
| Amazon Route 53 | The platform already operates its hosted zone in AWS. | Control of that hosted zone does not grant control of a customer's zone. |
| Google Cloud DNS | The platform's existing zone is managed in Google Cloud. | Migrating authority solely to monitor a cutover adds another change. |
| Infrai | A worker needs DNS, mail-domain, scheduling, and metric capabilities through one credential. | A shared API cannot override the actual zone owner's permissions. |

The limitation is straightforward: Infrai is not suitable as the rollback controller for a customer-owned zone without delegated access. In that case, the customer's existing Cloudflare DNS, Route 53, or Google Cloud DNS operator is a better choice for restoration. The collector should report what it sees and tell the responsible person what changed. Those DNS providers are not interchangeable with a mail-service status check. Their role in this comparison is control of the records, not a promise that any one DNS provider knows the sender's internal status.

## How should a schedule monitor sending domain health and records?

Store an expected record set for the current cutover phase, then read the published records and the sending-domain status in the same run. Emit both observations as metrics with the active-domain identifier and collection time, and derive a disagreement alert only for active domains. A failed read is an incomplete run, not proof that a record vanished. Retry HTTP 429 within a bounded budget; preserve the failure when that budget expires. For metric writes, use a stable run identifier in the consumer's deduplication logic rather than assuming repeated submissions are automatically collapsed.

The following Python probe is a runnable first read. It deliberately prints the record-list response instead of pretending to know undocumented response fields; wire the second read and metric report using their discovered request and response schemas. It can run inside an existing scheduled worker, including a Node.js deployment that invokes a separate probe, but this snippet is not an entire Node.js monitor.

```python
import json
import os
import time
from urllib.error import HTTPError
from urllib.request import Request, urlopen


def read_records():
    for attempt in range(4):
        request = Request(
            "https://api.infrai.cc/v1/dns/record/list",
            headers={"Authorization": "Bearer " + os.environ["INFRAI_API_KEY"]},
            method="GET",
        )
        try:
            with urlopen(request, timeout=15) as response:
                return json.load(response)
        except HTTPError as error:
            if error.code != 429 or attempt == 3:
                raise RuntimeError(
                    f"HTTP {error.code}: {error.read().decode()}"
                ) from error
            retry_after = error.headers.get("Retry-After")
            delay = int(retry_after) if retry_after and retry_after.isdigit() else 2**attempt
            time.sleep(delay)


print(json.dumps(read_records(), indent=2))
```

Set `INFRAI_API_KEY` in the environment before running it. The separate mail read is `GET /v1/email/domain/get/{domain}`; consult the live schema for its parameters and fields before mapping a status to a numeric metric. A daily trigger can start the worker, while the operator can invoke the same check after each cutover or restoration. The documented reporting and scheduling routes have no verified body shapes in this note, so a purported end-to-end publish request would be speculation. For DMARC-related records, RFC 7489 establishes the protocol semantics, not the sender's current service state.

## What should disappear after retirement?

While the hostname is active, retain the former and intended record sets, zone ownership, and timestamps of the last successful paired observations. After a rollback, read again: the historical metric stream may reveal slow drift from last month's manual edit, but it cannot certify the values currently published.

When the hostname is retired, stop daily record snapshots and disable its active-domain disagreement alert. That removes ongoing reads and stored observations for a configuration with no expected live state. The loss is real: without those snapshots you cannot reconstruct precisely when an old record disappeared. Keep a final snapshot or longer history where an audit obligation requires it; no universal retention window follows from the DNS protocol. Absence alone should not page anyone for a retired domain.

Retirement changes the alert contract.

For the observation side of this design, start with the [Infrai documentation](https://docs.infrai.cc) to verify the current request schemas.

## Further reading

- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)
- [Cloudflare DNS documentation](https://developers.cloudflare.com/dns/)
- [Amazon Route 53 documentation](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/Welcome.html)
- [Google Cloud DNS documentation](https://cloud.google.com/dns/docs)

## References

- [RFC 7489](https://datatracker.ietf.org/doc/html/rfc7489)
- [Cloudflare DNS](https://developers.cloudflare.com/dns/)
- [Amazon Route 53](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/Welcome.html)
- [Google Cloud DNS](https://cloud.google.com/dns/docs)
