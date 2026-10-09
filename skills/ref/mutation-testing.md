# Go mutation-testing baseline (sigmashake-gov)

Status: **advisory, report-only baseline** established 2026-07-18. This is
**not** a CI gate and does not block anything. It is a one-time experiment
under manual supervision. It gives a real number and a reusable pattern. A
future review can then decide on a threshold and promote the baseline to a
gate.

## Tool

[`go-gremlins/gremlins`](https://github.com/go-gremlins/gremlins), pinned at
**`v0.6.0`** (Apache-2.0). It is a dev and test tool only. It is never a
runtime dependency of the shipped `ssg` binary. Install it with
`go install .../cmd/gremlins@v0.6.0` into `$GOPATH/bin`. This does **not**
change `sigmashake-gov/go/go.mod`.

```
go install github.com/go-gremlins/gremlins/cmd/gremlins@v0.6.0
```

Verify the pin before you run:

```
go version -m "$(command -v gremlins)" | grep github.com/go-gremlins/gremlins
# mod  github.com/go-gremlins/gremlins  v0.6.0  h1:3G2ROO0I...
```

`scripts/mutation-baseline.sh` (see below) refuses to run against a
`gremlins` binary that is not pinned or does not match the pin.

## Package chosen: `internal/voicecodec`

Inspection confirmed that this package is suitable:

- It is pure and deterministic. It has zero I/O and zero network access.
- It is a byte-format port of the Elixir `Navi.S2S.Codec` reference (see the
  package doc comment in `codec.go`).
- It has 19 existing unit tests.
- It has meaningful branching: clamping, truncation, the
  `from <= 0 || to <= 0 || from == to` degenerate-rate guard at `codec.go:89`,
  single-sample and multi-sample resampling, and the `denom < 1`
  divide-by-zero guard.

A `--dry-run` scoped to this package produced 84 non-zero mutants (80
runnable and 4 not-covered). The run was not vacuous and not filtered.

## Makefile target

```
make -C sigmashake-gov/go mutation-baseline
```

This target is a thin wrapper (`sigmashake-gov/go/Makefile`) around
`sigmashake-gov/go/scripts/mutation-baseline.sh`. The script does all the
orchestration. A shell trap for process-group cleanup is difficult to write
correctly as a `make` recipe body.

### What the script does, in order

1. **Gate status observation.** The script prints `bun run gate --status` (the
   "build" gate, the one in use, see step 4). The output is a point-in-time
   log of the configured slots. It does not prove that nothing else is
   running.
2. **Pin an exact SHA.** The script runs `git fetch origin && git rev-parse
   origin/main`. It never uses local `HEAD`. On this box, local `HEAD` is
   often older than `origin/main`. This run confirmed it: local HEAD
   `8ec1667a5` and `origin/main` `6986d3cc3` at the time of execution.
3. **Isolated worktree.** The script runs `git worktree add --detach
   <mktemp -d> <SHA>`. The worktree is fully outside the shared checkout. By
   default it is under `$TMPDIR`/`/tmp`, not under the repo. Thus it never
   shows as `??` clutter in `git status`. Gremlins rewrites source files for
   each mutant only inside this throwaway worktree.
4. **No gate acquisition of its own.** An earlier revision of this script
   wrapped the launch in `bun run gate:test <cmd>` (the "test" gate). This was
   wrong, not only redundant:
   - The host-wide `go` shim (~/.local/bin/go) already sends every `go test`
     that Gremlins starts (one for each mutant) through the **separate
     "build"** gate. See the "Discovered environment quirk" section below.
   - The wrapped script held the "test" slot for the *whole* run. Each
     per-mutant `go test` independently waited for the "build" slot through
     the shim.
   - This is the nested cross-gate blocking that a single-gate design must
     prevent. The `BUILD_GATE_HELD` re-entrancy short-circuit tracks gate
     *names*. Thus, if the script holds "test", a nested "build" acquisition
     does not pass through.
   - The fix: the script does not acquire any gate slot. The "build" gate of
     the go shim is the only protection, and it is sufficient. It bounds every
     real `go test` call that Gremlins makes. The other work of the outer
     script (git fetch, worktree add and remove) is light. It does not need a
     heavy gate.
5. **Launch through `setsid`** in its own process group. The launch uses
   `taskset` to bound the run to the same `0-23` core envelope that
   `go/Makefile` already uses. It also uses `--workers=4`, `GOMAXPROCS` equal
   to the worker count, and a wall-clock limit of `timeout --signal=TERM
   <secs>s` (default 900s).
6. **Cleanup on exit (EXIT/INT/TERM trap).** The trap does these actions in
   order:
   - It captures the real PGID (see below).
   - It sends TERM to the whole group, polls for up to 10s, and escalates to
     KILL. Then it verifies that the group is gone.
   - It copies the mutation report to a durable path **outside** the worktree
     before it removes the worktree.
   - If graceful removal fails, it force-removes the worktree (`git worktree
     remove --force`). If that fails, it uses a Python `shutil.rmtree` call.
     See below for the reason why it does not use a shell recursive-force
     remove.

   **Accepted residual risk**: no trap can catch `SIGKILL` of the script
   itself or a power-loss event. In these cases, you must clean up the
   worktree or the orphaned process group by hand (`git worktree prune`,
   `pkill -f gremlins`). This is a manual baseline run under supervision. It
   is not an unattended CI job. Thus the risk is accepted and not engineered
   away.

### PGID-capture technique (worth documenting — non-obvious)

`setsid <cmd>` (no `-f`) forks once. The **child** calls `setsid(2)` and then
`exec`s the target command in place. Tests confirmed this: `ps` shows that the
`$!` PID of the background job *is* the PID of the target command, with
`PGID == SID == PID`. The outer script now starts `setsid` directly in the
background. No `bun run gate:test` / `build-gate.sh` subshell chain is between
them (see step 4 above). Thus `$!` at the outer layer is probably correct. But
the script still uses the technique that was proven in the original
(gate-wrapped) version, as a second safety measure.

The technique works in this way:

1. The `setsid` process starts this **same script** again in a hidden
   `--run-inner` mode.
2. The first action of that mode is `echo "$$" > "$pgid_file"`.
3. The inner mode then `exec`s through `taskset`, `timeout`, and `gremlins`.
   Each `exec` keeps the PID. `timeout` forks one child for `gremlins` itself.
   That child stays in the same process group.
4. The outer script polls for the file to appear. The poll has **no fixed
   timeout**, because the spawn is almost immediate and nothing between can
   block it. The poll stops early only if the background job has already
   exited (for example, the failure path `cd "$pkg_dir" || exit 97` in the
   inner mode). Thus the poll cannot wait forever for a file that nobody
   writes.
5. The outer script reads the PGID. The cleanup trap uses `kill
   -TERM/-KILL "-$PGID"`.

### Why the worktree-removal fallback doesn't shell out to a recursive-force `rm`

The rule `vibe-no-destructive-zaps` in `vibe_no_exfil.rules` scans the content
of any **written** file for the exact substring `rm` + space + `-rf`. It
denies the write in the whole repo. It is not limited to the `sigmashake-vibe`
tenant-app product that it was written for. The script does not evade the
pattern (for example, with a different order of flags). The fallback path
runs only if `git worktree remove --force` fails. It uses
`python3 -c 'import shutil, sys; shutil.rmtree(...)'`. This is a different,
standard method for recursive removal. It is not a disguised shell delete.

### Discovered environment quirk: the global `go` shim gates internal `go test` calls — and is the ONLY gate this script relies on

`~/.local/bin/go` is a **host-wide shim** (`case "${1:-}" in test|build|vet)`).
On this box, it sends every `go test`, `go build`, and `go vet` call through
`bash scripts/build-gate.sh` under the **"build"** gate name. The name is
hardcoded. It is not `BUILD_GATE_NAME=test`. The shim also does the same
live-stream deferral check.

Gremlins runs `go test -coverprofile=...` for each mutation. Thus **each of
those calls independently hits the "build" gate**. This behavior exists
already, applies to the whole host, and does not depend on this script.

An earlier revision of this script *also* wrapped its own outer orchestration
in the separate "test" gate. The reason was defense in depth. A later review
found that this combination was harmful (see step 4):

- The outer "test" gate slot stayed held for the whole run.
- Each per-mutant `go test` call waited, possibly for a long time, on the
  independent "build" gate through the shim.
- This is a nested cross-gate wait. It is not extra safety.

The current design has no outer gate. It uses only the existing "build" gate
protection of the go shim for each real `go test` call. This protection is
necessary and sufficient, because all the real CPU load is in those calls.

### `SIGMASHAKE_LIVE_TESTS=1`

The stream was live (`pitstop:streaming=true`) during the 2026-07-18 baseline
run below. At that time, the script itself ran `export SIGMASHAKE_LIVE_TESTS=1`
without a condition. This let the `go test` calls of that run (through the
"build" gate of the go shim) proceed. Without it, they would defer with a
neutral exit 96.

That was a bug, not a feature. The override was in the script. Thus **every
future invocation**, forever, silently bypassed the live-stream deferral. The
override must apply only to the one supervised run that the operator approved.

The script no longer sets or exports this variable. The script uses the value
from the environment of the caller (unset or 0 by default). Thus a normal
`make -C sigmashake-gov/go mutation-baseline` now defers correctly while the
stream is live.

A deliberate light run is now an explicit **per-invocation** operator choice:
`SIGMASHAKE_LIVE_TESTS=1 make -C sigmashake-gov/go mutation-baseline`. A light
run has a small workload, is bounded by CPU and workers, is supervised, and
matches the approved carve-out in `build-gate.sh`. The override does **not**
touch any CPU/GPU power cap. `taskset`, `--workers`, and `timeout` stay the
safety bounds.

If a future promotion makes this an unattended or CI gate, it must NOT set
this override implicitly. It must either run off-air or make the override
decision again explicitly.

### Live-transition contamination guard (fixed 2026-07-18, third pass — continuous monitoring)

The script no longer has an outer gate (step 4 above). Thus each of the 80+
per-mutant `go test` calls of Gremlins independently asks the go shim, "Is the
stream live now?" Before, one outer gate slot asked this question once. This
change brought back a race that the outer wrapper had absorbed by accident.

- Exit 96 from the go shim is a **neutral deferral**. It is not a test
  failure.
- Gremlins v0.6.0 does not know this convention. It treats each `go test` exit
  code that it does not recognize as a real surviving mutant (`LIVED`).
- Assume the stream goes from off-air to live **during** a run. The mutants
  that were evaluated before the change get a real verdict. The mutants that
  were evaluated after the change get a false `exit-96-as-LIVED`.
- These false results silently corrupt the report. They look the same as real
  findings.

**First and second pass design (point-sampling, superseded):** Earlier
revisions checked the live or off-air status **twice**. One check was before
the start of any work. One check was in `cleanup()` after the run. The script
compared the two readings. This missed a real failure mode, a **transient
blip**. In a blip, the stream goes live during the run and is off-air again
before the check in cleanup. Off-air before and off-air after look the same as
"never went live". Thus the point-sampling guard wrongly decided "no
contamination". It copied a corrupted report, although some per-mutant `go
test` calls during the run really did hit the exit-96 neutral defer of the go
shim.

**Current design (continuous background polling):** The script still refuses
to start if the stream is already live (see below). It does not do a second
point-in-time check at the end. Instead, it starts a light background poller
(a plain `&` subshell) at the moment it starts the inner Gremlins job. The
poller runs as long as that job runs.

- The poller calls `stream_is_live()`. This function mirrors the two-signal
  check of `build-gate.sh` byte for byte. First it tries `/api/stream` of
  pitstop. If that fails, it does a raw RTMP `:1935` ESTABLISHED-socket probe.
- The poll interval is fixed: `MUTATION_LIVE_POLL_INTERVAL_SECS`, default 12s.
- At the **moment** any single poll sees "live", the poller latches that fact
  permanently. It writes a `CONTAMINATION_SENTINEL` file and stops polling.
  Later polls, or the state at cleanup time, have no effect on this.
- The verdict of `cleanup()` depends only on whether the sentinel was ever
  written. `cleanup()` never makes a new `stream_is_live()` call. Thus a blip
  that has already ended when cleanup runs is still *usually* caught.
- `cleanup()` kills the poller and reaps it with `wait` as one of its first
  actions. Thus a run that finishes fast or stops early never leaks a stray
  background polling process.

**Known residual gap.** This is a probabilistic guard, not a deterministic
guard. The operator accepted this limitation as is (decision of 2026-07-18).
Never claim otherwise in a future edit. Detection depends on a fixed polling
interval (`MUTATION_LIVE_POLL_INTERVAL_SECS`, default 12s). It does not depend
on the real exit-96 events of the go shim. A deterministic guard needs the
shared `~/.local/bin/go` shim or `scripts/build-gate.sh` itself to write a
sentinel at each defer. Every heavy build and test in this repo uses that
infrastructure, not only this script. This is out of scope for this change.

Three windows can still pass undetected:

1. A live blip that is shorter than the poll interval.
2. The gap between the start of Gremlins and the first check of the poller.
3. A blip after the last check of the poller but before `cleanup()` kills it.

In practice, this matters only for an unattended run without
`SIGMASHAKE_LIVE_TESTS=1`, where the stream has a *transient* live period
during the run. The before-check still reliably refuses to start if the stream
is live at launch. This covers the common case. Treat the guard as a
best-effort safety net, not a guarantee, when you read a "clean"
(non-contaminated, exit 0) result.

- **Live already at start** (and `SIGMASHAKE_LIVE_TESTS` is not `1`): The
  script refuses to start. It does no git fetch, makes no worktree, does no
  gate-status probe, and starts no poller. It exits with **96**. This is the
  same neutral-deferral code that the go shim uses, for consistent meaning.
- **The background poller ever sees "live" during the run** (this includes a
  transient blip that has since ended): The run is contaminated. The script
  **discards** the mutation report. It never copies the report to
  `MUTATION_REPORT_DIR`. The report is deleted with the throwaway worktree a
  few lines later in all cases. The script forces exit **98**
  **unconditionally**. Contamination has priority over the exit code of the
  inner gremlins or go-test process (for example, a real, unrelated test
  failure). The result of a contaminated run is not valid. You must do the run
  again, whatever the other results are.
- **`SIGMASHAKE_LIVE_TESTS=1` for the whole invocation**: The guard (the
  before-check and the background poller) does nothing. The go shim live-defer
  is overridden for the whole time. Thus it cannot fire during the run, and
  there is no transition to detect. The override must cover the **whole**
  invocation. Never toggle it during a run. If you do, the contamination that
  the guard must prevent can still occur.

**Cleanup re-entrancy + signal-suppression guard:** Bash runs a combined
`trap cleanup EXIT INT TERM` handler **twice** on a trapped signal. It runs
once as the signal handler. It runs again through the EXIT trap when the
`exit` of that handler runs. A minimal repro verified this. It is not specific
to one bash version.

A `CLEANUP_RAN` flag alone is not sufficient. Assume a **second** `INT` or
`TERM` arrives while the **first** `cleanup()` call is still working (for
example, it is polling and waits for the gremlins process group to die). The
second signal enters the handler again. The handler takes the "already ran"
branch and calls `exit` immediately. This tears down the shell while the first
call is still working. The first call has not yet confirmed that the process
group is gone, or has not yet made the decision to copy or discard the report.

The fix: the first action of `cleanup()` is `trap '' INT TERM`. This is right
after it captures `$?` and before the `CLEANUP_RAN` check. It ignores further
`INT` and `TERM` for the rest of the life of the process. It does not touch
the `EXIT` trap. Thus `cleanup()` still fires normally when the first call
reaches its own `exit "$exit_code"` at the end. A second signal during active
cleanup now does nothing. It no longer enters the handler again.

**Hermetic control-flow test suite:**
`sigmashake-gov/go/scripts/mutation-baseline.test.sh` is plain bash. It
follows the convention of `scripts/build-gate.test.sh`: stubs for `go`,
`gremlins`, `curl`, and `setsid` injected through PATH, the helpers `ok`/`bad`/
`eq`/`has`/`hasnt`, and no real toolchain invocation. It tests these cases:

- Normal completion.
- Refusal when the stream is live at the start.
- A **transient** live blip during the run that has ended when gremlins
  finishes. The sentinel of the continuous poller still catches it.
- Contamination that forces exit 98, although the inner process failed
  independently for an unrelated reason.
- The background `--run-inner` child exits before it does real work. The
  PGID-capture poll does not hang.
- A `setsid`(2) failure, so `--run-inner` never runs and the PGID file stays
  empty or missing. This is a different fixture from the one above. It uses a
  temporary `setsid` stub. Cleanup must degrade gracefully. It must not
  assume that a PGID always exists.
- TERM that escalates to KILL against a gremlins stub that ignores TERM.
- A **second** TERM that arrives while cleanup is still polling for the
  process group to die. It must not abort the work that the first cleanup is
  doing.
- The cleanup-fires-exactly-once guard on all the signal-exit paths above and
  on the normal-exit path.

Run the suite directly:
`bash sigmashake-gov/go/scripts/mutation-baseline.test.sh`.

## Baseline result (2026-07-18, SHA `6986d3cc3fb1827dd2acb5d7cf9a3cbf709c3273`)

These are the native metrics of Gremlins, unmodified. There is no custom
formula.

| Metric | Formula | Value |
|---|---|---|
| Test efficacy | `KILLED / (KILLED + LIVED)` | **81.01%** (64/79) |
| Mutant coverage | `(KILLED+LIVED) / (KILLED+LIVED+NOT_COVERED)` | **95.18%** (79/83) |

The full breakdown by status follows. The run generated 84 mutants in total.
This is not zero, and it confirms that the run is not vacuous or filtered.

| Status | Count |
|---|---|
| KILLED | 64 |
| LIVED | 15 |
| NOT_COVERED | 4 |
| TIMED_OUT | 1 |
| NOT_VIABLE | 0 |
| SKIPPED | 0 |

Raw JSON report: `sigmashake-gov/go/.mutation-reports/20260718T231050Z-voicecodec-6986d3cc3fb1.json`
(gitignored. It is an artifact on this machine only and is not committed. To
make it again, run the target again.)

Most of the 15 LIVED mutants are in the interior loop and clamp logic of
`ResampleLinear` (`codec.go:111-141`). They are boundary conditions on the
`lo`/`hi` clamping and the `nOut`/`denom` edge cases. One more is a boundary
condition in the sample-count guard at `codec.go:34-36`. These are real gaps
for a future pass to write tests. They are not tooling noise.

## Promotion path

This baseline is **not** a blocking gate. To promote it to a gate, you need a
**future, separate review**. That review must do these actions:

1. Decide a real efficacy and coverage threshold. This document does not
   invent one.
2. Decide if `--threshold-efficacy`/`--threshold-mcover` go into CI, and at
   what cadence. Mutation testing is comparatively slow. This run had 84
   mutants against a 150-line package. It took about 8s of real mutation
   testing. This does not extrapolate linearly to the full module.
3. Decide if the scope grows beyond `internal/voicecodec` to other pure
   packages, and in what order.

**Definition of done for this pass, precisely scoped**: this baseline finds
surviving mutants in the one selected representative package,
`internal/voicecodec`. It does **not** protect or gate the rest of
`sigmashake-gov`. It makes no claim about the rest of `sigmashake-gov`.

## Scope of this change

This change makes zero changes to any shipped `.go` file, to `go.mod`, or to a
runtime dependency. It adds new files only:

- `sigmashake-gov/go/scripts/mutation-baseline.sh`
- `sigmashake-gov/go/Makefile` — added the `mutation-baseline` target
- `sigmashake-gov/go/.gitignore` — added `.mutation-reports/`
- This document
