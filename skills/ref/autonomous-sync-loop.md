# Autonomous cross-OS workspace sync loop

*Load this only when the task is "keep /mono running to reconcile `sigmashake-workspace` across Windows/macOS/Linux". It is a re-callable iteration loop, not a one-shot change.*

## What this is
One `/mono` call = **one iteration**. It moves `sigmashake-workspace` checks, parity, and tests one binding-constraint slice closer to full cross-OS sync. Then it commits and pushes, and it ledgers to RSI. If the loop has not converged, it continues itself or schedules the next iteration. **Convergence = the 3-OS parity matrix is all-`met` + tests are green on every peer that attested.** RSI is the **ledger and governor**. It is *not* the halt oracle. A parity-sync run does not move `active_subs`. So never block sync-convergence on the north star.

## Durable state — orient BEFORE you act (compaction-safe)
An auto-compaction (or a fresh re-invoke) wipes the in-context memory. **Every iteration starts by rebuilding "where we are" from the three durable sources. Never use what you think you remember:**
1. **RSI ledger** — `rsi_state` (or `curl -s 127.0.0.1:7919/api/state | jq`). Read the last `decisions`/`changes` for this sync loop (filter `kind` ∈ {lever,commit,ledger}, by machine). This is the iteration cursor.
2. **Git history** — `git log --oneline origin/main -15` + `git status`. They show which slices already landed on `origin/main`.
3. **Parity truth** — `.sigmashake/production-readiness.json`. Regenerate `sigmashake-workspace/src/readiness.generated.ts` with `bun --cwd sigmashake-workspace run generate`. Then read `met`/`partial`/`unmet`/`unknown` for each Win/Mac/Linux requirement (or `GET /api/json/workspace/readiness`).

**Durable state is the standing directive here. A `/goal` is not.** This mode skips meta-prompt synthesis (Prereq 2). You cannot arm `/goal` anyway, because it is a UI command that only the operator can type (SKILL.md Prereq 3). The convergence target lives in the three sources above and survives compaction on its own. **If a fresh call finds the durable state of the loop and parity is not all-`met`, resume the loop. Do not re-interview.** (If the operator did type a `/goal`, its Stop hook is a bonus backstop. It is never the orient signal.)

## One iteration (the cycle)
1. **Orient** (the three durable sources above) → the current parity matrix and the last ledgered iteration.
2. **Preflight peers** — mesh `/runners` + `curl :7929/health` per `ref/mesh-orchestration.md`. Classify each OS peer as online or offline (Windows QEMU = `sigmashake-win` through SSH-exec; Mac peer through mesh). Offline does not mean stop (see Peer policy).
3. **Select the binding-constraint slice** — the single highest-priority cross-OS parity gap that is not yet `met` (RSI-lever discipline: pick the constraint that unblocks the most). Pick **one slice**. Do not send the whole matrix at once.
4. **Dispatch through the normal phases** (Scope → Implement → Test, all gates apply). Send cross-OS legs to their owning OS peer. Use mesh `POST :7929/dispatch` with `govProfile`, or SSH-exec for the Windows peer per `ref/mesh-orchestration.md`, with `sha` pinned to `HEAD`. The local leg stays gate-wrapped. **Power-bounded: dispatch to peers serially. Never stack a peer build with live GPU inference.**
5. **Land it (Phase 5, EVERY iteration — "commit + push after major tasks"):** stage only the files of this slice, by explicit path (never `git add -A`). Push gate-wrapped (`bun run gate:push`) to `origin/main`. Verify that it converged (`HEAD == origin/main`). A slice that built and tested but is not pushed is **not done**.
6. **Ledger the verified outcome:** `rsi_log_change` `kind=lever` (the gap closed) + `kind=commit` (the SHA), and `rsi_propose_decision` (observe/act) for the iteration. **Verified outcomes ONLY.** Suppose you write an unverified growth *claim* to `docs/growth/GROWTH-LOOP-LEDGER.md`. A claim is a "HIT", or a users/signups/subscribers/installs figure. It trips `rsi-ledger-no-unverified-claim` of `rsi-anti-cheat`. That rule is an **ASK** write (pri 80). A human can approve it. It is not a hard DENY. In all cases, **do not hand-edit that ledger**. Use the `rsi_*` MCP and record only what you verified. The auto-approver stays **default-OFF**. The operator sanctioned the commit and push you just did through `/mono`. The auto-approver did not approve them. Write, deploy, secret, and agent-spawn shapes **always escalate to a human**. Never self-approve them.
7. **Convergence check:** regenerate and re-read the parity matrix. **Are all 3 OS columns `met`, with tests green on every attested peer?** If yes: `rsi_log_change` `kind=ledger` "workspace cross-OS sync reconciled", emit the final report, and **halt** (the goal clears automatically). If no: continue (step 8).
8. **Continue (self-drive, per the locked decision):** if more reachable work remains in this run and the budget and context allow, loop to step 1 **within the run**. Budget or compaction can interrupt the run. An offline peer can also block further progress. In these cases the cursor is already durable (step 6). **`ScheduleWakeup` the next `/mono <same sync task>`**, so the loop resumes unattended. The operator can re-invoke at any time. A fresh call re-orients from durable state (no progress is lost). Pick the wakeup delay by what you wait for (a peer to come up, or only continuing). See the ScheduleWakeup guidance. Use a long fallback (1200s+) when idle.

