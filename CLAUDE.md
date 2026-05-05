# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Smart garage door controller for a **Marantec Comfort 280** opener. ESP32-C3 Mini running ESPHome firmware, integrated into Home Assistant via the native ESPHome API, then exposed to Apple HomeKit via HA's built-in HomeKit Bridge integration.

Full design rationale and hardware details are in `garage_door_automation.md`. Read it before making any firmware or hardware decisions.

## Architecture

```
Apple Home / Siri
    ↕ HomeKit Accessory Protocol (local, via HA HomeKit Bridge)
Home Assistant  ←→  cover entity (device_class: garage)
    ↕ ESPHome Native API (encrypted, local WiFi)
ESP32-C3 Mini (ESPHome)
    GPIO10 → 220Ω → G3VM-61A1 SSR → Marantec XB03 terminals 1+3 (impulse)
    GPIO4  ← reed switch (door CLOSED), internal pullup, active LOW
    GPIO5  ← reed switch (door OPEN), internal pullup, active LOW
    5V pin ← MP1584EN buck converter ← Marantec XB03 terminal 2 (24V, 50mA max)
```

## ESPHome Commands

```bash
esphome run garage_door.yaml        # compile + OTA flash
esphome logs garage_door.yaml       # stream device logs
esphome compile garage_door.yaml    # compile only
```

`secrets.yaml` is required and not committed. It must define: `wifi_ssid`, `wifi_password`, `esphome_api_key`, `ota_password`.

## Key ESPHome Config Facts

- **Dev board:** `esp32: board: esp32dev` with `framework: type: arduino` — ESP32U, GPIO23 for SSR trigger
- **Final board:** `esp32: board: lolin_c3_mini` with `framework: type: arduino` — GPIO10 for SSR trigger
- GPIO23: SSR trigger on dev board; GPIO10 on final C3 Mini — pulse LOW for ~300ms to send an impulse to the Marantec
- GPIO4: door-closed reed switch (`INPUT_PULLUP`, `inverted: true`, 50ms debounce)
- GPIO5: door-open reed switch (`INPUT_PULLUP`, `inverted: true`, 50ms debounce)
- The `cover` component uses `platform: template` with `device_class: garage`
- The relay switch must use `restore_mode: ALWAYS_OFF` — it must never energise on boot

## Critical Hardware Constraints

- **Never connect ESP32 GPIO directly to Marantec XB03 terminals.** Any external voltage on those terminals destroys the control board. The G3VM-61A1 SSR provides mandatory galvanic isolation.
- **Do not touch terminals 70/71** — these carry the existing photocell obstacle-safety wiring.
- Buck converter output must be confirmed at 5.0V ±0.2V before connecting to the ESP32.
- The ESP32-C3 Mini is strictly 3.3V on all GPIO — unlike the D1 Mini, no pins are 5V tolerant.

## Door State Logic

| GPIO4 (CLOSED) | GPIO5 (OPEN) | State |
|---|---|---|
| LOW (active) | HIGH | Closed |
| HIGH | LOW (active) | Open |
| HIGH | HIGH | Moving (direction tracked by ESPHome from last command) |
| LOW | LOW | Error |

## Current Project Status

The ESPHome YAML (`garage_door.yaml`) does not yet exist — it is the primary deliverable to be generated. The design document (`garage_door_automation.md`) is the specification.
