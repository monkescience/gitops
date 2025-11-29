# gitops

This repository contains the GitOps source of truth for the eu-central-1 dev Kubernetes environment. It defines Argo CD Applications and supporting manifests that manage core platform agents (operators/controllers) and workloads.

Quick links:
- See AGENTS.md for a short tour of the managed agents and conventions.
- apps/eu-central-1-dev/: Argo CD Application CRs for each component (Helm/Kustomize sources).
- manifests/*/eu-central-1-dev/: Values, overlays, dashboards, and extensions kept separate from vendor charts.
