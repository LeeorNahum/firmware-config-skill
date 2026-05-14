---
name: firmware-config
description: Standard embedded firmware project configuration, board environments, hardware selectors, build flags, library dependencies, and runtime provisioning. Use when working in firmware repos with PlatformIO structure; editing `platformio.ini`, `hardware/`, config headers, board selectors, version flags, provisioning storage, or local value placeholders.
metadata:
  author: Leeor Nahum
  version: "1.0.0"
---

# Firmware Config

Firmware configuration starts with hardware reality: board targets, pins, sensors, build environments, device identity, and runtime provisioning. Keep those layers explicit so the project can build, flash, debug, and ship without hidden coupling.

## Layers

Use these layers deliberately:

- project constants: device name, manufacturer, public protocol names
- hardware selectors: board pins, sensor enables, hardware revision IDs
- build version: firmware semver and build mode flags
- runtime provisioning: Wi-Fi credentials, device API key, claimed user/device state
- local placeholders or injected values: developer-only credentials, demo values, or manufacturing inputs used for testing

Do not collapse all of these into one header.

## Files

Preferred firmware repo shape:

```text
platformio.ini
hardware/
  <target-name>/
    <target-name>.ini
    <target-name>.h
src/
  main.cpp
  ble/
  hardware/
  helpers/
```

If configuration is shared across targets but not target-specific, place it as a shared header directly under `hardware/`.

Use project-specific names when already established, but preserve the separation. Add `hardware/secrets.example.ini` only when local injected values are needed.

