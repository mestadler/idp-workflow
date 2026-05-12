# TODO

Single-page status index for phased delivery of the k0s-based, OCI-first, Flux-driven reference architecture.

---

## Status

Planning baseline is complete.
PR0 / M0 is complete (DG-0 approved).
Execution moves to PR1 / M1 (k0s baseline).

## Architecture / Delivery Context

- Platform: `k0s` (track current stable release).
- Scope: single-node (controller+worker) + one service-agnostic pilot workload.
- Supply chain: OCI artifacts in Forgejo, signed with cosign, keys managed via SOPS.
- Deployment model: Flux default path; manual deploy is break-glass only.

## Done

- [x] Established and refined `developer-workflow.md` as canonical workflow.
- [x] Added repository guidance in `AGENTS.md`.
- [x] Added tracking standard and templates in `docs/`.
- [x] PR0 — Architecture Contract Locked (M0 / DG-0).

## In flight

- [ ] **PR1 — k0s Baseline Ready (M1 / DG-1)**
      Define reproducible single-node k0s baseline and readiness checks.
      Detailed execution tracker: `docs/pr1-k0s-baseline-checklist.md`.

## Implementation pipeline (next)

Each phase lands as its own PR.

- [ ] **PR0 — Architecture Contract Locked (M0 / DG-0)**
      Freeze v1 boundaries and normative defaults.
      Detailed execution tracker: `docs/pr0-architecture-contract-checklist.md`.

- [ ] **PR1 — k0s Baseline Ready (M1 / DG-1)**
      Define reproducible single-node k0s baseline and readiness checks.
      Detailed execution tracker: `docs/pr1-k0s-baseline-checklist.md`.

- [ ] **PR2 — Toolchain + Supply Chain Baseline (M2 / DG-2)**
      Standardize k8s tooling source, registry conventions, and SOPS-backed cosign key model.
      Detailed execution tracker: `docs/pr2-toolchain-supply-chain-checklist.md`.

- [ ] **PR3 — Deterministic Release Contract (M3 / DG-3)**
      Lock build/sign/publish contract and mandatory failure gates.
      Detailed execution tracker: `docs/pr3-release-contract-checklist.md`.

- [ ] **PR4 — Flux GitOps Path (M4 / DG-4)**
      Implement OCI-driven Flux reconciliation as default deploy path.
      Detailed execution tracker: `docs/pr4-flux-gitops-checklist.md`.

- [ ] **PR5 — Policy Enforcement (M5 / DG-5)**
      Enable signature verification/admission policy and define enforce posture.
      Detailed execution tracker: `docs/pr5-policy-enforcement-checklist.md`.

- [ ] **PR6 — Break-glass SOP (M6 / DG-6)**
      Define constrained exception workflow and mandatory return-to-declarative reconciliation.
      Detailed execution tracker: `docs/pr6-break-glass-checklist.md`.

- [ ] **PR7 — E2E Reference Validation (M7 / DG-7)**
      Validate full lifecycle and publish reference-ready evidence.
      Detailed execution tracker: `docs/pr7-e2e-validation-checklist.md`.

## Open design items

- [ ] Define exact evidence artifact format for gate approvals (lightweight ADR vs checklist-only note).
- [x] Define approval authority mapping for DG-0 through DG-7: gate approval is PR sign-off with checklist evidence complete.

## Out of scope

- Multi-node HA control plane for v1.
- Service-specific implementation details.
- Non-Flux drift checker deployment in v1.

## How to refresh this file

Update this file when:

- A phase PR opens or merges (move bullets between Done / In flight / Next).
- Scope or sequencing changes at milestone boundaries.
- A design item is promoted to implementation or deferred.

## Methodology (Reusable)

Use a two-layer tracking model:

- `TODO.md` = high-level status and sequencing.
- `docs/pr*-*-checklist.md` = phase execution details.

Workflow:

1. Add/maintain phase bullets in `TODO.md`.
2. Link each phase to a dedicated checklist file in `docs/`.
3. Break work into atomic checkboxes in the checklist.
4. Define mandatory tests and log command-level evidence.
5. Mark phase review-ready only when all mandatory tests pass.

## References

| Document | Purpose |
|---|---|
| `developer-workflow.md` | Canonical workflow and lifecycle design. |
| `AGENTS.md` | Repo-specific rules for agent behavior and boundaries. |
| `docs/TRACKING-STANDARD.md` | Reusable phase tracking methodology. |
| `docs/pr-phase-checklist-template.md` | Template for per-phase checklists. |
