# Arduino Projects

Folder and file names are German; the descriptions below explain what each project does.

## Projects

| Project | Board | Description |
|---|---|---|
| [Reaction time](Reaktionszeit/Reaktionszeit.txt) | Arduino Uno (AVR) | An LED lights up randomly after 1–3 s, a button stops the timer, the result in ms is printed to the Serial Monitor. |
| [Temperature sensor](Temperatursensor/Temperatursensor.txt) | Arduino Uno (AVR) | A DHT11 measures the temperature; traffic-light LEDs show green (up to 25 °C), yellow (above 25 °C) or red (above 26 °C). |
| [Analog readout](AnalogeDatenauslesung/AnalogeDatenauslesung.txt) | Arduino Uno (AVR) | Reads a potentiometer on A3; three LEDs (red, yellow, blue) each show one third of the value range, the raw value goes to the Serial Monitor. |
| [Resistance measurement](Wiederstandsmessen/Wiederstandsmessen.txt) | Arduino Uno (AVR) | Placeholder: currently only switches on the built-in LED. |
| [LaptopKuehlung_Sensor](LaptopKuehlung_Sensor/LaptopKuehlung_Sensor.ino) | Arduino Uno (AVR) | Laptop cooling, sensor: a DHT22 measures every 5 s, LED traffic light, data goes via USB to the bridge. |
| [LaptopKuehlung_Luefter](LaptopKuehlung_Luefter/LaptopKuehlung_Luefter.ino) | Arduino Uno (AVR) | Laptop cooling, fan: switches based on the temperature (hysteresis, fail-safe) and receives the data via USB from the bridge. |
| [LaptopKuehlung_Bridge](LaptopKuehlung_Bridge/bridge.py) | Laptop (Python + Docker) | Laptop cooling, bridge: connects both Unos (USB) to the Mosquitto MQTT broker. |

## Dependencies

### Board core

| Core | Version | For | Installation |
|---|---|---|---|
| Arduino AVR Boards | 1.8.8 | all sketches | Included with the Arduino IDE |

In the IDE, select "Arduino Uno" under **Tools → Board** and set the COM port.

### Libraries (`libraries/`)

| Library | Version | For | Note |
|---|---|---|---|
| DHT sensor library (Adafruit) | 1.4.7 | Temperature sensor, LaptopKuehlung_Sensor | |
| Adafruit Unified Sensor | 1.1.15 | Temperature sensor, LaptopKuehlung_Sensor | Dependency of the DHT sensor library |

Both can be installed via **Tools → Manage Libraries**. The `libraries/` folder is excluded from the repository via `.gitignore` and has to be reinstalled on a new machine. Only these two libraries should live in `libraries/`.

### Laptop (for the bridge)

| What | Version | Installation |
|---|---|---|
| Python | 3.11 (tested) | |
| paho-mqtt, pyserial | see [requirements.txt](LaptopKuehlung_Bridge/requirements.txt) | `python -m pip install --user -r LaptopKuehlung_Bridge/requirements.txt` |
| Docker Desktop | | for the broker `eclipse-mosquitto:2` |

## Laptop cooling

The Uno has no Wi-Fi, so both Unos are connected to the laptop via USB. The bridge (`bridge.py`) reads both USB ports and talks to the MQTT broker. The Unos are connected only through MQTT, not directly.

```
Sensor Uno (COM7) ──USB──┐                              ┌──USB── Fan Uno (COM3)
  DHT22 + traffic light  ▼                              ▼          Relay + 12 V fan
                    bridge.py  ◄──── MQTT ────►  Mosquitto (Docker, 127.0.0.1:1883)
                         │
                         └──► Console + CSV (LaptopKuehlung_Bridge/logs/) + laptop/log
```

Flow of one measurement:

1. The sensor Uno sends `@D|12|temp=27.3&hum=41*..` via USB.
2. The bridge publishes `laptop/temperatur` and only **then** acknowledges to the sensor Uno with `@A|12|*..`. If the broker cannot keep up, it replies `@N|12|grund=mqtt_offline*..`.
3. The bridge receives `laptop/temperatur` back from the broker and sends `@T|..|temp=27.3&hum=41*..` to the fan Uno.
4. The fan Uno acknowledges, regulates and reports its state (`@S|..|fan=ON&mode=AUTO&failsafe=0*..`). The bridge publishes it to `laptop/luefter/...`.

