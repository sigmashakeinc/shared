# ref: Cross-OS mesh orchestration — load when cross-OS coordination is in scope

**Trigger condition:** Load this file if any of these is true:
- The task spans ≥2 desktop shells.
- The task needs Win/Mac/Linux parity, build, or test (for example, to drive `sigmashake-workspace` to 100% cross-OS parity).
- `sigmashake-mesh`, `sigmashake-orch`, or `sigmashake-cicd` is in `AFFECTED_SUBPROJECTS`.

---

## Two primitives

**mesh `/dispatch` — remote cross-OS governed Claude Code session**

`POST http://127.0.0.1:7929/dispatch` — body `{repo, prompt, requires?, sha?, govProfile?, dedupKey?}` → `ScheduleOutcome`. The planner routes by `requires` capability to the registered runner on the peer of the owning OS. The runner spawns a governed interactive Claude Code session through tmux (`-L sigmashake-mesh`). It never uses `claude -p`. The runner streams `RunnerEvents`: `Accepted`, `Frame(stage/preview)`, `ToolCall(tool/decision)`, `ApprovalRequest(id/tool/target)`, `Result`, `Heartbeat`, `RunError`. Start the planner with `bun run mesh:planner`. Start the runner (gRPC 127.0.0.1:50070) with `bun run mesh:runner`.

Built-in safety:
- Capability routing.
- Power-budget gating. It defers Linux dispatch when the box is not power-safe.
- Concurrency limits for each machine.
- Circuit breaker (3 fails → open, 60s cooldown).
- SSG governance approver with a PERMANENT PHYSICAL FLOOR. It never auto-approves GPU/CPU power manipulation, secret exfil, or load-bearing deletes.
- rsi monotonic decision-log sink (HMAC, fail-soft).

**HONEST-RED (state this plainly in every report):**
- The live cross-machine mTLS connect is `test.skip` under Bun (http2 ALPN limit). The mac/win peers can be offline at run time. Mesh is effectively host-OS-only until the peers and mTLS are live.
- The `Steer(task_id, guidance)` RPC is defined, but the steer branch of the runner is deferred.
- Autonomous triggers are INERT until `bun run mesh:triggers`.
- SSH-exec transport: a cross-OS leg to a peer without tmux (Windows QEMU VM at `sigmashake-win` / 100.92.244.45) dispatches through `src/ssh-transport.ts`. It uses `ssh -F ~/.ssh/sigmashake/config -o BatchMode=yes -o ConnectTimeout=8 -o ConnectionAttempts=1 sigmashake-win <cmd>`. This applies when the `transport` field of the machine is `'ssh'`. Governance screens the physical floor before the spawn.
- Fail-soft: if SSH is offline, the result is `RunError/BLOCKED`. Never fake success.

---

**SigmaShake Terminal (codename GHOSTMUX) — local governed observable PTY**

REST + WS through the ssg daemon: `GET/POST /api/terminal/sessions`, `DELETE /api/terminal/sessions/{id}`, `GET /api/terminal/agents`, `GET/POST/PUT /api/terminal/workspaces[/{id}]`, WS `/api/terminal/connect`. The agent launcher injects into a governed PTY shell for these agents: claude, codex, aider, opencode, cursor-agent, pi. The terminal is embedded in `sigmashake-workspace` through `src/client/components/TerminalView.tsx`. Go implementation: `sigmashake-gov/go/internal/terminal/`. SPA: `sigmashake-gov/src/client/terminal/`.

**HONEST-RED:** Terminal is LOCALHOST-ONLY (no remote tunneling). It has NO mesh integration today. It does NOT drive remote peers. Its role is this: the operator watches and drives the LOCAL leg in an observable governed PTY. It is not a remote driver.

---

## Cross-OS procedure (mirror orch/cicd pattern)

### (a) Classify legs
For each cross-OS requirement, identify these items:
- **Leg** — a discrete unit of work (build, attest, parity-column verify, desktop-shell smoke).
- **Owning OS/peer** — Linux (local), macOS (mesh peer), or Windows (mesh peer).
- **Capability requirement** — the `requires` field for the mesh dispatch (for example, `"os:macos"`, `"os:windows"`).

