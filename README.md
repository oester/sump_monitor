# Sump Pump Monitor for Home Assistant

A complete Home Assistant monitoring package for sump pumps that uses power
consumption to detect pump activity, track run intervals, and send intelligent
push notifications when something changes or the pump stops running.

---

## Features

- **Excessive runtime detection** — alerts if a single pump run exceeds a configurable threshold, indicating a possible stuck float switch or pump problem
- **Interval change detection** — alerts when the sump run interval changes
  suddenly by more than 25%, indicating a possible problem
- **Adaptive watchdog** — automatically tightens during high-frequency cycling
  (heavy rain) and relaxes as conditions normalize over days; alerts if the
  pump goes unexpectedly silent
- **Power sensor monitoring** — detects when the power sensor itself goes
  offline and suppresses false alarms during outages
- **Spring startup notification** — sends a friendly "Sump Resumed" message
  after winter inactivity instead of a misleading large-percentage-change alert
- **Human-readable display sensors** — formatted `HH:MM` and `N days, HH:MM`
  sensors for use in dashboards and badges
- **Daily cycle tracking** — a self-resetting daily counter plus an hourly
  rate sensor for judging storm intensity at a glance, and a simple
  running/not-running indicator sensor
- **Self-resetting counters** — cycle counters reset automatically at
  midnight, no manual reset needed
- **Fully tunable** — watchdog sensitivity controlled by three sliders, no
  YAML editing required after install

---

## How It Works

The system uses a **power monitoring sensor** on the sump pump circuit as its
primary signal. When the sensor exceeds a configurable wattage threshold, the
sump is considered to be running.

Seven automations work together:

### 1. Sump Pump Interval Monitor
Fires on every sump run. Tracks the time since the last run, stores it as the
current interval, and alerts if the interval changes by more than 25% — a sign
of an unexpected change in conditions. After a long winter absence, sends a
"Sump Pump Resumed" notification instead.

### 2. Sump Pump Not Running Alert
After each sump run, sets a watchdog timer to:

```
min( max( rolling_avg × multiplier, min_minutes ), max_minutes )
```

If the timer expires before the next run, an alert fires. The rolling average
of the last 5 intervals means the watchdog naturally tracks the current
cycling pattern — tight during rain, relaxed during dry periods.

### 3. Sump Power Sensor Unavailable Alert
Watches the power sensor for `unavailable` or `unknown` states. If the sensor
goes offline for more than 2 minutes, cancels the watchdog timer (preventing
false alarms) and sends an alert. Sends a recovery notification and restarts
the watchdog when the sensor comes back.

### 4. Sump Pump Excessive Runtime Alert
Records the start time when the sump begins running. When it stops, calculates
run duration and alerts if it exceeded `input_number.sump_max_run_seconds`
(default 30s). A stuck float switch or failing pump will often cause an
unusually long run before stalling.

### 5. Sump Pump Cycle Counter Reset
Resets `counter.sump_pump_cycles` to zero every night at midnight, so it
always shows today's cycle count instead of accumulating forever.

### 6. Sump Day Count Increment
Increments a second, independent counter (`counter.sump_day_count`) every
time the sump starts running. Exists specifically to feed the hourly rate
sensor below.

### 7. Sump Day Count Reset
Resets `counter.sump_day_count` shortly after midnight, keeping it and the
rate sensor scoped to today only.

---

## Requirements

- Home Assistant 2024.1 or later
- A power monitoring sensor on the sump pump circuit reporting wattage
  (e.g. Emporia Vue, Shelly EM, Z-Wave power meter, smart plug with power monitoring)
- Home Assistant companion app installed on a mobile device for push notifications

---

## Files

### `automations/`
| File | Description |
|---|---|
| `sump_pump_interval_monitor.yaml` | Automation 1 — interval change detection and spring startup |
| `sump_pump_not_running_alert.yaml` | Automation 2 — adaptive watchdog for complete stoppage |
| `sump_power_sensor_unavailable_alert.yaml` | Automation 3 — alerts when the power sensor goes offline |
| `sump_pump_excessive_runtime_alert.yaml` | Automation 4 — alerts if a single run exceeds a threshold |
| `sump_pump_cycle_counter_reset.yaml` | Automation 5 — resets the cycle counter nightly |
| `sump_day_count_increment.yaml` | Automation 6 — increments the daily counter on every run |
| `sump_day_count_reset.yaml` | Automation 7 — resets the daily counter nightly |

