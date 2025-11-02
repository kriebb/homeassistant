# Fix voor Automation Error

## Het probleem:
De automation "EV Load Balancer - Adjust Charging Amps" is al volledig gecommentarieerd in het bestand, maar Home Assistant heeft nog de oude versie in geheugen met de foutieve `enabled: false` parameter.

## Oplossing:

### Optie 1: Via UI (Aanbevolen)
1. Ga naar **Settings → Automations & Scenes**
2. Zoek "EV Load Balancer - Adjust Charging Amps" 
3. Klik op de 3 puntjes rechts → **Delete**
4. Bevestig verwijdering

### Optie 2: Via Developer Tools
1. Ga naar **Developer Tools → Services**
2. Service: `automation.reload`
3. Klik **Call Service**

Als dit niet werkt:
1. Service: `homeassistant.delete`
2. Target: `automation.ev_load_balancer_adjust_charging_amps`
3. Klik **Call Service**

### Optie 3: Herstart Home Assistant
1. Ga naar **Settings → System → Restart**
2. Kies **Restart Home Assistant**

## Controle:
Na een van bovenstaande acties zou de foutmelding weg moeten zijn. De automation is al correct gecommentarieerd in het YAML bestand, dus na herladen/herstarten zal hij niet meer terugkomen.

## Belangrijk:
- De nieuwe verbeterde versie in `packages/12_wallbox_load_balancer_improved.yaml` blijft gewoon actief
- Alleen de oude versie wordt verwijderd