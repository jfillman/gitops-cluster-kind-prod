# 60-backstage

`kind-man`-only: the fleet's developer portal — upstream Backstage (not Red Hat
Developer Hub, a licensing call made explicitly, see `idp/docs/backstage-design.md`),
built as its own image and onboarded onto `platform-cicd` (`kind-dev`) as an `appType:
infra` app rather than through this template. This directory only carries what's
hand-deployed straight to `kind-man`.

- `postgres/` — Bitnami's `postgresql` chart (OCI, `oci://registry-1.docker.io/
  bitnamicharts/postgresql`, pinned `18.8.13`), standalone, real as of 2026-08-26.
  Backstage's own catalog/scaffolder/auth state lives here. `auth.existingSecret`
  works cleanly on this chart (unlike Infisical's bundled subchart, see that
  Application's own header for why that one couldn't). The credentials Secret
  (`backstage-postgres-credentials`, keys `postgres-password`/`password`) is generated
  automatically and entirely in-cluster — no manual step, never plaintext anywhere —
  via ESO's `Password` generator + `ExternalSecret` (`backstage/postgres-credentials-
  external-secret.yaml`, applied by the `backstage/` Application below since
  `postgres/`'s own Application is an OCI Helm source and can't carry an extra plain
  manifest). See that file's header for why this isn't just the Bitnami chart's own
  auto-generation (`auth.existingSecret` left unset) — ArgoCD's Helm-template render
  path can't support that safely — and for the `refreshPolicy: CreatedOnce` +
  `target.immutable: true` pairing that keeps the generated password stable across
  ArgoCD resyncs, live-verified 2026-09-03.
- `backstage/` — the app itself. Plain Deployment/Service (no officially-maintained
  Backstage Helm chart exists to adopt). Manually bump the image tag here after each
  new CI build for now - no GitOps image-updater wired up yet (see TODO backlog).
  `POSTGRES_*` env vars wire it to `postgres/`'s own instance; `imagePullSecrets:
  registry-credentials` needs the `backstage` namespace's
  `platform.io/managed-secrets: "true"` label (declared in this directory's own
  `Namespace` manifest) for kind-man's `registry-credentials` `ClusterExternalSecret`
  to populate it - the image is genuinely private.
  - Reachable at `http://backstage.man.kiac.local/` (HTTP only - no port suffix,
    no TLS listener on kiac's Gateway, see `idp/docs/local-clusters.md`). Add
    `backstage.man.kiac.local` to `/etc/hosts` -> the cluster's current
    control-plane VM IP (`container list` - it changes on every VM restart,
    `refresh-kiac-hosts.sh` automates this). Superseded 2026-09-03: the old
    `https://backstage.man.local:8453` / `http://backstage.man.local:8090` pair
    and the Contour Gateway + cert-manager TLS listener behind them no longer
    exist - `kind-man` migrated to a kiac cluster. See `backstage/httproute.yaml`.
  - Auth is real GitHub OAuth (`github-oauth-secret.yaml`), sign-in resolved against
    the one hand-authored catalog `User` (`jfillman`, in the backstage repo's own
    `examples/org.yaml` - `jfillman` is a personal GitHub account, not an org, so
    there's no automated org-membership sync). Guest auth stays registered in code
    but is hard-refused outside `NODE_ENV=development` by Backstage itself, and this
    image runs with `NODE_ENV=production` - inert here regardless of app-config.
  - Permissions are a real hand-written policy (backstage repo's
    `packages/backend/src/permissionPolicy.ts`), not the allow-all default: catalog/
    scaffolder writes are gated to `user:default/jfillman`, reads stay open.

Manual secrets to plant by hand before first sync (never committed, same convention
everywhere in this platform) — `backstage-postgres-credentials` is no longer in this
list, see `postgres/`'s own section above for why:
- `backstage-github-oauth-creds`, synced via `ExternalSecret`
  (`backstage/github-oauth-secret.yaml`) from kind-man's shared Infisical project,
  keys `backstage-github-oauth-client-id` / `backstage-github-oauth-client-secret` -
  create a dedicated GitHub OAuth App (not the fleet's other two GitHub credentials -
  see that file's own header for why) with callback URL
  `http://backstage.man.kiac.local/api/auth/github/handler/frame` (updated
  2026-09-03 for the kiac migration - HTTP, not the retired HTTPS setup) and plant
  its client id/secret into Infisical under those two keys. See that file's header
  for the exact steps.

Gated by `components.backstage` in `cluster.yaml` (see `cluster.yaml.example`) -
`kind-man` is the only cluster that should ever set this `true`, matching the singleton
pattern `components.secrets.infisicalHost`/`components.platformCicd` already use for
"exactly one cluster in the fleet runs this."
