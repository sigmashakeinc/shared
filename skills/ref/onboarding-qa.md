# ref: Onboarding & Install QA Conformance — load when the impact set touches an onboarding/installation surface of SigmaShake Desktop or the `ssg` CLI (see the Conditional-Integrations row), OR a feature change adds a user-local install / first-run / `ssg init` requirement. Not for backend-only config, arbitrary gov eval/rule, or Worker business-logic changes.

**Only** the standing **qaonboard** team does onboarding/install QA (never general-purpose). Every `/mono` run that touches onboarding must clear the clean-room install → first-run suite. Run it the way a user runs it, on every OS, to a strict zero-tolerance bar. (background — currently staged pending operator `ssg rule sync`; `ssg recall mono_onboarding_qa_history`)

**Team** — root `.claude/agents/` (never general-purpose). The entry owner is **`qaonboard-lead`**. It is **dispatch-only**. It owns the standard, slices the work, and gates through the verification analyst. It never writes code. Its engineer slices are disjoint. Sometimes a needed slice does not exist yet. In that case the /mono orchestrator authors it (≤15k tokens) and runs `sync-root-agent-symlinks.sh` before dispatch (Prereq 4.3). The lead does not author slices. If a required slice cannot be authored, the result is `BLOCKED` (never a false pass):
- `qaonboard-install-engineer` — CLI install (hermetic fixture-CDN harness `sigmashake-dist/test/install/` + `SSG_INSTALL_BASE_URL`). Categories **O1–O4, O9–O10**.
- `qaonboard-playwright-engineer` — web onboarding (Playwright `onboarding` project vs the live `ssg serve` setup+wizard SPA on :5599). Categories **O5–O8, O12**.
- `qaonboard-desktop-shell-engineer` — native desktop glue (`sigmashake-desktop-linux/internal/…` supervise/adopt/port-readiness Go tests). It defines the win/mac conformance contract. It is honest-red where it cannot build. Categories **O5, O8, O11**.
- `qaonboard-verification-analyst` — read-only acceptance gate. It never edits.

**Source-of-truth ledgers** (the canonical "every onboarding surface accounted for"):
- Standard — `docs/qa/onboarding-install-qa-standard.md`: onboarding/install categories **O1–O12** (additive to the 19), how to test each surface, the **per-OS parity matrix (O11)**, and the **Strictness bar & friction budget**. ← OWNS the category list, parity matrix, and strict bar.
- Ledger — `docs/qa/onboarding-install-issues.md`: every finding with severity, per-OS status, testability fixes, per-OS rollup, and deploy log. It is also the baseline that the strict gate enforces against.

**What counts as an onboarding surface (the trigger):**
- `ssg init` / device-auth (`sigmashake-gov/go/internal/cli/{init,auth}.go` + `ssg init` output/templates).
- The curl-pipe installers (`sigmashake-web` `install.sh`/`install.ps1`, `install.sigmashake.com`), the `sigmashake-dist` install artifacts, and `sigmashake-dist/test/install/`.
- The `sigmashake-e2e` `ssg init` emulators.
- The SigmaShake Desktop setup splash, first-run wizard, first-run license/ToS gate, and daemon supervise/adopt/port-readiness in any `sigmashake-desktop-{linux,mac,windows}`.
- A feature change that adds a **user-local install / first-run / `ssg init` env or config requirement** (new setup step, install dependency, changed default port, new auth/permission prompt).
- The two `docs/qa/onboarding-install-*.md` docs.

**Not** backend-only config, arbitrary gov eval/rule bodies, or Worker business logic.

**The QA standard (what "onboarded" means, verified the way a user runs it):** clean-room install → first-run wizard walk → `ssg daemon` + `ssg serve` come up healthy → dashboard reachable on the local port → `ssg init` writes the host-correct artifacts that each AI-agent host actually parses. The standard has categories O1–O12.

**Strictness bar & friction budget (zero-tolerance).**
- **Mechanically-checkable friction budget** (the suite hard-gates it):
  - The happy path is clean install → first governed tool call with a **single command**.
  - **No manual config edits** on the happy path.
  - The required-**step count does not increase** vs the baseline.
  - **Every failure path gives an actionable, copy-pasteable next step.**
  - Time-to-first-success is an **advisory** ledger metric. Step-count is the enforceable proxy.
- **Zero-tolerance ship bar:** the onboarding suite is **green (0 failures)**. **Zero NEW P0/P1 onboarding blockers** are introduced. O11 parity is fully owned. Onboarding regressions are **release-blocking**.
- **Friction severity rubric:** P0 = the user cannot complete onboarding. P1 = it needs a manual workaround or an undocumented step. P2 = it is confusing but completable. P3 = polish.

**Owner-map gate.** Every *applicable* O1–O12 category (and every O11 per-OS parity row) maps to a **named** `qaonboard-*` owner. An honest `n/a — <reason>` is allowed. An **empty owner is a fail**. If an applicable category or row has no owner, the result is **`BLOCKED`** (Anti-Loop). The entry owner is `qaonboard-lead`. At dispatch time it resolves each category to its slice engineer. The orchestrator authors a missing slice first, as described above.

**Honest-red, never fake-green.** The macOS and Windows shells cannot build on the Linux box. The Linux leg hard-gates. Record specs that cannot run on mac/win as honest-red `partial` or `BLOCKED (peer offline)` in the O11 parity matrix. The operator attests them through the readiness manifest. Never mark them green.

**Pricing invariant.** The first-run license/ToS gate must fail **closed**. There is no anonymous or unauthenticated path to a value surface. Verify it. Never weaken it to make onboarding "smoother". (Free download and install stay open. The value is what is gated.)

**Phase 1 add-on:** classify which O1–O12 categories and owning-OS leg(s) the diff touches. Also classify whether it adds a user-local install or first-run requirement. Mark the mac/win legs honest-red at the start where they cannot build here.

**Phase 2 gate add-on:** the named team **must** be `qaonboard-lead` (never general-purpose). The design cites the per-category (O1–O12) → owning-agent map and the friction-budget DoD. Before Phase 3, every applicable category must have a named owner. The orchestrator authors any missing engineer slice first.

**Phase 4 gate:** run the `qaonboard-lead` gate. All of these must hold:
- `bun run gate:test bun run qa:onboard` (`e2e:onboarding` + hermetic `qa:install`) reports **0 failures**.
- The desktop-linux `internal/` onboarding tests pass.
- mac/win are honest-red in the O11 matrix.
- The first-run license/ToS gate is verified fail-closed.
- The happy path is one command, with no manual edits, no step-count increase, and actionable errors.
- Zero NEW P0/P1 blockers exist. Any regression → `BLOCKED`.

**Phase 6 trigger:** the qaonboard team runs the onboarding suite. Then it runs the **coverage-completeness gate**. Assert that every *applicable* O1–O12 category and O11 parity row maps to a named owning agent (an empty owner is a fail). If an applicable category or row has no owner, the result is `BLOCKED`. Do not claim "works-on-all-OS / approved" without the fully-owned parity matrix.
