# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Smart garage door controller for a **Marantec Comfort 280** opener. ESP32U (esp32dev) running ESPHome firmware, integrated into Home Assistant via the native ESPHome API, then exposed to Apple HomeKit via Homebridge.

Full design rationale and hardware details are in `garage_door_automation.md`. Read it before making any firmware or hardware decisions.

## Versions

| Version | Status | Firmware | Description |
|---|---|---|---|
| V1 | **Complete** | `garage_door.yaml` | ESP32U protoboard — impulse trigger, optimistic cover state |
| V2 | In progress | `garage_door_v2.yaml` | Adds VL53L0X ToF sensor for Closed / Open / Moving state |

## V1 Architecture (complete — do not modify `garage_door.yaml` for V1 features)

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

Cover state is derived from VL53L0X distance thresholds (substitutions in `garage_door_v2.yaml`):

| Distance reading | State |
|---|---|
| ≤ `closed_distance_m` | Closed |
| ≥ `open_distance_m` | Open |
| Between thresholds | Indeterminate — ESPHome shows Opening/Closing during action, else last known state |
| NaN / 0 / no reading | Indeterminate (sensor not ready or out of range) |

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

**Step 3 — Measure closed distance**

Close the door fully. Let the reading settle (a few seconds). Note the value, then set:
```yaml
closed_distance_m: "<reading> + 0.05"   # add 5cm margin against vibration flicker
```

**Step 4 — Confirm open threshold**

Open the door fully. Confirm the reading is well above your intended `open_distance_m` (0.5–0.8m is a safe default). The sensor does not need to see the door when open — any reading above the threshold declares it open.

**Step 5 — Measure travel time**

Time the door from fully closed to fully open (and back). Set:
```yaml
travel_time_s: "<seconds>s"
```
Add 2–3 seconds margin so the timeout only fires if the door genuinely stalls.

**Step 6 — Reflash and verify**

```bash
esphome run garage_door_v2.yaml   # OTA flash with updated substitutions
```
Trigger open and close from HA and confirm the cover entity transitions correctly through Opening → Open → Closing → Closed.

**Step 7 — Test stop behaviour**

Trigger open, then immediately send stop mid-travel. Observe what the Marantec does on the next open/close command — confirm whether it resumes or reverses. Update the stop caveat note once confirmed.

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

### Stop Button Caveat (unconfirmed — test before relying on it)

The Marantec Comfort 280 impulse cycle is: **Open → Stop → Close → Stop → Open**. After a stop mid-travel, the next impulse **reverses direction** rather than resuming. ESPHome does not model this — if it issues an open command after a mid-travel stop, the Marantec may close instead. Verify physical behaviour before building any automations that depend on stop + resume.
