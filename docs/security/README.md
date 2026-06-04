# Security study — attack surface & findings ledger

Defensive, in-depth audit of QuickLook. Goal: find real defects, propose minimal
fixes, ship them as atomic PRs (**one issue = one fix = one commit**).

## Ground rules

- **Verify before claiming.** Read the path from untrusted input to sink; build an
  adversarial input that triggers the defect; confirm it. No hypothetical findings
  in the ledger — `confirmed` rows only; `suspected` rows live in "Leads" below.
- **One issue → one minimal fix → one commit**, subject naming the issue. Land a
  regression test in the same commit when the bug is hermetically reproducible;
  otherwise state why not.
- **Responsible disclosure (public repo).** A confirmed, exploitable,
  upstream-unpatched vulnerability is NOT published here (commit / PR / public doc)
  before coordinating private disclosure with upstream `QL-Win`. Until coordinated,
  the public ledger and commit messages must not hand a reader a weaponized exploit.
  Hold the detail and ask when unsure.
- **Severity**: Critical = RCE / sandbox escape / silent code execution · High =
  memory corruption / arbitrary file write / privilege boundary break · Medium =
  DoS / info disclosure / traversal with constraints · Low = hardening / defense in
  depth.

## Attack surface (audit targets)

| # | Surface | Entry point(s) | Primary risks |
|---|---|---|---|
| S1 | Untrusted file parsers | every `QuickLook.Plugin.*` `IViewer` | native decoder memory corruption, .NET parser DoS, XXE, traversal |
| S2 | Plugin trust & install | `PluginManager.cs` (`Assembly.LoadFrom`), `PluginInstaller` (`.qlplugin` ZIP) | no signature check, zip-slip, in-proc full-priv code |
| S3 | Native shell hooks + hotkey | `QuickLook.Native*/Shell32.cpp`, `DialogHook.cpp`, `GlobalKeyboardHook.cs` | injection into Explorer, native memory safety, managed↔native boundary |
| S4 | IPC named pipe | `PipeServerManager.cs` | pipe ACL/SID scope, plaintext command parse, path/command injection |
| S5 | WebView2 content | `HtmlViewer`, `ImageViewer/Webview/*`, Markdown, Chm, Font | script exec, local-file reach, remote loads (SSRF/exfil), navigation |
| S6 | Auto-update | `Helpers/Updater.cs` | TLS trust, redirect handling, no SSL pinning |
| S7 | Mark-of-the-Web | `UnblockZoneIdentifier` dep, zone handling | previewing downloaded files, MotW stripping |
| S8 | Path handling | shared helpers, plugin file IO | symlink/junction follow, UNC, long paths, zone eval |

## Findings ledger (confirmed only)

| ID | Surface | Severity | Title | Status | Fix commit |
|----|---------|----------|-------|--------|------------|
| — | — | — | _none confirmed yet_ | — | — |

Status: `open` → `fix-proposed` → `fixed` (commit) / `disclosed-upstream` / `wontfix`.

## Leads (suspected, unverified)

Working notes for surfaces under review. Promote to the ledger only after the
defect is reproduced. Keep exploit-enabling detail out of this public file until
disclosure is coordinated (see Ground rules).

- _(none yet)_
