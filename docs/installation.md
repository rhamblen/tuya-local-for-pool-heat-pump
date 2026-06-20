# Installation Guide

Two stages:

1. **TinyTuya** — used once to retrieve your **Local Key** (and confirm IP + DPs).
2. **LocalTuya** — the permanent Home Assistant integration that maps DPs to
   entities over the LAN.

You do **not** need the Tuya cloud integration in Home Assistant. You *do* need a
free Tuya IoT developer account for the one-time key extraction.

---

## Before you start

- Put the heat pump on **Wi-Fi** and pair it in the **Tuya Smart** or **Smart Life**
  app first. It must be working in the app before LocalTuya can talk to it.
- Give the heat pump a **static IP / DHCP reservation** on your router so its
  address never changes (otherwise entities go `unavailable` after a lease change).
- Have Python 3.9+ available on any computer (your PC is fine — Windows works).

---

## 1. TinyTuya — get the Local Key

### 1.1 Install

```bash
pip install tinytuya
```

(On Windows: `py -m pip install tinytuya`.)

### 1.2 Run the wizard

```bash
python -m tinytuya wizard
```

The wizard will ask for **Tuya IoT Platform** API credentials. To get them:

1. Go to <https://iot.tuya.com> and create a free developer account.
2. **Cloud → Development → Create Cloud Project** (any name; pick your data centre
   region — e.g. Central Europe / Western America — to match where your app
   account lives).
3. After creating it, open the project → **Devices → Link Tuya App Account** and
   **scan the QR code** with the Tuya Smart / Smart Life app to link your account.
   Your heat pump now appears under the linked account.
4. From the project's **Overview / Authorization** tab copy the **Access ID
   (Client ID)** and **Access Secret (Client Secret)**.
5. Note your **API region** (e.g. `us`, `eu`, `cn`, `in`).

Feed those into the wizard when prompted. It then downloads your device list.

### 1.3 Read the results

The wizard writes **`devices.json`** (and `tinytuya.json`) in the working
directory and prints a table. For your heat pump record:

- **`id`** → your **Device ID**
- **`key`** → your **Local Key**
- **`ip`** → the device IP on your LAN (the wizard also runs a network scan)

### 1.4 Validate LAN communication

Scan the network and read live datapoints to confirm the LAN protocol works
before touching Home Assistant:

```bash
python -m tinytuya scan
```

Then a quick status read (replace the placeholders):

```python
import tinytuya

d = tinytuya.OutletDevice(
    dev_id="YOUR_DEVICE_ID",
    address="YOUR_DEVICE_IP",
    local_key="YOUR_LOCAL_KEY",
    version=3.3,            # try 3.4 if 3.3 returns nothing
)
print(d.status())          # should print a dict of {DP: value}
```

You should see DPs like `1`, `102`, `105`, `106`, `118`, `125`… If you get an
empty/error response, see [troubleshooting.md](troubleshooting.md).

> Keep `devices.json` private — the Local Key is a secret.

---

## 2. LocalTuya — Home Assistant integration

### 2.1 Install via HACS

1. In Home Assistant: **HACS → Integrations**.
2. Search for **LocalTuya**. (If it isn't indexed, **⋮ → Custom repositories**,
   add `https://github.com/xZetsubou/hass-localtuya` as an *Integration* — this is
   the actively maintained fork that works on HA 2026.)
3. Install it, then **restart Home Assistant**.

### 2.2 Add the device

1. **Settings → Devices & Services → Add Integration → LocalTuya**.
2. Choose to add the device manually and enter:
   - **Host / IP** → your device IP
   - **Device ID** → from `devices.json`
   - **Local Key** → from `devices.json`
   - **Protocol version** → **3.3** (try 3.4 if discovery fails)
3. LocalTuya connects and shows the live DP values — this confirms the key works.

### 2.3 Map the datapoints

Now add one entity per row in [datapoints.md](datapoints.md). For each entity
LocalTuya asks for:

- **Entity type** (switch / sensor / binary_sensor / number / select)
- **DP id**
- Type-specific options:
  - `number` (DP 106): min/max/step + optional scaling
  - `select` (DP 105, DP 140): the option list (use the exact strings)
  - `sensor`: unit, device_class, optional scaling factor
  - `binary_sensor`: the "on" value and a device_class

Add **all 25 entities** in one session so the supplied dashboard works without
edits. When done, you'll have a `pool_heat_pump` device with the full entity set.

### 2.4 Confirm scaling

Compare a couple of entities against the unit's own display:

- If `sensor.pool_heat_pump_water_inlet_temp` reads `285` but the unit shows
  `28.5 °C`, set a **scale / divide-by-10** on that entity.
- Do the same check for compressor current (DP 126) and the setpoint (DP 106).

---

## 3. Add the dashboard

1. Open the dashboard in **Edit mode → ⋮ → Raw configuration editor**, **or** add
   a single **Manual** card.
2. Paste the contents of
   [../dashboard/pool_heat_pump_tile.yaml](../dashboard/pool_heat_pump_tile.yaml).
3. If you used the suggested entity ids, it works as-is. Otherwise find/replace
   `pool_heat_pump` with your device slug.

The card uses **only core Home Assistant cards** (tile, grid, vertical-stack,
heading) — no extra HACS frontend cards required.

---

## 4. (Optional) Advanced template sensors

The dashboard YAML includes a commented `template:` block (COP proxy, heat-output
proxy, defrost detection, fault rollup). To enable it, move that block into your
`configuration.yaml` (or a `!include`d package) and restart. These are **proxy
estimates** — read the caveats in the README before relying on them.

---

## You're done

- Local control + full inverter telemetry, working without the cloud.
- If anything is `unavailable` or wrong, go to [troubleshooting.md](troubleshooting.md).
