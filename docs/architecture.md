# Architecture

> v1 design. Everything runs locally on the Raspberry Pi in Docker. No cloud — the
> platform is fully self-contained and keeps working without internet.

## System diagram

```mermaid
flowchart TB
    subgraph edge["Devices & data sources"]
        scd["SCD41 — CO2 · temp · humidity (self-built)"]
        sps["SPS30 — PM2.5 / PM10 (self-built)"]
        plugs["Shelly Plug S x3 — energy W/kWh"]
        smoke["Shelly Plus Smoke"]
        flood["Shelly Flood"]
        soil["ESP32 garden node — soil moisture (ESPHome)"]
        wx["Weather API — DWD / Open-Meteo"]
    end

    subgraph pi["Raspberry Pi 5 — Docker"]
        reader["sensor-reader — Python I2C loop (5s)"]
        wxpoll["weather-poller — Python (hourly)"]
        broker{{"MQTT broker — Mosquitto"}}
        writer["writer — MQTT to DB"]
        db[("Time-series DB — InfluxDB / TimescaleDB")]
        rules["rules + attention engine — thresholds · savings · ML"]
        dash["Dashboard — Metrics · Money · Attention"]
        alerter["alert dispatcher — urgent / normal"]
    end

    phone["Phone"]

    scd -->|I2C| reader
    sps -->|I2C| reader
    reader --> broker
    plugs -->|MQTT| broker
    smoke -->|MQTT| broker
    flood -->|MQTT| broker
    soil -->|MQTT| broker
    wx --> wxpoll --> broker

    broker --> writer --> db
    broker --> rules
    db --> rules
    rules --> db
    db --> dash
    rules --> alerter --> phone
```

## Data flow, in words

1. **Collect.** Two paths converge on one MQTT bus:
   - Self-built sensors (SCD41, SPS30) are wired to the Pi's I²C pins; a Python
     `sensor-reader` polls them every ~5 s and publishes readings to MQTT.
   - Shelly devices (plugs, smoke, flood) and the ESP32 garden node publish to the
     same broker over WiFi. A `weather-poller` fetches DWD/Open-Meteo and publishes too.
2. **Store.** A `writer` subscribes to all topics and appends to the time-series DB.
3. **Decide.** The `rules + attention engine` reads live MQTT (fast alerts) and the DB
   (trends/savings), and writes derived results back — the Attention list, € savings,
   washing-done, mold risk, heating-failure, watering recommendation.
4. **Show & notify.** The dashboard reads the DB and renders the three views
   (Metrics / Money / Attention). The alert dispatcher sends urgent (safety) and
   normal (everything else) notifications to the phone.

## Why this shape

- **One MQTT bus** = every device and service speaks the same way; adding a sensor
  later is just another topic. No per-device special-casing.
- **DB in the middle** = history and trends come for free; dashboard and ML read one store.
- **Rules engine separate from dashboard** = alerts fire even when no screen is open.
- **Local-first** = fully self-contained; no cloud dependency, keeps working offline.

## Data retention

Tiered, capped at **2 years** — enough for year-over-year seasonal comparison without
bloating the Pi's DB:

| Tier | Resolution | Kept for |
|------|-----------|----------|
| Raw | per-sample (~5 s) | 60–90 days |
| Mid | 1–5 min rollups | ~1 year |
| Long | hourly / daily rollups | 2 years |

Both InfluxDB and TimescaleDB implement this natively (retention policies +
continuous aggregates / downsampling tasks).

## Planned repo layout (code comes after this design is approved)

```
home-observability/
├── README.md
├── docs/
│   ├── requirements.md      # use cases + acceptance criteria
│   └── architecture.md      # this file
├── services/
│   ├── sensor-reader/       # Python I2C → MQTT
│   ├── weather-poller/      # API → MQTT
│   ├── writer/              # MQTT → DB
│   ├── rules-engine/        # alerts, attention, savings
│   └── dashboard/           # the three views
├── infra/
│   └── docker-compose.yml   # broker + DB + services
├── .env.example
└── .gitignore
```

## Open choices (decide during the software phase, not now)

- Time-series DB: **InfluxDB** vs **TimescaleDB (Postgres)**.
- Dashboard: **Grafana** (fast, prebuilt) vs **custom app** (full control of the 3 views).
- Phone channel: **ntfy** (self-hosted, simple) vs **Telegram bot** vs **Pushover**.
