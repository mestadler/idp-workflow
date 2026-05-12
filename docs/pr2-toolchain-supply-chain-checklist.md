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

- [x] Define canonical `k8s-tools` bootstrap/install flow usage points.
- [x] Define registry naming conventions (repo paths, tags, digest pinning rules).
- [x] Define cosign+SOPS key lifecycle (create/store/decrypt/use/rotate).
- [x] Define CI/runtime trust boundaries and secret handling responsibilities.
- [x] Define minimum verification gates for signed artifacts.

## Mandatory Test Requirements

- [x] Tool bootstrap guidance aligns with `developer-workflow.md`.
- [x] Signing and verification path is documented end-to-end.
- [x] Convention rules enforce traceability (tag -> digest -> signed artifact linkage).

## Test Evidence Log

| ID | Test | Command(s) | Expected | Actual | Result |
|---|---|---|---|---|---|
| T01 | Tooling source consistency | `rg -n "k8s-tools|kube-essential|cosign|oras|kubectl|flux" developer-workflow.md AGENTS.md` | Kubernetes operational CLI guidance consistently points to k8s-tools flow | Matches found in both files and install flows consistently use `k8s-tools`/`kube-essential` guidance for operational CLIs. | PASS |
| T02 | Trust flow completeness | _Manual review of `docs/toolchain-supply-chain-baseline.md`_ | Create/store/decrypt/use/rotate responsibilities are explicit | Lifecycle steps and rules are explicit in "Cosign + SOPS Key Lifecycle" with clear handling boundaries. | PASS |
| T03 | Traceability rule check | _Manual review + examples_ | Artifact naming and digest pinning rules are unambiguous | Registry naming, digest pinning pattern, verification gates, and required traceability fields are explicitly defined. | PASS |

## Exit Criteria (Review-Ready)

- [x] All implementation work items completed.
- [x] All mandatory tests marked pass with evidence in the log.
- [x] No out-of-scope behavior included.
- [x] DG-2 decision recorded and approved.

## Decision Gate (DG-2)

**Question:** Is the toolchain and trust model deterministic and ready for release contract implementation?

**Pass criteria:**
- k8s operational tooling source standardized.
- SOPS-backed cosign key model approved.
- Signature verification path and conventions proven.

## Handoff Notes

- M3 depends on this as immutable contract input.
- Any trust model change after DG-2 reopens DG-2 and downstream gates (DG-3+).

## DG-2 Decision Record

- Status: Approved via PR sign-off
- Decision: M2 toolchain and trust baseline is execution-ready for M3.
- Evidence: T01/T02/T03 all PASS in this checklist.
- Approval method: PR review approval (per `docs/TRACKING-STANDARD.md`).
- PR link: `https://github.com/mestadler/idp-workflow/pull/3`
