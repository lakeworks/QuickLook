---
status: complete
commits:
  - 95c321e  "fix: disable WebView2 GPU sandbox on GPU-less hosts (prevents blank previews)"
  - dc19b94  "docs: mark WebView2 GPU-sandbox auto-fix as implemented"
builds: clean
code_sourced_flag_verified: yes
---

# WebView2 GPU-less fork fix — implementation + verification (2026-06-04)

## What was implemented

Minimal, GPU-gated fork fix in
`QuickLook.Plugin/QuickLook.Plugin.HtmlViewer/WebpagePanel.cs`:

- Added `private static bool IsGpuUnavailable()` returning
  `(System.Windows.Media.RenderCapability.Tier >> 16) == 0` — the proven Tier-0
  (software-only WPF compositing) signal for GPU-less hosts.
- In `InitializeComponent()`, when `IsGpuUnavailable()` is true, set
  `CoreWebView2CreationProperties.AdditionalBrowserArguments = "--disable-gpu-sandbox"`.
  When a hardware GPU is present (Tier >= 1) nothing is set — the full GPU sandbox
  is retained (the security requirement).
- `_webView` behaviour is otherwise identical. Both HtmlViewer and MarkdownViewer
  inherit `WebpagePanel`, so the single change covers both viewers.

The obsolete WinForms-rehost approach (branch
`worktree-agent-a20141844c17fe140`) was NOT reused — this worktree started clean
from master.

## Commits (atomic, one concern each)

| SHA | Subject |
|---|---|
| 95c321e | fix: disable WebView2 GPU sandbox on GPU-less hosts (prevents blank previews) |
| dc19b94 | docs: mark WebView2 GPU-sandbox auto-fix as implemented |

Diff grep for credentials / machine-specific paths: clean.

## Build

- `QuickLook.Common` Release|x64: built clean →
  `Build/Release/QuickLook.Common.dll`.
- `QuickLook.Plugin.HtmlViewer` Release|x64 (AnyCPU): built clean →
  `QuickLook.Plugin/QuickLook.Plugin.HtmlViewer/bin/x64/Release/QuickLook.Plugin.HtmlViewer.dll`
  (20992 bytes, SHA256 `B6611EA7C053BCC4263F913271A2563A380CF2F71CBAB87715891DD6519B85CC`).
- MSBuild 17.14.40 (VS 2022 BuildTools). No warnings or errors on the touched
  project. No standalone `.exe` was built or run (ASR policy on this host).

## Runtime verification — code sources the flag, not the registry

Goal: prove the **rebuilt DLL's code** emits `--disable-gpu-sandbox`, independent
of the deployed registry policy.

1. **Host precondition confirmed:** `RenderCapability.Tier >> 16 == 0` measured on
   this host → `IsGpuUnavailable()` returns true, so the gate fires.
2. **Backups taken** to `./tmp/ql-dll-backup2/`:
   - installed `QuickLook.Plugin.HtmlViewer.dll`
     (orig SHA256 `290B9F8F...`, 23552 bytes) →
     `QuickLook.Plugin.HtmlViewer.dll.orig`.
   - registry value data (`--disable-gpu-sandbox`) →
     `registry-QuickLook.exe-value.txt`.
3. **Registry value REMOVED** (`HKLM\SOFTWARE\Policies\Microsoft\Edge\WebView2\
   AdditionalBrowserArguments\QuickLook.exe`) — confirmed absent.
4. **Rebuilt DLL installed** over the deployed one
   (installed hash now `B6611EA7...`, matching the rebuilt DLL).
5. **QuickLook restarted** from a shell with **no**
   `WEBVIEW2_ADDITIONAL_BROWSER_ARGUMENTS` env var; preview self-triggered via a
   second instance forwarding `./QuickLook/CLAUDE.md` over the named pipe.
6. **Result — flag is present and sourced from code:**
   - `msedgewebview2.exe` **browser** process (pid 12328): user-data-dir =
     `...\pooi.moe\QuickLook\WebView2_Data\EBWebView`, command line carries
     `--disable-gpu-sandbox`.
   - `msedgewebview2.exe` **gpu-process** (pid 12316) running with
     `--disable-gpu-sandbox` — i.e. the GPU process initialised and did **not**
     crash-loop.
   - Registry value confirmed still absent and env var confirmed unset at the
     moment of capture → the flag can only have come from the DLL's code path.
   - `%APPDATA%\pooi.moe\QuickLook\QuickLook.Exception.log` was **not created** —
     no new `E_ABORT`, no WebView2 init failure.
7. **Registry value RESTORED** afterward
   (`QuickLook.exe = --disable-gpu-sandbox`) so the host stays protected
   regardless of which DLL is installed. The **rebuilt DLL was left installed**
   (it is the real fix).

No Defender / ASR / firewall / other system-config controls were touched — only
QuickLook's own app-arg registry policy was toggled and restored.

## Verdict

- `code_sourced_flag_verified: yes` — the rebuilt DLL emits
  `--disable-gpu-sandbox` from code, with the registry policy removed and no env
  var set, and the WebView2 GPU process runs instead of crash-looping.
- Both commits clean; both builds clean.
