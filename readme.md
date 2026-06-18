# 🏠 Smart Room AQI Monitor

> A real-time indoor air quality monitoring system built with Arduino Uno, visualized through Grafana dashboards.

---

## 📸 Overview

This project continuously monitors indoor environmental conditions — temperature, humidity, particulate matter, gas levels, and ambient noise — and visualizes the data on a live Grafana dashboard backed by InfluxDB.

---

## ✨ Features

| Feature | Description |
|--------|-------------|
| 🌡️ Temperature & Humidity | DHT11 sensor with offset calibration |
| 💨 Air Quality Index | EPA-standard AQI calculated from PM2.5 readings |
| 🌫️ Particulate Matter | PM2.5 & PM10 via SDS011 laser dust sensor |
| 🧪 Gas Detection | MQ135 VOC / CO₂ indicator |
| 🔊 Noise Monitoring | MAX9814 microphone amplifier spike detection |
| 💡 Visual Alerts | Green / Yellow / Red LED status indicators |
| 🔔 Hourly Chime | Non-blocking buzzer notification every hour (3x at 12:00 & 17:00) |
| 🌬️ Auto Ventilation | Fan relay triggered every 3 minutes for 5 seconds |
| 🕐 Real-Time Clock | DS1302 RTC module with battery backup |
| 🐕 Watchdog Timer | 8-second hardware watchdog auto-recovers from system hangs |
| 📊 Live Dashboard | Grafana + InfluxDB time-series visualization |
| 📡 Data Pipeline | Arduino → Node-RED → InfluxDB → Grafana |

---

## 🔧 Hardware

### Components

| Component | Model | Purpose |
|-----------|-------|---------|
| Microcontroller | Arduino Uno | Main processing unit |
| Temperature/Humidity | DHT11 | Ambient readings |
| Dust Sensor | SDS011 | PM2.5 & PM10 measurement |
| Gas Sensor | MQ135 | VOC / CO₂ detection |
| Microphone | MAX9814 | Noise level monitoring |
| Clock Module | DS1302 | Real-time clock with battery |
| Display | 16×2 I2C LCD (0x27) | Local data display |
| Relay | 5V Single Channel | Fan control |
| Fan | 5V DC Mini Fan | Auto ventilation |
| LEDs | Red / Yellow / Green | Alert indicators |
| Buzzer | Active Buzzer | Audio alerts |

### 📌 Pin Configuration

```
D2  → DHT11 DATA          (10kΩ pull-up required)
D3  → SDS011 RX           (SoftwareSerial)
D4  → SDS011 TX           (SoftwareSerial)
D5  → DS1302 CLK
D6  → DS1302 DAT
D7  → DS1302 RST
D8  → Buzzer              (Active, 5V)
D9  → Relay IN            (LOW active)
D11 → Green LED           (220Ω resistor)
D12 → Yellow LED          (220Ω resistor)
D13 → Red LED             (220Ω resistor)
A0  → MQ135 AOUT
A1  → MAX9814 OUT
A4  → LCD SDA             (I2C)
A5  → LCD SCL             (I2C)
```

---

## 🏗️ System Architecture

```
┌─────────────────────────────────────────────────────────┐
│                     Arduino Uno                          │
│                                                         │
│  DHT11 ──┐                                              │
│  SDS011 ─┤                                              │
│  MQ135 ──┼──► Processing ──► LCD Display                │
│  MAX9814 ┤                ──► LED Alerts                 │
│  DS1302 ─┘                ──► Buzzer / Relay            │
│                                │                        │
│                         Serial (115200)                 │
└─────────────────────────────────┬───────────────────────┘
                                  │ JSON every 60s
                                  ▼
                         ┌────────────────┐
                         │   Raspberry Pi  │
                         │                │
                         │   Node-RED     │──► Parse & Route
                         └───────┬────────┘
                                 │
                                 ▼
                         ┌────────────────┐
                         │   InfluxDB 2.x │
                         │  (Docker)      │
                         └───────┬────────┘
                                 │
                                 ▼
                         ┌────────────────┐
                         │    Grafana     │
                         │  (Docker)      │
                         └────────────────┘
```

---

## 📦 Software Dependencies

### Arduino Libraries

```
LiquidCrystal_I2C    — Frank de Brabander
DHT sensor library   — Adafruit
DS1302               — Matt Sparks
SoftwareSerial       — Arduino built-in
avr/wdt.h            — AVR built-in (Watchdog)
```

### Server Stack

| Service | Version | Purpose |
|---------|---------|---------|
| Node-RED | Latest | Serial data ingestion & routing |
| InfluxDB | 2.7 | Time-series database |
| Grafana | Latest | Data visualization |

All services run as Docker containers.

---

## 📊 Data Format

The Arduino sends a JSON payload once per minute over Serial (115200 baud):

```json
{
  "temp": 22.4,
  "hum": 45.2,
  "noise_rel": 12,
  "ppm": 18,
  "pm25": 7.3,
  "pm10": 11.8,
  "aqi": 31
}
```

