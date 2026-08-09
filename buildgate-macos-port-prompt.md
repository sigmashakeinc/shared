# Build a cross-session build gate + live dashboard (macOS)

> **How to use this:** paste everything below the line into Claude Code, Codex, or
> Cursor, from inside the empty directory where the project should live. It is
> written to be self-contained — the recipient needs no access to the original
> Linux codebase. Adjust the two "you decide" spots (project name, port) first.

---

## The prompt

You are building a small, production-grade local tool on **macOS**. Read this
whole brief before writing any code. Do **Step 0** first — several design
decisions depend on what your machine actually has, and guessing wrong produces
a tool that looks like it works and silently doesn't.

### 1. What you are building, and why

Two pieces:

**A. `buildgate.sh` — a cross-session counting semaphore.**
A shell script that wraps any command and makes it wait its turn:

```bash
buildgate.sh cargo build --release
buildgate.sh go test ./...
BUILD_GATE_NAME=test buildgate.sh npm test
```

It is **not a daemon**. It is a set of lock files under a state directory, and
a kernel-backed advisory lock (`flock(2)`) on each one. Every shell, every
editor, every AI-agent session on the machine contends for the same N slots. If
all slots are held, the command **waits** — it does not fail — until a slot frees
or a timeout expires.

The problem it solves: a modern dev box runs several agent sessions at once, and
each one cheerfully kicks off an all-core build or test run. Individually fine;
simultaneously they saturate the machine, thrash swap, thermally throttle, and
make every session slower than running them one at a time would have been. Job
caps inside each tool (`-j`, `--jobs`) cannot bound the *aggregate* across
processes that don't know about each other. A shared lock can.

**B. A live dashboard service.**
An always-on local HTTP service on `127.0.0.1:<PORT>` that reads the gate's
state and renders it as a traffic light: who holds a slot right now, who is
queued, and what ran recently. Plus analytics over the event history. It is
**strictly read-only** — it can observe the gate but never influence
serialization.

### 2. Non-negotiables

These are the invariants. Violating any of them makes the tool worse than not
having it.

1. **The gate never corrupts a build.** If any part of the gating logic fails —
   unreadable file, missing binary, parse error — the command still runs. The
   worst acceptable outcome is "ran without waiting."
2. **The dashboard is read-only.** It reads state files and calls the gate's
   `--status-json` subcommand. It never writes a lock, never holds a slot, never
   calls the gate with anything but a read subcommand. A crashed dashboard must
   not be able to wedge a single build.
3. **Locks release on `SIGKILL`.** This is why it must be `flock(2)` and not a
   PID file you write yourself. Kernel-held locks vanish when the process dies,
   however it dies. Any scheme requiring cleanup code to run is wrong — agent
   sessions get force-killed constantly.
4. **Unknown state clamps, it does not widen.** See §5; this is the single most
   likely thing to get silently wrong in this port.
5. **No `sudo`, no root, no kernel extensions, loopback only.** Everything runs
   as the logged-in user.

### 3. Step 0 — probe the machine before designing

Run this and keep the output. It decides three implementation choices.

```bash
echo "bash:        $(/bin/bash --version | head -1)"
echo "bash5:       $(command -v /opt/homebrew/bin/bash /usr/local/bin/bash 2>/dev/null | head -1 || echo MISSING)"
echo "flock:       $(command -v flock || echo MISSING)"
echo "shlock:      $(command -v shlock || echo MISSING)"
echo "clang:       $(command -v clang || echo 'MISSING — run: xcode-select --install')"
echo "taskpolicy:  $(command -v taskpolicy || echo MISSING)"
echo "memory_pressure: $(command -v memory_pressure || echo MISSING)"
echo "gtimeout:    $(command -v gtimeout || echo MISSING)"
echo "arch:        $(uname -m)   ncpu: $(sysctl -n hw.ncpu)"
sysctl -n hw.perflevel0.logicalcpu hw.perflevel1.logicalcpu 2>/dev/null \
  | paste -sd'/' - | sed 's/^/P\/E cores:   /'
pmset -g therm 2>/dev/null | head -5
```

Three decisions fall out of it:

- **`flock` present?** (Homebrew `flock` formula, or a `discoteq/flock` tap.) If
  yes, use it. If no, build the 25-line C helper in §4.1 — do **not** fall back
  to a PID file.
- **bash 5 present?** Required — see §5, trap 1. If missing:
  `brew install bash`. Do not target `/bin/bash`.