### Getting started

1. Start Docker Desktop, then the broker:
   ```bash
   docker compose -f LaptopKuehlung_Bridge/docker-compose.yml up -d
   ```
2. Upload `LaptopKuehlung_Sensor` to the Uno on **COM7** and `LaptopKuehlung_Luefter` to the Uno on **COM3**.
3. Close the Arduino IDE, or at least the Serial Monitor (a port can only be opened by one program at a time).
4. Start the bridge (defaults: sensor COM7, fan COM3, broker 127.0.0.1:1883):
   ```bash
   python LaptopKuehlung_Bridge/bridge.py
   ```
   Other ports: `--sensor COM8 --luefter COM5`. Raw frames on the console: `--debug`. Stop with Ctrl+C.

When the ports are opened, the bridge restarts both Unos; they then report their boot information.

### Bridge start-up diagnostics

On start-up the bridge logs:

| Event | Content |
|---|---|
| `bridge_start` | Version, Python, system, log folder |
| `wlan_verbunden` | Adapter, state, SSID, band, channel, signal, authentication |
| `wlan_verfuegbar` | Every visible Wi-Fi network with band, signal, authentication |
| `wlan_ziel` | Broker address and whether Wi-Fi is needed to reach it |
| `com_port`, `com_zuordnung` | All COM ports (description, USB ID, serial number) and whether the ports for sensor and fan are present |
| `mqtt_connect`, `broker_erreichbar` | Broker address, TCP test with response time |
| `abbruch` | Broker unreachable after **2 attempts**: the bridge exits (exit code 2) |

A busy COM port (e.g. an open Serial Monitor) does not abort: the bridge reports `port_unavailable` and retries every 5 s.

### Wiring

**Sensor Uno (COM7):**

| Component | Connection |
|---|---|
| DHT22 module | `+ → 5V`, `OUT → D2`, `− → GND` (bare 4-pin sensor: 10 kΩ from DATA to 5V) |
| LED red / yellow / green | `D13` / `D12` / `D11`, approx. 220 Ω each, short leg to GND |

| LED | Meaning |
|---|---|
| green | Temperature ≤ 25.0 °C |
| yellow | Temperature > 25.0 °C |
| red | Temperature > 26.0 °C |
| red blinking | Sensor error (DHT22 returns no value) |

**Fan Uno (COM3):**

| Component | Connection |
|---|---|
| Relay module (high level trigger) | `IN → D3`, `VCC → 5V`, `GND → GND` |
| 12 V fan | through the relay contact (`COM`/`NO`) to its own 12 V supply |

If the relay switches on LOW ("low level trigger"), set `FAN_ACTIVE_LOW = true` in the sketch. There are **no** wires between the two Unos.

### Messages over USB

Each USB line (115200 baud) carries log lines and data frames mixed together. Data frames start with `@`:

```
@<TYP>|<NR>|<key=wert&key=wert>*<PRUEFSUMME>
```

| Part | Meaning |
|---|---|
| `TYP` (type) | `H` hello, `D` data, `S` status/state, `T` temperature, `C` command, `A` acknowledgement, `N` rejection |
| `NR` (number) | Sequence number 1–65535 per sender; `A`/`N` repeat the number of the message they answer |
| `PRUEFSUMME` (checksum) | XOR of all characters between `@` and `*`, two hex digits |

| Direction | Frames |
|---|---|
| Sensor → bridge | `H` (dev=sensor), `D` (temp, hum), `S` (sensor=ok/error) |
| Bridge → sensor | `A`, `N` (grund=mqtt_offline, checksum, …) |
| Bridge → fan | `H` (dev=bridge), `T` (temp, hum), `S` (sensor=ok/error/offline), `C` (mode=ON/OFF/AUTO) |
| Fan → bridge | `H` (dev=luefter), `S` (fan, mode, failsafe), `A`, `N` |

The fan Uno checks the bridge's sequence numbers and reports gaps (`nr_luecke`) and duplicates.

