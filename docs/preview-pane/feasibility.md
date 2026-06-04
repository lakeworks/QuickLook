# Explorer Preview Pane wrapper — feasibility & prior art

Status: research / scoping. Goal: determine whether QuickLook can leverage the
previewers integrated into the Windows 11 / Server 2025 Explorer **Preview Pane**.

## Disambiguation — two opposite "integrations"

"Integrate QuickLook with the Windows Explorer Preview" can mean either direction.
They are very different in difficulty:

- **Direction A — QuickLook *consumes* the OS preview handlers.** QuickLook hosts
  the same `IPreviewHandler` COM objects that Explorer's Preview Pane uses, and
  renders them inside QuickLook's own Space-to-preview window. This is the
  "wrapper" goal. **Highly feasible — prior art exists in this very repo.**
- **Direction B — QuickLook *provides* previews to Explorer's Preview Pane.**
  QuickLook registers itself as an `IPreviewHandler` so that selecting a file and
  opening the native Explorer Preview Pane shows QuickLook plugin output.
  **Much harder** — see below.

## Direction A — host the OS preview handlers (the wrapper)

### Prior art already in this codebase

`QuickLook.Plugin/QuickLook.Plugin.OfficeViewer/` already implements a
PowerToys-Peek-style host for OS-registered preview handlers, and its core is
**format-agnostic**:

- `ShellExRegister.cs` → `GetPreviewHandlerGUID(ext)` resolves the handler CLSID
  from `HKCR\.<ext>\shellex\{8895b1c6-b41f-4c1c-a562-0d564250836f}` (the standard
  preview-handler shellex key), with a ProgID/class-key fallback. Works for **any**
  extension that has a registered handler, not only Office.
- `PreviewHandlerHost.cs` → `Open(path)`:
  - Creates the COM server **out-of-process** first via `CoGetClassObject` with
    `CLSCTX_LOCAL_SERVER` (→ `prevhost.exe` surrogate when the CLSID's AppID has a
    DllSurrogate), caching the `IClassFactory` so the host process stays warm;
    **in-process fallback** via `Activator.CreateInstance(Type.GetTypeFromCLSID(...))`.
  - Initializes via `IInitializeWithStream → IInitializeWithItem → IInitializeWithFile`
    in priority order.
  - `SetWindow(hwnd)` + `SetRect` + `DoPreview()` into a WinForms `Control` host;
    `Unload()` / `FinalReleaseComObject` on teardown.

There is also a sibling external repo `QL-Win/QuickLook.Plugin.OfficeViewer-Native`
with the same `PreviewHandlerHost` / `IPreviewHandler` pattern.

So the wrapper is **already written and shipping** — it is just scoped to Office
extensions by the plugin's `CanHandle`.

### External reference implementation

Microsoft **PowerToys "Peek"** is the canonical open-source (MIT) example of a
Space-to-preview overlay that hosts OS preview handlers, including its own
`PreviewHandler` host and shell-preview-handler resolution. The QuickLook
OfficeViewer host explicitly mirrors it. Peek is the best reference for edge cases
(handler crashes, focus/keyboard, threading, DPI, surrogate lifetime).

### What the build task actually is

Generalize the existing host into a **low-priority fallback viewer plugin**:
1. New plugin (e.g. `QuickLook.Plugin.PreviewHandlerViewer`) reusing
   `PreviewHandlerHost` + `ShellExRegister`.
2. `CanHandle(path)` = "an `IPreviewHandler` CLSID is registered for this
   extension" (registry probe) — and return a **low `Priority`** so every
   format-specific QuickLook plugin still wins; this only catches the long tail
   (`.msg`, `.vsdx`, CAD, vendor formats, etc.) that QuickLook has no native
   viewer for.
3. Prefer the out-of-process (`prevhost.exe`) path; keep in-proc fallback bounded
   and guarded — an in-proc handler is untrusted native code running with full app
   privileges (see security notes).
4. Map a WPF host element to the WinForms `Control` HWND (WindowsFormsHost or an
   HWND-hosting element), handle resize/DPI/dark-mode, and dispose deterministically.

### Open questions / risks (Direction A)
- **Handler availability varies by machine** — relying on whatever is registered
  means inconsistent coverage; document it as best-effort fallback, not a feature
  promise.
- **In-proc fallback = trust escalation.** A buggy/malicious in-proc handler runs
  in QuickLook's process. Decide whether to *only* allow out-of-proc handlers and
  skip formats that have no surrogate.
- **Lifetime / leaks** of cached class factories and the surrogate host process.
- **Focus & keyboard** interplay with QuickLook's navigation (arrow keys, Esc).
- **ARM64**: out-of-proc handler bitness vs. QuickLook process; surrogate handles
  most of this, in-proc does not.
- **Licensing**: borrowing patterns from PowerToys (MIT) into a GPL-3.0 codebase is
  fine directionally; do not copy code verbatim without honoring attribution.

## Direction B — register QuickLook *into* the Explorer Preview Pane

To make the **native** Explorer Preview Pane render QuickLook content, QuickLook
would have to ship an `IPreviewHandler` shell extension registered under
`HKCR\.<ext>\shellex\{8895b1c6-…}`. Constraints make this a much larger effort:

- Preview handlers run **out-of-process in a low-integrity surrogate**
  (`prevhost.exe`) by design. QuickLook's WPF plugins are not built to run inside
  that sandbox.
- It would mean re-architecting plugins as COM preview handlers (init via
  `IInitializeWithStream`, render into an Explorer-owned HWND, obey
  `IPreviewHandlerVisuals` / `IPreviewHandlerFrame`).
- This duplicates what dedicated per-format preview handlers already do and gives
  up QuickLook's main value (full-window, interactive overlay).

No evidence of any QuickLook fork (QL-Win or the `emako` fork) attempting
Direction B. It is possible but not the recommended path; Direction A delivers the
user-visible goal (reuse the OS previewers) with prior art already in-tree.

## Prior-art search summary

- **In-tree**: `QuickLook.Plugin.OfficeViewer` hosts OS preview handlers
  (format-agnostic core, Office-scoped usage). ← strongest prior art.
- **QL-Win org**: `QuickLook.Plugin.OfficeViewer-Native` — same hosting pattern.
- **No generic "any registered handler" fallback plugin** found in QL-Win or the
  `emako/QuickLook` fork → this is the gap the build task fills.
- **External**: Microsoft PowerToys *Peek* (MIT) — reference implementation of the
  exact pattern.

## Sources

- [Preview Handlers and Shell Preview Host — Microsoft Learn](https://learn.microsoft.com/en-us/windows/win32/shell/preview-handlers)
- [How to Register a Preview Handler — Microsoft Learn](https://learn.microsoft.com/en-us/windows/win32/shell/how-to-register-a-preview-handler)
- [Building Preview Handlers — Microsoft Learn](https://learn.microsoft.com/en-us/windows/win32/shell/building-preview-handlers)
- [QuickLook.Plugin.OfficeViewer-Native (QL-Win)](https://github.com/QL-Win/QuickLook.Plugin.OfficeViewer-Native)
- In-tree: `QuickLook.Plugin/QuickLook.Plugin.OfficeViewer/PreviewHandlerHost.cs`, `ShellExRegister.cs`
- Microsoft PowerToys *Peek* (reference host implementation, MIT)