- **Apple Silicon or Intel?** Changes the widen probe (§4.4) and the Homebrew
  prefix (`/opt/homebrew` vs `/usr/local`).

### 4. Part A — the gate script

#### 4.1 The lock primitive

macOS has the `flock(2)` **syscall** but ships no `flock(1)` **command**. If
Step 0 found no `flock` binary, write and compile this once at install time:

```c
/* gatelock.c — flock(1) work-alike for macOS.
   Build: clang -O2 -o gatelock gatelock.c
   Usage: gatelock [-n] <lockfile> <cmd> [args...]
          -n  non-blocking: exit 1 immediately if the lock is held. */
#include <fcntl.h>
#include <stdio.h>
#include <string.h>
#include <sys/file.h>
#include <unistd.h>

int main(int argc, char **argv) {
  int nb = 0, i = 1;
  if (i < argc && strcmp(argv[i], "-n") == 0) { nb = 1; i++; }
  if (argc - i < 2) {
    fprintf(stderr, "usage: gatelock [-n] <lockfile> <cmd> [args...]\n");
    return 2;
  }
  const char *path = argv[i++];
  int fd = open(path, O_RDWR | O_CREAT, 0644);
  if (fd < 0) { perror("open"); return 2; }
  /* Deliberately NOT O_CLOEXEC. The descriptor must survive execvp so the
     kernel holds the lock for the child's entire lifetime and releases it
     automatically when the child dies — including on SIGKILL. */
  if (flock(fd, LOCK_EX | (nb ? LOCK_NB : 0)) != 0) return 1;
  execvp(argv[i], &argv[i]);
  perror("execvp");
  return 127;
}
```

Because the helper `exec`s the child, the script's control flow inverts relative
to a naive port. Structure it as a **re-entrant two-phase script**:

```
buildgate.sh <cmd...>
  └─ pick a free slot N (probe: gatelock -n slot-N /usr/bin/true)
  └─ gatelock slot-N  "$0" --run-locked N -- <cmd...>
                       └─ (second invocation, now holding the lock)
                          write lease metadata, emit "acquire" event,
                          run <cmd...>, emit "release" event with rc
```

Probing a slot for the dashboard's benefit is `gatelock -n <slot> /usr/bin/true`
— exit 0 means it was free (and was released immediately), exit 1 means busy.

#### 4.2 Slot acquisition

- `N` slots, files `<statedir>/<gate>.slot-1` … `.slot-N`.
- Named gates via `BUILD_GATE_NAME` (default `build`). Same name = same light.
  Ship three: `build`, `test`, and `push` (the last with 1 slot
  unconditionally — its job is collision prevention, not load control).
  Independent names must not block each other.
- Scan slots in order, take the first free one. Nothing fancier is needed.
- Poll fast at first (~0.5 s for the first ~3 s, so a just-freed slot is picked
  up snappily), then back off to ~2 s.
- `BUILD_GATE_TIMEOUT` (default 3600 s). On timeout, **do not run the command**;
  exit with a distinct reserved code (75) so callers can tell "gate timed out"
  from "the build failed."
- Sanitize every numeric env var. A malformed value must fall back to the
  *strictest* setting, never the loosest.

#### 4.3 Environment contract

| Variable | Default | Meaning |
|---|---|---|
| `BUILD_GATE_NAME` | `build` | Which light. Same name = same lock set. |
| `BUILD_GATE_SLOTS` | `2` | **Ceiling**, not a guarantee. See §4.4. |
| `BUILD_GATE_TIMEOUT` | `3600` | Seconds to wait before giving up (exit 75). |
| `BUILD_GATE_DIR` | `~/.buildgate/locks` | State dir. See §5, trap 8. |
| `BUILD_GATE_LABEL` | caller's cmd | Human label recorded for the UI. |
| `BUILD_GATE_PROBE` | *(built-in)* | Shell command; exit 0 = safe to widen. |

#### 4.4 The widen probe — reframed for macOS

The original tool's probe answered "are the hardware power caps armed?" because
it ran on a workstation whose PSU tripped on CPU+GPU power co-spikes. **That
failure mode does not exist on your Mac** — Apple Silicon manages its own power
envelope and exposes no user-settable caps. Do not port that logic; port the
*shape* of it.

`BUILD_GATE_SLOTS` is a **ceiling**. Before widening past 1 slot, the gate asks
"is the machine healthy enough to run two heavy things?" On macOS the honest
signals are:

- **Memory pressure** — `memory_pressure -Q`, or
  `sysctl -n kern.memorystatus_vm_pressure_level` (`1` normal, `2` warn,
  `4` critical). Clamp to 1 slot at warn or worse.
