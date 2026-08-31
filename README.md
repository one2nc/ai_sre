# AI for SRE

Exploring AI tooling for SRE and platform teams. Each experiment runs a tool against real failure scenarios - what it detects, what it misses, and how much cluster access it needs etc.

## Architecture

![Architecture](assets/architecture.png)

Each experiment is an independent silo. K8sGPT and the first HolmesGPT round were run against [iximiuz Labs](https://labs.iximiuz.com) challenge playgrounds; HolmesGPT was later re-run through the Alertmanager path on a kind cluster with kube-prometheus-stack ([`holmes-gpt/alert-driven-path/`](holmes-gpt/alert-driven-path/README.md)); OpenSRE was run against the local observability sandbox in [`opensre/observability-stack/`](opensre/observability-stack/README.md), which is where alert-driven RCA needs Prometheus, Loki, and Tempo present.


## Experiments


| SNo | Experiment | Version | Date | Test environment | README |
|-------|------------|---------|------|------------------|--------|
| 1 | `holmes-gpt/` | 0.30.0, `latest` | 2026-05-18, 2026-08-18 | iximiuz Labs (K8s + Linux challenges); kind + kube-prometheus-stack | [README](holmes-gpt/README.md) |
| 2 | `k8sgpt/` | 0.4.33 | 2026-05-18 | iximiuz Labs (K8s challenge) | [README](k8sgpt/README.md) |
| 3 | `opensre/` | 0.1 | 2026-05-24 | Local `observability-stack` sandbox | [README](opensre/README.md) |


## Test environments

| Environment | Used by | What it provides |
|-------------|---------|------------------|
| [iximiuz Labs](https://labs.iximiuz.com/playgrounds) challenge playgrounds | HolmesGPT (interactive), K8sGPT | Pre-broken K8s and Linux scenarios with a known correct answer, so tool output can be scored against ground truth |
| [`opensre/observability-stack/`](opensre/observability-stack/README.md) | OpenSRE | Local Docker Desktop K8s sandbox: instrumented Go services, Grafana stack, real alerts, chaos targets |
| kind + kube-prometheus-stack | HolmesGPT (alert path) | Single-node cluster with Prometheus and Alertmanager; fault injection into a 3-service app and the 12-service Online Boutique, so real alerts drive `holmes investigate alertmanager` |

What the local sandbox exposes:

| Layer | What's observable |
|-------|------------------|
| HTTP/API | Gateway → backend request latency, status codes, RED metrics |
| Logs | Structured `log` JSON on stdout carrying `trace_id`, shipped by Promtail to Loki |
| Traces | OpenTelemetry spans from both services into Tempo; pivot from a Loki `trace_id` to the trace and back |
| Datastores | Postgres and Redis calls as child spans, with up/down alerts |
| Alerting | `kube-prometheus-stack` `PrometheusRule`s firing real `AppPostgresDown` / `AppRedisDown` / `AppBackendDown` / `AppGatewayDown` into Alertmanager and Slack |

Quick links:

| Topic | Link |
|-------|------|
| Stack setup and `make` targets | [observability-stack README](opensre/observability-stack/README.md) |
| End-to-end verification (`make verify`) | [observability-stack Makefile](opensre/observability-stack/Makefile) |
| Chaos scenarios | [opensre Makefile](opensre/Makefile) |
| Captured Alertmanager payloads | [opensre/tests/](opensre/tests/) |

## Personas

| Persona | What they care about |
|---------|---------------------|
| On-call SRE | Does the agent shorten time-to-root-cause on a real incident? |
| Platform engineer | What cluster access does it need, and what can it write? |
| Security/Compliance | What cluster data leaves the perimeter, and to which provider |

## License

[MIT](LICENSE) © One2N
