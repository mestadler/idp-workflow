# PR0 Checklist — Architecture Contract Lock (M0 / DG-0)

## Purpose

Freeze the v1 architecture contract so implementation can proceed without scope drift.

## Scope

- Define and lock platform baseline assumptions.
- Define and lock delivery defaults and trust model.
- Define and lock non-goals and extension boundaries.

## Non-goals

- Implementing cluster bootstrap.
- Implementing CI/CD pipelines.
- Service-specific design choices.

## Implementation Work Items

- [x] Record v1 platform policy: `k0s` current stable release track.
- [x] Record v1 deployment policy: Flux is default path.
- [x] Record exception policy: manual deployment is break-glass only.
- [x] Record signing/key policy: cosign keys managed with SOPS.
- [x] Record pilot boundary: single-node cluster + single service-agnostic pilot workload.
- [x] Record explicit v1 non-goals and extension points.

## Mandatory Test Requirements

- [x] Contract statements are consistent with `developer-workflow.md`.
- [x] No contradictions with `AGENTS.md` constraints.
- [x] All defaults are explicit (no implied behavior).

## Test Evidence Log

| ID | Test | Command(s) | Expected | Actual | Result |
|---|---|---|---|---|---|
| T01 | Workflow consistency review | `rg -n "k0s|Flux|break-glass|SOPS|stable" developer-workflow.md AGENTS.md` | All required policy anchors present and non-conflicting | Anchors found in both files; no direct conflicts observed in current text. | PASS |
| T02 | Contract completeness review | _Manual review of PR0/TODO context_ | All required defaults/non-goals explicitly stated | Defaults present in `TODO.md` Architecture/Delivery Context; non-goals present in `TODO.md` Out of scope and PR0 Non-goals. | PASS |
| T03 | Ambiguity review | _Manual review against open items_ | No unresolved ambiguity that blocks M1 | DG approval model resolved in `docs/TRACKING-STANDARD.md`; remaining open item (evidence artifact format) does not block M1 baseline definition. | PASS |

## Exit Criteria (Review-Ready)

- [x] All implementation work items completed.
- [x] All mandatory tests marked pass with evidence in the log.
- [x] No out-of-scope behavior included.
- [ ] DG-0 decision recorded and approved.

## Decision Gate (DG-0)

**Question:** Is v1 scope frozen and unambiguous for execution?

**Pass criteria:**
- Platform, trust, and deploy defaults are explicit.
- Non-goals are explicit.
- No blocking ambiguities remain for M1.

## Handoff Notes

- M1 consumes this contract as normative input.
- Any scope change after DG-0 requires reopening DG-0 and re-validating downstream assumptions.

## DG-0 Decision Record

- Status: Pending PR approval sign-off
- Decision: PR0 scope is execution-ready for M1, pending reviewer approval.
- Evidence: T01/T02/T03 all PASS in this checklist.
- Approval method: PR review approval (per `docs/TRACKING-STANDARD.md`).
- PR link: _TBD_
