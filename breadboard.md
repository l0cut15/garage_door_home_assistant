# Breadboard Wiring — Garage Door Controller Prototype

**Board:** Wemos D1 Mini (ESP8266)  
**Power:** USB from bench supply (no buck converter required for prototype)  
**Isolation:** OMRON G3VM-61A1 MOSFET SSR (through-hole / DIP-4)  
**Sensors:** Two NO reed switches with jumper wires

---

## Pin Assignments

| Signal | D1 Mini Label | GPIO | Notes |
|---|---|---|---|
| SSR trigger | D5 | GPIO14 | Safe GPIO — GPIO10 (SD3) is the flash interface on ESP8266 and must not be used |
| Reed switch — CLOSED | D2 | GPIO4 | Matches final build GPIO |
| Reed switch — OPEN | D1 | GPIO5 | Matches final build GPIO |
| Power in | 5V | — | From USB via D1 Mini onboard regulator |
| Ground | GND | — | — |

> **Prototype vs final:** The final ESP32-C3 Mini uses GPIO10 for the trigger. Because GPIO10 is SD3 on the ESP8266, the prototype uses GPIO14 (D5) instead. Update the ESPHome YAML `pin:` accordingly when migrating.

---

## Components

| Item | Qty | Notes |
|---|---|---|
| Wemos D1 Mini | 1 | USB-powered from bench supply |
| OMRON G3VM-61A1 | 1 | Through-hole / DIP-4 package |
| Resistor, 220Ω | 1 | Sets SSR LED drive current to ~10mA |
| Reed switch (NO type) | 2 | Normally-open magnetic contact sensors |
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
| 3 | Output | Marantec XB03 Terminal 1 (Impulse) |
| 4 | Output | Marantec XB03 Terminal 3 (GND) |

Pins 3 and 4 are the MOSFET drain/source — polarity does not matter for the output side.

---

## Wiring

### 1 — SSR input drive

```
D1 Mini D5 (GPIO14) ────  220Ω  ────  G3VM Pin 1 (anode)
D1 Mini GND         ─────────────────  G3VM Pin 2 (cathode)
```

The 220Ω resistor limits LED current to ~10mA: (3.3V − 1.2V) / 220Ω ≈ 9.5mA. This is within the G3VM's 5–50mA control range.

### 2 — SSR output → Marantec XB03

```
G3VM Pin 3  ─────────────────────────  Marantec XB03 Terminal 1 (Impulse)
G3VM Pin 4  ─────────────────────────  Marantec XB03 Terminal 3 (GND)
```

When the input LED is energised, the MOSFET output closes, momentarily shorting Terminal 1 to Terminal 3 — identical to pressing the wall button. No voltage crosses from the ESP side to the Marantec side.

### 3 — Reed switch, CLOSED sensor

```
D1 Mini D2 (GPIO4) ──────────────────  Reed switch terminal A
D1 Mini GND ─────────────────────────  Reed switch terminal B
```

Internal pullup keeps D2 HIGH when the switch is open. When the magnet closes the switch, D2 is pulled LOW → door state = CLOSED.

### 4 — Reed switch, OPEN sensor

```
D1 Mini D1 (GPIO5) ──────────────────  Reed switch terminal A
D1 Mini GND ─────────────────────────  Reed switch terminal B
```

Same logic — D1 pulled LOW when magnet closes switch → door state = OPEN.

---

## Connection Diagram

```
                    D1 MINI
                 ┌──────────┐
           RST ──┤          ├── TX
            A0 ──┤          ├── RX
   [unused] D0 ──┤          ├── D1 (GPIO5) ────── Reed OPEN ──── GND
      [SSR] D5 ──┤          ├── D2 (GPIO4) ────── Reed CLOSED ── GND
   [unused] D6 ──┤          ├── D3
   [unused] D7 ──┤          ├── D4
   [unused] D8 ──┤          ├── GND ─────────────────────────── GND rail
           3V3 ──┤          ├── 5V
                 └──────────┘
                     │
                    USB (bench supply)

  D5 ──── 220Ω ──── G3VM Pin 1 (anode)
  GND ─────────────  G3VM Pin 2 (cathode)

  G3VM Pin 3 ──────  Marantec XB03 Terminal 1 (Impulse)
  G3VM Pin 4 ──────  Marantec XB03 Terminal 3 (GND)

  Reed CLOSED: one lead → D2, other lead → GND
  Reed OPEN:   one lead → D1, other lead → GND
```

---

## ESPHome YAML — Prototype Differences

The prototype YAML differs from the final build in two places only:

```yaml
# Prototype — replace the esp32 block with:
esp8266:
  board: d1_mini

# Prototype — trigger pin is GPIO14, not GPIO10:
switch:
  - platform: gpio
    pin: GPIO14          # D5 on D1 Mini; final build uses GPIO10
    id: garage_relay
    restore_mode: ALWAYS_OFF
```

Everything else (reed switch pins, cover config, WiFi, API, OTA) is identical between prototype and final.

---

## Bench Test Procedure (without Marantec connected)

1. Flash ESPHome firmware over USB. Confirm the device appears in Home Assistant.
2. With a multimeter set to continuity/resistance, probe across **G3VM pins 3 and 4**.
   - At rest: open circuit.
   - Trigger the SSR from HA (or Home Assistant dev tools): output should close (beep / near-0Ω) for ~300ms then release.
3. Touch a piece of wire across the two leads of each reed switch in turn.
   - Shorting the CLOSED reed: HA cover entity should report **Closed**.
   - Shorting the OPEN reed: HA cover entity should report **Open**.
   - Both open: entity should show **Opening** or **Closing** (depending on last command).
4. Only after bench test passes, connect G3VM pins 3 and 4 to Marantec XB03 terminals 1 and 3.
