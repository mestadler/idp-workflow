# PR7 Checklist — E2E Reference Validation (M7 / DG-7)

## Purpose

Validate the end-to-end reference lifecycle and produce release-ready evidence for adopters.

## Scope

- Full lifecycle validation for one service-agnostic pilot workload.
- Evidence capture for build/sign/publish/deploy/rollback.
- Final reference architecture signoff artifacts.

## Non-goals

- Service-specific benchmark tuning.
- Multi-cluster validation.
- Additional controller experimentation in v1.

## Implementation Work Items

- [ ] Execute full lifecycle dry run: build -> sign -> publish -> Flux reconcile.
- [ ] Execute verification checks for signatures, digest pinning, and running state.
- [ ] Execute declarative rollback drill and verify expected outcome.
- [ ] Capture evidence bundle with commands, expected, actual, and results.
- [ ] Produce final v1 signoff summary and extension notes for adopters.

## Mandatory Test Requirements

- [ ] End-to-end lifecycle succeeds without undocumented manual steps.
- [ ] Signed artifact and digest-pinning checks pass.
- [ ] Rollback drill succeeds and is reproducible.

## Test Evidence Log

| ID | Test | Command(s) | Expected | Actual | Result |
|---|---|---|---|---|---|
| T01 | E2E lifecycle execution | _Command sequence per release/deploy contract_ | Full lifecycle completes successfully | _TBD_ | ⬜ |
| T02 | Trust and immutability verification | _Signature + digest verification commands_ | Artifact trust and digest pinning validated | _TBD_ | ⬜ |
| T03 | Declarative rollback drill | _Rollback command sequence_ | Rollback succeeds and state converges | _TBD_ | ⬜ |

## Exit Criteria (Review-Ready)

- [ ] All implementation work items completed.
- [ ] All mandatory tests marked pass with evidence in the log.
- [ ] No out-of-scope behavior included.
- [ ] DG-7 decision recorded and approved.

## Decision Gate (DG-7)

**Question:** Is the v1 reference architecture release-ready for adoption and extension?

**Pass criteria:**
- E2E lifecycle passes.
- Trust and digest controls validated.
- Rollback and operational evidence complete.

## Handoff Notes

- Mark v1 reference architecture as complete only after DG-7 approval.
- Any post-v1 extension should start as a new milestone sequence (M8+).
