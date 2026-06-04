# QuickLook fork — work tracker

Forward-state tracker for the two work-streams in this fork. Companion to
`CLAUDE.md` (the how) — this file is the what-next. Detail lives in the study docs:
`docs/security/README.md` and `docs/preview-pane/feasibility.md`.

Glyphs: `[ ]` not started · `[~]` in flight · `[x]` done · `[-]` deferred.

## Status snapshot — 2026-06-04

- Scaffolding landed and pushed to `lakeworks/QuickLook` `master` (`ca5b9d7..1d6ae55`).
- No code changes yet; both work-streams are at the "start the real work" line.
- Nothing in flight.

## Work-stream 1 — Security audit (defensive)

Goal: find real, reproduced defects; ship each as **one issue → one fix → one
commit** on a `security/<issue>` branch → atomic PR. Confirmed findings go to the
`docs/security/README.md` ledger. Responsible-disclosure rule (CLAUDE.md) is binding.

Per-surface audit passes (read input→sink, build adversarial input, confirm):

- [ ] **S1** Untrusted file parsers (`QuickLook.Plugin.*` viewers) — highest blast radius
- [ ] **S2** Plugin trust & install (`PluginManager.cs` `Assembly.LoadFrom`; `.qlplugin` ZIP zip-slip / namespace validation)
- [ ] **S3** Native shell hooks + hotkey (`QuickLook.Native*`, `GlobalKeyboardHook.cs`)
- [ ] **S4** IPC named pipe (`PipeServerManager.cs` — ACL/SID scope, command parse)
- [ ] **S5** WebView2 content (`HtmlViewer`, `ImageViewer/Webview/*`, Markdown, Chm)
- [ ] **S6** Auto-update (`Helpers/Updater.cs` — TLS, redirects, pinning)
- [ ] **S7** Mark-of-the-Web (`UnblockZoneIdentifier`, zone handling)
- [ ] **S8** Path handling (symlink/junction, UNC, long paths)

Confirmed findings: **0** (see ledger). Recommended start: **S1**.

## Work-stream 2 — Explorer Preview Pane wrapper

Goal: generalize the in-tree OS-preview-handler host (`OfficeViewer`'s
`PreviewHandlerHost` + `ShellExRegister`, already format-agnostic) into a
low-priority fallback viewer so QuickLook reuses the same previewers the Explorer
Preview Pane shows. Verdict: highly feasible (prior art in-tree). Detail +
open questions in `docs/preview-pane/feasibility.md`.

- [ ] Create `QuickLook.Plugin.PreviewHandlerViewer` reusing `PreviewHandlerHost` + `ShellExRegister`
- [ ] `CanHandle(path)` = an `IPreviewHandler` CLSID is registered for the extension; assign a **low `Priority`** so format-specific plugins always win
- [ ] Prefer out-of-process (`prevhost.exe`) hosting; bound + guard the in-proc fallback (in-proc handler = untrusted native code in-process)
- [ ] WPF host integration: map host element to the WinForms `Control` HWND; resize / DPI / dark-mode; deterministic dispose + factory/surrogate lifetime
- [ ] Focus & keyboard interplay with QuickLook navigation (arrows, Esc)
- [ ] ARM64: handler bitness vs. QuickLook process (surrogate vs. in-proc)
- [ ] Test matrix: long-tail formats with registered handlers (`.msg`, `.vsdx`, CAD, vendor) and graceful skip when none registered

## Done

- [x] `CLAUDE.md` project guide — two work-streams, build, attack surface, hygiene, disclosure rule (`ca5b9d7`)
- [x] `docs/preview-pane/feasibility.md` — goal-2 research + prior-art study (`461b80f`)
- [x] `docs/security/README.md` — attack-surface inventory + findings ledger scaffold (`1d6ae55`)
