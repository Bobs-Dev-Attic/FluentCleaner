# FluentCleaner — Project Review & Security Audit

_Reviewer: senior-engineer / security-analyst / UX / pentester / product & legal pass._
_Date: 2026-09-05 · Scope: `FluentCleaner` (WinUI), `FluentCleaner.Core`, build/CI, docs._
_Classic is closed-source and out of scope (only its localization/docs are in-repo)._

Severity: **P0** data-loss/RCE · **P1** meaningful security/privacy/correctness ·
**P2** hardening/best-practice · **P3** polish. IDs map to `TODO.md`.

---

## Executive summary

FluentCleaner is a genuinely well-built, focused tool. The engine is small,
readable, and makes several *correct* safety decisions that most cleaners get
wrong: reparse-point/junction skipping (`CleaningService.cs:122`), a read-only
Analyze phase separate from Clean, per-entry + global exclusion rules, a
`DELETE`-share probe before counting a file (`TryGetDeletableSize`), and a
no-elevation (`asInvoker`) manifest. The "no registry-cleaner / no secure-wipe
theater" philosophy in the README is technically sound.

The risk profile is dominated by one fact: **the app deletes whatever a rule
tells it to, permanently, and it explicitly accepts untrusted rule sources**
(custom databases, AI-generated entries, imported settings). The strongest
findings below all follow from that: there is no blocklist of catastrophic
target paths/keys, deletion bypasses the Recycle Bin, and configuration
(`settings.json`) can carry both plaintext API keys and arbitrary post-clean
`cmd.exe` commands that execute silently.

Top priorities: **P0-1** (dangerous-root guard), **P0-2** (Recycle-Bin default),
**P1-1** (encrypt API keys), **P1-2** (consent for imported post-clean commands).

---

## P0 — Data loss / code execution

### P0-1 · No guard against catastrophic delete targets
`FluentCleaner/Services/CleaningService.cs` (whole clean path) + `PathExpander.cs`.

