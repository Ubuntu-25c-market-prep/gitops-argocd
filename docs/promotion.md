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
# apps/<app>/overlays/stage/kustomization.yaml
images:
  - name: <app>
    newTag: "1.4.2"        # ← this
```

There is no `newName`. The registry host embeds the AWS account id and this
repository is public, so it is never committed - the ApplicationSet reads
`newTag` out of this file and prepends the registry at sync time, taking it from
an annotation on the cluster Secret. `gitops-flux` keeps every IRSA role ARN out
of git the same way (`gitops-flux#97`); this is the Kustomize equivalent.

The practical consequence: **`kustomize build` on an overlay renders
`<app>:1.4.2`, with no registry.** That is not a bug and not a placeholder
to be filled in. What Argo CD applies is:

```
<account-id>.dkr.ecr.<region>.amazonaws.com/<ecr-namespace>/<app>:1.4.2
```

The real account id is deliberately **not** written here either. It arrives from
the cluster Secret annotation at sync time, and this file is not an exception to
its own rule.

To see the real rendered output locally:

```bash
cd apps/<app>/overlays/stage
kustomize edit set image <app>=$REGISTRY/<ecr-namespace>/<app>:1.4.2
kustomize build .
git checkout kustomization.yaml    # do not commit that
```

## Promoting, step by step

**1 · Check what is actually running in the previous environment.**

```bash
kubectl -n app-dev get deploy <app> -o jsonpath='{.spec.template.spec.containers[0].image}'
```

Read the tag from the *cluster*, not from the previous overlay file. They should
match; if they do not, something is wrong and promoting would carry the wrong
build forward.

**2 · Open a pull request changing `newTag` in the next overlay.** Nothing else.

```
git checkout -b feat/<issue>-promote-<app>-1.4.2
# edit apps/<app>/overlays/stage/kustomization.yaml — the newTag line only
```

**3 · Render it before you push.** This catches a typo that would otherwise
become a Pending pod.

```bash
kustomize build apps/<app>/overlays/stage | grep image:
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

Honest limits as of 2026-09-17, not permanent properties:

- **No applications are defined.** `apps/` holds only its README — the
  `storefront` sample was removed in `#16`. This document describes the
  mechanism; the first application added will be the first to use it.
- **A cluster runs only the environments its entrypoint lists.** Where that
  entrypoint lists `applicationsets/base/dev` alone, the `dev → stage → prod`
  path above is a one-step path in practice. Adding an environment is a line in
  that cluster's `clusters/<env>/<cluster>/kustomization.yaml`; nothing in
  `base/` or `apps/` changes.
- **The cluster Secret must carry the registry annotations before anything can
  sync.** The ApplicationSet joins the full image reference at sync time from
  annotations on that Secret rather than from anything in git, so a cluster
  whose Secret is missing them generates nothing — loudly, rather than
  deploying a wrong image. Created out of band, exactly like the ConfigMaps
  `gitops-flux` uses for IRSA ARNs:

  ```bash
  kubectl -n argocd annotate secret cluster-in-cluster \
    u25c.io/ecr-registry="$(aws sts get-caller-identity --query Account --output text)\
.dkr.ecr.us-east-1.amazonaws.com"
  ```

  The `<ecr-namespace>` half is moving to an annotation on the same Secret, so
  that a cluster whose registry organises repositories differently needs no
  change to any ApplicationSet.
