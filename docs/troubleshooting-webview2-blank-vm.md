# Troubleshooting: WebView2 viewers render blank on GPU-less hosts

**Date:** 2026-06-04
**Affects:** HTML viewer and Markdown viewer (both inherit `WebpagePanel`)
**Host class:** virtual machines and remote sessions with no hardware GPU —
Proxmox/QEMU with a SPICE "Red Hat QXL" virtual adapter, generic VMs, and RDP
sessions.

## Symptom

Pressing Space on an `.html`, `.md`, or other WebView2-backed file opens the
preview window, but the content area is **blank** (white or dark, depending on
theme). Other (non-WebView2) viewers — images, text, PDF — work normally.

## Root cause

WebView2 hosts a Chromium instance, and Chromium spawns a separate **GPU
process** that runs inside a sandbox. On a GPU-less host the GPU process cannot
initialise its graphics backend inside that sandbox and crashes at startup with
exit code `0x80000003` (`STATUS_BREAKPOINT`). Chromium retries the GPU process
six times; after the sixth crash it logs *"GPU process isn't usable. Goodbye"*
and **disables rendering entirely**. The WebView2 controller creation then throws
`E_ABORT`, and the panel never paints — hence the blank preview.

Evidence on the affected host:

- `%APPDATA%\pooi.moe\QuickLook\QuickLook.Exception.log` shows the WebView2
  controller-creation `E_ABORT`.
- WPF's `RenderCapability.Tier` measures **0** (software-only compositing) on the
  host — there is no hardware acceleration tier for either WPF or Chromium to use.
- The Chromium GPU-process crash loop (`STATUS_BREAKPOINT` ×6 then *"GPU process
  isn't usable. Goodbye"*) is visible in WebView2 verbose logs.

## The fix

The corrective browser argument is **`--disable-gpu-sandbox`**. This keeps the
overall renderer/network/utility sandboxes intact and drops **only** the
GPU-process sandbox, which lets the GPU process initialise (falling back to
software rendering) instead of crash-looping. Three ways to apply it, in
increasing order of "lives with the app":

### Option 1 — environment variable (per-process, manual)

Set `WEBVIEW2_ADDITIONAL_BROWSER_ARGUMENTS=--disable-gpu-sandbox` in the
environment of the process that launches `QuickLook.exe`. Useful for a one-off
diagnostic; does not survive a normal Start-menu launch.

### Option 2 — Edge WebView2 registry policy (machine-wide, deployment)

Set, under
`HKLM\SOFTWARE\Policies\Microsoft\Edge\WebView2\AdditionalBrowserArguments`,
a value named `QuickLook.exe` with data `--disable-gpu-sandbox`. This is applied
by the WebView2 runtime to every WebView2 host whose executable is named
`QuickLook.exe`. It is an external deployment control — it works without any code
change but must be provisioned separately on each host (and removed cleanly when
no longer wanted).

### Option 3 — fork code change (in-app, automatic) — IMPLEMENTED

`QuickLook.Plugin.HtmlViewer/WebpagePanel.cs` now sets
`CoreWebView2CreationProperties.AdditionalBrowserArguments = "--disable-gpu-sandbox"`
**only when no hardware GPU is detected** (`RenderCapability.Tier == 0`,
i.e. the high word of `RenderCapability.Tier` is `0`). When a real GPU is present
the property is left unset and the full GPU-process sandbox is retained. Because
both the HTML viewer and the Markdown viewer derive from `WebpagePanel`, the
single gate covers both.

This makes the fix travel with the application: GPU-less hosts get a working
preview automatically, and GPU-equipped hosts keep the unmodified, fully
sandboxed behaviour. With Option 3 in place, the **Option 2 registry policy is
now optional** — a belt-and-suspenders deployment control rather than the
primary mechanism.

#### Security note

Dropping the GPU-process sandbox slightly widens the attack surface of the GPU
process *only*. The gate restricts that to hosts that report no hardware GPU,
where Chromium is already confined to a software rendering path — the marginal
exposure is minimal, and the alternative on those hosts is a non-functional
viewer. Hosts with a real GPU are unaffected and keep the full sandbox.

## Verification (code path, independent of registry)

To confirm the **code** sources the flag rather than the deployed registry
policy, temporarily remove the
`HKLM\...\WebView2\AdditionalBrowserArguments\QuickLook.exe` value, run the
rebuilt `QuickLook.Plugin.HtmlViewer.dll` from a shell with no
`WEBVIEW2_ADDITIONAL_BROWSER_ARGUMENTS` environment variable, trigger a preview,
and check that the spawned `msedgewebview2.exe` command line carries
`--disable-gpu-sandbox` and that `QuickLook.Exception.log` shows no new
`E_ABORT`. Restore the registry value afterward so any host without the rebuilt
DLL stays protected. See `docs/webview2-fork-fix-verification-2026-06-04.md` for
the recorded run.
