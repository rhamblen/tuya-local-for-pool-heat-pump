# Troubleshooting

Work top to bottom — most problems are connection, key, or scaling.

---

## Device is `unavailable` / won't connect

**Only one local connection at a time.**
Tuya devices accept a single local session. If the **Tuya Smart / Smart Life**
phone app, the **cloud Tuya integration**, or a second TinyTuya script is talking
to the device, LocalTuya gets refused. Force-close the phone app and disable the
cloud Tuya integration in HA, then reload LocalTuya.

**IP changed.**
DHCP gave the unit a new address. Set a **DHCP reservation** on your router and
update the IP in LocalTuya.

**Wrong protocol version.**
Switch between **3.3** and **3.4** in the LocalTuya device config. Older firmware
may even be 3.1.

**Wi-Fi isolation / VLAN.**
If the heat pump and Home Assistant are on different VLANs/SSIDs with client
isolation (common on "IoT" networks), HA can't reach it. Allow the HA host to
reach the device IP on TCP **6668**.

---

## "Local Key is invalid" / decryption errors

**The key rotated.**
The Local Key changes **every time the device is re-paired** in the Tuya app
(and sometimes after a firmware update). Re-run the TinyTuya wizard
(`python -m tinytuya wizard`) and copy the fresh `key` into LocalTuya.

**Copied the wrong field.**
Use the `key` value, not the `id`, and not the cloud project secret.

---

## Entities exist but values look wrong

**Temperatures/current are 10× too big** (e.g. `285` instead of `28.5`).
Tuya transmits many values scaled ×10. Set a **divide-by-10 scale** on that
entity in LocalTuya. Validate against the unit's own display.

**EXV position huge** (e.g. `4200`).
DP 128 is 0–10000. Apply **`/100`** if you want a 0–100 % reading; otherwise leave
raw steps.

**Mode select shows raw numbers / wrong words.**
The option strings must match exactly what the device sends. Read DP 105 with
TinyTuya `status()` to see the literal values, then set the select options to
match (`smart` / `warm` / `cool`).

---

## Setpoint (DP 106) won't change

- Confirm **DP 1 (power) is on** — many units ignore setpoint writes when off.
- Confirm the **number range** in LocalTuya allows the value you're sending.
- In `smart` mode the unit may override your setpoint; switch to `warm`/`cool`
  to test.
- Check the unit isn't in a **flow-protection lockout** (pump not proving flow).

---

## Compressor won't start (but no fault)

This is usually normal OEM sequencing, not a bug:

1. **Circulation pump** (DP 135) must run first and prove water flow.
2. Compressor **frequency ramps slowly** (soft start) — a low Hz right after
   start is expected.
3. **Anti-short-cycle timer**: after a stop the unit waits several minutes before
   restarting the compressor.

Watch DP 135 → DP 134 → DP 125 in sequence on the dashboard to confirm.

---

## Defrost behaves oddly in winter

Defrost is normal. During it you'll typically see DP 130 = on, the 4-way valve
(DP 136) flip, coil temp (DP 120) climb, and the fan (DP 129) drop. It ends on
its own. If it **never** exits or cycles constantly, check coil/ambient sensors
and refrigerant charge (and the fault DPs 115/116).

---

## Faults (DP 115 / 116) are non-zero

These are **bitmaps** — each bit a distinct fault, OEM-specific. Read the value,
then cross-reference the unit's **error-code sheet / service manual**. Until you
have that mapping, treat any non-zero value as "check the unit's on-board display
for the error code." The "any fault" template binary sensor in
[datapoints.md](datapoints.md) is good for a single alarm.

---

## Home Assistant 2026 specifics

- Use the **maintained LocalTuya fork** (`xZetsubou/hass-localtuya`); the original
  `rospogrigio/localtuya` is no longer updated and may fail on current cores.
- After adding/removing entities, a **reload of the LocalTuya integration**
  (or HA restart) is sometimes needed before they appear.
- The dashboard uses only **core cards**, so no frontend-resource errors. If you
  swapped in Mushroom/custom cards, make sure those HACS frontend resources are
  installed.

---

## Still stuck — collect this before asking for help

1. Output of `python -m tinytuya scan`.
2. Output of a `d.status()` call (redact the key).
3. LocalTuya protocol version you're using.
4. Home Assistant version and LocalTuya version.
5. Relevant lines from **Settings → System → Logs** filtered for `localtuya`.
