# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Overview

This is a Home Assistant configuration repository focused on energy management and EV (Electric Vehicle) charging load balancing. It's not the Home Assistant core codebase but rather a personal configuration setup using YAML and Jinja2 templates.

## Development Commands

### Configuration Management
- **Check configuration validity**: `hass --script check_config`
- **Reload automations**: Via Home Assistant UI → Developer Tools → YAML Configuration Reloading
- **Version control**: Standard git commands for tracking configuration changes

### Testing
- Manual testing using the Pass/Fail Checklist in REPORT.md
- Verify `binary_sensor.ev_load_balancer_ready` status before enabling EV charging features

## Code Architecture

### Directory Structure
```
homeassistant/
├── configuration.yaml          # Main entry point, loads packages
├── packages/                   # Modular configuration packages
│   ├── 00_energy_core.yaml    # Core energy monitoring sensors
│   ├── 11_wallbox_load_balancer.yaml # EV load balancing system
│   └── [other packages]       # Additional feature modules
├── blueprints/                 # Reusable automation/script templates
├── dashboards/                 # Custom Lovelace UI configurations
└── automations/scripts/scenes.yaml # Simple automations/scripts/scenes
```

### Key Architectural Patterns

1. **Package-based Configuration**: All complex logic is modularized in `packages/` directory
2. **Template Sensors**: Heavy use of Jinja2 templates for calculated values with availability guards
3. **Helper Entities**: Input numbers/booleans for runtime configuration without YAML edits
4. **Load Balancing Logic**: Sophisticated EV charging control that:
   - Monitors grid capacity limits (31-day peak tracking)
   - Adjusts charging current dynamically with hysteresis
   - Includes safety margins and manual override capabilities
   - Pauses charging when capacity would be exceeded

### Important Entity References

**Core Power Monitoring**:
- `sensor.eth_dongle_pro_power_delivered` - Grid import power
- `sensor.eth_dongle_pro_power_returned` - Grid export power
- `sensor.home_import_power_peak_31d` - Rolling 31-day peak

**EV Charging Control**:
- `number.wallbox_kristof_riebbels_maximale_laadstroom` - Charging current control
- `switch.wallbox_kristof_riebbels_pauzeren_hervatten` - Pause/resume charging
- `sensor.wallbox_kristof_riebbels_laadvermogen` - Current charging power

**Configuration Helpers**:
- `input_number.ev_main_limit_w` - Grid capacity limit
- `input_number.ev_margin_w` - Safety buffer
- `input_boolean.ev_force_max` - Manual override switch

### Development Guidelines

1. **Availability Guards**: Always check sensor availability before calculations
2. **Hysteresis**: Implement buffers to prevent rapid oscillations in automations
3. **Fallback Values**: Provide sensible defaults when sensors are unavailable
4. **Entity Naming**: Follow existing patterns (e.g., `sensor.ev_*` for EV-related sensors)
5. **Package Organization**: Group related functionality into dedicated package files
6. **Testing**: Verify changes don't exceed grid limits using the Pass/Fail Checklist criteria