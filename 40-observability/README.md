# 40-observability

Prometheus/Grafana/Tempo/Loki stack — per `gitops-strategy.md` §3.

## Real, built, live-verified on `kind-prod` (2026-08-18)

Direct copy of `gitops-cluster-dev/40-observability/`'s own build — same six
Applications (`minio`, `kube-prometheus-stack`, `thanos`, `loki`, `tempo`,
`otel-collector`), same chart versions, same `valuesObject`s throughout. The one real
per-cluster edit is `minio-secrets/secret-application.yaml`'s own `repoURL` (has to
point at this repo, not `gitops-cluster-dev`). See that repo's own
`40-observability/README.md` for the full architecture rationale, the two real
`minio/minio` chart bugs it documents (missing `buckets:` field name,
`create-buckets-job.yaml`'s workaround for the chart's own broken bucket-creation
Job), and the chart-provenance note on why `charts.min.io`, not `bitnami/minio`. None
of that re-derived here — this is a faithful mirror, not an independent build.

**Sync order matters** (`SYNCPOLICY: Manual` throughout, same as `10-crds-operators/`
— no automated dependency ordering, sync by hand in this order): `minio` +
`minio-secrets` (bucket + `thanos-objstore-config` Secret) → `kube-prometheus-stack`
(Thanos sidecar needs the Secret) → `thanos` (Query needs
`kube-prometheus-stack-thanos-discovery`) → `loki` + `tempo` (need their MinIO
buckets) → `otel-collector`. Followed on this cluster by a re-sync of
`10-crds-operators/sloth/` (its `metrics.enabled: false` override, added when Sloth
was installed a few hours earlier the same day with no Prometheus Operator CRDs yet
present, was reverted once this stack landed).

**One real gotcha hit standing this up that `gitops-cluster-dev`'s own build never
needed**: `minio-secrets` (a directory-source `Application` whose manifests aren't
part of `minio`'s own Helm chart) requires `**/secret-application.yaml` in
`root-app-of-apps.yaml`'s own include glob — this repo's root never needed that
pattern before `minio/` existed. Adding the pattern to the file and pushing to git is
**not** enough on its own: `root-app-of-apps.yaml`'s own `metadata`/`spec` live
directly on the bootstrap `Application` object (nothing manages root's own spec from
git the way root manages everything else), so a plain `argocd.argoproj.io/refresh:
hard` annotation only busts root's cached view of what its *source path* renders to —
it does **not** pick up a change to root's own `spec.source.directory.include` field.
That needs a direct `kubectl apply -f root-app-of-apps.yaml` against the live cluster,
confirmed live: the hard-refresh alone left `minio-secrets` undiscovered indefinitely,
the follow-up `kubectl apply` picked it up within seconds.

**Result: zero pod restarts across the whole stack** — every pod came up clean on
first sync, unlike `kind-dev`'s own build (which crash-looped `tempo-0`/
`thanos-storegateway-0` on "bucket does not exist" before its buckets existed). Real
difference in execution order, not luck: `minio` + `minio-secrets` (which runs
`create-buckets-job.yaml`) were synced and confirmed to have actually created all
three buckets (`mc ls` output in the Job's own logs) *before* `loki`/`tempo` were
synced at all.