### MQTT topics

| Topic | From | Content |
|---|---|---|
| `laptop/temperatur` | Bridge (sensor) | Temperature in °C, e.g. `27.3` (retained) |
| `laptop/luftfeuchte` | Bridge (sensor) | Humidity in % (retained) |
| `laptop/sensor/status` | Bridge (sensor) | `ok`, `error` or `offline` (sensor Uno port gone) (retained) |
| `laptop/luefter/set` | you | `ON`, `OFF` or `AUTO` |
| `laptop/luefter/status` | Bridge (fan) | `ON` / `OFF` (retained) |
| `laptop/luefter/mode` | Bridge (fan) | `AUTO` / `ON` / `OFF` (retained) |
| `laptop/luefter/failsafe` | Bridge (fan) | `ON` / `OFF` (retained) |
| `laptop/sensor/stats`, `laptop/luefter/stats` | Bridge | Summary as JSON, every minute |
| `laptop/bridge/status` | Bridge | `online` / `offline` (last will) |
| `laptop/log` | Bridge | Every log line as JSON |

On start-up the bridge does not forward old (retained) temperatures to the fan, otherwise the fan would treat them as current.

### Fan behaviour

- **AUTO** (default): ON from `TEMP_ON` (23.0 °C), OFF below `TEMP_OFF` (22.0 °C). In between, the state stays as it is.
- **Fail-safe**, the fan turns ON:
  - immediately on sensor status `error` or `offline`,
  - when no temperature arrives for 60 s (bridge or sensor gone).
- **MQTT:** `laptop/luefter/set` with `ON`, `OFF` or `AUTO`. Without the bridge, type `ON`, `OFF`, `AUTO` or `STATUS` into the Serial Monitor (115200 baud).

Thresholds and timings are at the top of the sketch under "Einstellungen" (settings).

### Logging

All three parts log in the same format:

```
zeit_ms;geraet;level;ereignis;key=wert,key=wert
```

`level` is `DEBUG` (raw frames), `INFO`, `WARN` or `ERROR`. The bridge prepends date and time and writes everything to `LaptopKuehlung_Bridge/logs/laptopkuehlung_<datum>.csv` (columns `datum;zeit_ms;geraet;level;ereignis;details`). Example:

```
2026-11-11 12:50:05;5012;sensor;INFO;send;typ=D,nr=3,link=online
2026-11-11 12:50:05;5013;sensor;INFO;measure;temp=27.3,hum=41
2026-11-11 12:50:05;31842;bridge;INFO;mqtt_publish;quelle=sensor,nr=3,inhalt=laptop/temperatur=27.3
2026-11-11 12:50:05;5030;sensor;INFO;send_ok;nr=3,rtt_ms=17,link=online
2026-11-11 12:50:05;6120;luefter;INFO;temp;nr=4,temp=27.3,hum=41
2026-11-11 12:50:05;6121;luefter;INFO;fan;zustand=ON,grund=temp_hoch,temp=27.3,an_ab=23.0,aus_unter=22.0
```

**Sending with status:** The sensor Uno and the fan Uno log the connection status before each send (`send`) and the result afterwards (`send_ok`, `send_nak`, `send_timeout`, `send_skipped`):

| Connection status | Meaning |
|---|---|
| `unbekannt` (unknown) | right after start-up |
| `online` | the last message was acknowledged |
| `unsicher` (uncertain) | the last attempt failed, retrying |
| `offline` | 2 failures in a row: **no more sending** (`send_skipped`) until the Uno restarts. If the bridge restarts, it resets the Unos automatically. |

Every minute all Unos write a summary (`stats`): sent, acknowledged, timeouts, rejected, skipped, gaps, duplicates, fan state, last temperature.

### Testing without the sensor

Run only the broker, the bridge and the fan Uno, and publish values yourself (inside the broker container):

```bash
docker exec laptopkuehlung-mosquitto mosquitto_pub -t laptop/temperatur -m 27.5
```

```bash
docker exec laptopkuehlung-mosquitto mosquitto_pub -t laptop/luefter/set -m OFF
```

```bash
docker exec laptopkuehlung-mosquitto mosquitto_sub -t "laptop/#" -v
```
