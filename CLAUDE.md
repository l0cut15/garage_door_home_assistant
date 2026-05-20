# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Smart garage door controller for a **Marantec Comfort 280** opener. **V2 (ESP32-C3 Super Mini) is the current shipped version.** V1 (ESP32U protoboard) is a retired prototype — its firmware is frozen.

V1 design rationale is in `garage_door_automation.md`. V2 design rationale is in `garage_door_v2_design.md`. All active work is in `garage_door_v2.yaml`.

## Versions

| Version | Status | Firmware | Description |
|---|---|---|---|
| V1 | Prototype (retired) | `garage_door.yaml` | ESP32U protoboard — impulse trigger, optimistic cover state |
| V2 | **Current** | `garage_door_v2.yaml` | ESP32-C3 Super Mini — VL53L0X ToF sensor, real open/closed state |

## V1 Architecture (retired prototype — `garage_door.yaml` is frozen)

```
Apple Home / Siri
    ↕ HomeKit (via Homebridge)
Home Assistant  ←→  cover entity (device_class: garage)
    ↕ ESPHome Native API (encrypted, local WiFi)
ESP32U (ESPHome)
    GPIO17 → 220Ω → G3VM-61A1 SSR → Marantec XB03 terminals 1+2 (GND+Pulse)
    5V pin ← MP1584EN buck converter ← Marantec XB03 terminal 3 (24V, 50mA max), GND = terminal 1
```

Reed switch inputs (GPIO4/GPIO5) are **V2 scope** — not present in V1.

## ESPHome Commands

```bash
# V1
esphome run garage_door.yaml        # compile + OTA flash
esphome logs garage_door.yaml       # stream device logs

# V2
esphome run garage_door_v2.yaml
esphome logs garage_door_v2.yaml
```

`secrets.yaml` is required and not committed. It must define: `wifi_ssid`, `wifi_password`, `esphome_api_key`, `ota_password`, `ap_fallback_password`.

## V1 ESPHome Config Facts

- **Board:** `esp32: board: esp32dev` with `framework: type: arduino`
- **GPIO17:** SSR trigger — pulse HIGH for 300ms to send an impulse to the Marantec
- Cover uses `optimistic: true` + `assumed_state: true` — this is permanent for V1, not a placeholder
- The trigger switch uses `restore_mode: ALWAYS_OFF` — it must never energise on boot
- Bluetooth proxy is enabled (`active: true`)

## Critical Hardware Constraints

- **Never connect ESP32 GPIO directly to Marantec XB03 terminals.** Any external voltage on those terminals destroys the control board. The G3VM-61A1 SSR provides mandatory galvanic isolation.
- **Do not touch terminals 70/71** — these carry the existing photocell obstacle-safety wiring.
- Buck converter output must be confirmed at 5.0V ±0.2V before connecting to the ESP32.
- All ESP32 GPIO are 3.3V — not 5V tolerant.

## V2 Architecture

Board: **ESP32-C3 Super Mini** (`esp32-c3-devkitm-1`). Sensor: **CJVL53L0XV2** (VL53L0X breakout). Buck: **Mini 5605V**.

```
Apple Home / Siri
    ↕ HomeKit (via Homebridge)
Home Assistant  ←→  cover entity (device_class: garage)
    ↕ ESPHome Native API (encrypted, local WiFi)
ESP32-C3 Super Mini (ESPHome)
    GPIO7  → 220Ω → G3VM-61A1 SSR → Marantec XB03 terminals 1+2 (GND+Pulse)
    GPIO1  (SDA) ┐
    GPIO3  (SCL) ┘← I2C → CJVL53L0XV2 (ceiling-mounted, pointing down at door top)
    3.3V pin → CJVL53L0XV2 VCC   ← MUST be 3.3V, not 5V (C3 GPIO are 3.3V only)
    5V pin ← Mini 5605V buck converter ← Marantec XB03 terminal 3 (24V, 50mA max), GND = terminal 1
```

**CJVL53L0XV2 wiring note:** Power the module from the C3 Mini's **3.3V pin**, not 5V. The module accepts both voltages, but if powered at 5V its SDA/SCL lines will be 5V logic — this will damage the C3 Mini. At 3.3V, I2C lines are directly compatible with no level shifting required.

### V2 Door State Logic

The sensor sits on the ceiling pointing at the top of the door panel.

| Distance reading | Physical meaning | Cover state |
|---|---|---|
| ≤ `open_distance_m` | Panel is up near sensor — door is open | **Open** |
| > `open_distance_m` (valid) | Panel moving — door is transitioning | Indeterminate |
| NaN sustained ≥ `nan_closed_count` × 500ms | Panel dropped away — door is closed | **Closed** |
| NaN without prior valid reading | Boot state unknown (door closed before first open) | Indeterminate |

