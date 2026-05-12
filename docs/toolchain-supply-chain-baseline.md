# Toolchain and Supply Chain Baseline (M2)

## Purpose

Define deterministic tooling, artifact naming, and trust boundaries for build and deploy workflows.

## Scope

- Kubernetes operational CLIs from `k8s-tools` packages.
- OCI naming and tagging conventions for app and deploy artifacts.
- Cosign key lifecycle with SOPS-managed keys.
- CI and runtime trust boundaries.
- Minimum verification gates required for release and deploy.

## Canonical Toolchain Source

Use `k8s-tools` as the default source for:

- `kubectl`
- `helm`
- `jq`
- `skopeo`
- `oras`
- `cosign`
- `flux`

Reference install flow:

- `developer-workflow.md` Section 1.3
- `https://github.com/sansnom-co/k8s-tools/blob/main/README.md`

Usage points:

- Operator workstations
- CI runners executing deploy/sign/verify commands
- Utility containers used for cluster operations

## Registry and Artifact Conventions

Registry host:

- `git.sansnom.uk`

Repository naming:

- App image: `git.sansnom.uk/<org>/<service>`
- Deploy artifact: `git.sansnom.uk/<org>/<service>-deploy`

Tagging conventions:

- Release tags: semantic or release labels (for example `v1.2.3`)
- Mutable tags (for dev only): `dev-latest`
- Production references: digest-pinned image reference in overlay

Required production pinning pattern:

- `name: git.sansnom.uk/<org>/<service>`
- `newTag: <release-tag>`
- `digest: sha256:<digest-from-pushed-image>`

## Cosign + SOPS Key Lifecycle

Key model for v1:

- Cosign keypair signing model (no keyless flow in v1).
- Private key material stored encrypted with SOPS.
- Public key distributed to verification points (CI and cluster policy consumers).

Lifecycle:

1. Create keypair (`cosign generate-key-pair`) in controlled environment.
2. Encrypt private key with SOPS before committing or storing in shared repos.
3. Decrypt only at signing time in CI/runtime with least-privilege access.
4. Use public key for verification in release checks and admission policy.
5. Rotate keys by generating new keypair, updating SOPS material, and updating verifiers.

Rules:

- Never commit plaintext private keys.
- Restrict decryption rights to release/signing principals.
- Record rotation events and effective dates.

## CI and Runtime Trust Boundaries

CI responsibilities:

- Build image and push to OCI registry.
- Generate SBOM.
- Sign app image and deploy artifact.
- Publish deploy artifact.

Runtime/cluster responsibilities:

- Pull only approved/signed artifacts.
- Enforce signature verification policy for admission/deploy path.
- Reconcile desired state through Flux.

Boundary rules:

- CI can sign; runtime does not sign.
- Runtime verifies and reconciles; it does not define release intent.
- Manual deploy path remains break-glass only.

## Minimum Verification Gates

Gate G1 (post-push image trust):

- `cosign verify --key <pubkey> <app-image>:<version>` must pass.

Gate G2 (deploy artifact trust):

- `cosign verify --key <pubkey> <deploy-artifact>:<version>` must pass.

Gate G3 (traceability):

- Release record must map `<version> -> <image digest> -> <deploy artifact revision/tag>`.

Gate G4 (prod pinning):

- Production overlay must contain digest pin for the released image.

## Traceability Record (Required Fields)

For each release, record:

- Source revision/commit
- Version tag
- App image reference
- App image digest
- Deploy artifact reference
- Signature verification results (app + deploy)
- Overlay update reference (tag + digest)

## Non-goals

- Defining app-specific CI internals.
- Supporting non-SOPS signing key management in v1.
- Supporting non-Debian/Ubuntu package strategy in this baseline.
