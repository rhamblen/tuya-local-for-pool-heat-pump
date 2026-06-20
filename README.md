# Pool Heat Pump Tile for Home Assistant

Local control and full inverter telemetry for an **OEM Tuya inverter pool heat
pump** (Tuya category `rs`), plus a clean Home Assistant dashboard tile — all over
your **own network**, no cloud dependency for the data that matters.

It gives you a verified **datapoint → entity** map for LocalTuya, a tile dashboard
grouped into Thermal / Performance / System, and docs aimed at non-technical
installers (and at Claude — see [Install](#install)).

---

## ⚠️ Requirement: LocalTuya / TinyTuya (not cloud Tuya)

Your unit speaks two languages, and only one is useful:

| Channel | What you get |
|---|---|
| **Tuya Cloud** (official HA integration) | Power, setpoint, mode — a thin climate entity |
| **Tuya LAN v3.3** (LocalTuya / TinyTuya) | **Everything** the MCU reports: compressor frequency & current, EXV, coil/discharge/heat-sink temps, relays, defrost — in real time, on your LAN |

The OEM only certified a subset of datapoints for the cloud; the inverter board
broadcasts the full set locally. So this project uses:

- **[TinyTuya](https://github.com/jasonacox/tinytuya)** — once, to extract the
  device **Local Key**, IP, and validate the DPs.
- **[LocalTuya (xZetsubou fork)](https://github.com/xZetsubou/hass-localtuya)** —
  the permanent HA integration that maps DPs to entities over the LAN. *(Use this
  maintained fork; the original `rospogrigio/localtuya` is stale.)*

---

## What you get

- Real-time control: power, setpoint, mode, silent mode.
- Full inverter telemetry mapped to typed HA entities (**25 of 26 DPs**).
- A tile dashboard separating **Thermal**, **Performance**, and **System status**.
- A documented, authoritative [datapoint map](docs/datapoints.md) decoded from the
  device's own Tuya data model — including the fault-code bit labels.

---

## Install

### Option A — Let Claude set it up (recommended)

If you use **Claude** (Claude Code, or the Home Assistant MCP), point it at this
repo and say:

> *"Set up the Tuya pool heat pump from this repo."*

Claude should read **[docs/ai-context.md](docs/ai-context.md)** first, then:
extract the Local Key with TinyTuya → install LocalTuya → map the DPs from
[docs/datapoints.md](docs/datapoints.md) → create the helper sensors → deploy the
dashboard. The AI context file carries the install order and the gotchas
(scaling, the no-outlet-temp limit, dashboard section framing).

### Option B — Manual

Follow **[docs/installation.md](docs/installation.md)**. In short: install
TinyTuya and run the wizard for your Local Key → install LocalTuya in HA → add the
device and map every DP from [docs/datapoints.md](docs/datapoints.md) → paste the
dashboard from [dashboard/pool_heat_pump_tile.yaml](dashboard/pool_heat_pump_tile.yaml).

---

## Datapoint map

The authoritative, annotated table lives in
**[docs/datapoints.md](docs/datapoints.md)** — all 26 DPs with translated names,
types, ranges, the analog input each sensor uses, scaling, and the fault-code
labels. Highlights:

- **Control:** DP1 power, DP106 setpoint, DP105 mode, DP117 silent.
- **Temps:** DP102 inlet water, DP124 ambient air, DP120 air coil, DP122 discharge,
  DP127 electronics heatsink.
- **Inverter:** DP125 frequency, DP126 current (**scale 0.1**), DP104 speed %,
  DP128 EXV, DP129 DC fan.
- **State:** DP118 actual heat/cool, DP130 defrost, DP134/135/136/139 relays.

---

## Sensor design notes

- **DP124 ambient is NOT the weather temperature** — it's the air at the unit's
  intake. Use a `weather.*` entity for real outdoor temp.
- **DP118 (actual mode) beats DP105 (requested mode)** for diagnostics — in `smart`
  mode the unit decides for itself.
- **Compressor frequency (DP125) is the efficiency signal** — low, steady Hz = sipping.
- **DP127 "RadTemp" is the electronics heatsink, not delivered heat.** See the full
  temperature explainer in [docs/datapoints.md](docs/datapoints.md#temperature-sensors--what-each-one-actually-is).

---

## Heat output & COP (proxy only)

Electrical input is exact if you have a power meter on the circuit. Heat **output**
is not — **this controller exposes only inlet water temp, no outlet/return**. For a
real figure you must add an outlet sensor (e.g. a DS18B20 on the return pipe) and a
flow rate; until then COP is a rough trend, not accounting. Details in
[docs/datapoints.md](docs/datapoints.md#heat-output--cop--hardware-limitation).

---

## Troubleshooting

Common issues — single local connection at a time, rotating Local Keys, scaling,
compressor start sequencing — are covered in
**[docs/troubleshooting.md](docs/troubleshooting.md)**.

---

## Repository layout

```text
.
├── README.md
├── CHANGELOG.md
├── LICENSE
├── dashboard/
│   └── pool_heat_pump_tile.yaml
└── docs/
    ├── ai-context.md       # instructions for Claude / AI assistants
    ├── datapoints.md       # authoritative DP map + fault codes + scaling
    ├── installation.md     # manual TinyTuya → LocalTuya → mapping → dashboard
    └── troubleshooting.md
```

---

## Compatibility

- **Home Assistant 2026.x+**
- **LocalTuya** (xZetsubou fork, HACS) — Tuya LAN protocol **v3.3** (try 3.4)
- **TinyTuya** ≥ 1.13 (Python 3.9+)

## Changelog

See [CHANGELOG.md](CHANGELOG.md).

## Licence

[MIT](LICENSE) — no warranty. Working on refrigeration equipment and mains wiring
is at your own risk.
