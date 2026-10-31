# Agent notes — home-observability

Working notes for AI agents on this repo. Keep the formal docs (`docs/requirements.md`,
`docs/architecture.md`) lean; put rationale, context, and session continuity here.

## What this is
Personal, interest-driven home observability platform for a Berlin flat + garden.
Software-first. Full design lives in `docs/`.

## Status
Planning complete; **no code yet**; hardware not yet ordered. Next step: the software
build — start with `infra/docker-compose.yml` (MQTT broker + DB + Grafana) and the
`sensor-reader` service, which is testable on the Pi before the Shelly devices arrive.

## Core decisions (condensed)
- **Fully local** on a Raspberry Pi 5 (8 GB) + NVMe SSD. No cloud. No Home Assistant.
- **One MQTT bus** (Mosquitto). Services: `sensor-reader`, `weather-poller`, `writer`,
  `rules-engine`, `api` — all **Python**. (`writer` = optional later Rust port.)
- **Self-built air/climate monitor:** SCD41 (CO₂/temp/humidity) + SPS30 (PM) on the Pi's
  I²C, read in Python every ~5 s. No embedded; ESPHome only for the ESP32 garden node.
- **Buy the rest:** 3× Shelly Plug S (energy), Shelly Plus Smoke (EN 14604), Shelly Flood
  — one Shelly API, WiFi, no hub.
- **Dashboards:** Grafana (Metrics + Money), provisioned as code. **Client:** Flutter
  (iOS/Android/web) — Attention tab (custom) + Dashboards tab (embedded Grafana).
  **Push:** Web Push (VAPID) to the PWA; needs HTTPS and, on iPhone, home-screen install.
  App open = REST + WebSocket to `api`. PWA-first, app stores optional.
- **Retention:** 2 years, tiered (raw 60–90 d, 1–5 min ~1 y, hourly/daily 2 y).

## North-star (everything ladders to these)
1. **Metrics** · 2. **Money** (where to save) · 3. **Attention** (what needs me now)

## Working style & constraints
- No embedded C/C++. No cloud. No Home Assistant.
- Requirements-first; acceptance criteria are **loose/evolving**; decide one thing at a
  time; keep scope bounded (YAGNI). Diagrams in **Mermaid**.
- Out of scope v1: automatic watering, main electricity meter, floor-heating power.

## Repo / git ops
- Public repo: https://github.com/sjathin/home-observability (local: `~/Github/home-observability`).
- `gh`/`git push` work **only after `unset GITHUB_TOKEN`** (a stale invalid `GITHUB_TOKEN`
  env var overrides the keyring login).
- Don't push without the user's go-ahead. Use conventional-commit messages.
