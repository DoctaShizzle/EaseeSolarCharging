# Easee Solar Charging

A Home Assistant configuration for smart EV charging with an Easee charging station. The current Node-RED flow is retained during the staged migration.

---

## Overview

The Easee built-in schedule is **disabled**. The Home Assistant automation directly replaces the Node-RED controller and writes to the charger when the car is connected and required control inputs are valid. Invalid or unavailable inputs produce a persistent notification and no new charger command.

**Cutover safety:** Stop or disable Node-RED's Easee writes before loading/enabling the Home Assistant controller. Never run both controllers against the same charger. Keep `nodered-flow.json` available for rollback; disable the HA controller before re-enabling Node-RED.

There are three modes selectable from the Home Assistant UI:

- **Default**  the permanent mode: follows a configurable weekday/weekend schedule, with automatic solar ramp-up when the house is exporting energy.
- **Boost**  session-only: charges at maximum speed immediately, regardless of schedule or solar. Resets to Default when the car unplugs.
- **Solar only**  session-only: follows solar export only, pausing completely when there is nothing to give. Resets to Default when the car unplugs.

Every 10 seconds the Home Assistant controller evaluates the current mode and adjusts the Easee dynamic current limit.

Two physical buttons on a Shelly Pro 2 relay provide convenient local control, with LEDs indicating the active mode.

When a car connects to the charger, a Home Assistant automation asks the phones currently at home to choose a mode. The first choice applies; if nobody responds, the current mode is left unchanged.

---

## Modes

### Default (permanent)

This is the normal, always-on behaviour. It combines scheduled charging with opportunistic solar charging:

The Home Assistant controller tracks **combined P1 + battery power** (net household power) by default. The `use_marstek_battery` constant can disable battery input; when disabled, battery power is treated as 0 W and control uses P1 alone. Every 10 seconds it calculates whether the house needs to import more, export less, or stay balanced—accounting for both grid flow and battery charge/discharge state when enabled. The charger is adjusted in small 1 A steps to reach a target net power position, preventing oscillation when conditions stabilize.

1. **Inside a schedule window** — `input_number.easee_schedule_floor_a` is the shared minimum current for every active schedule block. From there:
   - If combined `netPower` is **below `TARGET_NET_W`** (more surplus than the target), the current ramps **up** toward `input_number.easee_solar_max_a`.
   - If combined `netPower` is **above `TARGET_NET_W`** (more import or less export than the target), the current ramps **down** toward the schedule floor.
   - Small adjustments (< 1 A) are ignored to prevent jitter — only moves of ≥1 A are applied, allowing transient spikes to settle.

2. **Outside every schedule window** — the floor drops to 0 A, but the solar+battery tracking still runs with **EV-minimum hysteresis** (`CAR_MIN_A`, default 6 A):
   - The car either charges at ≥ `CAR_MIN_A` or not at all — values 1–5 A are never commanded (the car ignores them anyway).
   - **OFF → ON**: only when surplus can sustain `CAR_MIN_A` (i.e. `idealDeltaA ≥ CAR_MIN_A`, meaning ≥ 1380 W of net export beyond target).
   - **ON → OFF**: only when the unclamped ideal target drops to 0 or below (real sustained deficit).
   - Between those thresholds the charging state is retained; while charging, the setpoint can still ramp toward the target by up to `MAX_CHANGE_PER_CYCLE` per cycle. This avoids rapid start/stop cycling.

The active windows come from one Home Assistant Schedule helper, `schedule.easee_charging_schedule`. It supports different time blocks for each weekday, so a single helper covers both weekday and weekend schedules. The shared `input_number.easee_schedule_floor_a` value is the minimum current for every active block.

When the schedule floor is at least `CAR_MIN_A`, it is guaranteed during active blocks. If it is set below `CAR_MIN_A`, the EV-minimum hysteresis applies instead; the charger will not be commanded to charge at 1–5 A.

### Boost (session-only)

Sets the dynamic current limit to `input_number.easee_boost_max_a` (recommended starting value 32 A) immediately and keeps it there. The schedule and solar tracking are ignored. When the car unplugs, the mode automatically resets to **Default**.

