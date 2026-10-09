# ref: Product-Feature QA (post-onboarding) — load when the impact set touches shipped ssg Go behavior, a dashboard SPA product journey, Desktop supervise/updater/tray/RPC, the path-aware pre-push feature gate, or the product-feature QA ledgers (see the Conditional-Integrations row). Not for onboarding/install (that's `ref/onboarding-qa.md` / qaonboard). Not Worker fleet except the `ssg fleet` CLI.

**Only** the standing **qafeature** team does post-onboarding product-feature QA (never general-purpose). The workspace 20-kind taxonomy and Bun `release-check` do **not** prove that the shipped Go binary and the three desktop shells still work after a push. Close that hole with these items: path-aware pre-push, an owner-mapped F1–F12 ledger, rocket-launch scores, and honest-red mac/win.

**Team** — root `.claude/agents/` (never general-purpose). The entry owner is **`qafeature-lead`**. It is **dispatch-only**. It owns F1–F12, the issues ledger, and the path-aware pre-push regression gate. It never writes code. Its engineer slices are disjoint. Sometimes a needed slice does not exist yet. In that case the /mono orchestrator authors it (≤15k tokens) and runs `sync-root-agent-symlinks.sh` before dispatch (Prereq 4.3). The lead does not author slices. If a required slice cannot be authored, the result is `BLOCKED` (never a false pass):
- `qafeature-gov-engineer` — shipped ssg Go + dashboard SPA. Categories **F1–F7** (eval/rules, daemon/serve, dashboard SPA, adapters/hooks, plugins, `ssg fleet` CLI, post-onboard license/entitlement; first-run ToS stays O7 / qaonboard).
- `qafeature-desktop-engineer` — native desktop after first-run. Categories **F8–F11** (supervise/adopt, updater, tray/window/RPC, cross-OS feature parity). Honest-red where it cannot build.
- `qafeature-prepush-engineer` — path-aware pre-push feature-regression membership. Category **F12** (`.githooks/pre-push`, `scripts/qa-feature-prepush.sh`, classifier self-test).
- `qafeature-verification-analyst` — read-only acceptance gate. It never edits.

**Source-of-truth ledgers** (the canonical "every product-feature surface accounted for"):
- Standard — `docs/qa/product-feature-qa-standard.md`: categories **F1–F12**, how to test each surface, per-OS parity (**F11**), rocket-launch scores. ← OWNS the category list, parity matrix, and scores.
- Ledger — `docs/qa/product-feature-issues.md`: every finding with severity, per-OS status, and rollup; residual P0s.

**What counts as a product-feature surface (the trigger):**
- Shipped ssg Go behavior (`sigmashake-gov/go/` eval/daemon/serve/hooks/plugins/fleet/entitlement).
- Dashboard SPA product journeys (`sigmashake-gov/src/client/**`, `e2e/gov`, `shared/desktop-views/**`).
- Desktop supervise/adopt, updater, tray/window/RPC in any `sigmashake-desktop-{linux,mac,windows}`.
- `.githooks/pre-push` and `scripts/qa-feature-prepush.sh` (and `scripts/qa-feature-prepush.test.ts`).
- The two `docs/qa/product-feature-*.md` docs.

**Not** onboarding/install (`ssg init`, curl-pipe installers, first-run wizard/ToS — qaonboard / `ref/onboarding-qa.md`). **Not** Worker fleet except the `ssg fleet` CLI (F6).

**Owner-map gate.** Every *applicable* F1–F12 category (and every F11 per-OS parity row) maps to a **named** `qafeature-*` owner. An honest `n/a — <reason>` is allowed. An **empty owner is a fail**. If an applicable category or row has no owner, the result is **`BLOCKED`** (Anti-Loop). The entry owner is `qafeature-lead`. At dispatch time it resolves each category to its slice engineer. The orchestrator authors a missing slice first, as described above.

**Honest-red, never fake-green.** The macOS and Windows shells cannot build on the Linux box. The Linux leg hard-gates. Record specs that cannot run on mac/win as honest-red `partial`, `BLOCKED (peer offline)`, or a printed SKIP. Never mark them green. `SSG_SKIP_PREPUSH=1` is for emergencies only. It must stay visible in the ledger.

**Pricing invariant.** Post-onboard entitlement and value-surface gates stay **fail-closed**. First-run license/ToS is O7 / qaonboard. Do not take over that surface.

**Phase 1 add-on:** classify which F1–F12 categories and owning-OS leg(s) the diff touches. Mark the mac/win legs honest-red at the start where they cannot build here.

**Phase 2 gate add-on:** the named team **must** be `qafeature-lead` (never general-purpose). The design cites the per-category (F1–F12) → owning-agent map. Before Phase 3, every applicable category must have a named owner. The orchestrator authors any missing engineer slice first (Prereq 4.3). `qafeature-lead` is **dispatch-only**.

**Phase 4 gate:** run the `qafeature-lead` gate. `bun test scripts/qa-feature-prepush.test.ts` must report **0 failures**. Do **not** require full `test:all` only because this ref loaded. The classifier self-test is the feature-QA gate. `test:all` stays the SKILL.md default unless the leaf carve-out applies. Keep mac/win honest-red (never fake-green a leg that did not run). Zero NEW P0/P1 feature blockers are allowed. Any regression → `BLOCKED`.

**Phase 6 trigger:** the qafeature team runs the **coverage-completeness gate**. Assert that every *applicable* F1–F12 category and F11 parity row maps to a named owning agent (an empty owner is a fail). If an applicable category or row has no owner, the result is `BLOCKED`. Do not claim "10/10 rocket-launch" without scores in `docs/qa/product-feature-qa-standard.md` (no rocket-launch / 10/10 while overall < 9).
