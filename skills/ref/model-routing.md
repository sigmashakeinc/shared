# Model routing — Claude-only bounded worker contract (Opus 5.5 orchestrator; Fable deprecated; external lanes are not dispatch)

This file loads **in every dispatch, including autonomous sync-loop runs**. Its scope is wider than
`ref/meta-prompt.md`, which skips autonomous runs. It is also wider than `ref/minimal-code.md`, which
loads only for implementing runs. Routing decisions happen in every type of run. **Claude Opus 5.5
(`claude-opus-5-5`; launch `/model opus`) is the orchestrator, the brain, and the architecture model.**
Route every slice by **task class**, not by how recent the model is. External lanes are not dispatched
slices. **Fable is deprecated (2026-09-21).** It is not an orchestrator, not a worker, and not an
admission check.

## Task-class slug map

| Task class | `portfolio_tranche` / `model` | `effort` |
|---|---|---|
| Explore / search / scope + read-only bulk (cartography, verbose test/log digestion, docs/ref sweeps) | `Claude Haiku` / `haiku` | `medium` |
| Implementation / PE / diagnostic (default engineer tranche; all coding) | `Claude Sonnet` / `sonnet` | `medium` (well-specified) / `high` (diagnostic, defect-repair, cross-package) |
| Architecture / brain + the named `*-lead.md` coordinators (dispatch/integration-only; never write feature code) | `Claude Opus` / `opus` | `high` |
| Orchestrator main loop (never a dispatched slice; never named in agent frontmatter) | `claude-opus-5-5` (not a portfolio tranche) | set once at launch (Prerequisite 0) |

**Dispatchable aliases.** Claude Opus is Opus 5.5 (`claude-opus-5-5`). Claude Sonnet is Sonnet 5.5
(`claude-sonnet-5-5`). Claude Haiku is Haiku 5.5 (`claude-haiku-5-5`). Dispatch **serially and
power-bounded** with `Workflow` (preferred for fan-out then verify) or `Agent`. Never dispatch
`general-purpose`/`claude`. Never dispatch `fable`.

## Tier economics (researched 2026-10-07)

| Tranche | Model ID | Input / output $/MTok | Cache read $/MTok | Cache write (5m / 1h) | Default effort | Refusal fallback |
|---|---|---|---|---|---|---|
| Claude Opus | `claude-opus-5-5` | 4 / 20 | 0.20 | 1.25× / 2× input | `medium` | server-side available |
| Claude Sonnet | `claude-sonnet-5-5` | 2 / 10 | 0.20 | 1.25× / 2× input | `high` (API) — re-calibrated; start `medium` for agentic coding | server-side retries only `cyber` + `frontier_llm` declines |
| Claude Haiku | `claude-haiku-5-5` | 0.10 / 0.50 (prompt ≤100K tok); 0.50 / 2.50 above | 0.1× input (0.01 / 0.05) | 1.25× / 2× input | `medium` | **none** — handle in the orchestrator |

What the numbers mean for routing:
- **Cache reads are the largest part of the spend of this repo.** In the 30-day ledger to 2026-10-07, Sonnet subagents read 10.8B cached tokens and wrote 19.9M output tokens. The price of a cached token is the tier lever that matters.
- **Opus 5.5 and Sonnet 5.5 have the same cache-read price ($0.20).** Thus, on a slice with many reads, the extra cost of Opus is only in cache writes and output. This is about 1.5× Sonnet on the measured 2026-10-07 mix, not 2×. Haiku 5.5 reads cache at 1/20th of the price of either, while its prompt stays ≤100K tokens.
- **Caches belong to one model.** A subagent starts a new prefix. It does not share a cache with the parent. If you change the model of the main loop during a run, the new model must prefill everything again.

Sources: bundled `claude-api` skill (Claude Code 2.1.293) `shared/models.md:79,85,87`, `shared/model-migration.md:2234,2334,2349,2183`, `shared/prompt-caching.md:144`, `shared/cost-optimization.md:5,140,182-187`, `shared/agent-design.md:94`; https://code.claude.com/docs/en/model-config.md; https://code.claude.com/docs/en/sub-agents.md; local `~/.sigmashake/tokens.sqlite` (read-only query, 2026-10-07).

## Delegation rules (every dispatch)

1. **Delegate bulk, not chains.** Fan out only independent work, or a self-contained step with bulky intermediate results (a sweep, a verbose job, a slice for one package). A dependent chain that fits in one context stays one slice. If you split it, you pay for a plan, a handoff, and a merge that a single model gets free. In Anthropic measurements, the coordinator model alone at lower effort won every such case (`cost-optimization.md:187`). Never delegate a step if the deciding model needs its intermediate context to judge well.
2. **Effort before tier.** Lower the effort on the current tier before you drop a tier. Judge by the cost per *completed* slice, not per request. A cheap slice that needs a retry is not cheap (`cost-optimization.md:5,182-184`).
3. **Give an explicit effort on every dispatch** (Agent `effort` param or agent frontmatter `effort:`; operator directive 2026-10-07):
   - `haiku` → `medium`. Never use `low` for a multi-step sweep. At `low`, Haiku skips searches and checks in long prompts (`model-migration.md:2336`). Use `low` only for a single lookup.
   - `sonnet` → `medium` for well-specified slices (bounded contract, mechanical work, docs/config). Use `high` for diagnostic, defect-repair, cross-package, or Phase-4 failure work. Never use `low` on a slice that can edit.
   - `opus` (named leads, architecture/brain) → `high`.
   - Standing verification gates, sentinels, and read-only reviewers may default to `effort: high` in frontmatter.
   - Orchestrator main loop: set once at launch (Prerequisite 0). Never change it during a run.
