# Capaciteitstarief Setup - Energie.be België

## Wat is geïmplementeerd:

### 1. **Basis Capaciteitstarief Monitoring** (`packages/50_capaciteitstarief_belgie.yaml`)
- Tracking van 31-dagen rollende piek
- Minimum bijdrage van 2.5 kW (volgens Energie.be)
- Real-time capaciteit status met percentages
- Waarschuwingen bij nieuwe maandpiek

### 2. **Uitgebreide Monitoring** (`packages/51_capaciteitstarief_monitoring.yaml`)
- Kwartier vermogen tracking (capaciteitstarief wordt per 15 min berekend)
- Voorspelling nieuwe maandpiek
- Ruimte tot volgende kW drempel
- Geschatte maandelijkse kosten
- Waarschuwingen bij naderen kW drempels

### 3. **Database Optimalisatie** (`recorder.yaml`)
- Excludes voor high-frequency sensors (voltage, current)
- Behoudt belangrijke energy/power sensors
- Reduceert database grootte voor backups

## Belangrijke verschillen met Energie.be:

| Wat | Home Assistant | Energie.be |
|-----|---------------|------------|
| Piek berekening | 31 dagen rollend | Per kalendermaand |
| Facturatie | n.v.t. | 12-maands gemiddelde |
| Minimum | 2.5 kW | 2.5 kW ✓ |
| Kwartiermeting | Benadering | Exacte 15-min metingen |

## Hoe te gebruiken:

### 1. **Dashboard Widgets**
Voeg deze cards toe aan je dashboard:

```yaml
# Capaciteit Status Card
type: custom:mushroom-template-card
primary: Capaciteit Status
secondary: "{{ states('sensor.capaciteit_status') }}"
icon: mdi:flash
icon_color: >
  {% set percentage = state_attr('sensor.capaciteit_status', 'percentage_used') | float(0) %}
  {% if percentage < 50 %} green
  {% elif percentage < 80 %} orange
  {% else %} red
  {% endif %}

# Maandpiek Card
type: entities
entities:
  - entity: sensor.home_import_power_peak_31d
    name: Huidige maandpiek
  - entity: sensor.voorspelde_maandpiek_indien_nu_stoppen
    name: Als je nu zou stoppen
  - entity: sensor.ruimte_tot_volgende_kw_drempel
    name: Ruimte tot volgende kW
```

### 2. **Automations voor Optimalisatie**

De EV load balancer houdt al rekening met de maandpiek. Additionele optimalisaties:

1. **Spreiding van verbruik**: Start zware apparaten (vaatwasser, wasmachine) op verschillende tijden
2. **Kwartier-aware**: Vermijd nieuwe pieken in laatste 5 minuten van een kwartier
3. **Drempel monitoring**: Blijf onder ronde kW getallen (4kW, 5kW, etc.)

### 3. **Tips voor Kostenbesparing**

1. **Ken je huidige piek**: Check `sensor.home_import_power_peak_31d`
2. **Gebruik de volledige piek**: Als je al 4.362 kW piek hebt, mag je tot die waarde zonder extra kosten
3. **Vermijd nieuwe pieken**: Elke kW extra kost ~€3/maand voor 12 maanden
4. **Spreiding**: Gebruik niet alles tegelijk (EV + kookplaat + oven = probleem)

## Monitoring Commands:

```yaml
# In Developer Tools -> Services

# Check huidige capaciteit info
service: system_log.write
data:
  message: >
    Maandpiek: {{ states('sensor.home_import_power_peak_31d') }}W
    Huidig verbruik: {{ states('sensor.ev_house_power') }}W
    Beschikbaar: {{ states('sensor.available_capacity') }}W
  level: warning

# Force database cleanup voor kleinere backup
service: recorder.purge
data:
  keep_days: 7
  repack: true
```

## Toekomstige Verbeteringen:

1. **Integratie met energieprijzen**: Combineer capaciteit met variabele prijzen
2. **12-maands historie**: Bouw eigen database op voor echte gemiddelde berekening
3. **Fluvius API**: Als beschikbaar, haal werkelijke maandpieken op
4. **Predictive scheduling**: Plan apparaten automatisch binnen capaciteitsgrenzen

## Troubleshooting:

**Q: Waarom wijkt mijn piek af van Fluvius?**
A: Wij meten rollend 31 dagen, Fluvius per kalendermaand

**Q: Kan ik mijn historische pieken zien?**
A: Ja, via History van `sensor.home_import_power_peak_31d`

**Q: Hoe reset ik mijn maandpiek?**
A: Dat kan niet - het is een rollend 31-dagen maximum. Na 31 dagen verdwijnt een oude piek vanzelf.