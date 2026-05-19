# Garage Door Controller — Marantec Comfort 280

ESP32-based smart garage door controller for the **Marantec Comfort 280** opener. Runs ESPHome firmware, integrates with Home Assistant, and is exposed to Apple HomeKit via Homebridge.

Fully local — no cloud dependency. Self-powered from the opener's own 24V DC auxiliary rail.

---

## Versions

| Version | Status | Board | State detection |
|---|---|---|---|
| V1 | Complete | ESP32U (esp32dev) | Optimistic (no sensor) |
| V2 | In progress | ESP32-C3 Super Mini | VL53L0X ToF — real open/closed/moving |

---

## Features

- Open / close / stop from Home Assistant dashboard or Apple Home
- Appears as a native garage door accessory in HomeKit
- Bluetooth proxy — extends Home Assistant Bluetooth range
- Galvanically isolated trigger (G3VM-61A1 MOSFET SSR)
- Self-powered from Marantec 24V rail via buck converter
- **V2:** Real-time door state via VL53L0X time-of-flight sensor (no door wiring required)

---

## Hardware

### V1

| Component | Part | Notes |
|---|---|---|
| Microcontroller | ESP32U (esp32dev) | Soldered protoboard build |
| Solid State Relay | OMRON G3VM-61A1 | SOP-4; galvanic isolation |
| Current limit resistor | 220Ω | SSR LED drive ~10mA |
| Buck converter | MP1584EN (preset 5V) | 24V → 5V |

### V2

| Component | Part | Notes |
|---|---|---|
| Microcontroller | ESP32-C3 Super Mini (`esp32-c3-devkitm-1`) | esp-idf framework |
| Solid State Relay | OMRON G3VM-61A1 | SOP-4; galvanic isolation |
| Current limit resistor | 220Ω | SSR LED drive ~10mA |
| Buck converter | Mini 5605V (preset 5V) | 24V → 5V |
| Distance sensor | CJVL53L0XV2 (VL53L0X breakout) | I2C, 3.3V, ceiling-mounted |

---

## Architecture (V2)

```
Apple Home / Siri
    ↕  HomeKit (Homebridge)
Home Assistant
    ↕  ESPHome Native API (encrypted, local WiFi)
ESP32-C3 Super Mini
    GPIO7  → 220Ω → G3VM-61A1 → Marantec XB03 terminals 1+2  (impulse trigger)
    GPIO1 (SDA) / GPIO3 (SCL) → I2C → CJVL53L0XV2 ToF sensor (ceiling, horizontal mount)
    5V  ← Mini 5605V buck converter ← Marantec XB03 terminal 3 (24V)
```

---

## Wiring

See [`breadboard.md`](breadboard.md) for full wiring diagrams.

### Marantec XB03 Terminal Block

```
[ 3 ][ 1 ][ 2 ][ 4 ][ 70 ][ 71 ]
```

| Terminal | Function | Connection |
|---|---|---|
| 1 | GND | G3VM pin 3 or 4; buck converter IN− |
| 2 | Pulse input | G3VM pin 3 or 4 |
| 3 | 24V DC output | Buck converter IN+ |
| 70/71 | Photocell safety | Do not touch |

---

## Firmware

### Prerequisites

- [ESPHome](https://esphome.io) installed
- `secrets.yaml` in project root (see `secrets.example.yaml`)

### secrets.yaml

```yaml
wifi_ssid: "your_ssid"
wifi_password: "your_password"
ap_fallback_password: "your_fallback_password"
esphome_api_key: "your_base64_api_key"
ota_password: "your_ota_password"
```

Generate the API key:
```bash
python3 -c "import secrets, base64; print(base64.b64encode(secrets.token_bytes(32)).decode())"
```

### Flash

```bash
# V1
esphome run garage_door.yaml

# V2 (active development)
esphome run garage_door_v2.yaml
```

Subsequent updates are OTA over WiFi.

---

## HomeKit via Homebridge

The [Homebridge addon](https://github.com/homebridge/homebridge-homeassistant) runs inside Home Assistant.

1. Install the **Homebridge** addon from the Home Assistant addon store
2. Start the addon and open the Homebridge UI from the HA sidebar
3. Open the **Home** app on iPhone and add an accessory
4. Scan the QR code displayed in the Homebridge UI

---

## Repository Structure

```
garage_door.yaml           # V1 ESPHome firmware (complete)
garage_door_v2.yaml        # V2 ESPHome firmware (active)
garage_door_automation.md  # V1 design document
garage_door_v2_design.md   # V2 design document
breadboard.md              # Wiring guide
secrets.example.yaml       # Secrets template
Board/                     # Fritzing layout files
Case/                      # 3D print files (base + lid STL)
```

---

## Roadmap

- [x] V1: impulse trigger via SSR
- [x] V1: optimistic cover state in HA/HomeKit
- [x] V2: board migrated to ESP32-C3 Super Mini
- [x] V2: VL53L0X ToF sensor — real closed/open state detection
- [x] V2: sustained-NaN open detection (Option B) implemented
- [ ] V2: calibrate thresholds in final installed position
- [ ] V2: install in garage and verify end-to-end
- [ ] Mount in 3D printed enclosure
