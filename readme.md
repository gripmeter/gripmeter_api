# GripMeter BLE API Documentation

**API version 2.0** (firmware `2.0`) — the Scale characteristic treams a **timestamp + raw ADC value** (8-byte payload). See [§2.1 Scale characteristic](#scale-characteristic--6261-74).

This document describes the BLE interface exposed by the GripMeter device, for anyone building a mobile app or other BLE client against the device.

## Table of Contents

- [1. Connecting](#1-connecting)
- [2. Services and Characteristics](#2-services-and-characteristics)
  - [2.1 GripMeter Service (custom)](#21-gripmeter-service-custom)
  - [2.2 Battery Service (standard BLE SIG service)](#22-battery-service-standard-ble-sig-service)
- [3. Typical client workflow](#3-typical-client-workflow)
- [4. Quick reference](#4-quick-reference)
- [5. Notes / caveats for integrators](#5-notes--caveats-for-integrators)
- [6. Changelog](#6-changelog)

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
| **Scale/force characteristic** | `67726970-6d65-7465-722e-6e6574626174` — streams `timestamp + raw ADC` (8 bytes) since API v2.0 |
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

**Notify payload (API v2.0):** 8 bytes — two little-endian 32-bit integers, **timestamp first, then raw ADC value**.

```
byte[0] = timestamp   & 0xFF        # uint32, milliseconds
byte[1] = (timestamp >> 8)  & 0xFF
byte[2] = (timestamp >> 16) & 0xFF
byte[3] = (timestamp >> 24) & 0xFF
byte[4] = raw_value   & 0xFF        # int32, raw ADC count
byte[5] = (raw_value >> 8)  & 0xFF
byte[6] = (raw_value >> 16) & 0xFF
byte[7] = (raw_value >> 24) & 0xFF
```

- **`timestamp`** — `uint32` little-endian, **milliseconds since the device booted** (Arduino `millis()`), sampled on-device the instant the ADC value is read, inside the high-priority sampling task. Use it to reconstruct exact inter-sample spacing and to detect dropped samples, rather than relying on notification arrival time (which is jittered by the BLE stack).
  - It is a **device-relative uptime clock, not wall-clock time** and is not synced to the client. On connect, record `t0 = first timestamp` and treat every later sample as `t - t0`.
  - It **wraps back to 0 after ~49.7 days** of uptime (`2^32` ms). Handle the wrap (unsigned subtraction) if you keep long-running sessions.
- **`raw_value`** — `int32` little-endian, a **raw ADC count**, not kilograms. Convert client-side (or read the device's own on-screen kg value, which the firmware computes) using:

  ```
  kg = (raw_value - tare) / scale_factor
  ```

  with the `tare` and `scale_factor` values from the config characteristic.

- Notifications are pushed continuously from the high-priority sampling task, as fast as the ADC produces a ready sample (up to ~80 Hz at high sample rate, ~10 Hz at low).
- **Bad-sample filter:** the firmware drops (does not notify) any sample whose `raw_value` has both `byte[4] == 0xFF` and `byte[7] == 0xFF` — a known class of corrupt ADC reads. Clients therefore see a small number of missing samples; the `timestamp` gap makes these visible.
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
5. Stream live force by decoding Scale notifications: split each 8-byte payload into `timestamp` (bytes 0–3, `uint32` LE) and `raw` (bytes 4–7, `int32` LE), then apply `(raw - tare) / scale_factor`. Store `t0` from the first notification and plot/record samples against `timestamp - t0` (in ms) instead of arrival time.

   ```js
   // Web Bluetooth: event.target.value is a DataView over the 8-byte payload
   const dv = event.target.value;
   const timestamp = dv.getUint32(0, /*littleEndian=*/true);   // ms since device boot
   const raw       = dv.getInt32(4, /*littleEndian=*/true);    // raw ADC count
   const kg        = (raw - tare) / scaleFactor;
   ```

6. To tare: Have the user unload the scale, then write `{"ID":"tare","data":"<current raw reading>"}` (or trigger a physical zero and let the device auto-tare on next boot).
7. To Calibrate:To set a known-weight calibration factor, compute `scale_factor = (raw_reading - tare) / known_kg` and write `{"ID":"scale_factor","data":"<value>"}`.
7. Wait for the Config notification confirming the write took effect before assuming success.

---

## 4. Quick reference

```
Service (GripMeter):     67726970-6d65-7465-722e-6e6574131000
  Config   (R/W/N/I):    67726970-6d65-7465-722e-6e6574636e66   JSON in/out
  Scale    (R/W/N/I):    67726970-6d65-7465-722e-6e6574626174   8 bytes: uint32 LE timestamp(ms) + int32 LE raw ADC
  Battery  (R/W/N/I):    67726970-6d65-7465-722e-6e6574727761   uint8 (0-100 %)

Service (Battery, SIG standard): 0x180F
  Battery Level (R/N):            0x2A19                        uint8 (0-100 %)

Recommended MTU: 303 bytes
```

---

## 5. Notes / caveats for integrators

- The firmware currently has **no BLE authentication or encryption** configured — anyone in range can connect and write configuration or trigger commands.
- `disp_smile` and `disp_error` commands **halt the normal live display** (`_haltScreen = true` internally) until the device is reset or re-tared through other means — use only for deliberate testing.
- The Scale payload grew from 4 to 8 bytes in API v2.0. Clients that assume a fixed 4-byte length will misread it — always slice by offset (`raw` is at bytes 4–7, **not** 0–3).

---

## 6. Changelog

### API v2.0 — firmware `2.0` ("Update API v2.0 (Time)")

- **Scale characteristic (`...6261 74`) payload changed from 4 bytes to 8 bytes.**
  - **Old (v1.x):** `int32` LE raw ADC value only.
  - **New (v2.0):** `uint32` LE **timestamp in milliseconds** (device uptime, `millis()`, captured at sample time) in bytes 0–3, followed by the `int32` LE raw ADC value in bytes 4–7.
  - **Migration:** read the raw ADC value from **byte offset 4** (not 0). Use bytes 0–3 as a monotonic per-sample time base (subtract the first value seen; handle the ~49.7-day wrap).
  - The internal bad-sample filter now tests the raw-value bytes (`byte[4]`/`byte[7]`) instead of `byte[0]`/`byte[1]`; behaviour for clients is unchanged (still just a dropped notification).
- Config, Battery, and standard Battery Service characteristics are **unchanged**.