| Field | Unit | Description |
|-------|------|-------------|
| `temp` | °C | Temperature (DHT11 offset applied) |
| `hum` | % | Relative humidity |
| `noise_rel` | count | Spike count per minute (MAX9814) |
| `ppm` | raw ADC | MQ135 gas indicator |
| `pm25` | µg/m³ | Fine particulate matter |
| `pm10` | µg/m³ | Coarse particulate matter |
| `aqi` | 0–500 | EPA Air Quality Index |

---

## 🚦 AQI Thresholds

| AQI Range | Category | LED |
|-----------|----------|-----|
| 0 – 50 | Good | 🟢 Green |
| 51 – 100 | Moderate | 🟡 Yellow |
| 101 – 150 | Unhealthy for Sensitive Groups | 🟡 Yellow |
| 151 – 200 | Unhealthy | 🔴 Red |
| 201 – 300 | Very Unhealthy | 🔴 Red |
| 301+ | Hazardous | 🔴 Red |

---

## ⏰ Alert Schedule

```
Every hour  (XX:00)  → 1× 500ms beep + all LEDs flash  (non-blocking)
12:00 & 17:00        → 3× 500ms beeps + all LEDs flash  (non-blocking)
ALARM state          → 1× 500ms beep on trigger, Red LED stays on
Fan                  → Runs 5 seconds every 3 minutes
Heartbeat            → "ALIVE HH:MM:SS" printed to Serial every 10 seconds
```

---

## 🖥️ LCD Display

The 16×2 LCD rotates through three screens every 2 seconds:

```
Screen 1:                 Screen 2:              Screen 3:
┌────────────────┐        ┌────────────────┐     ┌────────────────┐
│T:22.4C H: 45%  │        │   14:32:05     │     │PM2.5:  7.3 ug/m3│
│AQI:31  FAN:OFF │        │                │     │PM10:  11.8 ug/m3│
└────────────────┘        └────────────────┘     └────────────────┘
```

> LCD updates are paused during SDS011 measurement and fan operation to prevent I2C interference.

---

## 🔌 Node-RED Flow

The Node-RED function node parses incoming serial data and routes it to InfluxDB:

```javascript
var str = msg.payload.trim();
if (str.charAt(0) !== '{') return null;

try {
    var data = JSON.parse(str);
    msg.payload = [[{
        temp:      data.temp,
        hum:       data.hum,
        noise_rel: data.noise_rel,
        ppm:       data.ppm,
        pm25:      data.pm25,
        pm10:      data.pm10,
        aqi:       data.aqi
    }], { measurement: "aqi" }];
    return msg;
} catch(e) {
    return null;
}
```

> Lines not starting with `{` (e.g. `ALIVE`, `RESET=`) are automatically discarded.

---

## 🐳 Docker Setup

```yaml
services:
  influxdb:
    image: influxdb:2.7
    ports:
      - "8086:8086"

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    environment:
      - GF_AUTH_ANONYMOUS_ENABLED=true
      - GF_AUTH_ANONYMOUS_ORG_ROLE=Viewer
      - GF_SECURITY_ALLOW_EMBEDDING=true

  nodered:
    image: nodered/node-red:latest
    ports:
      - "1880:1880"
    devices:
      - /dev/ttyUSB0:/dev/ttyUSB0
    group_add:
      - dialout
```

---

## 📈 Grafana Queries

### Temperature
```flux
from(bucket: "smart_room")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r._measurement == "aqi")
  |> filter(fn: (r) => r._field == "temp")
  |> aggregateWindow(every: v.windowPeriod, fn: mean, createEmpty: false)
```

### PM2.5 & PM10
```flux
from(bucket: "smart_room")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r._measurement == "aqi")
  |> filter(fn: (r) => r._field == "pm25" or r._field == "pm10")
  |> aggregateWindow(every: v.windowPeriod, fn: mean, createEmpty: false)
```

### AQI Gauge
```flux
from(bucket: "smart_room")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r._measurement == "aqi")
  |> filter(fn: (r) => r._field == "aqi")
  |> last()
```

---

## ⚠️ Known Limitations

- **MQ135** requires 24–48 hours warm-up for stable readings. Initial values may be inaccurate.
- **DHT11** has ±2°C and ±5% RH tolerance. A fixed offset is applied in firmware.
- **DS1302** low-cost clones may drift over time. Battery maintains time across power cycles.
- **SoftwareSerial + I2C** can cause interrupt conflicts on Arduino Uno. LCD and sensor reads are paused during SDS011 active phases and fan operation to mitigate this.
- **Watchdog timer** (8s) automatically recovers from system hangs. Reset cause is logged to Serial on boot (`RESET=1` power-on, `RESET=10` brownout, `RESET=100` watchdog).
- **Fan relay** triggers only when SDS011 is idle to prevent power spike interference.

---

## 📁 Project Structure

```
smart-room-aqi/
├── firmware/
│   └── smart_room_aqi.ino     # Arduino firmware v4.1
├── nodered/
│   └── flows.json             # Node-RED flow export
├── grafana/
│   └── dashboard.json         # Grafana dashboard export
├── docker-compose.yml         # Server stack
└── README.md
```

---

## 🙏 Credits

Built with ❤️ by **Eren DOGAN** | [Erenotrics](https://www.instagram.com/eren.dogan83/)

> *"Measuring the invisible, one sensor at a time."*
