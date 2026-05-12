# PR6 Checklist — Break-glass SOP (M6 / DG-6)

## Purpose

Define a constrained emergency procedure that preserves deterministic, declarative operations as the default.

## Scope

- Break-glass invocation criteria.
- Approval and audit requirements.
- Mandatory post-incident return-to-declarative reconciliation.

## Non-goals

- Normalizing imperative deployment workflows.
- Expanding emergency powers beyond narrowly defined recovery actions.
- Bypassing trust-policy without accountability.

## Implementation Work Items

- [ ] Define explicit trigger conditions for break-glass activation.
- [ ] Define required approver role and authorization record.
- [ ] Define audit log requirements and retention expectations.
- [ ] Define mandatory reconciliation-back steps after incident resolution.
- [ ] Define criteria for closing a break-glass event.

## Mandatory Test Requirements

- [ ] Break-glass trigger criteria are narrow and objective.
- [ ] Approval/audit path is explicit and testable.
- [ ] Reconciliation-back workflow is mandatory and complete.

## Test Evidence Log

| ID | Test | Command(s) | Expected | Actual | Result |
|---|---|---|---|---|---|
| T01 | Trigger criteria review | _Manual SOP review_ | Trigger conditions are narrow and non-ambiguous | _TBD_ | ⬜ |
| T02 | Approval/audit review | _Manual SOP review_ | Approver and audit requirements are explicit | _TBD_ | ⬜ |
| T03 | Reconciliation requirement review | _Manual SOP review_ | Return-to-declarative steps are mandatory and sequenced | _TBD_ | ⬜ |

## Exit Criteria (Review-Ready)

- [ ] All implementation work items completed.
- [ ] All mandatory tests marked pass with evidence in the log.
- [ ] No out-of-scope behavior included.
- [ ] DG-6 decision recorded and approved.

## Decision Gate (DG-6)

**Question:** Is the break-glass path sufficiently constrained and recoverable?

**Pass criteria:**
- Trigger and approval requirements are explicit.
- Audit trail is mandatory.
- Post-incident reconciliation is mandatory.

## Handoff Notes

- M7 validation must include break-glass compatibility checks (without making it default).
- Any broadening of break-glass scope requires DG-6 reapproval.
