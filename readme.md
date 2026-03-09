# Easee Solar Charging

A Home Assistant + Node-RED solution for smart EV charging with an Easee charging station.

---

## Overview

The Easee built-in schedule is **disabled**. Node-RED takes full control of when and how fast the car charges. There are three modes selectable from the Home Assistant UI:

- **Default**  the permanent mode: follows a configurable weekday/weekend schedule, with automatic solar ramp-up when the house is exporting energy.
- **Boost**  session-only: charges at maximum speed immediately, regardless of schedule or solar. Resets to Default when the car unplugs.
- **Solar only**  session-only: follows solar export only, pausing completely when there is nothing to give. Resets to Default when the car unplugs.

Every 30 seconds Node-RED evaluates the current mode and adjusts the Easee dynamic current limit accordingly.

---

## Modes

### Default (permanent)

This is the normal, always-on behaviour. It combines scheduled charging with opportunistic solar charging:

1. **Outside every schedule window**  the dynamic current limit is set to 0 A. The car pauses charging.
2. **Inside a schedule window**  the current starts at the slot's configured `maxA`. From there:
   - If the house is **exporting** solar surplus (P1 < margin), the current ramps **up** by 1 A per tick (toward `SOLAR_MAX_A`).
   - If the house is **importing** AND the current is **above** the slot floor, it ramps **down** by 1 A per tick (back toward `slot.maxA`).
   - If the grid is balanced (within the dead-band), the current stays unchanged.

The schedule floor (`slot.maxA`) is never undercut during a window  the schedule is always guaranteed.

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
2. Resets `input_number.easee_dynamic_current` to 6 A (the EV minimum), ready for the next session.

This means pressing **Boost** or **Solar only** is always a "this session" decision  you never have to remember to reset it.

---

## Architecture

```
[Every 30s]
    
    
[Get mode]    input_select.easee_charging_mode
    
    
[Car connected?]    sensor.easee_charger_status
      (not connected  stop)
    
[Get P1 power (W)]    sensor.p1_meter_power_import_power
    
    
[Get current setpoint (A)]    input_number.easee_dynamic_current
    
    
[Brain  all config & logic]
    
    
[easee.set_circuit_dynamic_limit]
    
    
[input_number.set_value]  (keep HA helper in sync)


[Car connected?]    sensor.easee_charger_status (state-change trigger)
         |
     (connected)                  (disconnected)
         |                              |
[immediate evaluation]      [Reset mode  Default]
                            [Reset setpoint  6 A]
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

// Dead-band in watts before we adjust the current by 1 A.
// ~400 W  1.7 A on a single 230 V phase — prevents oscillation.
const SOLAR_MARGIN_W = 400;

// Hysteresis: consecutive ticks required before ramping UP or DOWN.
// Each tick is 30 s, so 10 ticks = 5 minutes of sustained condition before acting.
// Both directions use the same value so the system is symmetric and stable.
const HYSTERESIS_TICKS = 10;
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
| `configuration.yaml` | Home Assistant helpers (`input_select`, `input_number`) and scripts that set the mode. Merge into your `configuration.yaml`. |
| `ui.yaml` | Lovelace card definition  three mode buttons plus charger sensors. Paste into a manual card. |
| `nodered-flow.json` | Node-RED flow. Import via *Menu  Import* in Node-RED. |

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

## How solar control works

The P1 smart meter reports signed power in watts:

- **Positive** = the house is importing from the grid
- **Negative** = the house is exporting to the grid (solar surplus)

The algorithm runs every 30 seconds and adjusts by 1 A per tick:

| Condition | Action |
|---|---|
| P1 < −400 W (exporting) | Export counter +1; import counter resets to 0 |
| P1 > +400 W (importing) | Import counter +1; export counter resets to 0 |
| P1 within ±400 W (dead-band) | **Neither counter changes** — accumulated progress is preserved |
| Export counter reaches 10 (5 min) | Increase current by 1 A |
| Import counter reaches 10 (5 min) | Decrease current by 1 A (Default: floor at `slot.maxA`; Solar only: floor at 0 A) |

The 400 W dead-band prevents oscillation. A counter only resets when the **opposite** extreme is seen — a brief dip into the dead-band (cloud shadow, kettle switching on) does not wipe accumulated progress. This means the system ramps correctly on a normal variable-solar day rather than requiring an unbroken 5-minute block of clean export. The setpoint is stored in `input_number.easee_dynamic_current` so the UI shows the live target and the value survives a Node-RED restart.

**Default mode** floors at `slot.maxA` during schedule windows  the car always charges at least at the configured schedule rate, even when clouds appear.

**Solar only mode** allows 0 A  the car halts completely when there is nothing to give, and restarts automatically when solar returns.
