# AI Context

Reference for Claude / AI assistants setting up or editing this repo. **Read this
first.** It tells you what the project is, the order to install it in, the hard
rules that will trip you up, and how to cut a release.

This repo integrates an **OEM Tuya inverter pool heat pump** (Tuya category `rs`,
LAN protocol v3.3, `modelId 00000018tk`) into Home Assistant over the **local
network** via LocalTuya, and ships a dashboard tile for it.

---

## What this project is

- A verified **datapoint → entity map** ([docs/datapoints.md](datapoints.md)).
- A **dashboard** of core HA tile cards ([../dashboard/pool_heat_pump_tile.yaml](../dashboard/pool_heat_pump_tile.yaml)).
- **Install + troubleshooting** docs.

The interesting inverter telemetry (compressor frequency, EXV, coil/discharge
temps, relays) only exists on the **LAN** protocol — the Tuya **cloud**
integration does not expose it. **Never tell the user to use the cloud Tuya
integration for telemetry.**

---

## Install order (what you should do)

Do these in order. Prefer doing the work directly (TinyTuya, the Home Assistant
MCP) over telling the user to click through UIs, but fall back to the manual
guide ([docs/installation.md](installation.md)) when you lack access.

1. **Get LAN credentials with TinyTuya** (<https://github.com/jasonacox/tinytuya>):
   run the wizard to obtain the device **IP, Device ID, and Local Key**. Validate
   with `d.status()` that DPs are reported (try protocol 3.3, then 3.4).
2. **Install LocalTuya** — the maintained fork
   <https://github.com/xZetsubou/hass-localtuya> (HACS). The original
   `rospogrigio/localtuya` is stale; don't use it.
3. **Add the device and map the DPs** from [docs/datapoints.md](datapoints.md).
   Set scaling per the rules below.
4. **Create the derived helpers** (template sensors): heat-pump power and
   Heating/Cooling mode text (see the deployment reference in datapoints.md).
5. **Deploy the dashboard** — translate
   [../dashboard/pool_heat_pump_tile.yaml](../dashboard/pool_heat_pump_tile.yaml)
   into the user's view, adjusting entity ids to match their LocalTuya naming.

---

## Hard rules / gotchas (don't relearn these)

- **Cloud vs LAN:** telemetry is LAN-only. Cloud exposes a thin climate entity only.
- **Scaling:** DP126 compressor current = LocalTuya scale **0.1** (raw is deci-amps).
  Temperatures (102/120/122/124/127) = **no scaling**. EXV (128) = `/100` for %.
- **DP118 `WarmOrCool`** is the *actual* run mode (False = Heating) — better for
  diagnostics than the requested mode DP105.
- **No outlet/return water temp exists** (only AIN1 inlet). Real heat-output / COP
  needs an *added* sensor (DS18B20 on the return) + a flow rate. Don't pretend
  `RadTemp` (DP127, electronics heatsink) or `OutPipeTemp` (DP120, air coil) is
  delivered heat.
- **DP103** (°C/°F) — leave unmapped; keep the unit on °C so scaling stays sane.
- **Dashboard framing (sections view):** the "border" around a section is the
  section property `background: {color: lightgray, opacity: 50}` — **not** a tile
  or card border. Any new section must include it to match. **Do not** use
  `card-mod` for this; it's unreliable (and may be loaded more than once).
- **Power calibration (reference install only):** a Shelly meter reads the whole
  circuit; measured loads were pump ≈ 252 W, lights ≈ 26 W. Derived heat-pump
  watts = `meter − pump − lights`. These constants are install-specific.

---

## File map

| Path | What |
|---|---|
| `README.md` | Overview + install paths (Claude-first, then manual) |
| `docs/datapoints.md` | Authoritative DP map, scaling, fault codes, deployment reference |
| `docs/installation.md` | Manual TinyTuya → LocalTuya → mapping → dashboard |
| `docs/troubleshooting.md` | Connection, key rotation, scaling, sequencing |
| `dashboard/pool_heat_pump_tile.yaml` | Core-card tile dashboard |
| `CHANGELOG.md` | Versioned history |

---

## How to publish a new version

1. Make the changes; update [docs/datapoints.md](datapoints.md) etc. so docs match reality.
2. Add a dated entry to [../CHANGELOG.md](../CHANGELOG.md) under a new
   `## [x.y.z]` heading (Keep a Changelog format) and update the link at the bottom.
3. Commit (end commit messages with the project's `Co-Authored-By` line).
4. Tag: `git tag -a vX.Y.Z -m "vX.Y.Z"` then `git push origin vX.Y.Z`.
5. Optionally draft a GitHub Release from the tag.

Semantic versioning: patch = doc/fix, minor = new entities/features, major =
breaking changes to the DP map or dashboard entity ids.