- **Swap** — `sysctl -n vm.swapusage` gives total/used/free directly. Clamp
  above ~20% used.
- **Free memory** — `vm_stat` page counts × page size, against
  `sysctl -n hw.memsize`. Clamp below a floor (~8 GB free).
- **Thermal** — `pmset -g therm`; a `CPU_Speed_Limit` below 100 means the
  machine is already throttling. Clamp.

Cache the verdict for ~10 s in a file in the state dir (`<epoch> <verdict>|<why>`,
written tmp+rename). The probe runs on every status render; it must not spawn
four subprocesses per second.

> **This is the trap.** The Linux original *fails open* — an unreadable
> `/proc/meminfo` means "no opinion, allow widening" — because on Linux an
> unreadable procfs is a weird edge case. On macOS **every one of those files is
> absent**, so a literal port fails open on every check, always widens, and the
> gate does nothing while reporting green. **Invert it: on macOS, a probe that is
> unimplemented or unreadable clamps to 1 slot.** Fail-safe, and make the
> `reason` string say exactly which probe declined.

Optionally, confine the child's CPU footprint the way the original used
`taskset`: macOS has no CPU affinity API, but `taskpolicy -b <cmd>` runs a
command in the background QoS tier, which on Apple Silicon schedules it onto the
efficiency cores. It is the right analogue. Make it opt-in — it will slow builds
substantially.

#### 4.5 Read subcommands

`--status-json` is the dashboard's entire live interface. Emit exactly:

```jsonc
{
  "gate": "build",
  "slots": 2,             // the configured ceiling
  "ceiling": 2,
  "effective_slots": 1,   // how many the gate will actually use right now
  "widen": false,
  "reason": "memory pressure (warn)",   // why effective < ceiling; "" if widened
  "lock_dir": "/Users/you/.buildgate/locks",
  "in_use": 1,
  "slot": [
    { "n": 1, "busy": true, "pid": 1234, "start": "2026-08-09T10:00:00-0700",
      "label": "go test ./...", "cwd": "/Users/you/proj", "held_seconds": 45 },
    { "n": 2, "busy": false, "pid": null, "start": null,
      "label": null, "cwd": null, "held_seconds": 0 }
  ]
}
```

Parse lease metadata **only for busy slots**. A free slot's stale `.meta` file
is a lie — suppress it. Slot liveness comes from the live lock probe, never from
metadata. Also ship `--status` (human-readable) and `--help`.

#### 4.6 The event log

Append-only JSONL at `<statedir>/<gate>.events.jsonl`, one object per line:

```json
{"ts":"2026-08-09T10:00:00-0700","epoch":1754730000,"event":"acquire",
 "gate":"build","pid":1234,"slot":1,"label":"go test ./...",
 "cwd":"/Users/you/proj","waited":3,"ran":null,"rc":null,
 "widen":false,"reason":"memory pressure (warn)"}
```

- `event` ∈ `wait` | `acquire` | `release` | `timeout`.
- `widen`/`reason` are recorded on `acquire` (elsewhere `null` / `""`).
- `waited` on acquire, `ran` + `rc` on release.
- **Escape properly**: backslashes first, then quotes, then strip raw control
  characters. Labels contain arbitrary shell text and one bad line poisons the
  dashboard's parse.
- **Every emit failure is swallowed.** Logging must never be able to block or
  fail an acquire.
- The *dashboard* owns rotation (last ~20 000 lines), not the gate — the gate
  should never rewrite a file another process may be appending to.

### 5. macOS porting traps

Nine things that will bite. Most fail silently.

1. **`/bin/bash` is 3.2.57 (2007).** The idiomatic acquire loop uses
   `exec {fd}>"$file"` — automatic file-descriptor allocation, **bash 4.1+**. On
   3.2 it is a syntax error or, worse, misparses. Use `#!/usr/bin/env bash`,
   require Homebrew bash 5, and assert it at the top of the script:
   `((BASH_VERSINFO[0] >= 4)) || { echo "needs bash >= 4"; exit 2; }`.
   Nothing about a silent bash-3.2 misparse is easy to debug later.
2. **No `flock(1)`.** §4.1.
3. **BSD coreutils, not GNU.** `date -Iseconds` → `date +%Y-%m-%dT%H:%M:%S%z`.
   `stat -c` → `stat -f`. `readlink -f` → `cd "$(dirname "$f")" && pwd -P`.
   `sed -i` → `sed -i ''`. `nproc` → `sysctl -n hw.ncpu`. No `timeout` —
   use `gtimeout` (coreutils) or a background-and-kill pattern.
