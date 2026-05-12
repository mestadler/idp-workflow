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

- [ ] Define release inputs (source ref, version, target registry paths, keys).
- [ ] Define release outputs (signed image, SBOM, signed deploy artifact, digest-pinned overlay change).
- [ ] Define canonical operation order: build -> push -> SBOM -> sign -> digest pin -> publish deploy artifact -> sign deploy artifact.
- [ ] Define gate criteria and fail-fast behavior for each step.
- [ ] Define traceability requirements linking version, digest, signatures, and deploy artifact revision.

## Mandatory Test Requirements

- [ ] Contract order is explicit and unambiguous.
- [ ] Every release step has pass/fail criteria.
- [ ] Traceability requirements are testable from command outputs.

## Test Evidence Log

| ID | Test | Command(s) | Expected | Actual | Result |
|---|---|---|---|---|---|
| T01 | Release order completeness | `rg -n "build|push|sbom|sign|digest|flux push artifact" developer-workflow.md` | Canonical order present and consistent | _TBD_ | ⬜ |
| T02 | Gate definition review | _Manual contract review_ | Step-level pass/fail and stop conditions documented | _TBD_ | ⬜ |
| T03 | Traceability mapping review | _Manual review + sample mapping_ | Version/digest/signature/deploy artifact link is explicit | _TBD_ | ⬜ |

## Exit Criteria (Review-Ready)

- [ ] All implementation work items completed.
- [ ] All mandatory tests marked pass with evidence in the log.
- [ ] No out-of-scope behavior included.
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
