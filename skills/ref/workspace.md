# ref: Workspace UI Integration — load when `sigmashake-workspace` ∈ AFFECTED_SUBPROJECTS

One React/Vite codebase ships in three ways:
- **cloud** Worker (`workspace.sigmashake.com`, browse-only).
- **SSG plugin** (`/plugins/workspace/`). This is the *canonical local surface*. Standalone `ssg serve` and SigmaShake Desktop both render it.
- **VS Code extension** (`sigmashake.workspace`).

The bundled-into-desktop view was removed on 2026-05-26. The plugin is the only local path.

- **Build/test gates:** `bun run workspace:typecheck` (Worker and client tsconfigs), `bun run workspace:test` (Vitest `cloudflareTest` pool). The canonical local build is `bun --cwd sigmashake-workspace run build:plugin` → `dist-plugin/`. Do NOT use `build:client`. That command builds the cloud Worker assets.
- **Generated files — never hand-edit:** `src/registry.generated.ts`, `src/infrastructure.generated.ts`, and `src/readiness.generated.ts`. `prebuild` and `predeploy` regenerate them from the root `STRUCTURE.md` `<structure>` block, the wrangler configs, and `.sigmashake/production-readiness.json`. Edit the source, not the generated file.
- **A daemon-route change is also a gov change.** The local, plugin, and vscode clients call `/api/json/workspace/*`, which is registered in `sigmashake-gov/src/commands/serve.ts`. Add `sigmashake-gov` to AFFECTED_SUBPROJECTS and **port the route into `go/`**. The shipped daemon is Go. A TS-only route does not reach users.
- **Platform-parity catalog.** The cross-OS parity matrix renders here. A feature that ships on all three shells is not complete until its platform-parity catalog entry exists and is *honest*. `MANUAL`/`unknown` is allowed when no live health probe exists. Do not fake-green it.
