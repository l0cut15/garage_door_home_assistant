# Garage Door Automation — Design Document (V1)

> **This is the V1 design document.** V1 is complete and uses an optimistic cover with no sensor.  
> V2 supersedes sections 3.3 (state sensing), 5 (power — different buck converter), and 7 (ESPHome config).  
> For V2 design rationale see [`garage_door_v2_design.md`](garage_door_v2_design.md).  
> For current firmware facts see [`CLAUDE.md`](CLAUDE.md).

**Project:** Smart Garage Door Controller  
**Platform:** ESP32-C3 Mini (prototype: D1 Mini ESP8266)  
**Opener:** Marantec Comfort 280  
**Integration:** ESPHome → Home Assistant → HomeKit Bridge  
**Date:** May 2026  

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Architecture Stack](#2-architecture-stack)
3. [Hardware Design](#3-hardware-design)
4. [Wiring Diagram](#4-wiring-diagram)
5. [Power Design](#5-power-design)
6. [State Sensing](#6-state-sensing)
7. [ESPHome Configuration Notes](#7-esphome-configuration-notes)
8. [Prototype vs Final Build](#8-prototype-vs-final-build)
9. [Bill of Materials](#9-bill-of-materials)
10. [Safety Notes](#10-safety-notes)

---

## 1. Project Overview

Remote control of a Marantec Comfort 280 garage door opener via Home Assistant and Apple HomeKit, using a WiFi-connected ESP32-C3 Mini running ESPHome firmware. The solution is fully local — no cloud dependency — and self-powered from the opener's own 24V DC auxiliary rail.

### Goals

- Remote open/close via Home Assistant dashboard and Apple Home app
- Real-time door state feedback (open / closed / opening / closing)
- Siri and HomeKit automation support
- No separate power supply — powered from the Marantec 24V rail
- Clean PCB-based build, prototyped first on D1 Mini

### Out of Scope

- Video monitoring
- Automatic closing timers (can be added in HA automations later)
- Matter protocol (garage door device type not yet in Matter spec as of 2026)

---

## 2. Architecture Stack

```
┌─────────────────────────────────────────────────────────┐
│                    Apple Home / Siri                    │
└───────────────────────────┬─────────────────────────────┘
                            │ HomeKit Accessory Protocol (HAP)
                            │ via HA HomeKit Bridge (local)
┌───────────────────────────▼─────────────────────────────┐
│                   Home Assistant                        │
│         (HomeKit Bridge integration built-in)           │
│         (cover entity: device_class: garage)            │
└───────────────────────────┬─────────────────────────────┘
                            │ ESPHome Native API
                            │ (encrypted, local WiFi)
┌───────────────────────────▼─────────────────────────────┐
│              ESP32-C3 Mini (ESPHome firmware)           │
│   - Relay/SSR trigger output (GPIO)                     │
│   - Reed switch CLOSED input (GPIO)                     │
│   - Reed switch OPEN input (GPIO)                       │
└───────────────────────────┬─────────────────────────────┘
                            │ Isolated dry contact
┌───────────────────────────▼─────────────────────────────┐
│          Marantec Comfort 280 — Terminal XB03           │
│   Terminal 1 (GND) + Terminal 2 (Pulse)                 │
└─────────────────────────────────────────────────────────┘
```

### Why This Architecture

| Approach | Verdict | Reason |
|---|---|---|
| ESPHome + HA HomeKit Bridge | **Chosen** | Single device, dual platform, fully local |
| HomeSpan (HomeKit direct) | Rejected | HomeKit only, no HA integration |
| ESPHome + MQTT + Homebridge | Rejected | Extra complexity, two bridges |
| Matter | Not yet viable | Garage door type not in Matter spec |
| Dual firmware stacks on one ESP | Rejected | Unstable, unsupported |

---

## 3. Hardware Design

### 3.1 Marantec Comfort 280 — Terminal Block XB03

The Marantec exposes all external control connections on terminal block **XB03**, physically labelled on the unit as:

```
[ 3 ][ 1 ][ 2 ][ 4 ][ 70 ][ 71 ]
```

| Terminal | Function | Notes |
|---|---|---|
| 3 | 24V DC output | Max 50mA — used to power the ESP32 |
| 1 | GND | Common ground reference |
| 2 | Pulse input | Trigger input — momentary dry contact to GND opens/stops/closes |
| 4 | Hold circuit | Normally closed input, active after reset |
| 70 | GND (photocell) | Photocell ground — already wired, do not disturb |
| 71 | Photocell input | Obstacle sensor — already wired, do not disturb |

> **WARNING:** The Marantec manual explicitly states that applying any external voltage to XB03 terminals will irreparably destroy the control electronics. Only potential-free (isolated) contacts may be connected to terminals 1, 2, and 4. Never connect ESP32 GPIO directly to these terminals.

### 3.2 Trigger Interface — OMRON G3VM-61A1 SSR

A MOSFET solid-state relay provides full galvanic isolation between the ESP32 GPIO and the Marantec terminals. This is preferable to a mechanical relay for this application — silent, no moving parts, no contact bounce, and the isolation is essential given the Marantec's sensitivity to external voltages.

**G3VM-61A1 Specifications (relevant):**

| Parameter | Value |
|---|---|
| Package | SOP-4 (SMD) |
| Input | LED, 1.2V forward voltage |
| Control current | 5–50mA |
| Output voltage | 60V max |
| Output current | 500mA max |
| On-resistance | ~1Ω |
| Isolation | Full optical isolation |

**Input drive circuit:**

The ESP32-C3 GPIO outputs 3.3V. A current-limiting resistor sets LED current:

```
R = (Vgpio - Vf) / I
R = (3.3V - 1.2V) / 10mA
R = 210Ω → use 220Ω (standard value)
```

**GPIO → 220Ω → G3VM Pin 1 (anode)**  
**G3VM Pin 2 (cathode) → GND**  
**G3VM Pin 3/4 (output) → across Marantec terminals 1 and 2**

The output pins short together when the input LED is energised, momentarily bridging terminal 2 (Pulse) to terminal 1 (GND) — identical to pressing the wall button.

### 3.3 Door State Sensing — Reed Switches

Two magnetic reed switches provide full door state awareness. This enables the ESPHome `cover` entity to report proper HomeKit states: Closed / Opening / Open / Closing / Stopped.

| Switch | Mount Location | State when active |
|---|---|---|
| CLOSED sensor | Door frame, bottom — activates when door fully closed | GPIO pulled LOW |
| OPEN sensor | Door track, top — activates when door fully open | GPIO pulled LOW |

Use surface-mount alarm-type reed switches (NO type). Mount the magnet on the moving door panel, the switch body on the fixed frame/track.

ESP32 internal pull-ups handle these inputs — no external resistors needed. Configured as `pullup: true` in ESPHome.

**Door state logic:**

| CLOSED pin | OPEN pin | State |
|---|---|---|
| LOW (active) | HIGH | Closed |
| HIGH | LOW (active) | Open |
| HIGH | HIGH | Moving (opening or closing) |
| LOW | LOW | Error / misconfigured |

---

## 4. Wiring Diagram

```
MARANTEC COMFORT 280 — XB03
─────────────────────────────────────────────────────
Terminal 3 (24V DC) ────────────────┐
Terminal 1 (GND)    ────────────────┼──── MP1584EN Buck Converter
                                    │    IN+ = Terminal 3
                                    │    IN− = Terminal 1
                                    │    OUT+ (5V) ──── ESP32-C3 5V pin
                                    │    OUT− (GND) ─── ESP32-C3 GND

Terminal 2 (Pulse)  ───┐
Terminal 1 (GND)    ────┼───────── G3VM-61A1 Output (pins 3+4)
                        │          G3VM-61A1 Input pin 1 ── 220Ω ── ESP32 GPIO10
                        │          G3VM-61A1 Input pin 2 ──────────── ESP32 GND

Terminal 70/71 (photocell) ─────── EXISTING WIRING — DO NOT MODIFY

ESP32-C3 MINI
─────────────────────────────────────────────────────
GPIO10  ── 220Ω ── G3VM Pin 1        (SSR trigger output)
GPIO4   ── Reed switch CLOSED ── GND (door closed sensor, internal pullup)
GPIO5   ── Reed switch OPEN   ── GND (door open sensor, internal pullup)
5V pin  ── MP1584EN OUT+             (power in)
GND     ── MP1584EN OUT−             (power ground)
```

---

## 5. Power Design

### Source

Marantec terminal 3 provides **24V DC, max 50mA**. Terminal 1 is GND.

### Buck Converter — MP1584EN module (set to 5V)

| Parameter | Value |
|---|---|
| Input voltage range | 4.5–28V |
| Output voltage | 5V fixed (preset, no trimmer) |
| Max continuous output current | 3A |
| Switching frequency | ~1.5MHz |
| Efficiency | up to ~92% |
| Protection | OCP, OTP, UVLO |

### Current Budget

```
ESP32-C3 average (WiFi active):   ~80mA  @ 3.3V  =  0.26W
ESP32-C3 peak (WiFi TX burst):   ~500mA  @ 3.3V  =  1.65W  (brief)
G3VM LED drive (during pulse):    ~10mA  @ 3.3V  =  0.03W

Average power out of buck:        ~0.30W
Average current from 24V rail:    0.30W / (24V × 0.87) ≈ 14mA  ✓ well within 50mA
Peak current from 24V rail:       1.68W / (24V × 0.87) ≈ 80mA  (brief burst, acceptable)
```

> **NOTE:** Average draw is well within the 50mA limit. WiFi TX peaks are brief (<10ms) and the Marantec's output is likely current-limited by a poly-fuse rather than hard-cut, so brief transients are acceptable. Monitor for thermal issues on first installation.

### Connection

```
Marantec Terminal 3 (24V) ──── MP1584EN IN+
Marantec Terminal 1 (GND) ──── MP1584EN IN−
MP1584EN OUT+ (5V)         ──── ESP32-C3 Mini 5V pin
MP1584EN OUT− (GND)        ──── ESP32-C3 Mini GND
```

> **NOTE:** Measure MP1584EN output voltage with a multimeter before connecting to the ESP32. Confirm 5.0V ±0.2V.

---

## 6. State Sensing

### Reed Switch Mounting

```
DOOR FULLY CLOSED:
  ┌─ magnet on door ──────────────────────────────┐
  └─ reed switch on frame (bottom) ───────────────┘
     → GPIO4 pulled LOW → state = CLOSED

DOOR FULLY OPEN:
  ┌─ magnet on door (top edge) ───────────────────┐
  └─ reed switch on track (top) ──────────────────┘
     → GPIO5 pulled LOW → state = OPEN

DOOR IN MOTION:
     Both GPIOs HIGH → state = OPENING or CLOSING
     (ESPHome tracks direction from last command)
```

### Recommended Reed Switch Type

Standard surface-mount magnetic contact sensors (alarm/security type, NO — normally open). The magnet mounts on the moving door panel; the switch body on the fixed frame or track. Gap tolerance of 15–25mm is typical and sufficient.

---

## 7. ESPHome Configuration Notes

The following outlines the key ESPHome YAML structure. Full YAML to be generated separately for Claude Code.

### Board Definition

```yaml
# Prototype
esp8266:
  board: d1_mini

# Final build — change these two lines only
esp32:
  board: lolin_c3_mini
  framework:
    type: arduino
```

### Key Components Required

```yaml
# WiFi + API
wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password

api:
  encryption:
    key: !secret esphome_api_key

ota:
  password: !secret ota_password

# Cover entity — maps to HomeKit garage door accessory
cover:
  - platform: template
    name: "Garage Door"
    device_class: garage
    # open/close/stop actions → pulse GPIO10 via SSR
    # open_sensor: GPIO5 (reed open)
    # close_sensor: GPIO4 (reed closed)

# SSR trigger — 300ms pulse
switch:
  - platform: gpio
    pin: GPIO10
    id: garage_relay
    restore_mode: ALWAYS_OFF

# Reed switches
binary_sensor:
  - platform: gpio
    pin:
      number: GPIO4
      mode: INPUT_PULLUP
      inverted: true
    name: "Door Closed Sensor"
    id: door_closed
    filters:
      - delayed_on: 50ms   # debounce

  - platform: gpio
    pin:
      number: GPIO5
      mode: INPUT_PULLUP
      inverted: true
    name: "Door Open Sensor"
    id: door_open
    filters:
      - delayed_on: 50ms   # debounce
```

### HomeKit Bridge (Home Assistant side)

No additional configuration required beyond adding the HomeKit Bridge integration in HA. The `cover` entity with `device_class: garage` is automatically mapped to a HomeKit Garage Door Opener accessory type.

---

## 8. Prototype vs Final Build

| Aspect | Prototype (D1 Mini) | Final (C3 Mini) |
|---|---|---|
| MCU | ESP8266 | ESP32-C3 |
| Power | USB from bench supply | MP1584EN from Marantec 24V |
| SSR | Breadboard / relay module | G3VM-61A1 on PCB |
| Reed switches | Jumper wires | Screw terminals on PCB |
| ESPHome board | `d1_mini` | `lolin_c3_mini` |
| Framework | Arduino | Arduino |
| GPIO voltage | 3.3V (some 5V tolerant) | Strictly 3.3V all pins |
| Form factor | Breadboard | Custom PCB or stripboard |

### Migration Steps (D1 Mini → C3 Mini)

1. Change `esp8266: board: d1_mini` → `esp32: board: lolin_c3_mini` + framework block
2. Verify GPIO pin numbers match physical wiring on C3 Mini (pinout differs from D1 Mini)
3. Change power input from USB to MP1584EN 5V output on 5V pin
4. Re-flash via USB then confirm OTA works before installing in garage

---

## 9. Bill of Materials

| Component | Part | Qty | Notes |
|---|---|---|---|
| Microcontroller | Wemos C3 Mini (ESP32-C3) | 1 | Final build |
| Microcontroller | D1 Mini (ESP8266) | 1 | Prototype only |
| Solid State Relay | OMRON G3VM-61A1 | 1 | SOP-4 SMD package |
| Current limit resistor | 220Ω 0.25W | 1 | SSR LED drive |
| Buck converter | MP1584EN 5V variant | 1 | 24V→5V, self-powered |

| Reed switch (closed) | NO magnetic contact sensor | 1 | Door closed detection |
| Reed switch (open) | NO magnetic contact sensor | 1 | Door open detection |
| Screw terminals | 2-pin 2.54mm or 3.5mm pitch | 3 | Power in, trigger out, sensors |
| PCB / stripboard | To suit enclosure | 1 | Custom layout |
| Enclosure | Small project box | 1 | Mount near Marantec unit |
| Wire | 2-core alarm cable | ~3m | Reed switch runs to door |

---

## 10. Safety Notes

1. **Isolation is mandatory.** The G3VM-61A1 SSR must be used — never connect ESP32 GPIO directly to Marantec terminals. External voltage on XB03 will destroy the control board.

2. **Do not disturb terminals 70/71.** The obstacle photocell safety circuit is wired here. This is a functional safety device — modifying or interrupting it could cause the door to close on an obstruction.

3. **Measure buck converter output before connecting.** Confirm 5.0V ±0.2V with a multimeter before powering the ESP32 for the first time.

4. **Power down the Marantec before wiring.** Disconnect mains power before making any connections to XB03.

5. **Only potential-free contacts to terminals 1, 2, 4.** Confirmed by Marantec manual. The SSR output satisfies this requirement.

6. **Line-of-sight operation.** The Marantec is a safety device. Automations that trigger the door remotely should include a check (via reed switch state) that the area is clear, or require confirmation before closing.

7. **Test obstruction detection after installation.** After wiring is complete, verify the photocell/obstacle sensors still function correctly by testing with an object in the door path.

---

*Document version 1.0 — generated May 2026*  
*To be used in conjunction with ESPHome YAML (generate via Claude Code)*