The only built-in file safety net is a **single** segment,
`\IndexedDB\chrome-extension_` (`CleaningService.cs:419-424`). There is **no
blocklist** for root/system locations. A rule such as
`FileKey1=%SystemDrive%\|*.*|RECURSE` expands `%SystemDrive%`→`C:\`
(`PathExpander.cs:60-61` even *fixes* the bare-drive case so it targets the real
root) and then `EnumerateFilesSafe` walks the entire drive deleting everything
the user can touch. `%UserProfile%`, `%ProgramFiles%`, `%WinDir%` are equally
unguarded. Registry has **no** protected-key list at all — a rule
`RegKey1=HKCU\Software` wipes the user's entire per-user software hive (HKLM is
mostly saved only by the `asInvoker` ACL boundary, not by the app).

This matters because the app *invites* untrusted rules: **Settings ▸ Database ▸
Custom** (README line 204), **AI-generated entries** (`AiExplainer.GenerateEntryAsync`),
and third-party winapp2 flavors (README lines 194-212). A single bad/malicious/AI-
hallucinated entry = total data loss with no undo.

**Fix:** add a hard deny-list evaluated in `Analyze`/`FindFiles` before queuing
any path, and in `FindRegistryItems` before queuing keys. Refuse (and surface)
FileKeys whose resolved base is a drive root, `%WinDir%`, `%ProgramFiles%(x86)`,
`%UserProfile%` root, `%SystemDrive%\Users`, etc., and RegKeys at hive root or
`HK*\SOFTWARE`/`SYSTEM` top level. Require an explicit "I understand" for rules
that resolve above a depth threshold. This is defense-in-depth that pairs with P0-2.

### P0-2 · Permanent deletion — no Recycle Bin, no undo
`CleaningService.cs:183` `File.Delete(file)`; registry via `DeleteSubKeyTree`.

Every delete is irreversible. Users of a "cleaner" reasonably expect either a
Recycle-Bin round-trip or a restore point. Combined with P0-1, one wrong rule is
unrecoverable. CCleaner's own default sends to Recycle Bin for exactly this reason.

**Fix:** default file deletion to the Recycle Bin via
`SHFileOperation`/`IFileOperation` (`FOF_ALLOWUNDO`) or
`Microsoft.VisualBasic.FileIO.FileSystem.DeleteFile(..., RecycleOption.SendToRecycleBin)`;
offer a "permanent delete (faster)" opt-in. For registry, consider exporting a
`.reg` backup of each key tree before deletion.

---

## P1 — Security / privacy / correctness

### P1-1 · API keys stored (and exported) in plaintext
`FluentCleaner/Services/AppSettings.cs:73-75` (`GroqApiKey`, `OpenAiApiKey`,
`AnthropicApiKey`), persisted by `Save()` to `settings.json`; `ExportTo` (`:152`)
serializes the whole instance — so **backup/share exports leak the keys**, as does
the portable `settings.json` sitting next to the exe.

**Fix:** encrypt keys at rest with DPAPI (`ProtectedData.Protect`, `CurrentUser`
scope) or the Windows Credential Manager. Exclude secrets from `ExportTo` (or
prompt). Never write them to logs.

### P1-2 · Imported settings can execute arbitrary commands silently
`AppSettings.ImportFrom` (`:156`) loads an arbitrary JSON and replaces the live
instance, including `PostCleanEnabled` + `PostCleanCommands`. Those run as
`cmd.exe /c <line>` after the next clean — silently, `CreateNoWindow`
(`CleaningService.cs:290`, `SilentRunner.cs:97`). So "import this settings backup
a friend sent" ⇒ config-driven code execution. The portable `settings.json` next
to the exe is the same vector for anything that can write beside the binary.

**Fix:** on import, if `PostCleanCommands` is non-empty, force `PostCleanEnabled=false`
and show the commands for explicit re-consent. Consider signing/validating the
portable settings file, or dropping post-clean commands from import entirely.

### P1-3 · PowerShell command injection via AppX removal / raw name
`FluentCleaner/Services/AppxService.cs:74,101-106`. The command is built by string
interpolation into `-Command "…'*{PackageName}*'…"`; `BuildPsi` escapes `"` but
**not** `'`. `PackageName` comes from `Winappx.ini` and, in the terminal, from the
user's **raw input** fallback (`CliDebloatModule.cs:138`
`new AppxEntry(name, name, null)`). Input like `foo';<cmd>;#` breaks out of the
quoting. Self-inflicted for a local user, but a real injection sink if any name
ever comes from an untrusted list.

**Fix:** pass arguments via a script block with parameters, or validate
`PackageName` against `^[A-Za-z0-9.\-_ *]+$`, or invoke the package-manager API
directly instead of building a shell string.

### P1-4 · Third-party AI data flow is undocumented in-repo
`AiExplainer.cs`. Default provider is **Groq** (`AppSettings.cs:70`). "Explain"
sends the entry name, file-key paths, and registry keys to the provider
(`BuildPrompt`, `:225-249`). Paths are the *unexpanded* `%VAR%` forms (good — no
username leak), and the feature is opt-in (needs a key), but the README markets
"no telemetry / no spyware" while shipping an opt-in feature that sends data to a
US LLM API. That's defensible only if it's disclosed.

**Fix:** add a short privacy note (README + first-run tooltip) naming the
providers, what is sent, and that it's opt-in. Prefer an explicit consent toggle.

### P1-5 · No integrity verification of rule databases / downloads
Custom and third-party `winapp2.ini` files (and the bundled one) are trusted
verbatim. Given the README's own warning about counterfeit distribution sites,
the supply-chain story is incomplete: no checksum/signature on the bundled DB,
no signed releases mentioned.

**Fix:** publish SHA-256 for releases and the bundled DB; ship the app
**code-signed** (also fixes SmartScreen friction — see P2/marketer note); optionally
verify a hash/signature on custom DB import.

