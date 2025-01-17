---
title: "ESPHome thermostaat voor Daikin Altherma 3"
tags: ESPHome Daikin Warmtepomp
---

In [een vorig artikel](/daikin-altherma-3-lokaal-aansturen) keek ik naar welke manieren mijn warmtepomp lokaal aan te sturen is. Dit leverde een Modbus RTU interface op waarmee ik de gewenste functies van de warmtepomp kan aansturen. Ik dit artikel beschrijf ik hoe mijn eigen gebouwde thermostaat hiervan gebruik maakt, wat ik er mee probeer te bereiken en hoe dat werkt.

Inmiddels hangt aan de woonkamer muur een AirGradient luchtkwaliteitsmeter welke als thermostaat fungeert, in plaats van de Madoka thermostaat die met de warmtepomp mee kwam.
 
todo: Foto van de AirGradient thermostaat aan de muur

## Doel

Uiteraard is het super gaaf en leerzaam om zelf iets te bouwen, maar een lijst van redenen waarom je dit doet is een goed idee :P.

Na ruim een jaar de warmtepomp in gebruik te hebben, kwam ik de volgende problemen tegen die ik graag wil oplossen.

- **Verstoring gemeten temperatuur**: De Daikin thermostaat heeft een afwijking van 0,3-0,4 graden naar boven, zodra deze aanslaat. Dit komt doordat de ledjes die aan gaan bij aanslaan warm worden. Dit verstoort de meting van de thermostaat. Ik verwacht beter van een thermostaat
- **Afwezige modulatie**: De modulatie van de thermostaat werkt niet goed. Hoewel deze correct de aanvoertemperatuur aanpast gebaseerd op de huidige woonkamer temperatuur, zal hij toestaan dat de woonkamer temperatuur stijgt tot 1,5 graden boven het setpoint voordat hij afslaat. Dit is niet aan te passen. Na contact met Daikin geven zij toe dat dit niet goed werkt, en adviseren zij om modulatie te zetten. Dit maakt het effectief een aan/uit thermostaat, wat weer zorgt voor erg schommelde temperatuur in de woonkamer. Dit kan veel beter.
- **Schommelde woonkamer temperatuur**: De thermostaat heeft een hysteresis van 0,5 graden aan de boven en onder kant van de set temperatuur. Combineer dit met een correct afgestelde stooklijn die bedoeld is om de warmtepomp zo rustig mogelijk te laten draaien en dus een temperatuur aan te houden ipv dat de temperatuur stijgt, en je krijgt een schommeling van ruwweg 1,5 graden in de woonkamer over enkele dagen. Dit kan veel beter.
- **Beperkt programma**: De thermostaat is handmatig instelbaar in stappen van 0,5 graden. De planning is maar per hele graad in te stellen.
- **Efficienter gebruik van de backup heater**: Als het buiten warmer is dan 25 graden, zal de warmtepomp maximaal 45 graden water kunnen produceren met de buiten unit. Warmer water dan dat zal dan altijd met de backup-heater gemaakt worden. Deze is 2-3x zo inefficiënt als de buitenunit en wil je zo veel mogelijk voorkomen.
  - De wekelijkse desinfectie run (waarbij de tank op 60 graden wordt verwarmd) zal dus een groot deel op de backup heater draaien in de zomer. In de zomer voer je dit dus liever uit in de vroege ochtend, en in de winter op het warmst van de dag. Het moeten aanpassen van deze instelling elk seizoen is vervelend.
  - De tank continu instellen op 45 graden is onvoldoende voor ons op koudere dagen. Daarvoor heeft de warmtepomp een stooklijn voor de tank, zodat deze op koudere dagen op een hogere temperatuur aanhoudt. Dit werkt oke, behalve dat de lage buitentemperatuur maximaal 10 graden is. Dit is voor mijn usecase (2 regendouches en een groot bad) weer te laag, en wil ik graag hoger aanhouden.

Daarnaast mis ik op dit moment de volgende features:
- **Weersafhankelijke / zon sturing**: Als de zon veel schijnt, hoeft de verwarming niet zo hard te werken. Moduleer hem dan terug.
- **Openhaard modus**: Normaliter als je de openhaard aan maakt, zal de thermostaat de verwarming af laten slaan omdat het warm genoeg is in de woonkamer. Omdat we geen zoneregeling hebben zorgt dit er echter voor dat de bovenverdieping ook niet meer warm wordt. Graag negeer ik de woonkamer temperatuur en laat ik 'm doordraaien op zijn stooklijn (gebaseerd op de buiten temperatuur).
- **Efficiëntere desinfectie run**: We zetten geregeld zelf de tank op 60 graden om lang te kunnen douchen en uitgebreid in bad te zitten. Dit is effectief een desinfectie run, dus die laat ik dan ook graag als zodanig meetellen, zodat hij niet onnodig een dag later zelf verzint om de tank op 60 graden te verwarmen.

