# Easee Smart Charging Migration Plan

**Status:** Draft
**Created:** 2026-11-13
**Last updated:** 2026-11-13
**Scope:** Replace the Node-RED dependency first, then add provider-independent smart charging in Home Assistant.

## Executive summary

The migration is deliberately staged. The first implementation milestone ports the current, tested charging behavior from Node-RED to Home Assistant YAML without adding the smart scheduler. This makes the existing system usable in another home before tariff and forecast complexity is introduced.

After parity is established, the design adds normalized tariff and solar-forecast inputs, manually configured car profiles, per-session helpers, and a rolling 15-minute planner. The planner will not set a fixed current for an entire slot: the existing 10-second P1 + Marstek feedback controller will continue adjusting current within the planner's limits. Use direct Open-Meteo API calls for solar-radiation forecasts rather than relying on the Home Assistant weather integration's hourly data. Use a simple aggregate PV model and no historical load/production data. Home Assistant YAML is the preferred implementation; a small local Python planner is permitted only if the optimization cannot be expressed and tested reliably in YAML. Node-RED is not a target dependency.

The user-facing dashboard is for charging controls and session setup, not cost or forecast reporting. If Smart-mode tariff or forecast inputs are unavailable, notify the user and fall back to the existing scheduled behavior.

## Current repository

- [nodered-flow.json](./nodered-flow.json) contains the 10-second charging loop, P1/Marstek calculation, mode logic, helper initialization, Easee limit calls, and disconnect reset.
- [configuration.yaml](./configuration.yaml) defines Home Assistant helpers, template sensors, and mode scripts.
- [automations.yaml](./automations.yaml) handles Shelly controls/LEDs and the mobile mode-selection notification.
- [ui.yaml](./ui.yaml) contains the Lovelace card.
- [readme.md](./readme.md) documents setup and known configuration caveats.

The current Home Assistant configuration has both a `script: !include scripts.yaml` entry and an inline `script:` section. Consolidate these into one valid configuration structure during the parity migration; the README already warns about this.

## Confirmed decisions

### 2026-11-13 - Port current behavior before adding smart scheduling

**Status:** Accepted

**Context:** The system should be easier to migrate to another home and should no longer depend on Node-RED.

**Options considered:**
1. Add smart charging to Node-RED, then migrate later.
2. Rewrite the current behavior in Home Assistant first, verify parity, then add smart scheduling.

**Decision:** Option 2.

**Rationale:** It separates migration risk from algorithm risk and creates a portable Home Assistant baseline before adding new dependencies or behavior.

**Consequences:** No smart-planner work begins until the current modes, solar/battery control, resets, notifications, and settings initialization work without Node-RED.

### 2026-11-13 - Replace Node-RED directly, without a shadow controller

**Status:** Accepted

**Context:** The user prefers a direct migration of the existing controller rather than running a second HA controller in shadow mode.

**Decision:** Stop/disable Node-RED's Easee writes before enabling the Home Assistant control automation. The HA automation writes directly to the charger; do not add a shadow-current helper or live-control toggle.

**Rationale:** The user will perform a direct replacement and wants to avoid carrying an extra shadow path. A strict stop-before-start sequence prevents competing writers.

**Consequences:** Validate the HA controller after cutover using Home Assistant traces, entity history, and charger behavior. Keep the Node-RED flow available for rollback, but disable the HA writer before re-enabling Node-RED.

### 2026-11-13 - Use a 15-minute planner with live current control

**Status:** Accepted

**Context:** Tariffs and solar forecasts guide when charging is valuable, but live P1 and Marstek readings must continue to control the actual charging rate.

**Decision:** The planner allocates energy or a charging envelope to 15-minute intervals. The live controller continues to adjust current approximately every 10 seconds using P1 + Marstek data. Actual delivered energy is fed back into subsequent plans.

**Consequences:** A planned slot is not a fixed-amperage command. Forecasts influence scheduling; live readings remain authoritative for immediate current adjustments.

### 2026-11-13 - Define the modes and schedule behavior

**Status:** Accepted