### P1-6 · No test project at all
No unit tests anywhere (`FluentCleaner.Inspector` is a Blazor site, not tests).
The parser, `PathExpander` (wildcards, `%SystemDrive%` edge case), exclusion
matching (`ExclusionRule.Matches`), and the P0-1 deny-list are exactly the kind of
pure logic that must be pinned by tests before shipping deletion behavior.

**Fix:** add an `xUnit` project targeting `FluentCleaner.Core`; cover parser,
path expansion, exclusions, and (once added) the dangerous-path guard. Wire into
`build.yml`.

---

## P2 — Hardening & best practice

- **P2-1 · Blanket `catch { }` hides failures.** `CleaningService` Analyze/Clean,
  `AppSettings.Load/Save`, `SilentRunner` all swallow everything (e.g.
  `CleaningService.cs:63,188,201`). Intentional for resilience, but there is *no*
  diagnostic trail for "why didn't X get cleaned / why did a delete fail." Add
  opt-in debug logging (reuse the `crash.log`/`auto.log` pattern) without breaking
  the never-crash contract, and narrow catches where practical.
- **P2-2 · `AiExplainer._cache` is a non-concurrent `Dictionary`** (`:14`) mutated
  from async continuations; concurrent Explain calls can corrupt it. Use
  `ConcurrentDictionary`. Also `_http` (`:13`) never sets an explicit `Timeout`
  (defaults to 100 s) and the cache is unbounded — cap it.
- **P2-3 · Double `stat` per file.** `TryGetDeletableSize` reads
  `FileInfo.Length` in Analyze, then Clean re-reads it (`CleaningService.cs:182`).
  Capture size in the `ScanResult` to avoid a second syscall per file (matters on
  the ~8600-file Firefox entry the code itself cites).
- **P2-4 · Duplicated post-clean logic.** `RunPostCleanTasksAsync` exists twice
  (`CleanerPageViewModel.cs:277`, `SilentRunner.cs:86`) with the same `cmd /c`
  behavior — extract to one service so a hardening fix (P1-2) lands in both.
- **P2-5 · `auto.log` grows unbounded** (`SilentRunner.WriteLogAsync` appends
  forever). Rotate/cap like `CleanHistory` (capped at 50).
- **P2-6 · Full-file load + per-entry `RawText` copy.** `Winapp2Parser` reads the
  whole 1.8 MB DB into memory and stores a verbatim text copy per entry
  (`:33,74`). Fine today; if custom DBs get large, consider streaming and lazily
  reconstructing source text.
- **P2-7 · `Directory.GetDirectories(..., AllDirectories)` in `TryPruneEmptyDirs`
  then sorts by string length** (`CleaningService.cs:288-290`) — O(n log n) plus a
  full re-enumeration after the delete pass. Acceptable, but a bottom-up walk would
  be cheaper on deep trees.
- **P2-8 · No cancellation in the registry-delete loop** (`Clean`, `:191`) — long
  reg operations can't be interrupted like the file loop can.
- **P2-9 · `Environment.ExpandEnvironmentVariables` fallback** (`PathExpander.cs:52`,
  `AppSettings.NormalizePath:169`) will expand *any* `%VAR%`, so process-env
  poisoning could redirect a path. Minor given local trust, but worth pinning the
  known set only for rule paths.

---

## UX review

- **U-1 (ties P0-2):** deletion is permanent with no Recycle-Bin option — the
  single biggest UX/safety gap for a mainstream cleaner. Add it and say so.
- **U-2:** Good: browser-running warning (`CleanerPage.xaml.cs:233`), per-entry
  `Warning=` confirmation (`:254`), and Rule Lab dry-run. **Missing:** a final
  "about to permanently delete N files / X GB — continue?" summary before the
  destructive Clean in the main flow.
- **U-3:** `.ps1` custom scripts run with `-ExecutionPolicy Bypass`
  (`CustomPage.xaml.cs:281`) — there's a confirm dialog (`:241`) but no visible
  "this runs arbitrary PowerShell on your machine" framing, and AI-*generated*
  scripts land straight into that runner. Show the script and a clear risk notice.
