# PR4 Checklist — Flux GitOps Path (M4 / DG-4)

## Purpose

Define and validate Flux OCI reconciliation as the default deployment path.

## Scope

- Flux bootstrap profile for single-node k0s baseline.
- OCI source and reconciliation object model.
- Declarative promotion and rollback model.

## Non-goals

- Imperative deployment as normal operations.
- Non-Flux controllers in v1.
- Multi-cluster orchestration patterns.

## Implementation Work Items

- [ ] Define Flux bootstrap prerequisites and steps for baseline cluster.
- [ ] Define OCI source object and reconciliation flow.
- [ ] Define promotion workflow (version/artifact selection).
- [ ] Define rollback workflow (artifact/version re-selection).
- [ ] Define observability checkpoints for reconcile status and failures.

## Mandatory Test Requirements

- [ ] Flux path is complete without imperative deployment dependencies.
- [ ] Promotion and rollback are declarative and reproducible.
- [ ] Reconciliation status checks are clear and actionable.

## Test Evidence Log

| ID | Test | Command(s) | Expected | Actual | Result |
|---|---|---|---|---|---|
| T01 | Flux flow completeness | `rg -n "Flux|OCIRepository|Kustomization|reconcile|artifact" developer-workflow.md` | End-to-end Flux path is documented | _TBD_ | ⬜ |
| T02 | Declarative rollback review | _Manual review of rollback section_ | Rollback uses artifact/version selection, not imperative defaults | _TBD_ | ⬜ |
| T03 | Operational observability review | _Manual review of status checks_ | Reconcile success/failure checks are explicit | _TBD_ | ⬜ |

## Exit Criteria (Review-Ready)

- [ ] All implementation work items completed.
- [ ] All mandatory tests marked pass with evidence in the log.
- [ ] No out-of-scope behavior included.
- [ ] DG-4 decision recorded and approved.

## Decision Gate (DG-4)

**Question:** Can normal deployment occur only through declarative Flux flow?

**Pass criteria:**
- Flux path is complete for normal operations.
- Promotion/rollback are declarative.
- No imperative default path remains.

## Handoff Notes

- M5 policy enforcement must attach to this deployment model.
- If deploy model changes, reopen DG-4 and downstream gates.
