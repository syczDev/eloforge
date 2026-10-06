# Developing EloForge

Notes for working on the source. The public README is for people using the app.

## Building it yourself

Requires **Visual Studio 2022** (or just the Build Tools) with the C++ desktop workload. There is no solution file and no package restore — everything it depends on is vendored in `cpp/vendor`.

```bash
cd cpp
build.bat
```

That produces `cpp/build/EloForge.exe`. `build.bat debug` produces an unoptimised console build instead, which is the one to run when you want to see what it is doing.

To build the installer as well:

```bash
powershell -ExecutionPolicy Bypass -File cpp/tools/build-installer.ps1 -Version 0.8.1
```

That stamps the version into the exe's resource block, builds it, and wraps it in a per-user MSI. The version the app reports about itself is read back out of that resource block at runtime, so there is one place it is written down.

## Opening straight onto a page

```bash
EloForge.exe --page=live
```

`dashboard`, `live`, `history`, `stats`, `friends`, `agents`, `collection` and `settings` all work. Useful for screenshots, and for looking at one view without clicking through the app.

`--demo-live` fills the live scoreboard, the Store wallet and the account level with an invented lobby built from the real agent, rank and skin catalogue — parties on both teams, skins of every edition — so the screens that otherwise need a game running can be looked at. The names are tagged `#DEMO` and nothing is sent to Riot.

`--demo-lobby` is the same account in the menus instead, with an invented party a minute into a competitive queue, for the lobby screen.

`--ui-scale=1.5` draws the interface at a forced display scale, so a 150% layout can be checked on a 100% monitor. Normally the app follows the monitor it is on.

`--demo-lock=prompt|picker|confirm` opens a stage of the insta-lock flow with no match running, so it can be inspected without queueing. It never contacts the game.

## Architecture

```
cpp/
  src/app/        Win32 window, D3D11 device, global hotkey
  src/core/       Settings, paths, logging, launch telemetry
  src/riot/       HTTP, the local client API, pd/glz, content, event socket
  src/data/       SQLite archive and the domain types
  src/services/   Live match, history, stats, loadout, updater, changelog
  src/ui/         Frame, pages, widgets, drawing primitives, theme
  vendor/         Dear ImGui, SQLite, nlohmann/json, stb_image
```

Notable pieces:

- **`riot::Session`** — opens the lockfile with `FILE_SHARE_READ | FILE_SHARE_WRITE` (Riot keeps a handle on it) and checks the recorded PID is alive, because an unclean exit leaves a stale lockfile pointing at a dead port. A handshake that keeps failing backs off rather than retrying twice a second forever.
- **`services::LiveMatchService`** — the poll loop. Publishes an update only when something visible actually changes, so the scoreboard does not rebuild every few seconds.
- **`services::PlayerStatsService`** — TTL cache in front of rank lookups. Ten players polled every three seconds would otherwise mean twenty redundant MMR requests a minute.
- **`data::Archive`** — SQLite in WAL mode, one database file per account. Riot keeps a rolling window of matches; this is what makes stats, encounter history and session tracking possible past it.
- **`ui::TextureCache`** — four worker threads, a decoded texture cache in memory and an encoded one on disk, both bounded. D3D11 resource creation is free-threaded, which is what lets art decode off the UI thread.
- **`riot::Content`** — agent art, map names and rank icons from [valorant-api.com](https://valorant-api.com), cached to disk so a cold start without a network still renders.

The UI is immediate mode: there are no widget objects, so every page is a function that draws into a rectangle it is handed. A full walkthrough — every endpoint, the auth flow, the poll state machine, the UI layer — is in [ARCHITECTURE.md](ARCHITECTURE.md).