- **Default / Smart:** Target SoC by the departure date/time is a commitment while required planning inputs are available. Use the current tariff prices and solar forecast; grid energy is allowed if required and feasible. If tariff or solar-forecast data is unavailable, notify the user and fall back to the existing scheduled behavior. The target is not guaranteed during this fallback.
- **Solar only:** Best-effort progress toward the target without grid import. Preserve the existing tested P1 + Marstek balancing behavior that keeps Marstek state of charge stable during charging. Do not promise the target if available solar is insufficient.
- **Boost:** Charge immediately at the configured maximum, without tariff/solar scheduling.
- **No active target:** Keep the existing weekly schedule as a hard allowed charging window.
- **Active target:** The planner may schedule outside the weekly window.

**Open naming detail:** Decide whether to retain the internal mode value `Default` for compatibility while displaying “Smart” in the UI, or rename the internal option as part of the migration.

### 2026-11-13 - Keep tariff providers separate from the planner

**Status:** Accepted

**Decision:** Provider-specific adapters convert tariff data into a common per-interval representation. The planner must not contain ENGIE-specific entity IDs or rate-period assumptions. For the current home, use the existing ENGIE price sensors and put the three-rate time-period mapping in clearly grouped YAML constants. Do not add a user-facing cost/forecast overview or additional bill-cost model in this scope.

**Consequences:** Implement the current ENGIE three-rate contract as the first adapter, while keeping the interface suitable for dynamic EPEX tariffs and other providers/homes. A different home's provider needs its own adapter; provider independence does not mean every provider is automatically supported.

### 2026-11-13 - Prefer Home Assistant YAML; isolate Python if necessary

**Status:** Accepted

**Decision:** Use Home Assistant automations, scripts, helpers, and template entities wherever maintainable. A small local Python component may be used for the planner only if YAML/Jinja would make the algorithm brittle or untestable. Document every non-UI setup step in the README.

### 2026-11-13 - Use direct Open-Meteo API data and a simple PV model

**Status:** Accepted

**Context:** The Home Assistant Open-Meteo integration exposes hourly forecasts, while the planner is designed around 15-minute intervals.

**Decision:** Attempt direct Open-Meteo API calls for 15-minute solar-radiation data. Model the multiple PV arrays as one approximate aggregate array using the configured average orientation and tilt. Do not use historical production or household-load data.

**Known values:** 8 kW inverter AC limit; average orientation southwest; average tilt 20 degrees.

**Open input:** Total installed PV panel capacity in kWp is still needed. The inverter's 8 kW rating is an AC output cap and does not specify the panel array's DC capacity.

**Consequences:** Forecast accuracy is intentionally approximate. Live P1 + Marstek control remains authoritative, and unavailable tariff/forecast data in Smart mode causes a notification and scheduled fallback.

### 2026-11-13 - Use manual car profiles and a dashboard session form

**Status:** Accepted

**Decision:** Do not add vehicle integrations. Configure car profiles in Home Assistant YAML, enter start SoC and target/departure values manually in the dashboard, and use the Start action to switch from scheduled behavior to Smart mode.

**Current profiles supplied by the user:**

| Car | User-provided battery capacity | Charging capability notes |
|---|---:|---|
| VW ID.3 | 58 kWh | 1- or 2-phase capability on the home's 3x220 V service; 11 kW onboard charger rating is for 3-phase charging. |
| Peugeot e-Expert | 68 kWh | 1-phase on the home's 3x220 V service; 11 kW onboard charger rating is for 3-phase charging. |

**Open inputs:** Confirm whether the stated capacities are usable capacities and validate each car's effective maximum AC charging power on the 3x220 V installation. Do not assume either car can draw 11 kW on this service.

**Session flow:** On plug-in, immediately use the existing scheduled behavior. Notify the user to configure the car, current SoC, target SoC, and target/departure time. Pressing Start arms the session and switches to Smart. If Start is not pressed, scheduled behavior continues.

## Target behavior and architecture

### 1. Site configuration and stable entities

