# LaptopKuehlung_Luefter (laptop cooling – fan)

Switches a 12 V fan through a relay. The temperature arrives via USB from the [bridge](../LaptopKuehlung_Bridge/), which receives it over MQTT from the [sensor](../LaptopKuehlung_Sensor/).

## Hardware

| Component | Connection |
|---|---|
| Arduino Uno | USB to the laptop (default: COM3) |
| Relay module (high level trigger) | `IN → D3`, `VCC → 5V`, `GND → GND` |
| 12 V fan | through the relay contact (`COM`/`NO`) to its own 12 V supply |

If the relay pulls in on LOW, set `FAN_ACTIVE_LOW = true` in the sketch.

## Upload

Board **Arduino Uno**, port **COM3**. No additional libraries.

## Behaviour

| Situation | Fan |
|---|---|
| Temperature ≥ 23.0 °C | ON |
| Temperature < 22.0 °C | OFF |
| in between | stays as it is |
| Sensor reports `error` or `offline` | ON immediately (fail-safe) |
| No temperature for 60 s | ON (fail-safe) |

## Control

- **MQTT:** `laptop/luefter/set` with `ON`, `OFF` or `AUTO`
- **Without the bridge:** Serial Monitor (115200 baud, line ending "Newline"): `ON`, `OFF`, `AUTO` or `STATUS`

The fan Uno reports every change (`fan`, `mode`, `failsafe`) to the bridge, which publishes it to `laptop/luefter/...`.

## Log (USB, 115200 baud)

```
6120;luefter;INFO;temp;nr=4,temp=27.3,hum=41
6121;luefter;INFO;fan;zustand=ON,grund=temp_hoch,temp=27.3,an_ab=23.0,aus_unter=22.0
```

The Uno checks the checksum and the sequence number of every frame and reports errors (`bad_frame`), gaps (`nr_luecke`) and duplicates.

## Settings

At the top of the sketch: `FAN_PIN`, `FAN_ACTIVE_LOW`, `TEMP_ON`, `TEMP_OFF`, `TEMP_TIMEOUT_MS`, `ACK_TIMEOUT_MS`, `MAX_FAILS`, `LOG_FRAMES`.
