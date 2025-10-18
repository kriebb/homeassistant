# EV Load Balancing Audit

## Entity Inventory
| Entity | Platform | Unit | Availability | Defined at | Notes |
| --- | --- | --- | --- | --- | --- |
| `sensor.home_import_power_peak_31d` | `statistics` | W | n/a | `packages/00_energy_core.yaml:1` | Rolling 31-day import peak from `sensor.eth_dongle_pro_power_delivered`. |
| `sensor.fluvius_capacity_limit` | `template` | W | `input_number.ev_main_limit_w` & peak sensor numeric | `packages/00_energy_core.yaml:10` | Picks higher of contract limit or measured monthly peak. |
| `sensor.available_capacity` | `template` | W | Requires capacity limit + `sensor.eth_dongle_pro_power_*` | `packages/00_energy_core.yaml:26` | Dashboard helper: limit minus net grid import (delivered − returned). |
| `input_boolean.ev_force_max` | helper | n/a | n/a | `packages/11_wallbox_load_balancer.yaml:8` | Manual override switch for automation. |
| `input_number.ev_main_limit_w` | helper | W | n/a | `packages/11_wallbox_load_balancer.yaml:15` | Stores contractual limit (fallback when Fluvius sensor absent). |
| `input_number.wallbox_min_amp` | helper | A | n/a | `packages/10_wallbox_dynamic.yaml:2` | Minimum amperage accepted by charger. |
| `input_number.wallbox_max_amp` | helper | A | n/a | `packages/11_wallbox_load_balancer.yaml:25` | Caps commanded charging current. |
| `input_number.ev_margin_w` | helper | W | n/a | `packages/11_wallbox_load_balancer.yaml:34` | Safety buffer under main limit. |
| `input_number.ev_hysteresis_w` | helper | W | n/a | `packages/11_wallbox_load_balancer.yaml:43` | Prevents flapping while near limit. |
| `input_number.ev_step_amp` | helper | A | n/a | `packages/11_wallbox_load_balancer.yaml:52` | Step size for each adjustment. |
| `input_number.ev_last_set_current_a` | helper | A | n/a | `packages/11_wallbox_load_balancer.yaml:61` | Persists last commanded amperage (fallback when no live measurement). |
| `sensor.ev_house_power` | `template` | W | Needs `sensor.eth_dongle_pro_power_*` | `packages/11_wallbox_load_balancer.yaml:73` | Net grid import (delivered − returned, floored at 0). |
| `sensor.ev_capacity_limit` | `template` | W | Fluvius limit or helper | `packages/11_wallbox_load_balancer.yaml:90` | Mirrors live limit, falls back to helper. |
| `sensor.ev_estimated_charger_power` | `template` | W | Wallbox sensor or helper | `packages/11_wallbox_load_balancer.yaml:114` | Converts `sensor.wallbox_kristof_riebbels_laadvermogen`/amp sensors to W, falls back to last commanded amps. |
| `sensor.ev_non_ev_load` | `template` | W | `sensor.ev_house_power` & charger power | `packages/11_wallbox_load_balancer.yaml:159` | House load excluding EV draw. |
| `sensor.ev_available_capacity` | `template` | W | Capacity limit & non-EV load | `packages/11_wallbox_load_balancer.yaml:173` | Headroom after margin guard. |
| `binary_sensor.ev_load_balancer_ready` | `template` | on/off | EV sensors numeric | `packages/11_wallbox_load_balancer.yaml:188` | Blocks automation while inputs are invalid. |
| `automation.ev_load_balancer_adjust_charging_amps` | automation | n/a | Requires sensors & helpers | `packages/11_wallbox_load_balancer.yaml:220` | Core adaptive controller; drives `number.wallbox_kristof_riebbels_maximale_laadstroom` and pause switch with hysteresis. |
| `automation.ev_load_balancer_force_max_override` | automation | n/a | n/a | `packages/11_wallbox_load_balancer.yaml:378` | Handles manual override toggle. |

**External dependencies**: `sensor.eth_dongle_pro_power_delivered`, `sensor.eth_dongle_pro_power_returned`, `sensor.eth_dongle_pro_voltage_l1/2/3`, `sensor.wallbox_kristof_riebbels_laadvermogen`, `sensor.wallbox_kristof_riebbels_max_laadstroom`, `sensor.wallbox_kristof_riebbels_max_beschikbaar_vermogen`, `number.wallbox_kristof_riebbels_maximale_laadstroom`, and `switch.wallbox_kristof_riebbels_pauzeren_hervatten`. Confirm these IDs stay stable after reloading HA; adjust the package if variants exist.

