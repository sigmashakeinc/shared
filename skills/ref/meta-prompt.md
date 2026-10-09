# Meta-Prompt Synthesis (Prerequisite 2)

Load this file when you synthesize the meta-prompt. This is **every goal-driven `/mono` run**. The
typical workflow has these steps:
1. The operator gives `/mono <initial goal prompt>`.
2. You expand the raw prompt into a compact, structured **meta-prompt**.
3. You adopt it as your binding self-enforced goal.
4. You run the 7-phase pipeline against it.

You do **not** arm `/goal`. It is a UI command, not a skill. If you call it through the Skill tool, it
gives an error. Only the operator can type it. The meta-prompt binds you in the same way in all cases
(see SKILL.md Prerequisite 3). The meta-prompt is the single source of truth of the run. Phase 1 scope,
Phase 2 design + FMEA, and the Requirement of the final report all come from it.

Skip synthesis only in **autonomous re-call (sync-loop) mode**. There is no new raw goal in this mode.
Orient from durable state instead (`ref/autonomous-sync-loop.md`).

## Procedure
1. **Read the raw goal once.** Capture the intent. Do not open impact-set files. That is the work of the
   read-only scope subagent of Phase 1.
2. **Take constraints from the context that you already hold.** These are the invariants of this skill,
   `CLAUDE.md`, recalled memory, and the apparent surface. Do **not** spend reads to derive them again.
3. **Fill the 10 fields below**, compactly. If a field has nothing to say, write an explicit `none`.
4. **Emit it, adopt it automatically, and go straight into Phase 0.** There is no separate approval
   gate. The single Phase 2 `AskUserQuestion` (from the Open-questions field) is the one human
   checkpoint. If a field is really uncertain and the answer changes what you build, put it in Open
   questions. Otherwise, pick the obvious default and record it.
5. **If the scope changes during the run, update the meta-prompt** (and any `/goal` that the operator
   armed). Then the durable state stays true.

## Template
```
META-PROMPT
Objective:            <one sharpened sentence — the goal, de-fuzzed>
Acceptance criteria:  <testable bullets; what "every gate green" means FOR THIS goal>
  - <criterion 1>
  - <criterion 2>
Likely affected:      [sigmashake-<a>, sigmashake-<b>]   # initial guess; Phase 1 refines
Invariants in play:   [pricing/value-surface · power-safety · coverage-floor ·
                       legal-entity · Go-is-shipped-runtime · open-channel-reopened-except-cargo · …]
Risk hotspots:        <FMEA seeds — what could break, specific to this change>
Out of scope:         <explicit non-goals; the fence against scope creep>
Open questions:       <uncertainties → fold into the single Phase 2 AskUserQuestion>   # or: none
Assumptions:          <forks you resolved yourself — the reading the wording + surrounding code most directly support; state each, don't build for the other readings>   # or: none
Follow-ups:           <closure-queue seed: pre-existing bugs / perf / behaviour the Objective doesn't need — Phase 5.5 fixes each one after the primary land unless a closed-list blocker applies; never fixed or dropped silently>   # or: none
/goal string:         <the directive the operator COULD type as /goal; you self-enforce it regardless — never invoked>
```

## Synthesis rules
- **Clarify. Never invent.** Expansion means that you sharpen the intent and show the constraints that
  are already in force. Do not add features, surfaces, or "nice-to-haves" that the user did not ask
  for. Each acceptance criterion must trace to the raw goal or to an invariant that is in force.
- **Out-of-scope is mandatory** when the goal is broader than one change can land. Name what you do
  *not* do. Then scope creep is visible.
- **Stay lean.** The meta-prompt is a spec, not an essay. It stays in the context of the orchestrator
  as the standing directive (Token Discipline). Keep it to the 10 fields.
- **Invariants are a checklist, not prose.** List only the guardrails that this goal touches. Then the
  pricing, coverage, store, and power checks of Phase 6 are expected, not found late.
