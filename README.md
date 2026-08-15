# gitops-cluster-kind-prod

Cluster config for `kind-prod` - the first real upper-env cluster in this fleet.
Companion to [`idp-service-catalog`](https://github.com/jfillman/idp-service-catalog)
and [`gitops-cluster-dev`](https://github.com/jfillman/gitops-cluster-dev), following
the per-cluster repo shape `idp/docs/gitops-strategy.md` §1 establishes.

## Not a blank cluster

`kind-prod` already runs a live ArgoCD instance predating this repo by several days -
`platform-cicd`'s own staging release pipeline (`cicd-flow-test-app-cluster-root`/
`-staging` Applications, `project: default`). This repo's own `root-app-of-apps.yaml`
reuses that instance rather than standing up a second one (confirmed with the user,
not assumed) - same single-instance-for-now precedent already used on `kind-dev`
itself. `platform-cicd`'s existing Applications are untouched; this repo's
Applications coexist alongside them.

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

**Not installed here, deliberately**: Sloth/`kube-prometheus-stack` - a real `SLO` XR
created against this cluster would compose cleanly but sit un-reconciled by Sloth.
Acceptable for now: the cluster registry's `crossplaneReady` flag
(`gitops-cluster-dev/00-bootstrap/cluster-registry/kind-prod.yaml`) means "Crossplane
+ the Attached-tier catalog can reconcile," not "the full observability stack
exists."

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