4. **Haiku scope is explore + read-only bulk** (operator directive 2026-10-07). This includes cartography, search, scope, verbose test/log digestion (return the verdict line only), docs/ref sweeps, and cartographer agents. **All code edits and every honesty gate (`*-verification-analyst`, `*-sentinel`) stay on Sonnet.** Haiku 5.5 at `medium` sometimes reports work as done without a run of the check (`model-migration.md:2369`). A false CONFIRM is the failure that the gates exist to stop.
5. **Haiku context budget: ≤100K prompt tokens for each slice.** Above that limit, Haiku 5.5 bills 5× (0.50 / 2.50). Split a broad sweep into bounded slices. Do not resume a Haiku slice that is near the budget with `SendMessage`. Dispatch a new slice that carries a summary.
6. **Verification clause.** Put this sentence in every slice that edits files or runs checks: "Run the command that exercises your change and report its exit code; never report a change done without it." (`model-migration.md:2207,2369`).
7. **Refusal: go one tier up, once.** Haiku 5.5 has no server-side fallback (`model-migration.md:2349`). Sonnet 5.5 declines in five categories (`cyber`, `bio`, `frontier_llm`, `reasoning_extraction`, `general_harms`). Dispatch a refused or empty slice again, once, on the next tier (haiku → sonnet → opus). Record the refusal. For security-corpus work (SHAKEDOWN, MITRE ATT&CK, strike), expect `cyber` declines.
8. **Verify that the alias resolved.** The Agent tool `model` takes aliases only. On the first-party API, `haiku` resolves to `claude-haiku-5-5` (code.claude.com model-config). If the `resolved_model` of a slice is a Haiku ID older than 5.5, the slice ran in a session that still had the old alias map (observed 2026-10-07). Record this honestly. Never relabel it. A standing agent that must be deterministic pins the full ID in frontmatter (`model: claude-haiku-5-5`). The sub-agents doc accepts full IDs there. Pin the aliases in `.claude/settings.json` `env` (`ANTHROPIC_DEFAULT_HAIKU_MODEL=claude-haiku-5-5`, `ANTHROPIC_DEFAULT_SONNET_MODEL=claude-sonnet-5-5`, `ANTHROPIC_DEFAULT_OPUS_MODEL=claude-opus-5-5`; source https://code.claude.com/docs/en/model-config.md). Config Guard prevent mode blocks agent writes to that file, so the operator applies the pin. If you change a tier ID in this file, change the pin too.
9. **Never switch the model of the main loop during a run.** Route a cheaper sub-task to a subagent instead (`agent-design.md:94`).

**Explore, implement, and lead use different slugs** (`haiku` / `sonnet` / `opus`). Always record the **task
class** (`explore` / `implement` / `lead`).

**Explicit `model:` rule.** Always pass an explicit `model:` on every `Agent`/`Workflow` dispatch.
If you omit it, the slice inherits the Opus 5.5 parent. This is the mix bug. If a harness spawn API has no
model field, inherit the parent or the agent definition, and record `resolved_model` honestly.

**Fable is deprecated.** Do not dispatch `fable` / `claude-fable-5` / Fable 5. Do not pass `--model fable`.
Do not run a Fable ratio or admission check. Do not co-author as Claude Fable. The orchestrator confirms
that `/model` is `opus` (`claude-opus-5-5`) at launch. It does not change it during the run. If a report
still prints the historical fields, the only valid row is `fable-orchestration: deprecated`,
`fable-launch-admission: n/a-fable-deprecated`, `fable-entry-observation: n/a-no-fable-entry`.

**External lanes are not dispatched slices. They never appear in the Metric-1 mix.** Research and review
stay read-only (`ref/external-review.md`). **Codex** (GPT-5.6 sol/spark, `scripts/codex-route.sh`) is the
optional external reviewer. It is read-only and advisory. It spends tokens outside the dispatch family.
It carries no `portfolio_tranche`. Deep research is a normal Claude subagent (`haiku`, or `sonnet` for judgment). It is a dispatched slice.
The approval of an external worker is never evidence that a local command exited 0.

## Bounded worker contract — full per-field spec
Every dispatched slice carries these fields:
- `slice_id`, `role`.
- `portfolio_tranche` (`Claude Haiku` / `Claude Sonnet` / `Claude Opus`).
- `model` (exact ID or tier, or `unsupported: <reason>`).
- `effort` (the default of the tranche from the slug map; required on every slice).
- **task class** (`explore` / `implement` / `lead`).
- A bounded `task`, `scope`, `allowed_edits`, `constraints`, `expected_output`, and the required return shape.

Set `allowed_edits` to `none` by default for exploration and review. A worker that can edit gets exactly
one disjoint write scope. It never shares files with another live worker.

The required return also carries:
- `resolved_model`: the exact value that you read from the session metadata, or `unsupported`. A successful command is not proof of the model selection.
- `usage` token classes: `input`, `cached_read`, `cache_write`, `visible_output`, `reasoning_output`. Each is a number or `unavailable`. Never write zero if the value is unknown.
- `assumptions:` and `follow_ups:`. Each is a list, or `none`.

Treat all worker output as advisory. It stays advisory until the orchestrator integrates it and a
reviewer or verifier pass clears the blocking findings. This pass is optional and model-agnostic
(Phase 2.5/4). If one worker fails, keep the successful outputs. Do not discard the whole batch.

Orchestration must be non-interactive and headless. Do not use tmux panes, `send-keys`, terminal-focus
routing, or prompts that wait on a worker. If you use a non-native or headless runner, it must write
prompts, stdout, stderr, and results to a durable state directory. The orchestrator reads the summaries
and only the raw output that it needs for integration.

**Standing constraints** — apply them to every dispatched slice. The orchestrator is Opus 5.5. Fable is
deprecated and is not a tuning target:
1. Scope is the `task` of the slice. Put pre-existing bugs, performance problems, and behaviour outside the scope under `follow_ups`. Do not fix them, unless the slice cannot work without the fix. The orchestrator merges `follow_ups` into the Phase 5.5 closure queue and fixes them after the primary land. If the task is ambiguous, use the most direct reading and record it under `assumptions`.
2. Write tests only where the slice asks for them or the package already keeps them. Size them like the neighbouring test files. Do not commit scratch checks. An armed coverage floor overrides this upward.
3. Make surgical in-place edits. Do not rewrite a whole file when the result is the same.
4. Finish the slice. Do not ask "Shall I…?". Put questions in the return shape. The slice ends only when the worker process exits.
5. The dispatch instruction is written in STE (ASD-STE100). The worker writes new or changed comments and docs in STE (`ref/ste100.md`).

## Operational invariants for edit-capable dispatch
1. **One live worker that can edit for each write scope, always.** A slice ends when its process exits. It does not end when its output file appears. Wait for the exit status of the child, and confirm that no survivors remain, before you treat the slice as done. Observed 2026-07-16: a slice kept writing 17 minutes after its file landed, while a second slice was dispatched into the same scope. This gave two live workers in one write scope and mutations with a wrong owner.
2. **Nothing inside a scope that a child can write is evidence about itself.** A worker can rewrite in-scope evidence of its own compliance. Thus any verifier script, allowed-edits file, exec log, or prompt hash that you use to check the output of a worker must be outside the write scope of that worker. Verify from a trusted copy that is outside the writable area of the worker.

## Metric 1 — slice-accounting mechanics
Count one semantic executor for each logical slice. PE scoping and integration is a different logical slice
from engineer implementation. A failed attempt does not satisfy the executor share of that slice. If a
different tier completes a fallback, that tier is the terminal semantic executor. The failed attempt stays
separately visible. Metric 1 records the actual mix of each run **by task class AND slug** (explore,
implement, lead, plus the exact model). There is no target ratio for a run. Never create extra agents only
to change the mix. Explore (+ read-only bulk) is `haiku`, implement is `sonnet`, and lead is `opus`. These
are different slugs. Still record the task class.

### Spend target
- Target: 30% Opus / 65% Sonnet / 5% Haiku, as share of RECOMPUTED cost over a rolling 7-day window from sigmashake-tokens (127.0.0.1:7913, health route `/healthz`).
- Recompute cost from the token columns with the HEAD table in `sigmashake-tokens/src/pricing.ts`. Stored `events.cost` from before e66a1091e is wrong.
- Report the token share next to the cost share. Report the /mono main-loop share of Opus separately.
- It is an account-level measurement target. It is not a dispatch quota. Never add agents or move work to a wrong tier to change the ratio. Report a breach honestly.
- Main lever: keep the /mono main-loop context under the 300k ceiling. Summarize, discard raw material, delegate reading.
- Baseline 2026-10-01..08: Opus 42.6% / Sonnet 57.0% / Haiku 0.4% of $952.79. 95% of Opus spend was the /mono main loop. 14 of 23 main loops peaked above 300k.

## Model-independence contract
The harness is the quality bar. It consists of the phase gates, the optional model-agnostic review tooling,
the coverage/power/onboarding gates, and the specialized agents. The quality bar is not a claim that one
retired model is the target. The main loop is `claude-opus-5-5`. Fable is deprecated. Do not launch it to
reach a historical parity score. The durable quality bar that every agent and the orchestrator inherit is in
`shared/agent-config/AGENT-QUALITY-BAR.md`. It appears at session start through the
`succession_quality_bar` AMBIENT entry. This entry is **currently staged** in
`shared/agent-config/staged-rules/`. It waits for operator placement and `ssg rule sync`. It is active
after placement. `sigmashake-succession` stays a measurement tool. A Fable-5 parity figure is not a reason
to dispatch Fable.