- Put site-specific device IDs, source entity IDs, and fixed control constants in a clearly labeled configuration section at the start of the relevant YAML configuration.
- Centralize the Easee device ID, charger status/power/energy entities, P1 entity, Marstek entity, Shelly identifiers/entities, and any tariff-provider mappings.
- Expose stable internal Easee charging entities or helpers so automations and the dashboard do not repeat home-specific source IDs.
- Keep semi-static values as Home Assistant entities defined in YAML. This includes car capacities/limits, PV system values, and user-adjustable charging settings. Keep fixed algorithm constants and the ENGIE rate-period mapping in clearly grouped constants at the start of their relevant YAML sections.
- Keep per-session values in helpers: selected car, start SoC, target SoC, departure date/time, and an armed/active session state.
- Keep fixed algorithm constants distinct from editable entity values. Do not silently use placeholder/default data when required inputs are unavailable.
- Home Assistant does not provide one universal variable scope across all YAML files and dashboard YAML. Choose and document a central mapping/wrapper pattern that works for triggers, actions, templates, and Lovelace; do not assume YAML anchors can be shared across files.

### 2. Normalized input adapters

**Tariff adapter contract**

For each planning interval, expose at least:

- Interval start/end in Home Assistant local time.
- Import price in EUR/kWh.
- Export/injection price in EUR/kWh when available.
- Data source and freshness/availability.

The first adapter uses the existing ENGIE peak/off-peak/super-off-peak import and injection price sensors. Map them to future intervals using the contract's weekday/weekend and time-period rules, defined as constants at the start of the adapter's YAML section. The rate sensors alone do not define those periods. Keep provider entity IDs in the central site mapping. Use the resulting prices to rank charging opportunities; do not add an all-in bill-cost model or user-facing cost report in this scope.

**Solar-forecast adapter**

- Call the Open-Meteo API directly; the standard Home Assistant integration's hourly data is not sufficient for the intended 15-minute planner. Verify the API response provides suitable 15-minute radiation data for the configured location and forecast horizon. Do not silently treat hourly/interpolated data as true 15-minute data.
- Approximate the multiple PV arrays as one aggregate array with southwest orientation and 20-degree tilt, capped by the 8 kW AC inverter. Use the installed DC array capacity when supplied; it is still unknown. No historical household-load or PV-production model is in scope.
- Provide forecast timestamps, values, and freshness/quality information.
- Continue to use live P1 + Marstek data for immediate control and correction. The forecast is an estimate, not a replacement for live balancing or a guarantee of available surplus.

### 3. Planner and live controller

**Planner**

- Run at 15-minute boundaries and after relevant session, tariff, or forecast changes.
- Estimate energy required from the selected car's usable battery capacity and the start/target SoC difference, with a configurable charging-loss and reserve margin.
- Respect the selected vehicle's AC limit, charger/circuit limit, EV minimum current, and any active hard schedule constraint.
- In Smart mode, find a feasible schedule that meets the target by departure while minimizing the supplied ENGIE energy price and favoring forecast solar. Where useful, compare the injection price as the value of exporting solar instead of charging; do not calculate or display total bill cost.
- In Solar-only mode, use forecast solar to identify likely opportunities but never plan grid import. Preserve the real-time no-grid control even if forecast data is missing.
- Detect invalid or infeasible session inputs and notify only when an action is required; do not add a plan, cost, or solar-use overview.
- Store planner output only as needed for shadow validation and live control. Do not expose a user-facing planner dashboard in this scope.

**Live controller**

- Preserve the tested P1 + Marstek calculation and battery-neutral behavior.
- Preserve the existing EV-minimum hysteresis, current-step/dead-band behavior, configured limits, and helper synchronization unless parity testing identifies an intentional change.
- Run at the existing control cadence initially (about 10 seconds). The planner's slot allocation constrains the controller; it does not replace the live feedback loop.
- Track actual delivered charging energy and remaining target energy, then re-plan rather than assuming the planned current was achieved.
- Make charger writes from exactly one active controller at a time.

### 4. Session UI and mobile notification

- On plug-in, immediately use the existing scheduled behavior and send a notification prompting the user to configure the session.
- Configure opens a Home Assistant dashboard view containing the car selector, start SoC, target SoC, departure date/time, and Start action. Start arms the session and switches it to Smart mode; if not pressed, scheduled behavior continues.
- Keep existing mode actions, including Boost and Solar-only, available through the normal controls.
- The notification itself cannot provide a general dropdown-and-slider form. It can provide buttons and limited text input; use the dashboard for validated car and numeric settings.
- If session values are absent or invalid, do not start a Smart plan silently; prompt the user to configure them.
- Preserve disconnect cleanup: stop the active session, reset the active mode/setpoint as specified by parity behavior, and clear the Easee dynamic limit.

