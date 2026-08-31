# HolmesGPT — Sonnet 4.5 investigation via the Alertmanager path

Evaluation of `holmes investigate alertmanager` on the model tier the project
recommends (Sonnet 4.5). Companion to the interactive-mode evaluation in
[`../README.md`](../README.md); covers re-test items 1, 2, 3 (partially) and 4
of the blog post's fair re-test checklist.

## Abstract

We evaluated HolmesGPT's alert-driven investigation path on two Kubernetes
environments of different scale: a minimal three-service application (Setup A)
and the twelve-service Google Online Boutique demo (Setup B). We injected
configuration faults, let alerts fire through Alertmanager, and measured root
cause accuracy, evidence quality, correlation behaviour, honesty under broken
integrations, fix correctness, and cost. Across 21 investigations, 21
identified the true root cause with quoted evidence, including one fault the
operators had not injected. Holmes performed no alert correlation: one
incident producing 13 alerts resulted in 13 independent investigations, all
correct, at roughly 4x the cost of grouped handling. First-proposed fixes
resolved the full cascade in all three verification cases. Zero runs hung.
Holmes' internal cost accounting under-reported provider spend by 34%.

## 1. Test setup and methodology

### 1.1 Common infrastructure

- HolmesGPT image: `us-central1-docker.pkg.dev/genuine-flight-317411/devel/holmes:latest`
  (built 2026-08-10; the interactive-mode round used pinned 0.30.0)
- Model: `openrouter/anthropic/claude-sonnet-4.5` via OpenRouter
- Cluster: kind, single node, docker runtime
- Monitoring: kube-prometheus-stack (Prometheus + Alertmanager), installed
  via Helm with default selectors relaxed so namespace rules are picked up
- Invocation: `holmes investigate alertmanager --alertmanager-url http://localhost:9093
  [--alertmanager-label namespace=<ns>] [--alertmanager-alertname <regex>]
  --json-output-file <file>`
- Metrics captured per investigation from the JSON report: prompt,
  completion and cached tokens, tool calls, LLM calls, and Holmes-computed
  cost. Provider spend read from the OpenRouter key endpoint.

### 1.2 Alert rules

Custom rules (expression: `kube_deployment_status_replicas_available == 0`
for 1m) fired `ShopDatabaseDown`, `ShopApiDown`, `ShopWorkerDown` (Setup A)
and `BoutiqueServiceDown` (Setup B). Stack defaults contributed
`KubePodCrashLooping`, `KubePodNotReady`, `KubeDeploymentReplicasMismatch`,
`KubeDeploymentRolloutStuck`.

## 2. Setup A — minimal three-service application

### 2.1 Deployment (namespace `shop`, manifest `app.yaml`)

| Deployment | Image | Role |
|---|---|---|
| `postgres` | postgres:16 | Database; readiness probe on TCP 5432 |
| `api` | busybox:1.36 | Exits 1 when `nc -z postgres 5432` fails |
| `worker` | busybox:1.36 | Same check loop as `api` |

### 2.2 Experiments and faults

**Experiment A1 — multi-alert burst (runs 1–2).**
Fault: `kubectl set env deploy/postgres POSTGRES_PASSWORD=""`. Postgres
refuses to initialize with an empty superuser password; both dependents lose
their database and crashloop. Result: 13 firing alerts (3 custom `Shop*`,
10 stack `Kube*`). Holmes was run twice, filtered by alert name (`Shop.*`,
then `Kube.*`).

**Experiment A2 — broken integration (run 3).**
Two variants. (a) A synthetic `ShopPaymentsDown` alert was pushed to the
Alertmanager API for a `payments` namespace that does not exist; cluster
access was healthy. (b) The real `ShopDatabaseDown` alert fired again
(postgres re-broken as in A1), but Holmes' kubeconfig was pointed at a
closed port, so every kubectl call failed with connection refused.

### 2.3 Results

**A1.** 13/13 investigations correct. Every investigation — including
generic `Kube*` alerts on dependent pods — traced the cascade to the empty
`POSTGRES_PASSWORD`, quoting the postgres container's error message. One
investigation additionally detected the config drift by diffing the
deployment's `last-applied-configuration` annotation against the live spec.
Alertmanager held the 13 alerts as 3 groups; Holmes read the flat
`/api/v2/alerts` list and ran 13 independent investigations (Section 5.2).

**A2.** Variant (a): Holmes searched the namespace, then the cluster, and
reported "the `payments` namespace does not exist"; no confabulated cause.
Variant (b): Holmes labelled generic candidates as possibilities, then
reported "Investigation Blocked: ... Kubernetes API connectivity issues
(`connection refused` to 127.0.0.1:6444)", quoting its own tool failure.
Incidental: an alert that expires between fetch and filter produces a silent
`Analyzing 0 issues` and a clean exit.

**Fix verification.** Holmes' first-proposed fix (`kubectl set env
deployment/postgres POSTGRES_PASSWORD=<value>`) was applied manually; all
three services recovered within ~3 minutes and the alerts drained. Holmes
remained read-only throughout.

## 3. Setup B — Google Online Boutique

### 3.1 Deployment (namespace `boutique`, manifest `boutique.yaml`)

The full twelve-service demo: frontend, cartservice, productcatalogservice,
checkoutservice, recommendationservice, shippingservice, currencyservice,
paymentservice, emailservice, adservice, redis-cart, loadgenerator
(generating continuous traffic).

### 3.2 Experiment B1 — subtle misconfiguration (run 4)

Fault: `kubectl patch deploy redis-cart --type=json -p '[{"op":"add",
"path":"/spec/template/spec/containers/0/args","value":["--port","9999"]}]'`.
Redis starts healthy on port 9999, but its tcp-socket probes still check
6379; kubelet kills the container every 15 seconds. The faulty service looks
healthy in its logs; its victim (cartservice) fails. The fault was chosen so
that surface symptoms point away from the culprit.

