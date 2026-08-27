# backstage-ingestor-rbac

Read-only RBAC on this cluster (`kind-prod`) for the `kubernetes-ingestor` Backstage
plugin running on `kind-man` (`idp/docs/backstage-design.md`, "Catalog ingestion" -
Phase 2). Identical rationale/shape to `gitops-cluster-dev`'s own
`00-bootstrap/backstage-ingestor-rbac/README.md` - only the target key names below
differ.

## One-time manual step (do this yourself, not through an assistant)

1. Once this Application has synced, retrieve the real token value:
   ```
   kubectl --context kind-prod -n backstage-system get secret backstage-ingestor-token -o jsonpath='{.data.token}' | base64 --decode
   ```
2. Plant it into Infisical's `platform-cicd-kind-man` project under the key:
   ```
   backstage-kind-prod-sa-token
   ```
3. `gitops-cluster-kind-man/60-backstage/backstage/kind-prod-token-external-secret.yaml`
   pulls that key into a K8s Secret on `kind-man`, which `deployment.yaml` mounts as
   `KIND_PROD_SA_TOKEN` - `app-config.yaml`'s `kubernetes.clusterLocatorMethods` reads
   it via `${KIND_PROD_SA_TOKEN}` substitution.

If this cluster is ever rebuilt, repeat step 1-2 with the new value.