Record the leg table before the preflight. This is an example for workspace cross-OS parity:

| Leg | Owning OS | Capability | Primitive |
|---|---|---|---|
| Linux build + test + parity-column | Linux (local) | — | gate-wrapped local |
| macOS Wails/Swift desktop-shell attest | macOS peer | `os:macos` | mesh `/dispatch` |
| Windows C#/.NET desktop-shell attest | Windows peer | `os:windows` | mesh `/dispatch` |

### (b) Preflight peer liveness — BEFORE any dispatch
```bash
# Planner health
curl -s http://127.0.0.1:7929/health

# Registered runners (real — returns all Machine rows from DuckDB, incl. SSH peers)
curl -s http://127.0.0.1:7929/runners
```
For each remote leg, confirm that its owning peer is in the runners list with status `ready`. If a peer is not in the list, or is not `ready`, mark that leg `BLOCKED (peer offline)` immediately. Do NOT dispatch and wait for a timeout.

For Windows peers with `transport: 'ssh'`, do these checks:
1. Confirm that the `ssh_host` field of the peer is `sigmashake-win`.
2. Confirm that SSH is reachable (`ssh -F ~/.ssh/sigmashake/config -o BatchMode=yes -o ConnectTimeout=8 -o ConnectionAttempts=1 sigmashake-win echo ok`).

If a Windows SSH peer is not in `/runners`, the `MESH_PEERS` env is unset on the planner. Set it before you check again.

### (c) Dispatch each remote leg
`govProfile` is **MANDATORY** on every remote dispatch. Remote legs run on peer hardware. They must carry the same governance envelope as local work.

```json
POST http://127.0.0.1:7929/dispatch
{
  "repo": "sigmashake-mono",
  "prompt": "<leg-specific prompt>",
  "requires": "os:macos",
  "sha": "<pinned HEAD sha>",
  "govProfile": "default",
  "dedupKey": "<run-id>-mac-parity"
}
```

Pin `sha` to the current `HEAD`. Then the remote leg operates on the same tree. Remote legs run on peer hardware. They are power-safe by offload (peer PSU, not this box). Any LOCAL leg still goes through the build, test, and push gates of this box (`bun run gate <cmd>` / `bun run gate:test <cmd>`).

Collect `RunnerEvents` for each leg. Record `Result` or `RunError` for each leg. Do not aggregate until all legs have settled or are marked `BLOCKED`.

### (d) Aggregate RunnerEvents per leg
Stream events from the planner, or poll `/dispatch/<jobId>/status`. Record these items for each leg:
- `Result` → extract the structured summary (files, typecheck/test exit, coverage `total:`, parity columns attested).
- `RunError` → extract the error class and message. Classify it as retriable or `BLOCKED`.
- No event within the circuit-breaker window → `BLOCKED (circuit open or peer timeout)`.

Token discipline: record the verdict line for each leg, not the full event stream.

### (e) Reconcile — honest per-leg status
The task is NOT done until **every leg** returns `Result` OR is explicitly `BLOCKED (peer offline / mTLS not live / circuit open)`. Never fake-green a leg that did not run.

The reconciliation table is required in the Phase 4 gate and the Final Report:

| Leg | Status | Evidence |
|---|---|---|
| Linux local | PASS / FAIL | test:all exit, coverage total: |
| macOS peer | PASS / BLOCKED (peer offline) | Result summary or reason |
| Windows peer | PASS / BLOCKED (peer offline) | Result summary or reason |

---

## Honest-red fallback

If a peer is offline or mTLS is not live:
1. Run the Linux leg locally (gate-wrapped as normal).
2. Mark the Win/Mac legs `BLOCKED (peer offline — mesh mTLS not live)` and give the reason.
3. Show these legs in the /mono Final Report as `BLOCKED` with the peer state. Never show them as passing or skipped.
4. Keep the readiness attestation in `production-readiness.json` at `partial` / `unmet` for the desktop-shell columns that did not run. Never write `met` for a column that a peer did not attest. The operator attests manually when the peers are live (same path as the `sigmashake-desktop-{mac,windows}` honest-red in `ref/coverage.md`).

---

## Terminal's role (local leg only)

