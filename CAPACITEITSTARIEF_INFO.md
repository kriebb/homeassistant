# Capaciteitstarief België - Energie.be Implementatie Details

## Belangrijke informatie uit Energie.be artikel:

### 1. **Effectieve vs Gemiddelde Maandpiek**
- **Effectieve maandpiek**: Hoogste kwartiervermogen van de maand
- **Gemiddelde maandpiek**: Gemiddelde van 12 maanden (gebruikt voor facturatie)
- Dit betekent dat één hoge piek verzacht wordt over 12 maanden

### 2. **Minimumbijdrage**
- Klassieke meters: 2,5 kW minimum
- Dit is waarom onze default in de config 4362W is (jouw huidige piek)

### 3. **Maximumtarief**
- Voor kleine verbruikers (<750 kWh/jaar): max 0,35 EUR/kWh
- Niet van toepassing op jouw situatie met EV

## Aanpassingen voor je Home Assistant:

### Update Capaciteitstarief Package
We moeten het systeem aanpassen om rekening te houden met de 12-maands gemiddelde berekening:

```yaml
# In packages/50_capaciteitstarief_belgie.yaml toevoegen:
template:
  - sensor:
      - name: "Gemiddelde maandpiek 12 maanden"
        unique_id: average_month_peak_12m
        state: >
          {% set current_peak = states('sensor.home_import_power_peak_31d') | float(0) %}
          {# In werkelijkheid zou dit het gemiddelde van 12 maanden moeten zijn #}
          {# Voor nu gebruiken we de huidige maandpiek als benadering #}
          {{ current_peak }}
        unit_of_measurement: W
        attributes:
          info: "Dit is een benadering - werkelijke waarde komt van Fluvius/Energie.be"
          current_month_peak: "{{ states('sensor.home_import_power_peak_31d') }}"
```

### Voor je configuration.yaml:

```yaml
# Voeg dit toe aan configuration.yaml voor minimum bijdrage:
input_number:
  capaciteitstarief_minimum_kw:
    name: Minimum capaciteitstarief
    min: 2.5
    max: 5
    step: 0.1
    initial: 2.5
    unit_of_measurement: kW
    icon: mdi:flash-outline
```

## Praktische tips gebaseerd op het artikel:

1. **Spreiding is key**: Je EV load balancer helpt al door laden aan te passen
2. **Kwartiervermogen**: Elke 15 minuten wordt gemeten, dus korte pieken <15min zijn OK
3. **12-maands effect**: Een hoge piek blijft 12 maanden meetellen maar verzacht

## Monitoring suggestie:

We kunnen een sensor toevoegen die waarschuwt wanneer je boven bepaalde drempels komt:

```yaml
binary_sensor:
  - platform: template
    sensors:
      capaciteit_boven_5kw:
        friendly_name: "Capaciteit boven 5kW"
        value_template: >
          {{ states('sensor.ev_house_power') | float(0) > 5000 }}
      capaciteit_boven_6kw:
        friendly_name: "Capaciteit boven 6kW"
        value_template: >
          {{ states('sensor.ev_house_power') | float(0) > 6000 }}
```

Dit helpt je om pieken te vermijden die je 12-maands gemiddelde omhoog duwen.