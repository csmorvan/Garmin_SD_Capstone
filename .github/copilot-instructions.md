## Quick orientation for AI coding agents

This repository is a ConnectIQ (Monkey C) watch app that collects accelerometer, heart-rate and O2 data on a Garmin device and forwards it to a phone-side server (OpenSeizureDetector). The codebase is small and follows ConnectIQ / Toybox conventions.

Key high-level facts
- Language / platform: Monkey C (Garmin ConnectIQ). Look under `source/*.mc` for the app logic.
- App flow: data collection -> JSON payload -> `Toybox.Communications.makeWebRequest` -> phone (via Garmin Connect mobile) -> phone returns alarm state -> watch updates UI and signals user.
- Primary entry points: `source/GarminSDApp.mc` (AppBase), `source/GarminSDView.mc` (UI), `source/GarminSDDataHandler.mc` (sensor collection & timing), `source/GarminSDComms.mc` (network/Comm callbacks), `source/GarminSDState.mc` (app state).

Build / run notes (concrete)
- SDK: requires Garmin ConnectIQ SDK v6.4.2 or higher (see `README.md`).
- CLI build: set MB_HOME to the SDK install and MB_PRIVATE_KEY to your developer key, then run from project root:

  ./mb_runner.sh build

- VS Code: use the Monkey C extension. Copy `monkey.jungle.template` -> `monkey.jungle` and `manifest.xml.template` -> `manifest.xml` and configure the extension to point to the SDK. Use Run & Debug targeting the VenuSQ emulator (the repo references VenuSQ as the reference device).

Project-specific conventions and patterns
- Communications are asynchronous and callback-driven. Example pattern in `source/GarminSDComms.mc`:

  Comm.makeWebRequest(serverUrl + "/data", { "dataObj" => dataObj }, {...}, method(:onDataReceive))

  Callback handlers expect typed responses (often a Dictionary). Always follow existing checks: if responseCode == 200 then assert `data instanceof Dictionary` before reading keys.

- Status and UI updates: handlers set `mAccelHandler.mStatusStr` and `needs_update` flags. UI refreshes are driven from the view code reading those values (see `GarminSDView.mc`). Preserve this pattern when changing status logic.

- Settings and persistence: uses `Toybox.Application.Storage.getValue(...)` / `Storage.getValue(MENUITEM_SOUND)` for toggles. Follow the same boolean casting pattern used across the codebase.

- Error signaling: uses `Toybox.Attention` methods (playTone, vibrate, backlight) guarded with `Attention has :playTone` checks — replicate guards when calling device features.

Important files and what they show (use these as examples)
- `source/GarminSDComms.mc` — the communications lifecycle (sendAccelData, onDataReceive, onSdStatusReceive), timeout logic (`onTick`) and defensive response handling.
- `source/GarminSDDataHandler.mc` — how sensor data is sampled, buffered and converted to JSON (useful when changing payload format).
- `source/GarminSDView.mc` — UI rendering and how it reads `mStatusStr` and other state fields.
- `mb_runner.sh`, `mb_runner.cfg`, `monkey.jungle.template`, `manifest.xml` — build/run configuration. Changes here affect emulator/device targets and SDK versions.

Integration points & external expectations
- The watch app expects a phone-side HTTP endpoint (default `http://127.0.0.1:8080` in `GarminSDComms.mc`) reachable via the Garmin Connect mobile app; do not assume direct Internet access from the watch.
- The JSON payload keys are consumed by the OpenSeizureDetector phone app; changing payload keys requires coordinating with that server code.

Quick editing rules for agents
- Preserve public API behavior: keep same JSON keys and response semantics unless a coordinated change is done across phone + watch.
- Keep Toybox API usage patterns (guard `Attention has :X`, check `instanceof Dictionary` for responses).
- When adding new device features, guard with `has :feature` checks and fall back gracefully (no exception) on devices that don't support the feature.

If you change build configs
- Update `README.md` build section and `mb_runner.cfg` defaults. Document required SDK versions and how to set `MB_HOME` and `MB_PRIVATE_KEY`.

Where to look first when debugging
- Logs: create or inspect `GARMIN/APPS/LOGS/GarminSD.Log` on the device/emulator for runtime logs (the README documents this). Search for `writeLog(` in the source to find log calls.
- Comm failures: check `source/GarminSDComms.mc` for responseCode handling and `lastOnReceiveResponse`/`lastOnSdStatusReceiveResponse` state.

If anything here is unclear or you want more examples (small refactor, tests, or adding a CI build), tell me which area to expand and I will update this file.
