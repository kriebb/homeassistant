# Wallbox Load Balancing Investigation

This document outlines the investigation into the wallbox load balancing issue and provides a step-by-step guide to fix it.

## 1. Problem Analysis

The main goal is to dynamically adjust the wallbox's charging current based on the household's power consumption, ensuring it never exceeds the monthly peak capacity.

The current implementation has three main issues:

1.  **Incorrect Monthly Peak Calculation:** The `sensor.fluvius_capacity_limit` is not correctly calculating the monthly peak. It's using the live power consumption instead of the peak consumption for the current month.
2.  **Incorrect Wallbox Entity:** The automation is trying to set the charging current on a sensor entity (`sensor.wallbox_kristof_riebbels_max_laadstroom`) instead of the correct wallbox control entity.
3.  **Fixed Maximum Current:** The maximum charging current is hardcoded to 16A, while the user has a 32A capacity.

## 2. Proposed Solution

To address these issues, I will make the following changes:

1.  **Correctly Calculate Monthly Peak:** I will create a new utility meter sensor to track the monthly peak of `sensor.waterker_power`.
2.  **Use the Correct Wallbox Entity:** I will identify the correct entity to control the wallbox's charging current. Based on the `wallbox` integration's documentation, the service `wallbox.set_max_charging_current` should be called on the wallbox entity itself, which is likely named `switch.wallbox_laden` or similar.
3.  **Implement Dynamic Maximum Current:** I will modify the automation to use a dynamic maximum current, up to 32A, based on the available capacity.

## 3. Implementation Steps

### Step 1: Create a Utility Meter for Monthly Peak

First, I will add a utility meter to track the monthly peak power consumption. This will give us the correct value for the `fluvius_capacity_limit`.

I will add the following to a new file `packages/20_utility_meters.yaml`:

```yaml
utility_meter:
  waterker_monthly_peak:
    source: sensor.waterker_power
    cycle: monthly
    tariffs:
      - peak
```

### Step 2: Update the Capacity Limit Sensor

Next, I will update the `sensor.fluvius_capacity_limit` to use the new utility meter.

I will modify `packages/00_energy_core.yaml` as follows:

```yaml
sensor:
  - platform: template
    sensors:
      fluvius_capacity_limit:
        friendly_name: "Fluvius Capaciteitslimiet (W)"
        unit_of_measurement: "W"
        value_template: >
          {% set monthly_peak = states('sensor.waterker_monthly_peak') | float(4000) %}
          {% set contract_limit = 4000 %}
          {% if monthly_peak > contract_limit %}
            {{ monthly_peak }}
          {% else %}
            {{ contract_limit }}
          {% endif %}
```

### Step 3: Update the Load Balancing Automation

Finally, I will update the `10_wallbox_dynamic.yaml` automation to use the correct entity and a dynamic maximum current.

```yaml
input_number:
  wallbox_min_amp:
    name: Wallbox minimum stroom
    min: 6
    max: 32
    step: 1
    unit_of_measurement: "A"
    initial: 6

automation:
  - alias: "Wallbox - Dynamische laadregeling (Waterker)"
    id: wallbox_dynamic_waterker
    trigger:
      - platform: state
        entity_id:
          - sensor.waterker_power
          - sensor.fluvius_capacity_limit
        for: "00:00:10"
    variables:
      limit: "{{ states('sensor.fluvius_capacity_limit') | float(4000) }}"
      house_power: "{{ states('sensor.waterker_power') | float(0) }}"
      voltage: 230
      min_amp: "{{ states('input_number.wallbox_min_amp') | int(6) }}"
      max_amp: 32
    action:
      - variables:
          available_power: "{{ [limit - house_power, 0] | max }}"
          target_amp: >
            {% set amps = (available_power / voltage) | int %}
            {% if amps < min_amp %} {{ min_amp }}
            {% elif amps > max_amp %} {{ max_amp }}
            {% else %} {{ amps }} {% endif %}
      - service: wallbox.set_max_charging_current
        data:
          charging_current: "{{ target_amp }}"
          entity_id: switch.wallbox_laden
      - if:
          - condition: template
            value_template: "{{ available_power < 0 }}"
        then:
          - service: system_log.write
            data:
              level: warning
              message: >
                "⚠️ Capaciteit overschreden: {{ house_power|int }} W / limiet {{ limit|int }} W.
                 Wallbox verlaagd naar {{ target_amp }} A."
    mode: restart
```

I have now created the `gemini.md` file with the analysis and the proposed solution. I will now proceed with the implementation of the changes.

First, I will create the new file `packages/20_utility_meters.yaml` to track the monthly peak.
