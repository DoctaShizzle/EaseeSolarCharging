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
   - If the house is **exporting surplus** (P1 negative **and** batteries are helping), the current ramps **up** toward `SOLAR_MAX_A`.
   - If the house is **importing too much** (P1 positive **or** batteries are draining), the current ramps **down** toward the floor.
   - Small adjustments (< 1 A) are ignored to prevent jitter — only moves of ≥1 A are applied, allowing transient spikes to settle.

2. **Outside every schedule window** — the floor drops to 0 A, but the solar+battery tracking still runs with **EV-minimum hysteresis** (`CAR_MIN_A`, default 6 A):
   - The car either charges at ≥ `CAR_MIN_A` or not at all — values 1–5 A are never commanded (the car ignores them anyway).
   - **OFF → ON**: only when surplus can sustain `CAR_MIN_A` (i.e. `idealDeltaA ≥ CAR_MIN_A`, meaning ≥ 1380 W of net export beyond target).
   - **ON → OFF**: only when the unclamped ideal target drops to 0 or below (real sustained deficit).
   - Between those thresholds the charger **holds** at its current level — no oscillation.

The schedule floor (`slot.maxA`) is guaranteed during windows — the schedule rate is always met even when clouds appear or batteries are in use.

| Day type | Default window(s) | Floor (maxA) |
|---|---|---|
| Weekday | 01:00  07:00 | 16 A |
| Weekend | 01:00  07:00 and 11:00  17:00 | 16 A |

