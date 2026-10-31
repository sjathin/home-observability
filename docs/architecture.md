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
        api["api — serves Attention list + data to the app"]
        grafana["Grafana — Metrics · Money dashboards"]
        alerter["alert dispatcher — urgent / normal"]
    end

    subgraph clients["Clients — one Flutter codebase"]
        app["Flutter app — iOS · Android · web/PWA<br/>Attention tab + embedded Grafana tab"]
    end
    push["Web Push — Apple / Google push service"]

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
    db --> grafana
    rules --> api
    db --> api
    grafana -->|embedded webview| app
    api --> app
    rules --> alerter --> push --> app
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
4. **Show & notify.**
   - **Grafana** renders the Metrics + Money dashboards straight from the DB.
   - A small **`api`** serves the Attention list (and data) to the **Flutter app**, which
     has one codebase for iOS, Android, and web/PWA: an **Attention tab** (custom UI) and a
     **Dashboards tab** that embeds Grafana in a webview.
   - The **alert dispatcher** sends **Web Push** (VAPID, `pywebpush`) through Apple's and
     Google's push services to the installed PWA on both phones. Delivery is an outbound
     request from the Pi, so it works when the phone is away from home.

## Clients & distribution

- **One Flutter codebase → iOS + Android + web.** Covers both phones (iOS + Android) and a
  browser view from the same source.
- **Dashboards tab** = Grafana embedded via webview (don't rebuild charts).
  **Attention tab** = custom UI, the part worth building by hand.
- **App stores are optional.** The app ships as a **PWA** (Flutter web, hosted on the Pi,
  "add to home screen") with **Web Push** for notifications. No Apple Developer account.
- **Web Push requirements:**
  - The PWA must be served over **HTTPS** (service workers and push need a secure origin).
  - **iPhone:** iOS 16.4+, and push only works after "Add to Home Screen". Permission must be
    requested from a tap in the app.
  - A small JS **service worker** handles the `push` event and shows the notification.
  - **VAPID key pair** generated once; the private key lives in `.env`, never in the repo.
  - Subscriptions are stored on the Pi per device; drop them when the push service
    returns 404/410. Safety alerts are sent with `Urgency: high`.
- A native build (FCM + APNs) is only worth it if the app ever goes to the stores.

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
│   └── api/                 # serves Attention list + data to the app
├── app/                     # Flutter app (iOS · Android · web)
├── infra/
│   ├── docker-compose.yml   # broker + DB + Grafana + services
│   └── grafana/             # provisioned dashboards (Metrics · Money)
├── .env.example
└── .gitignore
```

## Decided

- **Backend language:** Python for all Pi services (official SCD41/SPS30 drivers +
  pandas/scikit-learn for the Money/ML work; perf is a non-issue at this volume).
  The `writer` may be ported to Rust later as an optional, self-contained exercise.
- **Dashboards:** Grafana (Metrics + Money), provisioned as code, embedded in the app.
- **App:** Flutter — one codebase for iOS + Android + web; Attention tab (custom) +
  Dashboards tab (embedded Grafana). PWA first; app stores optional.
- **Push:** Web Push (VAPID) to the PWA. No store account, no third-party messaging app.
- **App ↔ Pi (app open):** REST for loading data + WebSocket for live updates, served by `api`.

## Open choices (decide during the software phase, not now)

- Time-series DB: **InfluxDB** vs **TimescaleDB (Postgres)**.