For factual PlatformIO syntax, section names, and option behavior, check the official ["platformio.ini" Project Configuration File](https://docs.platformio.org/en/latest/projectconf/index.html) docs first.

## Hardware And Environments

- Keep `platformio.ini` as the coordinator: project metadata, optional `default_envs`, `extra_configs`, shared `[default]`, and shared `[debug]`.
- Put concrete build environments under `hardware/<target-name>/`, not directly in the main `platformio.ini`.
- Pair each concrete env `.ini` with a same-named hardware `.h`.
- Generic hardware config that is not tied to one target may live directly under `hardware/`.
- Use `[debug]` for shared debug flags and debug-only library settings.
- Create a separate env for any hardware difference: board revision, hardware variant, production vs. prototype, debug/release builds, or release targets.
- Keep reusable board or hardware selectors in `hardware/`, not scattered through app logic.
- Use `-include` board/hardware headers when that is the established repo pattern.
- Document what each environment builds, flashes, and assumes.
- Do not silently change the default upload environment if hardware could be affected.

Root `platformio.ini` pattern:

```ini
[platformio]
name = Project-Firmware
description = Short description of the firmware
default_envs = target_name ; set to the primary or production target
extra_configs =
  hardware/**/*.ini

[default]
build_flags =
  ${debug.build_flags}
lib_deps =
  ${debug.lib_deps}

[debug]
build_flags =
  -D DEBUG_ENABLE
lib_deps =
  https://github.com/LeeorNahum/SimpleDebug.git#main
```

Board/target example:

```ini
; hardware/target_name/target_name.ini
[env:target_name]
platform = https://github.com/example/platform-board-package.git#main
board = <pio-board-id>
framework = arduino
monitor_speed = 115200
build_flags =
  ${default.build_flags}
  -include hardware/target_name/target_name.h
lib_deps =
  ${default.lib_deps}
  https://github.com/example/Driver_Library.git#main
```

```cpp
// hardware/target_name/target_name.h
#ifndef PROJECT_TARGET_NAME_H
#define PROJECT_TARGET_NAME_H

#include <stdint.h>

// I2C
static const uint8_t SDA_PIN = 8;
static const uint8_t SCL_PIN = 6;

// Interrupts / alerts
static const uint8_t BUTTON_INT_PIN = 31;
static const uint8_t BATTERY_ALERT_PIN = 2;

// I2C addresses
static const uint8_t BATTERY_I2C_ADDRESS = 0x36;

// Hardware selection macros
#define DISPLAY_SSD1306
#define IMU_LSM6DS3

#endif // PROJECT_TARGET_NAME_H
```

Name the env and folder after the hardware target. Create a separate env for any meaningful hardware difference. If two targets are nearly identical, one env can inherit from the other's sections rather than duplicating build flags and lib deps.

Add suffixes only when the split exists, e.g. `target_name_dev`, `target_name_prod`, `target_name_debug`.

## Header Guards

Use classic include guards for firmware headers, including `src/**` and `hardware/**`.

Guard names should be uppercase, specific, and path-aware enough to avoid collisions:

```cpp
#ifndef HARDWARE_DISPLAY_DISPLAY_H
#define HARDWARE_DISPLAY_DISPLAY_H

// declarations

#endif // HARDWARE_DISPLAY_DISPLAY_H
```

Board headers should include the project identity in the guard:

```cpp
#ifndef PROJECT_TARGET_NAME_H
#define PROJECT_TARGET_NAME_H

// board constants and selection macros

#endif // PROJECT_TARGET_NAME_H
```

Always include the closing `#endif // GUARD_NAME` comment.

## Source Layout

- `src/main.cpp` is glue. It owns setup/loop orchestration, high-level state machines, timing, and calls into modules. It should be sleek in finished code.
- Temporary diagnostic bring-up code in `main.cpp` is acceptable, but do not treat it as the final architecture.
- `src/hardware/<domain>/` is generic reusable hardware wrapper code that could be copied to another project.
- `src/helpers/<domain>/` is project-specific behavior: provisioning flows, telemetry windows, power policy, state machines, storage policies, and service clients.
- `src/ble/<domain>/` owns BLE service/HID composition when BLE is present.
- Each domain should have a small public header and implementation files behind it.
- Use subfolders for domains. Do not let `src/helpers` or `src/hardware` become flat catch-all folders.

Wrapper example:

```text
src/hardware/display/
  display.h
  display_ssd1306.cpp
  display_virtual.cpp
```

Use board macros such as `DISPLAY_SSD1306` to select real implementations. Provide virtual implementations when they make bring-up, tests, or partial hardware useful.

Helper example:

```text
src/helpers/telemetry/
  telemetry.h
  telemetry.cpp
src/helpers/power/
  power.h
  power.cpp
```

Helpers can combine hardware wrappers, configuration, and runtime state. They should not contain reusable chip drivers or pin maps.

## Local Values And Provisioning

- Commit `hardware/secrets.example.ini` with placeholder values only when local injected values are needed.
- Ignore the real local-value file: `hardware/secrets.ini`.
- Never paste real Wi-Fi passwords, API keys, tokens, or private keys into chat, docs, screenshots, or commits.
- If real-looking secrets appear in tracked history, stop and recommend rotation before treating the project as clean.
- Prefer runtime provisioning or NVS storage for device API keys and Wi-Fi credentials when the device flow supports it.

Local value options:

| Pattern | Best for | Tradeoff |
| --- | --- | --- |
| Runtime provisioning / NVS | device credentials, Wi-Fi, per-device API keys | needs provisioning UI or manufacturing flow |
| Provisioning over Wi-Fi, BLE, serial, or app | user/device setup without recompiling firmware | requires a stable setup UX and validation |
| `hardware/secrets.ini` build flags | bring-up, demos, temporary bearer tokens | can leak in build logs or generated compile metadata |
| command-line `-D ...` | CI or one-off non-secret build switches | easy to lose, hard to reproduce, risky for credentials |

Recommendation: runtime provisioning first for shipped device credentials; `hardware/secrets.ini` only for local bring-up or temporary keys; command-line macros for non-secret build switches.

Default ignore shape:

```gitignore
.pio
.vscode
```

If `hardware/secrets.ini` or any other local secret files exist, ignore them too.

## Build Flags And Versions

- Define firmware semver through PlatformIO `build_flags` when the repo does not already have a stronger version generator.
- Headers should `#error` when required version flags are missing.
- Hardware version and firmware version are different concepts.
- Shared debug flags belong in `[debug]`.
- Env-specific flags belong in that env's `hardware/<target>/<target>.ini`.
- Use a release/versioning workflow when bumping versions or publishing artifacts.
- Always use the spaced `-D FLAG` form rather than `-DFLAG`. PlatformIO accepts both; the spaced form reads clearly and stays consistent across all flags.

Version flag pattern:

```ini
build_flags =
  -D FIRMWARE_VERSION_MAJOR=1
  -D FIRMWARE_VERSION_MINOR=3
  -D FIRMWARE_VERSION_PATCH=0
```

```cpp
#if !defined(FIRMWARE_VERSION_MAJOR) || !defined(FIRMWARE_VERSION_MINOR) || !defined(FIRMWARE_VERSION_PATCH)
#error "FIRMWARE_VERSION_MAJOR, FIRMWARE_VERSION_MINOR, and FIRMWARE_VERSION_PATCH must be defined via build flags."
#endif
```

## Library Dependencies

- Use GitHub URLs for all firmware library dependencies, including third-party libraries that also exist in the PlatformIO registry.
- Do not use bare PlatformIO registry names in new work.
- Default to the library repo's active top branch, usually `#main` or `#master`, so active firmware work stays current.
- If a repo already intentionally pins a tag, version branch, or commit, preserve that pin unless the task is to update dependencies.
- Keep project-owned libraries under GitHub URLs so agents can inspect source and behavior.

## Runtime Provisioning

Device credentials that users or manufacturing flows provide at runtime should live in persistent storage such as NVS, not compile-time headers.

Provisioning may happen through Wi-Fi captive portal, BLE, serial, a desktop/mobile app, or a manufacturing flow. Use compile-time placeholders only for local development, demos, or bring-up when runtime provisioning does not exist yet.

## Firmware Config Audit

When invoked:

1. Inspect `platformio.ini`, `hardware/`, `src/`, `.gitignore`, and docs.
2. Identify project constants, hardware selectors, version flags, build environments, runtime provisioning, and local-only values.
3. Confirm real local-value files are ignored and examples exist when needed.
4. Check for hardcoded credentials, board assumptions, or service endpoints that should be configurable.
5. Verify board environments, library dependency refs, and build flags are documented.
6. Report risks by key/file category, never by secret value.

## Stop And Ask

Ask before:

- changing board pin maps or hardware revision IDs
- changing upload targets or default PlatformIO environments
- replacing runtime provisioning with compile-time secrets
- rotating or moving device API keys
- editing recovery or bootloader behavior without hardware access
