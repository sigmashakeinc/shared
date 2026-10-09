# Session economics — cache, context, and session lifecycle for a `/mono` run

This is an on-demand ref. The **operational essentials already fire inline** from `SKILL.md` (Prerequisite 0 + ⚡ Token Discipline). You do not need this file to run the pipeline correctly. Load it in these cases:
- The run is long, or auto-compact is near.
- You must decide between `/compact`, `/rewind`, and `/clear`.
- You tune a noisy command.
- You wire a new autonomous loop.

Source: Anthropic, *Maximizing the value of your Claude Code sessions* (Lydia Hallie, 2026-08-14) — <https://claude.com/blog/maximizing-the-value-of-your-claude-code-sessions>. Everything below is the set of techniques of that article. It is mapped onto the invariants of this pipeline (PSU power gates, specialized-agent dispatch, rules-over-prose, honest-red gating).

---

## 1. The cost model (why any of this matters)

Three multipliers affect cost. They are in the order of their effect:

| Factor | Relative cost | `/mono` consequence |
|---|---|---|
| **Model** | larger tier = strictly more per token | Choose the tier at **launch**, not during the run (Prerequisite 4; orchestrator is `claude-opus-5-5`; Fable is deprecated — `ref/model-routing.md`). |
| **Token class** | output ≈ **5×** input | Long narration costs about 5× the price of the file that you did not read again. The verbosity policy is a cost control, not a style preference. |
| **Cache state** | cached_read ≈ **0.1×** input | A preserved prompt cache is the largest single lever in a long run. Protect it before you optimize anything else. |

**Cost-impact priority (highest first). Spend your attention in this order:**
1. Model selection
2. Number of tokens in context
3. Turn count / session length
4. Prompt-cache management

`/mono` already encodes 1 (model routing), 2 (Token Discipline / delegate-reading), and 3 (batched independent tool calls, one `AskUserQuestion`, gate-on-the-return). This file is mostly about **4**. It also covers the parts of 2 that the harness controls.

---

## 2. Prompt cache — what breaks it

Every turn sends the whole conversation again. If the conversation is cached, the cost is about 0.1× input. If it is not cached, the model must prefill everything again at full price. These actions **break the cache**:

- `/model`. Each model keeps its **own** cache. A switch prefills again from zero.
- `/effort`. A change of effort during a conversation prefills again.
- Turning **Fast mode** on.
- `/compact`. It rewrites the whole conversation. Thus the new text is uncached by construction.
- **Expiry.** The cache expires after 1 hour in a subscription session. It expires after 5 minutes in an API-key session. `ENABLE_PROMPT_CACHING_1H=1` extends the API-key case to 1 hour.

What this means for this pipeline:

- **Decide the tier at launch.** The launch-time check of Prerequisite 4 confirms the orchestrator `claude-opus-5-5` and `/effort`. There is no Fable admission check (`ref/model-routing.md`). It is a *launch-time* check for this reason: a tier switch during the run costs a full prefill, in addition to the saving that the check tried to make.
- **Never toggle Fast mode inside a run.**
- **A phase boundary is not a reason to change effort.** The Operating Policy says "extended thinking only for Phase 2 and Phase 4 diagnosis". This is about *how you think*. It does not mean that you type `/effort` twice in a run.

### `/rewind` vs `/compact` vs `/clear`

| Situation | Use | Why |
|---|---|---|
| The last few turns went wrong (bad dispatch, wrong file, dead-end diagnosis) | **`/rewind`** | It drops only the last turns. Every earlier turn **stays cached**. It costs nothing. |
| You will leave the keyboard, and the 1h cache will expire | **`/compact`** | A summary *while the cache is still valid* costs much less than the prefill after expiry. Compact, then walk away. |
| The run is finished and the next task is unrelated | **`/clear`** | It gives a fresh context. Use `/rename` first if you can want this session again. |
| The run is finished and the next task is unrelated | ~~`/compact`~~ | Wrong tool. You keep paying for a summary of work that is done. |

Default: **use `/rewind` and not `/compact`.** `/compact` has exactly one good trigger: the break-before-expiry case.

---

## 3. Session preflight (Prerequisite 0, expanded)

