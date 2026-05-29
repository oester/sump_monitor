# Sump Pump Monitor — Installation Guide

This package installs three automations and a set of helpers that work together
to monitor a sump pump, detect anomalies, and send push notifications when the
pump stops running, runs too long, or the power sensor goes offline.

---

## Requirements

- Home Assistant 2024.1 or later
- A **power monitoring sensor** on the sump pump circuit (e.g. Emporia Vue,
  Shelly EM, Z-Wave power meter, or similar). The sensor must report wattage
  and go above a known threshold when the sump is running.
- A **mobile notification service** configured in HA (e.g. the Home Assistant
  companion app installed on a phone).
- Basic familiarity with editing HA YAML configuration files.

---

## Files in This Package

| File | Description |
|---|---|
| `helpers.yaml` | All required helpers: input_datetime, input_number, counter, timer, and template sensors |
| `statistics_sensor.yaml` | Instructions for creating the rolling average statistics sensor (UI only) |
| `automations.yaml` | All four automations |
| `INSTALL.md` | This file |

---

## Step 1 — Identify Your Customization Values

Before installing anything, note the following from your HA instance:

### 1a. Sump power sensor entity ID
Find the entity that reports wattage on the sump pump circuit. It should be a
`sensor` with `unit_of_measurement: W` or similar.

Common examples:
- `sensor.sump_power_electric_consumption_w` (Emporia Vue)
- `sensor.sump_plug_power` (smart plug)
- `sensor.zwave_sump_power` (Z-Wave meter)

### 1b. Running wattage threshold
Check what wattage the sensor reports when the sump pump is actively running.
The default threshold is **50W** — adjust if your pump draws more or less at startup.
A conservative value slightly above idle noise is best (e.g. if the pump draws
5W idle and 300W running, 50W is a safe threshold).

### 1c. Notification service
Find your notification service entity. In HA, go to Developer Tools → Services,
search for `notify`, and find your mobile device.

Common examples:
- `notify.mobile_app_johns_iphone`
- `notify.mobile_app_pixel_8`
- `notify.all_devices` (if you want all devices notified)

---

## Step 2 — Add Helpers

### Simple helpers (input_datetime, input_number, counter, timer)

**Option A — YAML (recommended for version control)**

Add the contents of `helpers.yaml` to your HA configuration. Depending on
your setup, either:

1. Paste the relevant sections directly into `configuration.yaml`, or
2. Use `!include` files — for example:

```yaml
# configuration.yaml
input_datetime: !include input_datetimes.yaml
input_number: !include input_numbers.yaml
counter: !include counters.yaml
timer: !include timers.yaml
```

Then paste the corresponding blocks from `helpers.yaml` into each file.

**Option B — UI**

Go to **Settings → Devices & Services → Helpers → Add Helper** and create
each helper manually using the values in `helpers.yaml` as reference.

---

### Template sensors

**Option A — YAML**

Add the `template:` block from `helpers.yaml` to your `configuration.yaml`,
or to a `templates.yaml` file referenced via:

```yaml
# configuration.yaml
template: !include templates.yaml
```

**Option B — UI (flow-based template helper)**

> ⚠️ Important: If you create template sensors through the UI
> (Settings → Helpers → Add Helper → Template), you **must** paste the
> state template into the UI editor and save. The YAML template expressions
> cannot be loaded by the API for this helper type — the UI editor is required.

1. Go to **Settings → Devices & Services → Helpers → Add Helper → Template → Sensor**
2. Name: `Sump Pump Interval Formatted`
3. Paste this into the State template field:
```
{% set total_minutes = states('sensor.sump_pump_interval') | float(0) | int %}{% set days = (total_minutes // 1440) | int %}{% set hours = ((total_minutes % 1440) // 60) | int %}{% set mins = (total_minutes % 60) | int %}{% if days > 0 %}{{ days }} day{{ 's' if days != 1 }}, {{ '%02d:%02d' | format(hours, mins) }}{% else %}{{ '%02d:%02d' | format(hours, mins) }}{% endif %}
```
4. Save, then repeat for `Sump Pump Last Run Formatted` replacing
   `sensor.sump_pump_interval` with `sensor.sump_pump_time_since_last_run`.

Do the same for `Sump Pump Interval` and `Sump Pump Time Since Last Run` if
not using YAML.

---

## Step 3 — Create the Rolling Average Statistics Sensor

This sensor **must** be created through the HA UI — it cannot be added via YAML.

1. Go to **Settings → Devices & Services → Helpers → Add Helper**
2. Select **Statistical characteristic**
3. Configure as follows:

| Field | Value |
|---|---|
| Name | `Sump Pump Interval Rolling Average` |
| Entity | `sensor.sump_pump_interval` |
| Characteristic | Mean |
| Sampling size | 5 |
| Max age | 7 days |

4. Click **Submit**. The entity ID will be `sensor.sump_pump_interval_rolling_average`.

> **Note:** This sensor needs 5 sump pump cycles to fully populate. Until then,
> the watchdog automation falls back to the maximum alert threshold (240 min default).

---

## Step 4 — Add the Automations

**Option A — Import via UI**

1. Go to **Settings → Automations & Scenes → Automations**
2. Click the three-dot menu → **Import automation from YAML** (or use the
   raw config editor)
3. Paste each automation from `automations.yaml` one at a time

**Option B — YAML file**

If you manage automations via a YAML file, add the contents of `automations.yaml`
to your automations file (typically `automations.yaml` in your config directory).

---

## Step 5 — Customize

Open `automations.yaml` and replace all items marked `# <<< CONFIGURE`:

### Power sensor entity ID
Replace every occurrence of:
```
sensor.your_sump_power_sensor
```
With your sensor entity ID from Step 1a.