> Note: because the Easee built-in schedule is disabled, the old "override schedule" button is not used here. The Home Assistant automation owns the current limit.

### Solar only (session-only)

Tracks solar export using `input_number.easee_solar_only_min_a` and `input_number.easee_solar_only_max_a`. With the recommended 0 A minimum, the car pauses when surplus cannot sustain the EV minimum. It resumes only when combined net power reaches the start threshold, `TARGET_NET_W - (CAR_MIN_A × PHASE_VOLTAGE)`. With the current defaults, that is `-1,480 W`, not merely any grid export. When the car unplugs, the mode automatically resets to **Default**.

---

## Auto-reset on car disconnect

When the Easee charger status changes to a non-connected state (e.g. car unplugged, standby), the active controller resets:
1. Resets `input_select.easee_charging_mode` to **Default**.
2. Resets `input_number.easee_dynamic_current` to **0 A** (off), ready for the next session.
3. Sends 0 A to the Easee circuit dynamic limit.

This means pressing **Boost** or **Solar only** is always a "this session" decision; you never have to remember to reset it.

---

## Shelly Pro 2 Integration

A [Shelly Pro 2](https://shelly.cloud/products/shelly-pro-2/) relay provides physical button control and LED feedback:

- **Button 1 (Input 0)**: Toggles between **Default** and **Solar only** modes.
- **Button 2 (Input 1)**: Toggles between **Default** and **Boost** modes.
- **LED 0 (Output 0)**: Lights up when mode is **Solar only**.
- **LED 1 (Output 1)**: Lights up when mode is **Boost**. Both LEDs are off in **Default** mode.

The buttons send `shelly.click` events to Home Assistant, which trigger automations that change the mode accordingly. A separate automation keeps the Shelly outputs (LEDs) in sync with the selected mode at all times, including on Home Assistant startup.

## Mode selection notification

The `automations.yaml` automation triggers when the internal `sensor.easee_control_charger_status` changes from `disconnected` and then confirms that the car is connected. It finds `device_tracker` entities from the Home Assistant Mobile App integration that are currently `home`, then derives the corresponding `notify.mobile_app_*` targets. Phone IDs are not listed in the automation. The Companion App tracker entity ID must retain the same device suffix as its notification service.

The notification has **Default**, **Boost**, and **Solar only** actions. The first response selects the mode and clears the prompt from the recipients. If the car disconnects or nobody chooses within 24 hours, the current mode is unchanged. Running the automation manually sends a 10-minute test notification and never changes the charging mode.

---

## Architecture

```
[Every 10s / connected status stable for 10s]
    ↓
[HA automation: validate mode, site mapping, P1 + Marstek, schedule, helpers]
    ↓
[Calculate next current]
    ↓
[easee.set_circuit_dynamic_limit]
    ↓
[input_number.easee_dynamic_current]

[Disconnected for 10s]
    ↓
[Reset mode → Default, setpoint → 0 A, Easee limit → 0 A]
```

Site-specific device IDs and external entity IDs are grouped in the `input_text` mapping in `configuration.yaml`. The HA automation keeps fixed control constants at its start; UI-adjustable amperage settings remain Home Assistant number helpers, and schedule windows remain in the Schedule helper.

---

## Customising the parameters

### Weekly schedule

Create one Schedule helper in Home Assistant named **Easee Charging Schedule**. Use its weekly editor to define blocks for each day, including different weekday and weekend times. Its entity ID should be `schedule.easee_charging_schedule`; keep this ID when moving homes so the HA controller and dashboard need no edits. The helper uses Home Assistant's configured timezone.

### UI-adjustable amperage settings

The following `input_number` helpers are adjustable in Home Assistant and are shown in the Lovelace card:

| Entity | Range | Recommended starting value |
|---|---:|---:|
| `input_number.easee_schedule_floor_a` | 0–32 A | 6 A |
| `input_number.easee_solar_max_a` | 0–32 A | 32 A |
| `input_number.easee_boost_max_a` | 0–32 A | 32 A |
| `input_number.easee_solar_only_min_a` | 0–32 A | 0 A |
| `input_number.easee_solar_only_max_a` | 0–32 A | 32 A |

`easee_schedule_floor_a` is a shared minimum for all active schedule blocks, not a cap. If it is below `CAR_MIN_A`, the EV-minimum hysteresis applies instead. On first startup, the HA initialization automation seeds these helpers with the recommended values above and marks `input_boolean.easee_charging_settings_initialized` on. Later UI changes are restored across Home Assistant restarts and are not overwritten.

### Fixed tweakable constants

The HA controller constants are grouped in the `variables` block at the start of `easee_home_assistant_charging_controller` in `automations.yaml`:

| Constant | Current value | Purpose |
|---|---:|---|
| `phase_voltage_v` | 230 | Converts the P1/Marstek power difference to an ampere adjustment. |
| `target_net_power_w` | -100 | Desired combined P1-minus-Marstek power. |
| `max_change_per_cycle_a` | 1 | Limits the change on each 10-second controller run. |
| `car_minimum_current_a` | 6 | EV start/stop hysteresis threshold. |
| `use_marstek_battery` | `true` | Set to `false` when no Marstek battery power is available; P1 alone is then used. |

Keep these fixed control constants together. Amperage floors and ceilings remain user-adjustable Home Assistant helpers; site-specific IDs belong in the mapping block in `configuration.yaml`.

**Finding your device ID**: in Home Assistant go to *Settings → Devices & Services → Easee*, open your charger device, and copy the device ID from the URL into `input_text.easee_site_charger_device_id`.

### Central site mapping

Edit only the `input_text` site-mapping block near the start of `configuration.yaml` when adapting the setup to another home. These mapping helpers are created from YAML; no manual helper setup is needed. They hold integration-provided IDs, while template sensors expose stable internal entities so automations and the dashboard do not repeat home-specific entity IDs. The weekly Schedule helper is the only UI-created exception.

| Mapping helper | Value to configure |
|---|---|
| `input_text.easee_site_charger_device_id` | Easee device ID used by `easee.set_circuit_dynamic_limit`. |
| `input_text.easee_site_charger_status_entity` | Source Easee charger-status entity. |
| `input_text.easee_site_charger_power_entity` | Source Easee charger-power entity; wrapper normalizes to kW and displays two decimal places (for example, 1.373 kW as 1.37 kW). |
| `input_text.easee_site_charger_session_energy_entity` | Source Easee session-energy entity; wrapper normalizes to kWh and displays one decimal place without rounding its underlying state. |
| `input_text.easee_site_p1_power_entity` | Source P1 instantaneous import/export power entity in W. |
| `input_text.easee_site_marstek_power_1_entity` / `input_text.easee_site_marstek_power_2_entity` | Power entities for the two Marstek meter plugs, in W. |
| `input_text.easee_site_shelly_device_id` | Device ID reported by the Shelly `shelly.click` event. |
| `input_text.easee_site_shelly_output_0_entity` / `_1_entity` | Shelly output switch entities used for the mode LEDs. |

Unavailable or non-numeric power sources make the corresponding internal sensor unavailable. The controller then notifies and skips the charger write instead of assuming missing power is zero.

---

## Files

| File | Purpose |
|---|---|
| `configuration.yaml` | Central site-ID/entity mapping, stable template sensors, Home Assistant helpers, and mode scripts. Integrate with your existing configuration. |
| `automations.yaml` | One-time settings initialization, the 10-second charging controller, disconnect reset, Shelly automations, and mobile mode notification. |
| `ui.yaml` | Lovelace card with mode buttons, stable internal charger sensors, schedule, and editable charging settings. Paste into a manual card. |
| `nodered-flow.json` | Previous controller retained only as a rollback reference after direct cutover. |

---

## Home Assistant setup

1. Copy `automations.yaml` beside your Home Assistant `configuration.yaml`, then integrate this repository's configuration. Update the YAML-defined `input_text` site-mapping values for the Easee device ID, source entities, and Shelly ID/entities.
2. In **Settings → Devices & Services → Helpers**, create the **Easee Charging Schedule** helper with entity ID `schedule.easee_charging_schedule` and configure weekday/weekend blocks.
3. Ensure each phone runs the Home Assistant Companion App and has a Mobile App `device_tracker` that reports `home`; the automation discovers these devices automatically.
4. Before reloading Home Assistant or enabling the new automations, stop/disable Node-RED's Easee writes. This is a direct replacement; do not run both writers together.
5. Restart Home Assistant, confirm configuration validation succeeds, and check the persistent notifications and automation traces. The HA initialization automation seeds the five amperage helpers once; later user changes are preserved.
6. Add the card in `ui.yaml` to a Lovelace dashboard (Edit dashboard → Add card → Manual → paste YAML).
7. **Disable the Easee schedule** in the Easee app (or set it to always allow charging; the HA automation manages limits through the dynamic current limit API).

## Node-RED rollback reference

Keep `nodered-flow.json` available until the HA controller is accepted. If rolling back, first disable the HA charging-controller and disconnect-reset automations, then re-enable Node-RED. Never leave both controllers writing to Easee.

---

## How solar + battery control works

The HA controller runs every 10 seconds and reads P1 power (grid), the Marstek battery power sensor, the Home Assistant setpoint helper, schedule state, and amperage settings. `use_marstek_battery` defaults to `true` in the controller constants; if set to `false`, the controller uses 0 W for battery power instead. Its combined-power calculation is:

### P1 Power (grid)
- **Positive** = the house is importing from the grid
- **Negative** = the house is exporting to the grid (solar surplus)

### Battery Power (Marstek)
- **Positive** = batteries are charging (consuming household power)
- **Negative** = batteries are discharging (supplying household power)

### Combined Net Power
```
netPower = P1 − Battery
```

This interpretation means:
- P1 = −500W (exporting) + Battery = +300W (charging) → netPower = −800W (strong export signal → increase charger)
- P1 = +500W (importing) + Battery = −300W (discharging) → netPower = +800W (strong import signal → decrease charger)

### Adjustment Logic

Every 10 seconds, the controller uses the **setpoint helper** (`input_number.easee_dynamic_current`)—what we commanded last cycle—together with the latest P1 and Marstek readings.

It calculates the ideal adjustment:

```
netPower = p1Power − marstek
idealDeltaA = (TARGET_NET_W − netPower) / PHASE_VOLTAGE
limitedDeltaA = (abs(idealDeltaA) < 1A) ? 0 : clamp(idealDeltaA, −MAX_CHANGE_PER_CYCLE, +MAX_CHANGE_PER_CYCLE)
new_setpoint = setpoint_helper + limitedDeltaA
```

This dual approach ensures:
- **Smooth ramping**: ±1A per cycle limit based on our last command
- **Accurate feedback**: even if the charger enforces a 6A minimum when we set 0A, the next cycle reads actual power and adjusts correctly

| Condition | Action |
|---|---|
| netPower far above target (strong import) | Current decreases by 1 A (toward floor) |
| netPower far below target (strong export) | Current increases by 1 A (toward ceiling) |
| netPower close to target (within ±PHASE_VOLTAGE W) | **Current stays unchanged** — 1A dead-band prevents jitter |
| Floor limit (Default) | `input_number.easee_schedule_floor_a` inside an active schedule block; 0 A outside |
| Ceiling limit (Default & Solar only) | `input_number.easee_solar_max_a` and `input_number.easee_solar_only_max_a` respectively |

The **1 A dead-band** prevents oscillation when the house is close to target. Brief cloud shadows or battery fluctuations won't trigger charger changes — only sustained deviation does.

### EV-minimum hysteresis (`CAR_MIN_A`)

Outside schedule windows (floor = 0 A), the car's physical minimum charging current creates a step function: it either draws ≥ `CAR_MIN_A` × `PHASE_VOLTAGE` or nothing. The algorithm uses the household power readings and the last commanded setpoint to handle this minimum and avoid repeatedly starting and stopping the charger.

The setpoint is stored in `input_number.easee_dynamic_current` so the UI shows the live target and the value survives a Home Assistant restart.

**Default mode** uses `input_number.easee_schedule_floor_a` as its floor during active schedule blocks. Outside blocks the floor is 0 A, so the car only charges if there is sufficient combined surplus.

**Solar only mode** also allows 0 A floor — the car halts completely when there is insufficient surplus, and restarts automatically when it returns.
