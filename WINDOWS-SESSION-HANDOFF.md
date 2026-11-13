# Windows Session Handoff — PcmHammer Avalonia Logger

> **Read this first.** It is written for a Claude Code session starting fresh on Gene's
> **Windows** machine. The memory files from the Linux sessions live on the Linux box only
> (`~/.claude/...`) and are **not** available here, so this document is your context. It is a
> scratch handoff for the machine switch — **delete it before any PR to atshaw/pcmhammer.**

_Last updated: 2026-11-13 (from the Linux session, commit after `cf2a3f2`)._

---

## TL;DR — where we are
We are building the PcmHammer **logger** as a standalone, cross-platform **Avalonia** app
(`PCMLoggerAvalonia`), on top of atshaw1994's Avalonia fork, porting Gene's earlier logger work.
The logger is **feature-complete and bench-validated on Linux** (P01/P59 via OBDX Pro VT). We are
now **moving active development to Windows** to validate on Windows and continue.

Branch: **`logger`** on Gene's fork **GT238/PcmHammer**. Last feature commit on the Linux side was
`cf2a3f2` ("Borrow the bus to read codes instead of stopping logging").

---

## People & hard rules (IMPORTANT — follow exactly)
- **Gene** = the user (GitHub **GT238**). Treat his email and personal details as private; never put
  them in external requests.
- **Antus** = PcmHammer maintainer. **atshaw1994** = author of the Avalonia fork `PcmHammerMVVM`.
- **No-push rule:** never push / open PRs / post to **atshaw or mainstream PcmHammer** without Gene's
  explicit go-ahead. Local commits only. **Gene drives every push himself** (he has the PAT). Pushing
  WIP to his **own** fork `GT238/PcmHammer` for machine-sync is fine **when he asks**. Get approval
  **before every PCM write** (flashing/clearing).
- **Commit attribution:** end commit messages with
  `Co-Authored-By: Claude Opus 4.8 <noreply@anthropic.com>`.
  End PR descriptions with the "Generated with Claude Code" line.
- **Antus's architecture directive (verbatim):** _"keep it seperate, but try to adhear to MVVM type
  design and put as much of the shared logic in pcmlibrary as possible with the seperate front end
  only doing dumb front end stuff."_ → Logger stays **standalone**, **MVVM**, heavy lifting in
  **PcmLibrary**, front end just binds/renders.
- **Gene's steer:** mimic the **WinForms PcmLogger** where reasonable (it builds Antus's trust).

---

## Repo layout on THIS (Windows) machine
- **Work here:** `E:\Python Apps\PcmHammer-logger` — a **git worktree** on branch **`logger`**. This is
  the Avalonia logger checkout. Build/run and commit from here.
- **Do NOT disturb:** `E:\Python Apps\PcmHammer-fork` — Gene's clone used for **WinForms** PR work
  (branch `integration-test`). The worktree above shares this repo's `.git`.
- Ignore the non-git ZIP copy at `C:\Users\genet\source\repos\PcmHammer-develop` (no `.git`, stale).
- **Remotes** (in the shared repo): `origin` = `https://github.com/GT238/PcmHammer.git` (Gene's fork —
  where `logger` lives); `upstream` = `https://github.com/PcmHammer/PcmHammer.git` (mainstream).
  The Avalonia base is **atshaw1994/PcmHammerMVVM** `develop-avalonia` (not a remote here by default).
- **Sync:** both machines push/pull `logger`. The Linux repo is `~/PcmHammer-avalonia`
  (its remotes: `mine` = GT238/PcmHammer, `origin` = atshaw1994/PcmHammerMVVM).
- To update this worktree: `cd "E:\Python Apps\PcmHammer-logger"` then `git pull`.

---

## What's built (the logger: `Apps/UI/Avalonia/PCMLoggerAvalonia`)
Standalone MVVM Avalonia app with a **WinForms-style tab bar**: **Configuration**, **Grid**,
**Dashboard**, **Monitors**, **Trouble Codes** (the `TabControl` replaced the old toolbar
view-switch buttons as of commit `2b6eb5c`). The **Configuration** tab mirrors WinForms: Devices
(Select OBD2 Device + summary), Paths (Set/Open Log Folder, default Documents — recording writes
there), View (show/hide Grid/Dashboard/Monitors + Theme System/Light/Dark), Options (Auto start
logging). WinForms items without a backend yet (Zoom, Auxiliary bus, Profiles, Parameters/Debug
tabs) are intentionally omitted, not shown as dead controls.

Shared backend (in **PcmLibrary**, per Antus):
- `Logging/LoggingController.cs` — framework-agnostic polling loop; events (RowAvailable, Started,
  Stopped, Failed, RecordingStarted/Stopped); CSV recording; **`BorrowBusAsync`/`ReturnBus`** so a
  reader (trouble codes) can borrow the bus without stopping logging.
- `LogSession`, `ParameterDatabase`, `LogProfileReader`, `LogFileWriter`, `DashboardLayout` /
  `MonitorLayout` / `GaugeLayout`, `PlzFormat` (.plz save/load), `AdxDocument` + `LoggerPackage.FromAdx`
  (TunerPro .adx import), `DiagnosticCode` / `DiagnosticCodeDefinitions`, `Vehicle.TroubleCodes.cs`.

