# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Smart garage door controller for a **Marantec Comfort 280** opener. ESP32U (esp32dev) running ESPHome firmware, integrated into Home Assistant via the native ESPHome API, then exposed to Apple HomeKit via Homebridge.

Full design rationale and hardware details are in `garage_door_automation.md`. Read it before making any firmware or hardware decisions.

## Architecture

```
Apple Home / Siri
    ↕ HomeKit (via Homebridge)
Home Assistant  ←→  cover entity (device_class: garage)
    ↕ ESPHome Native API (encrypted, local WiFi)
ESP32U (ESPHome)
    GPIO17 → 220Ω → G3VM-61A1 SSR → Marantec XB03 terminals 1+3 (impulse)
    GPIO4  ← reed switch (door CLOSED), internal pullup, active LOW  [planned]
    GPIO5  ← reed switch (door OPEN), internal pullup, active LOW    [planned]
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

- **Board:** `esp32: board: esp32dev` with `framework: type: arduino`
- **GPIO17:** SSR trigger — pulse HIGH for 300ms to send an impulse to the Marantec
- **GPIO4:** door-closed reed switch (`INPUT_PULLUP`, `inverted: true`, 50ms debounce) — not yet wired
- **GPIO5:** door-open reed switch (`INPUT_PULLUP`, `inverted: true`, 50ms debounce) — not yet wired
- Cover uses `optimistic: true` + `assumed_state: true` until reed switches are installed
- The relay switch must use `restore_mode: ALWAYS_OFF` — it must never energise on boot
- Bluetooth proxy is enabled (`active: true`)

## Critical Hardware Constraints

- **Never connect ESP32 GPIO directly to Marantec XB03 terminals.** Any external voltage on those terminals destroys the control board. The G3VM-61A1 SSR provides mandatory galvanic isolation.
- **Do not touch terminals 70/71** — these carry the existing photocell obstacle-safety wiring.
- Buck converter output must be confirmed at 5.0V ±0.2V before connecting to the ESP32.
- All ESP32 GPIO are 3.3V — not 5V tolerant.

## Door State Logic (when reed switches are added)

| GPIO4 (CLOSED) | GPIO5 (OPEN) | State |
|---|---|---|
| LOW (active) | HIGH | Closed |
| HIGH | LOW (active) | Open |
| HIGH | HIGH | Moving (direction tracked by ESPHome from last command) |
| LOW | LOW | Error |
