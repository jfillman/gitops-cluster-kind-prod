# 50-gateway-routes

Gateway API `HTTPRoute`s exposing this cluster's UI-bearing Services through the
kiac-installed Gateway (`kiac`, ns `kiac-gateway`, `GatewayClass traefik` →
`traefik.io/gateway-controller`), reached via `kiac-prod`'s control-plane VM IP
(`192.168.64.6`) on port 80.

Each `HTTPRoute` lives in the same namespace as its backend `Service` — Gateway API
requires that unless a `ReferenceGrant` explicitly allows cross-namespace
`backendRefs`, and none of these need one. That's why `application.yaml` is a
multi-namespace directory source rather than one fixed `destination.namespace`.

| Hostname | Service | Namespace |
|---|---|---|
| `argocd.prod.kiac.local` | `argocd-server` | `argocd` |
| `argocd-apps.prod.kiac.local` | `argocd-apps-server` | `argocd-apps` |
| `grafana.prod.kiac.local` | `kube-prometheus-stack-grafana` | `observability` |
| `minio-console.prod.kiac.local` | `minio-console` | `observability` |

To reach these from the Mac, point them at the gateway IP, e.g. in `/etc/hosts`:

```
192.168.64.6 argocd.prod.kiac.local argocd-apps.prod.kiac.local grafana.prod.kiac.local minio-console.prod.kiac.local
```

HTTP only — the `kiac` Gateway only has an `http`/port-80 listener today, no TLS
listener. `kiac-dev`'s equivalent (larger) route set lives in `gitops-cluster-dev`'s
own `60-gateway-routes/`, using `.dev.kiac.local` hostnames.

Both ArgoCD instances' `server.insecure: "true"` is set in `01-argocd-platform/`
(`install.yaml`'s `argocd-cmd-params-cm` for the `argocd` instance,
`argocd-apps-install/application.yaml`'s `configs.params` for `argocd-apps`) —
without it, `argocd-server` redirects plain HTTP to HTTPS by default, which the
Gateway's HTTP-only listener can't follow.

Deliberately **not** routed here: `backstage-system` and `infisical`/
`tekton-pipelines` aren't deployed on this cluster at all yet. (The earlier
second, unexplained ArgoCD install in the `default` namespace has been removed.)
