# AGENTS.md

## Scope
- This repository is documentation-only; currently the only tracked project file is `developer-workflow.md`.
- There is no build/test/lint pipeline here. Do not invent one or run workspace-wide commands from this directory.

## Source of truth
- Treat `developer-workflow.md` as the canonical artifact for this repo.
- Prefer executable examples inside that file (commands and ordered workflows) over prose summaries when updating guidance.
- For Kubernetes operational CLIs (`kubectl`, `helm`, `jq`, `skopeo`, `oras`, `cosign`, `flux`), default to the `k8s-tools` packaging flow documented in the `k8s-tools` README and mirrored in `developer-workflow.md`.

## What to change (and what not to)
- Make focused edits to `developer-workflow.md` and keep command sequences internally consistent (toolchain setup -> runtime/registry -> build/sign -> deploy/GitOps).
- Do not add unrelated project scaffolding (`package.json`, CI workflows, formatter configs) unless explicitly requested.
- If a command/version is changed, update all dependent references in the same pass (for example: install snippets, release pipeline examples, and verification commands).

## Navigation discipline
- Use targeted search first (`rg`, then `fzf` if needed) before reading files.
- Read only files required for the current task; avoid broad scans unless explicitly needed for consistency checks.

## Verification expectations
- Verification here is editorial/consistency-based: check for broken command flow, conflicting versions, and contradictory lifecycle claims.
- If runtime verification is requested, call out that most commands target external systems (containerd, registry, Flux/Kubernetes) and require environment access not provided by this repo.

## Workspace boundary reminder
- `~/Devops` contains many independent repositories; this one is `devops-config` only.
- Do not assume tools, conventions, or git history from sibling directories apply here unless explicitly referenced.
