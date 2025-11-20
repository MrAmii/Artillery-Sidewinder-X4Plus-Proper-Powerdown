# Artillery-Sidewinder-X4Plus-Proper-Powerdown
This will allow you to power down your system responsibly. Once executed, allow ten seconds before flipping the power switch off.


# Klipper Host Power-Off Integration (Artillery X4 Plus)

This document describes the working configuration used to:
- Power off the Klipper host immediately from a macro.
- Schedule a shutdown **after a delay in hours**.
- Schedule a shutdown **at a specific clock time** like `11:59PM` or `23:10`.
- Cancel any pending shutdowns.

It uses:
- A Klipper config fragment: `power_off.cfg`
- A helper script: `/usr/local/bin/schedule_poweroff_at.sh`
- Moonraker's HTTP API (`curl` to `127.0.0.1:7125`) to trigger `POWER_OFF`.

> Note: This assumes your host user is allowed to run `systemctl poweroff`
> (on the stock image used here, that works as-is; on other systems you may
> need sudo/polkit changes).

---

## 1. Create the helper script

Create `/usr/local/bin/schedule_poweroff_at.sh`:

```bash
sudo nano /usr/local/bin/schedule_poweroff_at.sh
```

Paste:

```bash
#!/usr/bin/env bash
set -euo pipefail

# Accept: "11:59PM", "7pm", "23:10"
[[ $# -ge 1 ]] || { echo "Could not parse time: <none>"; exit 2; }
RAW="$*"

now_s=$(date +%s)

# Normalize "7pm" -> "7:00pm"
norm="$(echo "$RAW" | tr 'A-Z' 'a-z')"
if [[ "$norm" =~ ^([0-9]{1,2})(am|pm)$ ]]; then
  norm="${BASH_REMATCH[1]}:00${BASH_REMATCH[2]}"
fi

ts_today=$(date -d "today $norm" +%s 2>/dev/null || true)
[[ -n "$ts_today" ]] || { echo "Could not parse time: $RAW"; exit 2; }

target_s=$ts_today
(( target_s > now_s )) || target_s=$(date -d "tomorrow $norm" +%s)

delay=$(( target_s - now_s ))
echo "Scheduled POWER_OFF in ${delay}s for $(date -d @"$target_s")"

nohup bash -c "sleep ${delay}; curl -s --globoff 'http://127.0.0.1:7125/printer/gcode/script?script=POWER_OFF' >/dev/null" \
  >/dev/null 2>&1 & disown
```

Make it executable:

```bash
sudo chmod +x /usr/local/bin/schedule_poweroff_at.sh
```

---

## 2. Create the Klipper config fragment

Create `power_off.cfg` in your Klipper config directory, e.g.:

```bash
sudo nano /home/mks/printer_data/config/power_off.cfg
```

Paste:

```ini
# ================================
# POWER OFF AT – unified config
# ================================

# Immediate host shutdown (assumes permissions already set)
[gcode_shell_command HOST_POWEROFF]
command: bash -lc 'systemctl poweroff'
timeout: 5.0
verbose: True

[gcode_macro POWER_OFF]
description: Power off the host immediately (requires FORCE=1 if printing/paused/complete)
gcode:
  {% set state = printer.print_stats.state %}
  {% if state in ['printing','paused','complete'] and (params.FORCE|default(0)|int) != 1 %}
    RESPOND TYPE=error MSG="Job active. Use POWER_OFF FORCE=1 to override."
  {% else %}
    RUN_SHELL_COMMAND CMD=HOST_POWEROFF
  {% endif %}

# Delayed path used by quick-delay buttons
[delayed_gcode DELAYED_POWER_OFF]
gcode:
  RESPOND TYPE=echo MSG="DELAYED_POWER_OFF firing now."
  POWER_OFF

# ---- POWER_OFF_IN ----
[gcode_macro POWER_OFF_IN]
description: Schedule shutdown by decimal HOURS only (e.g. HOURS=1.5)
gcode:
  {% if params.HOURS is defined %}
    RESPOND TYPE=command MSG="action:prompt_end"
    {% set HOURS = params.HOURS|float %}
    {% if HOURS <= 0 %}
      RESPOND TYPE=error MSG="POWER_OFF_IN needs HOURS > 0 (e.g. 1.5)"
    {% else %}
      {% set S = (HOURS * 3600)|int %}
      UPDATE_DELAYED_GCODE ID=DELAYED_POWER_OFF DURATION={S}
      RESPOND TYPE=echo MSG="Shutdown scheduled in { '%.2f'|format(HOURS) } hour(s)."
    {% endif %}
  {% else %}
    RESPOND TYPE=command MSG="action:prompt_begin Power-Off Delay"
    RESPOND TYPE=command MSG="action:prompt_text Enter decimal HOURS (e.g. 0.25, 1.5, 2.0)"
    RESPOND TYPE=command MSG="action:prompt_input POWER_OFF_IN HOURS|primary"
    RESPOND TYPE=command MSG="action:prompt_button Close|action:prompt_end"
    RESPOND TYPE=command MSG="action:prompt_show"
  {% endif %}

# ---- POWER_OFF_AT ----
[gcode_macro POWER_OFF_AT]
description: Schedule shutdown at a clock time. Example: POWER_OFF_AT TIME=11:59PM
gcode:
  {% if params.TIME is defined %}
    RESPOND TYPE=command MSG="action:prompt_end"
    RUN_SHELL_COMMAND CMD=SCHEDULE_POWER_OFF_AT PARAMS="{params.TIME}"
  {% else %}
    RESPOND TYPE=command MSG="action:prompt_begin Shutdown Timer"
    RESPOND TYPE=command MSG="action:prompt_text Type:  POWER_OFF_AT TIME=5:30AM  or  TIME=23:10"
    RESPOND TYPE=command MSG="action:prompt_text Or pick a quick HOURS delay:"
    RESPOND TYPE=command MSG="action:prompt_button +0.5h|POWER_OFF_IN HOURS=0.5|primary"
    RESPOND TYPE=command MSG="action:prompt_button +1.0h|POWER_OFF_IN HOURS=1.0|primary"
    RESPOND TYPE=command MSG="action:prompt_button +1.5h|POWER_OFF_IN HOURS=1.5"
    RESPOND TYPE=command MSG="action:prompt_button +2.0h|POWER_OFF_IN HOURS=2.0"
    RESPOND TYPE=command MSG="action:prompt_button Close|action:prompt_end"
    RESPOND TYPE=command MSG="action:prompt_show"
  {% endif %}

[gcode_macro _POA_CONSOLE]
gcode:
  RESPOND TYPE=command MSG="action:prompt_end"
  RESPOND TYPE=echo MSG="POWER_OFF_AT TIME="
  RESPOND TYPE=echo MSG="(Type the time right after TIME=, e.g. 5:30AM or 23:10)"

# Bridge to helper script.
[gcode_shell_command SCHEDULE_POWER_OFF_AT]
command: bash /usr/local/bin/schedule_poweroff_at.sh
timeout: 10.0
verbose: True

# Cancel all timers (Klipper delayed + host background job)
[gcode_macro CANCEL_POWER_OFF]
description: Cancel delayed and host-scheduled shutdowns
gcode:
  UPDATE_DELAYED_GCODE ID=DELAYED_POWER_OFF DURATION=0
  RUN_SHELL_COMMAND CMD=CANCEL_HOST_POWER_OFF
  RESPOND TYPE=echo MSG="Canceled any pending POWER_OFF timers."

[gcode_shell_command CANCEL_HOST_POWER_OFF]
command: bash -lc 'pkill -f /usr/local/bin/schedule_poweroff_at.sh >/dev/null 2>&1 || true; pkill -f "printer/gcode/script?script=POWER_OFF" >/dev/null 2>&1 || true'
timeout: 5.0
verbose: True
```

