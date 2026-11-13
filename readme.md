# Easee Solar Charging

A Home Assistant + Node-RED solution for smart EV charging with an Easee charging station.

---

## Overview

The Easee built-in schedule is **disabled**. Node-RED takes full control of when and how fast the car charges. There are three modes selectable from the Home Assistant UI:

- **Default**  the permanent mode: follows a configurable weekday/weekend schedule, with automatic solar ramp-up when the house is exporting energy.
- **Boost**  session-only: charges at maximum speed immediately, regardless of schedule or solar. Resets to Default when the car unplugs.
- **Solar only**  session-only: follows solar export only, pausing completely when there is nothing to give. Resets to Default when the car unplugs.

Every 10 seconds Node-RED evaluates the current mode and adjusts the Easee dynamic current limit accordingly.

Two physical buttons on a Shelly Pro 2 relay provide convenient local control, with LEDs indicating the active mode.

---

## Modes

### Default (permanent)

This is the normal, always-on behaviour. It combines scheduled charging with opportunistic solar charging:

The Brain tracks **combined P1 + battery power** (net household power). Every 10 seconds it calculates whether the house needs to import more, export less, or stay balanced—accounting for both grid flow and battery charge/discharge state. The charger is adjusted in small 1 A steps to reach a target net power position, preventing oscillation when conditions stabilize.

1. **Inside a schedule window** — the current floors at the slot's configured `maxA`. From there:
   - If the house is **exporting surplus** (P1 negative **and** batteries are helping), the current ramps **up** toward `input_number.easee_solar_max_a`.
   - If the house is **importing too much** (P1 positive **or** batteries are draining), the current ramps **down** toward the floor.
   - Small adjustments (< 1 A) are ignored to prevent jitter — only moves of ≥1 A are applied, allowing transient spikes to settle.

2. **Outside every schedule window** — the floor drops to 0 A, but the solar+battery tracking still runs with **EV-minimum hysteresis** (`CAR_MIN_A`, default 6 A):
   - The car either charges at ≥ `CAR_MIN_A` or not at all — values 1–5 A are never commanded (the car ignores them anyway).
   - **OFF → ON**: only when surplus can sustain `CAR_MIN_A` (i.e. `idealDeltaA ≥ CAR_MIN_A`, meaning ≥ 1380 W of net export beyond target).
   - **ON → OFF**: only when the unclamped ideal target drops to 0 or below (real sustained deficit).
   - Between those thresholds the charger **holds** at its current level — no oscillation.

The active windows come from one Home Assistant Schedule helper, `schedule.easee_charging_schedule`. It supports different time blocks for each weekday, so a single helper covers both weekday and weekend schedules. The shared `input_number.easee_schedule_floor_a` value is the minimum current for every active block.

When the schedule floor is at least `CAR_MIN_A`, it is guaranteed during active blocks. If it is set below `CAR_MIN_A`, the EV-minimum hysteresis applies instead; the charger will not be commanded to charge at 1–5 A.

### Boost (session-only)

Sets the dynamic current limit to `input_number.easee_boost_max_a` (recommended starting value 32 A) immediately and keeps it there. The schedule and solar tracking are ignored. When the car unplugs, the mode automatically resets to **Default**.

> Note: because the Easee built-in schedule is disabled, the old "override schedule" button is not used here. Node-RED owns the current limit directly.

### Solar only (session-only)

Tracks solar export using `input_number.easee_solar_only_min_a` and `input_number.easee_solar_only_max_a`. With the recommended 0 A minimum, the car pauses when there is no solar and resumes automatically as soon as the house starts exporting again. When the car unplugs, the mode automatically resets to **Default**.

---

## Auto-reset on car disconnect

When the Easee charger status changes to a non-connected state (e.g. car unplugged, standby), a state-change trigger in Node-RED fires and:
1. Resets `input_select.easee_charging_mode` to **Default**.
2. Resets `input_number.easee_dynamic_current` to **0 A** (off), ready for the next session.
3. Sends 0 A to the Easee circuit dynamic limit.

This means pressing **Boost** or **Solar only** is always a "this session" decision  you never have to remember to reset it.

---

## Shelly Pro 2 Integration

