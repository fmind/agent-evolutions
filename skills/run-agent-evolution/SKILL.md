---
name: run-agent-evolution
description: Run phase of an agent evolution — plan a batch of up to `budget.parallel` variants, dispatch sub-agents in parallel to materialize and evaluate them, learn from results, repeat until budget / wall-clock / plateau. Writes `winner` on a clean stop. Requires an evolution captured via `/new-agent-evolution`; `/apply-agent-evolution` lands the winner afterward.
---

# run-agent-evolution

Drive the genetic loop for `<id-or-slug>`. State is derived from `evolution.yaml` field presence — re-read between phases.

## 0. Dispatch

Resolve the argument:

- Pure integer → match the leading `<id>` under `.agents/evolutions/<id>-*/`.
- String → match a slug fragment; on multi-match pick the lowest `<id>` and note in chat.

Read `evolution.yaml` (validated against `evolution.schema.json`). Branch:

- Directory missing → `"No evolution found for <arg>. Run /new-agent-evolution <title> to capture one."` Stop.
- `applied` set → terminal. `"Evolution <id> already applied (winner: <variant_id>)."` Stop.
- `winner` already set → `"Winner already picked (v<n>). Run /apply-agent-evolution <id> to land it."` Stop.
- Otherwise → continue.

Each iteration is one **generation** — a batch of up to `budget.parallel` variants planned, executed, recorded together. The generation boundary is where the loop _learns_.

## 1. Resume sweep

On entry, reset any `running` variants without a `result.json` to `pending` and re-dispatch them. Variants with `result.json` already on disk → ingest immediately and mark `evaluated`.

## 2. Stop conditions (first match wins)

- `len(variants) >= budget.max_variants` → stop, pick winner.
- `budget.max_minutes` set and elapsed ≥ it → stop, pick winner.
- `budget.plateau_generations` set and top score has not improved over the last N gens (need ≥ 2 full gens) → stop, pick winner.
- `gen >= 2` and zero eligible variants → abort; gate spec is likely broken.

## 3. Plan the next batch

Compute `gen = max(variant.generation) + 1` (or `1`). `batch_size = min(parallel, max_variants - len(variants))`.

**Generation 1 — seed for diversity.** For each seed, pick a distinct _dimension to vary_ (algorithm, data structure, library, prompt style, control flow). Write a one-line falsifiable hypothesis and a 5–15 line approach concrete enough that a sub-agent can implement it without re-deriving. `parents: []`.

**Generation 2+ — mutate, cross, explore.** Read survivors (status `evaluated`, all gates passing, sorted by the composite score from §5). Mix per your judgment: mutations (`parents: [v_n]`), crossovers (`parents: [v_a, v_b]`), and one or two explores (fresh dimension; `parents: []`). When all prior variants failed gates, do not propagate them — diagnose and seed fresh.

Append each new variant to `evolution.yaml.variants[]` with `id: v<n>`, `generation`, `parents`, `hypothesis`, `approach`, `status: pending`. Add a one-line entry to `EVOLUTION.md` §Variants: `- v<n> (gen <gen>, parents <ids or "—">) — <hypothesis>`.

## 4. Materialize and dispatch

For each `pending` variant in this generation:

1. Workspace at `.agents/evolutions/<id>-<slug>/variants/<vid>/workspace/`. Per `scope.kind`: `worktree` → `git worktree add <path> HEAD` (fail loudly on dirty repo); `files` → copy each `scope.include` path preserving structure.
2. Set `status: running`, `workspace: <path>`. Save yaml once before dispatch.

Spawn one sub-agent per variant in **one message** (parallel Agent tool calls). Each prompt is self-contained: workspace absolute path (sub-agent's cwd), the §Brief section of `EVOLUTION.md` verbatim, hypothesis, approach, every gate (`G<id>: <description> | cmd: <cmd>`), every rubric axis (`R<id>: <description> | direction | cmd | extract`), and the **output contract** — write `result.json` one directory above the workspace (i.e. `.agents/evolutions/<id>-<slug>/variants/<vid>/result.json`), validated against `result.schema.json`:

```json
{
  "status": "evaluated",
  "gates": [{ "id": "G1", "passes": true, "output": "tsdown ✓" }],
  "rubric": [{ "id": "R1", "value": 98.6 }]
}
```

Hard rules for the sub-agent: implement the approach in the workspace; run gates in declared order; if any gate fails, write `status: "evaluated"` with that gate's `passes: false` and omit `rubric`; if all pass, run rubric measures, extract per `extract`, write `status: "evaluated"` with both arrays. Don't modify files outside the workspace. Don't commit. Don't push. The chat reply is advisory — `result.json` is the contract.

When sub-agents finish, read each `result.json`. Missing file → `status: errored`, retry **once**. Malformed JSON → `errored`, retry once. Gate failures and `killed` are honest signals, never retried. Set `status`, `gates`, `rubric` on the matching variant; bump `updated_at`; save.

## 5. Score, refresh, maybe stop

After the batch, recompute §Results in `EVOLUTION.md` from `evolution.yaml`. The yaml stores raw inputs only — composite scores and ranks are derived on read.

Eligible variants: `status == evaluated` AND every `gates[].passes == true`. For each rubric axis, rank eligible variants by `direction` (best → 1, worst → n; average ranks for ties). Normalize each rank to [0, 1] with `(n - rank) / (n - 1)` (n = 1 → 1.0). Composite = weighted mean of normalized ranks across axes (default weight 1.0). Render `## Results (after gen <N>)` as a Markdown table with columns `Rank | Variant | Gen | Parents | Score | <R1..Rk> | Notes`, sorted by composite descending (tie-break on lower `id`); append failed variants below with score `—` and Notes showing failed gate ids (`G2 ✗`) or non-evaluated status.

If no stop condition fired, loop back to §2.

**Pick the winner (when stopping).** Pick rank 1 from §5; tie-break on lower `id`, then shorter `parents[]`. Write to `evolution.yaml`:

```yaml
winner:
  variant_id: v<n>
  rationale: |
    <one paragraph — composite score, axis-by-axis comparison vs runner-up,
    any caveats from harness or wall-clock cap.>
```

Refresh §TL;DR in `EVOLUTION.md` (Winner, Coverage, Next: `/apply-agent-evolution <id>`).

If no eligible winner exists, don't write `winner`. Stop with a chat sentence naming the failing gates and recommending the user revise or relax them.

## 6. Hand off

End with one sentence: generations run, what changed, next pointer. On a clean stop with a winner: `/apply-agent-evolution <id>`. On no-winner: revise gates and re-run `/run-agent-evolution <id>`.

**Invariants.** Every variant has a unique `id` and a generation strictly greater than its parents'. `parents[]` references only ids already in `variants[]`. `winner.variant_id` is set iff a stop condition fired with at least one eligible variant.
