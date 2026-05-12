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

- [ ] Define baseline bootstrap sequence for single-node k0s.
- [ ] Define readiness checks (API health, scheduling, storage baseline, registry reachability assumptions).
- [ ] Define stable release tracking and upgrade validation posture.
- [ ] Define baseline evidence format for reproducibility.

## Mandatory Test Requirements

- [ ] Baseline procedure is replayable from a clean host profile.
- [ ] Readiness checks produce deterministic pass/fail outcomes.
- [ ] Upgrade posture is explicit and actionable.

## Test Evidence Log

| ID | Test | Command(s) | Expected | Actual | Result |
|---|---|---|---|---|---|
| T01 | Baseline procedure completeness | _Runbook review_ | All required bootstrap steps defined in order | _TBD_ | ⬜ |
| T02 | Readiness check coverage | `rg -n "readiness|health|scheduling|storage|registry" <baseline-doc>` | Required readiness dimensions present | _TBD_ | ⬜ |
| T03 | Stable-track policy clarity | `rg -n "stable|upgrade" <baseline-doc>` | Upgrade policy and validation cadence documented | _TBD_ | ⬜ |

## Exit Criteria (Review-Ready)

- [ ] All implementation work items completed.
- [ ] All mandatory tests marked pass with evidence in the log.
- [ ] No out-of-scope behavior included.
- [ ] DG-1 decision recorded and approved.

## Decision Gate (DG-1)

**Question:** Is the single-node k0s baseline reproducible and ready for supply-chain integration?

**Pass criteria:**
- Bootstrap sequence complete.
- Readiness checks complete.
- Stable-track upgrade posture documented.

## Handoff Notes

- M2 assumes this baseline and must not redefine platform scope.
- If baseline assumptions change, reopen DG-1 before continuing.
