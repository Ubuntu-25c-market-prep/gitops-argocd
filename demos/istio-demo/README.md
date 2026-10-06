# istio-demo

A small, removable demo of what Istio gives us on `dev-eks-us-east-1`: a live traffic graph in Kiali,
blue-green and canary releases, circuit breaking, fault injection and mutual TLS. It exists for team
presentations and onboarding, not as a product workload.

## What runs (synced by Argo CD)

Application `istio-demo-dev` (project `business`, namespace `app-dev`) deploys `app/`, the pods and Services only:

| Object | Purpose |
|---|---|
| `helloworld-v1`, `helloworld-v2` | Istio's own `helloworld` sample. `GET /hello` answers `Hello version: vN, instance: <pod>` |
| Services `helloworld`, `helloworld-v1`, `helloworld-v2` | `helloworld` selects both versions (mesh traffic, Istio rules); the per-version Services are what Gateway API weights split between |
| `fortio` | Background load, 1 request/second to `helloworld`, so Kiali always has a live graph |

Cost: 3 pods in `app-dev`, about 150m CPU / 192Mi in requests plus sidecars. They run on the base spot pool.

## Why it is not under `apps/`

The `business-apps-dev` ApplicationSet rewrites the first image of every app to our ECR registry by directory
name. This demo uses public, pinned Docker Hub images and two different helloworld images, so it has its own
Application instead.

## What is NOT synced: `scenarios/`

Every route and every Istio-specific object is applied by hand during a live demo and deleted afterwards. Argo CD
would revert any manual change to an object it owns within seconds (`selfHeal`), which would break the demo, so
nothing in `scenarios/` is referenced by the Application. Argo CD reports the applied objects as orphaned (a
warning only). Each file's header explains the step, the commands and the cleanup. Run `kubectl apply -f <file>`
against the cluster yourself.

| File | Step | API |
|---|---|---|
| `00-ingress-blue-green.yaml` | public ingress `https://helloworld.25c-team1.art/hello`, and blue-green by patching the weights | Gateway API HTTPRoute |
| `01-mesh-canary-90-10.yaml`, `02-mesh-canary-v2.yaml` | canary inside the mesh | Gateway API (GAMMA HTTPRoute on a Service) |
| `03-circuit-breaker.yaml` | circuit breaking | Istio `DestinationRule` |
| `04-fault-injection.yaml` | fault injection | Istio `VirtualService` |
| `05-mtls-strict.yaml` | mTLS STRICT | Istio `PeerAuthentication` |

Apply `00-ingress-blue-green.yaml` first: until it is applied the demo has no public URL (the fortio traffic and
the Kiali graph do not need it). Only `/hello` is routed, over https only; any other path is a 404 from the Gateway.

Routing is Gateway API only. Istio objects appear where Gateway API has no equivalent. Fault injection is the
only reason a `VirtualService` is used. Never keep the mesh HTTPRoute (steps 1-2) and a `VirtualService` or
`DestinationRule` for `helloworld` at the same time.

## Remove

Delete `- ../../../demos/istio-demo` from `clusters/dev/dev-eks-us-east-1/kustomization.yaml`. The `bootstrap`
Application prunes `istio-demo-dev`, and its finalizer deletes everything in `app/`. Then remove everything
applied by hand from `scenarios/`:

```bash
kubectl -n app-dev delete httproute helloworld --ignore-not-found
kubectl -n app-dev delete httproute helloworld-mesh --ignore-not-found
kubectl -n app-dev delete destinationrule helloworld --ignore-not-found
kubectl -n app-dev delete virtualservice helloworld --ignore-not-found
kubectl -n app-dev delete peerauthentication helloworld-strict --ignore-not-found
```
