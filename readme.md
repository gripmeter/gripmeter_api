# GripMeter BLE API Documentation

This document describes the BLE interface exposed by the GripMeter device, for anyone building a mobile app or other BLE client against the device.

## Table of Contents

- [1. Connecting](#1-connecting)
- [2. Services and Characteristics](#2-services-and-characteristics)
  - [2.1 GripMeter Service (custom)](#21-gripmeter-service-custom)
  - [2.2 Battery Service (standard BLE SIG service)](#22-battery-service-standard-ble-sig-service)
- [3. Typical client workflow](#3-typical-client-workflow)
- [4. Quick reference](#4-quick-reference)
- [5. Notes / caveats for integrators](#5-notes--caveats-for-integrators)

---

## 1. Connecting

- **Device name:** advertised as the firmware name, `"GripMeter"`.
- **Advertised services:** the custom GripMeter service and the standard Battery Service (see below). Scan for either UUID.
- **MTU:** the device requests an MTU of **303 bytes**. Android clients in particular must explicitly request this MTU after connecting, or payloads will be silently truncated:

  ```java
  connection.requestMtu(303)
  ```

  (GATT reserves 3 bytes of overhead, leaving ~300 usable payload bytes.)

- **Connect/disconnect feedback:** the device gives a short buzzer beep on every connect and disconnect event, and briefly (2 s) shows a "New Device CONNECTED" / "Device Disconnected" banner on its own screen. There is no client-facing notification characteristic for this — it's local device feedback only.

---

## 2. Services and Characteristics

### 2.1 GripMeter Service (custom)

| | UUID |
|---|---|
| **Service** | `67726970-6d65-7465-722e-6e6574131000` |
| **Config characteristic** | `67726970-6d65-7465-722e-6e6574636e66` |
| **Scale/force characteristic** | `67726970-6d65-7465-722e-6e6574626174` |
| **Battery characteristic** | `67726970-6d65-7465-722e-6e6574727761` |

All three characteristics support **Read, Write, Notify, and Indicate**, and each has a CCCD (0x2902) descriptor — a client must write `0x0001` (or `0x0002` for indicate) to the CCCD to receive notifications.

#### Config characteristic — `...6e66`

Bidirectional JSON channel used both to read/broadcast current device configuration and to send commands to the device.

**Read / Notify payload** — the full contents of the device's `config.json`, e.g.:

```json
{
  "serial": "rv00000",
  "tare": 7000,
  "scale_factor": 22000,
  "rate": 0,
  "dm": 0
}
```

The device notifies this characteristic automatically after boot and after any successful config write.

**Write payload** — a small JSON command object:

```json
{ "ID": "<command>", "data": "<value>" }
```

`ID` is case-insensitive. Supported commands:

| `ID` | `data` | Effect |
|---|---|---|
| `tare` | integer (as string) | Sets the zero-offset ("tare") for the load cell. Persisted to flash. |
| `scale_factor` | integer (as string) | Sets the calibration factor used to convert raw ADC counts to kg. Persisted to flash. |
| `rate` | `"hi"` or anything else | Sets sample rate: `"hi"` → high rate (80 Hz), otherwise → low rate (10 Hz). Persisted to flash. |
| `dm` | `"m"` or anything else | Sets display mode: `"m"` → **MAX** (holds peak reading), otherwise → **LIVE**. Persisted to flash. |
| `cmd` | `"short_buzz"` | Triggers a short buzzer beep. |
| `cmd` | `"long_buzz"` | Triggers a long buzzer beep. |
| `cmd` | `"double_buzz"` | Triggers a double buzzer beep. |
| `cmd` | `"disp_smile"` | Shows a "happy face" easter-egg screen (halts normal display until reset). |
| `cmd` | `"disp_error"` | Shows a test error screen. |
| `cmd` | `"dev_bat"` | Reserved/dev command; currently a no-op on-device besides logging. |

After any successful `tare`/`scale_factor`/`rate`/`dm` write, the device re-serializes its config and **notifies the config characteristic** with the updated JSON — treat that notification as your write-acknowledgement rather than assuming success from the write response alone.

⚠️ **Known firmware limitation:** every accepted write persists a full config JSON to flash immediately (no debouncing). Avoid writing this characteristic in a tight loop (e.g. while a user drags a calibration slider) — batch/settle changes client-side before sending, to avoid excessive flash wear.

#### Scale characteristic — `...6261 74` 

**Notify payload:** 4 bytes, little-endian signed 32-bit integer — the raw ADC reading.

```
byte[0] = value        & 0xFF
byte[1] = (value >> 8)  & 0xFF
byte[2] = (value >> 16) & 0xFF
byte[3] = (value >> 24) & 0xFF
```

Decode as `int32_t` (little-endian). This is a **raw ADC count**, not kilograms — convert client-side (or read the device's own on-screen kg value, which the firmware computes) using:

```
kg = (raw_value - tare) / scale_factor
```

using the `tare` and `scale_factor` values from the config characteristic.

- Notifications are pushed continuously from the high-priority sampling task, as fast as the ADC produces a ready sample (up to ~80 Hz at high sample rate, ~10 Hz at low).
- No dedicated "get raw kg" read value is exposed beyond this; the raw ADC integer plus tare/scale_factor is the full picture.

#### Battery characteristic — `...7261 77`

**Read / Notify payload:** 1 byte, unsigned — battery percentage, 0–100.

Notified whenever the device polls its battery (every `BAT_PERIOD` = 4000 ms).

---

### 2.2 Battery Service (standard BLE SIG service)

For generic BLE clients (e.g. OS-level battery widgets) that don't know the custom GripMeter protocol.

| | UUID |
|---|---|
| **Service** | `0x180F` (Battery Service) |
| **Characteristic** | `0x2A19` (Battery Level) |

**Read / Notify payload:** 1 byte, unsigned, 0–100 — identical value to the custom battery characteristic above, mirrored here for standard-compliant clients. Supports Read + Notify only (no write).

---

## 3. Typical client workflow

1. Scan for a device advertising `"GripMeter"` or the GripMeter service UUID.
2. Connect, then request MTU 303.
3. Discover services/characteristics; enable notifications (write CCCD) on the **Config** and **Scale** characteristics (and **Battery** if you want live battery updates).
4. Read the Config characteristic once on connect to get `tare`, `scale_factor`, `rate`, `dm`, `serial`.
5. Stream live force by decoding Scale notifications and applying `(raw - tare) / scale_factor`.
6. To tare: Have the user unload the scale, then write `{"ID":"tare","data":"<current raw reading>"}` (or trigger a physical zero and let the device auto-tare on next boot).
7. To Calibrate:To set a known-weight calibration factor, compute `scale_factor = (raw_reading - tare) / known_kg` and write `{"ID":"scale_factor","data":"<value>"}`.
7. Wait for the Config notification confirming the write took effect before assuming success.

---

## 4. Quick reference

```
Service (GripMeter):     67726970-6d65-7465-722e-6e6574131000
  Config   (R/W/N/I):    67726970-6d65-7465-722e-6e6574636e66   JSON in/out
  Scale    (R/W/N/I):    67726970-6d65-7465-722e-6e6574626174   int32 LE (raw ADC)
  Battery  (R/W/N/I):    67726970-6d65-7465-722e-6e6574727761   uint8 (0-100 %)

Service (Battery, SIG standard): 0x180F
  Battery Level (R/N):            0x2A19                        uint8 (0-100 %)

Recommended MTU: 303 bytes
```

---

## 5. Notes / caveats for integrators

- The firmware currently has **no BLE authentication or encryption** configured — anyone in range can connect and write configuration or trigger commands.
- `disp_smile` and `disp_error` commands **halt the normal live display** (`_haltScreen = true` internally) until the device is reset or re-tared through other means — use only for deliberate testing.
