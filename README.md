# Pool Heat Pump Tile for Home Assistant

A complete, beginner-installable Home Assistant package for monitoring and
controlling an **OEM Tuya inverter pool heat pump** (Tuya device category `rs`)
over the **local network** — no cloud dependency for telemetry.

It gives you:

- A full, verified **datapoint (DP) → entity** map for LocalTuya
- A clean, tile-style **Lovelace dashboard** ("Pool Heat Pump Tile")
- Step-by-step **installation** for TinyTuya + LocalTuya
- **Sensor design notes** explaining what the inverter telemetry actually means
- An **advanced section** with COP / heat-output proxy models and defrost detection

> Treat this as a real industrial inverter monitoring system, not a toy thermostat.
> The interesting data — compressor frequency, EXV position, discharge temperature,
> coil temperature — only exists on the LAN protocol.

---

## What this project does

It turns the raw Tuya datapoints exposed by your heat pump's MCU/inverter board
into named, typed Home Assistant entities, then presents them in a dashboard that
separates **thermal**, **performance**, and **system status** at a glance.

You get real-time control (power, setpoint, mode, silent mode) plus deep inverter
telemetry that the Tuya cloud integration simply does not surface.

---

## Why LocalTuya / TinyTuya is required (cloud vs LAN)

Your unit speaks two different "languages":

| Channel | What you get | What you DON'T get |
|---|---|---|
| **Tuya Cloud** (official HA Tuya integration) | Power, setpoint, mode, a handful of "official" DPs the vendor chose to publish | Compressor frequency, current, EXV position, coil/discharge/heat-sink temps, relay states, defrost flag — i.e. everything useful for diagnostics |
| **Tuya LAN v3.3** (LocalTuya / TinyTuya) | **Every** datapoint the MCU reports, in real time, on your own network | — |

The OEM only certified a subset of datapoints for the cloud. The inverter board,
however, broadcasts the full set locally. **LocalTuya talks directly to the device
on your LAN**, so it sees all of it — and it keeps working if your internet is down.

- **TinyTuya** is used *once*, up front, to extract your device's **Local Key**,
  confirm its **IP**, and validate which DPs are live.
- **LocalTuya** is the permanent Home Assistant integration that maps those DPs to
  entities and handles live control + telemetry over the LAN.

---

## Device access requirements

To enable local communication you need three things:

1. **Device IP** (its address on your LAN — give it a DHCP reservation)
2. **Device ID** (the Tuya `gwId` / `devId`)
3. **Local Key** (the per-device LAN encryption key — rotates if you re-pair)

All three are obtained with the **TinyTuya wizard** (see
[docs/installation.md](docs/installation.md)).

---

## Quick start

