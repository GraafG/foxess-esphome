# FoxESS T-Series — ESPHome / Home Assistant

Read your FoxESS T-Series solar inverter into Home Assistant in real time using an ESP32-C3 and RS485.

## Hardware

| Component | Detail |
|-----------|--------|
| Microcontroller | ESP32-C3 DevKitM-1 |
| RS485 interface | [Smart-Stuff P1 Modbus Dongle](https://smart-stuff.nl/product/p1-naar-modbus-dongle/) |
| Protocol | RS485 Modbus RTU, 9600 baud |
| UART TX | GPIO0 |
| UART RX | GPIO10 |
| RS485 flow control (DE/RE) | GPIO4 |
| Status LED | GPIO7 (active low) |

The **Smart-Stuff P1 Modbus Dongle** exposes RS485 Modbus RTU via screw terminals (A/B/GND). Connect A and B to the inverter's RS485 port. The dongle's USB-C port can provide additional power if needed.

## Software

- **ESPHome** (tested on 2026.4.3+)
- **Home Assistant** (any recent version)

## Folder structure

```
esphome/
├── components/
│   └── foxess_solar/        # Local external component (vendored from PR #26)
│       ├── __init__.py
│       ├── foxess_solar.h
│       ├── foxess_solar.cpp
│       └── sensor.py
├── foxess-inverter.yaml     # Main ESPHome config
├── secrets.yaml             # Credentials (not committed)
└── old/                     # Archived — no longer needed
```

## Setup

### 1. secrets.yaml

Create (or update) `secrets.yaml` with these keys:

```yaml
wifi_ssid: "your-wifi"
wifi_password: "your-wifi-password"
esphome_ota_pass: "your-ota-password"
esphome_api_encryption_key: "base64-32-byte-key"   # generate: esphome generate-encryption-key
foxess_ap_ssid: "FoxESS Inverter Fallback"
foxess_ap_pass: "your-fallback-ap-password"
```

Generate an API encryption key:
```bash
esphome generate-encryption-key
```

### 2. Flash

```bash
esphome run foxess-inverter.yaml
```

Subsequent updates are via OTA (no USB needed).

### 3. Add to Home Assistant

ESPHome will be discovered automatically. Accept the device and enter your API encryption key when prompted.

## Configuration

### Battery model sensors

`T-Series Grid Power` and `T-Series Loads Power` are only populated on **battery-equipped** T-Series models. On non-battery inverters these registers always return `0 W`.

They are hidden by default (`internal: true`). To expose them, remove the `internal: true` lines from both sensors in `foxess-inverter.yaml`:

```yaml
grid_power:
  name: "T-Series Grid Power"
  id: "grid_power"
  icon: mdi:lightning-bolt
  # internal: true   ← delete or comment out this line

loads_power:
  name: "T-Series loads Power"
  id: "loads_power"
  icon: mdi:lightning-bolt
  # internal: true   ← delete or comment out this line
```

> **Note:** Per-phase grid power (`T-Series Grid Power R/S/T`) works on all models and is always exposed — it is the recommended way to monitor grid import/export on non-battery inverters.

### PV strings

The YAML exposes PV1–PV4 sensor groups by default, but most inverters only use 1 or 2 strings. Unused strings show constant `0 W / 0 V / 0 A` in Home Assistant, cluttering dashboards and writing unnecessary history.

**To hide unused PV strings**, add `internal: true` to each sensor under the unused `pv2` / `pv3` / `pv4` block:

```yaml
pv2:
  voltage:
    name: "T-Series PV2 Voltage"
    id: "pv2_voltage"
    icon: mdi:flash
    internal: true   # ← hides from HA; data is still read from inverter
  current:
    name: "T-Series PV2 Current"
    id: "pv2_current"
    icon: mdi:current-ac
    internal: true
  active_power:
    name: "T-Series PV2 Power"
    id: "pv2_power"
    icon: mdi:lightning-bolt
    internal: true
```

`internal: true` means ESPHome still polls the register and makes the value available in C++ automations, but the entity is **never sent to Home Assistant**. This is preferred over deleting the block entirely because it keeps the YAML as a single file that works for all T-Series variants — just comment/uncomment one line per sensor to adapt.

| Model | Typical active strings |
|-------|------------------------|
| T3/T5/T6 (single MPPT) | PV1 only |
| T8dual–T12dual (dual MPPT) | PV1 + PV2 |
| T15–T25 (quad MPPT) | PV1 + PV2 + PV3 + PV4 |

**To re-enable a string**, remove (or set to `false`) the `internal: true` lines and reflash via OTA.

A comment block at the top of `foxess-inverter.yaml` documents which strings are active for your install:

```yaml
# ============================================================
# PV STRING CONFIGURATION
# Set internal: false for strings connected to your inverter.
# Set internal: true (or remove) for unused strings.
# This inverter uses 1 PV string (PV1 only).
# ============================================================
```

---

## Sensors exposed to Home Assistant

### Power (live, 1 s update)

| Entity | Unit | Notes |
|--------|------|-------|
| T-Series Grid Power | W | Signed: + = import, − = export. **Battery models only** — hidden by default |
| T-Series Generation Power | W | Total inverter output |
| T-Series Loads Power | W | House consumption. **Battery models only** — hidden by default |
| T-Series Grid Power R/S/T | W | Per-phase |

### Voltage / Current / Frequency (throttled to 1 h)

Per-phase R, S, T — voltage (V), current (A), frequency (Hz).

### PV Strings

PV1–PV4 voltage, current, and power (calculated as V×I).  
PV3/PV4 only relevant on T8dual–T12dual and T15–T25 models.  
Unused strings should be set to `internal: true` — see [Configuration → PV strings](#pv-strings) above.

### Temperature (throttled to 10 min)

Boost, Inverter, and Ambient temperatures in °C.

### Energy (HA Energy Dashboard)

| Entity | Unit | state_class |
|--------|------|-------------|
| T-Series Today Yield | kWh | `total_increasing` |
| T-Series Generation Total | kWh | `total_increasing` |

Both are pre-configured for the **HA Energy Dashboard** — just add "T-Series Generation Total" as your solar production source.

### Status

| Entity | Type | Values |
|--------|------|--------|
| T-Series Inverter Mode | Text sensor | Offline / Online / Error / Waiting… |

## Component notes

The `components/foxess_solar/` folder is a vendored copy of [PR #26](https://github.com/assembly12/Foxess-T-series-ESPHome-Home-Assistant/pull/26) by [@SibrenVasse](https://github.com/SibrenVasse). It replaces the original `platform: custom` approach (removed in ESPHome 2026.4.3) with a proper `external_components` architecture.

Key improvements over the original `foxess_t_series.h`:

| Issue | Old | New |
|-------|-----|-----|
| ESPHome compatibility | Broken (`platform: custom` removed) | ✅ `external_components` |
| `delay()` calls | 30+ blocking delays (≥50 ms each) | ✅ Non-blocking, event-driven |
| CRC validation | None — corrupt frames accepted silently | ✅ CRC16 checked |
| Grid power sign | Unsigned (import/export indistinguishable) | ✅ Signed int16 |
| Buffer | Growing `std::vector` → heap fragmentation | ✅ Fixed 256-byte `std::array` |
| Sensor allocation | 37× `new Sensor` — heap never freed | ✅ Stack-allocated structs |
| AP credentials | Hardcoded in YAML | ✅ `!secret` references |
| API security | Unencrypted | ✅ Encrypted with key |

## Troubleshooting

**Inverter shows "Waiting for response…"**  
Check RS485 wiring polarity (swap A/B if needed). Verify GPIO4 is your DE/RE pin.

**CRC errors in logs**  
Cable too long (>4 m can be marginal). Try ferrite bead on RS485 cable.

**Message too short warning**  
Your inverter sent a frame shorter than the known T-Series sensor offsets. Capture a verbose UART log before trusting the parsed values.

**HA Energy Dashboard shows wrong values**  
Ensure "T-Series Generation Total" is added as the solar source (not "Today Yield" — that resets daily).
