# Fix voor Dubbele Automations en Unavailable Entities

## STAP 1: Schakel de oude EV load balancer uit

Ga naar Home Assistant UI:
1. Ga naar Settings → Automations & Scenes
2. Zoek "EV Load Balancer - Adjust Charging Amps"
3. Klik erop en schakel het UIT met de toggle

OF voeg dit toe aan je automation in `packages/11_wallbox_load_balancer.yaml`:

```yaml
  - alias: "EV Load Balancer - Adjust Charging Amps"
    id: ev_load_balancer_adjust_charging_amps
    enabled: false  # <-- ADD THIS LINE
```

## STAP 2: Verwijder "unavailable" restored automations

Deze oude automations kunnen verwijderd worden via Developer Tools:

### Lijst van te verwijderen automations:
- automation.capaciteitsbescherming_slimme_apparaten
- automation.dal_uren_capaciteitsaanpassing
- automation.droogkast_cyclus_detectie
- automation.droogkast_cyclus_detectie_2
- automation.droogkast_cyclus_einde_detectie
- automation.droogkast_opstart_evaluatie
- automation.droogkast_opstart_evaluatie_2
- automation.evohome_eco_during_peak_hours
- automation.evohome_eco_mode_off
- automation.evohome_eco_mode_toggle

### Hoe te verwijderen:
1. Ga naar Developer Tools → Services
2. Service: `homeassistant.delete`
3. Target: Selecteer elke unavailable automation één voor één
4. Call Service

OF gebruik dit script om ze allemaal in één keer te verwijderen:

```yaml
service: homeassistant.delete
target:
  entity_id:
    - automation.capaciteitsbescherming_slimme_apparaten
    - automation.dal_uren_capaciteitsaanpassing
    - automation.droogkast_cyclus_detectie
    - automation.droogkast_cyclus_detectie_2
    - automation.droogkast_cyclus_einde_detectie
    - automation.droogkast_opstart_evaluatie
    - automation.droogkast_opstart_evaluatie_2
    - automation.evohome_eco_during_peak_hours
    - automation.evohome_eco_mode_off
    - automation.evohome_eco_mode_toggle
```

## STAP 3: Herstart Home Assistant

Na het uitschakelen van de oude automation en verwijderen van unavailable entities:
1. Ga naar Settings → System → Restart
2. Kies "Restart Home Assistant"

## Waarom dit belangrijk is:

1. **Dubbele load balancers** = Conflicterende commando's naar je wallbox = Veel Kia meldingen
2. **Unavailable automations** = Onnodige database entries = Grotere backup
3. **Restored entities** = Kunnen errors veroorzaken in logs

Na deze stappen zou je:
- Veel minder Kia meldingen moeten krijgen
- Een schonere entity lijst hebben
- Mogelijk een kleinere database voor backups