1. **Get your Local Key** — install TinyTuya, run the wizard, note the IP / ID / Key.
   → [docs/installation.md](docs/installation.md#1-tinytuya)
2. **Install LocalTuya** via HACS in Home Assistant.
   → [docs/installation.md](docs/installation.md#2-localtuya)
3. **Add the device** in LocalTuya and **map every DP** from
   [docs/datapoints.md](docs/datapoints.md).
4. **Add the dashboard** from
   [dashboard/pool_heat_pump_tile.yaml](dashboard/pool_heat_pump_tile.yaml)
   (edit the entity IDs to match yours).
5. Read the [Sensor Design Notes](#sensor-design-notes) so the numbers mean something.

---

## Full datapoint map (summary)

The authoritative, annotated table lives in
[docs/datapoints.md](docs/datapoints.md). Short version:

### Control
| DP | Entity | Type | Notes |
|---|---|---|---|
| 1 | Power | switch | Main on/off |
| 106 | Set Temperature | number | -22 → 104 °C (range is the raw DP range; clamp to a sane pool range in the UI) |
| 105 | Mode | select | `smart` / `warm` / `cool` |
| 117 | Silent Mode | switch | Low-noise / night mode |

### Temperatures
| DP | Entity | Type |
|---|---|---|
| 102 | Water Inlet Temperature | sensor °C |
| 124 | Ambient Air Temperature (internal unit) | sensor °C |
| 120 | Evaporator / Coil Temperature | sensor °C |
| 122 | Compressor Discharge Temperature | sensor °C |
| 127 | Heat Sink Temperature | sensor °C |

### Inverter / performance
| DP | Entity | Type | Notes |
|---|---|---|---|
| 125 | Compressor Frequency | sensor Hz | Key efficiency signal |
| 126 | Compressor Current | sensor A | |
| 104 | Speed Percentage | sensor % | |
| 128 | EXV Position | sensor | 0–10000; may need `/100` |
| 129 | DC Fan Speed | sensor | RPM/PWM raw value |

### System state (binary sensors)
| DP | Entity |
|---|---|
| 130 | Defrost Active |
| 134 | Compressor Relay |
| 135 | Circulation Pump |
| 136 | 4-Way Valve |
| 139 | Charge Relay |

### Fan control
| DP | Entity | Type |
|---|---|---|
| 140 | AC Fan Speed | select: `LowSpeed` / `MidSpeed` / `HighSpeed` |

### Faults
| DP | Entity | Type |
|---|---|---|
| 115 | Fault Group 1 | sensor (bitmap) |
| 116 | Fault Group 2 | sensor (bitmap) |

### Operating state
| DP | Entity | Notes |
|---|---|---|
| 118 | Actual Operating Mode | `0` = Heating, `1` = Cooling — the **real** state |
| 103 | Celsius / Fahrenheit toggle | leave on °C |

---

## Sensor Design Notes

**DP 124 is NOT the weather temperature.**
It is the air temperature measured *at the unit's air intake / internal sensor*.
On a sunny day next to a wall it can read several degrees above the published
weather temp; in airflow it drops. Use it for **delta-T against the coil** and
defrost reasoning — never as your local outdoor temperature. If you want true
weather temp, use a `weather.*` entity instead.

**DP 118 matters more than DP 105 for diagnostics.**
DP 105 is the *requested* mode (`smart`/`warm`/`cool`) — what you asked for.
DP 118 is the *actual* operating mode (`0` heating / `1` cooling) — what the
machine is really doing right now. In `smart` mode the unit decides for itself,
so 105 tells you nothing about current behaviour. Always diagnose, automate, and
alarm on **118**.

**Compressor frequency (DP 125) is the efficiency signal.**
An inverter heat pump is most efficient at **low, steady frequency**. A unit
holding 40–60 Hz is sipping power for a given heat output; one pinned at max Hz
is working hard and its COP is falling. Frequency is your single best proxy for
"how hard is it trying" and feeds the COP estimate below.

**EXV position (DP 128) is for advanced users.**
The electronic expansion valve meters refrigerant. Its position (0–10000, often
`/100` for %) tracks superheat control. Stable EXV = healthy refrigerant cycle;
hunting/erratic EXV or a pegged valve hints at charge issues, sensor faults, or
a struggling cycle. Most users can ignore it; refrigeration-literate users will
want it on the chart.

---

## Advanced section

See the formulas and templates in
[dashboard/pool_heat_pump_tile.yaml](dashboard/pool_heat_pump_tile.yaml) (commented
template-sensor block) and the explanation here:

- **COP estimate (proxy, not metered):** without a flow meter and a power meter
  you cannot measure true COP. As a *trend* proxy you can relate useful heat
  (from water delta-T × assumed flow) to electrical input (≈ current × voltage,
  or a frequency-based input model). Treat the number as **relative**, good for
  "is it better than yesterday," not for billing.
- **Heat-output estimate via delta-T:** `Q ≈ ṁ · c · ΔT`, where ΔT is across the
  heat exchanger. With only an inlet sensor (DP 102) you approximate ΔT against
  ambient/coil and a fixed assumed flow rate — again a **trend**, not a watt-meter.
- **Defrost cycle detection:** trust DP 130 first. As a cross-check, defrost
  typically shows the 4-way valve (DP 136) flipping, coil temp (DP 120) rising
  sharply while ambient (DP 124) is low, and the fan (DP 129) dropping.

⚠️ These are **proxy models**. They are useful for spotting trends and faults,
not for exact thermodynamic accounting.

---

## Troubleshooting

Common issues (full list in [docs/troubleshooting.md](docs/troubleshooting.md)):

- **"Connection refused" / device unreachable** → another app holds the local
  connection (Tuya/Smart Life app, or the cloud integration). The device allows
  only one local session. Force-close the phone app.
- **Local Key stopped working** → it rotates whenever the device is re-paired in
  the Tuya app. Re-run the TinyTuya wizard.
- **Entities show `unavailable`** → wrong protocol version (try 3.3 / 3.4),
  wrong IP after a DHCP change (set a reservation), or wrong DP id/type.
- **Setpoint won't change** → DP 106 is an integer/number; confirm range and that
  the unit is powered (DP 1 on) and in a mode that accepts a setpoint.

---

## Notes on OEM heat pump behaviour

- Many OEM units **gate the compressor behind a flow switch** — the circulation
  pump (DP 135) must run and prove flow before the compressor (DP 134) starts.
- In `smart` mode (DP 105) the controller chooses heat/cool autonomously; expect
  DP 118 to disagree with what you "set."
- Compressor **frequency ramps slowly** by design (soft start) — don't read a low
  Hz right after start as a fault.
- DP ranges (e.g. DP 106 at -22→104 °C) are the **raw protocol limits**, not the
  unit's safe operating envelope. Clamp setpoints in the UI to a sane pool range.

---

## Repository layout

```text
pool-heat-pump-ha/
 ├── README.md
 ├── dashboard/
 │    └── pool_heat_pump_tile.yaml
 └── docs/
      ├── datapoints.md
      ├── installation.md
      └── troubleshooting.md
```

---

## Compatibility

- **Home Assistant 2026.x+**
- **LocalTuya** (HACS) — Tuya LAN protocol **v3.3** (try 3.4 if 3.3 fails)
- **TinyTuya** ≥ 1.13 (Python 3.9+)

## License

MIT — do whatever you like; no warranty. Working on refrigeration equipment and
mains wiring is at your own risk.