- **Coverage class goes here.** If the Objective touches a critical-production surface (gov Go runtime,
  Worker value-surface fleet, the three desktop shells, `sigmashake-workspace`), say so in Invariants
  (`coverage-floor`). Then Phase 1 arms `ref/coverage.md`.

## Standing directives every meta-prompt binds
These directives also bind the `constraints` of every dispatched slice (`ref/model-routing.md` bounded
worker contract). The orchestrator that binds them is Claude Opus 5.5. Fable is deprecated and is not a
tuning target.
- **Completion.** The run is autonomous. The operator cannot answer during the task. Thus "Shall I…?"
  blocks the work. If an action is reversible and the Objective already covers it, do it. Stop only for
  actions that are destructive or change the scope (the Phase 2 `AskUserQuestion` is the one
  checkpoint). Before you end a turn, a plan or an unasked question means that the work is not done.
  Act with tool calls instead. The run does not end while a `FIX-NOW` follow-up is open (Phase 5.5). The run ends only at the Final Report or at an honest `BLOCKED`. A command
  that changes state needs evidence for that action.
- **Scope = deliverable.** The acceptance criteria are the whole deliverable. Never narrow them, widen
  them, or swap them. Keep the primary slices focused: do not widen the primary diff. Put unrequested
  bugs, performance problems, and behaviour in `Follow-ups`. Phase 5.5 fixes them after the primary land,
  unless a closed-list blocker applies. If the goal is ambiguous, use the most direct reading and
  record it in `Assumptions`. If a part is blocked, finish the rest and say what you left out.
- **Tests sized to the task.** Commit tests only where the acceptance criteria or the own convention of
  the package call for them. Size them like the neighbouring test files. Do not commit scratch checks.
  An armed coverage floor overrides this upward.
- **Surgical edits.** Make in-place edits and not whole-file rewrites when the result is the same. A
  rewrite costs about 5× the output tokens.
- **External facts.** Tag fast-moving facts with `external-fact:` in Risk hotspots and send them to the
  Phase 2.5 research. These facts are: an upstream version, a vendor or store policy, a CVE, and a
  post-cutoff SDK surface. A fact with no citation is `UNVERIFIED`. Never state it with confidence.
- **Batching.** List what you need next. Then request, in one tool-call turn, every item that does not
  depend on the result of another item.
- **Progress + prose.** State the start of each phase in one line. Close with a recap that stands on its
  own: the Final Report. Do not use mannered prose. Use direct statements, not metaphor or flourish.
- **Compaction.** If auto-compact fires, preserve these items:
  - difficulties and how you resolved them;
  - approaches that you tried or dropped, and why;
  - the exact META-PROMPT block;
  - the current phase and every gate verdict so far;
  - open or promised items;
  - details that are hard to reconstruct (SHAs, coverage %, paths, board work-id);
  - the follow-up queue file `~/.sigmashake/mono-run-<work-id>/followups.md` (Phase 5.5).

  Keep the wording of the operator verbatim. Condense your own reasoning to conclusions. (This extends
  the compaction contract of SKILL.md.)
- **STE.** Write documentation, code comments, and instructions in ASD-STE100 (`ref/ste100.md`).

## How it threads into the pipeline
- **Phase 1 (Scope):** `Likely affected` is the seed for the blast-radius map of the scope subagent. The
  subagent confirms or corrects the list. `Invariants → coverage-floor` decides if `ref/coverage.md` arms.
- **Phase 2 (Architecture):** `Risk hotspots` are the seed for the FMEA. `Open questions` are the single
  `AskUserQuestion`. `Acceptance criteria` + `Out of scope` bound the design.
- **Phase 3 (Implement):** The standing directives and the current `Assumptions` go in the `constraints`
  of every dispatch. Each engineer returns `assumptions:` + `follow_ups:`. The orchestrator merges
  `assumptions:` back into the meta-prompt and `follow_ups:` into the follow-up queue (Phase 5.5).
