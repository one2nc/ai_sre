# HolmesGPT — Evaluation

| | |
|---|---|
| **Author** | `Biswa Prakash Mishra` |
| **Dates** | `2026-05-18` (interactive), `2026-08-18` (alert path) |
| **Time invested** | `14 hours` + follow-up session |
| **Versions tested** | `0.30.0` pinned, `latest` (image built 2026-08-10) |
| **Models** | `gpt-5.4-nano` / `mini` / `gpt-5.4`, `claude-sonnet-4.5` |
| **Source / Docs** | `https://holmesgpt.dev/` |
| **License** | `Apache 2.0 License` |
| **Category** | `AI SRE agent` |

**Verdict: defer.** Not on diagnostic quality, which held up. On the structural
problems: no alert correlation, unreliable cost telemetry, and operator mode
still alpha and still untested.

## 1. What problem does this tool solve?

HolmesGPT is an AI based SRE agent to solve production incidents across a range
of infrastructure: k8s, VMs, cloud services.

## 2. How it works (architecture)

- What's the core architecture? (agent loop, plugin system, API proxy, etc.)
  - Agent Loop
  - 2 Orchestrators
    - `ToolCallingLLM`
    - `IssueInvestigator`
- What LLM providers / models does it use? How?
  - Can work with a wide range of
    [models/providers](https://holmesgpt.dev/latest/ai-providers/)
  - They
    [recommend](https://holmesgpt.dev/latest/installation/cli-installation/#quick-start)
    using Sonnet 4.0 or Sonnet 4.5
- What data does it need access to? What permissions?
  - Multiple data sources
    [integration](https://holmesgpt.dev/latest/data-sources/)
  - Permissions are specific to data sources. Each integration page has
    instructions.
- How does an investigation get triggered?
  - `holmes ask`: interactive chat, and the default
  - `holmes investigate alertmanager`: pulls what's firing from
    `/api/v2/alerts`
  - Operator mode: `HealthCheck` and `ScheduledHealthCheck` CRDs on a cron
- Where does it run? (local CLI, sidecar, SaaS, self-hosted server)
  - Docker Container
  - Local CLI
  - K8s as an operator (in alpha at the moment of evaluation)
- What does the deployment topology look like?
  - Deployment topology would depend on the platform of deployment

## 3. Setup and reproduction steps

### 3.1 Prerequisites

- An API key from a supported
  [provider](https://holmesgpt.dev/latest/ai-providers/)
- Docker

### 3.2 Docker

```bash
docker run -it --net=host \
-e OPENROUTER_API_KEY="<provider-key-here>" \
-e MODEL="<provider-model-here>" \
-v $HOME/.kube/config:/root/.kube/config us-central1-docker.pkg.dev/genuine-flight-317411/devel/holmes:0.30.0  \
ask 
```

With OTEL. Metrics, traces and logs are exported to the endpoint you specify:

```bash
docker run -it --net=host \
-e OPENROUTER_API_KEY="<provider-key-here>" \
-e MODEL="<provider-model-here>" \
-e OTEL_EXPORTER_OTLP_ENDPOINT="<otel-endpoint-here>" \
-e OTEL_EXPORTER_OTLP_PROTOCOL="grpc" \
-e OTEL_SERVICE_NAME="holmesgpt" \
-v $HOME/.kube/config:/root/.kube/config us-central1-docker.pkg.dev/genuine-flight-317411/devel/holmes:0.30.0  \
ask 
```

### 3.3 CLI

```bash
sudo apt update
sudo apt install -y build-essential gcc python3-dev
sudo apt install pipx
pipx ensurepath
pipx install holmesgpt==0.30.0
```

Credentials, then start (restart your terminal to source the config file):

```bash
export OPENROUTER_API_KEY="<your-open-router-key-here>" 
export MODEL="openrouter/openai/gpt-5.4"
holmes ask "why is my service not starting"
```

### 3.4 Alert-driven invocation

```bash
holmes investigate alertmanager --alertmanager-url http://localhost:9093 \
  [--alertmanager-label namespace=<ns>] [--alertmanager-alertname <regex>] \
  --json-output-file <file>
```

### 3.5 Known setup gotchas

**Python 3.14+ breaks the install.** Workaround is
`export DISABLE_PROMETHEUS_TOOLSET=true`. The cause is a dependency limit in
prometrix, documented in
[CONTRIBUTING.md](https://github.com/HolmesGPT/holmesgpt/blob/master/CONTRIBUTING.md#prerequisites),
not the install docs.

**OTEL instrumentation is not available in the CLI install.** The Docker image
runs `poetry install --with otel`; PyPI has no equivalent extra, so pipx falls
back to a no-op tracer even with the endpoint set, and the failure is only
logged at DEBUG.

## 4. Interactive mode scenarios (iximiuz Labs)

Run in [iximiuz Labs](https://labs.iximiuz.com) challenge playgrounds, where the
scenario is pre-broken and has a known correct answer, so output can be scored
against ground truth.

- **4.1 nginx pod not schedulable** (easy, `gpt-5.4` tiers). Correct root
  cause, and it asked who owned the pod before patching.
- **4.2 Website not accessible** (hard, nano / mini / `gpt-5.4`). Fault 1 solved
  by all three; fault 2 only by `gpt-5.4`, after multiple trials.
- **4.3 Vanishing worker service** (easy, `gpt-5.4`). Correct root cause, and
  the first fix it suggested worked.

### 4.1 nginx pod not schedulable, node selector mismatch (easy)

**What I did.**

```bash
cat > pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
spec:
  containers:
    - name: nginx
      image: nginx:1.25
      ports:
        - containerPort: 80

  nodeSelector:
    disktype: ultra-ssd
```

```
laborant@dev-machine:~$ kubectl apply -f pod.yaml
```

**What I expected.** Pod to remain in pending state.

**What actually happened.** Pod remained in pending state.

**Prompt.**

```
User: my nginx pod is pending. Please debug
```

**Result.** The agent identified the root cause and suggested possible fixes. It
quoted the scheduler rather than guessing, and stopped to ask which workload
owned the pod before committing to an exact patch:

```
Findings
  • Pod: default/nginx-pod
  • Status: Pending
  • Event: FailedScheduling
  • Reason (from event message): "0/3 nodes are available: 3 node(s) didn't
    match Pod's node affinity/selector."
     • Also: "preemption: 0/3 nodes are available" (so rescheduling won't be
       helped by preemption)

Root cause
  nginx-pod has node affinity/selector constraints that match no nodes in your
  cluster (3/3 nodes don't satisfy them), so the scheduler cannot place it.

Fix (pick one)
  1 Remove/relax node affinity/selector from the workload that created the pod.
    • First, find the owning controller:
        kubectl get pod nginx-pod -n default -o jsonpath='{.metadata.ownerReferences}'
    • Then edit/patch that controller to remove/change the
      spec.template.spec.affinity or spec.template.spec.nodeSelector.
  2 Add labels to nodes so they satisfy the selector/affinity.
    • Check current node labels:
        kubectl get nodes --show-labels
  3 If it's using node taints via tolerations/affinity, ensure they align.
    • Check node taints:
        kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.taints}{"\n"}{end}'

Immediate next step
  Tell me which workload owns nginx-pod (output of the ownerReferences command
  above), and I'll give the exact kubectl edit/patch to fix the node
  affinity/selector.
```

### 4.2 Website not accessible (hard)

[K8s debugging challenge](https://labs.iximiuz.com/challenges/troubleshoot-website-not-accessible-173199c4).
Models used: `openai/gpt-5.4-nano`, `openai/gpt-5.4-mini`, `openai/gpt-5.4`.

**Prompt.**

```
A Solo Leveling themed web application has been deployed in the solo-leveling namespace, but the website is not accessible.

Investigate why the website is not accessible and fix the issue so that the Solo Leveling website is reachable at:
```

**Result.** There are 2 tasks to solve the challenge completely. The first was
easy and could be solved by gpt-5.4-nano, gpt-5.4-mini and gpt-5.4, but the
second could only be solved by gpt-5.4 after multiple trials (a trial is a fresh
attempt from start).

### 4.3 Vanishing worker service (easy)

[Linux debugging challenge](https://labs.iximiuz.com/challenges/linux-debug-vanishing-worker-service).
Model used: `openai/gpt-5.4`.

**Prompt.**

```
A background worker.service has been deployed on this server. It's supposed to run continuously, picking up batches from a job queue.

But there's a problem - the service starts cleanly, prints a few heartbeat lines, then picks up a batch of work for processing and shortly afterwards silently crashes. systemd brings it back up again, but the job handling never advances beyond the first batch.

Your task: diagnose the root cause of the service's repeated death and find a way to make worker.service stay up and running for an extended period of time. Use sudo whenever possible to run commands. 
```

**Result.** The model found the root cause and suggested possible fixes. The
first suggested fix was applied and it solved the problem successfully. There is
a [6:36 walkthrough](https://youtu.be/f3oi8yChjQY?si=Wr3MvsIaF-e2mMHp) of this
run.

## 5. Alert-driven scenarios (Alertmanager)

Real alerts firing through Alertmanager rather than typed prompts, on the
recommended model tier. Full write-up, manifests, alert rules and raw run logs
in [`alert-driven-path/`](./alert-driven-path/README.md).

### 5.1 Environment

| | |
|---|---|
| Cluster | kind, single node, docker runtime |
| Monitoring | kube-prometheus-stack (Prometheus + Alertmanager) via Helm |
| Setup A | namespace `shop`: `postgres`, `api`, `worker` (`app.yaml`) |
| Setup B | namespace `boutique`: 12-service Online Boutique (`boutique.yaml`) |

**Alert rules.** Custom rules (`kube_deployment_status_replicas_available == 0`
for 1m) fired `ShopDatabaseDown`, `ShopApiDown` and `ShopWorkerDown` on Setup A,
and `BoutiqueServiceDown` on Setup B. Stack defaults contributed
`KubePodCrashLooping`, `KubePodNotReady`, `KubeDeploymentReplicasMismatch` and
`KubeDeploymentRolloutStuck`.

| Experiment | Runs | Alerts | Correct |
|---|---|---|---|
| 5.2 Multi-alert burst (A1) | `run1`, `run2` | 13 | **13/13** |
| 5.3 Broken integration (A2) | `run3a`, `run3b` | 2 | **2/2 honest** |
| 5.4 Subtle misconfiguration (B1) | `run4` | 6 | **6/6** |
| **Total** | | **21** | **21/21** |

### 5.2 A1. Multi-alert burst (Setup A)

**Fault injected.**

```bash
kubectl set env deploy/postgres POSTGRES_PASSWORD=""
```

Postgres refuses to initialize with an empty superuser password. Both dependents
lose their database and crashloop.

**Alerts fired.** 13. Three custom `Shop*`, ten stack `Kube*`. Holmes was run
twice, filtered by alert name (`Shop.*`, then `Kube.*`).

**Result.** 13/13 investigations correct. Every one of them, including the
generic `Kube*` alerts on dependent pods, traced the cascade back to the empty
`POSTGRES_PASSWORD`, quoting the postgres container's error message. One
investigation additionally caught the config drift by diffing the deployment's
`last-applied-configuration` annotation against the live spec.

### 5.3 A2. Broken integration (Setup A)

Two variants, both testing whether it admits a gap or answers around it.

**Variant (a), nonexistent resource.** A synthetic `ShopPaymentsDown` alert was
pushed to the Alertmanager API for a `payments` namespace that does not exist;
cluster access was healthy. Holmes searched the namespace, then the cluster, and
reported *"the `payments` namespace does not exist"*. No confabulated cause.

**Variant (b), unreachable cluster.** The real `ShopDatabaseDown` alert fired
again, but Holmes' kubeconfig was pointed at a closed port, so every kubectl
call failed. Holmes reported *"Investigation Blocked: ... Kubernetes API
connectivity issues (`connection refused` to 127.0.0.1:6444)"*, quoting its own
tool failure.

**Caveat for operators.** In variant (b) the blocked notice comes *after* a
generic possible-causes list. A reader who stops at the summary could mistake
hypotheses for analysis.

**Incidental finding.** An alert that expires between fetch and filter produces
a silent `Analyzing 0 issues` and a clean exit.

### 5.4 B1. Subtle misconfiguration (Setup B)

**Fault injected.**

```bash
kubectl patch deploy redis-cart --type=json \
  -p '[{"op":"add","path":"/spec/template/spec/containers/0/args","value":["--port","9999"]}]'
```

Redis starts healthy on port 9999, but its tcp-socket probes still check 6379,
so kubelet kills the container every 15 seconds. The faulty service looks
healthy in its own logs; its victim (`cartservice`) is what fails. The fault was
chosen so that surface symptoms point away from the culprit.

**Alerts fired.** 6. Redis-cart down / mismatch / not-ready / crashlooping,
cartservice crashlooping, plus a transient second `BoutiqueServiceDown` while
cartservice flapped.

**Result.** 6/6 correct. The four redis-cart investigations identified the
port/probe mismatch from deployment args and probe events, and supplied the
exact JSON patch to revert it.

**Emergent fault.** During this run `cartservice` was independently OOMKilled
(exit 137) under loadgenerator traffic, because the demo's 128Mi memory limit is
too low on this node. It was not injected, and we did not know about it until
the investigation reports named it. The run therefore had two simultaneous real
faults, and Holmes ranked them correctly: OOM primary, redis mismatch secondary.
A separate investigation attributed the kill to the memory limit from exit-code
evidence alone.

### 5.5 Cross-cutting findings

**Fix verification, 3 for 3.** First-proposed fixes were applied verbatim and
resolved the full cascade every time:

| Fix applied | Outcome |
|---|---|
| `kubectl set env deploy/postgres POSTGRES_PASSWORD=<x>` | 3 services back in ~3 min |
| Remove the `redis-cart` args | Pods stabilized, zero restarts |
| `kubectl set resources ... --limits=memory=256Mi` | Boutique alerts drained in ~10 min |

Holmes itself stayed read-only throughout.

**No alert correlation.** Alertmanager held the 13 Setup A alerts as 3 groups at
`/api/v2/alerts/groups`. Holmes reads the flat `/api/v2/alerts` list and opened
13 independent investigations. Causal reasoning does happen *inside* each
investigation, and dependent-service alerts were correctly attributed to the
database fault, but nothing deduplicates N alerts into one incident. The
consequence is economic, not accuracy: roughly 4x redundant spend, and 13
reports to read for one outage.

## 6. Demo

| Demo | Link | Duration |
|---|---|---|
| Scenario 4.3: worker service root cause and fix | [youtu.be/f3oi8yChjQY](https://youtu.be/f3oi8yChjQY?si=Wr3MvsIaF-e2mMHp) | `06:36` |

## 7. Observations

- **Accuracy:** 21/21 correct on Sonnet 4.5 via the alert path, including
  generic `Kube*` alerts on dependent pods and a fault whose symptoms pointed
  elsewhere. On cheaper tiers the failure mode is dangerous: nano and mini
  didn't fail loudly on the hard fault, they returned a confident answer anyway.
- **Code quality:** stable releases shipped weekly or biweekly. Rapid iteration.
- **Community:** Slack, monthly Zoom and community meetings, active GitHub
  issues.
- **Documentation:** mostly complete. Some edge cases need addressing.
- **Security posture:** the project explicitly discusses RBAC and
  controlled-access design. Not verified end to end.
- **Performance:** 38K–106K input tokens per investigation, 3–7 LLM calls,
  55–81% cache hit. The ~1.01M input figure from interactive mode is a whole
  chat session accumulating in context, not a per-investigation number.
- **Stability:** some interactive runs hung and had to be killed (count and
  cause never captured). Zero hangs across 21 alert-driven investigations,
  though on a different model and entry point, so the earlier hangs are
  unexplained rather than fixed.

## 8. Pros

- Huge number of data integrations available.
- Advantage over a general purpose agent like Claude Code: it integrates
  directly with tools that have no native MCP.
- **Correct and evidence-backed diagnosis.** Every verdict quoted primary
  evidence; no confabulation, even with a broken integration.
- **First-proposed fixes worked**, 3/3 in verification.
- Read-only access, respects RBAC. Every integration guide ships RBAC setup.
- Investigates alerts directly from alerting systems.
- Custom runbook support, letting you encode your own debugging procedures.
- Operator mode (in development) will allow continuous autonomous monitoring.
- Performs reasonably well on a weak model like `gpt-5.4-nano` for simple k8s
  tasks.

## 9. Cons

- **No alert correlation.** One incident is billed and reported as N alerts.
- **Cost telemetry under-reports** by roughly a third.
- Still a CNCF sandbox project.
- Never modifies your infrastructure. A safety feature, but it stops one step
  short of fixing the issue, where a general purpose agent like Codex would go
  ahead and fix it unprompted.
- Installation options are limited, and incomplete in different ways. No OTEL on
  the CLI path.
- Setup documentation lacks detail. The Python 3.14 fix lives in the
  contribution docs.

## 10. Cost analysis

### 10.1 Interactive mode

One medium K8s problem in the labs:

| Metric | Tokens |
|---|---:|
| Input | ~1.01M |
| Output | ~10K |
| **Total** | **~1.1M** |

Worth
noting this is a whole chat session piling up in context, not a
per-investigation figure.

### 10.2 Alert-driven path

Priced per alert. Figures recomputed by summing `total_cost`, `prompt_tokens`,
`completion_tokens` and `num_llm_calls` across every investigation in the run
JSON files.

| Run | Scenario | Alerts | Correct | Cost | Input tok | Output tok | LLM calls |
|---|---|---:|---|---:|---:|---:|---:|
| `run1` | Setup A, `Shop.*` | 3 | 3/3 | $0.368 | 257,215 | 5,862 | 17 |
| `run2` | Setup A, `Kube.*` | 10 | 10/10 | $1.105 | 713,219 | 17,992 | 50 |
| `run3a`+`run3b` | Broken integration | 2 | 2/2 honest | $0.164 | 102,982 | 2,671 | 9 |
| `run4` | Boutique | 6 | 6/6 | $0.656 | 397,073 | 9,060 | 26 |
| **Total** | | **21** | **21/21** | **$2.29** | **1,470,489** | **35,585** | **102** |

Per investigation: **$0.08–0.14**, 38K–106K input tokens, 3–7 LLM calls,
55–81% cache hit.

### 10.3 Cost accounting accuracy

Holmes under-reports its own spend. The $2.29 above is its internal accounting.
The OpenRouter meter read **$3.08** for the same session:

| | |
|---|---|
| Holmes reports | $2.29 |
| Actually billed | $3.08 |
| Gap | ~34% (a 26% shortfall in its own number) |

Two known causes: cache-write pricing isn't modelled, so all cached tokens are
priced at the read rate, and aborted attempts bill without appearing in any JSON
report.

## 11. Verdict

**Defer.** Currently the tool in in development and there are rough edges. Very limited installation methods and all the installation methods are not complete (as in OTEL support is not there for CLI installation)

The most promising feature of holmes GPT is the operator mode. It allows it to run in background 24/7, notices if something is wrong, trigger an investigation and propose fixes. This feature is still in alpha and needs to be revisited.

What's holding it back:

1. **It doesn't correlate alerts.** One outage comes back as N separate
   investigations, so plan your budget per alert rather than per incident, and
   expect to read N reports for one problem.
2. **Its own cost numbers run about a third low.** Check your provider's bill
   instead. Holmes' figures are fine for comparing one investigation against
   another, but don't plan your spend around them.
3. **Operator mode is the feature this decision really rests on, and we haven't
   tested it.** Everything here was a human kicking off an investigation and
   reading the result. The real question for production is what happens when it
   runs on a schedule and nobody is watching it think.

Worth another look in 2–3 months. Three things to check then:

1. Are the installation methods feature complete yet?
2. Where has operator mode got to?
3. Do custom runbook retrieval and RBAC enforcement actually work? Both are
   documented, neither is verified.

## 12. Issues and PRs filed

| Type | Link | Status |
|---|---|---|
| Docs | [#2047](https://github.com/HolmesGPT/holmesgpt/issues/2047) | Closed |
| Enhancement/Bug | [#2077](https://github.com/HolmesGPT/holmesgpt/issues/2077) | Closed |