These values are easy to change  see [Customising the parameters](#customising-the-parameters).

### Boost (session-only)

Sets the dynamic current limit to `BOOST_MAX_A` (default 32 A) immediately and keeps it there. The schedule and solar tracking are ignored. When the car unplugs, the mode automatically resets to **Default**.

> Note: because the Easee built-in schedule is disabled, the old "override schedule" button is not used here. Node-RED owns the current limit directly.

### Solar only (session-only)

Tracks solar export with no schedule floor  the current can go all the way down to 0 A. The car pauses when there is no solar and resumes automatically as soon as the house starts exporting again. When the car unplugs, the mode automatically resets to **Default**.

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
[Brain — all config & logic]
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

The **Brain** function node is the single place where all behaviour is defined. All configurable parameters are constants at the top of that node.

---

## Customising the parameters

Open the `Brain (all config here)` function node in Node-RED. At the top of the code you will find the configuration block:

```javascript
// ============================================================
// CONFIGURATION  edit all parameters here
// ============================================================

const DEVICE_ID = 'b6930861fd4fcd8ce162a7643da311f5';

// Default mode: schedule time slots.
// Each slot: { startHour, endHour, maxA }
// startHour inclusive, endHour exclusive (24-hour format).
// maxA = guaranteed minimum charging speed within this window.
const SCHEDULE_WEEKDAY = [
    { startHour: 1, endHour: 7, maxA: 16 }
];

const SCHEDULE_WEEKEND = [
    { startHour: 1,  endHour: 7,  maxA: 16 },
    { startHour: 11, endHour: 17, maxA: 16 }
];

// Absolute maximum that solar surplus may ramp up to in Default mode.
const SOLAR_MAX_A = 32;

// Maximum current for Boost mode.
const BOOST_MAX_A = 32;

// Solar only mode: current range.
// Min = 0 so the car pauses when no solar is available.
const SOLAR_ONLY_MIN_A = 0;
const SOLAR_ONLY_MAX_A = 32;

// Timezone for schedule evaluation.
// Node-RED in Docker typically runs UTC even if your house is CET/CEST.
// Set this to your IANA timezone name so the schedule follows wall-clock time.
const TIMEZONE = 'Europe/Brussels';

// Target net power (P1 + Marstek combined) in watts.
// Algorithm balances charger + batteries to achieve this net position.
// -100 = aim for 100W available margin; charger responds fastest, batteries absorb spikes.
const TARGET_NET_W = -100;

// Mains voltage used to convert watts to amps. 230 V for single-phase Europe.
const PHASE_VOLTAGE = 230;

// Gradient limit: maximum current change per 10-second cycle (amps).
// 1 A per cycle = smooth ramps. Batteries absorb cloud spikes/transients.
// Increase for faster response (more jitter), decrease for more stability.
const MAX_CHANGE_PER_CYCLE = 1;

// EV minimum charging current (amps).
// The car won't charge below this — values 1–5 A are the same as 0 in practice.
// OFF → ON:  only when surplus can sustain CAR_MIN_A.
// ON  → OFF: only when unclamped ideal target ≤ 0.
const CAR_MIN_A = 6;
```

**Adding a schedule slot** (e.g. weekday evenings 20:0022:00 at 10 A):
```javascript
const SCHEDULE_WEEKDAY = [
    { startHour: 1,  endHour: 7,  maxA: 16 },
    { startHour: 20, endHour: 22, maxA: 10 }
];
```

**Finding your device ID**: in Home Assistant go to *Settings  Devices & Services  Easee*, open your charger device, and copy the device ID from the URL  or use the value already in the file.

---

## Files

| File | Purpose |
|---|---|
| `configuration.yaml` | Home Assistant helpers (`input_select`, `input_number`), mode-change scripts, Shelly button event automations, and LED sync logic. Merge into your `configuration.yaml`. |
| `ui.yaml` | Lovelace card definition  three mode buttons plus charger sensors. Paste into a manual card. |
| `nodered-flow.json` | Node-RED flow with the 10-second evaluation cycle and Brain logic. Import via *Menu → Import* in Node-RED. |

---

## Home Assistant setup

1. Merge the contents of `configuration.yaml` into your Home Assistant `configuration.yaml`.
2. Restart Home Assistant.
3. Add the card in `ui.yaml` to a Lovelace dashboard (Edit dashboard  Add card  Manual  paste YAML).
4. **Disable the Easee schedule** in the Easee app (or set it to always allow charging  Node-RED manages limits directly via the dynamic current limit API).

## Node-RED setup

1. Make sure **node-red-contrib-home-assistant-websocket** is installed in Node-RED.
2. Import `nodered-flow.json` via *Menu  Import*.
3. Open the **Home Assistant** server node and point it at your HA instance if needed.
4. Deploy.
5. Open the `Brain (all config here)` function node, adjust the parameters for your situation.
6. Deploy again.

---

## How solar + battery control works

The algorithm runs every 10 seconds and reads **both P1 power (grid) and battery power (Marstek) instantly**. It treats them as a combined system:

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

Every 10 seconds, the Brain calculates the ideal current change needed to reach `TARGET_NET_W`:

```
idealDeltaA = (TARGET_NET_W − netPower) / PHASE_VOLTAGE
limitedDeltaA = (abs(idealDeltaA) < 1A) ? 0 : clamp(idealDeltaA, −MAX_CHANGE_PER_CYCLE, +MAX_CHANGE_PER_CYCLE)
```

| Condition | Action |
|---|---|
| netPower far above target (strong import) | Current decreases by 1 A (toward floor) |
| netPower far below target (strong export) | Current increases by 1 A (toward ceiling) |
| netPower close to target (within ±230W) | **Current stays unchanged** — 1A dead-band prevents jitter |
| Floor limit (Default) | `slot.maxA` inside schedule window; 0 A outside |
| Ceiling limit (Default & Solar only) | `SOLAR_MAX_A` and `SOLAR_ONLY_MAX_A` respectively |

The **1 A dead-band** prevents oscillation when the house is close to target. Brief cloud shadows or battery fluctuations won't trigger charger changes — only sustained deviation does.

### EV-minimum hysteresis (`CAR_MIN_A`)

Outside schedule windows (floor = 0 A), the car's physical minimum charging current creates a step function: it either draws ≥ `CAR_MIN_A` × 230 V or nothing. The algorithm handles this with hysteresis:

| Transition | Condition | Effect |
|---|---|---|
| OFF → ON | `idealDeltaA ≥ CAR_MIN_A` (surplus ≥ 1380 W) | Jump directly to `CAR_MIN_A` |
| ON → hold | target in dead zone [1, `CAR_MIN_A`−1] but ideal > 0 | Stay at `CAR_MIN_A` |
| ON → OFF | unclamped ideal target ≤ 0 | Drop to 0 A |
| Never | set 1–5 A | Dead zone eliminated |

This prevents the classic oscillation where the car repeatedly starts and stops because partial surplus (e.g. 300 W) can't sustain the car's minimum draw (1380 W).

The setpoint is stored in `input_number.easee_dynamic_current` so the UI shows the live target and the value survives a Node-RED restart.

**Default mode** floors at `slot.maxA` during schedule windows — the car always charges at least at the configured schedule rate, even when clouds appear or batteries are charging. Outside windows the floor is 0 A, so the car only charges if there is sufficient combined surplus.

**Solar only mode** also allows 0 A floor — the car halts completely when there is insufficient surplus, and restarts automatically when it returns.