**Missing-data behavior:** In Smart mode, unavailable/stale tariff or solar-forecast data triggers a notification and returns control to the existing scheduled behavior. Solar-only must retain its no-grid guarantee; whether it should also switch to the schedule when its forecast is missing remains an open point because a scheduled fallback could allow grid import.

## Phased implementation plan

### Step 1 - Rewrite the existing logic in Home Assistant and remove the Node-RED dependency

This is the first implementation step. Do not add tariff optimization or solar forecasting before it passes its exit criteria.

1. Record the current constants, helper defaults, entity/device dependencies, mode transitions, and known P1/Marstek test scenarios.
2. Create the central site-ID/configuration map and stable internal entities as part of the port. Keep fixed constants clearly grouped at the top of the relevant YAML section; keep semi-static settings as HA helpers.
3. Port the current behavior to Home Assistant YAML:
   - 10-second evaluation and connected-car gating.
   - P1 + Marstek net-power calculation and the battery-enable setting.
   - Existing schedule behavior, helper initialization-once behavior, and amperage limits.
   - Default, Boost, and Solar-only behavior, including the current EV minimum, hysteresis, dead-band, and per-cycle current change.
   - Easee dynamic-limit writes and setpoint helper synchronization.
   - Disconnect reset of mode, setpoint, and Easee limit.
   - Shelly button/LED automations and mobile mode-selection/test notification.
4. Consolidate the duplicate top-level `script:` configuration and update the README's migration/setup instructions.
5. Before enabling the HA controller, stop/disable the Node-RED flow's Easee writes. This is a direct replacement; no shadow controller or second live writer is used.
6. Validate the HA controller after cutover with Home Assistant traces, entity history, and charger behavior. Keep a documented rollback sequence: disable HA writes before re-enabling Node-RED.
7. Remove Node-RED as a required runtime dependency only after all exit criteria pass. Keep the existing flow available in version history for rollback during the migration.

**Exit criteria**

- Home Assistant operates without Node-RED deployed or required.
- No duplicate/competing Easee limit writers are active.
- Current behavior matches the existing implementation for representative mode, schedule, P1, Marstek, and disconnect scenarios.
- Helper defaults are initialized once and subsequent user changes survive restarts.
- Invalid/unavailable readings cause an explicit warning/notification and no unsafe success-shaped command.
- Site-specific IDs are changed in one documented mapping area, not scattered across automations and dashboard logic.
- The README contains clear Home Assistant setup and migration steps.

### Step 2 - Add per-car/session helpers and normalize provider inputs

1. Define the car selector and profile entities for usable capacity, maximum AC charging power/current, and phase/connection limits. Seed the profiles with the user-provided ID.3 (58 kWh, 1/2 phase) and e-Expert (68 kWh, 1 phase) values, pending confirmation of usable capacity and effective power on 3x220 V.
2. Define per-session start SoC, target SoC, departure date/time, and session armed/active state.
3. Define HA entities for the semi-static PV/charger values, including the 8 kW inverter limit, southwest orientation, 20-degree tilt, installed panel capacity when confirmed, and planner safety margins.
4. Implement the ENGIE three-rate adapter using the existing import/injection price sensors and the tariff-period mapping supplied as YAML constants.
5. Define the provider-neutral interval contract before adding any other tariff provider.
6. Implement a direct Open-Meteo API adapter and verify actual 15-minute radiation-data availability for the home's location and forecast horizon. Do not use historical load or production data.

**Exit criteria:** Every planner input has a documented unit, source, timestamp/availability rule, and failure behavior. Provider-specific IDs remain inside the adapter/config mapping.

### Step 3 - Validate the rolling planner with its output isolated from the live controller

This planner-only validation phase occurs after the HA controller replaces Node-RED. It must not introduce a second Easee writer or change the proven 10-second P1 + Marstek control until planner behavior is accepted.

