# Changelog

All notable changes to this project are documented here.
Format follows [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/).

## [0.1.0] - 2026-06-21

First public release — a complete, documented LocalTuya integration and Home
Assistant dashboard for an OEM Tuya inverter pool heat pump (Tuya category `rs`,
LAN protocol v3.3, `modelId 00000018tk`).

### Added
- **Authoritative datapoint map** (`docs/datapoints.md`) decoded directly from the
  device's Tuya data model — all 26 DPs with translated names, types, ranges,
  access, and which analog input each sensor uses.
- **Fault-code reference** — bit labels for DP115 (`E1…F7`) and DP116
  (`F8,F9,Fb,Fa`) plus a template to list active codes.
- **Verified scaling**: compressor current (DP126) = ×0.1; temperatures = none.
- **Tile dashboard** (`dashboard/pool_heat_pump_tile.yaml`) using only core HA
  cards — Overview / Performance & Power / Temperatures / System & Faults, with a
  water-temperature trend and a measured-power graph.
- **Temperature explainer** clarifying that ambient ≠ weather, the coil is the
  air-side source, discharge is refrigerant, and "RadTemp" is the electronics
  heatsink (not pool heat).
- **Installation guide** (`docs/installation.md`): TinyTuya → LocalTuya → DP
  mapping → dashboard, and **`docs/troubleshooting.md`**.
- **AI context** (`docs/ai-context.md`) so Claude can set the whole thing up.
- Notes on the **no-outlet-water-temp** limitation and a COP proxy approach.

### Known limitations
- The controller exposes only **inlet** water temperature (no outlet/return), so
  true heat-output / COP requires an added sensor (e.g. DS18B20) plus a flow rate.
- DP103 (°C/°F toggle) is intentionally left unmapped — keep the unit on °C.

[0.1.0]: https://github.com/rhamblen/tuya-local-for-pool-heat-pump/releases/tag/v0.1.0
