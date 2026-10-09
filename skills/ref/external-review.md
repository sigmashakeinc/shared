# External review + deep research — Codex reviewer, Claude research subagent

Load this ref when a run needs an **external** opinion: live-web research, or an adversarial
review of a plan or diff. Both are **advisory**. Neither blocks Phase 3 dispatch. A Phase 4 pass
does not need either of them (review-as-a-mandatory-gate stays deprecated per Phase 2.5).

## Two lanes — pick by task

| Lane | Provider | Route | Use it for |
|---|---|---|---|
| **Deep research** | Claude subagent (`haiku`; `sonnet` if judgment is needed) | Dispatch it as a normal slice. Give it WebSearch/WebFetch and a read-only prompt. | Anything that needs the **live web**: upstream release/version truth, vendor policy (Apple/MS store, npm/PyPI), CVEs, API/SDK docs beyond the knowledge cutoff, "is this still the recommended approach", prior art. |
| **Code / design review** | GPT-5.6 (sol/spark) via Codex | `bash scripts/codex-route.sh <model> <effort> < prompt.txt` | **Optional reviewer.** Design soundness, missed failure modes, invariant violations, diff critique, implementation diagnosis. The verified-pair matrix and `gpt56-routing.rules` govern it. |

## Research subagent contract

- The subagent returns each fact with the source URL that it fetched.
- A fact without a URL is `UNVERIFIED`. Treat that label as literal.
- Never turn an `UNVERIFIED` claim into a confident assertion in the final report.
- Never assert an external fact from memory.
- Research answers expire. If a fetched fact goes into a durable doc, record the date with it.

## Reading the Codex result — the fake-green trap

| Exit | Meaning | Correct handling |
|---|---|---|
| `0` | Non-empty response | Read it. |
| `2` | Usage error / rejected flag | Caller bug. Fix the invocation. |
| `3` | Codex missing, timed out, or empty response | **Reviewer unavailable.** Record an honest advisory skip. This is **never** "no findings" and **never** a green gate. |

Branch on the exit code. Never branch on empty output. One retry is reasonable. Do not treat an empty run as a pass.

## Prompt discipline

- Send the plan and `file:line` anchors, not file bodies.
- Use read-only framing: "identify", "assess", "cite". Never "fix", "apply", or "run the tests".
- Demand citations. Repo claims cite `file:line`. External claims cite a fetched URL.

## What neither lane can do

Neither lane can attest a gate. They do not run the tests of this repo. They cannot see the
coverage floor. Their approval is not evidence that `test:all` passed. The orchestrator verifies every enforced gate
locally (coverage ≥90/90/90/80, power gating, onboarding, store/SEO owner-maps). The blessing of a reviewer never replaces
a command exit code.