4. **No `/proc`.** No `meminfo`, no `pressure/memory`, no `stat`. §4.4.
5. **No `taskset`, no `nvidia-smi`, no `cpufreq`.** §4.4.
6. **launchd is not systemd.** §6.
7. **App Nap / timer coalescing** can stretch a background agent's 1 s poll into
   something much longer. Set `ProcessType=Interactive` in the plist if the live
   view feels laggy, and never assume your interval fired on time — derive
   freshness from timestamps, not from tick counts.
8. **TCC will silently break you** if state lives under `~/Documents`,
   `~/Desktop`, or `~/Downloads` — a LaunchAgent reading those triggers a
   privacy prompt it cannot answer, and gets an empty read. Put everything in
   `~/.buildgate/` or `~/Library/Application Support/`.
9. **Homebrew prefix differs**: `/opt/homebrew` (Apple Silicon) vs `/usr/local`
   (Intel). Resolve it at install time; never hardcode.

### 6. Part B — the dashboard service

Runtime: **Bun** (TypeScript, no build step, so a service restart picks up
edits). Node works if you prefer; the shape is identical.

#### 6.1 Module layout

Separate **pure logic** from **I/O** rigorously. This is what makes the thing
testable — the pure modules get real unit tests with injected clocks, and the
I/O adapter stays thin enough to eyeball.

| Module | Responsibility | Purity |
|---|---|---|
| `state.ts` | Parse events + status JSON → derive holders, queue, history | pure |
| `analytics.ts` | KPIs, throughput buckets, histograms, top commands | pure |
| `supervisor.ts` | Child restart policy, backoff, fault-intensity window | pure (inject clock/timers) |
| `health.ts` | Poll age + circuit state → `ok`/`degraded`/`wedged` | pure |
| `gate.ts` | Spawn `--status-json` w/ timeout + breaker; read/rotate events | I/O |
| `server.ts` | HTTP, SSE, wiring, signals | I/O |
| `cli.ts` | `report` / `health` / `analytics` headless output | I/O |
| `public/` | Dashboard: vanilla HTML/CSS/JS, inline SVG, zero deps | — |

#### 6.2 Endpoints

- `GET /healthz` — plain text `ok` (200) only when a recent poll succeeded and
  the breaker is closed; **503 + a short reason** otherwise. A brief 503 right
  after start, before the first poll lands, is correct.
- `GET /api/state` — the current snapshot (derived from `--status-json` +
  recent events). When the breaker is open, serve **last-known-good flagged
  degraded** rather than erroring. The live path never blocks.
- `GET /api/analytics` — aggregates over the event window: contention rate,
  wait/run p50/p95/max, timeout rate, fail rate, widened rate, throughput
  buckets, wait + duration histograms, top commands by gate-time, realized
  concurrency from interval overlap. Compute behind a ~5 s cache and poll it on
  a slower cadence — keep it off the 1 s live path.
- `GET /api/stream` — SSE. `event: hello` on connect, `event: state` with the
  JSON snapshot whenever it changes (1 s poll), periodic `: heartbeat` comments.
  The client falls back to polling `/api/state` on disconnect.
- `GET /` + `/app.js` + `/app.css` — the dashboard.

#### 6.3 The dashboard

Traffic light (green idle / red held), holder cards with label + cwd + pid +
elapsed, an ordered queue longest-waiting-first, a recent-runs table with
waited / ran / exit code, and an analytics section. Vanilla everything, inline
SVG, no dependencies, no bundler. It gets loaded and left open all day — keep
the DOM updates surgical rather than re-rendering the page each tick.

#### 6.4 Resilience

Three tiers, cheapest first:

1. **In-process supervisor.** Supervised children — `poller`, `rotator`,
   `sse-heartbeat` — each a run-once-then-reschedule loop. A child that
   **throws** counts toward a fault-intensity window (5 faults / 60 s);
   exceeding it exits the process so launchd restarts it. **Clean-exit cycles
   are the normal loop and must not count** — otherwise a healthy service
   escalates itself to death every minute. Expose `restarts` in the health
   payload and make it mean *faults*: a healthy service sits at 0 forever.
2. **Circuit breaker + spawn timeout** around the `--status-json` call. A hung
   status probe gets killed by timeout; K consecutive failures open the breaker
   → serve last-known-good; half-open after a cooldown.
