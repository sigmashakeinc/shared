# SigmaShake Mono — Agent Guide

All agents MUST read this first. Detail is not duplicated here — it lives in: per-project `CLAUDE.md`/`README.md` (every `sigmashake-*/` dir has one), the `bun run <project>:<action>` scripts in `package.json`, and `/skill-name` skills in `shared/agent-config/skills/`. SSG governs every tool call automatically (`.sigmashake/rules/*.rules`, 199 transpiled into `.gemini/policies/ssg-governance.toml` — never hand-edit); 118 ambient knowledge entries load at session start, query more with `ssg recall <keyword>` before debugging. Two product lines: the governance product (`ssg` / sigmashake.com, the B2B business) + the Twitch/YouTube streaming op that markets it.

## Guardrails (rule-enforced where a rule-id is listed; otherwise behavioral — honor them yourself)
- **Low dissent / do not debate:** execute the requested edit immediately. No alternative-architecture pitches, stylistic critiques, or "are you sure?" prompts unless explicitly asked to weigh tradeoffs.
- **Zero sycophancy:** no greetings or trailing summaries. Output the fix and the relevant code blocks, nothing else.
- **Single-path:** keep subagents aligned; no parallel reasoning or multi-agent validation loops (conserve tokens).
- **Elixir tests:** NEVER invoke `mix test` (any flags, incl. `--only`, `--listen-on-stdin`, `timeout … mix test`) from a subagent on this 128-core box — the OTP 28 erl_child_setup bug silently hangs the wrapper. Verify with `mix compile`/grep instead; the main agent runs test verifications synchronously when asked. (Detail: `ssg recall elixir_otp28_mix_test_detail`)
- **Power safety (PSU OCP — this box hard-resets on CPU+GPU co-spikes, no thermal/MCE trace):** NEVER lift/clear the GPU cap (`nvidia-smi -pl >250`, `-rgc`, `-pm 0`) or CPU boost/`scaling_max_freq`/`cpu_power_limit.sh --reset`; never stop/disable the powercap services or manually start the gate-owned GPU services (`sigmashake-ollama`, `sigmashake-navi-tts`). Route heavy build/test through `bun run gate`/`gate:test`; Dockerized Go tests ONLY via `scripts/docker-build-gate.sh` (alias `bun run docker:gate`) — a raw `docker run … go test/build/vet` is DENY'd even with --cpus/--memory. (Enforced: `power_*` family in `power-safety.rules` + `bash_exec_powergate.rules` · detail: `ssg recall power_safety_runbook_detail`, `docs/ops/power/README.md`)
- **Git/deploys:** stage files explicitly, never commit compiled binaries (enforced: `no-git-add-all` + `no-commit-binaries`, `bash_exec_agent_constraints.rules`); Worker deploys pass-through by default (`SSG_DEPLOY_STRICT=1` re-gates).
- **Pricing model — free to download + install; $5/month unlocks every value surface (2026-06-19 pivot).** No free tier/trial of the *value*, no free/anonymous/unauthenticated path to a value surface — fail-closed, gated-by-default. Open distribution channels (npm/PyPI/Homebrew/apt/Flatpak/AUR/GHCR/winget) REOPENED 2026-07-01; Cargo alone stays hard-blocked (Rust→Go removed the crate) — reopening channels broadens free-download distribution only, it is NOT a value leak. (Detail — full 2026-06-19 + 2026-07-01 semantics verbatim: `ssg recall pricing_value_surface_model`)
- **Legal entity is "SigmaShake" — never claim incorporation.** Sole-operator individual entity, NOT a corporation — never write the ', Inc.' suffix, a Delaware corporation, or an 'incorporated in <US-state>' claim on any surface. If a legal slot is needed: `SigmaShake (sole-operator individual entity, USA)`. (Enforced: `rsi-no-incorporation-claim`, `rsi-anti-cheat.rules` · detail: `ssg recall legal_entity_policy_detail`)
- **Coordination board (all harnesses, all hosts — `sigmashake-messageboard`, 127.0.0.1:7934):** the SessionStart hook injects the digest; `bun run messageboard:cli -- claim <paths> -m "<what>"` before editing, `-- done <work_id>` when finished, push via `board push -- <gated push>` (rule-forced: `messageboard-force-board-push`), `-- attest` per host so ssg + Desktop health shows per platform. Detail: `ssg recall messageboard`.

## Stop and ask ONLY when
Code execution fails, a file path is missing, or the instruction is a direct logical paradox. Otherwise proceed.

## Disk
`bun run disk-report|cleanup`; never delete load-bearing paths under `~/.sigmashake/`. (Detail: `ssg recall disk_protected_paths`)

## Structure

See [STRUCTURE.md](STRUCTURE.md) — parser-canonical project map (parsed by `sigmashake-workspace/src/build-registry.ts`).

## Agent OS → graphify (codebase context, in this order)

**Agent OS first** (`sigmashake-agent-os/`, 127.0.0.1:7930, entitlement-gated, `ssg recall agent_os_context_first`): run `bun run agent-os:context` (also injected by the project SessionStart hook) → `freshness` → `search-route` → `search`/`explain`/`diagnose` via CLI or MCP before working on the codebase; daemon down → record `agent-os: unavailable — <reason>` and continue. Then graphify/oracle for graph + symbol detail.

Knowledge graph at `graphify-out/` (enforced routing: `force-grep-tool-to-graphify.rules` redirects the Grep tool here). Commands: `graphify query "<question>"` / `graphify explain "<concept>"` / `graphify path "<A>" "<B>"` — a scoped subgraph, usually much smaller than `GRAPH_REPORT.md` or raw grep output; fall back to `graphify-out/wiki/index.md` for broad navigation or `GRAPH_REPORT.md` for architecture review. After code: `bash scripts/graphify-stamp-update.sh` (bounded incremental, 30s, skip-if-lock); full: `bun run gate graphify update .`.
