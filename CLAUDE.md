# CLAUDE.md — FluentCleaner agent & architecture guide

> Purpose: give Claude Code / Codex a durable map of this repo so you can act
> without re-reading everything. Keep this file updated when structure changes.
> Companion files: `AGENTS.md` (short pointer), `docs/AUDIT.md` (findings),
> `TODO.md` (prioritized work).

## What this is
FluentCleaner is a Windows disk/junk cleaner (a modern, telemetry-free take on
old CCleaner). It parses the community **Winapp2.ini** rule format and deletes
the files/registry keys those rules target. Two front-ends share one engine.

## Solution layout (`FluentCleaner.slnx`)
| Project | TFM | Role |
|---|---|---|
| `FluentCleaner.Core` | `netstandard2.0` | **The engine.** Parser, models, path expansion, detection, cleaning helpers. No UI. Source of truth. |
| `FluentCleaner` | `net10.0-windows` (WinUI 3, self-contained) | Modern GUI + CLI terminal + `/AUTO` silent runner. **Links Core `.cs` files in as source**, and keeps its *own* copies of some services (`AppSettings`, `AiExplainer`, `CleaningService`) — see csproj comment lines 33-49. |
| `FluentCleaner.Classic` | .NET Framework 4.8 (WinForms) | **Closed source.** Only localization JSON + docs are in this repo. Do not look for its `.cs`. |
| `FluentCleaner.Inspector` | Blazor WASM (`docs/`) | Static GitHub-Pages "rule inspector" site. Not tests. |

⚠️ **Core-vs-app duplication trap:** `CleaningService.cs`, `AppSettings.cs`,
`CustomEntryService.cs`, `AiExplainer.cs` exist in the WinUI app under
`FluentCleaner/Services/`. The csproj also *links* the Core copies of some
files. When editing engine logic, confirm **which copy actually compiles into
the target** before editing — the WinUI app compiles `FluentCleaner/Services/CleaningService.cs`
(app copy), and links Core's `Winapp2Parser/PathExpander/DetectionService/etc.`.
Grep both locations and check `FluentCleaner.csproj` `<Compile Include>` list.

## Core data flow (read this before touching cleaning logic)
```
Winapp2.ini ──Winapp2Parser.Parse──▶ List<CleanerEntry>
CleanerEntry { DetectKeys, DetectFiles, SpecialDetect, FileKeys[], RegKeys[], ExcludeKeys[] }
   │
   ├─ DetectionService.IsInstalled(entry)  → show only installed apps
   │
   └─ CleaningService.AnalyzeAsync(entry)  → ScanResult (read-only, builds delete list)
             │  PathExpander.ResolvePaths(FileKey.Path)  → expands %VARS% + * wildcards
             │  EnumerateFilesSafe → applies ExcludeKeys + global exclusions + ProtectedSegments
             ▼
      CleaningService.CleanAsync(ScanResult) → File.Delete + registry delete (PERMANENT)
```
Key files: `FluentCleaner.Core/Services/{Winapp2Parser,PathExpander,DetectionService}.cs`,
`FluentCleaner/Services/CleaningService.cs`.

## Winapp2 format cheat-sheet (avoid re-deriving)
- `[Section]` = one cleaner entry. Trailing ` *` marks community entry (stripped).
- `DetectN` / `DetectFileN` / `SpecialDetect` = "is this app installed?" (OR logic).
- `FileKeyN=<path>|<pattern(s)>|<FLAG>` — FLAG ∈ `RECURSE`, `REMOVESELF`. patterns `;`-sep.
- `RegKeyN=<HIVE\SubKey>[|ValueName]` — no ValueName = delete whole key tree.
- `ExcludeKeyN=<FILE|PATH|REG>|<path>[|pattern]` — protects paths/keys.
- Path vars handled in `PathExpander.BuildVarMap()`; unknown `%VAR%` fall through to `Environment.ExpandEnvironmentVariables`.
- Full spec: `Winapp2-Format_EN.md`.

## Entry points & side-effecting surfaces (security-relevant)
- `App.xaml.cs:52` `OnLaunched` — parses `/AUTO` and `/SHUTDOWN` flags.
- `Services/SilentRunner.cs` — headless clean, writes `%AppData%\FluentCleaner\auto.log`, can run `shutdown.exe`, runs **post-clean commands**.
- `Services/CleaningService.cs` — `File.Delete` (no Recycle Bin), registry tree delete, `CreateFileW` P/Invoke.
- `Services/AppxService.cs` — spawns `powershell.exe` to remove AppX packages.
- `Services/TaskSchedulerService.cs` — spawns `schtasks.exe` (uses `ArgumentList`, safe).
- `Services/AiExplainer.cs` — HTTPS calls to Groq/OpenAI/Anthropic; keys from `AppSettings`.
- `ViewModels/CleanerPageViewModel.cs:277` + `SilentRunner.cs:86` — `RunPostCleanTasksAsync` runs `cmd.exe /c <line>` from settings (**duplicated logic**).
- `Views/CustomPage.xaml.cs:259` — runs user `.ps1` via `powershell -ExecutionPolicy Bypass`.
- `Services/AppSettings.cs` — single `settings.json` (portable next to exe OR `%AppData%`); **shared with Classic**; stores AI keys in plaintext; `Export/ImportFrom` copies whole file incl. keys + post-clean commands.

## Build / run
```
dotnet build FluentCleaner/FluentCleaner.csproj -c Release -p:Platform=x64
```
Needs Windows + .NET 10 SDK + Windows App SDK 2.0.1. CI: `.github/workflows/build.yml`
(x64+arm64 build only, **no tests**), `codeql.yml` (security-extended, actions pinned — good).
There is **no test project** — see TODO P1.

## Conventions
- `Nullable` + `ImplicitUsings` enabled. C# 12/13 idioms (collection exprs `[]`, primary ctors, `record struct`).
- Style: dense one-liners, `//` inline comments, occasional lowercase/informal prose. Match it.
- Errors are frequently swallowed with `catch { }` by design (a broken rule must never crash a clean). When adding logging, do not change this contract for the clean loop; log via the existing `crash.log`/`auto.log` pattern.
- Localization: WinUI uses `.resw` under `Strings/{locale}/`; Classic uses `Localization/*.json`. Never edit XML structure of `.resw`, only `<value>`.
- Manifest is `asInvoker` (no elevation) — keep it that way; it's a deliberate safety boundary.

## Known correctness/version notes
- Version is inconsistent: `FluentCleaner.csproj` = `26.08.01`, `version.txt` / `Inspector/wwwroot/versions.json` = `26.08.03`. Bump together.
- See `docs/AUDIT.md` for the full bug/security/UX list and `TODO.md` for priority.
