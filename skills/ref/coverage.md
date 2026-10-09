# ref: Coverage ≥90% gate — load when the Coverage floor is armed (impact set touches a critical-production surface)

**Critical-production** = the shipped, customer-facing runtime. It has four parts: the **gov Go runtime** (`sigmashake-gov/go/`), the **Worker value-surface fleet**, the **three desktop shells**, and **`sigmashake-workspace`** itself. Local-only operator tooling (tokens, pitstop, trends, emotes, tutorials, screenshots) is **advisory**. Measure and report its coverage, but never gate on it.

**Thresholds (single source of truth: `sigmashake-workspace/vitest.config.ts` → `thresholds`; do not invent new numbers):** lines 90 / statements 90 / functions 90 / branches 80. The bar applies to the **whole affected package or subproject**, not only the diffed lines. A small edit to a critical package below 90 arms a backfill of that package to ≥90.

**A missing coverage command is a gate failure, not a skip.** If the affected critical subproject has no coverage script, the team must wire one (script + 90/90/90/80 threshold) before the gate can pass.

**Run discipline.** Coverage is an all-core instrumented full-test pass. It is the heaviest single burst in the pipeline. Run it **wrapped in the cross-session gate** (`bun run gate <cmd>`) and **serially** (`TEST_JOBS` default). **Never run it while GPU inference is live** (PSU OCP). `power_bun_test_ungated` and `power_make_cover_ungated` DENY a raw, ungated run.

| Surface | Coverage command (gate-wrapped) | Pass condition |
|---|---|---|
| `sigmashake-workspace` | `bun run gate bun --cwd sigmashake-workspace run test:coverage` | Non-zero exit on miss — `vitest.config.ts` enforces 90/90/90/80 |
| `sigmashake-gov` — Go (shipped runtime) | `bun run gate make -C sigmashake-gov/go cover` → read the final `total:` line | `total:` ≥ 90.0% — **the skill asserts this; the Makefile only prints it** |
| `sigmashake-gov` — TS reference / SPA | `bun run gate bun --cwd sigmashake-gov run coverage` (`…run coverage:batch` for the full split) | lines/statements/functions ≥90, branches ≥80 |
| `sigmashake-desktop-linux` — Go internals | `bun run gate go test -C sigmashake-desktop-linux ./internal/... -covermode=atomic -coverprofile=/tmp/dl.out && go tool cover -func=/tmp/dl.out \| tail -1` | `total:` ≥90% (reuse `dlcov-*`) |
| Worker value-surface fleet (vitest) | `bun run gate bun --cwd <subproject> run test:coverage` | 90/90/90/80; if the script is missing, wire it first — gate failure until then |
| `sigmashake-desktop-{mac,windows}` | host-only — cannot measure on this Linux box | honest-red: operator attests via the readiness manifest; never fake-green |

**Token rule:** read only the `total:` line or the threshold-summary line of a coverage run. Never keep the full per-file coverage table in orchestrator context.

**Reuse the existing `<prefix>cov-*` coverage teams** (`wscov-*` workspace, `dlcov-*` desktop-linux, `twcov-*` accounts). `<prefix>cov-coverage-analyst` is the read-only acceptance gate. Author a new team only when none fits. Sometimes the package cannot reach ≥90 within the scope of this change. In that case the result is **`BLOCKED`**, with the named uncovered packages and the percent (Anti-Loop). Never lower the threshold. Never exempt the package. Never mark it local-only to avoid the gate. Never write `met` into `production-readiness.json` without the real number.

**Phase 5/6 readiness attestation** (a critical-production feature shipped, and the floor was armed and passed): update the matching **`quality`** requirement in `.sigmashake/production-readiness.json`. Set `manual.status: "met"` with dated `evidence` that cites the **actual coverage number** (for example `"gov Go runtime 91.4% lines, verified 2026-06-18"`). Or use the `cert-signal` sentinel where one exists. Never hand-write `met` over a live cert-signal. Then run `bun --cwd sigmashake-workspace run generate` and stage `src/readiness.generated.ts` (generated — never hand-edit). Do not write `met` without the real number. `partial` or `unmet` is the honest state when the bar does not hold.
