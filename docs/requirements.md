# Home Observability Platform — Requirements

> Personal, interest-driven project. Berlin flat + garden. Approach: software-first,
> off-the-shelf sensors, no embedded programming. Requirements are being built
> **use cases first, tech later**, one section at a time.

Status legend: ✅ in scope v1 · 🟡 proposed (needs your call) · ⛔ out of scope v1

---

## Section 1 — Use Cases (DRAFT, awaiting approval)

Each use case is phrased as: *as the person living here, I want ... so that ...*,
with the concrete action it drives. Measurable acceptance criteria come in the next
section once this list is agreed.

### Energy
- **UC-E1** ✅ See per-device and total electricity cost in € (day / week / month),
  so I know what each appliance actually costs me.
- **UC-E2** ✅ Get an alert when the washing machine / TV / workstation is drawing
  power when it shouldn't (left on), so I can act while away.
- **UC-E3** ✅ Know when the washing machine finishes (power drops to idle),
  so laundry doesn't sit forgotten.
- **UC-E4** ✅ Spot standby / vampire draw and quantify the € I'd save by cutting it.

### Garden
- **UC-G1** ✅ Each morning, get a "water today / skip" recommendation based on recent
  rain + forecast (DWD / Open-Meteo), so I don't over- or under-water.
- **UC-G2** ✅ Get notified when a garden soil-moisture node drops below a threshold
  during a dry spell. (Needs an ESP32 + soil sensor outdoors.)

### Air / Cleanliness
- **UC-A1** ✅ See indoor particulate levels (PM2.5 / PM10) and trends, and get notified
  when air quality is poor, so I know to ventilate or run a purifier.

### Climate / Ventilation
- **UC-V1** ✅ Get a "ventilate now" nudge when indoor humidity + CO₂ + temperature
  indicate mold risk or stale air (Berlin-rental relevant).

### Safety (away-from-home alerts)
- **UC-S1** ✅ Immediate phone alert when the smoke alarm triggers, even when away.
- **UC-S2** ✅ Immediate alert on water-leak detection (under sink / behind washing machine).
- **UC-S3** ✅ Alert if heating appears to have failed — indoor temp dropping abnormally
  vs. outdoor temp in winter — while away.

### Platform (cross-cutting, confirm expectations)
- **UC-P1** ✅ Keep historical data so I can look at trends over weeks/months.
- **UC-P2** ✅ One dashboard to see everything at a glance (at home and remotely).
- **UC-P3** ✅ Alerts reach me on my phone through one channel (to be chosen later).

### Out of scope for v1
- ⛔ Automatic watering (actuation) · ⛔ main electricity meter · ⛔ floor-heating power.


---

## Section 2 — Acceptance Criteria (loose, evolving targets)

> These are **targets, not gates** — expected to change as a personal project grows.
> Numbers are tunable defaults. Everything ladders up to the three jobs of the platform:
>
> 1. **Metrics** — see the numbers and trends.
> 2. **Money** — how much, and *where* I can save.
> 3. **Attention** — what needs me to act right now (water, ventilate/clean, device left on, safety).

### Energy

**UC-E1 — Cost per device in €**
- [ ] Energy (kWh) for each monitored device is recorded continuously, with data
      gaps under 5 min not losing a reading (catch-up on reconnect).
- [ ] Cost is derived from a configurable tariff (€/kWh); changing the tariff
      re-computes displayed costs.
- [ ] Dashboard shows per-device **and** total cost for day / week / month.
- [ ] Reported energy is within the plug's own metering tolerance (≈ ±1%).

**UC-E2 — "Left on" alert**
- [ ] "On when it shouldn't be" = power above a per-device idle threshold for longer
      than a configurable duration (default 30 min), optionally only outside
      expected hours.
- [ ] Alert reaches my phone within 2 min of the condition being met.
- [ ] Alert names the device and its current draw (actionable while away).
- [ ] One alert per event with a cooldown (default 60 min) — no spam.

**UC-E3 — Washing-machine-done notice**
- [ ] Cycle end = power drops below the idle threshold and stays there for a
      sustained window (default 5 min) after a period of active draw.
- [ ] Notification arrives within 5 min of cycle completion.
- [ ] Mid-cycle pauses do not trigger a false "done" (the sustained-low window
      must distinguish them).

**UC-E4 — Vampire / standby draw**
- [ ] Devices with continuous standby draw over a trailing period (default 7 days)
      are identified.
- [ ] Report shows, per device, estimated annual kWh and € if left as-is.
- [ ] Surfaced in the dashboard/report (not a push alert).

### Garden
**UC-G1 — Morning water/skip recommendation**
- [ ] Pulls recent rainfall + forecast (DWD / Open-Meteo) for Berlin daily.
- [ ] Produces a plain "water today" / "skip" call each morning, with the reason
      (e.g. "35 mm rain in last 48 h").
- [ ] Shows up in the **Attention** view, not buried in a chart.

**UC-G2 — Soil-moisture alert**
- [ ] Soil moisture from an outdoor node is tracked (metric + trend).
- [ ] Notifies when moisture stays below a configurable threshold during a dry spell.

