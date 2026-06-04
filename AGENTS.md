# QuickLook Agent Guide

## Overview

QuickLook is a GPL-3.0 Windows file-preview app that brings macOS-style Space-key previews to File Explorer and related shells. The main app is a C#/.NET Framework 4.6.2 WPF application, shared contracts live in `QuickLook.Common/`, shell selection hooks are native C++ DLLs under `QuickLook.Native/`, and file-format support is split across the `QuickLook.Plugin/QuickLook.Plugin.*` projects. The solution is `QuickLook.slnx` and builds with MSBuild on Windows.

## Project Layout

- `QuickLook.slnx`: solution containing the main app, common library, native hook projects, plugin projects, and WiX installer project.
- `QuickLook/`: main WPF app, tray UI, preview window, plugin host, IPC, global hotkey, updater, and native bridge.
- `QuickLook.Common/`: shared helper code and plugin contracts, including `QuickLook.Common/Plugin/IViewer.cs`.
- `QuickLook.Native/QuickLook.Native32/`, `QuickLook.Native/QuickLook.Native64/`, `QuickLook.Native/QuickLook.NativeArm64/`: MSVC v143 native shell hook DLL projects.
- `QuickLook.Plugin/QuickLook.Plugin.*`: viewer/plugin projects for app packages, archives, binary files, certificates, CSV, databases, dumps, ELF, fonts, HTML, images, mail, Markdown, media info, Office, PDF, PE, text, thumbnails, video, and plugin installation.
- `QuickLook.Installer/`: WiX MSI packaging project.
- `QuickLook.Appx/`: Windows app package project and manifest.
- `Scripts/`: version, packaging, signing, and artifact helper scripts.
- `Build/`: build outputs and release assets.
- `docs/`: project notes, including security and preview-pane research.

## Build And Run

Prerequisites: Visual Studio 2022 Build Tools or IDE with .NET desktop build support, MSVC v143 C++ tools, NuGet, and a Windows SDK. Building `QuickLook.Installer/QuickLook.Installer.wixproj` also requires WiX Toolset v3.14 or newer.

Restore packages:

```powershell
nuget restore QuickLook.slnx
```

CI-equivalent release build:

```powershell
msbuild /m /p:BuildInParallel=true /p:Configuration=Release QuickLook.slnx
```

Architecture-specific local release build:

```powershell
msbuild QuickLook.slnx /p:Configuration=Release /p:Platform=x64 /m
```

Debug build:

```powershell
msbuild QuickLook.slnx /p:Configuration=Debug /p:Platform=x64 /m
```

Run the built app:

```powershell
.\Build\Release\QuickLook.exe
```

For a debug build, run:

```powershell
.\Build\Debug\QuickLook.exe
```

## Tests And Lint

No test projects, test framework references, or dedicated CLI lint targets are defined in the current solution. Use `nuget restore QuickLook.slnx` plus an MSBuild build as the baseline validation. Formatting/style configuration exists in `CodeMaid.config` and `Settings.XamlStyler`, but those are IDE/tool configuration files rather than repo-provided lint commands.

## Key Entry Points

- `QuickLook/App.xaml.cs`: application startup, single-instance handling, native initialization, and service startup.
- `QuickLook/PluginManager.cs`: plugin discovery and loading.
- `QuickLook/ViewWindowManager.cs`: preview selection flow and viewer window orchestration.
- `QuickLook/ViewerWindow.xaml` and `QuickLook/ViewerWindow.xaml.cs`: main preview window UI.
- `QuickLook/PipeServerManager.cs`: named-pipe IPC between instances and preview commands.
- `QuickLook/KeystrokeDispatcher.cs` and `QuickLook/Helpers/GlobalKeyboardHook.cs`: Space/Esc and navigation hotkey handling.
- `QuickLook/NativeMethods/QuickLook.cs`: managed bridge into the native hook DLLs.
- `QuickLook/Helpers/Updater.cs`: update checks and changelog preview.
- `QuickLook.Common/Plugin/IViewer.cs`: plugin interface implemented by viewer projects.
- `QuickLook.Plugin/QuickLook.Plugin.*/*Plugin*.cs` and `QuickLook.Plugin/QuickLook.Plugin.*/Plugin.cs`: plugin implementations.
- `QuickLook.Plugin/QuickLook.Plugin.OfficeViewer/PreviewHandlerHost.cs`: existing Windows preview-handler hosting code.

## Code Conventions

- Managed projects target `net462`; most use SDK-style project files, `LangVersion` `latest`, WPF support, and output to `Build/<Configuration>/`.
- Current C# code uses file-scoped namespaces, explicit public API members where needed by interfaces, `using var`, collection expressions, and existing `QuickLook.Common` helpers/contracts.
- Plugin implementations should implement `IViewer` members: `Priority`, `Init`, `CanHandle`, `Prepare`, `View`, and `Cleanup`. `CanHandle` should check file headers when applicable, `Prepare` should stay lightweight, `View` should clear `context.IsBusy` when loading is complete, and `Cleanup` should release unmanaged or long-lived resources.
- Native projects are Unicode dynamic libraries using MSVC v143, warning level 3, SDL checks, and architecture-specific preprocessor definitions.
- Preserve existing GPL headers in first-party source files and follow local formatting in nearby files. Use `CodeMaid.config` for C# cleanup expectations and `Settings.XamlStyler` for XAML formatting.
- This is a public repository. Do not add credentials, API keys, tokens, private keys, personal emails, or machine-specific absolute paths to tracked files.
- Do not commit or push changes unless explicitly asked.
