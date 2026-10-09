# ref: ssg Debugger Integration — load when `sigmashake-gov` ∈ AFFECTED_SUBPROJECTS

The debugger is the unified `ssg://` resolver inside the eval daemon. The CLI, the dashboard `/debug`, and MCP all see identical envelopes (`{uri, fetched_at_ms, snapshot_hash, lens, data}`). `?since=<hash>` returns `{unchanged: true}` for cheap polling. A 30 s stale-cache fallback keeps reads alive across daemon hiccups.

**Go-parity gate.** The shipped daemon, CLI, and server are **Go** (`go/`). A TS edit in `src/` to eval, engine, CLI, daemon, or server behavior is only half the change. Port it into `go/` (`github.com/sigmashakeinc/ssg-go`, pure-Go in-process engine). Prove it with `bun run gate go test -C go ./...` (Phase 4). The surfaces below read the *running Go daemon*. They show shipped behavior, not the TS reference.

**Precondition for every debug call:** the daemon is up with frame emission.
```bash
ssg daemon --status || SSG_DEBUG_ENABLED=1 ssg daemon
ssg debug snapshot ssg://frames/recent --limit=1   # sanity: frames flowing
```

**Token rule:** consume these snapshots. Record the one verdict line, hash, or failing stage. Then drop the body. Never keep a full snapshot or frame dump in orchestrator context.

| Surface | Used in | Purpose |
|---|---|---|
| `ssg debug snapshot ssg://rules/list?filter=<area>` | Phase 1 | Inventory existing rules before edits |
| `ssg debug snapshot ssg://daemon/state` | Phase 1 + 4 | Baseline `snapshot_hash`; diff post-impl |
| `ssg debug snapshot ssg://audit/recent?limit=20` | Phase 1 | Real recent eval activity = realistic test surface |
| `ssg debug follow --tool=<X> --rule=<Y>` (background) | Phase 3 | Live JSONL frame stream — trace-driven dev replaces `console.log` |
| `ssg debug snapshot ssg://rules/<id>` | Phase 3 | Inspect parsed rule for the new feature |
| `ssg debug watch <uri> --since=<hash> --interval=500` | Phase 4 | Delta-poll; `{unchanged: true}` when nothing moved |
| `ssg debug rca --markdown` | Phase 4 (on failure), Anti-Loop | Pre-computed root cause: hypotheses + evidence + frame ids |
| `ssg debug snapshot ssg://frames/<trace_id>` | Phase 4 (on failure) | Full per-stage timeline for the failing trace |
| `ssg debug snapshot ssg://frames/<trace_id>/<frame_id>` | Phase 4 (on failure) | Single-stage detail (error, payload preview) |
| `ssg debug snapshot ssg://audit/<id>` | Phase 4 (on failure) | Single audit row — input, decision, duration |
| `ssg debug snapshot ssg://daemon/metrics --lens=gauges` | Phase 4 + Reporting | Latency + counters vs Phase 1 baseline |
| `ssg debug snapshot ssg://native/stats` | Phase 4 (sidecar only) | Native span buffer — only under `SSG_EVAL_ENGINE=sidecar`; default Go in-process engine doesn't emit it |
| Dashboard `/debug` (RCA/Timeline/Watch/State/Metrics) | Manual review | Human view of the same envelopes |

**`DebugFrame` stages** (`hook-ingress → parse → match → ffi-call → decide → audit → respond`). The failing stage in the RCA pack names the layer:
- `parse` → parser (`src/engine/parser.ts` / `go/internal/goengine`).
- `match` → evaluator (`src/engine/evaluate.ts` / `go/internal/goengine` + `go/internal/nfaregex`).
- `ffi-call` → sidecar (only under `SSG_EVAL_ENGINE=sidecar`).
- `decide` → rule resolution.
- `audit` → SQLite layer.

## Phase steps (gov in scope)

**Phase 0 — signal capture (fire in parallel, <5 s, eliminates speculative Phase 1 reads):**
```bash
ssg debug rca --markdown                          # existing failure context, ≤3 KB
ssg debug snapshot ssg://frames/recent --limit=1  # confirm daemon + frames flowing
```
If the RCA pack is non-null, record `trace_id`, `hypothesis[0].cause`, and `severity`. Narrow the Phase 1 impact set to the implicated files only.

**Phase 1 baseline:** record `snapshot_hash` from `ssg://daemon/state`, `ssg://rules/list?filter=<area>`, and `ssg://audit/recent?limit=20` for the Phase 4 diff. For any eval, CLI, daemon, or server behavior change, the impact set **includes the `go/` mirror**.

**Phase 3:** the impact set can include an eval-path file (`src/engine/`, `src/commands/hook.ts`, `src/commands/eval.ts`, `src/commands/eval/`, `go/internal/goengine/`, `go/internal/nfaregex/`). If it does, run `ssg debug follow --tool=<X> --rule=<Y>` in the background (`run_in_background: true`). Verify the new path with streaming frames, not `console.log`. If the stream is empty, check `SSG_DEBUG_ENABLED=1` again. Skip this for config, server, SPA, non-eval-CLI, and docs changes (no eval frames). Port the behavior into `go/`. `bun run gate go build -C go ./...` must succeed (`power_go_test_ungated` DENYs a raw ungated `go build`).

**Phase 4 triage (on `gov:test`/`go test` failure — REQUIRED before code-spelunking):**
1. `ssg debug rca --markdown` — read the pack first. Note `id`, `severity`, `summary`.
2. `ssg debug snapshot ssg://frames/<trace_id>` — identify the failing stage.
3. If the pack points at one frame, run `ssg debug snapshot ssg://frames/<trace_id>/<frame_id>` for the full error and payload.
4. Form ONE hypothesis from the pack's `hypothesis[]` and `evidence[]`. Do not open source until steps 1–3 are done.
5. Patch and re-run. If the next failure has a different `trace_id`, repeat from step 1.

**Phase 4 gate add-ons:**
- `bun run gate go test -C go ./...` shows 0 failures (if Go behavior changed).
- `ssg://daemon/metrics` after the test, compared with the Phase 1 baseline, shows no unexplained `error_count`/`block_count` deltas.
- Record the RCA pack `id` if you triaged any failure.
- Stop the background `ssg debug follow` before Phase 5 (`pgrep -f "ssg debug follow"` is empty).

**Anti-Loop:** if you are stuck in Phase 3/4 on a gov change, invoke `ssg debug rca --markdown` BEFORE the 3rd retry. If you loop past it, the result is `STUCK`. Before the 3rd retry on any gov/desktop failure, run `ssg debug analyze-sessions --last=24h`. If it flags `retry-loop`, `blind-build`, or `state-speculation` for this session, act on it before you retry.
