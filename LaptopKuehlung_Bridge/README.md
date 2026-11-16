# LaptopKuehlung_Bridge (laptop cooling – bridge)

Connects the two Unos to the MQTT broker. The Unos have no Wi-Fi, so the bridge runs on the laptop.

```
Sensor Uno (COM7) ─USB─► bridge.py ◄─MQTT─► Mosquitto ◄─MQTT─► bridge.py ─USB─► Fan Uno (COM3)
```

## Requirements

- Python 3.11 and `python -m pip install --user -r requirements.txt`
- Docker Desktop

## Start

In the `LaptopKuehlung_Bridge` folder:

```bash
docker compose up -d
```

```bash
python bridge.py
```

Close the Arduino IDE or the Serial Monitor first, otherwise the ports are busy. Stop with Ctrl+C, stop the broker with `docker compose down`.

| Option | Default |
|---|---|
| `--sensor` | `COM7` |
| `--luefter` | `COM3` |
| `--broker` / `--port` | `127.0.0.1` / `1883` |
| `--debug` | also print raw frames to the console |

## Start-up diagnostics

On start-up the bridge logs Wi-Fi (connected and visible networks), all COM ports and whether the broker is reachable. If the broker is still unreachable after **2 attempts**, it exits with exit code 2. A busy or missing COM port is retried every 5 s.

## MQTT topics

| Topic | Content |
|---|---|
| `laptop/temperatur`, `laptop/luftfeuchte` | Measurements (retained) |
| `laptop/sensor/status` | `ok`, `error`, `offline` |
| `laptop/luefter/set` | Command `ON`, `OFF`, `AUTO` |
| `laptop/luefter/status`, `/mode`, `/failsafe` | State of the fan |
| `laptop/sensor/stats`, `laptop/luefter/stats` | Summary, every minute |
| `laptop/bridge/status` | `online` / `offline` |
| `laptop/log` | Every log line as JSON |

## Logs

Console and `logs/laptopkuehlung_<datum>.csv` with the columns `datum;zeit_ms;geraet;level;ereignis;details`.

## Testing without the sensor

```bash
docker exec laptopkuehlung-mosquitto mosquitto_pub -t laptop/temperatur -m 27.5
```

```bash
docker exec laptopkuehlung-mosquitto mosquitto_sub -t "laptop/#" -v
```
