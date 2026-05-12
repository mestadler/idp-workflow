# PR3 Checklist — Deterministic Release Contract (M3 / DG-3)

## Purpose

Define the release interface and mandatory gates so build/sign/publish behavior is deterministic and reusable.

## Scope

- Build/sign/publish contract definition.
- Input/output interface and artifact metadata requirements.
- Mandatory failure gates and stop conditions.

## Non-goals

- Service-specific pipeline optimizations.
- Multi-environment promotion automation details beyond contract boundaries.
- Alternative release ordering models.

## Implementation Work Items

- [x] Define release inputs (source ref, version, target registry paths, keys).
- [x] Define release outputs (signed image, SBOM, signed deploy artifact, digest-pinned overlay change).
- [x] Define canonical operation order: build -> push -> SBOM -> sign -> digest pin -> publish deploy artifact -> sign deploy artifact.
- [x] Define gate criteria and fail-fast behavior for each step.
- [x] Define traceability requirements linking version, digest, signatures, and deploy artifact revision.

## Mandatory Test Requirements

- [x] Contract order is explicit and unambiguous.
- [x] Every release step has pass/fail criteria.
- [x] Traceability requirements are testable from command outputs.

## Test Evidence Log

| ID | Test | Command(s) | Expected | Actual | Result |
|---|---|---|---|---|---|
| T01 | Release order completeness | `rg -n "build|push|sbom|sign|digest|flux push artifact" developer-workflow.md` | Canonical order present and consistent | Ordered build-sign-publish flow and gating anchors are present in workflow release section. | PASS |
| T02 | Gate definition review | _Manual contract review (`docs/release-contract.md`)_ | Step-level pass/fail and stop conditions documented | "Step Gates and Fail-Fast Rules" defines pass/fail behavior for each release step. | PASS |
| T03 | Traceability mapping review | _Manual review + sample mapping_ | Version/digest/signature/deploy artifact link is explicit | "Traceability Requirements" lists required mapping fields linking source/version/digests/signatures/artifacts. | PASS |

## Exit Criteria (Review-Ready)

- [x] All implementation work items completed.
- [x] All mandatory tests marked pass with evidence in the log.
- [x] No out-of-scope behavior included.
- [ ] DG-3 decision recorded and approved.

## Decision Gate (DG-3)

**Question:** Is the release contract deterministic and enforceable for all adopters?

**Pass criteria:**
- Ordered contract is fixed.
- All gates and failures are explicit.
- Traceability requirements are complete.

## Handoff Notes

- M4 implementation must follow this contract without reordering steps.
- Contract changes after DG-3 require re-validation of M4 and M5.

## DG-3 Decision Record

- Status: Pending PR approval sign-off
- Decision: M3 release contract is execution-ready for M4, pending reviewer approval.
- Evidence: T01/T02/T03 all PASS in this checklist.
- Approval method: PR review approval (per `docs/TRACKING-STANDARD.md`).
- PR link: _TBD_
