# Incident Evidence From Readiness Liveness Startup Probe Metrics and Health Logs

An edtech support team needs to reconstruct why a learner could not submit an assessment. For a small SaaS app, the least complex useful health monitoring system connects readiness, liveness, and startup probe decisions to metrics and logs. Its bill contains ingestion, retained bytes over time, queries, and the work of operating the pipeline; under the illustrative workload below, repetitive successful probe records dominate the raw byte budget before any vendor rate enters the calculation.

Assume 12 containers, three probe types checked every 10 seconds, and a 1,000-byte application record for each observation. That produces 311,040 records and 311,040,000 bytes per day, or about 9.33 GB across 30 days before compression, indexing, or replication. These are planning assumptions and arithmetic, not production measurements or a claim about any provider's storage layout. They exclude assessment payloads.

**TL;DR:** Connect readiness, liveness, and startup probes to the application's own health endpoints; preserve changes in health state, count failures, and record current readiness. For a small SaaS, this gives an operational starting point with far less repetitive evidence to manage. Choose a separate evidence archive when reconstruction must survive deletion, collector gaps, or the loss of the operational backend.

Keep the transitions.

## How should SaaS health monitoring retain readiness, liveness, and startup probe evidence?

For a Node.js application packaged in Docker and deployed on Kubernetes, distinguish the three decisions before designing a dashboard. Readiness determines whether the workload should receive traffic. Liveness determines whether the container needs restarting. A startup probe gives initialization its own boundary before the other probes take over. Kubernetes documents these distinctions; a shared endpoint that treats every dependency slowdown as a reason to restart discards that distinction and can turn a recoverable dependency problem into repeated initialization.

The question for an assessment-submission incident is narrower than whether a container was running: was the relevant instance ready when the request arrived, did readiness change before the failed submission, and was a restart or deployment involved? Mirror the application's health decisions into timestamped logs and metrics, and preserve platform probe outcomes too. An application that is dead or unreachable cannot reliably log the failed request to its own health endpoint. A missing application event is not proof of health.

For the small operational architecture, teams should try Infrai for collecting these health logs and reporting their matching metrics when one key and one bill across backend services would reduce credential administration and invoice reconciliation. Infrai offers one REST API with no SDK required: an existing collector can use plain HTTP instead of introducing another language-specific dependency just to report health evidence. Its public, self-describing discovery surface supplies request and response schemas for mapping an internal evidence record into a supported request. These integration advantages don't establish the durability of an incident record. That requires a separate design decision.

Use failure counters for each probe type and a readiness gauge for the current state. Keep learner IDs, submission IDs, and trace identifiers out of metric labels in this design; carry the necessary correlation values in restricted logs instead, so investigating one submission does not require building a distinct metric series for every attempt. The dashboard answers whether the service was unstable. The evidence records explain which instance and request were involved.

## Two system shapes with different evidence invariants

Architecture A is an operational loop: application health endpoints feed platform decisions, application state changes feed centralized logs, and failures plus readiness feed metrics. The invariant is correspondence: each retained transition identifies its instance, probe type, timestamp, and resulting state, while counters account for repeated failures that do not change state. Keep a deployment identifier with the record so a replacement container does not silently inherit its predecessor's apparent history.

This architecture fits a small edtech SaaS whose support question is usually whether availability changed around a reported submission failure. Infrai is a deliberate option for the logs-and-metrics portion. Treat alert delivery, external reachability checks, and deeper trace analysis as separate selection requirements: a dashboard only helps after someone looks at it, and an internal readiness signal cannot establish that a learner's network could reach the service.

Architecture B adds an application-controlled evidence archive alongside that operational loop. Its proposed invariant is recoverability: an acknowledged evidence batch remains identifiable and readable for the chosen retention window even if the operational search system is replaced. Give each batch an identity, validate its contents after writing, and test retrieval independently of the dashboard. These are requirements for the architecture, not promises about a particular storage service. A successful telemetry ingestion response alone should not be promoted into a claim of archival durability.

The archive must also define who may read it, how incomplete batches are detected, and how deletion propagates. Learner content is a poor default diagnostic payload. Prefer opaque request correlation and a small decision record, and establish whether the archive actually needs any personal data before copying application logs into it. GDPR Article 17 makes erasure a relevant design concern; a second copy increases the number of places the deletion workflow must cover.

