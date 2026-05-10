# Breadboard Wiring — Garage Door Controller Prototype

**Board:** ESP32U dev board (ESP32 with U.FL antenna)  
**Power:** USB from bench supply for ESP logic; MP1584EN buck on breadboard for standalone buck test  
**Isolation:** OMRON G3VM-61A1 MOSFET SSR (through-hole / DIP-4)  
**Sensors:** Two NO reed switches with jumper wires

---

## Pin Assignments

| Signal | GPIO | Notes |
|---|---|---|
| SSR trigger | GPIO17 | General purpose output, safe on ESP32; final C3 Mini uses GPIO10 |
| Reed switch — CLOSED | GPIO4 | Matches final build GPIO |
| Reed switch — OPEN | GPIO5 | Matches final build GPIO |
| Power in | 5V | From USB via dev board onboard regulator |
| Ground | GND | — |

> **Dev board vs final:** The final ESP32-C3 Mini uses GPIO10 for the trigger. GPIO17 is used here because it is a clean general-purpose output on the ESP32U dev board. Update the ESPHome YAML `pin:` to GPIO10 when migrating to the C3 Mini.

---

## Components

| Item | Qty | Notes |
|---|---|---|
| ESP32U dev board | 1 | USB-powered from bench supply |
| OMRON G3VM-61A1 | 1 | Through-hole / DIP-4 package |
| Resistor, 220Ω | 1 | Sets SSR LED drive current to ~10mA |
| Reed switch (NO type) | 2 | Normally-open magnetic contact sensors |
| MP1584EN buck module | 1 | Trimmer pre-set to 5V before bench test |

| Breadboard | 1 | Half-size is sufficient |
| Jumper wires | ~10 | Male-to-male for breadboard; male-to-bare for Marantec terminals |
| USB cable | 1 | Power and serial flash |

---

## G3VM-61A1 Pin Reference (DIP-4)

```
        ┌───────┐
 Pin 1 ─┤ anode │
 Pin 2 ─┤cathode│  ← Input (LED) side
        │       │
 Pin 3 ─┤       │
 Pin 4 ─┤       │  ← Output (MOSFET) side
        └───────┘
```

| Pin | Function | Connection |
|---|---|---|
| 1 | Input anode (LED +) | 220Ω resistor → D5 (GPIO14) |
| 2 | Input cathode (LED −) | GND |
| 3 | Output | Marantec XB03 Terminal 1 (GND) |
| 4 | Output | Marantec XB03 Terminal 2 (Pulse) |

Pins 3 and 4 are the MOSFET drain/source — polarity does not matter for the output side.

---

## Wiring

### 0 — Buck converter test

Wire the MP1584EN as a standalone sub-circuit on the breadboard. The ESP32 is **not** connected during the buck test — power the ESP via USB as normal.

```
Bench supply 24V (or 12V) ──── MP1584EN IN+
Bench supply GND          ──── MP1584EN IN−

MP1584EN OUT+ ──── [test point / meter probe]
MP1584EN OUT− ──── GND rail
```

> **Pre-test:** Module output is preset to 5V — no trimmer adjustment needed. Power up and confirm with a multimeter before connecting the ESP.

---

### 1 — SSR input drive

```
ESP32U GPIO17 ────  220Ω  ────  G3VM Pin 1 (anode)
ESP32U GND    ─────────────────  G3VM Pin 2 (cathode)
```

The 220Ω resistor limits LED current to ~10mA: (3.3V − 1.2V) / 220Ω ≈ 9.5mA. This is within the G3VM's 5–50mA control range.

### 2 — SSR output → Marantec XB03

```
G3VM Pin 3  ─────────────────────────  Marantec XB03 Terminal 1 (GND)
G3VM Pin 4  ─────────────────────────  Marantec XB03 Terminal 2 (Pulse)
```

When the input LED is energised, the MOSFET output closes, momentarily shorting Terminal 2 (Pulse) to Terminal 1 (GND) — identical to pressing the wall button. No voltage crosses from the ESP side to the Marantec side.

### 3 — Reed switch, CLOSED sensor

```
ESP32U GPIO4 ────────────────────────  Reed switch terminal A
ESP32U GND   ────────────────────────  Reed switch terminal B
```

Internal pullup keeps GPIO4 HIGH when the switch is open. When the magnet closes the switch, GPIO4 is pulled LOW → door state = CLOSED.

### 4 — Reed switch, OPEN sensor

```
ESP32U GPIO5 ────────────────────────  Reed switch terminal A
ESP32U GND   ────────────────────────  Reed switch terminal B
```

Same logic — GPIO5 pulled LOW when magnet closes switch → door state = OPEN.

---

## Connection Diagram

```
                   ESP32U
                 ┌──────────┐
           EN  ──┤          ├── GPIO17 ── 220Ω ── G3VM Pin 1 (anode)
          3V3 ──┤          ├── GND  ─────────────  G3VM Pin 2 (cathode)
          GND ──┤          ├── GPIO5  ───────────  Reed OPEN  ──── GND
           5V ──┤          ├── GPIO4  ───────────  Reed CLOSED ─── GND
                 └──────────┘
                     │
                    USB (bench supply)

  G3VM Pin 3 ──────  Marantec XB03 Terminal 1 (GND)
  G3VM Pin 4 ──────  Marantec XB03 Terminal 2 (Pulse)

  Reed CLOSED: one lead → GPIO4, other lead → GND
  Reed OPEN:   one lead → GPIO5, other lead → GND
```

---

## ESPHome YAML — Prototype Differences

The dev board YAML differs from the final build in two places only:

```yaml
# Dev board — replace the esp32-c3 block with:
esp32:
  board: esp32dev
  framework:
    type: arduino

# Dev board — trigger pin is GPIO17, not GPIO10:
switch:
  - platform: gpio
    pin: GPIO17          # ESP32U dev board; final C3 Mini uses GPIO10
    id: garage_relay
    restore_mode: ALWAYS_OFF
```

Everything else (reed switch pins GPIO4/GPIO5, cover config, WiFi, API, OTA) is identical between dev board and final.

---

## Bench Test Procedure (without Marantec connected)

### Step 0 — Buck converter test (before any ESP connection)

1. Wire MP1584EN IN+/IN− to bench supply set to 24V (or 12V).
2. Do **not** connect the ESP yet. Power up and measure OUT+ to GND — should read **5.0V ±0.1V** (preset, no adjustment needed).
3. Power off. Buck sub-circuit is verified.

### Step 1 — Firmware and SSR

8. Flash ESPHome firmware over USB. Confirm the device appears in Home Assistant.
9. With a multimeter set to continuity/resistance, probe across **G3VM pins 3 and 4**.
   - At rest: open circuit.
   - Trigger the SSR from HA (or Home Assistant dev tools): output should close (beep / near-0Ω) for ~300ms then release.

### Step 2 — Reed switches

10. Touch a piece of wire across the two leads of each reed switch in turn.
    - Shorting the CLOSED reed: HA cover entity should report **Closed**.
    - Shorting the OPEN reed: HA cover entity should report **Open**.
    - Both open: entity should show **Opening** or **Closing** (depending on last command).

### Step 3 — Connect to Marantec

11. Only after all above steps pass, connect G3VM pins 3 and 4 to Marantec XB03 terminals 1 and 2.