1. Build the 15-minute interval horizon from now through the departure time, using verified forecast resolution and the ENGIE period constants.
2. Calculate remaining battery energy from the selected profile, start SoC, target SoC, measured delivered energy, and charge-efficiency margin.
3. Create per-slot charge-energy budgets/limits using ENGIE prices, the simplified aggregate PV forecast, EV/charger power limits, and schedule constraints. Do not estimate household demand from history.
4. Keep planner output separate from the live Easee command path during validation.
5. Retain only the internal planner state needed for validation; do not add user-facing cost, plan, or solar-use overviews.
6. Recalculate after each slot and when target/session state, tariff data, or forecasts materially change.
7. Prefer a simple, explainable optimizer first. Prototype a cost-ranked feasible slot allocator in YAML; use an isolated Python planner only if YAML/Jinja cannot express the constraints safely and testably.

**Exit criteria:** Simulations show target feasibility, tariff ranking, solar preference, and deadline reserve behave as intended. Missing/stale data triggers the documented notification and scheduled fallback in Smart mode; do not silently substitute assumptions.

### Step 4 - Integrate the planner with the live controller and modes

1. Apply the planner's current-slot budget/limit to the existing live controller.
2. Keep P1 + Marstek adjustments active within the slot; update remaining energy from actual charger data.
3. Implement Smart target commitment with grid fallback when required and feasible. On tariff or forecast loss, notify and return to existing scheduled behavior.
4. Implement Solar-only as best-effort, no-grid charging while preserving the tested Marstek-neutral behavior.
5. Keep Boost immediate and independent of tariff/forecast planning.
6. Preserve weekly schedule enforcement when no target is set.
7. Add explicit handling for infeasible targets, forecast loss, tariff loss, invalid car profile, unavailable charger, and disconnection. Never let a scheduled fallback in Solar-only violate its no-grid guarantee.

**Exit criteria:** Live tests confirm that forecasts never override the real-time P1/Marstek safety behavior, Smart meets feasible targets, Solar-only does not import from the grid, and Boost remains immediate.

### Step 5 - Cut over, prove portability, and finish documentation

1. Run the Home Assistant controller against real conditions with monitoring and a documented stop/rollback procedure.
2. Confirm only one charger writer is active and that mode/session cleanup works after restart and unplug.
3. Re-run setup using a second-home configuration map to verify that changing site IDs/settings does not require editing planner logic.
4. Remove obsolete Node-RED setup instructions and document any remaining UI-only tasks (for example the weekly Schedule helper).
5. Document ENGIE rate-period constants and sensor mapping, direct Open-Meteo API setup/resolution assumptions, car/PV helper setup, dashboard use, troubleshooting, and any optional Python setup.
6. Remove `nodered-flow.json` from the active setup once the migration is accepted; retain recoverability through version history during rollout.

## Validation plan

### Current-behavior parity

- Schedule on/off, and missing schedule state.
- P1 import/export signs; Marstek charging/discharging signs; battery enabled/disabled.
- EV minimum current start/stop thresholds and behavior around the dead-band.
- Current floor/ceiling and per-cycle ramp limits.
- Default, Boost, and Solar-only transitions.
- First-run helper initialization versus restored user settings.
- Connected/disconnected state, reset of mode/setpoint/Easee limit, and startup behavior.
- Shelly button events, LED synchronization, mobile mode choices, test notification, timeout, and first-response behavior.

### Planner and integration behavior

- Target already met, target feasible, target infeasible, and departure in the next day/DST transition.
- Lowest-cost intervals selected when no solar surplus is forecast.
- Solar surplus changes the per-slot plan without fixing the actual current for the whole slot.
- Smart uses grid fallback when needed; Solar-only does not.
- Battery-neutral P1 + Marstek control remains intact.
- ENGIE time bands map correctly to local-time intervals; missing/stale provider data triggers notification and scheduled fallback in Smart mode.
- Direct Open-Meteo response has verified 15-minute radiation data; unavailable, stale, or lower-resolution data triggers notification and scheduled fallback in Smart mode.
- Solar-only remains no-grid if forecast data is unavailable; verify the selected safe behavior against the open fallback question below.
- Vehicle/charger/site limits, EV minimum current, and safety reserve are enforced.
- Notification links open the setup view and invalid session inputs cannot start a plan.

Use Home Assistant configuration validation and observe the direct controller after cutover. If Python is used, add focused tests for the planner's energy, price, deadline, and failure cases. The repository currently has no dedicated automated test suite.

