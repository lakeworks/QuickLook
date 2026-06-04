# QuickLook — project guide

QuickLook brings macOS-style "press Space to preview" to Windows. C#/.NET (WPF) app + a native C++ shell hook + ~23 viewer plugins. GPL-3.0.

- **This repo is a public fork.** `origin` = `lakeworks/QuickLook` (public), `upstream` = `QL-Win/QuickLook`. Treat everything committed here as world-readable. See **Public-repo hygiene** below before every commit/push.
- **Platform**: Windows 10/11 + Windows Server 2025. Builds x64 / ARM64 / x86. Main managed code targets **.NET Framework 4.6.2** (`net462`); native code is **C++ (MSVC v143)**.

## Mission — two work-streams

1. **Security study (defensive).** Audit the application's attack surface in depth and propose fixes. **One issue = one fix = one commit** so each can become a standalone atomic PR. Track findings in `docs/security/README.md`. Responsible-disclosure rule below is binding.
2. **Explorer Preview Pane wrapper (feasibility → build).** Study whether QuickLook can render previews by hosting the OS-integrated Windows Explorer preview handlers (the same `IPreviewHandler` COM objects the Explorer Preview Pane uses), as a generic fallback viewer. Prior art already exists in-tree — see **Goal 2** and `docs/preview-pane/feasibility.md`.

## Repository map

| Project | Lang / target | Role |
|---|---|---|
| `QuickLook/` | C# `net462`, WPF+WinForms, WinExe | Main app: tray process, preview window, plugin host, IPC, hotkey, updater |
| `QuickLook.Common/` | C# `net462` | Shared `IViewer` plugin contract, helpers, native-method decls |
| `QuickLook.Native/Native32\|Native64\|NativeArm64/` | C++ v143 (`.vcxproj`) | Shell hooks: detect focused Explorer/file-manager + selected file path; `WoW64HookHelper` for 32-on-64 |
| `QuickLook.Plugin/QuickLook.Plugin.*/` | C# `net462` | ~23 viewers (Image, Archive, PDF, Html, Markdown, Font, Text, Office, ELF/PE, Db, Mail, Chm, Video, Binary, Cert, …) |
| `QuickLook.Installer/` | WiX (`.wixproj`) | MSI installer |
| `QuickLook.Appx/` | WAP | Microsoft Store / Appx package |

**Key files** (cite by `path:line`):
- Plugin loader: `QuickLook/PluginManager.cs` — `Assembly.LoadFrom` of `QuickLook.Plugin.*.dll` from `%APPDATA%\QuickLook.Plugin\` + app dir; in-process; no signature check; `IViewer` + integer `Priority`.
- IPC: `QuickLook/PipeServerManager.cs` — named pipe `QuickLook.App.Pipe.<user-SID>`, plaintext `|`-delimited commands (`Invoke|<path>|<opts>`, `Switch`, `Toggle`, `Quit`, …).
- Native bridge: `QuickLook/NativeMethods/QuickLook.cs` → exports in `QuickLook.Native*/Shell32.cpp`, `DialogHook.cpp`.
- Global hotkey: `QuickLook/Helpers/GlobalKeyboardHook.cs` (`WH_KEYBOARD_LL`).
- Updater: `QuickLook/Helpers/Updater.cs` — GitHub releases API, manual download (no silent auto-install).
- OS preview-handler host (goal-2 prior art): `QuickLook.Plugin/QuickLook.Plugin.OfficeViewer/PreviewHandlerHost.cs` + `ShellExRegister.cs`.

## Build & run

Requires Visual Studio 2022 (MSVC v143 + .NET desktop + WiX). The solution is the new XML format `QuickLook.slnx` (needs recent MSBuild/VS).

```powershell
# Full solution (managed + native), Release x64
msbuild QuickLook.slnx /p:Configuration=Release /p:Platform=x64 /m
# Just the main app + its native deps during iteration
msbuild QuickLook/QuickLook.csproj /p:Configuration=Debug /p:Platform=x64
```

Build output lands in `Build/<Configuration>/`. Run `Build/<Configuration>/QuickLook.exe`; it lives in the tray. Select a file in Explorer and press **Space**. CI is AppVeyor (`.appveyor.yml`).

## Goal 1 — Security study

This is a previewer: it parses **attacker-controlled file bytes** the moment a user presses Space, and it injects native code into Explorer. Treat it as a high-blast-radius, untrusted-input application.

**Attack surface (audit targets):**
1. **Untrusted file parsers** — every viewer decodes hostile input in-process: image codecs, archive extraction, PDF, fonts, ELF/PE, SQLite/DB, mail, CHM, media. Native decoder deps are the memory-safety risk; .NET parsers risk DoS / path traversal / XXE.
2. **Plugin trust** — `Assembly.LoadFrom` with no strong-name/signature verification; `.qlplugin` is a ZIP extracted via `ZipFile.ExtractToDirectory` (verify zip-slip + namespace validation). Installed plugins run with full app privileges.
3. **Native shell hooks + `WH_KEYBOARD_LL`** — code mapped into Explorer / third-party file managers; large native surface; the managed↔native boundary.
4. **IPC named pipe** — pipe ACL/SID scoping, plaintext command parsing, path handling, injection.
5. **WebView2 content** — HTML/SVG/Lottie/Markdown/CHM rendering of untrusted content: script execution, local-file (`file://` / virtual-host mapping) reach, remote resource loads (SSRF/exfil), navigation control.
6. **Auto-update** — TLS trust to GitHub API, redirect handling, no SSL pinning.
7. **Mark-of-the-Web** — previewing downloaded files; the `UnblockZoneIdentifier` dependency and any MotW stripping.
8. **Path handling** — symlink/junction following, UNC paths, long paths, zone evaluation.