Make three cheap calls, **once**, at the start of the run. Never repeat them as a "check":

1. **`/model` + `/effort`.** Confirm both. Then do not change them for the whole run (§2).
2. **`/context`.** Run it **exactly once** in a fresh session. It shows what is already loaded before you type anything: `CLAUDE.md`, MCP tool definitions, skills, and memory. Remove what this run does not need:
   - `/mcp` disables unused MCP servers. On this box this matters. `claude-in-chrome` and `phaser-game-agent` carry large sets of tool definitions. Most `/mono` runs do not need them.
   - A deferred tool costs only its name until you fetch it. This is already the cheap path. Do not fetch schemas in advance "just in case".
3. **`@`-mention. Do not name.** `@path/to/file` attaches the file directly to the message and **replaces a Read call**. Mention each file **once for each conversation**. A second mention attaches the same bytes again. This is the correct way to give the orchestrator its ≤2 pivotal files (Phase 1). Everything else stays in the context of a subagent.

If the session already has an unrelated task, run `/clear` before you start `/mono`. A `/mono` run inherits every token that is already in the window, on every later turn, for the whole run.

---

## 4. Keeping the window small

- **Everything is permanent.** The output of a file read or a command stays in the conversation for every remaining turn. A log of 40k tokens costs 40k tokens *on each turn* after that (at cached rates, but still). This is the reason for delegate-reading and gate-on-the-return.
- **Point. Do not search.** It is cheaper to name the file (or `@`-mention it) than to let a search find it. The Phase 1 scope subagent returns `file:line` anchors for this reason. Later turns can then point and not search.
- **Skills over CLAUDE.md.** Put instructions for a specific workflow in a skill. A skill loads on demand. `CLAUDE.md` loads into **every** session. This is the same ladder that the repo already enforces as *rules-over-prose*. SSG ambient entries and skills are the on-demand tiers. The prose of CLAUDE.md is the always-loaded tier. It must stay bootstrap-only.
- **Compact instructions.** The article suggests a "Compact instructions" section in `CLAUDE.md`, so that compaction keeps the correct things. **Do not add one here.** It makes the always-loaded file larger. Instead, if a `/mono` run is long enough to risk auto-compact, state the set to preserve in the run:

  > Preserve across compaction: the meta-prompt (Objective + acceptance criteria), current phase and every gate verdict so far, `AFFECTED_SUBPROJECTS`, the dispatch table (`subagent_type` + model per slice), coverage baseline, and commit SHAs. Discard: file bodies, full command logs, superseded diagnosis.

- **`/autocompact 200k`** puts back the auto-compact safety net if it was disabled (Claude Code ≥ v2.1.221). Arm it on a long pipeline run. A context overflow that nobody catches in Phase 3 is worse than a compaction that you planned.

---

## 5. Noisy commands — three tiers, in order

1. **Quiet flag first.** Make the command emit less output. Examples: `--reporter=dot` (vitest), `-q` / `--silent`, `--quiet`, `--no-progress`, `--stat` instead of a full `git diff`, and `--json` + `jq` to one field. If a repo-standard command has a known quiet form, put that form in the documented command set of the project. Then every agent uses it.
2. **Subagent second.** Send a really verbose job (full-suite run, large audit, broad sweep) to a subagent. The subagent gets its **own** context. Only the **result** returns to the main session. For repeated mechanical jobs, pin the explore tranche (`haiku`). Pin the implement tranche (`sonnet`) where the job needs coding judgment. Use `opus` only for named leads and architecture (slug map in `ref/model-routing.md`). Fable is deprecated. The pipeline already uses this mechanism for Explore/scope and engineer slices. The cost argument is only the second reason for it.
3. **Truncation backstop last.** Set `BASH_MAX_OUTPUT_LENGTH`. The system writes output beyond this limit to a **file**. Only a preview and the path go into the conversation. This is a safety net for output that you did not predict. It is **not** a plan. **Set it repo-wide to `30000` in `.claude/settings.json` `env`**. Set `ENABLE_PROMPT_CACHING_1H=1` there also. It has no effect on subscription sessions. It raises the cache TTL of an API-key session from 5 minutes to 1 hour.