## Peer-offline policy (honest-red — never fake-green)
A peer that is down (Mac peer unreachable, Windows VM not up) does **not** halt the loop. It also does **not** let you mark its leg `met`:
- Reconcile the slices of every **reachable** peer in this iteration (honest partial progress).
- The requirements of the offline peer stay `partial`/`unmet` in `production-readiness.json`. **Probe it again in each iteration.**
- **Full-sync (the step 7 halt) needs EVERY peer to have attested its legs green.** An unattested leg is `BLOCKED (peer offline)`. Show it in the report. Never skip it silently. Never assume it is green.
- Starting the Windows QEMU peer (`virsh start`) is an **operator action**, unless the operator opted in for this run. Note it as `BLOCKED` and continue.

## SSG rules awareness (an autonomous loop must not wedge itself)
An unattended loop cannot answer an interactive ASK. Know which rules govern its bursts **before** you start them:
- When you debug, first run `ssg recall <keyword>` on the 119 ambient entries. An established runbook is better than a new discovery.
- **Gate-wrap proactively:** `power_{go,bun,make}_*_ungated` and `power_git_push_ungated` (`.sigmashake/rules/power-safety.rules`) DENY or ASK on ungated heavy commands. So send every build, test, and push through `bun run gate` / `gate:test` / `gate:push`. The same applies to every dispatched peer agent.
- **Do not auto-edit rules:** writes to `.sigmashake/rules/*` are governed (agent-config-integrity). `rsi-anti-cheat` **ASKs** (a human can approve) on an unverified growth *claim* in the ledger. It is not a blanket DENY, but it is still a hard stop for an unattended loop. The loop *reads* rules (`ssg recall`, hook audit). It does not change them during the run.
- **Hook audit every iteration** (Phase 6): `ssg debug snapshot ssg://audit/recent?limit=50`. A DENY or ASK that blocked a legitimate loop command is a false positive. Fix the rule and commit, or the next iteration wedges on it.
- After any sanctioned rule edit, the **operator** runs `ssg rule sync` (overwrite) in an ungoverned terminal. `ssg init` does not fix a bad existing row.

## RSI as governor (not halt oracle)
The loop *uses* RSI machinery: the immutable ledger, decision sequencing, priority-queue lever discipline, and the peer registry (`rsi_machines`). But the convergence of the sync task is the **cross-OS parity matrix**, not `active_subs`. Record every iteration to the RSI ledger, so the growth loop sees the work. Do not gate sync-convergence on the north star. Use `rsi-lead` for lever-priority arbitration. Use `coordagent-lead` for the mesh dispatch surface.

## Safety invariants (assert every iteration)
- **Power:** dispatch to peers serially. Gate-wrap all heavy local commands. Never stack a peer build with live inference. Verify that the caps are live before a burst (`scaling_max_freq`→`3500000`, `power.limit`→`250.00 W`; do **not** read `cpufreq/boost`).
- **Pricing:** no slice adds a free or anonymous path to a value surface.
- **Honest-health:** no fake-green legs. `BLOCKED` is a valid iteration outcome that you can report.
- **Token discipline:** orient from compact durable state (ledger/git/json summaries). Do not re-read the workspace. Delegate reads and edits to peer and scope subagents.
