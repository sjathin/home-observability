# Home Observability Platform

A personal, end-to-end home monitoring platform for a Berlin flat + garden — built as
a software project, with off-the-shelf sensors where sensible and a **self-built
air/climate monitor** read directly by the Raspberry Pi.

## The three jobs

1. **Metrics** — see the numbers and trends across energy, air, climate, and garden.
2. **Money** — what things cost in €, and *where* I can save (vampire draw, big consumers).
3. **Attention** — one view of what needs me now: water the garden, ventilate, a device
   left on, poor air, or a safety event.

## What it watches

- **Energy** — per-device cost, "left on" alerts, washing-machine-done, standby waste (Shelly plugs)
- **Air** — indoor PM2.5 / PM10 (self-built SPS30)
- **Climate / mold** — CO₂ + humidity + temperature, "ventilate now" nudge (self-built SCD41)
- **Garden** — morning water/skip recommendation (weather API) + soil-moisture node
- **Safety** — smoke, water leak, heating failure, as away-from-home phone alerts (Shelly)

## How it works

Every device lands on one **MQTT bus**; a **writer** stores readings in a time-series
DB; a **rules engine** turns data into alerts, the Attention list, and € savings; a
**dashboard** and **phone alerts** read from there. All local on a Raspberry Pi 5;
GCP is an optional later add-on. See [`docs/architecture.md`](docs/architecture.md).

## Status

Planning. Requirements and architecture are drafted; hardware not yet ordered;
no code yet.

- Requirements → [`docs/requirements.md`](docs/requirements.md)
- Architecture + diagram → [`docs/architecture.md`](docs/architecture.md)

## Hardware (summary)

Raspberry Pi 5 (8 GB) + NVMe SSD hub · self-built SCD41 + SPS30 air/climate monitor ·
3× Shelly Plug S · Shelly Plus Smoke + Shelly Flood · ESP32 garden node. Full list and
rationale in the requirements doc. Rough total €400–450 (phaseable from ~€215).
