# gitops-cluster-kind-prod

Cluster config for `kind-prod` - the first real upper-env cluster in this fleet.
Companion to [`idp-service-catalog`](https://github.com/jfillman/idp-service-catalog)
and [`gitops-cluster-dev`](https://github.com/jfillman/gitops-cluster-dev), following
the per-cluster repo shape `idp/docs/gitops-strategy.md` §1 establishes.

## ArgoCD bootstrap - self-contained as of 2026-08-18

Originally this repo reused a live ArgoCD instance predating it by several days
(`platform-cicd`'s own staging release pipeline, installed imperatively by that repo's
`hack/bootstrap-upper-cluster.sh`, no values file, nothing committed anywhere). That
instance was deliberately removed 2026-08-18 as part of proving this repo's own
disaster-recovery story end-to-end - "can you delete this cluster and rebuild it from
git" needs a real answer, and "reuse whatever ArgoCD happens to already be running"
isn't one. `01-argocd-platform/install.yaml` closes that gap: the same
vendored-upstream-manifest pattern `gitops-cluster-dev/01-argocd-platform` already
established, not an ArgoCD `Application` object (see that directory's own README for
why not - the same bootstrap-ordering risk applies here). Pinned to
`v3.5.0`/chart `10.3.2` specifically - matching what was actually running here before
the rebuild (deliberately newer than `gitops-cluster-dev`'s own `v3.4.5` pin).

## argocd-platform / argocd-apps split - added 2026-08-18

Two instances now, mirroring `kind-dev`: `01-argocd-platform/argocd-apps-install/
application.yaml` installs a second `argo-cd` Helm release (chart `10.3.2`, matching
this cluster's own platform instance exactly - both share the CRDs
`01-argocd-platform/install.yaml` installs, `crds.install: false` on the second
release) into its own `argocd-apps` namespace. `02-argocd-apps/`'s three
ApplicationSets/AppProjects (`tenant-appprojects`, `tenant-onboarding`, `xr-requests`)
now target `namespace: argocd-apps` throughout - same mechanism `gitops-cluster-dev`'s
own split uses (the platform instance's root `Application` is what writes these
objects, via its own recurse-glob; only `argocd-apps`'s own controller, scoped to its
own release namespace, actually reconciles them). `argocd-apps` needs its own copy of
`argocd-repo-creds-jfillman` - doesn't carry over from the platform instance
automatically, same gotcha `gitops-cluster-dev`'s own split hit.

**Real, non-hypothetical risk hit doing this migration, not present in `kind-dev`'s own
precedent**: `kind-dev`'s split happened before any tenant had been onboarded ("zero
live generated children existed under the old namespace before this move"). This
cluster's split happened with `checkout-api` and three leftover test scenarios already
live under the single-instance setup, so moving the ApplicationSets' target namespace
left duplicate same-named `Application` objects (old copies in `argocd`, new in
`argocd-apps`) both carrying the `resources-finalizer.argocd.argoproj.io` cascade-delete
finalizer. Naive deletion of the old copies would have cascaded into deleting the real
resources the new copies now also manage. Fixed by stripping the finalizer and deleting
immediately in the same command (finalizer patch alone doesn't stick - ArgoCD's own
reconcile loop re-adds it within seconds if there's any delay before the delete lands),
verified live after every single deletion that the real resource it used to own was
still present. One real transient side-effect from a race during this cleanup: the
still-alive old `xr-requests`/`tenant-onboarding` ApplicationSets briefly recreated a
couple of already-deleted children in `argocd` before their own deletion landed, which
the new `argocd-apps` copies self-healed (SecretStore XRs, `AppProject`, and the
underlying `InfisicalProject`/`InfisicalEnvironment` all show a fresh `AGE` from this
churn, not the original provisioning time) - functionally fine, but worth knowing the
real Infisical backend may carry a short-lived duplicate project/environment entry from
the moment of the recreate, harmless but not automatically cleaned up here.

**Bootstrap steps, in order** (same shape as `gitops-cluster-dev`'s own):

```
kind create cluster --name prod --config hack/kind-config.yaml
# install Calico (matching kind-dev's pinned v3.29.1) before anything else touches the cluster
kubectl create namespace argocd
kubectl apply --server-side -n argocd -f 01-argocd-platform/install.yaml
# restore argocd-repo-creds-jfillman (see "Credentials" below) before the next step -
# root's own repo is private
kubectl apply -f root-app-of-apps.yaml
```

`--server-side`, not plain `apply` - same `applicationsets.argoproj.io` CRD-too-large-
for-`last-applied-configuration` failure `gitops-cluster-dev`'s own README documents,
hit here too rather than assumed to carry over.

**kindnet → Calico, also fixed by this rebuild.** kind-prod ran kind's default
`kindnet` CNI until 2026-08-18, which silently ignores every `NetworkPolicy` in this
repo (same gap `gitops-cluster-dev` closed 2026-08-13, `idp/README.md`'s Phase 1 entry
- confirmed live there that kindnet drops NetworkPolicy on the floor rather than
rejecting it, so the failure mode is silent). kind-prod now gets the identical
treatment: `disableDefaultCNI: true` at cluster-create time, Calico installed before
anything else, pinned to the exact version already proven on `kind-dev` (`v3.29.1`)
rather than "latest" - matching this platform's existing convention of pinning every
version it can (Tekton/PaC/ArgoCD chart versions all have the same rationale, see
`platform-cicd/hack/bootstrap.sh`'s own comments).

**Known fragile point, re-check after every from-scratch rebuild, not just this one**:
`10-crds-operators/crossplane/provider-kubernetes-config.yaml`'s `ClusterRoleBinding`
hardcodes a Crossplane-generated `ServiceAccount` name
(`provider-kubernetes-f6665ef36536`) that that file's own header already flags as "a
fact to re-check, not an assumption to carry forward blindly." A fresh Crossplane +
`provider-kubernetes` install may generate a different revision-hash suffix, silently
breaking this binding (the provider's pods would lack permissions to manage
`ClusterSecretStore` objects) until someone notices and re-pins it against
`kubectl get sa -n crossplane-system` on the rebuilt cluster.

## Deliberately scoped, not a full mirror of gitops-cluster-dev

Only what Bootstrap-tier's centralization design
(`idp/docs/service-catalog-design.md` §0 "Where Crossplane runs across a
multi-cluster fleet") actually requires here:

- `10-crds-operators/crossplane/` - Crossplane core + `provider-kubernetes` only
  (**no** `provider-github` - that stays `kind-dev`-only, Bootstrap-tier XRDs never
  run here) + `function-go-templating`/`function-auto-ready` (**no**
  `function-rollout-watcher` - AI-triage isn't built for this deployment model yet).
- `20-service-catalog/idp-service-catalog/application.yaml` - explicitly scoped to
  `xrds/slo.yaml`+`compositions/slo/composition.yaml` only, not a wildcard.
  `NodeJSApplication`/`ApplicationEnvironment` are deliberately excluded - they stay
  centralized on `kind-dev` permanently, regardless of fleet size (pure
  `provider-github` writers, no locality requirement at all).
- `02-argocd-apps/` - `tenant-appprojects`/`tenant-onboarding`, same mechanism as
  `kind-dev`'s own, pointed at `gitops-cluster-kind-prod-tenants` and hardcoding
  `cluster: kind-prod` where `kind-dev`'s own hardcodes `kind-dev`.
- `10-crds-operators/external-secrets/` - added 2026-08-17, same default install as
  `gitops-cluster-dev`'s own (chart `2.8.0`, controller only - no ClusterSecretStore/
  ExternalSecret here, those are per-app). Not part of the original "deliberately
  scoped" cut above - added after a real ArgoCD sync failure on `checkout-api-prod`
  (`failed to discover server resources for group version external-secrets.io/v1`)
  once that app actually started using `idp-application`'s `registryCredentials`/
  `secrets:` mechanisms, both of which render an `ExternalSecret`.

**`platform-secrets` namespace - real but currently inert, confirmed live 2026-08-18.**
`checkout-api-prod`'s own `values.yaml` (`gitops-checkout-api/kind-prod/prod/
values.yaml`) sets `registryCredentials.enabled: true`, which makes `idp-application`
render an `ExternalSecret` named `registry-credentials` sourcing from
`platform-secret-store` (`ClusterSecretStore`, `kubernetes` provider) →
`platform-secrets` namespace. As of this date, though: zero `ExternalSecret` objects
exist anywhere on this cluster and `platform-secrets` holds zero real `Secret`s - the
mechanism has never actually been exercised. The `ghcr-pull-secret` currently live in
`app-checkout-api-prod` is a separate, manually-applied `Secret` (different name,
no ESO involvement at all), not what this pathway produces. Net: don't remove
`platform-secrets`/`platform-secret-store` - they're load-bearing for the intended
design the moment checkout-api's real workload actually deploys - but don't expect
them to be doing anything today either. The "manual by design" step (`kubectl create
secret` with the real GHCR credential, in `platform-secrets`) has never been done here.

**Not installed here, deliberately**: `kube-prometheus-stack` - the cluster registry's
`crossplaneReady` flag
(`gitops-cluster-dev/00-bootstrap/cluster-registry/kind-prod.yaml`) means "Crossplane
+ the Attached-tier catalog can reconcile," not "the full observability stack exists."
Sloth itself **was** in this category too until 2026-08-18 (`10-crds-operators/sloth/`)
- narrows the gap (a real `SLO` XR now gets its `PrometheusServiceLevel` turned into a
real `PrometheusRule`) but doesn't close it: nothing on this cluster evaluates that
`PrometheusRule` yet without Prometheus. Argo Rollouts (`10-crds-operators/
argo-rollouts/`) and Contour (`10-crds-operators/contour/`, this cluster's actual
Ingress controller - **not** used for Rollout traffic management, `idp-application`'s
`rollout.yaml` carries no `trafficRouting` stanza, canary here is native
replica-weighting) were added the same day, closing real gaps rather than deliberate
scoping: neither had ever been installed here at all.

## Credentials (not GitOps-tracked, deliberately)

`argocd-repo-creds-jfillman` - a `repo-creds` `Secret` in `argocd`
(url-prefix `https://github.com/jfillman`), reusing `provider-github`'s own
already-cluster-resident PAT from `kind-dev` (piped cluster-Secret-to-cluster-Secret
during setup, never printed to chat or committed anywhere). Needed because ArgoCD
has no git credentials for any of `jfillman`'s **private** repos by default, and
every app's own `gitops-<app-name>` repo is private - confirmed live the same way
`kind-dev`'s own equivalent Secret was (see `gitops-cluster-dev/README.md`). Each
ArgoCD instance needs its own copy; this doesn't carry over from `kind-dev`
automatically.

**Real Secrets on this cluster, none GitOps-tracked, none ever printed/committed**
(current list as of the 2026-08-18 rebuild - back these up before any future
deliberate teardown, same as this one):
`argocd/argocd-repo-creds-jfillman`, `app-checkout-api-prod/ghcr-pull-secret`,
`app-checkout-api-prod/platform-outcome-relay-token`,
`app-checkout-api-xrs/checkout-api-kind-prod-infisical-creds`,
`app-cicd-flow-test-app-staging/platform-outcome-relay-token` (test-app scaffolding,
see "Status" below - lower value but cheap to keep),
`infisical/infisical-bootstrap-secret`. Everything else on this cluster (ArgoCD's own
admin/redis secrets, every component's TLS webhook certs) auto-regenerates on
reinstall and was deliberately not backed up.

## Status

Bootstrapped 2026-08-15 and **live-verified end-to-end the same day**: both the
registry-gate rejection path (`crossplaneReady: "false"` → zero resources created)
and the full success path (real commits, this cluster's own ArgoCD picking up the
new tenant unprompted, a real namespace/`ServiceAccount`/`NetworkPolicy`) proven with
a throwaway app, fully torn down after - see `idp-service-catalog`'s own README and
`idp/docs/service-catalog-design.md` §0 for the full design and
[[idp_session_applicationenvironment_xrd]]/[[idp_session_multicluster_architecture]]
memory for the session detail. One real bug found live: the cluster's shared
`app.yaml` needed `managementPolicies` excluding `"Delete"`, not
`spec.deletionPolicy: Orphan` as originally planned - `provider-upjet-github`
v0.19.1's `RepositoryFile` CRD has no such field.

**2026-08-18: full delete-and-rebuild-from-git, real not theoretical.** Triggered by
adding ArgoCD health checks for Crossplane XRs (see `gitops-cluster-dev`'s own
`01-argocd-platform/README.md`) surfacing that this cluster's ArgoCD was never
GitOps-managed at all - reused live state, not reproducible from this repo alone.
Fixed by vendoring `01-argocd-platform/install.yaml` (see above) rather than just
patching the old imperative instance, then proving it: `checkout-api-prod` had no live
`Rollout`/`Deployment` at deletion time (`WorkloadDeployed=False/NoImageYet` on its
`ApplicationEnvironment` XR the whole time - a scaffold, not a running service, so
"rebuild and it reinstalls" mostly means "the scaffold reappears," not zero-downtime
recovery of live traffic). Real secrets backed up first (see "Credentials" above);
`deadlock-repro`/`env-pattern-verify`/`usage-guard-verify` XR-request Applications
(leftover test scaffolding from past investigation sessions, not real tenants) were
allowed to go with the old cluster rather than migrated forward.