**Method:** for each surface, read the code path end-to-end from untrusted input to sink; write adversarial inputs (boundary, malformed, oversized, traversal); confirm the defect before claiming it. Don't enumerate hypothetical bugs — verify.

**Commit discipline (binding):**
- One finding → one minimal fix → one commit. Subject names the issue, not the file (`fix: reject zip entries that escape the plugin dir`, not `update PluginInfoPanel.cs`).
- Land a regression test in the **same commit** when the defect is hermetically reproducible; if it isn't, say why in the body.
- Record every finding in `docs/security/README.md` (one row: surface · severity · status · fix commit).

**Responsible disclosure (binding — public repo):**
- A confirmed, exploitable, currently-**unpatched-upstream** vulnerability must NOT be published here (public commit, public PR, or public findings doc) before coordinating private disclosure with upstream `QL-Win`. Public commit of a working exploit path endangers all users.
- Until coordinated, keep the public ledger to: hardening items, already-public/low-severity issues, and defensive fixes whose commit message does not hand a reader a weaponized exploit. When in doubt, hold the detail and ask.

## Goal 2 — Explorer Preview Pane wrapper

**Verdict up front: highly feasible — the wrapper already exists in this repo, scoped to Office.** `QuickLook.Plugin.OfficeViewer` hosts OS-registered preview handlers and is already format-agnostic at its core:
- `ShellExRegister.GetPreviewHandlerGUID(ext)` resolves the handler CLSID from `HKCR\.<ext>\shellex\{8895b1c6-b41f-4c1c-a562-0d564250836f}` (or via the ProgID/class key) — **works for any extension** with a registered handler, not only Office.
- `PreviewHandlerHost.Open(path)` does the full host dance: out-of-process via `prevhost.exe` (DllSurrogate, `CLSCTX_LOCAL_SERVER`) with in-process fallback, init priority `IInitializeWithStream → IInitializeWithItem → IInitializeWithFile`, then `SetWindow` + `DoPreview`. Mirrors PowerToys Peek.

**The study/build task** is to generalize this into a low-priority fallback plugin that previews *any* file whose extension has a registered preview handler, so QuickLook transparently reuses the same previewers the Explorer Preview Pane shows. Design notes, security considerations (handlers run out-of-proc at low IL — a plus), plugin-priority ordering vs. format-specific plugins, and open questions live in `docs/preview-pane/feasibility.md`.

Out-of-process preview-handler hosting is the safer mode (process isolation); prefer it and keep the in-proc fallback bounded.

## Public-repo hygiene

This fork is public. **Never commit credentials or local-environment context.**
- No API keys, tokens, private keys, passwords, signing material, personal emails, internal hostnames/paths, or machine-specific configuration.
- Do not paste local development-environment paths or host tooling into tracked files; keep this CLAUDE.md and `docs/` about the QuickLook project only.
- `Build/sideload.{pfx,key,crt}` is a throwaway self-signed Appx **sideload** cert inherited from upstream (intentionally public, not an account credential) — leave it; do not treat it as a secret to manage and do not add real signing material.
- Run a pre-push grep for emails/keys/tokens against the diff before pushing. A hit is a hard stop.
- Author identity is the GitHub noreply alias (`…+lakeworks@users.noreply.github.com`); keep it — a real email in a public commit is a credential-class leak.

## Working conventions

- **Commit on a regular basis**, at every coherent unit of work — small, atomic, one concern per commit. Don't accumulate large uncommitted deltas.
- Do `security/` and `preview-pane/` work on feature branches (`security/<issue>`, `feat/preview-pane-host`) so each becomes a clean PR; keep `master` tracking upstream-compatible history.
- **Do not push** to `origin` without explicit confirmation, and never to `upstream`. Pushing is the irreversible, public step — gate it.
- Rebasing onto / pulling from `upstream/master` is the way to stay current; keep fork-specific changes isolated and rebasable.