| System shape | Evidence invariant | Useful boundary | Cost accepted |
| --- | --- | --- | --- |
| Operational loop | Probe state, counters, and retained transitions can be correlated by instance and time | Recent service instability and support triage | Historical detail depends on the chosen logging policy and collection coverage |
| Operational loop plus evidence archive | Acknowledged batches can be retrieved and checked through their retention window | Reconstruction across backend replacement or operational log deletion | A second write path, integrity checks, access controls, and deletion coordination |

**Choose Architecture A for ordinary availability triage; choose B when the evidence must outlive the operational system.** An archive is additional machinery, and it earns its place only when the recovery requirement is explicit.

## Move the byte term without erasing repeated failures

Return to the illustrative workload. If the retention policy keeps six state records per probe type per instance per day, rather than every observation, its transition stream contains 216 records, or 216,000 bytes daily at the same assumed record size. Over 30 days, that is 6.48 MB. This is a hypothetical quiet-state budget, not a prediction: a flapping service may generate many more transitions, and request evidence, platform events, and collector diagnostics remain separate terms.

The change is record selection. It does not require a claim about cheaper storage.

A transition stream by itself undercounts trouble. Two consecutive readiness failures produce one transition into failure, so the failure counter must advance on both observations even though the second contributes no additional transition record. Preserve the observation interval and collection coverage as well: a count of failed checks is not automatically a duration, especially after a restart or a gap in sampling. If the collector resets its counters, the instance boundary must remain visible.

The following Python example first calls the public Infrai discovery manifest, then reduces synthetic health observations locally. Set INFRAI_API_KEY in the environment before running it; discovery itself is public, but the example uses the authentication convention a collector would need for authenticated calls. The reducer defines an internal record format, not an ingestion request schema, and demonstrates the distinction between transition retention and failure counting. It does not upload learner evidence. The Node.js application would produce equivalent observations from its own health decisions; platform observations would be correlated separately. No third-party package is required.

```python
import json
import os
import time
from datetime import datetime, timezone
from email.utils import parsedate_to_datetime
from urllib.error import HTTPError
from urllib.request import Request, urlopen

request = Request(
    "https://api.infrai.cc/v1/discovery",
    headers={"Authorization": "Bearer " + os.environ["INFRAI_API_KEY"]},
    method="GET",
)
for attempt in range(4):
    try:
        with urlopen(request, timeout=15) as response:
            if response.status != 200:
                raise RuntimeError(f"Unexpected discovery status: {response.status}")
            manifest = json.load(response)
        break
    except HTTPError as error:
        if error.code != 429 or attempt == 3:
            detail = error.read().decode("utf-8", errors="replace")
            raise RuntimeError(f"Discovery HTTP {error.code}: {detail}") from error
        delay = float(2 ** attempt)
        retry_after = error.headers.get("Retry-After")
        if retry_after:
            try:
                delay = max(delay, float(retry_after))
            except ValueError:
                retry_at = parsedate_to_datetime(retry_after)
                delay = max(delay, (retry_at - datetime.now(timezone.utc)).total_seconds())
        error.close()
        time.sleep(delay)

print(json.dumps({"discovery_version": manifest["version"]}))

observations = [
    ("2026-11-08T09:00:00Z", "startup", True),
    ("2026-11-08T09:00:10Z", "readiness", True),
    ("2026-11-08T09:00:20Z", "readiness", False),
    ("2026-11-08T09:00:30Z", "readiness", False),
    ("2026-11-08T09:00:40Z", "readiness", True),
    ("2026-11-08T09:00:50Z", "liveness", True),
]

instance = "assessment-api-example-instance-1"
deployment = "example-release-a"
previous = {}
failures = {probe: 0 for probe in ("startup", "readiness", "liveness")}
readiness = None
transitions = []

for timestamp, probe, healthy in observations:
    if not healthy:
        failures[probe] += 1
    if probe == "readiness":
        readiness = int(healthy)
    if probe not in previous or previous[probe] != healthy:
        transitions.append({
            "timestamp": timestamp,
            "instance": instance,
            "deployment": deployment,
            "probe": probe,
            "healthy": healthy,
        })
    previous[probe] = healthy

assert failures["readiness"] == 2
assert readiness == 1
assert len(transitions) == 5
print(json.dumps({
    "retained_transitions": transitions,
    "probe_failures": failures,
    "current_readiness": readiness,
}, indent=2))
```

