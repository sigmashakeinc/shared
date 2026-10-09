# ref: Trace-Debug Discipline — pstack-derived reproduce/instrument/prove playbook

Load this for: a bug-fix, regression, crash, flaky, or perf-diagnosis run; a Phase-4 test failure that needs
diagnosis; or any forensics (live or post-hoc). It is adapted from cursor/plugins `pstack` (Sources below) onto
the existing tools of this repo. No new tooling was built for this ref (reuse only; gaps in §11).

## 1. Reproduce first
You cannot verify a fix that you cannot reproduce. Reproduce **yourself** on the matching control surface
(gov → the running `ssg` daemon; desktop → the shipped shell; a Worker → `wrangler dev`/staging). Never
give the repro to the user. Record `repro: <command>` + the observed failure (rung 5). Does it not fire on the first
try? Force it: synthesize the trigger, tighten the timing, ordering, or load, and instrument until it fires. Only
after a genuine attempt: `repro: NOT-REPRODUCED — <what was tried>` ⇒ **diagnosis-only** until it fires.
Never make a blind fix. `mono-bugfix-dispatch-repro-contract` (staged) requires this block on every
bug/regression/crash/flaky/root-cause dispatch prompt.

## 2. Hypothesis bisect
Start the first hypothesis from real signal, not a guess: `ssg debug rca --markdown`, `graphify query
"<symptom>"`, Agent OS `diagnose`, or git history (`git log -p --follow`/`git blame`) on the implicated
`file:line`. Ask "why" until you name the *mechanism*. A plausible but unconfirmed cause can be
unanimously wrong while the real cause is one subsystem away. Confirm the mechanism before any
redesign. Eliminate with runtime evidence, not by re-reading harder. After the 3 probes of the Debug Budget,
write ONE hypothesis and test it directly (patch + rerun, or a targeted probe). **Refuted ⇒ revert what
it motivated.** A guard, nil-check, or retry that you keep to test a wrong hypothesis is a hedge, not a fix
(`mono-no-hedge-fix-guard`, staged). Does it fail only after a restart? Suspect stale state first — §7.

## 3. Instrument, don't guess
| Tool | Yields | Token rule |
|---|---|---|
| `ssg debug rca --markdown` | RCA pack: hypotheses+evidence+`trace_id` | record `id`+verdict, drop body |
| `ssg debug follow` (bg) | streaming JSONL frames, live eval path | verdict line only; kill before Phase 5 |
| `ssg debug snapshot ssg://frames/<trace_id>` | per-stage timeline | pull only the failing stage |
| `ssg debug analyze-sessions --last=24h` | retry-loop/blind-build/state-speculation flags | act before the 3rd retry |
| `ssg probe desktop --json` | daemon/port/log state | verdict fields only |
| sigmashake-ssqa | SQLite findings → Tier-0 candidates | query findings, don't dump the DB |
| `headroom wrap <cmd>`/`compress <file>` | retrievable artifact via `get <handle>` | store the handle, not the blob |
| `stk <cmd>` | deduped/truncated verbose output | prefer over raw, non-power-heavy |
| sigmashake-vitals (:7914) | host CPU/mem/fork-rate | one reading, not a poll loop |
| Agent OS `diagnose` | pre-computed codebase diagnosis | front door before graphify/oracle |
| graphify `query`/`path`/`explain` | scoped subgraph | prefer over raw grep |

## 4. Runtime forensics vs trace forensics
**Runtime (live):**
1. Capture a real artifact (profile/heap/trace) while the app runs.
2. Reduce it to the smoking gun.
3. **Prove** the mechanism with a live probe or hotfix before you believe it.
4. When it is confirmed, hand back to §2.

**Trace (post-hoc):** the capture already exists. Never run it again.
1. Identify its format.
2. Dump it to a queryable form (`headroom` or the own reader of the tool).
3. Query the hottest frames, the retainer chain, or the blocked thread.
4. Attribute it to `file:symbol:line` with the own symbols of the artifact. If it is unmapped, say so plainly.
5. Confirm it against a paired before/after capture. If there is none, label it the "strongest hypothesis supported."

## 5. Evidence ladder
| Rung | Meaning | Here, concretely |
|---|---|---|
| 1 | You said so | A claim with no pointer — worthless alone |
| 2 | Pointed at file:line | Cited the code, didn't run anything |
| 3 | Walked the failure step-by-step | Traced the call/data path by reading, not executing |
| 4 | Ran it | Gate-wrapped test/command with an exit code — `bun run gate:test <cmd>`, `go test`, an RCA pack `id` |
| 5 | Reproduced live | Fired in the running app/daemon — `ssg debug follow` frames, a live probe, a confirmed hotfix |
A "verified", "fixed", or "passes" claim needs a rerunnable pointer at rung ≥4. Otherwise label it `unproven`.

## 6. Confidence tiers + mandated phrasing
| Tier | When | Phrasing |
|---|---|---|
| Direct citation | Read the exact code/log/doc stating it | "because…", "designed to…" |
| Convergent indirect evidence | ≥2 independent signals agree | "the evidence indicates…" |
| Inferred, explicit chain | Reasoned from adjacent facts, chain stated | "appears to…", "likely…" |
| Speculative | One weak signal or none | "possibly…" — flagged explicitly |
| Unknown | Not established | say so — never round up |
Code is never evidence of its own intent (a function named `validateX` is not proof that it validates). A
skipped search is never evidence of absence. A null result from a search that you DID run is a finding.

