# ref: SigmaShake Desktop Debug — load when any `sigmashake-desktop-{linux,mac,windows}` ∈ AFFECTED_SUBPROJECTS

There are three native shells, one for each OS. Each shell supervises `ssg daemon` and `ssg serve`, and hosts the dashboard in a native WebView:

| OS | Subproject | Shell stack | Build |
|---|---|---|---|
| Linux | `sigmashake-desktop-linux` | Go + Wails v2 (WebKitGTK) | `scripts/build-publish-linux.sh` / `wails build` — **PSU-gated** serial cgo/WebKitGTK link, `-tags webkit2_41`. `internal/*` builds and tests without the Wails runtime: `go build ./internal/...`, `go test ./internal/...`. |
| macOS | `sigmashake-desktop-mac` | Swift / AppKit (`WKWebView`) | host-only — cannot build on this Linux box (honest-red). |
| Windows | `sigmashake-desktop-windows` | C# / .NET 8 WinForms + WebView2 | host-only — cannot build on this Linux box (honest-red). |
| (any) | release | `bun run release:desktop:{linux,mac,win}` → `sigmashake-dist/scripts/release.sh` | per-OS build + R2 channel + version train. |

All three shells reuse the **web UI** verbatim. `ssg serve` serves the dashboard SPA and the setup/wizard. The shells do not bundle them again. The workspace UI is the plugin at `/plugins/workspace/`. It is not bundled. So most "desktop" runtime bugs are really `ssg serve`, daemon, or SPA bugs. **First, reproduce the bug against `ssg serve` in a browser.**

**Phase 0 baseline (any shell):**
```bash
ssg probe desktop --json   # ssg path/version, daemon adopted, dashboard_port, updater, wizard state
```
If `ssg_path` is null or the command fails, that is the bug. Do not read more files. Record `ssg_path`, `dashboard_port`, `adopted`, and `wizard_complete`.

**Logs.** Every shell writes a daily rolling log, `~/.sigmashake/desktop.YYYY-MM-DD.log`. Tail it BEFORE you open the source. Record only the stack frame line. Do not keep the dump:
```bash
tail -n 200 ~/.sigmashake/desktop.$(date +%F).log
```

**Phase 4 triage — check in order. Stop at the first match:**
1. **ssg not found / version mismatch** — in the probe, `ssg_path` is null or the version is wrong. Fix: PATH or install. Do not read source.
2. **Dashboard does not load** — `dashboard_port` is not in 5599–5610, or `adopted` is false when it must be true. Do not read source.
3. **Runtime error in app** — tail the log first. Open only the source file that the log names.
4. **Wizard state broken** — `wizard_complete` or `wizard_profile` is wrong. Read only the wizard-state source of the shell.
5. **Native shell build/link error** — read **only the file that the error names**. A Linux WebKitGTK/cgo link failure means that `-tags webkit2_41` or the system headers are missing. It is not an app logic bug. Mac and Windows cannot build here. Report honest-red. Do not search the code.
6. **Webview/HTML failure** — `ssg serve` serves the SPA. Reproduce the failure in a browser against standalone `ssg serve`. This shows if it is an SPA bug (gov `src/client/`) or a shell bug.

**Debug budget:** make a maximum of 3 probe, snapshot, or exploratory-Read calls before you form a hypothesis. This applies to `ssg debug snapshot`, `ssg probe`, and source Reads.