Use `GET/POST /api/terminal/sessions` and WS `/api/terminal/connect` to launch and observe the LOCAL leg as a governed PTY session. The operator can watch or drive the local Claude Code session live through GHOSTMUX. This is the observable local surface. It does NOT replace mesh dispatch to remote peers. It does NOT drive them. Keep the two primitives separate in every plan and report.

---

## Worked example — sigmashake-workspace 100% cross-OS parity

**Goal:** drive `sigmashake-workspace` to 100% parity across all three desktop shells (Wails/Go Linux, Swift macOS, C#/.NET Windows).

1. **Classify legs.** There are three legs, as in the table above.
2. **Preflight.** Run curl on `/health` and `/runners`. If the macOS peer is offline, the mac leg is `BLOCKED` before any dispatch.
3. **Linux leg (local, gate-wrapped):**
   - `bun run gate bun --cwd sigmashake-workspace run build:plugin` (the plugin is the canonical local surface).
   - `bun run gate:test bun --cwd sigmashake-workspace run test:coverage`. Make sure that coverage is 90/90/90/80.
   - Parity catalog Linux column: the `workspace-principal-engineer` verifies or updates the Linux entries in `src/platform-parity-catalog.ts` and `src/harness-parity-catalog.ts`.
   - Option: observe through Terminal. Send `POST /api/terminal/sessions` with agent `claude`. Watch GHOSTMUX.
4. **macOS leg (mesh dispatch, if the peer is live):**
   ```json
   { "repo": "sigmashake-mono", "prompt": "Verify sigmashake-workspace parity catalog macOS column: build:plugin passes, Swift desktop-shell smoke passes, update src/platform-parity-catalog.ts macOS entries to honest attested state. Return structured summary.", "requires": "os:macos", "sha": "<HEAD>", "govProfile": "default", "dedupKey": "wsp-parity-mac" }
   ```
   Collect `Result`. Extract the parity column summary.
5. **Windows leg (mesh dispatch, if the peer is live):** use the same shape with `requires: "os:windows"`.
6. **Reconcile.** Build the table for each leg. If any leg is `BLOCKED`, the desktop-shell parity column in `production-readiness.json` stays `partial`. Show this honestly in the Final Report.
7. **Platform-parity KV write.** Each attested peer writes its OS column to `cell:<feature_id>:<os>` with `POST /api/workspace/parity-matrix` (bearer `PARITY_AGENT_TOKEN`). A `BLOCKED` peer does NOT write. The merged cloud view shows the last known value. This value can be old. Note this in the report.

---

## Invariants

- **govProfile mandatory** on every `POST :7929/dispatch`. Never dispatch without it.
- **Remote = peer hardware, power-safe by offload.** The own PSU and power gates of the peer govern remote legs. Local legs still go through the gates of this box. Never lift the caps of this box, whatever the remote legs do.
- **No new free/anonymous path to a value surface.** Mesh dispatch does not open a new entitlement path. Governed Claude Code sessions on peers are subject to the same SSG governance and license gate.
- **Peer circuit breaker.** After 3 consecutive failures, the circuit opens (60s cooldown). A peer with an open circuit is `BLOCKED`. Do not retry it immediately. Show it in the report.
- **`bun run mesh:planner` / `mesh:runner` start.** If the planner is not running, `/health` does not respond. Start it before the preflight, not during the preflight.
- **Orch/cicd pattern.** This mirrors `sigmashake-orch` (Linux controller + HMAC-signed HTTP dispatch to Tailscale peers, `/retry` + `/preflight` for each leg, honest status for each leg). It also mirrors `sigmashake-cicd` (conductor above orch, polls and reconciles, records into the workspace distribution-runs tracker). If you are not sure, read how those services classify, dispatch, and reconcile. The shape is the same.
- **`mesh:terminal` for live visibility.** Run `bun run mesh:terminal --local <session>` to attach a visible terminal to a live local tmux mesh session. Run `bun run mesh:terminal --ssh sigmashake-win --cmd '<cmd>'` to open a live SSH session to the Windows peer in a titled terminal window. Emulator fallback: cosmic-term → alacritty → xterm. This is power-safe (terminal only, no build load).
