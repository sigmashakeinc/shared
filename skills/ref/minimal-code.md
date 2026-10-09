# Minimal-Code / Decision Ladder (ponytail technique)

Load this on **every implementing `/mono` run** (Phase 3 dispatch and Phase 4 review). It is the
"lazy senior developer" posture. Write only the code that must exist. Climb a decision ladder
before you write. This is adapted from the ponytail ruleset. The default intensity is **full**. The operator can
override it with the `/mono` argument, for example `/mono lite: <goal>` or `/mono ultra: <goal>`.

SSG enforces the rules. Prose alone does not:
- The rule `package-install-guard` enforces the **dependency** rung (ASK on new installs).
- The rule `agent_decision_ladder` (`decision-ladder-new-source-file`, LOG nudge on `Write` of a source file) enforces the **new-file / new-abstraction** rung.

The orchestrator gives this ref to the Phase-3 engineers. In Phase 4, the orchestrator gates the diff against it.

## The Decision Ladder — climb before you write
Before you write new code, evaluate the rungs **in order**. Stop at the first rung that satisfies the
requirement:
1. **Does this need to exist at all?** → If no, skip it (the cheapest code is no code).
2. **Is it already in the codebase?** → Reuse or extend it. Search before you author. Do not re-implement.
3. **Does the standard library cover it?** → Use stdlib.
4. **Is a native platform feature available?** → Use it.
5. **Does an already-installed dependency do it?** → Use it. Do **not** add a new dependency. `package-install-guard` ASK-gates that rung. Verify on the registry first.
6. **Can you express it as one line or inline?** → Do that. Do not make a new file or abstraction.
7. **Only then** write the minimal implementation. If an existing module fits, edit it. Do not add a
   new wrapper, adapter, util, or helper file.

## The boundary — lazy about solutions, NEVER negligent about requirements
Minimal does not mean unsafe. This is **not** code-golf. The reduction comes from necessity. It does not come from
less rigor. These items are **non-negotiable**. Never trim them to save lines:
- input **validation** and error handling at trust boundaries,
- **security** (authz/authn, injection, secret handling — the existing DENY safety rules do not change),
- **accessibility** (a11y),
- **data-loss** and durability handling,
- the project **invariants** (pricing/value-surface, power-safety, coverage floor, legal-entity, …).
If a rung forces you to give up any of these items, you have not satisfied the requirement. Keep climbing,
or write the necessary code. Requirements always win over brevity.

## Intensity dial (default: full)
| Level | What the ladder gates |
|---|---|
| `off` | ladder disabled (record why in the report; only for a forced exception). |
| `lite` | rungs 1–2–5 only (need-to-exist? · reuse existing · already-installed dep). Skips deep abstraction review + LOC-delta scrutiny. Fastest. |
| **`full`** (default) | all 7 rungs every implementing run + the Phase-4 over-engineering diff review below. |
| `ultra` | full **plus** a repo-wide reuse search before any new file and a mandatory LOC-delta justification per new module. Most thorough. |

## Phase-3 engineer directive (put this in every implement dispatch)
The structured return of each engineer must include a **Decision-ladder** block:
- the highest rung that satisfied each piece of work (reused X / used stdlib Y / one-liner / new file Z);
- every **net-new file or abstraction**, with a one-line reason why a lower rung did not fit;
- at `ultra`: the LOC delta and the result of the repo-wide reuse search that cleared the new file.

Do not add a new dependency without the `package-install-guard` ASK. Do not add a new wrapper around something that already
exists.

## Phase-4 over-engineering diff review (the ponytail-review equivalent — `full`/`ultra`)
Scan **the diff of this run** (`git diff --stat origin/main HEAD` + targeted reads only of new files). Record a one-line **verdict** in the report. Flag these items. If you find one, send the work back to Phase 3:
- a **new dependency** that an installed dep, stdlib, or native feature already covers;
- a **new file / wrapper / adapter / util** that duplicates existing functionality (rung 2 missed);
- **speculative abstraction** (generality, config knobs, hooks) for a need that does not exist yet (YAGNI);
- multi-line code that collapses to a clear one-liner (rung 6) **without** harm to readability or the
  requirements boundary above.

This review is **advisory with a recorded verdict**. It is not fail-closed. Over-engineering is a judgment, not
a binary gate. But a flagged item with no justification goes back to Phase 3. It does not ship. Never trade
away a requirement to clear a flag. Note "kept: <requirement>" instead.
