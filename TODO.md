# TODO — FluentCleaner

Prioritized backlog from the review in [`docs/AUDIT.md`](docs/AUDIT.md).
Order = do top-down. IDs match the audit. `file:line` points at the code to change.

Legend: **P0** data-loss/RCE · **P1** security/privacy/correctness · **P2** hardening · **P3** polish.

---

## P0 — Do first (data loss / code execution)

- [ ] **P0-1 · Dangerous-target deny-list.** Before queuing any path in
  `CleaningService.FindFiles` and any key in `FindRegistryItems`, refuse rules
  whose resolved base is a drive root, `%WinDir%`, `%ProgramFiles(x86)?%`,
  `%UserProfile%` root, `%SystemDrive%\Users`, or a registry hive root /
  `HK*\SOFTWARE`/`SYSTEM` top level. Surface refusals in the UI/log instead of
  silently running. Files: `FluentCleaner/Services/CleaningService.cs`,
  `FluentCleaner.Core/Services/PathExpander.cs`. **Add tests (see P1-6).**
- [ ] **P0-2 · Recycle Bin by default.** Replace `File.Delete` (`CleaningService.cs:183`)
  with a Recycle-Bin send (`IFileOperation`/`SHFileOperation` `FOF_ALLOWUNDO`, or
  `Microsoft.VisualBasic.FileIO.FileSystem.DeleteFile(..., SendToRecycleBin)`).
  Add an opt-in "permanent delete (faster)" toggle. For registry, export a `.reg`
  backup of each tree before `DeleteSubKeyTree`.

## P1 — Security / privacy / correctness

- [ ] **P1-1 · Encrypt API keys at rest.** DPAPI (`ProtectedData`, CurrentUser) or
  Credential Manager for `Groq/OpenAi/AnthropicApiKey`; strip secrets from
  `AppSettings.ExportTo`. Files: `FluentCleaner/Services/AppSettings.cs:73-75,152`.
- [ ] **P1-2 · Consent for imported post-clean commands.** On `AppSettings.ImportFrom`
  (`:156`), if `PostCleanCommands` non-empty force `PostCleanEnabled=false` and
  require explicit re-consent; consider dropping post-clean commands from import
  entirely. Same guard for the portable `settings.json` next to the exe.
- [ ] **P1-3 · Fix PowerShell injection in AppX removal.** Validate `PackageName`
  (`^[A-Za-z0-9.\-_ *]+$`) or use parameterized script block / package API.
  Files: `FluentCleaner/Services/AppxService.cs:74,101`, `ViewModels/CliDebloatModule.cs:138`.
- [ ] **P1-4 · Disclose AI data flow.** README privacy note + first-run/inline hint
  naming providers (default Groq) and what "Explain"/generation sends; keep opt-in.
  Files: `FluentCleaner/Services/AiExplainer.cs`, `README.md`.
- [ ] **P1-5 · Supply-chain integrity.** Code-sign releases, publish SHA-256 for
  releases + bundled `Winapp2.ini`; optionally verify a hash on custom DB import.
- [ ] **P1-6 · Add a test project.** xUnit against `FluentCleaner.Core`: parser,
  `PathExpander` (wildcards + `%SystemDrive%` bare-drive case), `ExclusionRule.Matches`,
  and the P0-1 deny-list. Wire into `.github/workflows/build.yml` (currently build-only).

## P2 — Hardening & best practice

- [ ] **P2-1 · Opt-in diagnostic logging** for skipped/failed deletes without
  breaking the never-crash `catch { }` contract. (`CleaningService.cs:63,188,201`)
- [ ] **P2-2 · `AiExplainer`: `ConcurrentDictionary` cache + cap + explicit HTTP timeout.** (`:13-14`)
- [ ] **P2-3 · Carry file size in `ScanResult`** to drop the second `FileInfo.Length`
  `stat` in Clean. (`CleaningService.cs:182`, `ScanResult.cs`)
- [ ] **P2-4 · De-duplicate `RunPostCleanTasksAsync`** into one service.
  (`CleanerPageViewModel.cs:277` + `SilentRunner.cs:86`)
- [ ] **P2-5 · Rotate/cap `auto.log`.** (`SilentRunner.WriteLogAsync`)
- [ ] **P2-6 · Consider streaming parse / lazy `RawText`** for large custom DBs. (`Winapp2Parser.cs`)
- [ ] **P2-7 · Bottom-up empty-dir prune** instead of enumerate-all-then-sort. (`CleaningService.cs:283-300`)
- [ ] **P2-8 · Honor cancellation in the registry-delete loop.** (`CleaningService.cs:191`)
- [ ] **P2-9 · Restrict rule-path env expansion to the known var set.** (`PathExpander.cs:52`)

## P2/P3 — UX

- [ ] **U-2 · Final "permanently delete N files / X GB?" summary** before Clean in the main flow. (`CleanerPage.xaml.cs`)
- [ ] **U-3 · Risk framing for `.ps1` execution** and a preview for AI-generated scripts. (`CustomPage.xaml.cs:259`, `NewCleanerDialog`)
- [ ] **U-4 · "Explain uploads rule data" inline hint.**
- [ ] **U-1 · Surface Recycle-Bin behavior in UI copy** once P0-2 lands.

## P3 — Polish / product / legal

- [ ] **U-5 · Single-source the version.** Reconcile `FluentCleaner.csproj` (26.08.01)
  with `version.txt` / `Inspector/wwwroot/versions.json` (26.08.03); derive from one file.
- [ ] **Add a "Safety" section to README** (exclusions, no elevation, Recycle Bin, open engine).
- [ ] **Qualify "no telemetry"** claim to exclude opt-in AI, add privacy statement.
- [ ] **Add THIRD-PARTY/NOTICE** for bundled `Winapp2.ini` (confirm license + attribution).
- [ ] **In-app "use at your own risk / back up first" notice** (README FAQ already says it).
- [ ] **Link `WHY_CLASSIC_IS_CLOSED_SOURCE.md`** from the README.

---

### Suggested better options / services (from the audit)
- Deletion: `IFileOperation` (COM) gives progress + undo + collision handling — a
  strict upgrade over raw `File.Delete`.
- Secrets: Windows **Credential Manager** or **DPAPI** instead of plaintext JSON.
- Releases: **code signing** (even a cheap OV cert) removes SmartScreen friction and
  directly supports the README's anti-counterfeit message.
- Tests/CI: xUnit + coverage gate in Actions; keep CodeQL (already good).