### Closed Detection Design Decision

**Sensor range confirmed from bench test (2026-05-19):** 20mm minimum, ~1.38m stable maximum, sporadic NaN above ~1.4m, unreliable past ~1.5m.

**Real install data (2026-05-20):**

Full close-then-reopen cycle observed (door started open):

| Phase | Reading | Notes |
|---|---|---|
| Door open, stationary | ~0.67m stable | Sensor sees top panel ~67cm below ceiling |
| Closing transition | 0.67m → 0.99m → NaN over ~3s | Panel dropping away from sensor |
| Door fully closed | NaN sustained | Panel out of sensor range |
| Opening transition | NaN → 0.99m → 0.67m over ~6s | Panel returning toward sensor |
| Door open, stationary | ~0.67m stable | Same as open above |

**Conclusion:** OPEN = ~0.67m (reliable, immediate). CLOSED = sustained NaN (reliable after 3s). Implemented as Option B (sustained NaN).

Calibrated values:
- `open_distance_m: "0.75"` — ~0.704m measured + 5cm margin (sensor repositioned 2026-05-20)
- `nan_closed_count: "6"` — 6 × 500ms = 3s sustained NaN → CLOSED

**Option B — Sustained NaN for CLOSED detection:**
After `nan_closed_count` consecutive NaN readings → CLOSED. The garage defaults to CLOSED on boot (normal physical state — low risk of incorrect assumption). No `had_valid_reading` guard is needed.

### V2 Calibration

**Step 1 — Confirm sensor is detected**

Flash the firmware and check logs immediately:
```bash
esphome logs garage_door_v2.yaml
```
Look for the I2C scan output (logged once on boot due to `scan: true`):
```
[I][i2c.idf:069]: Found i2c device at address 0x29
```
If `0x29` is not listed, stop and check wiring before proceeding.

**Step 2 — Read live distance**

With the device running, open Home Assistant → Developer Tools → States and find the `Garage Door Distance` sensor entity. This updates every 500ms and is the most convenient calibration tool.

Alternatively, watch the log stream:
```
[D][sensor:094]: 'Garage Door Distance': Sending state 0.18 m
```

**Step 3 — Measure open distance**

Open the door fully. Let the reading settle (a few seconds). Note the value — this is the distance from the sensor to the top of the door panel when retracted. Set:
```yaml
open_distance_m: "<reading> + 0.05"   # add 5cm margin against vibration flicker
```

**Step 4 — Confirm closed-state reading**

Close the door fully and watch the `Garage Door Distance` sensor in HA for 10–15 seconds. It should read NaN consistently (panel has dropped below sensor range). If it returns a valid distance, the sensor range is longer than expected — reassess the mounting position.

**Step 5 — Reflash and verify**

```bash
esphome run garage_door_v2.yaml   # OTA flash with updated substitutions
```
Trigger open and close from HA and confirm the cover entity transitions correctly through Opening → Open → Closing → Closed.

### V2 Key Config Facts

- **Board:** `esp32-c3-devkitm-1`, `variant: esp32c3`, `framework: type: esp-idf`
- **GPIO7:** SSR trigger (replaces GPIO17 from V1)
- **GPIO1/GPIO3:** I2C SDA/SCL
- **VL53L0X:** address 0x29 (default), `long_range: true`, 500ms update interval
- Cover lambda replaces optimistic mode with threshold-based state
- Raw distance published to HA as `Garage Door Distance` sensor (use for calibration)

### ESP32-C3 WiFi Requirements

The C3 requires specific WiFi config to connect reliably — arduino framework and default wifi settings do not work:

- **Framework must be `esp-idf`** (not arduino) — arduino framework has known WiFi instability on C3
- **Disable WPA3 SAE** via sdkconfig: `CONFIG_ESP_WIFI_ENABLE_WPA3_SAE: "n"` — WPA3 negotiation causes connection failures on many routers
- **`power_save_mode: none`** — power saving causes frequent disconnects
- **`output_power: 8.5`** — full TX power can cause brownouts on the small board; 8.5 dBm is stable
- **`fast_connect: true`** — skips full channel scan, connects to known AP directly
- **`reboot_timeout: 15min`** — reboots if WiFi never connects (recovery from bad state)
- **2.4 GHz only** — ESP32-C3 has no 5 GHz radio; SSID must be on 2.4 GHz band

### Stop Button Behaviour (confirmed 2026-05-20)

The Marantec Comfort 280 impulse cycle is: **Open → Stop → Close → Stop → Open**. After a stop mid-travel, the next impulse **reverses direction** rather than resuming. ESPHome does not model this — if it issues an open command after a mid-travel stop, the Marantec will close instead. Do not build automations that depend on stop + resume from HA.
