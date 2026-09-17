# apps/

One directory per business application. This directory is empty by design —
the `storefront` sample was removed in #16 ahead of the `25c-shared`
decommission.

## What the ApplicationSet expects

The generator in `applicationsets/base/dev` is a git **files** generator over:

```
apps/*/overlays/dev/kustomization.yaml
```

So an application appears in Argo CD when that file exists, and for no other
reason. There is no index to edit and no ApplicationSet change — adding a
directory here is the whole registration step.

```
apps/<app>/
├── base/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── kustomization.yaml
└── overlays/
    ├── dev/kustomization.yaml       namespace + newTag (+ any route)
    ├── stage/kustomization.yaml
    └── prod/kustomization.yaml
```

The directory name becomes the Application name and the image name, via
`{{index .path.segments 1}}`. Name it after the ECR repository.

## Two rules the overlays have to follow

**Set `newTag` only, never `newName`.** The registry host contains the AWS
account id and this repository is public, so it is supplied at sync time from an
annotation on the cluster Secret. `kustomize build` here therefore renders
`<app>:<tag>` with no registry — intentionally not deployable, because a
placeholder that looks real is worse than one that does not.

⚠️ If you ever override the image name yourself, set name **and** tag together.
`kustomize edit set image` replaces the entry rather than merging with it, so a
registry-only override silently drops the tag.

**Prod gets no `syncPolicy.automated`.** That, the RBAC policy that grants sync
on `*-dev` and `*-stage` but never `*-prod`, and CODEOWNERS on the prod overlay
are three independent controls. Any one alone is a convention.

See [`docs/promotion.md`](../docs/promotion.md) for how a tag moves between
environments.
