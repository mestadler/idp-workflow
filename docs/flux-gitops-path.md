# Flux GitOps Path (M4)

## Purpose

Define Flux OCI reconciliation as the default deployment path for v1.

## Scope

- Flux bootstrap prerequisites for single-node k0s baseline.
- OCI source and reconciliation object model.
- Declarative promotion and rollback workflow.
- Reconcile observability checkpoints.

## Bootstrap Prerequisites

- Single-node k0s baseline is healthy (M1).
- Signed app and deploy artifacts are available in OCI registry.
- Registry pull credentials are available to the cluster.
- Flux CLI is installed from `k8s-tools`.

Bootstrap and verify:

```bash
flux check --pre
flux install
flux check
```

## OCI Source and Reconciliation Model

Default model:

- `OCIRepository` points to deploy artifact repository/tag.
- `Kustomization` reconciles manifests from that OCI source.

Minimal object shape:

```yaml
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: OCIRepository
metadata:
  name: myapp-deploy
  namespace: flux-system
spec:
  interval: 1m
  url: oci://git.sansnom.uk/martin/myapp-deploy
  ref:
    tag: v1.0.0
```

```yaml
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: myapp
  namespace: flux-system
spec:
  interval: 1m
  path: ./overlays/prod
  prune: true
  sourceRef:
    kind: OCIRepository
    name: myapp-deploy
```

## Declarative Promotion Workflow

Promotion is a source ref change, not an imperative apply.

1. Publish new signed deploy artifact `<deploy-repo>:<version>`.
2. Update `OCIRepository.spec.ref.tag` to target version.
3. Commit and apply the reconciliation config update.
4. Wait for Flux reconcile success.

## Declarative Rollback Workflow

Rollback is also a source ref change.

1. Select previous known-good deploy artifact version.
2. Set `OCIRepository.spec.ref.tag` back to that version.
3. Commit and apply config update.
4. Confirm reconcile success and workload health.

No default imperative rollback path is used for normal operations.

## Observability Checkpoints

Use these checks for reconcile visibility:

```bash
flux get sources oci -n flux-system
flux get kustomizations -n flux-system
flux reconcile source oci myapp-deploy -n flux-system
flux reconcile kustomization myapp -n flux-system
kubectl get events -n flux-system --sort-by=.lastTimestamp
```

Expected outcomes:

- Source is `Ready=True`.
- Kustomization is `Ready=True` and observed revision matches target.
- Reconcile commands complete without errors.

## Operational Rules

- Flux is the default and only normal deployment path.
- Manual apply is break-glass only and must reconcile back to declarative source.
- Promotion and rollback are done by changing OCI source version references.
