# Deterministic Release Contract (M3)

## Purpose

Define a deterministic, enforceable release interface for OCI build/sign/publish.

## Scope

- Release inputs and outputs.
- Fixed operation order.
- Step-level pass/fail gates.
- Required traceability fields.

## Release Inputs

- `SOURCE_REF`: source commit/tag being released.
- `VERSION`: release identifier (for example `v1.2.3`).
- `APP_REPO`: app image repository (`git.sansnom.uk/<org>/<service>`).
- `DEPLOY_REPO`: deploy artifact repository (`git.sansnom.uk/<org>/<service>-deploy`).
- `COSIGN_KEY`: signing key path/handle for cosign.
- `OVERLAY_PATH`: production overlay path for digest pin updates.

## Release Outputs

- OCI app image pushed to registry.
- SBOM generated for released image.
- Signed app image.
- Production overlay image updated with `newTag` + `digest`.
- Deploy manifests pushed as OCI artifact.
- Signed deploy artifact.
- Traceability record linking source, version, digests, and signatures.

## Canonical Operation Order

The release order is fixed and must not be reordered:

1. Build image.
2. Push image.
3. Generate SBOM.
4. Sign image.
5. Resolve pushed image digest and update production overlay pin.
6. Push deploy artifact.
7. Sign deploy artifact.

## Step Gates and Fail-Fast Rules

Each step is a hard gate. On failure, stop immediately.

1. **Build gate**
   - Pass: image build command succeeds.
   - Fail: no downstream step allowed.
2. **Push gate**
   - Pass: pushed image is addressable in registry.
   - Fail: stop; do not generate release evidence.
3. **SBOM gate**
   - Pass: SBOM file generated for pushed image reference.
   - Fail: stop before signing.
4. **Image signature gate**
   - Pass: `cosign sign` and `cosign verify` succeed for app image.
   - Fail: stop before overlay/deploy updates.
5. **Digest pin gate**
   - Pass: production overlay contains release tag + resolved digest.
   - Fail: stop before deploy artifact publish.
6. **Deploy artifact publish gate**
   - Pass: deploy artifact exists at `<DEPLOY_REPO>:<VERSION>`.
   - Fail: stop before deploy signature.
7. **Deploy signature gate**
   - Pass: `cosign sign` and `cosign verify` succeed for deploy artifact.
   - Fail: release is incomplete and must not be promoted.

## Traceability Requirements

Each release must record:

- `source_ref`
- `version`
- `app_image_ref`
- `app_image_digest`
- `deploy_artifact_ref`
- `deploy_artifact_revision`
- `app_signature_verification` (pass/fail + verifier key id/path)
- `deploy_signature_verification` (pass/fail + verifier key id/path)
- `overlay_pin_reference` (`newTag` + `digest`)

## Minimal Verification Commands

```bash
cosign verify --key cosign.pub "${APP_REPO}:${VERSION}"
cosign verify --key cosign.pub "${DEPLOY_REPO}:${VERSION}"
```

## Contract Invariants

- Production references are digest-pinned.
- Unsigned artifacts are not release-complete.
- Version-to-digest mapping is immutable once published.
- Deploy path consumes OCI artifacts, not mutable local state.

## Non-goals

- Defining service-specific CI implementation details.
- Supporting alternative operation orders.
- Defining multi-environment orchestration logic in this contract.