### `helpers/`
| File | Description |
|---|---|
| `sump_input_datetime.yaml` | Input datetime helpers — last run and run start timestamps |
| `sump_input_number.yaml` | Input number helpers — interval storage and watchdog thresholds |
| `sump_counter.yaml` | Counter helpers — today's cycle count and the daily counter feeding the rate sensor |
| `sump_timer.yaml` | Timer helper — watchdog timer |
| `sump_template_sensors.yaml` | Template sensors — interval, time-since, formatted display, and running-indicator sensors |
| `statistics_sensor.yaml` | Instructions for the rolling average statistics sensor (UI-only creation) |

### Root
| File | Description |
|---|---|
| `derivative_sensor.yaml` | Instructions for the daily cycle rate sensor (UI-only creation) |
| `INSTALL.md` | Full step-by-step installation and customization guide |
| `README.md` | This file |

---

## Quick Start

1. **Identify** your sump power sensor entity ID and the wattage it reads when
   the pump is running
2. **Add helpers** from the `helpers/` directory to your HA config (or create via UI)
3. **Create** the rolling average and daily rate sensors via the HA UI (see
   `helpers/statistics_sensor.yaml` and `derivative_sensor.yaml`)
4. **Add automations** from the `automations/` directory to your HA config
5. **Replace** the placeholder values marked `# <<< CONFIGURE` in each
   automation file and in the "Sump Running" template sensor:
   - `sensor.your_sump_power_sensor` → your power sensor entity ID
   - `50` (wattage threshold) → your pump's running wattage threshold
   - `notify.mobile_your_device` → your notification service

   `sump_pump_cycle_counter_reset.yaml` and `sump_day_count_reset.yaml`
   need no customization.

See [INSTALL.md](INSTALL.md) for the full guide including tuning, seasonal
operation, and troubleshooting.

---

## Tuning

Three `input_number` helpers control watchdog sensitivity. Add them as sliders
to any dashboard for easy adjustment — no YAML editing required.

| Helper | Default | Purpose |
|---|---|---|
| `input_number.sump_alert_multiplier` | 2.5× | How many times the rolling average interval to wait before alerting |
| `input_number.sump_alert_min_minutes` | 30 min | Minimum watchdog duration — floor during heavy rain cycling |
| `input_number.sump_alert_max_minutes` | 240 min | Maximum watchdog duration — ceiling and fallback when no history exists |
| `input_number.sump_max_run_seconds` | 30s | Maximum single run duration before excessive runtime alert fires |

---

## Seasonal Operation

| Season | Action |
|---|---|
| **Spring** | Enable `automation.sump_pump_not_running_alert` |
| **Summer / Fall** | All automations run year-round, no intervention needed |
| **Winter** | Disable `automation.sump_pump_not_running_alert` to prevent repeated alerts |

The Interval Monitor, Sensor Unavailable, and Excessive Runtime automations can remain enabled year-round.

---

## Post-Rain Behavior

The sump naturally cycles frequently during heavy rain and slows over
days/weeks as the water table drops. The rolling average tracks this gradual
change so the watchdog extends naturally — it will not false-alert during a
normal post-rain slowdown. Only a *sudden unexpected stop* against the current
baseline triggers an alert.

---

## Notifications

| Scenario | Notification |
|---|---|
| Interval changed suddenly | *"Sump Pump Interval Changed — changed by 32.4%. Previous: 35 min, Current: 46 min."* |
| Pump stopped running | *"⚠️ Sump Pump May Not Be Running — No activity for 94 min. Last interval: 37 min."* |
| Excessive single run | *"⚠️ Sump Pump Excessive Runtime — Sump ran for 47.3 seconds (threshold: 30s). Possible pump or float switch issue."* |
| Power sensor offline | *"⚠️ Sump Power Sensor Offline — monitoring inactive until recovered."* |
| Power sensor recovered | *"✅ Sump Power Sensor Back Online — monitoring resumed."* |
| Spring startup | *"Sump Pump Resumed — first run in 1,127 hours. Monitoring is now active."* |

---

## License

MIT — free to use, modify, and share.
