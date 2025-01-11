---
title: "ESPHome thermostaat voor Daikin Altherma 3"
tags: ESPHome Daikin Warmtepomp
---

In [een vorig artikel](/daikin-altherma-3-lokaal-aansturen) keken we naar welke manieren mijn warmtepomp lokaal aan te sturen is. Dit bracht ons tot de Daikin Home Hub welke een Modbus interface beschikbaar stelt.

In dit artikel beschrijf ik hoe ik een ESPHome-based thermostaat heb gemaakt voor de Daikin Altherma 3 warmtepomp.

## Waarom een eigen thermostaat?

Ik heb besloten mij eigen thermostaat te bouwen om de volgende problemen op te lossen met de Daikin Altherma 3 warmtepomp

- De Daikin thermostaat heeft een afwijking van 0,3-0,4 graden te hoog zodra deze aanslaat. Dit komt doordat de ledring aan gaat en warm wordt. Dit verstoort de meting van de thermostaat. Ik verwacht beter van een merk thermostaat.
- De modulatie van de thermostaat werkt niet goed. Hoewel deze correct de aanvoertemperatuur aanpast gebaseerd op de huidige woonkamer temperatuur, zal hij toestaan dat de woonkamer temperatuur mag stijgen tot 1,5 graden boven het setpoint voordat hij afslaat. Dit is niet aan pas passen en Daikin zelf adviseert om deze functie uit te zetten. Dit maakt het effectief een aan/uit thermostaat, wat weer zorgt voor erg schommelde temperatuur in de woonkamer.
- De thermostaat heeft een hysteresis van 0,5 graden aan de boven en onder kant van de set temperatuur. Met een correct afgestelde stooklijn zorgt dit er voor dat hij er meerdere dagen over kan doen om de bovenkant van die setpoint te bereiken. Zodra hij dat doet schakelt hij de warmtepomp uit totdat de onderkant van het setpoint bereikt is. Effectief is dit dus een graad verschil. Vooral wanneer het kouder is buiten, zal hij, wederom met een correct afgestelde stooklijn, er weer even over doen om de daling te stoppen, wat dus voor verdere daling van de woonkamer temperatuur zorgt, voordat deze weer stijgt. In de praktijk berekent dat, dat de woonkamer elke paar dagen ruwweg 1,5 graden kan schommelen. Dit is onacceptabel.
- De thermostaat is instelbaar in stappen van 0,5 graden. De planning is nog onnauwkeuriger en maar per hele graad in te stellen. Dit moet beter kunnen.
- De buitenunit weigert warmter dan 45 graden water te producteren bij een buitentemperatuur boven de 25 graden. 
  - Het uitvoeren van een desinfectie run, waarbij de tank op 60 graden wordt verwarmd, zorgt er dan voor dat een groot deel op de backup-heater gedraaid wordt. In de zomer voer je dit dus liever uit in de vroege ochtend, en in de winter op het warmst van de dag / overdag. Het moeten aanpassen van deze instelling elk seizoen is vervelend.
  - De tank ondersteund een stooklijn, waardoor hij bij koudere buitentemperaturen een hogere temperatuur in de tank aanhoudt, wat ik kan gebruiken om dit probleem heen te werken, echter is deze stooklijk aan de max buitentemperatuur kant beperkt tot 10 graden of lager. Dit wil ik graag verder kunnen tweaken.

Ik zou graag de volgende slimme features willen implementeren

- slimme aansturing / buffering tank/woonkamer
- openhaard modus
- desinfectie neemt handmatig naar 60-graden verwarmen mee

## Vereisten

- Volledig op zichzelf staand werkende thermostaat. Zelfs als het netwerk en Home Assistant uit valt, moet dit nog blijven werken.
- Liever off-the-shelf oplossinngen, eventueel flashed met open source software, maar zo min mogelijk zelfbouw hardware.

## Implementatie

Modbus RTU ipv Modbus TCP maakt dit onafhankelijk van netwerk. De verbinding is simpel en betrouwbaar en goed ondersteund door ESPHome.



Modbus is een makkelijk aan te sturen protocol met een microcontroller. Door het in een microcontroller te implementeren, kan deze zelfstandig functioneren en verbruikt het weinig stroom. Door Modbus RTU 