The system **enforces** tier 1 in part. It is not only advice. The rule `session-economics-quiet-reporter` (`.sigmashake/rules/session_economics.rules`) makes a FORCE redirect on a *bare* `vitest run` / `bun test` / `git log` / `git diff`. It uses `ENDS_WITH` and not a regex. The DSL has no negation operator. nfaregex (Thompson NFA, Zig dialect) has no lookahead. The rule anchors on the bare suffix. Thus it stops matching when you add any flag. It has no quantifier or alternation. Thus it cannot become a match-everything predicate. `bun run test:all` / `gov:test` are not matched on purpose.

**Order with the local stack:** `stk` / `headroom` reduce the output *before* it reaches the context. They are still the first choice for output that is verbose but wanted. `stk` is stateless line reduction. `headroom` is type-aware, and you can retrieve its output byte for byte with `headroom get <handle>`. `BASH_MAX_OUTPUT_LENGTH` fires after them, on what is left. Power-heavy commands still go through `bun run gate` / `gate:test` / `gate:push` in all cases. A quiet flag does not replace a power gate. A gate wrapper does not replace a quiet flag.

---

## 6. Thinking budget

`MAX_THINKING_TOKENS=0` turns off extended thinking. Use it for a **grunt-work session** (mechanical port, docs sync, rename sweep, mechanical parity fill).

Constraints here:

- Set it **at launch**. Never set it during a run.
- Do **not** use it for the orchestrator main loop (`claude-opus-5-5`). That loop is exempt from the intent of this setting. Fable is deprecated and is not a main loop.
- Do **not** use it for a run that will reach Phase 2 (architecture + FMEA) or a Phase 4 failure diagnosis. The pipeline keeps thinking for these phases.
- It is a *session* setting. Thus it applies to the orchestrator. A mechanical dispatched slice gets its low cost from its **explore tranche** (`Claude Haiku`) and a bounded contract. It does not get it from this setting. Slug map: `ref/model-routing.md`.

---

## 7. Loops

**Never drive an iteration loop with `/loop` from inside a live `/mono` session.** `/loop` replays the full conversation on each iteration. A loop that starts at turn 60 pays again for turns 1–60 on every tick. The cost grows with the square of the run.

Correct shapes:

- Start the loop in a **fresh terminal session**. The context has only the prompt of the loop.
- Or use the own design of the pipeline. The **autonomous sync loop** (`ref/autonomous-sync-loop.md`) is a *re-callable* `/mono`. In each iteration, it orients again from **durable state** (RSI ledger + git + `.sigmashake/production-readiness.json`). It schedules the next iteration with `ScheduleWakeup`. Thus each iteration starts almost cold by design. It also survives compaction. This is a cost property, not only a resilience property.
- Do not schedule wakeups at short intervals to poll work that the harness already reports to you. A long fallback heartbeat costs less and is equally safe.

---

## 8. `/mono` run checklist (the whole file, condensed)

**At launch**
- [ ] Fresh session (`/clear`, after `/rename` if you can want the prior session again).
- [ ] `/model` + `/effort` set. Do not touch them again in this run. Do not toggle Fast mode.
- [ ] `/context` run **once**. Unneeded MCP servers dropped with `/mcp`.
- [ ] `MAX_THINKING_TOKENS=0` only if this is a mechanical run with no diagnosis (never the orchestrator main loop — `claude-opus-5-5` — never a Phase-2/4-diagnosis run).

**During**
- [ ] Pivotal files `@`-mentioned once. Not searched. Not read again.
- [ ] Noisy commands: quiet flag → subagent → `BASH_MAX_OUTPUT_LENGTH`; `stk`/`headroom` before the backstop; power gates unchanged.
- [ ] Bad turns rolled back with `/rewind`, not `/compact`.
- [ ] Long run: set to preserve stated in the run; `/autocompact 200k` armed if the net was off.

**At the break**
- [ ] Leaving the keyboard for >1h → `/compact` **while the cache is still valid**, then walk away.

**At the end**
- [ ] Report emitted. Work committed **and pushed** (Phase 5).
- [ ] `/clear` before the next unrelated task. Not `/compact`.