Front-end features:
- Connect (serial **and** J2534 — J2534 gated to Windows at runtime), Open `.LogProfile`, Start/Stop.
- Live **Grid**; **Dashboard** gauges (add / remove / drag-to-move / repoint to any supported param;
  three styles: dial / text / bar; save & load `.plz`; import `.adx`).
- **Monitor** WinForms-style strip chart (pause, window zoom, hover cursor, per-trace show/hide).
- **CSV recording** (Record button).
- **Trouble Codes** (newest): Read Codes / Clear Codes + Code/Status/Description list. Read & clear
  call shared `Vehicle.ReadTroubleCodes` / `Vehicle.ClearTroubleCodes` (VPW + CAN handled in
  PcmLibrary). It **borrows the bus** from the running logging loop, so the strip chart/recording keep
  running. Clear confirms first (FluentAvalonia `FAContentDialog` — 3.1.0 renamed ContentDialog ⇒
  `FAContentDialog`/`FAContentDialogResult`/`FAContentDialogButton`) and notes permanent codes remain.
- **Decision:** keep explicit **Connect/Start/Stop** buttons for now. Not porting WinForms'
  continuous/auto-start-on-connect logging yet — that is the remaining WinForms-parity gap.

Key files: `ViewModels/MainWindowViewModel.cs`, `Views/MainWindow.axaml(.cs)`,
`Views/GaugeEditDialog`, `Views/DeviceSelectDialog`, `ViewModels/GaugeTile.cs`,
`ViewModels/TroubleCodeRow.cs`; shared loop `Apps/PcmLibrary/Logging/LoggingController.cs`.

---

## Device & platform facts (confirmed on Windows)
- **Gene's interface = OBDX Pro VT.** In code it is a **serial** device:
  `public class OBDXProDevice : SerialDevice` (`DeviceType = "OBDX Pro"`). It connects over a **COM
  port** (`COM3` on Windows, `/dev/ttyACM0` on Linux). **Use Serial port → "OBDX Pro Serial (COM3)".
  Do NOT use J2534 for this hardware.**
- **CAN is native to the OBDX serial driver** (`OBDXProDevice : ICanTarget, ICanChannel`), not via J2534.
- **J2534** is only for generic pass-through interfaces (Tactrix, etc.). On Windows the J2534 code path
  lights up (the dialog finds the installed driver), but vendor J2534 DLLs are **32-bit**, and a 64-bit
  process cannot load them ("Unable to load the J2534 DLL"). To exercise generic J2534 you must run
  **x86**: `dotnet run ... -r win-x86 --self-contained`. **Not needed for Gene's OBDX** — ignore it.
- **.NET SDK = 10.0.403** installed (same as Linux). Plain `dotnet run` (64-bit) is correct for the
  serial/OBDX path.

---

## Build / run on Windows (from `E:\Python Apps\PcmHammer-logger`)
```powershell
# Logger (use this for Gene's OBDX, serial path, 64-bit is fine):
dotnet run --project Apps\UI\Avalonia\PCMLoggerAvalonia.Desktop

# Flasher (Avalonia PCMHammer) — use the .Desktop head, NOT .Linux:
dotnet run --project Apps\UI\Avalonia\PCMHammerAvalonia.Desktop
```
- In **Select Device**, choose **Serial port → OBDX Pro Serial (COM3)** (type "OBDX Pro" or Auto-detect).
- The flasher needs PCM **kernel** binaries beside the exe for read/write/flash (atshaw's Avalonia
  csproj doesn't embed them). Launch/connect/identify works without them; a read that fails for a
  missing kernel means the `Kernel-*.bin` must sit next to the built exe.

---

## Bench status
- **Linux:** full logger validated on bench P01/P59 via OBDX on `/dev/ttyACM0` — connect, identify,
  live log, dashboard, monitor, trouble-codes read, and borrow-bus (chart keeps running during a read).
- **Windows:** app launches; Select Device shows J2534 enabled and finds the OBDX J2534 driver, but the
  **serial OBDX path is the one to use**. A full serial Connect on Windows was not yet reported — that
  is the next thing to confirm.

---

## PR / push status
- `logger` is pushed to **GT238/PcmHammer** (Gene's fork) **for machine sync only — NOT the atshaw PR.**
- The real logger PR to atshaw/pcmhammer is pending Gene + Antus agreeing the approach: **one complete
  push** of the working foundation vs **stepped PRs** (a ~A–G grouping: shared backend → app scaffold →
  dashboard → monitor → files/record → device dialog → trouble codes). **Do not push to
  atshaw/upstream** without Gene's explicit say-so.
- Separate, Linux-only backlog item (not a Windows concern): a 2nd Linux serial fix — port Gene's CLI
  "CRC listen-again" into the Avalonia `KernelVerifier.cs`.

---

## Immediate next steps
1. Confirm the logger **connects over the OBDX serial path on Windows** (COM3) — connect, identify,
   open a profile, log live, read trouble codes.
2. Continue feature work as Gene directs; keep heavy logic in PcmLibrary (MVVM, dumb front end).
3. Prep the atshaw PR only when Gene says, in the form he and Antus agree.

(The detailed Linux running journal `AVALONIA-PORT-NOTES.md` is git-excluded and does **not** sync here;
this handoff is the Windows-side substitute.)
