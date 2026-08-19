---
name: observability-csp-investigation
description: >
  Investigate cloud-provider (CSP) service issues — AWS (RDS, Lambda, SQS, ALB, EC2),
  and in future GCP and Azure — using OTel-schema cloud metrics in Elastic. Use when
  diagnosing cloud-resource alerts: database connection/CPU/memory pressure, function
  error rates and throttling, queue backlog and consumer lag, load-balancer 5xx,
  instance saturation. Correlates the alerting resource against its own baseline,
  classifies the failure mode, and rules out co-occurring but unrelated conditions.
metadata:
  author: elastic
  version: 0.1.0
---

# Cloud Provider (CSP) Investigation

Diagnose cloud-provider service issues from OTel-schema metrics and logs in Elastic. The guidelines, flow, and synthesis
below are provider-agnostic and apply to every investigation. The per-provider **reference** carries the schema facts and
failure-mode signatures that cannot be guessed — read it first.

## Router — read the provider reference first

Identify the provider from the alerting resource, then read its reference before writing any query (the index patterns,
field shapes, and stat/aggregation rules there are unguessable, and a query written without them fails or silently
corrupts its aggregates):

| Provider | Resource types | Reference |
| --- | --- | --- |
| AWS | RDS, Lambda, SQS, ALB/ELB, EC2 (`metrics-aws.*`) | `references/aws.md` |
| GCP, Azure | — | not yet covered (see below) |

If the provider is unclear, determine it from the data before assuming. For a provider without a reference yet, apply
the provider-agnostic guidance below, discover the schema empirically (list indices → field mapping → probe query), cap
confidence at medium, and say the reference was unavailable — never transplant another provider's schema.

## Guidelines

**Investigate the resource the alert names before anything else.** Shared clusters carry many systems' telemetry
(Kubernetes workloads, other teams' accounts, demo apps). Signals from co-resident systems are not evidence about this
alert unless a dependency between them is demonstrated. Do not promote a louder co-resident anomaly to root cause.

**The alert's own metric, compared to its own baseline, outranks everything else.** Before considering any other metric,
quantify the alerting metric's incident-window value against the same resource's trailing baseline (45–60 minutes
earlier). A 10×+ delta on the alert's metric is the primary thread; small wiggles in other metrics are secondary until
the primary thread is exhausted. "High" is only meaningful relative to this resource's normal.

**Counters up ≠ failing harder. Always compute the rate.** For error counters, divide by the volume counter
(errors/invocations, 5xx/requests) in BOTH windows. Volume up with rate flat = load change, not a failure. Rate up with
volume flat = a real failure in the resource's own code/config path.

**One instance degrading while its fleet siblings stay healthy = the cause is scoped to that instance.** Check the same
metric on sibling resources. Healthy siblings rule out platform-wide issues, shared event sources, and shared downstream
dependencies — the fault is in the affected instance's own code, configuration, or deployment.

**Co-occurring incidents are not causally linked by timing alone.** Attribute cross-service causation only when (a) a
dependency actually exists (the service calls that resource), and (b) the upstream degradation preceded the downstream
symptom. Two resources alerting in the same window on a busy cluster is the norm, not a clue. When you notice another
resource anomalous in the same window, TEST the dependency before linking: does the affected resource actually call it?
A worker that only touches a queue cannot be broken by a database incident. If you cannot demonstrate the dependency
from the data, report the other incident as concurrent-and-unrelated and keep diagnosing the named resource on its own
signals.

**A healthy verdict is a first-class outcome, not a failure to find the problem.** Many workloads run with an ambient
error rate of several percent at all times. If the named resource's incident-window numbers are indistinguishable from
its baseline (rate ratio near 1×; volume, duration, throttles, backlog at baseline), the correct conclusion is that the
alert is spurious or already resolved: state "ALERT FIRED BUT SYSTEM APPEARS HEALTHY", show the incident-vs-baseline
numbers, and stop. Constructing a root-cause story out of ambient fluctuation is a worse failure than reporting no
incident.

**Name exactly one primary cause; everything else is an effect, a contributor, or unrelated.** When one resource
saturates (e.g. CPU pinned at 100%), the metrics it drives rise with it — IO, latency, queue depth. Those are downstream
effects, not co-equal causes. A conclusion naming two "co-primary" bottlenecks is usually an unfinished diagnosis:
determine which one explains the other and say so.

**Logs carry the "why" that metrics cannot — cite them when available.** Metrics localize *which* component failed and rule failure modes in or out; the specific cause (the exception, the failing query, the bad config) lives in the resource's logs. After classifying from metrics, check the implicated resource's logs and cite the concrete error — turn "the function's code path is failing" into "RuntimeError at index.py:11". Logs arrive in one of two shapes and you should check both: OTel-native `logs-*.otel-*` (filter by `service.name`, line in `body.text`) and the classic cloud-integration shape (e.g. AWS `logs-aws_logs.*`, line in the `message` field). Zero rows in one shape is not "no errors" — the logs may be in the other, or not shipped at all; say which you checked. Reading a log line does not excuse misattribution: if a function logs its own exception, that is the cause, even when another resource is alerting in the same window.

**Absence of a metric field is data.** Cloud providers omit zero-valued sparse counters (error counts, per-code 5xx,
throttle counts), so the field may be unmapped in Elastic until the first such event ever occurs — and ES|QL fails the
whole query on an unknown column. Query sparse fields in separate small queries; a verification failure or empty result
on a sparse counter means "no such events ever recorded," never "data missing."

**Report insufficient evidence rather than manufacturing a conclusion.** If the named resource has zero datapoints in
the window, say so, list the exact index patterns and filters you tried, and stop. Recent windows may simply not be
ingested yet (cloud-metric pipelines typically lag by minutes — the provider reference gives the number).

## Flow

Orient (resolve the resource + window; default incident window = alert time −20m to +5m, baseline = −50m to −20m, non-
overlapping) → quantify the alert's own metric vs baseline → classify against the provider reference's failure-mode
signatures → corroborate (siblings, dependent/upstream resources only where a real dependency exists, shipped SLOs and
active alerts in `.alerts-*` via `kibana.alert.instance.id LIKE "*<resource>*"`) → synthesize and stop. Terminate as
soon as the evidence supports a classification at known confidence; exhaustive exploration is a failure mode. Anchor
severity on a VIOLATED SLO or active alert when present; never invent one.

## Synthesis

State: root cause (one sentence, named mode + resource), evidence with concrete numbers (incident vs baseline), causal
chain, scope (which resources affected, which ruled out and why), recommended action targeting the actual mode, and
confidence. Downgrade confidence when a discriminating signal was unavailable and say which. "ALERT FIRED BUT THE SYSTEM
APPEARS HEALTHY" and "insufficient evidence for the named resource" are valid, complete conclusions when the data
supports them.