- **U-4:** AI errors are surfaced inline as text; good. But there's no visible
  indication (before running) that Explain uploads rule data — add a one-line hint.
- **U-5:** Version shown to users can mismatch (`csproj` 26.08.01 vs
  `version.txt` 26.08.03). Single-source the version.
- **U-6:** `/SHUTDOWN` alone is a no-op by design (README) — fine, but the log/UI
  should say so if a user passes it alone.

---

## Pentester / white-hat perspective

- **Primary attack surface is configuration + rule files, not the network.** The
  highest-value local attacker moves are: drop a portable `settings.json` beside
  the exe with `PostCleanEnabled=true` (silent `cmd` execution on next clean — P1-2),
  point `CustomWinapp2Path` at a malicious DB (mass deletion — P0-1), or socially
  engineer a settings "backup" import (P1-2 + key theft via P1-1 export).
- **Privilege:** `asInvoker` (`app.manifest`) correctly limits blast radius to the
  user's own ACLs — keep it. Scheduled `/AUTO` task runs at that same level.
- **Injection:** the AppX PowerShell path (P1-3) is the one string-built shell
  command; `TaskSchedulerService` and `schtasks`/`shutdown` calls correctly use
  `ArgumentList` / fixed args.
- **Network:** all AI calls are HTTPS with default TLS validation (no cert
  bypass) — good. No auto-update/downloader in the code reviewed, so no unsigned
  fetch-and-run path (keep it that way).
- **Positive controls worth keeping:** reparse-point skip (anti-loop / anti-
  junction-escape), read-only Analyze, `DELETE`-share probe, exclusion precedence.

---

## Product / marketer / founder perspective

- The "honest, no dark patterns" positioning is a real differentiator — but it
  is undercut by (a) shipping unsigned binaries (SmartScreen "unknown publisher"
  scares the exact privacy-minded user you want — see P1-5) and (b) an opt-in
  cloud-AI feature under a "no telemetry" banner (disclose it — P1-4).
- README voice is charming but very informal/self-deprecating ("i'll probably get
  bored") for a tool that **permanently deletes files**. Keep the personality, but
  add a crisp, confidence-inspiring "Safety" section (exclusions, no elevation,
  Recycle Bin once P0-2 lands, open-source engine).
- Donation-funded, solo-dev, closed-source Classic tier is a coherent model; make
  the closed-source rationale (`WHY_CLASSIC_IS_CLOSED_SOURCE.md`) linked from the README.

## Legal / compliance perspective

- **Liability from destructive behavior:** MIT `LICENSE` disclaims warranty, but
  permanent deletion + silent `cmd`/PowerShell execution + `/SHUTDOWN` warrant an
  explicit in-app "use at your own risk / take a backup" notice (the README FAQ
  says it; the app should too).
- **Privacy (GDPR/CCPA):** the AI feature is a transfer of user-derived data to
  third-party US processors. Even opt-in, add a privacy statement naming Groq/
  OpenAI/Anthropic and what is sent (P1-4). "No telemetry" claims should be
  qualified to exclude the opt-in AI calls.
- **Third-party content:** confirm the bundled `Winapp2.ini` license permits
  redistribution and that attribution (already in README) satisfies it. Note the
  winapp2 source in a NOTICE/THIRD-PARTY file.
- **Trademark:** "Fluent" overlaps Microsoft's Fluent design branding; low risk
  but worth awareness. The README already handles the CCleaner/counterfeit-site
  angle well.

---

## Notable things done right (keep)
Reparse-point skipping · read-only Analyze/Clean split · exclusion rule engine +
global exclusions + `ProtectedSegments` scaffold · `asInvoker` manifest ·
`schtasks` via `ArgumentList` · CodeQL `security-extended` with SHA-pinned actions
· HTTPS with default TLS validation · Renovate for dependency updates ·
cancellation tokens through the file scan · `HashSet` dedup on large entries.
