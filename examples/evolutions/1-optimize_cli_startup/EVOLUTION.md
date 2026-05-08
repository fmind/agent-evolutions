# Evolution — `1-optimize_cli_startup`

## TL;DR

- **Winner:** `v7` — score 1.00. v7 (gen 2) hits 98.6 ms vs 142.0 ms (v2) and 184.2 ms (v1) on cold-cache `--version`; both gates green.
- **Coverage:** 24/30 evaluated · 4 generations.
- **Next:** `/apply-agent-evolution 1` to land the winner in the repo.

## Brief

> Loaded verbatim by every variant sub-agent. Short, declarative, no prose explaining _why_.

### Objective

Minimize cold-cache wall-clock startup of `mycli --version` while keeping every existing test green and the public CLI surface unchanged. The `--version` code path runs hundreds of times per shell session in CI; trimming it pays off across the fleet.

### Held fixed

- **Stack:** Node.js 22.18 + TypeScript 5.7 + tsdown bundler. No runtime swap.
- **Public surface:** `mycli --help`, `mycli --version`, `mycli build`, `mycli check` — flags, output schema, exit codes are immutable.
- **Dependencies:** Existing `dependencies` in `package.json`. No new top-level runtime deps (devDeps OK).
- **Tests:** Every test under `tests/` keeps passing — covered by gate G2.

### Variants may change

- `src/cli.ts` and any module imported during the `--version` code path. Lazy-load, defer, or inline.
- `src/index.ts` barrel — collapse, fan out, or replace with explicit imports.
- Bundler config (`tsdown.config.ts`) — chunking, treeshake settings, target.
- Internal data structures used at boot (config parsing, plugin registry).

### Scope (filesystem)

- **Workspace kind:** `worktree` — variants need a real tsdown build to measure end-to-end.
- **In scope:** `src/`, `tsdown.config.ts`, `package.json`.
- **Excluded from apply:** `tests/`, `docs/`.

### Gates

- **G1** — Build succeeds. `cmd: pnpm build`.
- **G2** — Every existing test passes. `cmd: pnpm test`.

### Rubric

- **R1** — Cold-cache wall-clock for `node dist/cli.js --version`, milliseconds (mean of 5 hyperfine runs). `direction: minimize` · `weight: 1.0` · `cmd: hyperfine --warmup 0 --runs 5 'node dist/cli.js --version'`.

### Budget

- `max_variants: 30` · `parallel: 6` · `plateau_generations: 2`.

### Out of scope

- Switching the bundler, package manager, or runtime.
- Refactoring command handlers beyond what the version path imports.
- Reducing the public CLI surface or moving features to subcommands not in the public list.
- Touching anything under `tests/` or `docs/`.

## Variants

- v1 (gen 1, parents —) — Lazy-load every command module behind `--help` / `--version` short-circuits.
- v2 (gen 1, parents —) — Inline the version string at build time so `--version` skips package.json IO.
- v3 (gen 1, parents —) — Replace the plugin-registry barrel with a generated map of import functions.
- v4 (gen 1, parents —) — Strip `core-js` polyfills from the cli entry chunk.
- v5 (gen 1, parents —) — Pre-bundle the arg parser as a single CommonJS file.
- v6 (gen 1, parents —) — Move dotenv loading behind a `--no-env` short-circuit.
- v7 (gen 2, parents v2) — v2 + bundle-time tree-shake the help-text module out of the version path.
- v8 (gen 2, parents v1) — v1 + skip plugin-registry init when `--version` is the first arg.
- … (24 evaluated total across 4 generations)

## Results (after gen 2)

| Rank | Variant | Gen | Parents | Score |    R1 | Notes |
| ---: | :------ | --: | :------ | ----: | ----: | :---- |
|    1 | v7      |   2 | v2      |  1.00 |  98.6 |       |
|    2 | v2      |   1 | —       |  0.50 |   142 |       |
|    3 | v1      |   1 | —       |  0.00 | 184.2 |       |
|    — | v4      |   1 | —       |     — |     — | G2 ✗  |
