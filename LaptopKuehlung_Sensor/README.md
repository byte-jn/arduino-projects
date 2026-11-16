# LaptopKuehlung_Sensor (laptop cooling – sensor)

Measures temperature and humidity every 5 s and sends them via USB to the [bridge](../LaptopKuehlung_Bridge/). An LED traffic light shows the temperature directly on the board.

## Hardware

| Component | Connection |
|---|---|
| Arduino Uno | USB to the laptop (default: COM7) |
| DHT22 module | `+ → 5V`, `OUT → D2`, `− → GND` |
| LED red / yellow / green | `D13` / `D12` / `D11`, approx. 220 Ω each, short leg to GND |

## Upload

Board **Arduino Uno**, port **COM7**. Libraries: *DHT sensor library* and *Adafruit Unified Sensor*.

## Behaviour

| LED | Meaning |
|---|---|
| green | ≤ 25.0 °C |
| yellow | > 25.0 °C |
| red | > 26.0 °C |
| red blinking | Sensor error |

- Every measurement is sent to the bridge as a frame `@D|<nr>|temp=27.3&hum=41*<checksum>`.
- The bridge only acknowledges once the MQTT broker has the message.
- After **2 failures in a row** the connection counts as `offline`: nothing is sent anymore until the Uno restarts. Measuring and the traffic light keep running.

## Log (USB, 115200 baud)

```
5012;sensor;INFO;send;typ=D,nr=3,link=online
5030;sensor;INFO;send_ok;nr=3,rtt_ms=17,link=online
5031;sensor;INFO;measure;temp=27.3,hum=41
```

Format: `zeit_ms;geraet;level;ereignis;details`. A summary (`stats`) follows every minute.

## Settings

At the top of the sketch: pins, traffic-light thresholds (`WARN_TEMP`, `BAD_TEMP`), measuring interval, `ACK_TIMEOUT_MS`, `MAX_FAILS`, `LOG_FRAMES`.