### Air / Cleanliness
**UC-A1 — Indoor particulates**
- [ ] PM2.5 / PM10 recorded with trend over time (metric).
- [ ] Flags **Attention** ("ventilate / run purifier") when levels exceed a
      configurable threshold for a sustained window.

### Climate / Ventilation
**UC-V1 — Ventilate-now nudge**
- [ ] Humidity + CO₂ + temperature tracked per room (metric + trend).
- [ ] A simple mold-risk / stale-air signal raises an **Attention** nudge to ventilate.

### Safety (urgent, away-from-home)
**UC-S1/S2/S3 — Smoke · water leak · heating failure**
- [ ] Each raises an immediate phone alert (urgent tier, within ~1 min of detection).
- [ ] Alert says what and where, so it's actionable remotely.
- [ ] Heating-failure uses indoor temp falling abnormally vs. outdoor (Open-Meteo) in winter.
- [ ] Safety alerts bypass any quiet-hours / cooldown rules.

### Platform — the three views
**UC-P1 — History & trends**
- [ ] Readings retained long enough to see weeks/months of trend (metric).

**UC-P2 — One dashboard, three lenses**
- [ ] **Metrics** view: current readings + trends across all modules.
- [ ] **Money** view: live/periodic € cost, and a "where I can save" list (vampire draw,
      biggest consumers) with estimated savings.
- [ ] **Attention** view: a single aggregated list of everything that needs me now —
      water, ventilate/clean, device left on, poor air, plus any active safety alert.

**UC-P3 — Alerts to phone**
- [ ] Actionable items reach my phone through one channel (chosen later).
- [ ] Two tiers: urgent (safety) vs. normal (everything else), with quiet-hours on normal.


---

## Section 3 — Tech

### 3A. Physical layer (devices → use cases)

> Approach: BUILD the air/climate monitor from raw I²C sensors read by the Pi in
> Python (Path A, no embedded). BUY the rest via open APIs — Shelly (energy + safety,
> one API, WiFi, no hub) and a weather API. Prices ≈ Berlin retail, for planning.

**1 — Hub (the platform)**

| Item | ~€ | Why |
|------|----|-----|
| Raspberry Pi 5 **8 GB** | 85 | Always-on host for broker + DB + dashboard + Python + light ML. 8 GB = headroom (see note). |
| Official 27 W USB-C PSU | 13 | Pi 5 needs the real PSU to drive the SSD reliably. |
| Official **M.2 HAT+** | 13 | PCIe adapter for the SSD. |
| **NVMe SSD 256 GB** (2230/2242, M-key) | 35 | microSD wears out under 24/7 DB writes; SSD is the reliable choice. |
| Active cooler **or** official case w/ fan | 6–12 | 24/7 operation needs cooling. |
| microSD 32 GB | 8 | Initial imaging only. |

**2 — Self-built air/climate monitor (Path A: I²C → Pi → Python)**

| Item | Covers | ~€ | Why |
|------|--------|----|-----|
| Sensirion **SCD41** (Qwiic/STEMMA QT) | V1 + feeds S3 | 30 | CO₂ **and** temp + humidity in one chip. |
| Sensirion **SPS30** (PM) | A1 | 40 | PM1/2.5/4/10. Budget swap: Plantower PMS5003 (~€18, UART). |
| SparkFun **Qwiic pHAT** for Pi + 2 cables | — | 12 | Solderless plug-in wiring to the Pi's I²C bus. |
| Small enclosure (optional) | — | 10 | Tidy the build. |

**3 — Energy**

| Item | Covers | ~€ | Why |
|------|--------|----|-----|
| **Shelly Plug S Gen3** ×3 (3-pack) | E1–E4 | 55 | Washing machine, TV, workstation. Meters W/kWh, Shelly API, no hub. |

**4 — Safety (all Shelly WiFi, same API, no hub)**

| Item | Covers | ~€ | Why |
|------|--------|----|-----|
| **Shelly Plus Smoke** ×1 | S1 | 30–40 | WiFi smoke alarm, **EN 14604** (legal DE Rauchmelder). |
| **Shelly Flood** ×1–2 | S2 | 26 ea | WiFi water-leak + temperature. Under sink / behind washer. |
| *(heating failure S3)* | S3 | 0 | No hardware — SCD41 indoor temp vs. free outdoor weather. |

**5 — Garden** *(Phase 2 — outdoor power is the only tricky part)*

| Item | Covers | ~€ | Why |
|------|--------|----|-----|
| ESP32 + capacitive soil sensor (ESPHome→MQTT) | G2 | 15 | Outdoor node; needs enclosure + power. The only "flashing" task. |
| Weather data (DWD / Open-Meteo) | G1, S3 | free | No hardware. |

**Rough totals:** full build ≈ **€400–450**. Phased start (Hub + air monitor only) ≈ **€215**,
then add energy (~€55), safety (~€80), garden (~€30) as you go.

**Coverage check:** every use case maps to a device or data source. Energy and safety
share one Shelly API; air/climate is your own sensor code; garden reuses weather + one node.
