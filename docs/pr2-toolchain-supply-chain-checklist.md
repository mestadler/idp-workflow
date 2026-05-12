# PR2 Checklist — Toolchain + Supply Chain Baseline (M2 / DG-2)

## Purpose

Standardize tool installation, artifact conventions, and trust/key handling for deterministic release and deploy flow.

## Scope

- `k8s-tools` as default Kubernetes CLI source.
- Registry naming and artifact convention definitions.
- Cosign key lifecycle with SOPS-managed material.
- Verification boundary for CI/runtime.

## Non-goals

- App-specific pipeline implementation.
- Alternate signing model (e.g., keyless) in v1.
- Non-Debian/Ubuntu packaging strategy in v1 baseline.

## Implementation Work Items

- [ ] Define canonical `k8s-tools` bootstrap/install flow usage points.
- [ ] Define registry naming conventions (repo paths, tags, digest pinning rules).
- [ ] Define cosign+SOPS key lifecycle (create/store/decrypt/use/rotate).
- [ ] Define CI/runtime trust boundaries and secret handling responsibilities.
- [ ] Define minimum verification gates for signed artifacts.

## Mandatory Test Requirements

- [ ] Tool bootstrap guidance aligns with `developer-workflow.md`.
- [ ] Signing and verification path is documented end-to-end.
- [ ] Convention rules enforce traceability (tag -> digest -> signed artifact linkage).

## Test Evidence Log

| ID | Test | Command(s) | Expected | Actual | Result |
|---|---|---|---|---|---|
| T01 | Tooling source consistency | `rg -n "k8s-tools|kube-essential|cosign|oras|kubectl|flux" developer-workflow.md AGENTS.md` | Kubernetes operational CLI guidance consistently points to k8s-tools flow | _TBD_ | ⬜ |
| T02 | Trust flow completeness | _Manual review of PR2 key/trust doc_ | Create/store/decrypt/use/rotate responsibilities are explicit | _TBD_ | ⬜ |
| T03 | Traceability rule check | _Manual review + examples_ | Artifact naming and digest pinning rules are unambiguous | _TBD_ | ⬜ |

## Exit Criteria (Review-Ready)

- [ ] All implementation work items completed.
- [ ] All mandatory tests marked pass with evidence in the log.
- [ ] No out-of-scope behavior included.
- [ ] DG-2 decision recorded and approved.

## Decision Gate (DG-2)

**Question:** Is the toolchain and trust model deterministic and ready for release contract implementation?

**Pass criteria:**
- k8s operational tooling source standardized.
- SOPS-backed cosign key model approved.
- Signature verification path and conventions proven.

## Handoff Notes

- M3 depends on this as immutable contract input.
- Any trust model change after DG-2 reopens DG-2 and downstream gates (DG-3+).
