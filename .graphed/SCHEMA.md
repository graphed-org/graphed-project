# `.graphed/` state backing — schema & update discipline

Live progress state for the `graphed` project. Defers to the root `CLAUDE.md`; the project plan
(`graphed-project-plan-gated.md`, Part B) always wins. This state is the durable backing the
deterministic orchestrator reads/writes; until the orchestrator is built, **maintain it by hand and
keep it honest** — it must reflect git + CI evidence, never an optimistic claim.

## Files

- **Meta repo (`graphed-project`) `.graphed/state.json`** — project-level rollup: pipeline status,
  global circuit breakers (§B.7), the repo inventory, and one entry per milestone (status, phase,
  which repos, what it's blocked on). One source of truth for "where is the whole project."
- **Each repo's `.graphed/state.json`** — authoritative per-repo detail: for every milestone that
  repo owns, the phase, freeze tag, budget, the seven mechanical gates (§B.3), the §B.4 iteration
  metrics, reviewer status, and pointers to `attempts.md` / `incident.md` / `disputes/`.
- **`.graphed/<Mx>/attempts.md`** — created when a milestone enters `IMPLEMENTING`; one structured
  entry per implementer iteration (§B.4). Survives context resets.
- **`.graphed/<Mx>/incident.md`** — written on an L2 PAUSE (§B.7); triggering signal + metric
  history + last diff + integrity scan.
- **`.graphed/<Mx>/disputes/<test_id>.md`** — a filed Test Dispute (§B.5).

## Phases (§B.1)

`PENDING → DECOMPOSE → TEST_AUTHORING → TEST_SANITY → FROZEN → IMPLEMENTING → REVIEW → DONE`
Side states: `TEST_DISPUTE`, `ESCALATED`, `PAUSED`, `ABORTED`.

## Gate fields (§B.3) — values: `true` (green) / `false` (red) / `null` (not yet run)

`frozen_tests`, `coverage` (≥90% line+branch diff, from the frozen suite), `lint` (ruff+clippy),
`types` (mypy --strict), `determinism`, `benchmark` (where defined), `integrity_scan`.
A reviewer `APPROVE` may be recorded **only when every gate is green** (§B.3 #4).

## Metric fields (§B.4)

`pass_count`, `fail_set_hash`, `tree_hash` (excl. `tests/frozen/**`), `diff_lines`, `coverage`,
`benchmark_ok`, `lint_ok`, `types_ok`, `determinism_ok`, `integrity_ok`, `tokens_spent`,
`wall_clock_s`, `iteration_index`.

## Update discipline

1. **On every state-machine transition**, update the owning repo's `.graphed/state.json` (phase,
   gates, metrics, reviewer), then update the rollup in the meta repo's `.graphed/state.json`, then
   bump both `updated_at` timestamps (UTC ISO-8601).
2. A sub-repo change is committed and pushed in that repo first, then the **submodule pointer** is
   advanced and committed in `graphed-project`.
3. **Never** set a gate to `true` without the CI artifact that proves it. Never mark a milestone
   `DONE` without all gates green **and** a recorded reviewer `APPROVE`.
4. Integrity incidents (§B.6) and budget overruns (§B.5 #6) set `pipeline_status: "PAUSED"` at the
   project level and require an explicit human resume — no agent self-clears a PAUSE.
5. When a `[lazy]` package repo is created for its milestone, flip its `repos.<name>.created` to
   `true` (and `visibility`) in the meta state, and add its `.graphed/state.json`.