## Information required before smart-planner implementation

| Input | Supplied/current value | Still required |
|---|---|---|
| Home electrical service | 3x220 V | Confirm per-car maximum AC power/current on this service and the effective charger/circuit limit. Do not use the 11 kW three-phase onboard rating as the home charging rate. |
| VW ID.3 | 58 kWh; 1- or 2-phase capability; 11 kW onboard rating on 3 phases | Confirm whether 58 kWh is usable capacity and validate effective AC power/current on 3x220 V. |
| Peugeot e-Expert | 68 kWh; 1-phase capability; 11 kW onboard rating on 3 phases | Confirm whether 68 kWh is usable capacity and validate effective AC power/current on 3x220 V. |
| Tariffs | Use the existing ENGIE exposed import/injection price sensors; no other bill-cost model | Supply exact weekday/weekend and time boundaries for peak, off-peak, and super-off-peak periods. Keep them as YAML constants. |
| PV | 8 kW AC inverter; multiple arrays approximated as one southwest-facing array at 20-degree tilt; no history | Supply total installed PV panel DC capacity in kWp. Optionally provide a conservative derating/shading factor if known. |
| Solar forecast | Direct Open-Meteo API calls; planner interval 15 minutes | Verify the API's actual 15-minute radiation variables/resolution for the home's location and required horizon. |
| Vehicle session data | Manual entry; no vehicle integration | None for the planned flow: select car, enter start SoC/target SoC/departure time, then press Start. |
| Existing control | Preserve P1/Marstek behavior and battery-neutral control | Confirm units/signs/update rates as part of Step 1 parity testing. |
| Fallback scope | Smart-mode tariff/forecast loss notifies and returns to schedule | Confirm whether Solar-only should also change modes on forecast loss. Safe default: notify and keep the live no-grid controller rather than risk a scheduled grid import. |
| Runtime | Home Assistant YAML preferred; local Python only if YAML is not maintainable/testable | Confirm installation restrictions only if a Python planner becomes necessary. |

## Risks and mitigations

| Risk | Mitigation |
|---|---|
| Two controllers write to Easee at the same time | Stop Node-RED writes before enabling the HA controller. For rollback, disable HA writes before re-enabling Node-RED; never run both writers concurrently. |
| PV forecast is approximate or unavailable | Use a single aggregate array model, verify forecast resolution, re-plan from live P1/Marstek and delivered energy, and notify/fall back to schedule in Smart mode. Do not use historical load/production data. |
| Tariff data is provider-specific or incomplete | Normalize through adapters; validate period constants and reject missing/stale interval data explicitly. |
| YAML/Jinja optimizer becomes too complex | Keep the planner isolated behind a stable HA interface; allow a small tested local Python component as the documented fallback. |
| Home-specific IDs leak into control logic | Central site map and stable internal entities; validate the second-home onboarding path. |
| Target SoC estimate is inaccurate | Use usable capacity, charge-efficiency margin, actual delivered-energy feedback, and report estimated versus achieved SoC. |

## Current design state

The Home Assistant direct controller, centralized site mapping, and migration documentation are in place. YAML syntax/duplicate-key checks and a local nine-case current-calculation comparison against the Node-RED function pass; these do not replace Home Assistant configuration validation or live charger testing. Step 1 remains in progress until the direct cutover, representative behavior, and rollback sequence are verified on the Home Assistant instance. Smart scheduling, tariff adapters, direct Open-Meteo forecasting, and the session UI remain planned work. The supplied vehicle/PV characteristics and the no-history/no-overview constraints are recorded above; PV DC capacity, tariff period boundaries, usable battery capacities, and effective per-car AC limits remain open.

## Revisions

- 2026-11-13 - Initial plan created from the agreed migration order, mode semantics, provider separation, slot/live-control split, and YAML-first preference.
- 2026-11-13 - Updated from user feedback: direct Open-Meteo API attempt, supplied vehicle and PV details, ENGIE rate-period constants, manual dashboard session flow, no historical estimates or user-facing overviews, and Smart-mode scheduled fallback.
- 2026-11-13 - Removed the Step 1 HA shadow-controller approach per user preference; require stopping Node-RED before direct HA cutover, while keeping planner-output validation separate from live Easee writes.
