---
title: "ESPHome thermostaat voor Daikin Altherma 3"
tags: ESPHome Daikin Warmtepomp
hidden: true
---

In [een vorig artikel](/daikin-altherma-3-lokaal-aansturen) keek ik naar welke manieren mijn warmtepomp lokaal aan te sturen is. Dit leverde een Modbus RTU interface op waarmee ik de gewenste functies van de warmtepomp kan aansturen. Ik dit artikel beschrijf ik hoe mijn eigen gebouwde thermostaat hiervan gebruik maakt, wat ik er mee probeer te bereiken en hoe dat werkt.

Inmiddels hangt aan de woonkamer muur een AirGradient luchtkwaliteitsmeter welke als thermostaat fungeert, in plaats van de Madoka thermostaat die met de warmtepomp mee kwam.
 
![](/assets/images/daikin_altherma_3/airgradient_aan_muur.jpg){: width="300" } ![](/assets/images/daikin_altherma_3/airgradient.png){: width="300" }

Links zoals hij aan de muur hangt met aangepaste ESPHome firmware, en rechts een productfoto met originele firmware. Ja, ik moet de stroomvoorziening nog netjes wegwerken, maar het is een mooi apparaat aan de muur. Het kan naast de temperatuur ook de luchtkwaliteit meten en heeft ledjes en een display voor nuttige informatie, niet alleen voor de luchtkwaliteit, maar ook andere meldingen vanuit Home Assistant kunnen hier als notificaties op weergegeven worden.

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
- Alles van netstroom voorzien. Geen batterijen vervangen elk jaar.

## Implementatie

### Modbus RTU aansturing met een ESP32 microcontroller

Modbus RTU is een simpele 2 aderige interface (alleen de A+/B- aansluitingen zijn nodig) die direct tussen deze 2 apparaten werkt. Door een simpele fysieke verbinding te gebruiken en een microcontroller, kan dit zelfstandig functioneren.

![](/assets/images/daikin_altherma_3/uart-to-rs485.png){: width="300" } ![](/assets/images/daikin_altherma_3/homehub-connectors.png){: width="300"} 

De Daikin Home Hub (rechts) heeft naast stroom en een P1P2 verbinding (zie X6A op de afbeelding), ook een RS485 interface (zie X8A). Dit is voor Modbus RTU. Deze is makkelijk aan te sturen met een ESP32 microcontroller met een UART-naar-RS485 transceiver (links). 

In de afbeelding hieronder zie je een behuizing met daarin een ESP32 microcontroller, een RS485 transceiver (MAX485 IC) en een display. Jurjen had deze al eerder gebouwd voor een ander project, en heeft voor mij een zelfde combinatie gebouwd met een display en transparante behuizing, welke ik zonder enige verdere aanpassingen kan gebruiken. Neat. De combinatie van ESP32 en RS485 transceiver is heel gangbaar, waardoor dit in de toekomst ook makkelijk te vervangen is mocht het defect raken. 

[![](/assets/images/daikin_altherma_3/esp32_modbus.jpg){: width="400" }](/assets/images/daikin_altherma_3/esp32_modbus.jpg)

De grijze kabel is de RS485 verbinding met de Daikin Home Hub, en de zwarte USB kabel dient alleen voor voeding. Communicatie met de AirGradient verloopt via Bluetooth Low Energy en met Home Assistant via Wifi.

Voor de software op de ESP32 heb ik ESPHome gekozen, omdat dit al ondersteunding voor [modbus](https://esphome.io/components/modbus_controller.html) en thermostaat logica aan boord heeft, ik het ook gebruik voor de AirGradient, en de verdere (optionele) communicatie met Home Assistant dan erg gemakkelijk is.

Deze microcontroller plaatsen we samen met de Daikin Home Hub in de meterkast.

### De woonkamer temperatuur meten

![](/assets/images/daikin_altherma_3/airgradient.png){: width="300" }

Ik de woonkamer had ik al een AirGradient luchtkwaliteitsmeter staan, die ik ook al voorzien had van ESPHome en een display heeft die wat nuttige informatie kan tonen. Ik hergebruik graag deze in plaats van weer een extra apparaat in de woonkamer te hebben staan. Deze kan prima aan de muur hangen waar nu de thermostaat zit. Aangevuld met wat icoontjes voor warmtevraag en of de tapwater tank gevuld wordt.

In theorie kan ik deze 2 apparaten samenvoegen, maar aangezien ik graag zo min mogelijk aanpassingen of zelfbouw hardware wil gebruiken, is het handiger om deze aan elkaar te koppelen. Aangezien ze beiden een ESP32 gebruiken en deze beschikken over Bluetooth, is dit erg gemakkelijk draadloos te doen met Bluetooth Low Energy! De thermostaatdraden in de muur zijn dus alleen nodig voor voeding.

### Thermostaat logica

Nu is de vraag, in welke van de twee ESP32s draaien we de thermostaat logica? Traditioneel draait dit in het apparaat aan de muur in de woonkamer. Maar, ik heb een beter idee en kies er voor om dat in de meterkast te draaien. Dit maakt dat de AirGradient in de woonkamer zich alleen als een generieke bluetooth temperatuur sensor hoeft te gedragen en dus in de toekomst eventueel te vervangen is door iets anders, of meerdere sensoren bijv voor zoneregeling.

Ook maakt dit het systeem in zijn geheel robuster, omdat bij het wegvallen van de bluetooth verbinding met de AirGradient, de verwarming nog steeds beperkt zal kunnen functioneren, gebaseerd op alleen de buiten temperatuur die door de buitenunit wordt gemeten.

## Tijd om te bouwen!

Goed, we hebben alles compleet om te kunnen bouwen!

todo: left here.

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