**Emergent fault.** During the experiment, cartservice was independently
OOMKilled (exit 137) under loadgenerator traffic; the demo's 128Mi memory
limit is too low on this node. This fault was not injected and was unknown
to the operators until the investigation reports identified it. The
experiment therefore ran with two simultaneous real faults.

Result: 6 firing alerts (redis-cart down / mismatch / not-ready /
crashlooping; cartservice crashlooping; a transient second
`BoutiqueServiceDown` while cartservice flapped).

### 3.3 Results

6/6 investigations correct. The four redis-cart investigations identified
the port/probe mismatch from deployment args and probe events, and supplied
the exact JSON patch to revert it. The cartservice `BoutiqueServiceDown`
investigation identified both faults and ranked them (OOMKilled primary,
redis mismatch secondary). The cartservice `KubePodCrashLooping`
investigation attributed the kill to the memory limit from exit-code
evidence alone.

**Fix verification.** Both first-proposed fixes were applied verbatim
(remove redis args; `kubectl set resources ... --limits=memory=256Mi`); all
pods stabilized with zero restarts and all boutique alerts drained within
~10 minutes.

## 4. Aggregate results

| Run | Setup | Alerts | Correct | Cost (Holmes) | Input tok | Output tok | LLM calls |
|---|---|---|---|---|---|---|---|
| run1 (`Shop.*`) | A1 | 3 | 3/3 | $0.368 | 257,215 | 5,862 | 17 |
| run2 (`Kube.*`) | A1 | 10 | 10/10 | $1.105 | 713,219 | 17,992 | 50 |
| run3 (broken integration) | A2 | 2 | 2/2 honest | $0.164 | 102,982 | 2,671 | 9 |
| run4 (Boutique) | B1 | 6 | 6/6 | $0.656 | 397,073 | 9,060 | 26 |
| **Total** | | **21** | **21/21** | **$2.29** | **1,470,489** | **35,585** | **102** |

All figures recomputed by summing `total_cost`, `prompt_tokens`,
`completion_tokens` and `num_llm_calls` across every investigation in the
run JSON files.

Per investigation: 3–7 LLM calls, 38K–106K input tokens (55–81% cache-hit),
$0.08–0.14. Zero hangs (the previous gpt-5.4 round reported uncounted
hangs). OpenRouter meter for the session: $3.08.

## 5. Discussion

### 5.1 Accuracy

On the recommended tier, the "model answers confidently and wrongly" failure
mode observed with cheaper tiers in the interactive-mode round did not occur
in any of 21 investigations. Accuracy held for generic alerts on dependent
pods, for a fault whose surface symptoms pointed away from the culprit, and
for an emergent fault the operators did not know existed. Every verdict
quoted primary evidence.

### 5.2 Correlation

Holmes performs no incident-level correlation. It reads the flat
`/api/v2/alerts` list and opens one investigation per alert, ignoring
Alertmanager's grouping (`/api/v2/alerts/groups` showed 3 groups for the 13
Setup A alerts). Causal reasoning does occur inside each investigation —
dependent-service alerts were correctly attributed to the database/redis
fault — but nothing deduplicates N alerts into one incident. The observed
consequence is economic (4x redundant spend; 13 reports for one incident),
not accuracy loss. One incident should be budgeted as N alerts.

### 5.3 Honesty under broken integrations

Missing data was reported as missing data in both variants: a nonexistent
resource was reported as nonexistent, and an unreachable cluster was
reported as blocked, with the tool error quoted. Caveat for operators: the
blocked notice follows a generic possible-causes list; a reader who stops at
the summary could mistake hypotheses for analysis.

### 5.4 Fix quality and safety

First-proposed fixes were correct and sufficient in all three verification
cases (3/3 cascades resolved). Holmes itself remained read-only throughout,
consistent with the interactive-mode round.

### 5.5 Cost accounting

Holmes' internal counter summed $2.29 against $3.08 measured provider spend
(34% under-report). Two identified sources: cache-write pricing is not
modelled (all cached tokens priced at the read rate), and aborted attempts
bill without appearing in any JSON report. Holmes also prices input at
$3.75/M from its own model table. Per-alert figures are suitable for
relative comparison; provider meters should be used for budgeting.

## 6. Limitations

Single session, single cluster (one node), two fault classes (env-var
misconfiguration, port misconfiguration) plus one emergent resource-limit
fault. Larger clusters will raise the token floor per investigation. Custom
runbook retrieval, RBAC enforcement and operator mode were not tested
(re-test items 5 and 6 remain open).

## 7. Conclusion

On the recommended model tier, HolmesGPT's alert-driven path delivered
correct, evidence-backed root causes in 21 of 21 investigations, including
faults not known to the operator, at $0.08–0.14 per alert with no hangs. Its
problems are structural rather than diagnostic: no alert correlation
multiplies cost and report volume by alert count, and its cost telemetry
under-reports by roughly a third.

## Files

- `run1-full.log`, `run1.json` — Setup A1, custom `Shop*` alerts
- `run2-full.log`, `run2.json` — Setup A1, stack `Kube*` alerts
- `run3a-full.log`, `run3a.json` — Setup A2, nonexistent resource
- `run3b-full.log`, `run3b.json` — Setup A2, unreachable cluster
- `run4-full.log`, `run4.json` — Setup B1, redis misconfig + emergent OOM
- `app.yaml`, `rules.yaml` — Setup A environment and alert rules
- `boutique.yaml`, `boutique-rules.yaml` — Setup B environment and alert rules
