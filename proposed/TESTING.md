## EV Load Balancer Simulation & Validation

### Pre-checks
- Confirm the following entities report numeric values in *Developer Tools → States*: `sensor.ev_house_power`, `sensor.ev_capacity_limit`, `sensor.ev_available_capacity`, `sensor.wallbox_kristof_riebbels_laadvermogen` (or whatever live wallbox power/current sensor exists).  
- Set `input_number.ev_main_limit_w` to your contractual limit (e.g., 4000 W) and ensure `input_boolean.ev_force_max` is **off**.
- Optional: adjust `input_number.ev_margin_w` (default 500 W) and `input_number.ev_hysteresis_w` (default 300 W) to match desired buffer.

### Optional Test Script
Add this helper script (or adapt it) to `scripts.yaml` to emulate house-load swings by nudging the stored amperage helper. Replace the `service` calls if you already have better simulation inputs.

```yaml
script:
  ev_load_balancer_simulation:
    alias: "EV Load Balancer Simulation"
    mode: queued
    sequence:
      - service: system_log.write
        data:
          level: info
          message: "🔎 Starting EV load-balancer simulation"
      - service: input_number.set_value
        data:
          entity_id: input_number.ev_last_set_current_a
          value: 6
      - delay: "00:00:05"
      - service: input_number.set_value
        data:
          entity_id: input_number.ev_last_set_current_a
          value: 10
      - delay: "00:00:05"
      - service: input_number.set_value
        data:
          entity_id: input_number.ev_last_set_current_a
          value: 16
      - delay: "00:00:05"
      - service: system_log.write
        data:
          level: info
          message: "✅ Simulation run complete"
```

If you prefer live-data testing, simply energise appliances (oven, dryer, PV export, etc.) while watching the same key sensors.

### Manual Test Steps
1. **Baseline:** With low household load, run `automation.ev_load_balancer_adjust_charging_amps` from *Developer Tools → Services*. Expect the wallbox limit to ramp up by `input_number.ev_step_amp` until it reaches `input_number.wallbox_max_amp` or the available capacity.
2. **High Load Spike:** Increase household consumption (or simulate by briefly raising `sensor.ev_house_power` via the Developer Tools template). When total load exceeds `input_number.ev_main_limit_w - input_number.ev_margin_w`, the automation should step the charger current down within 30 s.
3. **Sustained Overload:** Push additional load so even 6 A cannot keep the limit. After ~20–30 s the automation should turn `switch.wallbox_kristof_riebbels_pauzeren_hervatten` off and set `input_number.ev_last_set_current_a` to `0` (with the wallbox max-current number holding the previous limit).
4. **Recovery:** Reduce house load below `limit - hysteresis`. The charger should turn the pause switch back on and ramp toward the available capacity.
5. **Manual Override:** Toggle `input_boolean.ev_force_max` to **on**. The charger should jump immediately to `input_number.wallbox_max_amp` and stay there. Toggle back to **off** and ensure the adaptive automation resumes (the adjust automation should be triggered automatically).
6. **Sensor Dropout:** Simulate an unavailable value (Developer Tools → States → set `sensor.eth_dongle_pro_power_delivered` to `unknown`). The binary sensor `binary_sensor.ev_load_balancer_ready` should go `off`, preventing further adjustments. Restore a numeric value and confirm it returns to `on`.

### Acceptance Checklist
- [ ] Cannot exceed contractual limit (`input_number.ev_main_limit_w`) for more than 30 s while automation is active.
- [ ] Recovers gracefully from sensor dropouts (binary sensor goes `off`, charging paused or held steady).
- [ ] Current adjustments respect hysteresis: no setpoint change more often than once per 30 s and only in `input_number.ev_step_amp` increments.
- [ ] Manual override (`input_boolean.ev_force_max`) instantly forces max amps and releases control when switched **off**.
- [ ] Emergency pause triggers when even minimum amps exceed the allowed limit, and the charger resumes automatically when safe.