3. **Internal watchdog.** Three consecutive checks with a poll age ≥10 s means
   the loop is wedged in a way the supervisor can't see → `exit(1)`.

Graceful shutdown: `SIGTERM`/`SIGINT` drain SSE clients and exit 0 within ~2 s.

### 7. launchd

`~/Library/LaunchAgents/com.<you>.buildgate.plist`, loaded with
`launchctl bootstrap gui/$UID <plist>` (`bootout` to remove;
`kickstart -k gui/$UID/<label>` to restart). Not `launchctl load` — that's the
deprecated spelling.

```xml
<key>RunAtLoad</key>            <true/>
<key>KeepAlive</key>            <dict><key>SuccessfulExit</key><false/></dict>
<key>ThrottleInterval</key>     <integer>3</integer>
<key>ProcessType</key>          <string>Interactive</string>
<key>StandardOutPath</key>      <string>/Users/you/.buildgate/log/out.log</string>
<key>StandardErrorPath</key>    <string>/Users/you/.buildgate/log/err.log</string>
```

Differences from systemd you must design around, not paper over:

- **No `sd_notify`, no `WatchdogSec`.** launchd has no readiness or heartbeat
  protocol. Delete that tier; tiers 1 and 3 in §6.4 carry the load alone.
- **No `MemoryMax` / `CPUQuota`.** Closest are RLIMIT-based
  `HardResourceLimits` / `SoftResourceLimits` plus `Nice`. The service is tiny
  (a poll loop and some JSON); this is acceptable.
- **No `ProtectSystem` / `ProtectHome` / `PrivateTmp`.** There is no plist-level
  sandbox. Compensate by binding loopback-only and touching nothing outside the
  state dir. Say so honestly in the README rather than implying parity.
- **`ThrottleInterval` ≈ `RestartSec`**; there is no `StartLimitBurst`
  equivalent. A genuine crash-loop will spin at the throttle interval forever —
  which is why the supervisor's fault-intensity window matters.
- **The environment is nearly empty.** No `PATH`. Use absolute paths in
  `ProgramArguments`, and set what you need in `EnvironmentVariables`. Point at
  the real Bun binary (`/Users/you/.bun/bin/bun`), not a shim or a shell alias.
- **No journald.** Logs go to the files above; rotate them yourself or they grow
  without bound. `log stream --predicate 'process == "bun"'` for live tailing.

### 8. Definition of done

- [ ] Two shells race `buildgate.sh sleep 20` with `SLOTS=1`; the second visibly
      waits and the events log shows `wait` → `acquire` → `release` in order.
- [ ] `kill -9` the holding process. The slot frees **immediately** with no
      cleanup code involved, and the next waiter picks it up.
- [ ] `BUILD_GATE_NAME=test` and the default `build` gate run concurrently
      without blocking each other.
- [ ] Timeout path exits 75 and the command provably did **not** run.
- [ ] Every gating signal is broken deliberately (delete the state dir, corrupt
      the events file, remove the lock helper) — the wrapped command still runs
      every time.
- [ ] With no probes implemented, `effective_slots` is **1**, not `SLOTS`
      (§4.4).
- [ ] `--status-json` output parses and matches §4.5 exactly, including a busy
      and a free slot in the same response.
- [ ] Kill the dashboard mid-build: the build is unaffected. Restart it: state
      re-derives correctly from disk.
- [ ] `/healthz` returns 503 while wedged, 200 when healthy, and the transition
      is observable.
- [ ] `restarts` stays at 0 across a 10-minute healthy run.
- [ ] Pure modules (`state`, `analytics`, `supervisor`, `health`) are ≥90%
      covered by unit tests with injected clocks. Aim the coverage there, not at
      the I/O adapter.
- [ ] Reboot. The LaunchAgent comes back on its own and the dashboard is live.
- [ ] A label containing quotes, backslashes, and a newline round-trips through
      the events log without breaking the parse.

### 9. Build order

Bottom-up, verifying each layer before the next:

1. `gatelock` (or confirm Homebrew `flock`) + the two-shell race test.
2. The gate script: acquire, timeout, events, `--status-json`.
3. Pure `state.ts` + its tests, driven by fixture JSON — no service yet.
4. `gate.ts` I/O adapter, then `server.ts` with `/api/state` and `/healthz`.
5. The dashboard HTML, then SSE.
6. Supervisor + breaker + watchdog.
7. launchd install script.
8. Analytics last — it is the only genuinely optional piece.

Do not start §6 until §4 survives the first five checks in §8. A dashboard over
a gate that doesn't actually serialize is a very convincing lie.