Save and exit.

Then make sure this file is included in your main Klipper config (if not already).  
In `printer.cfg` or `macros.cfg`, add:

```ini
[include power_off.cfg]
```

Restart Klipper:

```bash
sudo systemctl restart klipper
```

---

## 3. Usage (buttons in Fluidd/Mainsail interface)

### Immediate power off

```gcode
POWER_OFF
```

- If a job is `printing`, `paused`, or `complete`, this **refuses** to shut down unless you override:

```gcode
POWER_OFF FORCE=1
```

Use the `FORCE=1` override carefully.

---

### Schedule shutdown after N hours

```gcode
POWER_OFF_IN HOURS=1.5
```

Examples:

- `POWER_OFF_IN HOURS=0.25`  → 15 minutes  
- `POWER_OFF_IN HOURS=1.0`   → 1 hour  
- `POWER_OFF_IN HOURS=2.0`   → 2 hours  

If `HOURS` is omitted, an interactive Fluidd/Mainsail prompt is opened so you can type the value.

---

### Schedule shutdown at a specific clock time

```gcode
POWER_OFF_AT TIME=11:59PM
POWER_OFF_AT TIME=5:30AM
POWER_OFF_AT TIME=23:10
```

The `POWER_OFF_AT` macro:

- Forwards the `TIME` string to `schedule_poweroff_at.sh`.
- The script:
  - Parses the time (supports `7pm`, `11:59PM`, `23:10`, etc.).
  - Chooses **today** if the time is still in the future, otherwise **tomorrow**.
  - Starts a background `sleep` and then calls:

    ```bash
    curl -s --globoff 'http://127.0.0.1:7125/printer/gcode/script?script=POWER_OFF'
    ```

If `TIME` is omitted, the macro opens a UI prompt with examples and quick buttons for `+0.5h`, `+1.0h`, etc., wired through `POWER_OFF_IN`.

---

### Cancel any pending shutdown

```gcode
CANCEL_POWER_OFF
```

This:

- Clears the Klipper delayed gcode timer `DELAYED_POWER_OFF`.
- Kills any background `schedule_poweroff_at.sh` jobs.
- Kills any queued `printer/gcode/script?script=POWER_OFF` calls.

You should see:

```text
Canceled any pending POWER_OFF timers.
```

---

## 4. Notes and Portability

- The script assumes Moonraker is on `http://127.0.0.1:7125`.  
  If your setup uses a different host/port, adjust the `curl` URL.
- If `systemctl poweroff` requires sudo on your system, you may need a sudoers rule and change:

  ```ini
  command: bash -lc 'systemctl poweroff'
  ```

  to something like:

  ```ini
  command: bash -lc 'sudo -n systemctl poweroff'
  ```

  plus the appropriate sudoers entry.

This configuration has been tested in a real setup and supports both
delay-based and clock-time-based shutdown, with safeguards and an
explicit cancel path.