- **Reporting:** The `Objective` is the **Requirement** line of the final report. The acceptance
  criteria are the pass/fail checklist that the gates attest against. `Assumptions` has its own report
  line. `Follow-ups` is the closure ledger (found · fixed with SHAs · `BLOCKED` with reason · `FIX-NOW` open = 0).

## Ambiguity handling
- **Underspecified goal** ("make onboarding better"): Synthesize the most defensible Objective. Put the
  real forks in **Open questions**. Let the Phase 2 `AskUserQuestion` resolve them. Do not stall before
  Phase 0.
- **Goal that violates an invariant** (for example, a free path to a value surface, lifting a power
  cap, or an incorporation claim): The meta-prompt names the conflict in Risk hotspots. The Objective
  changes to the compliant intent. If the goal *is* the violation, show it as the first Open question.
  Do not build it silently.
- **Multi-product goal:** List every product in `Likely affected`. Phase 2 gives the dispatch to
  `monorepo-team-lead` (one disjoint slice for each `<slug>-principal-engineer`).

## Worked example 1 — single-product feature
Raw goal: `/mono add a rate-limit header to the metering counter response`
```
META-PROMPT
Objective:            Return remaining-quota + reset headers on metering's POST /v1/counter so the ssg CLI can warn before cap.
Acceptance criteria:
  - POST /v1/counter responds with X-RateLimit-Remaining + X-RateLimit-Reset (epoch s).
  - Header values match the KV-tracked per-device monthly count/cap.
  - sigmashake-metering tests cover present + over-cap + new-device cases; test:all green.
Likely affected:      [sigmashake-metering]
Invariants in play:   [pricing/value-surface]   # metering gates the free tier — must stay fail-closed
Risk hotspots:        header drift vs body tier; clock source for reset; KV read race on first call.
                       external-fact: IETF ratelimit-headers draft field names → Phase 2.5 research or UNVERIFIED
Out of scope:         changing the cap value; CLI-side rendering of the header.
Open questions:       none
Assumptions:          header names follow the X-RateLimit-* convention
Follow-ups:           none
/goal string:         add remaining-quota + reset headers to metering POST /v1/counter
```

## Worked example 2 — cross-product change
Raw goal: `/mono surface the new SHAKEDOWN score on the marketing site`
```
META-PROMPT
Objective:            Publish the latest public SHAKEDOWN benchmark score on sigmashake.com, sourced from the bench leaderboard.
Acceptance criteria:
  - sigmashake-web renders the current score from a bench-published source (no hand-typed number).
  - bench exposes the score via its existing public leaderboard surface (public proof surface — OK).
  - Both packages' tests green; no value-surface gated content exposed.
Likely affected:      [sigmashake-bench, sigmashake-web]
Invariants in play:   [pricing/value-surface (SHAKEDOWN is a deliberately-public proof surface) · legal-entity (no "Inc." in copy)]
Risk hotspots:        stale/cached score; fabricated number if the feed is missing; marketing copy overclaim.
Out of scope:         redesigning the leaderboard; per-model breakdown on the marketing page.
Open questions:       Show a single headline score, or score + date + model? → Phase 2 AskUserQuestion.
Assumptions:          none — the fork goes to the AskUserQuestion
Follow-ups:           none
/goal string:         publish the live SHAKEDOWN score on sigmashake.com from the bench leaderboard
```
Two products mean that Phase 2 dispatches `monorepo-team-lead` → `bench-principal-engineer` + `web-principal-engineer`.

## Autonomous re-call adaptation
In sync-loop mode, there is no raw goal to synthesize. Use the **binding-constraint slice** that you
chose from durable state (RSI ledger + git + `.sigmashake/production-readiness.json`) in place of the
Objective. Derive the acceptance criteria from the parity-matrix cell that you close. Then continue.
Do not use `AskUserQuestion`. See `ref/autonomous-sync-loop.md`. `Assumptions` and `Follow-ups` still
apply. The ledger entry records both. It also records the closure ledger of the iteration.
