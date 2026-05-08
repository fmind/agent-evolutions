---
name: apply-agent-evolution
description: Apply phase of an agent evolution — diff the winner workspace against the live repo, get explicit user confirmation, copy files in, optionally re-run gates against the live tree. Writes `applied`. Requires `winner` set by `/run-agent-evolution`.
---

# apply-agent-evolution

Land the winner of `<id-or-slug>` into the live repo. Read-only until the user confirms.

## 0. Dispatch

Resolve the argument:

- Pure integer → match the leading `<id>` under `.agents/evolutions/<id>-*/`.
- String → match a slug fragment; on multi-match pick the lowest `<id>` and note in chat.

Read `evolution.yaml` (validated against `evolution.schema.json`). Branch:

- Directory missing → `"No evolution found for <arg>. Run /new-agent-evolution <title> to capture one."` Stop.
- `applied` already set → terminal. `"Evolution <id> already applied (winner: <variant_id>)."` Stop.
- `winner` not set → `"No winner picked yet. Run /run-agent-evolution <id> first."` Stop.

## 1. Pre-flight

Confirm `winner.variant_id` references a variant with `status: evaluated` and every gate passing — otherwise stop. Confirm `git status --porcelain` is clean over `scope.include` paths; on overlap, list conflicts and stop (commit, stash, or revert first).

## 2. Compute the diff

Branch on `scope.kind`:

- `worktree` — `git -C <workspace> diff --no-index --stat HEAD <live>` if the workspace shares git history; otherwise plain `diff -r`. Restrict to `scope.include`, exclude `scope.exclude`.
- `files` — for each `scope.include` path, diff workspace copy against live file. Treat missing-on-one-side as add/delete.

Surface `git diff --stat` summary plus per-file diff body. If > 400 lines combined, show the stat and offer the full diff on request.

Reject paths under `scope.exclude` — list offenders and stop.

## 3. Confirm and apply

Wait for explicit confirmation ("looks good", "apply", "go"). On abort, leave the evolution as-is.

Copy each changed file (create parent dirs); `rm` deleted ones. Don't run formatters — gates already enforced quality. Capture `git status --porcelain`; write to `evolution.yaml`:

```yaml
applied:
  at: "<now>"
  files_changed: [...]
```

Bump `updated_at`. Refresh §TL;DR in `EVOLUTION.md` (Applied, files changed).

## 4. Verify (recommended)

Run each `gates[].cmd` against the live repo. Report results in chat (not persisted). If any fail, name the gates and the relevant output. The user investigates.

Variant workspaces stay as audit trail — the user removes them with `git worktree remove` (kind: worktree) or `rm -rf` (kind: files) when ready.

## 5. Hand off

End with one sentence: files changed, gate re-verification result, next pointer (commit and PR).

**Invariants.** Writes happen only after explicit confirmation; the diff displayed equals the diff applied; `applied.files_changed` is exactly the set of paths whose live content changed.
