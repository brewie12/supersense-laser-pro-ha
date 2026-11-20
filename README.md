# SuperSense Pro water-tank fill level (`supersense`)

Local BLE integration for the Comworks SuperSense Laser Pro fill sensor, derived from
[adahmen/supersense-homeassistant](https://github.com/adahmen/supersense-homeassistant) but
heavily customised. Manifest v1.0.0, `iot_class: local_push`, config flow, no HACS release.
Protocol details were reverse-engineered from the vendor Android app
`de.comworks.supersense` v2.11.115.

## Single-client limitation

The sensor accepts **one** BLE client at a time. Keep the vendor phone app closed and
disconnected on every phone; if it is connected the sensor stops advertising and the
integration sits logging “not visible yet, waiting for advertisement” even with a healthy
Bluetooth adapter.

## Installation and configuration

1. Copy `custom_components/supersense/` into `/config/custom_components/` and restart HA.
2. With the sensor powered and in range it should appear under
   Settings → Devices & Services → Discovered (matched on service UUID `a0eeb190-…`).
   If not, add it manually — the flow accepts a discovered device or a free-form MAC.
3. Setup asks for a name, tank capacity in litres, tank inner depth in mm, and sensor
   mounting depth in mm. Defaults: `SuperSense Wassertank`, 100 L, 350 mm, 379 mm
   (tank depth plus ~30 mm clearance above the maximum water line). All geometry values can
   be changed later via **Reconfigure**, which reloads the entry.

## GATT layout

| Role | UUID | Notes |
|---|---|---|
| Service | `a0eeb190-6b2a-45fb-966e-d8e49323691f` | Registered with the HA Bluetooth matcher |
| Data channel | `8215dd0f-cc20-4d2a-8b64-73907ec4bd16` | Read + notify; carries the 8-byte reading |
| Command channel | `8d81b2f7-e56f-44bf-94ee-94083f5ea6d7` | Write `0E 01 02 03 04 05 06` to trigger a fresh laser measurement; also subscribed for notify |
| Unused notify | `0bd4c5d4-62e3-4946-82ea-d7643834d368` | Always empty on this hardware |

**Do not write the init packet to the data channel** — that would overwrite the stored reading.

## Payload

The 8-byte reading is four **BCD-encoded** pairs, not hex: bytes `0x04 0x27` decode as
`int("0427", 10) = 427`, not `0x0427 = 1063`. `9999` is the “not available” sentinel.

| Field | Meaning |
|---|---|
| `[0]` | Water column height in 0.1 mm; rises and falls directly with the level (it is *not* an air gap to subtract from tank depth) |
| `[1]` | Temperature in 0.1 °C (e.g. 235 → 23.5 °C) |
| `[2]`, `[3]` | Reserved; always `9999` on this hardware |

## Fill calculation

The sensor has a hardware dead zone: even with a completely empty tank the raw value stays
at about 20.0 mm (confirmed live 2026-06-23). That floor is subtracted first:

```
height_fraction = clamp((raw_01mm × 0.1 − 20.0) / (tank_depth_mm − 20.0), 0, 1)
volume_fraction = height_fraction ** 0.7716
fill_litre      = volume_fraction × tank_capacity
```

The exponent corrects for RV tanks being wider at the bottom than the top. It was fitted on
2026-07-15 from two hand-measured points on this vehicle's tank: empty (raw = 20 mm → 0 %)
and raw = 154.4 mm holding 60 L of a 120 L tank (50 %). It is tank-shape-specific — for a
different tank, recalibrate it from at least one known mid-fill measurement. The
`sensor_depth_mm` field is stored in the config entry but is not used by the current fill
calculation.

## Entities

Device: “SuperSense Wassertank” (manufacturer Comworks, model SuperSense Pro, software
2.11.115).

| Key | Name | Unit | Notes |
|---|---|---|---|
| `fill_level_liter` | Füllstand | L | `volume_storage` |
| `fill_level_percent` | Füllstand % | % | Clamped to 100 % |
| `fill_level_raw` | Laser Rohwert | 0.1 mm | Disabled by default |
| `temperature` | Temperatur | °C | Unavailable when the sensor sends the sentinel |

These are `RestoreSensor` entities, so the last value survives restarts and temporary
unavailability. Entity IDs in this installation:

```
sensor.supersense_wassertank_fullstand
sensor.supersense_wassertank_fullstand_2
sensor.supersense_wassertank_temperatur
```

## Connection behaviour

A passive Bluetooth callback on the configured MAC triggers a connect. On connect the
integration first reads the data channel to recover the retained reading, subscribes to both
the data and command characteristics, then writes the 7-byte init packet to the command
channel. A connect lock prevents overlapping connection attempts. A watchdog reconnects after
70 s without data, and a forced reconnect runs once an hour as a safety net against silent
connections. Reuse delay is 10 s and connect timeout 15 s.

This installation additionally runs a nightly automation that reloads the integration's
config entry as an extra reconnect safety net.