## Existing Load-Balancing Logic
- **Legacy automation (`wallbox_dynamic_waterker`)** at `packages/10_wallbox_dynamic.yaml` (now commented) tried to set current directly from available capacity but:
  - Called `wallbox.set_max_charging_current` on `sensor.wallbox_kristof_riebbels_max_laadstroom`, which cannot accept service calls (should target the wallbox device domain).
  - Hardcoded `max_amp: 16`, ignoring higher contract capacity.
  - Forced `min_amp` even when headroom was negative, never pausing the charger.
  - Lacked hysteresis or manual override, so it would flap with noisy measurements.
- No other active EV-specific automations were found; predictive scheduler targets whitegoods only (`packages/30_predictive_scheduler.yaml`).

## Gaps Identified
- Contract limit used the boiler sensor (`sensor.waterker_power`) and defaulted to 4 kW; no monthly peak tracking on the real mains feed.
- No availability guards; if the mains readings went `unknown`, the old automation still acted.
- Wallbox control entities are not formalised in configuration; automation assumed wrong IDs.
- No load-aware pause: the charger kept pulling `min_amp` even when the main fuse limit was exceeded.
- Manual override absent, preventing easy temporary full-power charging.

## Remediation Implemented
1. **Capacity normalisation**  
   - Added a rolling 31-day peak sensor (`sensor.home_import_power_peak_31d`) on the mains import feed and refactored `sensor.fluvius_capacity_limit` to respect `input_number.ev_main_limit_w` (`packages/00_energy_core.yaml:1-36`).
   - Introduced availability templates for capacity and dashboard helpers, all based on `sensor.eth_dongle_pro_power_delivered/returned`.

2. **New EV load-balancer package** (`packages/11_wallbox_load_balancer.yaml`)  
   - Helpers for contract limit, safety margin, hysteresis, step size, manual override, and last commanded current.
   - Template sensors that:
     - Sanitise house load via the mains P1 import/export readings and compute non-EV load.
     - Estimate EV draw using `sensor.wallbox_kristof_riebbels_laadvermogen` (or amp readings) with a fallback to stored setpoints.
     - Derive safe headroom after applying a configurable safety margin.
   - `binary_sensor.ev_load_balancer_ready` halts adjustments when inputs are missing.
   - Main automation ramps charger current up/down in `input_number.ev_step_amp` increments every ≥30 s, observing hysteresis and pausing the charger (`switch.wallbox_kristof_riebbels_pauzeren_hervatten`) when even minimum current would violate the limit.
   - Override automation that ties `input_boolean.ev_force_max` to an immediate max-current command and re-enables adaptive control when released.

3. **Change management**  
   - Legacy automation preserved but commented with a migration note in `packages/10_wallbox_dynamic.yaml:2-41`.
   - All new YAML delivered under `proposed/` for easy review alongside `changes.diff`.

## Operational Notes & Recommendations
- Validate actual wallbox control entities. If your installation exposes different IDs (e.g., `switch.wallbox_kristof_riebbels_start_stop`), adjust lines `packages/11_wallbox_load_balancer.yaml:331`, `354`, `395`, and `398`.
- Confirm the average phase voltage logic meets your installation. If you prefer a fixed value, edit the `voltage_avg` calculation at `packages/11_wallbox_load_balancer.yaml:243`.
- Old automations (`automation.wallbox_dynamische_laadregeling_waterker`, `automation.wallbox_dynamische_laadregeling_fluvius`, etc.) are still visible as restored entities. After reloading automations, ensure only the new `automation.ev_load_balancer_*` remain active or explicitly keep the legacy ones disabled.
- Consider adding per-phase or PV surplus logic once corresponding sensors (e.g., `sensor.pv_power`) are confirmed; hooks can reuse the existing template sensors.
- Run `hass --script check_config` after applying the diff and review `binary_sensor.ev_load_balancer_ready` before re-enabling charging.

## Pass/Fail Checklist
- [ ] Cannot exceed configured limit (`input_number.ev_main_limit_w`) for more than ~30 s.
- [ ] Recovers from sensor dropouts: `binary_sensor.ev_load_balancer_ready` turns `off`, charger pauses, and resumes when data returns.
- [ ] Does not adjust faster than every 30 s and only in `input_number.ev_step_amp` increments.
- [ ] Manual override (`input_boolean.ev_force_max`) forces max current immediately and releases control when switched off.
