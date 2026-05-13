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

- [x] Define Flux bootstrap prerequisites and steps for baseline cluster.
- [x] Define OCI source object and reconciliation flow.
- [x] Define promotion workflow (version/artifact selection).
- [x] Define rollback workflow (artifact/version re-selection).
- [x] Define observability checkpoints for reconcile status and failures.

## Mandatory Test Requirements

- [x] Flux path is complete without imperative deployment dependencies.
- [x] Promotion and rollback are declarative and reproducible.
- [x] Reconciliation status checks are clear and actionable.

## Test Evidence Log

| ID | Test | Command(s) | Expected | Actual | Result |
|---|---|---|---|---|---|
| T01 | Flux flow completeness | `rg -n "Flux|OCIRepository|Kustomization|reconcile|artifact" developer-workflow.md` | End-to-end Flux path is documented | Flux OCI source and Kustomization reconciliation model are documented in workflow with aligned references. | PASS |
| T02 | Declarative rollback review | _Manual review of `docs/flux-gitops-path.md` rollback section_ | Rollback uses artifact/version selection, not imperative defaults | Rollback section defines OCI source tag re-selection and reconcile, with no normal imperative fallback. | PASS |
| T03 | Operational observability review | _Manual review of status checks in `docs/flux-gitops-path.md`_ | Reconcile success/failure checks are explicit | Observability commands and expected outcomes are explicitly defined for source/kustomization readiness. | PASS |

## Exit Criteria (Review-Ready)

- [x] All implementation work items completed.
- [x] All mandatory tests marked pass with evidence in the log.
- [x] No out-of-scope behavior included.
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

## DG-4 Decision Record

- Status: Pending PR approval sign-off
- Decision: M4 Flux GitOps path is execution-ready for M5, pending reviewer approval.
- Evidence: T01/T02/T03 all PASS in this checklist.
- Approval method: PR review approval (per `docs/TRACKING-STANDARD.md`).
- PR link: _TBD_
