# AGENTS.md

## Overview

- agent-evolutions ships three slash-command skills — `/new-agent-evolution`, `/run-agent-evolution`, `/apply-agent-evolution` — that together drive a coding agent through a genetic exploration loop: gather a verifiable objective, generate variants in parallel batches, learn from each generation, pick the winner by exit codes and numeric rubric, port the winner into the repo. One skill per phase (Capture / Run / Apply); each refuses to operate outside its phase based on `evolution.yaml` field presence.
- Skills are file-based and harness-agnostic: the same `skills/` tree works in Claude Code, Gemini CLI, and GitHub Copilot.
- End-user state lives in `.agents/evolutions/<id>-<slug>/`: `evolution.yaml` (machine state, validated against `evolution.schema.json`), `EVOLUTION.md` (single human/agent surface — TL;DR, Brief, Variants, Results; the §Brief section is loaded verbatim by every variant sub-agent), `variants/v<n>/workspace/` (per-variant isolated checkout or file copy), `variants/v<n>/result.json` (sub-agent output, validated against `result.schema.json`).
- State is **derived from field presence**, not an explicit step machine: empty `variants[]` = ready (run `/new-agent-evolution` first); has variants but no `winner` = running; has `winner` but no `applied` = evaluated; has `applied` = done. Each skill checks reality on entry and refuses if invoked in the wrong phase — no `step` enum, no `pause` flag.

## Conventions

- **Line caps.** Skill files load into agent context on every run; verbosity directly costs context budget. Targets:
  - `skills/<name>/SKILL.md` — ≤ 100 lines per file (each phase skill is self-contained).
  - Project-root markdown (`README.md`, `AGENTS.md`) — uncapped, but tighten when it sprawls.

  When the file approaches its cap, prefer cutting motivational/explanatory prose over removing structural rules. Each rule should be stated once, in the imperative, in the file the agent loads when it needs it. The "why" lives in commit messages and PR descriptions.

- **Lint.** Run `npm run lint` to validate changes — Prettier formatting plus markdownlint-cli2. `pre-commit` and CI run the same hook.

- **Voice.** The skill body addresses the agent in the imperative ("Read X, then write Y"). Avoid second-person flavor text aimed at the human reader — that belongs in the README.

- **Evidence over opinion.** The genetic loop replaces taste with verification. Claims about which variant wins are backed by exit codes (gates) and numeric measures (rubric). Frozen criteria; no result-driven additions; no silent retries; no narrative scoring.

- **Compute on read, not on write.** Composite scores and ranks are derived from `variants[i].rubric` values by the run skill — never persisted in `evolution.yaml`. The yaml stores raw inputs; views are recomputed.

- **File contract for sub-agents.** Variant sub-agents return their evaluation by writing `variants/v<n>/result.json` (schema'd by `result.schema.json`), not by formatting JSON in chat. The parent reads files; the chat reply is advisory.

## Workflow

Three skills, one per phase, each enforcing its own preconditions on `evolution.yaml` (this repo does not consume its own skills — the evolution state lives in end-user projects):

- `/new-agent-evolution <title>` — **Capture.** Discuss objective, gates, rubric, scope, budget; commit `.agents/evolutions/<id>-<slug>/{evolution.yaml, EVOLUTION.md}` only on agreement. Refuses if the directory already exists.
- `/run-agent-evolution <id>` — **Run.** Plan a batch of up to `budget.parallel` variants, dispatch sub-agents in parallel to materialize and evaluate them (each writes `result.json`), learn from results, plan the next batch as mutations + crossovers + explores. Stop on budget exhaustion, generation cap, wall-clock cap, or score plateau. Write `winner` on a clean stop. Refuses if `winner` is already set or the evolution does not exist.
- `/apply-agent-evolution <id>` — **Apply.** Diff the winner workspace against the live repo, get explicit user confirmation, copy files in, optionally re-run gates against the live tree. Write `applied`. Refuses if `winner` is unset or `applied` already set.

Cancellation is manual: delete the evolution directory, or stop running and let the partial state stay as audit. Inspecting state without advancing it is a plain read of `EVOLUTION.md` (no status skill — the file is the status).

## Layout

- `skills/new-agent-evolution/SKILL.md`, `skills/run-agent-evolution/SKILL.md`, `skills/apply-agent-evolution/SKILL.md` — one Agent Skill per phase. No `references/` or `templates/` — each SKILL.md is the whole skill.
- `evolution.schema.json` — JSONSchema for end-user `.agents/evolutions/*/evolution.yaml`.
- `result.schema.json` — JSONSchema for the per-variant `variants/v<n>/result.json` written by sub-agents.
- `examples/evolutions/` — runnable walkthroughs.
- `.claude-plugin/`, `gemini-extension.json`, `plugin.json` — harness manifests for Claude Code, Gemini CLI, GitHub Copilot.
- `CLAUDE.md`, `GEMINI.md` — single-line `@AGENTS.md` includes so Claude Code and Gemini CLI pick up the canonical context.
- `package.json`, `.markdownlint-cli2.jsonc`, `.prettierrc.json`, `.prettierignore` — `npm run lint` config (Prettier + markdownlint-cli2 only — no runtime deps).
- `.pre-commit-config.yaml`, `.github/workflows/ci.yml` — pre-commit hooks (large files, merge conflicts, private keys, lint) run locally and in CI on push and PR.
