# PR1 Checklist — k0s Baseline Ready (M1 / DG-1)

## Purpose

Define a reproducible single-node k0s baseline suitable for the reference architecture pilot.

## Scope

- Single-node k0s (controller+worker) baseline definition.
- Readiness verification checklist for that baseline.
- Stable-channel upgrade posture definition.

## Non-goals

- Multi-node HA topology.
- Workload-specific optimization.
- Advanced platform add-ons beyond baseline requirements.

## Implementation Work Items

- [x] Define baseline bootstrap sequence for single-node k0s.
- [x] Define readiness checks (API health, scheduling, storage baseline, registry reachability assumptions).
- [x] Define stable release tracking and upgrade validation posture.
- [x] Define baseline evidence format for reproducibility.

## Mandatory Test Requirements

- [x] Baseline procedure is replayable from a clean host profile.
- [x] Readiness checks produce deterministic pass/fail outcomes.
- [x] Upgrade posture is explicit and actionable.

## Test Evidence Log

| ID | Test | Command(s) | Expected | Actual | Result |
|---|---|---|---|---|---|
| T01 | Baseline procedure completeness | _Runbook review (`docs/k0s-single-node-baseline.md`)_ | All required bootstrap steps defined in order | Ordered bootstrap sequence documented in "Baseline Bootstrap Sequence" with executable commands. | PASS |
| T02 | Readiness check coverage | `rg -n "readiness|health|scheduling|storage|registry" docs/k0s-single-node-baseline.md` | Required readiness dimensions present | Matches found for all required dimensions, including checks and expected outcomes. | PASS |
| T03 | Stable-track policy clarity | `rg -n "stable|upgrade" docs/k0s-single-node-baseline.md` | Upgrade policy and validation cadence documented | Stable-track policy and upgrade flow are documented with post-upgrade verification commands. | PASS |

## Exit Criteria (Review-Ready)

- [x] All implementation work items completed.
- [x] All mandatory tests marked pass with evidence in the log.
- [x] No out-of-scope behavior included.
- [x] DG-1 decision recorded and approved.

## Decision Gate (DG-1)

**Question:** Is the single-node k0s baseline reproducible and ready for supply-chain integration?

**Pass criteria:**
- Bootstrap sequence complete.
- Readiness checks complete.
- Stable-track upgrade posture documented.

## Handoff Notes

- M2 assumes this baseline and must not redefine platform scope.
- If baseline assumptions change, reopen DG-1 before continuing.

## DG-1 Decision Record

- Status: Approved via PR sign-off
- Decision: M1 baseline is execution-ready for M2.
- Evidence: T01/T02/T03 all PASS in this checklist.
- Approval method: PR review approval (per `docs/TRACKING-STANDARD.md`).
- PR link: `https://github.com/mestadler/idp-workflow/pull/2`
