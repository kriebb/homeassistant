# Opschoning Plan - Van 20 naar 5 packages

## Huidige Chaos (20 packages)
Te veel overlap, dubbele sensors, test code, lege files...

## Nieuwe Structuur (5 essentiële packages)

### 1. **energy_core.yaml**
**Combineert**: 00_energy_core + 50/51_capaciteitstarief + basis sensors
- Maandpiek tracking (sensor.home_import_power_peak_31d)
- Capaciteitslimiet en beschikbare capaciteit
- Belgisch capaciteitstarief berekeningen
- Geen dubbele sensors meer!

### 2. **ev_charging.yaml**
**Combineert**: Alleen 12_wallbox_improved + 55_laadsessie
- Verbeterde load balancer (minder Kia meldingen)
- Laadsessie tracking
- Verwijder oude/legacy wallbox packages

### 3. **energy_costs.yaml** 
**Combineert**: 52_gas + 53_totaal + 54_tarieven
- Alle energie tarieven op één plek
- Gas + elektriciteit kosten
- Totale kosten berekeningen

### 4. **appliances.yaml**
**Combineert**: 30_predictive + 40_monitoring
- Slimme apparaat monitoring
- Alleen relevante notificaties
- Verwijder device_fingerprints (onafgemaakt)

### 5. **dashboard_helpers.yaml**
**Combineert**: 60 + 61 dashboard helpers
- Alleen echt gebruikte helpers
- Verwijder placeholders

## Te Verwijderen
- 00_energy_test.yaml (test data)
- 10_wallbox_dynamic.yaml (legacy)
- 11_wallbox_load_balancer.yaml (vervangen door 12)
- 20_device_fingerprints.yaml (onafgemaakt)
- 20_utility_meters.yaml (leeg)
- 62_system_monitoring_placeholders.yaml (fake data)
- 70_recorder_optimization.yaml (naar configuration.yaml)
- 99_cleanup_unavailable.yaml (eenmalig)

## Voordelen
✅ 75% minder packages (20 → 5)
✅ Geen dubbele sensors meer
✅ Duidelijke structuur
✅ Makkelijker te beheren
✅ Betere performance

## Wil je dat ik:
1. **OPTIE A**: De 5 nieuwe packages maak en je kan zelf de oude verwijderen?
2. **OPTIE B**: Een migratie script maak dat alles automatisch doet?
3. **OPTIE C**: Stap voor stap met jou doorloop?

De dashboards blijven werken, maar met een veel schonere backend!