There are **7 occurrences** across the four automations (triggers in automations
#1, #3, and #4 plus the guard condition in automation #2).

### Wattage threshold
Replace every occurrence of:
```
above: 50
```
With your threshold from Step 1b. Also replace the matching `below: 50` in
automation #4. There are **3 occurrences** (automations #1, #2, and #4 above/below pair).

### Notification service
Replace every occurrence of:
```
notify.mobile_your_device
```
With your notification service from Step 1c. There are **6 occurrences** across
the four automations.

---

## Step 6 — Reload / Restart

After adding helpers and automations:

1. **Settings → System → Restart → Quick reload** (reloads YAML without full restart), or
2. Do a full HA restart if you added new YAML files

Verify in **Developer Tools → States** that these entities exist and have valid states:
- `input_datetime.sump_last_run`
- `input_datetime.sump_run_start`
- `input_number.sump_last_interval`
- `input_number.sump_alert_multiplier`
- `input_number.sump_alert_min_minutes`
- `input_number.sump_alert_max_minutes`
- `input_number.sump_max_run_seconds`
- `counter.sump_pump_cycles`
- `timer.sump_pump_watchdog`
- `sensor.sump_pump_interval`
- `sensor.sump_pump_time_since_last_run`
- `sensor.sump_pump_interval_rolling_average`
- `sensor.sump_pump_interval_formatted`
- `sensor.sump_pump_last_run_formatted`

---

## Step 7 — Initial State Setup

The helpers start with no history. On first use:

1. Wait for the sump to run once naturally, or manually trigger a test by
   temporarily lowering the wattage threshold and toggling a switch on the circuit.
2. After the first run, `input_datetime.sump_last_run` will be populated and
   the watchdog will arm automatically.
3. After 5 runs, `sensor.sump_pump_interval_rolling_average` will be fully
   populated and the adaptive watchdog will use real interval data.

---

## Tuning the Excessive Runtime Alert

One `input_number` helper controls the runtime threshold:

| Helper | Default | Range | Effect |
|---|---|---|---|
| `input_number.sump_max_run_seconds` | 30s | 10–300s | Alert fires if a single run exceeds this duration |

Set this to a value comfortably above your pump's normal runtime. For example,
if your sump typically runs for 6 seconds, 30 seconds (5×) is a safe starting
threshold. If you get false alerts, increase it.

---

## Tuning the Watchdog

Three `input_number` helpers control watchdog sensitivity. Adjust them from
**Settings → Devices & Services → Helpers**, or add an Entities card to a
dashboard with all three for easy slider access.

| Helper | Default | Effect |
|---|---|---|
| `input_number.sump_alert_multiplier` | 2.5× | How many times the rolling average to wait before alerting. Higher = more patient. |
| `input_number.sump_alert_min_minutes` | 30 min | Never alert faster than this, even during heavy rain cycling. |
| `input_number.sump_alert_max_minutes` | 240 min | Never wait longer than this to alert, and used as fallback when no interval history exists. |

**Example scenarios:**

| Rolling avg | Multiplier | Min | Max | Watchdog set to |
|---|---|---|---|---|
| 5 min (heavy rain) | 2.5× | 30 min | 240 min | 30 min (floored) |
| 35 min (normal) | 2.5× | 30 min | 240 min | 88 min |
| 120 min (dry) | 2.5× | 30 min | 240 min | 240 min (capped) |
| unknown (spring) | — | — | 240 min | 240 min (fallback) |

---

## Seasonal Operation

| Season | Action |
|---|---|
| **Spring** | Enable `automation.sump_pump_not_running_alert`. A "Sump Pump Resumed" notification confirms it's active after the first run. |
| **Summer/Fall** | All three automations run year-round with no intervention. |
| **Winter** | **Disable** `automation.sump_pump_not_running_alert` to avoid repeated alerts when the sump is intentionally not running. The other three automations can remain enabled. |

---

## Troubleshooting

**Automation never triggers**
- Verify the power sensor entity ID is correct
- Check the wattage threshold — confirm the sensor actually exceeds it when the sump runs (Developer Tools → States → watch the sensor while the sump runs)

**`sensor.sump_pump_interval` shows 0**
- `input_datetime.sump_last_run` has never been set — the sump hasn't run since install, or the datetime helper wasn't created. Check Developer Tools → States.

**`sensor.sump_pump_interval_rolling_average` shows `unknown`**
- Fewer than 5 sump cycles have occurred since install or since the last HA restart. The watchdog will use `max_minutes` as a fallback until it populates.

**Getting interval-changed alerts every run**
- The previous interval stored in `input_number.sump_last_interval` may be stale (e.g. from before install). Let it run a few cycles to normalize, or manually set `input_number.sump_last_interval` to a reasonable value (in seconds) via Developer Tools.

**Formatted sensors show `unknown`**
- If created as UI helpers, the state template must be pasted and saved via the UI editor — see Step 2, Option B. The API cannot write template expressions for this helper type.

**False "not running" alerts during HA restart**
- Normal on the first restart after install while the watchdog timer repopulates. The `sump_power_sensor_unavailable_alert` automation handles sensor offline scenarios with a 2-minute delay to absorb restart blips.

**Getting excessive runtime alerts unexpectedly**
- Check `input_number.sump_max_run_seconds` — it may be set too low for your pump's actual runtime. Watch a few sump cycles in Developer Tools → States to see the real duration, then set the threshold comfortably above that value.
- If the power sensor reads noisy around the threshold (bouncing above/below 50W during a run), the automation may record a shorter run than actual. Consider adding a `for: seconds: 2` to the `below:` trigger, or adjusting the wattage threshold.
