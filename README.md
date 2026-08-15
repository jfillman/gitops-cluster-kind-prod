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

## Status

Bootstrapped 2026-08-15, live-verified alongside `idp-service-catalog`
`ApplicationEnvironment`'s new `spec.cluster` field/registry gate - see
`idp-service-catalog`'s own README and `idp/docs/service-catalog-design.md` §0 for
the full design.