A [Shelly Pro 2](https://shelly.cloud/products/shelly-pro-2/) relay provides physical button control and LED feedback:

- **Button 1 (Input 0)**: Toggles between **Default** and **Solar only** modes.
- **Button 2 (Input 1)**: Toggles between **Default** and **Boost** modes.
- **LED 0 (Output 0)**: Lights up when mode is **Solar only**.
- **LED 1 (Output 1)**: Lights up when mode is **Boost**. Both LEDs are off in **Default** mode.

The buttons send `shelly.click` events to Home Assistant, which trigger automations that change the mode accordingly. A separate automation keeps the Shelly outputs (LEDs) in sync with the selected mode at all times, including on Home Assistant startup.

---

## Architecture

```
[Every 10s]
    ↓
[Tweakable constants]
    ↓
[Get mode]    input_select.easee_charging_mode
    ↓
[Car connected?]    sensor.easee_charger_status
      (not connected → stop)
    ↓
[Get P1 power (W)]    sensor.p1_meter_power_import_power (instant reading)
[Get battery power (W)]    sensor.marstek_power_import (instant reading)
    ↓
[Get current setpoint (A)]    input_number.easee_dynamic_current
    ↓
[Get schedule state]    schedule.easee_charging_schedule
[Get amperage settings]    input_number.easee_*
    ↓
[Brain — control logic]
  Combines: netPower = P1 − Battery
  Targets: TARGET_NET_W
  Limits: ±1A per cycle, ±1A dead-band
  Hysteresis: CAR_MIN_A on/off threshold
    ↓
[easee.set_circuit_dynamic_limit]
    ↓
[input_number.set_value]  (keep HA helper in sync)


[Car connected?]    sensor.easee_charger_status (state-change trigger)
         |
     (connected)                  (disconnected)
         |                              |
[immediate evaluation]      [Reset mode → Default]
                            [Reset setpoint → 0 A]
                            [Reset Easee limit → 0 A]
```

The **Brain** function node contains the charging logic. Fixed tweakable constants live in the `Tweakable constants` function node; UI-adjustable amperage settings live in Home Assistant number helpers, and schedule windows live in the Schedule helper.

---

## Customising the parameters

### Weekly schedule

Create one Schedule helper in Home Assistant named **Easee Charging Schedule**. Use its weekly editor to define blocks for each day, including different weekday and weekend times. Its entity ID should be `schedule.easee_charging_schedule`; if you use another ID, update the `Get schedule state` node in Node-RED. The helper uses Home Assistant's configured timezone.

### UI-adjustable amperage settings

The following `input_number` helpers are adjustable in Home Assistant and are shown in the Lovelace card:

| Entity | Range | Recommended starting value |
|---|---:|---:|
| `input_number.easee_schedule_floor_a` | 0–32 A | 16 A |
| `input_number.easee_solar_max_a` | 0–32 A | 32 A |
| `input_number.easee_boost_max_a` | 0–32 A | 32 A |
| `input_number.easee_solar_only_min_a` | 0–32 A | 0 A |
| `input_number.easee_solar_only_max_a` | 0–32 A | 32 A |

`easee_schedule_floor_a` is a shared minimum for all active schedule blocks, not a cap. If it is below `CAR_MIN_A`, the EV-minimum hysteresis applies instead. On the first Node-RED tick, the flow seeds these helpers with the recommended values above and marks `input_boolean.easee_charging_settings_initialized` on. Later UI changes are restored across Home Assistant restarts and are not overwritten.

### Fixed tweakable constants

The `Tweakable constants` function node at the start of the Node-RED flow holds the charger device ID, phase voltage, target net power, maximum current change per cycle, and EV minimum current. Edit those constants there rather than in the Brain. The schedule uses Home Assistant's timezone, so no Node-RED timezone constant is needed.

**Finding your device ID**: in Home Assistant go to *Settings → Devices & Services → Easee*, open your charger device, and copy the device ID from the URL.

---

## Files

| File | Purpose |
|---|---|
| `configuration.yaml` | Home Assistant helpers (`input_select`, `input_number`), mode-change scripts, Shelly button event automations, and LED sync logic. Merge into your `configuration.yaml`. |
| `ui.yaml` | Lovelace card definition with mode buttons, charger sensors, and editable charging settings. Paste into a manual card. |
| `nodered-flow.json` | Node-RED flow with the 10-second evaluation cycle and Brain logic. Import via *Menu → Import* in Node-RED. |

---

## Home Assistant setup

1. Merge the contents of `configuration.yaml` into your Home Assistant `configuration.yaml`.
2. Restart Home Assistant.
3. In **Settings → Devices & Services → Helpers**, create an **Easee Charging Schedule** helper and configure its weekday/weekend blocks.
4. The first Node-RED tick seeds the five amperage helpers with the recommended starting values from the table. Adjust them from the dashboard afterwards if needed.
5. Add the card in `ui.yaml` to a Lovelace dashboard (Edit dashboard → Add card → Manual → paste YAML).
6. **Disable the Easee schedule** in the Easee app (or set it to always allow charging; Node-RED manages limits directly via the dynamic current limit API).

## Node-RED setup

1. Make sure **node-red-contrib-home-assistant-websocket** is installed in Node-RED.
2. Import `nodered-flow.json` via *Menu  Import*.
3. Open the **Home Assistant** server node and point it at your HA instance if needed.
4. Deploy.
5. Open the `Tweakable constants` function node to edit its fixed constants if needed. Adjust the amperage settings in Home Assistant.
6. Deploy again.

---

## How solar + battery control works

The algorithm runs every 10 seconds and reads P1 power (grid), battery power (Marstek), the Home Assistant setpoint helper, schedule state, and amperage settings. It treats the grid and battery readings as a combined system:

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

Every 10 seconds, the Brain uses the **setpoint helper** (`input_number.easee_dynamic_current`)—what we commanded last cycle—together with the latest P1 and Marstek readings.

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

The setpoint is stored in `input_number.easee_dynamic_current` so the UI shows the live target and the value survives a Node-RED restart.

**Default mode** uses `input_number.easee_schedule_floor_a` as its floor during active schedule blocks. Outside blocks the floor is 0 A, so the car only charges if there is sufficient combined surplus.

**Solar only mode** also allows 0 A floor — the car halts completely when there is insufficient surplus, and restarts automatically when it returns.
