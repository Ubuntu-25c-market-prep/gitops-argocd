# gitops-argocd

Argo CD and Argo Workflows: delivery for **business** applications. Platform
add-ons are Flux's job and live in `gitops-flux`.

**Owner:** `@argocd` · **Wave:** 6

## The boundary

Flux owns the platform; Argo owns what runs on it. The split exists so that
application releases and platform upgrades do not block each other, and so the
people reviewing a product deploy are not the people reviewing a mesh upgrade.

**A resource has exactly one controller.** If Argo reconciles it, Flux must not.
Overlap is not redundancy — it is two controllers reverting each other.

## Layout

```
apps/
├── <app>/
│   ├── base/                 Kustomize base
│   └── overlays/
│       ├── dev/
│       ├── stage/
│       └── prod/
applicationsets/
└── business-apps.yaml        generates Applications from apps/
projects/
└── business.yaml             AppProject: RBAC and allowed destinations
```

Environments are **namespaces in one cluster**, not clusters. An overlay changes
the namespace, replica counts and resource limits — never the cluster target.

## Promotion

```
dev  →  stage  →  prod
```

Promotion is a pull request that changes the image tag in the next overlay. It is
never a change to `base`, and never an Argo UI action — a UI sync that is not in
git will be reverted at the next reconcile, silently.

Prod syncs require manual approval. Dev and stage auto-sync.

## Images

- Multi-arch (`linux/amd64` and `linux/arm64`). Nodes are Graviton by default;
  an amd64-only image silently fails to schedule or forces an expensive nodepool.
- Immutable tags. Never `:latest` — a rollback needs a tag to roll back *to*.
- ECR only. The Kyverno registry allowlist enforces it.

## AppProject scoping

The `business` AppProject restricts destinations to `app-*` namespaces. An
Application that could deploy into `platform-istio` is a way to take down the
mesh from a product pull request.

## Standards

[Engineering Handbook](https://github.com/Ubuntu-25c-market-prep/ops-program/blob/main/docs/engineering-handbook.md) ·
[ADR 0002 — single cluster](https://github.com/Ubuntu-25c-market-prep/ops-program/blob/main/docs/adr/0002-single-cluster.md)
