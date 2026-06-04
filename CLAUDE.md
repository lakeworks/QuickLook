# QuickLook — project guide

macOS-style "press Space to preview" for Windows. C#/.NET WPF app + native C++ shell hook + ~23 viewer plugins. GPL-3.0.

- **Public fork.** `origin` = `lakeworks/QuickLook`, `upstream` = `QL-Win/QuickLook`. Everything committed here is world-readable.
- Main managed code: .NET Framework 4.6.2 (`net462`). Native: C++ (MSVC v143). Builds x64 / ARM64 / x86.

## Goals

1. **Security audit** (defensive) — find real defects, propose fixes. One issue = one fix = one commit. Detail + ledger: `docs/security/README.md`.
2. **Explorer Preview Pane wrapper** — host the OS `IPreviewHandler` previewers as a fallback viewer. Detail: `docs/preview-pane/feasibility.md`.

## Repo map

| Project | Role |
|---|---|
| `QuickLook/` | Main app: tray, preview window, plugin host, IPC, hotkey, updater |
| `QuickLook.Common/` | `IViewer` plugin contract + shared helpers |
| `QuickLook.Native/Native32\|64\|Arm64/` | C++ shell hooks (detect focused window + selected file) |
| `QuickLook.Plugin/QuickLook.Plugin.*/` | ~23 viewers |
| `QuickLook.Installer/` `QuickLook.Appx/` | WiX MSI / Store package |

Entry points: `QuickLook/PluginManager.cs` (plugin load), `PipeServerManager.cs` (named-pipe IPC), `NativeMethods/QuickLook.cs` (native bridge), `Helpers/GlobalKeyboardHook.cs` (hotkey), `Helpers/Updater.cs` (update check), `QuickLook.Plugin.OfficeViewer/PreviewHandlerHost.cs` (goal-2 prior art).

## Build

VS 2022 (MSVC v143 + .NET desktop + WiX); solution is `QuickLook.slnx`.

```powershell
msbuild QuickLook.slnx /p:Configuration=Release /p:Platform=x64 /m
```

Output in `Build/<Config>/`; run `QuickLook.exe`, select a file, press Space. CI: `.appveyor.yml`.

## Rules

- **Commit discipline**: one concern per commit, subject names the issue. Regression test in the same commit when reproducible. Commit regularly; don't accumulate large deltas.
- **Disclosure**: do not publicly commit a working exploit for an upstream-unpatched bug before coordinating with `QL-Win`.
- **No credentials** in tracked files: no keys, tokens, private keys, personal emails, or machine-specific paths. Grep the diff before pushing. (`Build/sideload.*` is a throwaway upstream sideload cert — leave it.)
- Author identity is the `lakeworks` GitHub noreply alias — keep it.
- Work on feature branches; **don't push without explicit confirmation**, never to `upstream`.
