# AGENTS.md

FluentCleaner — Windows junk cleaner (WinUI 3 + .NET Framework Classic) that
parses Winapp2.ini rules and deletes the files/registry keys they target.

**Read `CLAUDE.md` first** — it is the canonical architecture/agent guide.
This file exists so Codex and other agents find it under the conventional name.

## 30-second orientation
- Engine lives in `FluentCleaner.Core` (netstandard2.0). Everything else is a front-end.
- The WinUI app (`FluentCleaner/`) **links Core source files in** AND keeps its own
  copies of `CleaningService`, `AppSettings`, `AiExplainer`, `CustomEntryService`.
  Before editing engine code, confirm which copy compiles into your target
  (grep both `FluentCleaner/Services/` and `FluentCleaner.Core/Services/`, and
  check `<Compile Include>` in `FluentCleaner/FluentCleaner.csproj`).
- `FluentCleaner.Classic` is **closed source** — only localization/docs are here.
- Deletion is **permanent** (`File.Delete`, no Recycle Bin). Registry keys are deleted directly.
- App runs as `asInvoker` (no elevation) — do not change the manifest.

## Before you start work
1. Skim `CLAUDE.md` (map + data flow + security-relevant surfaces).
2. Check `docs/AUDIT.md` for the current findings so you don't re-discover them.
3. Check `TODO.md` for the prioritized backlog; pick from the top.

## Build
`dotnet build FluentCleaner/FluentCleaner.csproj -c Release -p:Platform=x64`
(Windows + .NET 10 SDK + Windows App SDK 2.0.1 required.) No test project exists yet.

## Guardrails
- Don't weaken the destructive-operation safety story (exclusions, `ProtectedSegments`,
  reparse-point skipping, `asInvoker`). If anything, strengthen it — see AUDIT P0 items.
- Keep the `catch { }` "never crash a clean" contract in the clean loop; add logging without throwing.
- AI features send rule data to third parties and are opt-in; keep them opt-in and disclosed.
