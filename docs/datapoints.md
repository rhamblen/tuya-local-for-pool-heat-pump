# Datapoint (DP) Map

Authoritative DP map for the OEM Tuya inverter pool heat pump
(Tuya category `rs`, LAN protocol v3.3). The table below is decoded directly
from the device's own Tuya data model (`modelId 00000018tk`, dumped via the
Tuya IoT `device/{id}/model` API), with the original Chinese names translated —
so it reflects what the MCU *actually* exposes, including DPs not yet turned
into Home Assistant entities.

When adding the device in LocalTuya, create one entity per row using the
**Platform**, **DP id**, and the suggested **entity id**. Suggested entity ids
assume the device is named `pool_heat_pump` — rename to match yours.

---

## Authoritative DP table

Access: `rw` = read/write (controllable), `ro` = read-only (telemetry).
`AIN` = which analog input the sensor is wired to on the controller.

| DP | Tuya code | Meaning (EN) | Type / range | Access | AIN | Notes |
|---|---|---|---|---|---|---|
| 1 | Power | Power on/off | bool | rw | — | Master power |
| 102 | WInTemp | **Inlet** water temp | value -22…250 | ro | AIN1 | °C (or °F per DP103). **No scaling** (scale 0) |
| 103 | change_tem | °C/°F toggle (0=°C, 1=°F) | bool | rw | — | Keep 0 (°C) |
| 104 | SpeedPercentage | Compressor speed % | value 0…150 | ro | — | % |
| 105 | SetMode | Mode (0 smart / 1 warm / 2 cool) | enum `smart`/`warm`/`cool` | rw | — | Requested mode |
| 106 | SetTemp | Set temperature | value -22…104 | rw | — | Clamp to a sane pool range |
| 107 | SetDnLimit | Setpoint **lower** limit | value -22…104 | ro | — | Use to bound the `number` entity |
| 108 | SetUpLimit | Setpoint **upper** limit | value -22…104 | ro | — | Use to bound the `number` entity |
| 115 | fault1 | Fault group 1 (bitmap) | bitmap, 30 bits | ro | — | See [Fault codes](#fault-codes) |
| 116 | fault2 | Fault group 2 (bitmap) | bitmap, 4 bits | ro | — | See [Fault codes](#fault-codes) |
| 117 | SilentMode | Silent / night mode | bool | rw | — | |
| 118 | WarmOrCool | **Actual running mode** (heat/cool) | bool | ro | — | The real state — better than DP105 for diagnostics |
| 120 | OutPipeTemp | **Outdoor air-coil** temp | value -22…250 | ro | AIN3 | The *cold/input* side in heating. No scaling |
| 122 | ExhaustTemp | Compressor **discharge** temp | value -22…250 | ro | AIN5 | Hot gas. No scaling |
| 124 | AmbTemp | Outdoor **ambient air** temp | value -22…250 | ro | AIN7 | NOT weather temp. No scaling |
| 125 | CompFreAct | Compressor frequency | value 0…150 | ro | — | Hz. Key efficiency signal |
| 126 | CompressorCurrent | Compressor current | value 0…100 | ro | — | **Scale ×0.1 → A** (raw 50 = 5.0 A) |
| 127 | RadTemp | Electronics **heatsink** temp | value -22…250 | ro | — | Inverter/IPM board — NOT water heat |
| 128 | EXVPosition | EXV opening | value 0…10000 | ro | — | `/100` for % if desired |
| 129 | DCFanSpeed | DC fan speed | value 0…10000 | ro | — | RPM/PWM |
| 130 | Defrost | Defrost active | bool | ro | — | |
| 134 | CompRly | Compressor contactor relay (OUT1) | bool | ro | — | Compressor actually energised |
| 135 | CyclePump | Circulation water pump relay (OUT2) | bool | ro | — | The pump the controller calls for |
| 136 | ReserveValve | 4-way reversing valve | bool | ro | — | Heat/cool/defrost |
| 139 | ChargeRly | Current-limiting charge relay | bool | ro | — | DC-bus pre-charge |
| 140 | ACFanSpeed | AC fan speed | enum STOP/`LowSpeed`/`MidSpeed`/`HighSpeed` | ro | — | |

### Scaling — verified

- **Temperatures (102, 120, 122, 124, 127): no scaling.** The model declares
  `scale 0` and the live values read sensibly (water 30, ambient 22, discharge
  35). Leave the LocalTuya scaling factor at **1**.
- **Compressor current (126): scale 0.1.** Although the model says `scale 0`, the
  raw value is deci-amps — set the LocalTuya **Scaling factor to `0.1`** so it
  reads amps (raw 50 → 5.0 A, raw 75 → 7.5 A). The value tracks frequency, which
  confirms the factor. Note this is **compressor/inverter-side** current, not
  mains — don't compute power from it (use a real power meter; see COP notes).
- **EXV (128): 0–10000.** Apply `/100` if you want 0–100 %.

---

## Temperature sensors — what each one *actually* is

The five temperature DPs are easy to misread (the names don't mean what you'd
guess). Here's what each one physically measures and how to read it:

| DP | Tuya code | Where it's measured | What it tells you | Watch out for |
|---|---|---|---|---|
| **102** | WInTemp (AIN1) | Pool water **entering** the heat pump | Your actual pool water temp — the number you care about | This is **inlet only**; there is **no outlet sensor**, so you can't get water-side ΔT / heat output from the unit (see [COP note](#heat-output--cop--hardware-limitation)) |
| **124** | AmbTemp (AIN7) | Air at the unit's **intake** | Outdoor air the unit pulls heat *from* | **NOT the weather temperature** — it sits by the unit, reads high in sun / by a wall, and drops in airflow. Use a `weather.*` entity for real outdoor temp |
| **120** | OutPipeTemp (AIN3) | The outdoor **air-side coil** (evaporator in heating) | The *cold/input* side. In heating it runs **below ambient** (refrigerant absorbing heat from air); a sharp rise while ambient is low signals a defrost | It is **not** "coil that heats the water" — it's the air heat-exchanger, the source side |
| **122** | ExhaustTemp (AIN5) | Compressor **discharge** (hot refrigerant gas) | How hard the compressor is working; rises with load. Very high discharge can indicate low charge or restricted flow | Refrigerant-side, not water — don't read it as delivered heat |
| **127** | RadTemp | The **inverter electronics heatsink** | Health of the power board / IPM. Should stay moderate; a climbing value in heat means the electronics cooling is stressed | **Not a "radiator" heating the pool** — it's the circuit-board heatsink. Irrelevant to heat output |

**Rule of thumb for "is it heating well?":** watch **inlet water (102)** rising
over time, **ambient (124)** as the heat source, and **discharge (122)** /
**compressor frequency** as the effort. The coil (120) and heatsink (127) are
diagnostics, not output. None of these give kW of heat delivered — that needs an
added outlet-water sensor + flow rate.

---

## Fault codes

Both fault DPs are **bitmaps** — each bit is one error code. The device model
ships the bit labels (the codes shown on the unit's display); the *meaning* of
each code still needs the unit's service manual / error sheet.

**DP 115 `fault1`** (bit 0 → 29):

```
E1 E2 E3 E4 E5 E6 E7 E8 E9 EA EB ED
P0 P1 P2 P3 P4 P5 P6 P7 P8 P9 PA
F1 F2 F3 F4 F5 F6 F7
```

**DP 116 `fault2`** (bit 0 → 3): `F8  F9  Fb  Fa`

A value of `0` on both = no faults. To decode a non-zero value, read it as a
bitmask: bit *n* set ⇒ the *n*-th code above is active. A template that lists
active DP115 codes:

```yaml
template:
  - sensor:
      - name: Pool Heat Pump Active Faults
        state: >
          {% set codes = ['E1','E2','E3','E4','E5','E6','E7','E8','E9','EA','EB','ED',
                          'P0','P1','P2','P3','P4','P5','P6','P7','P8','P9','PA',
                          'F1','F2','F3','F4','F5','F6','F7'] %}
          {% set v = states('sensor.pool_error_code1') | int(0) %}
          {% set ns = namespace(active=[]) %}
          {% for i in range(codes | length) %}
            {% if v.__and__(2 ** i) %}{% set ns.active = ns.active + [codes[i]] %}{% endif %}
          {% endfor %}
          {{ ns.active | join(', ') if ns.active else 'OK' }}
```

---

## Heat output & COP — hardware limitation

**This controller does not expose an outlet/return water temperature.** Its
analog inputs are: AIN1 = inlet water (DP102), AIN3 = outdoor air coil (DP120),
AIN5 = discharge gas (DP122), AIN7 = ambient air (DP124). AIN2/4/6 are not
populated. So there is **no on-board way to measure water-side ΔT**, which is
what heat output needs:

> Q (kW) = flow (kg/s) × 4.186 × (T_outlet − T_inlet)

Consequences:

- `RadTemp` (DP127) is the **electronics heatsink**, not delivered heat — do not
  use it for output.
- `OutPipeTemp` (DP120) is the **air-side coil** (input side), not output.
- To get real heat output / COP you must **add an outlet water sensor**
  (e.g. a DS18B20 on the return pipe via ESPHome) **and a flow rate** (pump
  spec/curve or a flow meter). Electrical input is already exact if you have a
  power meter on the circuit.

Until then, COP can only be a rough proxy (inlet vs ambient ΔT) — good for
trends, not accounting.

---

## Mapping completeness

**25 of 26 DPs are mapped** in the reference install. The four diagnostics below
were added after the initial pass:

| DP | Added as | Why |
|---|---|---|
| 118 `WarmOrCool` | `sensor` (True/False; `False` = Heating) | The **actual** heat/cool running state — best diagnostic signal. A template helper turns it into "Heating"/"Cooling" text (see below) |
| 134 `CompRly` | `binary_sensor` | Confirms the compressor contactor is closed (running vs commanded) |
| 107 `SetDnLimit` / 108 `SetUpLimit` | `sensor` (read 18 / 40 here) | Read once, then set the `number` (106) min/max to match the unit's real limits |

**Only DP 103 (`change_tem`, °C/°F toggle) is intentionally left unmapped** — keep
the unit on °C so temperature scaling stays consistent.

> Note: a LocalTuya `number`'s min/max is set in the LocalTuya config flow, not via
> the HA API — so bounding DP106 with the DP107/108 values is a manual edit.

---

## LocalTuya add-device cheat sheet

When you click **Add a new device** in LocalTuya and then **add entities**, for
each row above you'll be asked for:

- **Entity type / platform** → switch / sensor / binary_sensor / number / select
- **DP id** → the *DP* column
- For `number` (106): min/max/step → bound with DP107/DP108
- For `select` (105, 140): the option list → use the exact enum strings
- For `sensor`: scaling (current = `0.1`, temps = `1`), unit, device_class

---

## Reference: a real deployment

The bundled dashboard (`dashboard/pool_heat_pump_tile.yaml`) is wired to the
actual entity ids from a working install, so it doubles as a concrete example.
Naming there uses `pool_*` / `pool_heatpump_*` rather than the generic
`pool_heat_pump_*` suggestions — adjust to whatever your LocalTuya setup
produced.

| DP | This install's entity_id | Friendly name |
|---|---|---|
| 1 | `switch.pool_power_switch` | Heatpump Power |
| 106 | `number.pool_target_temperature` | Target Temperature |
| 105 | `select.pool_set_mode` | Heatpump Mode |
| 117 | `switch.pool_silent_mode` | Heatpump Silent Mode |
| 102 | `sensor.pool_current_temperature` | Heatpump Inlet Water Temp |
| 124 | `sensor.pool_ambient_temperature` | Heatpump Ambient Temp |
| 120 | `sensor.pool_heatpump_coil_pipe_temp` | Heatpump Coil / Pipe Temp |
| 122 | `sensor.pool_exhaust_temperature` | Heatpump Discharge Temp |
| 127 | `sensor.pool_radiator_temperature` | Heatpump Heat Sink Temp |
| 125 | `sensor.pool_heatpump_compressor_frequency` | Heatpump Compressor Frequency |
| 126 | `sensor.pool_heatpump_compressor_current` | Heatpump Compressor Current (scale 0.1) |
| 104 | `sensor.pool_compressor_speed` | Heatpump Speed Percentage |
| 128 | `sensor.pool_heatpump_exv_position` | Heatpump EXV Position |
| 129 | `sensor.pool_heatpump_dc_fan_speed` | Heatpump DC Fan Speed |
| 130 | `binary_sensor.pool_heatpump_defrost` | Heatpump Defrost |
| 134 | `binary_sensor.pool_heatpump_compressor_relay` | Compressor Contactor Relay |
| 135 | `binary_sensor.pool_heatpump_water_pump` | Heatpump Water Pump |
| 136 | `binary_sensor.pool_heatpump_4_way_valve` | Heatpump 4-way Valve |
| 139 | `binary_sensor.pool_heatpump_charge_relay` | Heatpump Charge Relay |
| 140 | `select.pool_heatpump_ac_fan_speed` | Heatpump AC Fan Speed |
| 118 | `sensor.pool_heatpump_actual_operating_mode` | Actual Operating Mode (False = Heating) |
| 107 | `sensor.pool_heatpump_setpoint_lower_limit` | Setpoint lower limit |
| 108 | `sensor.pool_heatpump_setpoint_upper_limit` | Setpoint upper limit |
| 115 | `sensor.pool_error_code1` | Heatpump Fault Group 1 |
| 116 | `sensor.pool_error_code2` | Heatpump Fault Group 2 |

**Derived template helpers (this install):**

| Helper entity | Purpose |
|---|---|
| `sensor.pool_heat_pump_power` | `max(0, pool_power_power − 252 − 26·lights)` — measured heat-pump-only watts |
| `sensor.pool_heat_pump_mode` | Maps DP118 to **Heating / Cooling** text for the dashboard |

**Measured electrical input (not a Tuya DP):** this install has a **Shelly power
meter** on the circuit (`sensor.pool_power_power` W, `sensor.pool_power_energy`
kWh). A power-decomposition test (heat pump off vs running, lights off vs on)
gave: **circulation pump ≈ 252 W**, **lights (MiLight) ≈ 26 W**, heat-pump
standby ≈ 0 W. So live heat-pump electrical power:

> **P_heatpump = `sensor.pool_power_power` − 252 (pump) − 26 (only while lights on)**

Measured running draw: ~724 W at 50 Hz/67 %, ~1246 W at 75 Hz/100 %.
