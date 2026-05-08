---
name: new-agent-evolution
description: Capture phase of an agent evolution — discuss the objective, gates, rubric, scope, and budget; write `.agents/evolutions/<id>-<slug>/{evolution.yaml, EVOLUTION.md}` only on explicit user agreement. Pairs with `/run-agent-evolution` (next) and `/apply-agent-evolution` (later).
---

# new-agent-evolution

Capture a verifiable optimization brief for `<title>`. Writes nothing until the user explicitly agrees.

## 0. Pre-flight

Resolve the argument. If it matches an existing `.agents/evolutions/<id>-*/` (pure integer id or slug fragment), stop with `"Evolution <id> already exists. Run /run-agent-evolution <id> (or /apply-agent-evolution <id> if the winner is set)."` This skill only seeds new evolutions.

Read-only setup: `AGENTS.md`, `README.md`, the user ask, any spec the ask points at. Identify project test/build/lint commands — candidate gates. Confirm the cwd is a git repo (`git rev-parse --is-inside-work-tree`) — variants default to git worktrees.

## 1. Dialog first

First-turn output is chat-only: clarifying questions and a draft brief inline. Don't create files until the user explicitly agrees ("looks good", "ship it", edits incorporated).

Five elements. Propose concrete answers; ask only when irreducible (cap at 5 questions).

- **Objective.** One sentence — "Optimize X for Y." Reject vague answers; insist on a measurable Y.
- **Gates.** Hard pass/fail shell commands (exit 0 = pass). 2–4 typical (`pnpm typecheck`, `pnpm test`, `cargo build`, custom verifiers). A variant failing any gate is excluded from ranking.
- **Rubric.** Numeric ranking dimensions (≥ 1, typically 1–3). Each: `direction` (minimize | maximize), `cmd`, optional `extract` (regex with one capture group, or `wall_clock_seconds`). Composite is the rank-normalized weighted mean across axes, recomputed on read.
- **Scope.** `kind: worktree` (default — `git worktree add HEAD`) or `files` (copy listed paths). For `worktree`, list `include` informationally so apply knows what to diff. For `files`, `include` is required. `exclude` blocks paths apply must not touch.
- **Budget.** `max_variants` (total), `parallel` (concurrent per generation; sets the learning cadence). Optional `max_minutes`, `plateau_generations` (early stop on no-improvement). Default `parallel = ceil(max_variants / 5)` so the loop runs ~5 generations.

Reply in chat with: clarifying questions, a draft brief inline, a one-line apply summary. No file writes.

## 2. Commit on agreement

After explicit agreement:

1. `<id>` = highest leading integer in `.agents/evolutions/<n>-*/`, +1 (start at 1).
2. Slug the objective to lower snake_case ≤ 40 chars.
3. `mkdir -p .agents/evolutions/<id>-<slug>/variants/`.
4. Add `.agents/evolutions/*/variants/` to `.gitignore` if missing — workspaces are large and ephemeral.

Write two files (validate `evolution.yaml` against `evolution.schema.json`):

- **`evolution.yaml`** — seed with `evolution_id`, `slug`, `created_at`, `updated_at` (same), `objective`, `scope`, `budget`, `gates`, `rubric`, `variants: []`. Field shape lives in `evolution.schema.json`; mirror its required keys.
- **`EVOLUTION.md`** — single human/agent surface. §TL;DR (objective, budget summary, next-step pointer to `/run-agent-evolution <id>`); §Brief — the verbatim context every variant sub-agent receives — with subsections Objective, Held fixed (stack, public surface, deps, tests, anything that must not change), Variants may change (the design space — specific files / dimensions), Scope (filesystem `kind` / `include` / `exclude`), Gates, Rubric, Budget, Out of scope (explicit non-goals); §Variants (empty); §Results (empty). Short, declarative, no prose explaining _why_.

End with `"Brief captured. Run /run-agent-evolution <id> to begin generation 1."` If the user abandons before agreement, write nothing.
