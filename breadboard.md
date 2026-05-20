# Wiring Guide — Garage Door Controller V2

**Board:** ESP32-C3 Super Mini  
**Power:** Mini 5605V buck converter (Marantec 24V → 5V)  
**Isolation:** OMRON G3VM-61A1 MOSFET SSR (SOP-4)  
**Sensor:** CJVL53L0XV2 (VL53L0X ToF breakout) — ceiling-mounted, I2C

---

## Pin Assignments

| Signal | GPIO | Notes |
|---|---|---|
| SSR trigger | GPIO7 | Drives G3VM-61A1 via 220Ω resistor |
| I2C SDA | GPIO1 | VL53L0X data |
| I2C SCL | GPIO3 | VL53L0X clock |
| Power in | 5V pin | From Mini 5605V buck converter |
| Sensor power | 3.3V pin | VL53L0X VCC — must be 3.3V, not 5V |
| Ground | GND | Shared: ESP, buck converter, sensor |

---

## Components

| Item | Qty | Notes |
|---|---|---|
| ESP32-C3 Super Mini | 1 | `esp32-c3-devkitm-1`, esp-idf framework |
| OMRON G3VM-61A1 | 1 | SOP-4 package |
| Resistor, 220Ω | 1 | Sets SSR LED drive current to ~10mA |
| CJVL53L0XV2 breakout | 1 | VL53L0X ToF sensor, I2C address 0x29 |
| Mini 5605V buck module | 1 | Pre-set to 5V; powers ESP from Marantec 24V rail |

---

## G3VM-61A1 Pin Reference (SOP-4)

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
| 1 | Input anode (LED +) | 220Ω resistor → GPIO7 |
| 2 | Input cathode (LED −) | GND |
| 3 | Output | Marantec XB03 Terminal 1 (GND) |
| 4 | Output | Marantec XB03 Terminal 2 (Pulse) |

Pins 3 and 4 are the MOSFET drain/source — polarity does not matter on the output side.

---

## Wiring

### 1 — Buck converter

```
Marantec XB03 Terminal 3 (24V) ──── Mini 5605V IN+
Marantec XB03 Terminal 1 (GND) ──── Mini 5605V IN−

Mini 5605V OUT+ ──── ESP32-C3 5V pin
Mini 5605V OUT− ──── ESP32-C3 GND
```

> **Before connecting the ESP:** Measure OUT+ to GND with a multimeter — must read **5.0V ±0.2V**.

---

### 2 — SSR input drive

```
ESP32-C3 GPIO7 ──── 220Ω ──── G3VM Pin 1 (anode)
ESP32-C3 GND   ─────────────  G3VM Pin 2 (cathode)
```

The 220Ω resistor limits LED current to ~10mA: (3.3V − 1.2V) / 220Ω ≈ 9.5mA, within the G3VM's 5–50mA control range.

---

### 3 — SSR output → Marantec XB03

```
G3VM Pin 3 ──── Marantec XB03 Terminal 1 (GND)
G3VM Pin 4 ──── Marantec XB03 Terminal 2 (Pulse)
```

When GPIO7 goes HIGH, the MOSFET output closes, momentarily shorting Terminal 2 to Terminal 1 — identical to pressing the wall button. No voltage crosses from the ESP side to the Marantec side.

---

### 4 — VL53L0X ToF sensor

```
ESP32-C3 3.3V pin ──── VL53L0X VCC
ESP32-C3 GND      ──── VL53L0X GND
ESP32-C3 GPIO1    ──── VL53L0X SDA
ESP32-C3 GPIO3    ──── VL53L0X SCL
```

> **3.3V only:** Powering the module from 5V will put 5V logic on the I2C lines and damage the ESP32-C3. Use the 3.3V pin exclusively.

The sensor is ceiling-mounted pointing down at the top of the door panel. No pull-up resistors are needed — the CJVL53L0XV2 breakout includes them.

---

## Connection Diagram

```
                 ESP32-C3 Super Mini
                 ┌──────────────────┐
          3.3V ──┤                  ├── GPIO7 ── 220Ω ── G3VM Pin 1 (anode)
           GND ──┤                  ├── GND  ─────────── G3VM Pin 2 (cathode)
            5V ──┤                  ├── GPIO1 (SDA) ──── VL53L0X SDA
                 │                  ├── GPIO3 (SCL) ──── VL53L0X SCL
                 └──────────────────┘
                       │       │
                  5V pin     3.3V pin
                       │       │
              Mini 5605V    VL53L0X VCC
              OUT+

  G3VM Pin 3 ──── Marantec XB03 Terminal 1 (GND)
  G3VM Pin 4 ──── Marantec XB03 Terminal 2 (Pulse)

  Mini 5605V IN+ ──── Marantec XB03 Terminal 3 (24V)
  Mini 5605V IN− ──── Marantec XB03 Terminal 1 (GND)
```

---

## Marantec XB03 Terminal Block

```
[ 3 ][ 1 ][ 2 ][ 4 ][ 70 ][ 71 ]
```

| Terminal | Function | Connection |
|---|---|---|
| 1 | GND | G3VM Pin 3; buck IN− |
| 2 | Pulse input | G3VM Pin 4 |
| 3 | 24V DC output | Buck IN+ |
| 70/71 | Photocell safety | Do not touch |

---

## Bench Test Procedure

### Step 1 — Buck converter (before connecting ESP)

1. Connect Mini 5605V IN+/IN− to Marantec XB03 terminals 3 and 1.
2. Measure OUT+ to GND — must read **5.0V ±0.2V** before proceeding.

### Step 2 — Sensor detection

3. Flash firmware and connect to logs: `esphome logs garage_door_v2.yaml`
4. On boot, look for: `[I][i2c.idf:069]: Found i2c device at address 0x29`
5. If 0x29 is not listed, stop and check SDA/SCL wiring and VCC before proceeding.

### Step 3 — SSR trigger

6. In Home Assistant → Developer Tools → Actions, call `cover.open_cover` on the Garage Door entity.
7. Probe G3VM pins 3 and 4 with a multimeter set to continuity — should close (beep) for ~300ms then release.

### Step 4 — End-to-end

8. Connect G3VM output to Marantec XB03 terminals 1 and 2.
9. Trigger open and close from HA and confirm the cover entity transitions correctly through Open → Closing → Closed → Opening → Open.
