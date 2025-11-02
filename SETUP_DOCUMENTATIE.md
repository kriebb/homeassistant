# Home Assistant Energie Setup Documentatie

## Overzicht Systeem

Deze Home Assistant setup is geoptimaliseerd voor energie management in België met energie.be als leverancier. Het systeem beheert EV (Electric Vehicle) laden, slimme apparaten, en capaciteitstarief optimalisatie.

### Hardware Setup
- **Slimme meter**: ETH Dongle Pro (192.168.5.26) via MQTT naar HA (192.168.5.41:1883)
- **EV Lader**: Wallbox Kristof Riebbels (Pulsar charger)
- **Auto**: Kia EV3
- **Slimme stopcontacten**:
  - Sonoff smartmeter stopcontacten voor vaatwasser, droogkast, wasmachine, boiler, waterkoker
  - Fritz smart stopcontacten voor living (home cinema) en bureau (werk)

### Energie Leverancier
- **Provider**: energie.be
- **Tarief**: Variabel tarief (geen dag/nacht verschil)
- **Capaciteitstarief**: Vlaams capaciteitstarief van toepassing
- **Huidige maandpiek**: 4,362 kW (november 2025)

## Packages Overzicht

### Core Energy Packages

#### `00_energy_core.yaml`
- **31-day peak tracking**: `sensor.home_import_power_peak_31d`
- **Capacity limit**: Gebruikt de hoogste waarde tussen maandpiek en contract limiet
- **Available capacity**: Berekent beschikbare capaciteit voor extra verbruik

#### `11_wallbox_load_balancer.yaml` (Legacy)
- Originele EV load balancer
- Nu gecommentarieerd maar behouden voor referentie

#### `12_wallbox_load_balancer_improved.yaml` (Nieuwe versie)
- **Verminderde Kia notificaties**: Alleen aanpassen bij significante wijzigingen (≥2A)
- **Cooldown periode**: Minimaal 60 seconden tussen aanpassingen
- **Safety first**: Onmiddellijke actie bij capaciteitsoverschrijding
- **Hysterese**: Voorkomt constant schakelen

### Appliance Management

#### `40_smart_appliance_monitoring.yaml`
- **Status tracking**: Monitort vaatwasser, wasmachine, droogkast
- **Alleen relevante meldingen**: Enkel bij onderbreking tijdens actief gebruik
- **Auto-dismiss**: Meldingen verdwijnen automatisch bij herinschakeling

#### `30_predictive_scheduler.yaml` (Bestaand)
- Voorspellende planning voor witgoed
- Start apparaten wanneer capaciteit beschikbaar is

### Capacity Management

#### `50_capaciteitstarief_belgie.yaml`
- **Vlaams capaciteitstarief**: Beheert maandpiek optimalisatie
- **Waarschuwingen**: Bij nieuwe piek instelling
- **Status monitoring**: Real-time capaciteit percentage

#### `70_recorder_optimization.yaml`
- **Database optimalisatie**: Automations voor periodieke opruiming
- **Backup support**: Reduceert database grootte voor succesvolle backups

### Dashboard Support

#### `60_dashboard_missing_sensors.yaml`
- **Ontbrekende sensors**: Alle sensors die dashboards nodig hebben
- **Tarief simulatie**: Dag/nacht tarief voor energie.be (informatief)
- **Appliance helpers**: Input booleans voor automation modes

## Configuratie Changes Needed

### 1. Recorder Optimalisatie (BELANGRIJK voor backup fix)

Voeg toe aan `configuration.yaml`:

```yaml
recorder:
  commit_interval: 30
  purge_keep_days: 7
  auto_purge: true
  auto_repack: true
  exclude:
    domains:
      - automation
      - camera
      - weather
      - sun
    entity_globs:
      - sensor.*_linkquality
      - sensor.*uptime*
      - sensor.*time*
      - sensor.*date*
    entities:
      - sensor.time
      - sensor.date
      - sensor.uptime
    event_types:
      - service_registered
      - component_loaded
      - time_changed
```

### 2. Legacy Load Balancer Uitschakelen

De oude `automation.ev_load_balancer_adjust_charging_amps` in `11_wallbox_load_balancer.yaml` moet uitgeschakeld worden om conflicten te voorkomen met de nieuwe verbeterde versie.

### 3. Predictive Scheduler Aanpassen

Overweeg de notifications in `30_predictive_scheduler.yaml` te reduceren tot alleen onderbreking meldingen.

## Key Features

### EV Load Balancing
1. **Dynamische aanpassing**: Past laadstroom aan binnen capaciteitslimiet
2. **Maandpiek benutting**: Mag volledige maandpiek gebruiken (4,362 kW)
3. **Veiligheidsmarges**: Instelbare margins en hysterese
4. **Manual override**: Force max laden mogelijk via `input_boolean.ev_force_max`

### Capaciteitstarief Optimalisatie
1. **Real-time monitoring**: Huidige verbruik vs maandpiek/contract
2. **Preventie nieuwe piek**: Waarschuwingen bij overschrijding
3. **Apparaat integratie**: Slimme apparaten rekening houdend met capaciteit

### Database Management
1. **Geoptimaliseerde opslag**: Exclude high-frequency sensors
2. **Automatische cleanup**: Weekly purge and repack
3. **Backup support**: Reduced size voor succesvolle backups

## Troubleshooting

### Kia App Meldingen
- **Oorzaak**: Load balancer past amperage te vaak aan
- **Oplossing**: Nieuwe load balancer met 2A minimum change en cooldown

### Database Backup "Lack of Resources"
- **Oorzaak**: Database te groot voor beschikbare ruimte
- **Oplossing**: Recorder optimalisatie + manual purge met repack

### Dashboard Configuratie Fouten
- **Oorzaak**: Ontbrekende sensors en verkeerde entity namen
- **Oplossing**: Package 60 met alle missing sensors

### Te Veel Meldingen (1354)
- **Voorkomen**: Optimized notification automations
- **Focus**: Alleen relevante onderbrekingen en waarschuwingen

## Monitoring & Maintenance

### Dagelijkse Checks
- Dashboard functionaliteit (geen rode errors)
- EV load balancer status (`binary_sensor.ev_load_balancer_ready`)
- Capaciteit status (`sensor.capaciteit_status`)

### Wekelijkse Maintenance
- Database purge logs controleren
- Maandpiek progression monitoren
- Notification relevantie evalueren

### Maandelijkse Review
- Energy consumption trends
- Capacity tariff optimization effectiveness
- System performance metrics