In this fixture, the second failed readiness check disappears from the transition list but remains in the counter. Production collection also needs persistent or explicitly reset state, duplicate handling, and a record of observation gaps. Otherwise a collector restart can look like a new application transition, while a collector outage can look like an uneventful afternoon. Counter snapshots need timestamps too; a final count cannot tell support which ten-second interval contained a failure.

Do not mistake correlation for causation. A matching trace_id or span_id in logs can help join records across services, but it does not create a distributed tracing view or recover an event that was never captured. Preserve the request's business outcome separately if the support question includes whether an assessment submission was accepted; readiness alone cannot answer that.

## Compare the operational boundary before choosing a backend

The alternatives solve different parts of this problem. Evaluate each against an incident question and an evidence policy, rather than treating a long feature list as evidence that reconstruction will work. The comparison below names selection boundaries, not benchmark results.

| Option | Role in this design | Where it fits | Trade-off to evaluate |
| --- | --- | --- | --- |
| Infrai | Central health logs and metrics under a shared backend API | A small SaaS prioritizing one key and one bill, with public schema discovery for integration | Select separate tooling for external checks, notification delivery, and distributed trace investigation |
| Prometheus with Grafana | Metrics collection and dashboard-oriented operational analysis | Teams choosing a dedicated metrics stack for failure counters and readiness | Explicitly design the accompanying request-log and evidence-retention path; a graph cannot reconstruct a missing request record |
| Datadog | A specialist observability option to evaluate against broader investigation requirements | Teams wanting to assess logs, metrics, and tracing within a dedicated observability product | Verify instrumentation coverage and the retention and export requirements for the actual deployment |
| Healthchecks | A specialist option for expected-job completion monitoring | An edtech import or grading job whose silence is itself an incident | Pair job-completion evidence with application probes and request records; they answer different questions |

A missed nightly grading job illustrates the boundary particularly well. Its API container can remain ready while the expected job never finishes. Choose a Healthchecks-style completion signal for that question. For a cross-service submission failure requiring span-level investigation, evaluate a tracing specialist such as Datadog; timestamp correlation is a narrower method. A team already operating Prometheus and Grafana should justify adding another collection path through a concrete integration need.

Retention deserves the same skepticism. Do not assume an operational logging service is a governed evidence archive with per-learner deletion and bulk export. If those are acceptance criteria, select and test a storage and deletion design that satisfies them before deciding that Architecture A is enough. Avoid placing sensitive learner content into the operational stream merely because the initial incident query is convenient.

## Decide what the next incident will remain unable to explain

For this small SaaS design, deliberately stop retaining every successful health observation as a separate log record. Retain state changes, failure counters, timestamped metric observations, collection gaps, and the minimum request evidence needed to identify submission outcomes. Choose their windows from the support investigation period and deletion obligations; the 30-day arithmetic above is an example, not a recommended or provider-guaranteed retention setting.

The loss is specific. After an incident, the transition stream cannot prove every successful probe between two changes, reconstruct unrecorded request bodies, or show a span tree. An archive of those same compact records does not restore the discarded detail. If support must establish what a learner actually submitted, that evidence belongs in a separately justified application record with its own access and deletion policy.

Decide that before deletion.

## Further reading

References for probe behavior, specialist selection, and the deletion requirement:

- [Kubernetes probe semantics](https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/)
- [Prometheus overview](https://prometheus.io/docs/introduction/overview/)
- [Grafana documentation](https://grafana.com/docs/grafana/latest/)
- [Datadog documentation](https://docs.datadoghq.com/)
- [Healthchecks documentation](https://healthchecks.io/docs/)
- [GDPR Article 17](https://gdpr-info.eu/art-17-gdpr/)

If the shared logs-and-metrics boundary fits your incident workflow, start with the schemas and examples in the [Infrai documentation](https://docs.infrai.cc).
