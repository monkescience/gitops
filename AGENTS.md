# Agents Overview

This repository is a GitOps source of truth for the eu-central-1 dev environment. It defines a set of Kubernetes "agents" (operators, controllers, and system components) that continuously reconcile desired state declared here with the cluster's actual state.

The term "agent" below refers to any controller/operator or component that:
- Runs a reconciliation loop (polling or event-driven) to converge toward desired state
- Is managed via Argo CD Application manifests under `apps/eu-central-1-dev/`
- May have extended configuration or dashboards under `manifests/*/eu-central-1-dev/`

## Core GitOps Flow
1. You commit declarative YAML/Helm values here.
2. Argo CD Applications (defined in `apps/eu-central-1-dev/`) point either to external Helm chart repositories or to local `manifests/` overlays.
3. Argo CD reconciles on a schedule (and via webhook triggers) applying diffs to the cluster.
4. Individual agents (operators) then further reconcile their own CRDs.

## Argo CD
- Kind: Application CR controller
- Responsibility: Deploy and keep all other agents/apps in sync.
- Extensions: Dashboards & alerting rules under `manifests/argocd-extension/`.
- Sync waves: Controls ordering (lower waves apply first). Example: Istio base (`-1`) before Istio control plane.

## Service Mesh (Istio Ambient Components)
| Component | File | Purpose | Notes |
|-----------|------|---------|-------|
| base | `apps/eu-central-1-dev/istio.yaml` (first section) | CRDs and cluster-scoped resources | sync-wave `-1` ensures CRDs exist first |
| istiod | same file | Control plane / discovery | requests/limits tuned small for dev |
| cni | same file | Sidecar-injection alternative / ambient infra | profile ambient |
| ztunnel | same file | Data plane (ambient) | wave `1` after control plane |

Ignore differences: We selectively ignore webhook `failurePolicy` drift to prevent constant re-sync noise.

## Security & Identity
| Agent | Path | Description |
|-------|------|-------------|
| cert-manager | `apps/eu-central-1-dev/cert-manager.yaml` + `manifests/cert-manager/` | Issues TLS certs via configured issuers |
| hydra | `apps/eu-central-1-dev/hydra.yaml` + `manifests/hydra/` | OAuth2 / OIDC provider (values & gateway config) |

## Observability Stack
| Agent | Path | Description |
|-------|------|-------------|
| kube-prometheus-stack | `apps/eu-central-1-dev/kube-prometheus-stack.yaml` + `manifests/kube-prometheus-stack/` | Prometheus, Alertmanager, node exporters, etc. | Extended mixins & dashboards under `manifests/kube-prometheus-stack-extension/` |
| grafana | `apps/eu-central-1-dev/grafana.yaml` + `manifests/grafana-extension/` | Visualization layer; dashboards from mixins |
| loki | `apps/eu-central-1-dev/loki.yaml` + values under `manifests/loki/` | Log aggregation |
| jaeger | `apps/eu-central-1-dev/jaeger.yaml` + `manifests/jaeger-operator/` | Tracing backend |
| kiali | `apps/eu-central-1-dev/kiali.yaml` + `manifests/kiali-operator/` | Istio mesh observability |
| metrics-server | `apps/eu-central-1-dev/metrics-server.yaml` | Resource metrics API |
| alloy | `apps/eu-central-1-dev/alloy.yaml` + `manifests/alloy-extension/` | OTEL style telemetry collection / pipelines |

## Namespacing & Resource Ownership
- Namespaces: Managed declaratively via `manifests/namespaces/eu-central-1-dev/manifest.yaml`.
- Each operator generally owns CRDs in its chart; avoid editing generated CRDs manually.
- Custom dashboards and rules live in extension folders to keep vendor chart values minimal and separated from bespoke logic.

## Sync Waves Convention
Use `argocd.argoproj.io/sync-wave` annotation to order applications:
- `-1` CRDs / foundational infra
- `0` Control planes / core services
- `1` Data planes / dependent components
- Higher numbers for optional/edge components

Keep waves sparse; only introduce ordering where real dependency exists.

## Drift & Ignore Configuration
If an agent frequently mutates a field (e.g., webhook `failurePolicy`) that is safe to ignore, add an entry under `spec.ignoreDifferences` in its Application. Only ignore fields that are truly harmless drift to preserve Git as source of truth.

## Adding a New Agent
1. Choose chart source: external Helm repo or local Kustomize overlay.
2. Create `apps/eu-central-1-dev/<name>.yaml` Argo CD Application manifest.
3. (Optional) Create `manifests/<name>/eu-central-1-dev/values.yaml` (for Helm values) or a Kustomize base/overlay.
4. Assign a sync wave if ordering matters.
5. Keep resource requests modest for dev; set limits only where necessary.
6. Add dashboards/alerts under `<name>-extension` if needed instead of inflating chart values.
7. Commit & push; Argo CD will reconcile automatically.

## Resource Tuning Guidelines (Dev)
- Prefer small CPU request (10m–100m) and memory request (32Mi–256Mi) for control plane components.
- Avoid premature memory limits that may cause OOM unless well understood.
- Review Prometheus retention and scrape intervals—shorten for dev if cost-sensitive.

## Troubleshooting Cheat Sheet
| Symptom | Likely Agent | Action |
|---------|--------------|--------|
| Application OutOfSync | Argo CD | Check diff; ensure repoURL & targetRevision valid |
| Mesh pods not injected / ambient not working | Istio (cni/ztunnel) | Verify CRDs applied (base) then check daemonset logs |
| TLS issuance failing | cert-manager | Inspect Issuer/ClusterIssuer conditions |
| Missing metrics | Prometheus stack | Check ServiceMonitor / scrape configs |
| Dashboards missing | Grafana | Confirm ConfigMap sync & provisioning logs |
| Logs absent in Loki | Loki | Check distributor & ingester readiness |
| Traces missing | Jaeger / Alloy | Verify collectors running & sampling config |

## Label / Annotation Conventions
- Applications may use `argocd.argoproj.io/sync-wave` for ordering only.
- Prefer standard Kubernetes labels: `app.kubernetes.io/name`, `app.kubernetes.io/component`, `app.kubernetes.io/part-of` when extending manifests.

---
_Last updated: 2025-11-07_