## Vereisten

- Een op zichzelfstaand systeem, onafhankelijk van het functioneren van Home Assistant of het computer netwerk. Optimalisaties mogen best afhankelijk zijn van informatie van Home Assistant of het internet, maar de basisfunctionaliteit voor het verwarmen van het huis moet robust en onafhankelijk functioneren.
- Bij voorkeur zo min mogelijk zelfbouw hardware en zo veel mogelijk gebruik makend van bestaande hardware en open source software.
- Mogelijkheden om eigen logica te implementeren of bestaande logica zodanig aan te passen naar mijn wensen. Dit is ten slotte een hobby project :P.

## Implementatie

### Modbus RTU aansturing

todo: afbeeldingen Modbus RTU interface tussen Home Hub en ESPHome microcontroller.

Door de thermostaat logica te draaien in een microcontroller in plaats van Home Assistant, kan deze zelfstandig functioneren en verbruikt het weinig stroom. Omdat ik Home Assistant gebruik, en ESPHome ondersteuning heeft voor Modbus RTU en verschillende thermostaat logica's, is het ook een logische keuze om ESPHome te gebruiken als firmware voor de microcontrollers.

Jurriaan heeft voor een van zijn projecten al eerder een ESP32 met een modbus RTU interface gebouwd. De combinatie is heel gangbaar, waardoor dit in de toekomst ook makkelijk te vervangen is mocht het defect raken. Hij heeft voor mij eenzelfde combinatie gebouwd met een display en transparante behuizing, welke ik zonder enige verdere aanpassingen kan gebruiken. Alleen de software die er op draait (ESPHome) moet ik zelf inrichten, wat precies is wat ik zocht.

todo: afbeeldingen modbus ESP32 met behuizing en display
subtext: Modbus RTU is een simpele interface en makkelijk aan te sturen met een microcontroller. Het werkt met maar 2 aders tussen 2 apparaten en werkt onafhankelijk van het thuisnetwerk.

Deze plaatsen we in de meterkast. Ik kom later terug op waarom.

### Woonkamer temperatuur meten

todo: afbeeldingen AirGradient

todo: afbeeldingen bluetooth sensor

Vervolgens moet ik de huidige woonkamer temperatuur meten. Ik heb daar al een AirGradient luchtkwaliteitsmeter staan. Die is ook al voorzien van ESPHome en een display met wat nuttige informatie. Ik hergebruik graag deze in plaats van weer een extra apparaat in de woonkamer te hebben staan. Deze kan prima aan de muur hangen waar nu de thermostaat zit. 

Aangezien de AirGradient een standaard ESP32 gebruikt, beschikt hij zowel wifi als bluetooth. Het is dan gemakkelijk de sofware uit te breiden om deze als een generieke bluetooth temperatuur sensor te laten gedragen.

### Thermostaat logica

Nu is de vraag, waar draaien we de thermostaat logica? Ik heb 2 opties:
- **In de AirGradient** aan de muur in de woonkamer, en de microcontroller in de meterkast alleen een simpele Modbus RTU client.
- **In de microcontroller in de meterkast**, en de AirGradient een generieke bluetooth temperatuur sensor.

Ik ga voor de tweede optie, omdat:
- Bij wegvallen van de bluetooth verbinding met de AirGradient, zal de verwarming nog steeds beperkt kunnen functioneren, gebaseerd op alleen de buiten temperatuur.
- De AirGradient gedraagt zich als een generieke bluetooth temperatuur sensor, wat het makkelijk maakt om deze in de toekomst eventueel te vervangen door een andere sensor.

## Tijd om te bouwen!

Goed, we hebben alles compleet om te kunnen bouwen!

todo: diagram esphome microcontroller verbonden met de Home Hub. En een thermometer in de woonkamer. Ook waar wifi op zit en wat verbonden is met Home Assistant. Ook de binnenunit met espaltherma is leuk om weer te geven.

### Modbus registers

### BLE verbinding betrouwbaar maken

### ESPHome thermostaat component en overnemen bestaande thermostaat

- vergeet ook niet heractivatie bij herstart, want restore werkt niet?

### Openhaard modus

### Overnemen tank stooklijn en desinfectie run

- stooklijn zat nog in de binnenunit, dus die moet ik uitzetten.
- desinfectie run zat volledig in home assistant, en zal nu gesplits zijn tussen esphome en home assistant.

...

## Testen

Wokwi simulatie

## Ervaringen tot dusver