# ref: Search Console / SEO Conformance — load when a sigmashake.com SEO / structured-data / canonical / robots / sitemap surface ∈ AFFECTED_SUBPROJECTS

Some changes affect a sigmashake.com `<head>`, canonical, `robots` meta, `sitemap.xml`, or `application/ld+json` surface. This applies wherever the surface is emitted: `sigmashake-web`, the bench blocks, the trust, security, playground, hub, and accounts surfaces, or the workspace catalog. **Only the standing SEOGUARD team does this work. Never use a general-purpose agent.** Every applicable Google Search Console issue maps to a **named owning agent**. The orchestrator dispatches the team. The orchestrator does not advance while any applicable issue has no owner.

**Team — root `.claude/agents/` (never general-purpose):**
- `seoguard-lead`.
- The **web engineer** (`sigmashake-web` `<head>`/canonical/robots/sitemap/JSON-LD).
- The **workspace engineer** (the `seo-compliance-catalog.ts` source of truth and its integrity test).
- The **docs engineer** (the ledger and the skill pointer).
- `seoguard-honesty-sentinel`.
- `seoguard-verification-analyst` (read-only outside-in CONFIRM against live pages).

Send every public-facing string through the honesty-sentinel.

**Source-of-truth ledgers:**
- Catalog — `sigmashake-workspace/src/seo-compliance-catalog.ts`. It lists every GSC issue with its applicability and mapped check. It also holds the map from each issue to its **owning agent**.
- Ledger — `sigmashake-workspace/docs/seo/search-console-conformance-ledger.md`. It lists every GSC issue (issue → requirement → applicability → mapped check → status). It also has the **`## Owning-agent map`** section, which matches the catalog.

**Coverage-completeness gate.** The catalog integrity test asserts that every *applicable* GSC issue maps to a **named owning agent**. An honest `n/a — <reason>` is allowed. An **empty owner is a fail**. If an applicable issue has no owner, the result is **`BLOCKED`** (Anti-Loop). Never report a fake-green "covered".

**Accepted decisions (the four standing SEOGUARD calls).**
1. **Omit ratings.** Never fabricate `aggregateRating` or `review`. Keep them absent until a real, verifiable source exists.
2. **`SoftwareApplication` + a single `Offer`** for the SaaS (not `Product`). The one `Offer` carries a valid `price`/`priceSpecification.price`.
3. **Hard gate** = the sigmashake-web seo-conformance test (`bun --cwd sigmashake-web test test/seo-conformance.test.ts`) **and** the workspace catalog-integrity test. Both must pass at **90/90/90/80**.
4. **Host-aware canonical.** Each indexable page has exactly one self-referential `<link rel="canonical">`. Alias hosts self-canonicalize. Never set `noindex` on marketing pages. Keep `noindex` pages out of `sitemap.xml`.

**Phase 6 trigger:** the SEOGUARD team runs the catalog-integrity check. Then it runs the coverage-completeness gate above. The hard gate is the two tests at 90/90/90/80. If an applicable issue has no owner, the result is `BLOCKED`.

**Honesty (no-overclaim).** Do not fabricate `aggregateRating` or `review`. Do not write "Google-approved", "fully compliant", or "guaranteed indexed" in any agent, ledger, or page copy. Do not list a legal entity on any page. Never claim incorporation. Measure verdicts against the repo at HEAD. Legs that need live Search Console or a crawl stay **honest-red**. Never fake-green them.
