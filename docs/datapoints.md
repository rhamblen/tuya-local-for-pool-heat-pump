# Datapoint (DP) Map

This is the authoritative DP → Home Assistant entity mapping for the OEM Tuya
inverter pool heat pump (Tuya category `rs`, LAN protocol v3.3).

When adding the device in LocalTuya, create one entity per row using the
**Platform**, **DP id**, and the suggested **entity id**. Suggested entity ids
assume the device is named `pool_heat_pump` — rename to match yours and update
the dashboard accordingly.

> **Scaling note:** LocalTuya can apply a scaling factor per entity. Where a
> column says "scale", set it in the entity options. Tuya often transmits
> temperatures and currents ×10 (e.g. `285` = `28.5`). **Validate against the
> unit's own display** before trusting a scale — OEM firmware varies.

---

## Control

| DP | Name | Platform | Suggested entity_id | Range / values | Notes |
|---|---|---|---|---|---|
| 1 | Power | `switch` | `switch.pool_heat_pump_power` | on/off | Master power |
| 106 | Set Temperature | `number` | `number.pool_heat_pump_set_temperature` | -22 → 104 °C (raw) | Clamp to a pool range (e.g. 18–40 °C) in the UI. Possible ×10 scale |
| 105 | Mode | `select` | `select.pool_heat_pump_mode` | `smart` / `warm` / `cool` | Requested mode (see DP 118 for actual) |
| 117 | Silent Mode | `switch` | `switch.pool_heat_pump_silent_mode` | on/off | Night / low-noise |

---

## Temperature sensors

| DP | Name | Platform | Suggested entity_id | Unit | Notes |
|---|---|---|---|---|---|
| 102 | Water Inlet Temperature | `sensor` | `sensor.pool_heat_pump_water_inlet_temp` | °C | Primary water temp. Possible ×10 scale |
| 124 | Ambient Air Temperature | `sensor` | `sensor.pool_heat_pump_ambient_air_temp` | °C | **Internal unit air sensor — NOT weather temp** |
| 120 | Evaporator / Coil Temperature | `sensor` | `sensor.pool_heat_pump_coil_temp` | °C | Drives defrost logic |
| 122 | Compressor Discharge Temperature | `sensor` | `sensor.pool_heat_pump_discharge_temp` | °C | High value = hard work / possible low charge |
| 127 | Heat Sink Temperature | `sensor` | `sensor.pool_heat_pump_heat_sink_temp` | °C | Inverter/IPM board temperature |

Set `device_class: temperature` and `state_class: measurement` on these for nice
history graphs.

---

## Inverter / performance sensors

| DP | Name | Platform | Suggested entity_id | Unit | Notes |
|---|---|---|---|---|---|
| 125 | Compressor Frequency | `sensor` | `sensor.pool_heat_pump_compressor_frequency` | Hz | Key efficiency signal |
| 126 | Compressor Current | `sensor` | `sensor.pool_heat_pump_compressor_current` | A | Possible ×10 scale; `device_class: current` |
| 104 | Speed Percentage | `sensor` | `sensor.pool_heat_pump_speed_percent` | % | |
| 128 | EXV Position | `sensor` | `sensor.pool_heat_pump_exv_position` | steps / % | 0–10000; apply `/100` if you want % |
| 129 | DC Fan Speed | `sensor` | `sensor.pool_heat_pump_dc_fan_speed` | RPM / PWM | Raw value; meaning is firmware-specific |

---

## System state (binary sensors)

Map these as `binary_sensor`. Choose a sensible `device_class` (e.g. `running`,
`power`, `opening`) per the table.

| DP | Name | Platform | Suggested entity_id | device_class | Notes |
|---|---|---|---|---|---|
| 130 | Defrost Active | `binary_sensor` | `binary_sensor.pool_heat_pump_defrost` | `running` | True during a defrost cycle |
| 134 | Compressor Relay | `binary_sensor` | `binary_sensor.pool_heat_pump_compressor_relay` | `running` | Compressor energised |
| 135 | Circulation Pump | `binary_sensor` | `binary_sensor.pool_heat_pump_circulation_pump` | `running` | Water pump |
| 136 | 4-Way Valve | `binary_sensor` | `binary_sensor.pool_heat_pump_4way_valve` | `opening` | Reversing valve (heat/cool/defrost) |
| 139 | Charge Relay | `binary_sensor` | `binary_sensor.pool_heat_pump_charge_relay` | `power` | Inverter DC-bus charge relay |

---

## Fan control

| DP | Name | Platform | Suggested entity_id | Values |
|---|---|---|---|---|
| 140 | AC Fan Speed | `select` | `select.pool_heat_pump_ac_fan_speed` | `LowSpeed` / `MidSpeed` / `HighSpeed` |

---

## Fault system

Fault DPs are **bitmaps** — each bit is a distinct fault. Map as plain sensors;
decode bits in templates (see below) or just alarm on "non-zero".

| DP | Name | Platform | Suggested entity_id | Notes |
|---|---|---|---|---|
| 115 | Fault Group 1 | `sensor` | `sensor.pool_heat_pump_fault_group_1` | Bitmap |
| 116 | Fault Group 2 | `sensor` | `sensor.pool_heat_pump_fault_group_2` | Bitmap |

A simple "any fault" template binary sensor:

```yaml
template:
  - binary_sensor:
      - name: Pool Heat Pump Fault
        device_class: problem
        state: >
          {{ (states('sensor.pool_heat_pump_fault_group_1') | int(0)) > 0
             or (states('sensor.pool_heat_pump_fault_group_2') | int(0)) > 0 }}
```

> Individual bit meanings are OEM-specific. Cross-reference the unit's service
> manual / error-code sheet to label specific bits. Until you have that, treat
> any non-zero value as "fault present, check the unit display."

---

## Operating state

| DP | Name | Platform | Suggested entity_id | Values | Notes |
|---|---|---|---|---|---|
| 118 | Actual Operating Mode | `sensor` | `sensor.pool_heat_pump_actual_mode` | `0` = Heating, `1` = Cooling | The **real** running mode — use for diagnostics/automations |
| 103 | Celsius / Fahrenheit toggle | `switch` or `select` | `switch.pool_heat_pump_unit_toggle` | °C / °F | Leave on °C to keep scaling consistent |

Human-readable actual mode:

```yaml
template:
  - sensor:
      - name: Pool Heat Pump Actual Mode (text)
        state: >
          {% set m = states('sensor.pool_heat_pump_actual_mode') %}
          {{ 'Heating' if m == '0' else 'Cooling' if m == '1' else 'Unknown' }}
```

---

## LocalTuya add-device cheat sheet

When you click **Add a new device** in LocalTuya and then **add entities**, for
each row above you'll be asked for:

- **Entity type / platform** → the *Platform* column
- **DP id** → the *DP* column
- For `number`: min/max/step → use the *Range* column (clamp setpoint sensibly)
- For `select`: the option list → the *values* column (use exact strings)
- For `sensor`: optional scaling, unit, device_class → see notes

Do all 25 entities in one pass so the dashboard works without edits.