## 7. Stale-state-first for restart bugs
A bug that appears only after a restart, redeploy, or reconnect is more often stale state than new code.
Check config, cache, lock file, or a serialized or pinned artifact BEFORE you re-read the logic. Precedent:
a chat-elixir release that ran from a pinned checkout diverged from the repo. A stale v1.1.1 hook binary
hid a degraded daemon. A persisted `armed:false` flag stayed after the caption code was believed fixed. Diagnose
by a diff of the RUNNING artifact or pin against HEAD before you read the code of HEAD.

## 8. Verification
Verify the real artifact, never a proxy. "It compiles", "the function exists", and a self-report are not
verification. If verification fails, suspect the **observation method** first (wrong probe, stale cache,
wrong control surface). Do this before you conclude that the fix failed. Script the check so that a reviewer can rerun it
verbatim. A passing unit test shows that the tested branch behaves as coded. It is not proof that the bug is absent
elsewhere. State that difference. Do not overclaim "the bug is fixed."

## 9. Delivery
Stage the failing repro **before** the fix, so the diff tells the story (it fails for the intended reason →
fix → it passes). Sequence verifiable units: the smallest diff that you independently showed to work. Do not make one large
unverifiable patch. Skip a bad test (mocks-only, timing-dependent, expensive infra) and say so:
`no-test: <why>`. A bad test that passes trivially is worse than an honest gap.

## 10. Rule map
| Rule id | Tier | Enforces | Escape hatch |
|---|---|---|---|
| `mono-no-hedge-fix-guard` | FORCE write | A **comment-context** hedge (`// just in case`, `# to be safe`, `/* might help */`) in a source file redirected unless the same edit carries `evidence: rca:<id> \| test:<path> \| trace:<file:line>` — product strings/fixture text never match (Codex-reviewed 2026-08-26) | Cite real evidence in the edit, or drop the hedge |
| `mono-bugfix-dispatch-repro-contract` | FORCE agent | An engineer/lead dispatch whose prompt pairs a **fix/debug/diagnose/investigate/resolve verb** with a **defect noun** (bug/regression/crash/flaky/root cause) redirected unless it carries `repro:` (+ `expected_failure:`/`evidence_rung_target: >=4`); a bare `repro` word is not a contract, and copy work naming a defect noun alone does not fire | `repro: NOT-REPRODUCED — <tried>` ⇒ diagnosis-only slice |
| `mono-trace-before-fix-gov-eval` | LOG nudge | Instrument the gov eval path before patching blind | Non-blocking |
| `mono-forensics-artifact-budget` | LOG nudge | Flags a capture not routed through `headroom` | Non-blocking |
| `mono-evidence-ladder-dispatch` | LOG nudge | Flags a "verified"/"fixed" claim with no rung ≥4 pointer | Non-blocking |
All five rules are in `.sigmashake/rules/mono_pstack_trace_debug.rules` (placed 2026-08-26). The file is the truth until
`ssg rule sync` puts them in the DB. `ssg test-rule` proved them. It reads the rule files. Ambient:
`ssg recall mono_pstack_trace_debug_discipline`.

## 11. Gaps not buildable in this run
A single-call rule DSL evaluates one tool call at a time. It cannot see cross-call state. So patterns
like "3 edits with no test run since" or "debug budget exceeded this session" cannot become `.rules`
regexes. That class belongs in the Tier-0 detectors of `sigmashake-ssqa` or in the `ssg debug analyze-sessions`
pattern flags (`retry-loop`/`blind-build`/`state-speculation`). It does not belong in a new rule. There is also no
append-only decision-trail tool here. The closest is the RSI ledger. It records verified *outcomes*.
It does not record the intermediate evidence trail that `show-me-your-work` calls for. This is a genuine gap. It is reported, not built.
**`ssg lint` is not a regex compile check** (rung-4 evidence, 2026-08-26). It reported `(?i)zzz` and
`(zzz|yyy)+canary` as "native-compatible, 0 warnings". But `nfaregex.Compile` rejects both
(`invalid pattern` / `pattern too complex`), and `Match(p, "hello")` returns **true**. A rejected pattern
fails secure to match-everything. So one such FORCE/DENY rule blocks every tool call once placed. Before
you stage any enforcing rule, prove its patterns with a throwaway `_test.go` in
`sigmashake-gov/go/internal/nfaregex/`. It calls `Compile` + `Match` on positive and negative cases. Run it
gate-wrapped (`bun run gate go test -C sigmashake-gov/go ./internal/nfaregex -run <Name> -count=1 -v`).
Then delete it. `ssg test-rule` and `ssg probe hook` evaluate only the *loaded* rule set. The `protect`
hook denies rule-file writes under any `.sigmashake/rules` path (scratch dirs included). So the Go probe is
the only sanctioned pre-placement fire test. Follow-up (not in scope here): make `ssg lint` call
`nfaregex.Compile`, so lint catches this class.

## Sources
Fetched 2026-08-26, HTTP 200 — `github.com/cursor/plugins/tree/main/pstack/skills/`:
`principle-{fix-root-causes,prove-it-works,sequence-verifiable-units,never-block-on-the-human,
encode-lessons-in-structure}/SKILL.md`, `blast-radius/SKILL.md`, `why/references/epistemics.md`,
`poteto-mode/playbooks/{bug-fix,investigation,runtime-forensics,trace-forensics}.md`, `tdd/SKILL.md`,
`show-me-your-work/SKILL.md` (raw: `raw.githubusercontent.com/cursor/plugins/main/pstack/<path>`).
