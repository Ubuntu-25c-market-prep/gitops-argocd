# Promotion

How a build reaches production.

```
dev  →  stage  →  prod
```

Each arrow is a pull request that changes **one line**: the image tag in the next
environment's overlay. That is the whole mechanism. Everything below is either a
consequence of it or a rule that protects it.

## The one line

```yaml
# apps/storefront/overlays/stage/kustomization.yaml
images:
  - name: storefront
    newName: <registry>/25c-project/storefront
    newTag: "1.4.2"        # ← this
```

## Promoting, step by step

**1 · Check what is actually running in the previous environment.**

```bash
kubectl -n app-dev get deploy storefront -o jsonpath='{.spec.template.spec.containers[0].image}'
```

Read the tag from the *cluster*, not from the previous overlay file. They should
match; if they do not, something is wrong and promoting would carry the wrong
build forward.

**2 · Open a pull request changing `newTag` in the next overlay.** Nothing else.

```
git checkout -b feat/<issue>-promote-storefront-1.4.2
# edit apps/storefront/overlays/stage/kustomization.yaml — the newTag line only
```

**3 · Render it before you push.** This catches a typo that would otherwise
become a Pending pod.

```bash
kustomize build apps/storefront/overlays/stage | grep image:
```

**4 · Merge.** dev and stage sync automatically. **Prod does not** — see below.

## Rules, and why each exists

**Never edit `base/` to promote.** `base` is environment-neutral. A tag in `base`
would move every environment at once, which is the precise opposite of a promotion
path.

**Never sync from the Argo CD UI.** A UI sync that is not in git is reverted at
the next reconcile — *silently*, with no error and no event you would think to
look for. The UI is for reading.

**The tag must already be running in the previous environment.** Promotion moves a
build that has been observed working. Skipping stage is not a faster promotion, it
is an untested deploy.

**Tags are immutable and never `:latest`.** ECR is configured `IMMUTABLE`, so a tag
cannot be repointed at different bytes. That is what makes a one-line change
sufficient evidence of what will run — "the tag we tested" and "the tag we
deployed" cannot drift apart. It is also what makes rollback possible: you need a
tag to roll back *to*.

**One image, promoted — never rebuilt per environment.** A rebuild produces
different bytes, so stage would not have tested what prod runs. This is why the
registry has no environment in its name.

## What an overlay may change

| May change | Must not change |
|---|---|
| `namespace` | the cluster target — there is one cluster (ADR 0002) |
| image `newTag` | anything in `base/` |
| `replicas` | application configuration that differs by environment for no reason |
| resource requests and limits | |

If an overlay needs to change something not in the left column, that is usually a
sign the difference belongs in `base` or should not exist.

## Why prod is different

Prod has **three independent controls**, deliberately. Any one alone would be a
convention; three make it a control.

1. **No auto-sync.** The prod ApplicationSet omits `syncPolicy.automated`
   (`gitops-argocd#2`), so a merge to `main` does not deploy. Someone syncs it.
2. **Argo RBAC.** The policy grants `sync` on `*-dev` and `*-stage` and never on
   `*-prod` (`gitops-argocd#1`). Most people cannot press the button at all.
3. **CODEOWNERS.** `/apps/*/overlays/prod/` requires `@cto` review, so the pull
   request cannot merge without it.

The order matters: 3 stops the change landing, 1 stops it deploying itself, 2 stops
the wrong person deploying it.

## Rolling back

A rollback is a promotion in reverse, and uses the same mechanism:

```yaml
newTag: "1.4.1"        # the previous known-good tag
```

Open it as a pull request like any other. In a real incident, ask for review in
whatever channel is fastest rather than bypassing the process — a rollback merged
without review is how the second incident starts.

Do **not** use `kubectl rollout undo`. It changes the cluster without changing git,
so Argo CD reverts it at the next reconcile and you are back to the broken version
with no record of why.

## Current constraints

Honest limits as of 2026-08-16, not permanent properties:

- **Prod runs 2 replicas, not 3.** The cluster has 3 free pod slots in total
  (`gitops-flux#94` has the arithmetic). Raise it once Karpenter has a NodePool.
- **The registry host is a placeholder.** `ECR_REGISTRY_PENDING_DECISION` — the
  real value embeds the AWS account id and this repository is public. Pending a
  decision from `@cto` / `@security`.
- **Nothing is deployed yet.** Argo CD is not installed (`gitops-argocd#1`), which
  is itself waiting on `ops-program#48` and `gitops-flux#93`.
