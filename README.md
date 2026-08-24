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
clusters/
└── <env>/<cluster>/          what THIS cluster's Argo CD reconciles
applicationsets/
└── base/<env>/               one directory per environment
projects/
└── base/business.yaml        AppProject: allowed destinations and kinds
apps/
└── <app>/
    ├── base/                 Kustomize base, environment-neutral
    └── overlays/
        ├── dev/              namespace, image tag, replicas, limits
        ├── stage/
        └── prod/
```

Deliberately the same shape as `gitops-flux`: environment-neutral definitions
under `base/`, and one entrypoint directory per cluster saying what that cluster
does with them.

**Every cluster runs its own Argo CD** (ADR 0014), installed by that cluster's
Flux (ADR 0013). There is no hub, and no Argo CD holds credentials for a cluster
other than its own — which is why `AppProject.spec.destinations[].server` is
pinned to the in-cluster address permanently, rather than being a single-cluster
shortcut to generalise later.

### Adding a cluster

A new directory under `clusters/`, listing the environments that cluster serves.
Nothing in `base/` or `apps/` changes. A production cluster listing only
`applicationsets/base/prod` is *incapable* of generating a dev Application —
not because policy forbids it, but because the generator is not installed there.

Today there is one cluster, and `dev`, `stage` and `prod` are namespaces within
it (ADR 0002). An overlay changes the namespace, replica counts, image tag and
resource limits — never the cluster target.

## Promotion

```
dev  →  stage  →  prod
```

Promotion is a pull request that changes the image tag in the next overlay. It is
never a change to `base`, and never an Argo UI action — a UI sync that is not in
git will be reverted at the next reconcile, silently.

Prod syncs require manual approval. Dev and stage auto-sync.

## Images

- Multi-arch (`linux/amd64` and `linux/arm64`). Graviton is the intent; today the
  node group is `t3.medium`/amd64 and Karpenter has no NodePools, so the arm64
  half is built and unused. Building both is what makes that switch a non-event
  rather than a rebuild.
- Immutable tags. Never `:latest` — a rollback needs a tag to roll back *to*.
- ECR only. The Kyverno registry allowlist enforces it.

## AppProject scoping

The `business` AppProject restricts destinations to `app-*` namespaces. An
Application that could deploy into `platform-istio` is a way to take down the
mesh from a product pull request.

## Standards

[Engineering Handbook](https://github.com/Ubuntu-25c-market-prep/ops-program/blob/main/docs/engineering-handbook.md) ·
[ADR 0002 — single cluster](https://github.com/Ubuntu-25c-market-prep/ops-program/blob/main/docs/adr/0002-single-cluster.md) ·
[ADR 0013 — the Flux/Argo boundary](https://github.com/Ubuntu-25c-market-prep/ops-program/blob/main/docs/adr/0013-flux-argo-boundary.md) ·
[ADR 0014 — one Argo CD per cluster](https://github.com/Ubuntu-25c-market-prep/ops-program/blob/main/docs/adr/0014-per-cluster-argo-cd.md)
