
[🇬🇧 English](README.md) | [🇩🇪 Deutsch](README.de.md) | [🇳🇱 Nederlands](README.nl.md)

# DCC Command Station LZ210-TMC

Modulair DCC command station met Ethernet- en USB-aansluiting, en (gedeeltelijke) ondersteuning voor het XpressNet- en Z21-protocol. Het board ondersteunt LocoNet-T- en XpressNet-handhelds en overige componenten, biedt een interface voor externe boosters via CDE en LocoNet-B, en heeft een aansluiting voor RS-Bus-terugmeldmodules.

![DCC Command Station](docs/images/DCC-CommandStation-LZ210-TMC.png)


## Waarom dit board
De [Twentse Modelspoorweg Club (TMC)](https://twentsemodelspoorweg.club) is voor de digitale baanbesturing haar electronika en software aan het moderniseren. Voor ieder van haar stations (Hengelo, Enschede, Oldenzaal, Almelo) heeft het een eigen, nieuwe DCC centrale nodig om alle wissels en seinen te bedienen, en bezetmeldingen te ontvangen. Alhoewel DCC centrales natuurlijk kant-en-klaar te koop zijn, is zelfbouw niet alleen een mooie uitdaging, maar ook een potentiële kostenbesparing.

Het uitgangspunt is een modulair ontwerp, zoals de [Z21PG](https://pgahtow.de/w/Zentrale_Z21PG/de). In eerste instantie is gekeken of we deze centrale niet gewoon konden aanpassen of nabouwen. Al snel kwamen we tot het inzicht dat een upgrade naar modernere processoren en modules tot betere resultaten en een lagere prijs zou leiden.

Het doel is een centrale te ontwikkelen die stapsgewijs kan worden uitgebreid en **door clubleden en andere geïnteresseerden makkelijk kan worden aangepast en nagebouwd**. Daarom is uitdrukkelijk gekozen voor een makkelijk te solderen en goedkope THT (Through-Hole Technology) printplaat, en wat betreft de cruciale componenten kant-en-klare modules die eenvoudig via het Internet gekocht kunnen worden.

Als processor hebben we gekozen voor de RP2350B, vooral vanwege de PIO-peripheral, waarmee een volledig jittervrij DCC-signaal kan worden gegenereerd. Het betreft hier een goedkope en moderne dual core processor, die op 125 Mhz draait. Zie [processorkeuze](docs/processor.nl.md) voor een vergelijking met ESP32, DxCore en STM32.

Als naam is gekozen voor LZ210-TMC, wat een samentrekking is van Z21 (Roco, Z21-protocol, LocoNet) en LZ100 (Lenz, XpressNet, RS-Bus interface en CDE booster aansluiting).


## Schema
Het volledige schema (alle hiërarchische sheets) staat in [Schematic.pdf](docs/Schematic.pdf).

Het schema is opgedeeld in de volgende functionele blokken (klik op de naam voor meer details):

<details>
<summary><strong>Voeding</strong></summary>

De centrale dient gevoed te worden met een gelijkspanning die gelijk is aan de gewenste railspanning. In de praktijk kan een geschakelde of laptop voeding gebruikt worden die een spanning kan leveren tussen de 14 en 20 volt, en 3 tot 4 ampère.

⚠️ **Let op:** niet elke geschakelde voeding is geschikt of veilig voor deze toepassing. Een praktisch advies: kies een voeding zoals gebruikt bij andere DCC-centrales, bijvoorbeeld de [Roco 10851](https://www.roco.cc/rde/10851-schaltnetzteil-54-watt.html) of [YaMoRC YD7460-18](https://yamorc.de/products/?singleproduct=13843).

De externe voedingsspanning komt binnen via een 3 ampère eFuse en een zogeheten "ideale diode", die bij verkeerde polariteit de schakeling beschermt, en loopt naar VCC en een tweetal step-down converter modules.

VCC voedt de H-Bruggen voor de rail spanning en het programmeerspoor. VCC loopt eveneens richting processor, zodat via een weerstandsdeler en een ADC-ingang gemeten kan worden of de externe voedingsspanning de juiste waarde heeft.

De 12V step-down converter levert de spanning voor LocoNet, XpressNet, het CDE booster interface en de RS-Bus terugmelding. Ook de 12V spanning wordt, ter controle, via een weerstandsdeler doorgestuurd naar een ADC-ingang.

De keuze voor de 5V step-down converter behoeft enige toelichting, omdat de processor en peripherals juist 3V3 verwachten. De reden dat voor 5V is gekozen, ligt in het feit dat op de Olimex RP2350 Pico2-XL module al een eigen step-down chip is gemonteerd. Op de 3V3 uitgang van deze specifieke chip mag *geen* externe 3V3 converter worden aangesloten. De uitgang van deze chip levert wel voldoende vermogen om de rest van de peripherals met 3V3 te voeden. De diode tussen de 5V step-down converter en de Olimex RP2350 Pico2-XL module voorkomt dat er schade ontstaat als er, naast de externe voeding, tevens gevoed wordt via de USB aansluiting.  


><details>
><summary>Uitleg: waarom geen externe 3V3 op de Olimex-module?</summary>
>
> De TPS62A02A chip op de Olimex-module heeft een actieve output-discharge: zodra de TPS62A02A wordt uitgeschakeld (bijvoorbeeld door 3V3_EN naar GND te leggen), wordt een interne FET ingeschakeld om de energie die nog in de externe spoel is opgeslagen actief te ontladen. Als in die toestand toch 3,3V op de uitgang van de TPS62A02A wordt gezet, dan blijft de interne FET continu geleiden en tientallen mA uit de externe voeding trekken. De SOT-563 behuizing kan hierdoor oververhit raken. Dit gedrag wordt ook beschreven op [TI's E2E-forum](https://e2e.ti.com/support/power-management-group/power-management/f/power-management-forum/1305135/tps62a01-active-output-discharge).
>
> Vandaar de keuze om niet de 3V3-uitgang, maar de VSYS ingang te voeden. De onboard TPS62A02A blijft hierdoor altijd actief, ongeacht of via de 5V step-down converter of via USB wordt gevoed. De TPS62A02A kan voldoende vermogen leveren om ook de overige peripherals van stroom te voorzien.
>
> Dit gedrag is anders dan bij de originele Raspberry Pi Pico2: die gebruikt de RT6150B als step-down, welke deze actieve output-discharge niet kent. Bij de Pico2 is 3V3_EN naar GND leggen en vervolgens extern 3,3V op de uitgang zetten dan ook juist de door Raspberry Pi zelf gedocumenteerde methode. Voor andere RP2350B-modules geldt: check eerst welke step-down converter erop zit, want het gedrag kan per board verschillen.
>
> </details>

</details>


<details>
<summary><strong>Processor</strong></summary>

Als processor is gekozen voor een Raspberry Pi Pico processor; zie [processorkeuze](docs/processor.nl.md) voor details. Een belangrijk pluspunt van de Raspberry Pi Pico is dat het een Programmable IO (PIO) peripheral bezit, wat een gespecialiseerde coprocessor is waarmee zeer nauwkeurig getakte uitgangssignalen kunnen worden gegenereerd.

Binnen de Pico serie zijn er twee varianten: de RP2040, met twee PIO's, en de RP2350, met drie PIO's. Omdat we drie PIO's gebruiken, is gekozen voor de RP2350. De eerste PIO wordt gebruikt voor het DCC signaal, de tweede voor de RS-Bus master, en de derde voor de XpressNet RS-485 aansluiting.

De RP2350 kent eveneens twee varianten: de "gewone" RP2350, die 30 General Purpose IO (GPIO) heeft, en de RP2350B, die er 18 extra heeft. Voor een eerste test versie van de centrale is de "gewone" RP2350 gebruikt, maar omdat we meer IO pinnen wilden gebruiken is uiteindelijk voor de RP2350B gekozen.

Er worden door meerdere leveranciers (zoals Olimex, Waveshare en WeAct) verschillende RP2350B modules aangeboden. Gekozen is voor de RP2350B-X(X)L module van Olimex; zie voor de motivatie hieronder bij Nabouwen.

><details>
><summary>Uitleg: Zijn er alternatieve modules mogelijk?</summary>
>
> Als er minder pinnen en minder PIO's worden benodigd, bijvoorbeeld omdat er geen behoefte is aan een RS-Bus master of XpressNet, kan ook gekozen worden voor RP2040 modules. In dat geval moet wel de print layout en eventueel de voeding worden aangepast.  
>
> </details>

><details>
><summary>Uitleg: waarom een PIO voor XpressNet?</summary>
>
> Bij de meeste processoren is het mogelijk een UART te gebruiken voor het versturen en ontvangen van XpressNet berichten. Dat kan, omdat de UART op veel processoren zowel 8 als ook 9 databits ondersteunt. XpressNet vereist echter 9 databits (waarbij het negende bit aangeeft of het een adres of data byte betreft). In vergelijking met andere processoren is de UART op de Raspberry Pi Pico relatief beperkt en ondersteunt alleen 8 databits. Door een klein PIO programmaatje te maken is het echter mogelijk om ook op de Raspberry Pi Pico  met 9 databits te werken, en dus XpressNet te ondersteunen.
>
> Eventueel is het met enige trucs toch mogelijk om de UART op de Raspberry Pi Pico te gebruiken voor XpressNet. Omdat er echter maar twee UARTs zijn, is daarvan afgezien.  
>
> </details>

</details>

<details>
<summary><strong>Ethernet</strong></summary>

Het Ethernet schema bevat enkel de WIZnet W5500 module. Alle pinnen zijn doorverbonden met de processor.

De oudere (en vaak duurdere) W5100 en W5100S modules werken waarschijnlijk ook, maar dat is niet getest.

</details>

<details>
<summary><strong>Rail uitgang</strong></summary>

Het DCC signaal wordt versterkt door een DRV8874 module, met daarop eenzelfde driver-IC die ook in het DCC-EX-project wordt gebruikt.

Het schema bevat weinig onderdelen omdat de module zelf al een aantal weerstanden bevat waardoor voor onze centrale bruikbare instellingen ontstaan.

Standaard staat I<sub>TRIP</sub> ingesteld op ongeveer 3 ampère. Indien gewenst kan een andere waarde worden gekozen door de 2,49 kΩ weerstand op de module te verwijderen en R601 te monteren met een andere waarde.

Standaard is **PMODE** verbonden met 3V3, waardoor PWM-mode is geactiveerd. Indien gewenst kan de "PWM (PH / EN) solder jumper" aan de onderkant van de print worden gebruikt om **PMODE** te verbinden met GND. Verwijder hiertoe de bestaande koperverbinding naar 3V3, zodat er geen kortsluiting ontstaat, en maak de andere soldeerverbinding naar GND.

![CD booster ingang](docs/images/PCB-Bottom-Jumpers.png)

Standaard is de module ingesteld op Quad-Level 2, maar dat kan worden aangepast door een tweede "solder jumper" aan de onderkant van de print **IMODE** te verbinden met GND. Zie het DRV8874 datasheet voor de verschillen tussen beide modes.

Op de print is geen ruimte gereserveerd voor een snubberschakeling en/of power inductor. Indien gewenst kunnen die na de connector op de print in de bedrading naar de rails worden geplaatst.

Gedetailleerde achtergrond informatie omtrent de keuze en de beperkingen van dit driver IC zijn beschreven op een aparte pagina. Op die pagina wordt uitgebreid ingegaan op de problemen met inschakelstromen die deze en vergelijkbare chips kunnen hebben bij capacitieve belastingen. Zie: [DRV8874 eindtrap](docs/DRV8874.nl.md)

</details>

<details>
<summary><strong>Programmeerspoor</strong></summary>

Op de print kan gekozen worden voor dezelfde DRV8874 H-brug module als voor de normale rail uitgang. Het voordeel van het hergebruiken van modules is een uniforme onderdelenlijst en een eenvoudige nabouw.

Het is echter beter om, in plaats van een DRV8874 module, een DRV8876 module te nemen. Deze modules kunnen wat minder vermogen leveren, wat voor een programmeerspoor juist een voordeel is, en ze zijn bovendien wat goedkoper.

Een belangrijk technisch verschil tussen de DRV8874 en DRV8876 modules is dat de laatste een hogere waarde heeft voor de stroomspiegel-versterkingsfactor **AIPRPOPI**, namelijk 1000 (in plaats van 450). Als gevolg hiervan is de spanning die de ADC-ingang van de processor ziet bij een (60 mA) DCC-ACK signaal 150 mV (in plaats van 67 mV). Hierdoor is een betrouwbaardere meting van het DCC-ACK signaal mogelijk.

Net als bij de normale rail uitgang is het mogelijk de 2,49 kΩ weerstand die standaard op de module is gemonteerd te verwijderen en te vervangen door R801. Een goede waarde voor R801 is 6,8 kΩ. Met deze waarde wordt de maximale stroom I<sub>TRIP</sub> die tijdens programmeren kan worden geleverd beperkt tot 500 mA. RCN-216 adviseert bij het inschakelen een (niet-verplichte) begrenzing tot 250 mA ±20%, maar staat voor vast ingestelde stroombegrenzers ook waarden tot 1 A toe; 500 mA past daar goed binnen.

Een bijkomend voordeel van het vervangen van de 2,49 kΩ weerstand door een 6,8 kΩ weerstand, is dat de spanning tijdens een DCC-ACK verhoogd wordt tot ruim 400 mV, dus beter te meten op de ADC-ingang van de processor.

</details>

<details>
<summary><strong>CDE booster interface</strong></summary>

Op de print is gekozen voor eenzelfde type H-brug module als voor het programmeer spoor (DRV8876 of DRV8874). Technisch gezien kan ook gekozen worden voor een DRV8871-module van AliExpress. De afmetingen van deze module passen echter niet op de huidige print.

Voor de CDE-uitgang gelden de bekende problemen met inschakelstromen van de DRV887x ICs niet. De belasting is niet capacitief en voor het CD-signaal is niet meer nodig dan een paar honderd milliampères. De interne kortsluitbeveiliging van de DRV887x springt niet aan tijdens het opstarten, tenzij de uitgang echt is kortgesloten. De DRV887x is weliswaar overgedimensioneerd maar kan in principe probleemloos worden toegepast.

Gedetailleerde informatie betreffende het CDE interface is beschreven op een aparte pagina: [CDE Interface](docs/CDE.nl.md).

</details>

<details>
<summary><strong>LocoNet</strong></summary>

Het schema voor het LocoNet deel is standaard, en op veel plaatsen op het internet beschreven. Wel zijn de waarden van een aantal onderdelen aangepast zodat het geheel ook op 3V3 werkt.

Naast de LocoNet-T interface, is ook een LocoNet-B interface aanwezig, die op de buitenste pinnen van de connector een RailSync signaal voor boosters levert. Voor het genereren van het RailSync signaal kan gekozen worden uit meerdere driver ICs; in het schema worden een aantal opties genoemd. Van de genoemde opties heeft alleen het UCC27425 IC een ENABLE ingang, waardoor het RailSync-signaal eenvoudig kan worden uitgeschakeld.

Op de uitgangen van de driver zijn 33 Ω / 2 Watt weerstanden opgenomen. Enerzijds dienen deze weerstanden als kortsluitbeveiliging (daarom moeten ze 2 Watt kunnen dissiperen). Anderzijds vormen deze weerstanden, samen met de 4,7 nF condensatoren, een laagdoorlaat filter, waardoor storingen verminderd worden. De P6KE15A zijn ESD diodes die het LocoNet driver IC beschermen.  

</details>

<details>
<summary><strong>XpressNet</strong></summary>

Het XpressNet interface is eenvoudig en standaard. Op de [OpenDCC site](https://www.opendcc.de/elektronik/opendcc/xpressnet_hw.html) is een basisschakeling te vinden, die door velen is nagebouwd. De enige aanpassing is dat de 1K5 weerstanden vervangen zijn door 560 Ω, omdat het driver IC niet met 5 V, maar 3V3 gevoed wordt. De 8K2 weerstand (R504) zorgt ervoor dat de driver start in RX-mode.

Vaak wordt als driver een MAX(3)485 IC gebruikt. Een betere keus is de MAX(3)483, dus de Slew Rate Limited versie van deze driver, omdat deze minder gevoelig is voor hoogfrequente storingen op de RS485-lijn. Helaas is dit IC duidelijk duurder. Bovengenoemde OpenDCC pagina geeft verdere details over mogelijke alternative driver ICs.   

</details>

<details>
<summary><strong>RS-Bus terugmelding</strong></summary>

Het schema voor de RS-Bus is overgenomen van de [Der-Moba website](https://www.der-moba.de/index.php/RS-Rückmeldebus).
De daarin vermelde transistor voor zenden is echter vervangen door een (IRLZ44) MOSFET, waardoor een wat hogere belasting mogelijk wordt.

</details>


## Nabouwen
Om de centrale na te bouwen, is het makkelijkst de huidige print te laten namaken door de file [production/TMC-Centrale-V2.1.zip](production/TMC-Centrale-V2.1.zip) te sturen naar een bedrijf zoals JLCPCB. Deze file bevat alle (Gerber) bestanden die de fabrikant nodig heeft. In de zomer van 2026 koste het laten maken van 5 printplaten inclusief verzending ongeveer €25.

Het ontwerp is open source. Nabouwen, aanpassen en verspreiden wordt aangemoedigd onder de voorwaarden van de [licentie](LICENSE), en met referentie naar deze bron.

De complete lijst met componenten is: [production/bom.csv](production/bom.csv).

Om het nabouwen zo eenvoudig mogelijk te maken, is qua hardware zo veel mogelijk gekozen voor kant en klare modules, zoals die op het Internet door AliExpress en anderen verkocht worden. Hieronder een overzicht van de belangrijkste modules, alsmede de connectoren zoals die op de print passen:


<details>
<summary><strong>RP2350B (MCU-module)</strong></summary>

Als processor is gekozen voor de [Olimex RP2350B-XL](https://www.olimex.com/Products/RaspberryPi/PICO/PICO2-XXL/open-source-hardware) module. Olimex garandeert dat de productie van deze modules tot minstens 2035 wordt ondersteund en heeft alle KiCad-ontwerpbestanden en productieschema's openbaar op GitHub gezet.

Er zijn twee varianten van deze module: de RP2350B-XL (€5) en de RP2350B-XXL (€9). Het verschil is dat de XXL wat extra mogelijkheden heeft die voor deze centrale niet nodig zijn. Alhoewel in het schema de XXL wordt genoemd, heeft de XL variant vanwege de prijs de voorkeur. Beide varianten passen echter op de print.

De modules kunnen bij [Olimex](https://www.olimex.com/Products/RaspberryPi/PICO/PICO2-XXL/open-source-hardware) zelf gekocht worden, maar ook via distributeurs zoals [TME](https://www.tme.eu).

</details>

<details>
<summary><strong>DRV8874 / DRV8876 (motordrivers)</strong></summary>

Voor DCC, programmeerspoor, en CDE uitgang wordt gebruik gemaakt van kant-en-klare DRV8874 / DRV8876 modules.

Pololu heeft enige jaren geleden modules voor de [DRV8874](https://www.pololu.com/product/4035) en [DRV8876](https://www.pololu.com/product/4037) op de markt gebracht. Sinds het voorjaar 2026 zijn (goedkopere) varianten ook verkrijgbaar bij AliExpress. De Pololu-module bevat als extra beveiliging een polariteitsbewakende MOSFET, die de AliExpress-modules niet hebben. Deze extra beveiliging is voor deze centrale niet nodig.

![AliExpress DRV8876](docs/images2/DRV8876.png)

</details>

<details>
<summary><strong>Step-down module (5V/12V)</strong></summary>

De afmetingen op de print zijn afgestemd op de bekende MP1584 Step-Down modules, die door meerdere leveranciers op AliExpress worden aangeboden. Er zijn twee modules nodig. De eerste levert de 5 V voedingsspanning voor de processor (merk op: de Olimex processor module zet deze 5 V zelf weer om naar 3V3) en de tweede module levert 12 V. In plaats van modules met vaste 5 / 12 V uitgangsspanning kunnen ook modules gebruikt worden waarbij de uitgangsspanning instelbaar is.

Andere modules dan de MP1584 werken in principe ook, maar passen niet op de printplaat.

![AliExpress MP1584](docs/images2/MP1584.png)

</details>

<details>
<summary><strong>Ideal-diode module</strong></summary>

Als beveiliging voor de juiste polariteit wordt een XL74610 module gebruikt, zoals die bij AliExpress en andere leveranciers te koop is.

![AliExpress XL74610](docs/images2/XL74610.png)

</details>

<details>
<summary><strong>W5500 Ethernet</strong></summary>

Voor de Ethernet verbinding wordt een W5500 module gebruikt, zoals die bij AliExpress en andere leveranciers te koop is.

![AliExpress W5500](docs/images2/W5500.png)

</details>

<details>
<summary><strong>Connectoren</strong></summary>

De print is gemaakt voor de volgende connectoren

><details>
><summary>Loconet: 6P6C (RJ12)</summary>
>
> Er zijn verschillende merken en types 6P6C connectoren op de markt, met verschillende footprints. De printplaat is gemaakt voor connectoren met een footprint gelijk aan de WayConn MJEA connectoren. Deze hebben de elektrische aansluitingen aan de onderkant (printplaat).
> ![AliExpress 6P6C](docs/images2/6P6C.png)
>
> </details>
><details>
><summary>Barrel Jack</summary>
>
> Voor de voedingsconnector is een 5.5x2.1mm barrel jack gebruikt.
> ![AliExpress BarrelJack](docs/images2/BarrelJack.png)
>
> </details>
><details>
><summary>Terminal Blocks</summary>
>
> Voor de DCC railuitgang is een PhoenixContact MSTBA 2,5/2-G genomen, met een pinafstand van 5.00 mm. Terminal blocks met een pinafstand van 5.08 mm passen echter ook.
>
> Voor de programmeeruitgang is een Phoenix Contact MC 1.5/ 2-G-3.81 terminal block gebruikt.
>
> Voor de CDE-uitgang is een Phoenix Contact MC 1.5/ 3-G-3.81 terminal block gebruikt.
>
> Voor de RS-Bus-ingang is een Phoenix Contact MC 1.5/ 2-G-3.81 terminal block gebruikt.
>
> Voor de XpressNet-uitgang is een Phoenix Contact MC 1.5/ 4-G-3.81 terminal block gebruikt. Aan de voorkant is een 5-polige DIN connector gemonteerd.
>
> Varianten van deze connectoren worden door meerdere fabrikanten aangeboden, waaronder PTR/Hartmann.
>
> ![AliExpress DIN](docs/images2/DIN.png)
>
> </details>

</details>


## Software

De firmware voor deze centrale is te vinden in een eigen GitHub repository, en wordt hier dan ook niet verder besproken (link - **TODO**)


## Vergelijking met andere centrales

Deze centrale staat niet op zichzelf, maar staat in een lange traditie van open-source DCC-centrales:

<details>
<summary><strong>OpenDCC</strong></summary>

Een van de eerste open-source DCC-centrales is de [OpenDCC Z1](https://www.opendcc.de/elektronik/opendcc/opendcc.html). Deze centrale is twintig jaar geleden ontworpen door Wolfgang Kufer.
- **Processor.** OpenDCC draait op een 8-bit Atmel AVR (ATmega32 of ATmega644P) op 16 MHz. Deze centrale gebruikt een dual-core RP2350 op 125 MHz met een PIO-peripheral, en is daarmee ordes van grootte krachtiger.
- **Eindtrap.** OpenDCC gebruikt een vaste H-brug (STM L6206), theoretisch geschikt voor 2,8 A per uitgang, maar thermisch op de print beperkt tot ongeveer 1,5 A continu. Deze centrale gebruikt een DRV8874-module (6 A theoretisch), waarvan de stroom op de module zelf wordt beperkt tot ongeveer 2,9 A.
- **Terugmelding.** OpenDCC ondersteunt S88, met een uitbreiding voor wisselstand-terugmelding. Deze centrale gebruikt RS-Bus en LocoNet.
- **Aansluiting op een PC.** OpenDCC communiceert via RS232/USB met een PC, en als hogere laag protocol XpressNet of P50X; een netwerk- of WiFi-verbinding kent OpenDCC zelf niet. Deze centrale heeft, naast USB, een eigen Ethernet-aansluiting, en ondersteunt als hogere laag protocol XpressNet en Z21.
- **Boosters.** OpenDCC heeft geen apart interface voor externe boosters, zoals CDE of LocoNet-B.
- **Handhelds.** Net als deze centrale ondersteunt OpenDCC XpressNet-handhelds (zoals de Roco Multimaus). OpenDCC ondersteunt echter geen LocoNet handhelds.

De OpenDCC Z1 centrale kan gezien worden als voorloper van open source DCC centrales. Veel van de ideeën, schakelingen en software zijn (in aangepaste vorm) door anderen gekopieerd en worden nog steeds toegepast. De Z1 nu nog nabouwen lijkt echter geen goed idee: moderne microcontrollers zijn goedkoper én vele malen krachtiger dan de 8-bit ATmega-generatie waarop de OpenDCC Z1 destijds is gebaseerd.
</details>


<details>
<summary><strong>BiDiB</strong></summary>

[BiDiB](https://www.bidib.org/) kan gezien worden als een vervolgstap op de OpenDCC Z1 centrale. BiDiB is vanaf 2010 ontwikkeld door — opnieuw — Wolfgang Kufer, en wordt onder meer door Fichtelbahn in hardware uitgebracht.
- **Architectuur.** BiDiB is primair een bus waarop meerdere intelligente componenten (zoals GBMboost/GBM16T/OpenSwitch/IF2) kunnen worden aangesloten. Het is geen centrale in de traditionele zin, met een interne booster en interfaces zoals XpressNet, Loconet en de RS-Bus.
- **Processor.** BiDiB implementaties zijn gebaseerd op de Atmel XMEGA processor. Alhoewel dit één van de krachtigste 8/16-bit / 32 Mhz processoren is, is het ook een vrij oude processor en, in vergelijking met de dual-core RP2350 / 125 MHz processor van deze centrale, behoorlijk prijzig.
- **Zelfbouw.** De BiDiB prints zijn ontworpen voor SMD componenten. Alhoewel de BiDiB ontwerpen geavanceerder en van hogere kwaliteit zijn dan die van deze (THT) centrale, zijn ze ook moeilijker na te bouwen, aan te passen en uit te breiden.
- **RailCom.** BiDiB excelleert in het ontvangen, analyseren en verwerken van RailCom terugmeldingen. Deze centrale biedt hiervoor geen ondersteuning.

BiDiB biedt een compleet ecosysteem met een focus op intelligente componenten en uitgebreide RailCom-verwerking. Deze centrale is vooral een traditionele centrale zoals die van Roco en Lenz, die via gevestigde protocollen met apparatuur van verschillende fabrikanten communiceert.

</details>


<details>
<summary><strong>Z21PG</strong></summary>

[Z21PG](https://pgahtow.de/w/Zentrale_Z21PG/) is een Arduino-gebaseerd zelfbouwproject van Philipp Gahtow voor een goedkope Roco Z21 clone.
- **Eenvoud om na te bouwen.** Philipp Gahtow publiceert zelf vooral software (Arduino-sketches) en losse schema's voor een aantal hardwarevarianten (Arduino MEGA, ESP32, ...). In tegenstelling tot deze centrale, bestaat er geen officieel Z21PG (KiCad) printontwerp. Voor de Z21PG bestaan wel een aantal door derden gemaakte printontwerpen. Beide centrales zijn makkelijk na te bouwen.
- **Processor en DCC-signaal.** De meest nagebouwde Z21PG variant draait op een (verouderde) ATmega2560 (16 MHz), alhoewel de ESP32 ook als mogelijkheid wordt genoemd. Het DCC-signaal wordt via Timer-interrupts in software gegenereerd, niet via een gespecialiseerde hardware-peripheral. Deze centrale gebruikt een (moderne) dual-core RP2350 op 125 MHz met een PIO-peripheral, meer dan tien keer zo snel en met een jittervrij DCC-signaal.
- **Interfaces.** De Z21PG biedt vergelijkbare aansluitingen als de Roco Z21 centrale: LAN, X-Bus (XpressNet), L-Bus (Loconet-T), Booster aansluiting, rail en programmeeruitgang. Deze centrale heeft daarnaast aansluitingen voor de RS-Bus en de booster aansluitingen zijn compatibel met CDE en LocoNet-B.

Van alle hier besproken projecten heeft de Z21PG de meeste overeenkomsten met deze centrale. Beide zijn modulair opgebouwd en relatief makkelijk voor weinig geld na te bouwen. De Z21PG heeft primair de Roco Z21 als voorbeeld, deze centrale heeft daarnaast ook de Lenz centrales als voorbeeld. De Z21PG is een wat ouder ontwerp; deze centrale maakt gebruik van modernere componenten.
</details>


<details>
<summary><strong>DCC-EX</strong></summary>

Een populaire zelfbouw centrale is momenteel [DCC-EX](https://dcc-ex.com/). Er zijn functionele overeenkomsten (onder andere de DRV8874-eindtrap), maar ook een aantal wezenlijke verschillen:
- **Verkrijgbaarheid.** DCC-EX biedt zowel fabrieksmatig geassembleerde boards (EX-CSB1, vanaf ca. €75–115 excl. btw/verzending) als doe-het-zelf-opbouw op Arduino-vormfactor boards. Deze centrale is bedoeld voor zelfbouw, en de onderdelenkosten zijn lager (ca. €40).
- **Aansluiting op een PC/netwerk.** Deze centrale gebruikt Ethernet (XpressNet en Z21); DCC-EX WiFi (met JMRI-, WiThrottle- en Engine Driver-ondersteuning).
- **Interfaces voor handhelds.** Deze centrale heeft native (RS485) XpressNet- en LocoNet-interfaces, waarop Lenz, Roco en andere handhelds aangesloten kunnen worden. DCC-EX richt zich primair op WiFi-throttles en JMRI.
- **Interfaces voor terugmelding.** Deze centrale heeft native RS-Bus en LocoNet-interfaces. DCC-EX heeft dergelijke interfaces niet.
- **Booster interfaces.** Deze centrale ondersteunt externe boosters via LocoNet-B en het CDE-interface. DCC-EX heeft geen CDE/LocoNet-B-ondersteuning, maar kent met RailSync wel een mechanisme waarmee externe boosters kunnen worden aangesloten.
- **Gemengd DCC/DC-bedrijf.** DCC-EX kan per uitgang tussen DCC en DC PWM schakelen, zodat ook klassieke DC-locomotieven kunnen rijden. Deze centrale ondersteunt alleen DCC.

Waar deze centrale zich het duidelijkst onderscheidt, is het ecosysteem waarin hij past. DCC-EX is sterk gericht op de Amerikaanse hobbyscene: WiFi-throttles, JMRI, en voor accessoires het LCN (Layout Control Node)-concept, waarbij goedkope Arduino Nano's met nRF24L01-radiomodules wissels, seinen en verlichting draadloos aansturen. Deze centrale is daarentegen meer gericht op traditioneel Europese gebruikers: XpressNet- en LocoNet-handhelds, CDE/LocoNet-B-boosters en RS-Bus-terugmelding. Voor clubs die al met Lenz-, Roco- of vergelijkbare apparatuur werken, is dat een directer aansluitpunt dan de WiFi/JMRI/LCN-route van DCC-EX.
</details>


<details>
<summary><strong>OpenRemise</strong></summary>

[OpenRemise](https://github.com/OpenRemise) is nadrukkelijk gericht op **gebruiksgemak zonder soldeer- of elektronicakennis**: het systeem is vrijwel plug-and-play en wordt na installatie volledig bediend via een webinterface op smartphone, tablet of PC. Daarmee staat het qua filosofie bijna tegenover deze centrale.
- **Hardware.** OpenRemise richt zich vooral op gebruikers die een kant-en-klaar print willen; de print bevat vooral SMD componenten en is niet makkelijk met de hand te solderen. Deze centrale is vooral bedoeld voor mensen die zelf willen solderen en eventueel aanpassen.
- **Processor.** OpenRemise is gebaseerd op een moderne en krachtige processor (ESP32), net als deze centrale (RP2350). Voor jittervrije DCC signaalgeneratie gebruikt OpenRemise een speciale peripheral (RMT), net als deze centrale (PIO).
- **Eindtrap.** OpenRemise heeft een op losse MOSFETs gebaseerde eindtrap, die superieur is aan de DRV8874 eindtrap die in deze centrale wordt gebruikt.
- **Doelgroep.** OpenRemise excelleert in het bijwerken van decoderfirmware, maar kan ook als complete centrale worden gebruikt. Deze centrale is meer bedoeld als traditionele DCC centrale, en heeft meer standaard interfaces (XpressNet, LocoNet, CDE).
- **Netwerk.** OpenRemise ondersteunt WIFI en richt zich op web-bediening. Deze centrale ondersteunt Ethernet en richt zich op PC besturing via het XpressNet / Z21 protocol.

OpenRemise is vooral een kant-en-klaar SMD print ontwerp met bijbehorende firmware. Deze centrale richt zich vooral op hobbyisten die een goedkope centrale willen die makkelijk is na te bouwen en aan te passen. Deze centrale heeft daarom een aantal compromissen moeten sluiten (zoals de DCC eindtrap) die OpenRemise niet heeft hoeven maken.  
</details>


## Status en toekomst
De centrale is sinds zomer 2026 in gebruik bij de TMC voor de aansturing van wissels, seinen, en voor bezetmeldingen. Alhoewel de meeste functies zijn getest en goed bevonden, is er geen garantie dat alles werkt zoals zou moeten.

Er zijn plannen om van de centrale ook een SMD-versie te maken, alhoewel die plannen niet definitief zijn.

Op de printplaat is plaats voor een extensie connector, met daarop onder andere SPI en I2C. Op een extensie print kunnen additionele functies worden geïmplementeerd, zoals een display, RailCom terugmelding, S88n of de CAN-bus.

In principe moet het ook mogelijk zijn de centrale van een WIFI interface te voorzien. Omdat Ethernet echter betrouwbaarder is, hebben wij van WIFI afgezien.

## Referenties

<details>
<summary><strong>Details van dit ontwerp</strong></summary>

- [Processorkeuze](docs/processor.nl.md) — vergelijking RP2350 met ESP32, DxCore en STM32
- [DRV8874 eindtrap](docs/DRV8874.nl.md) — achtergrond bij de gekozen driver-IC en inschakelstroomproblematiek
- [CDE interface](docs/CDE.nl.md) — achtergrond bij de externe boosterinterface

</details>

<details>
<summary><strong>Protocollen en schakelingen</strong></summary>

- [RailCommunity (RCN)](https://railcommunity.de) — DCC-standaarden
- [XpressNet Specificatie](https://www.lenz-elektronik.de/media/0a/d1/4e/1760451943/XpressNet_V40_2.pdf?ts=1760451943) — XpressNet Versie 4.0 (Lenz)
- [XpressNet schakeling](https://www.opendcc.de/elektronik/opendcc/xpressnet_hw.html) — Basisschakeling (OpenDCC)
- [LocoNet](https://www.digitrax.com/static/apps/cms/media/documents/tech_notes/loconet-personal-edition-1-0.pdf) — Personal Use Edition (Digitrax)
- [RS-Bus](https://www.der-moba.de/index.php/RS-Rückmeldebus) — Basisschakeling (Der Moba)
- [Paco's Official Web Site](https://usuaris.tinet.cat/fmco/home_en.htm) — Basisschakelingen voor XpressNet, LocoNet, CDE en boosters

</details>

<details>
<summary><strong>Commerciële centrales en vergelijkbare zelfbouwprojecten</strong></summary>

- [Lenz](https://www.digital-plus.de/) — LZV100/LZV200 centrales (XpressNet, CDE en RS-Bus)
- [Roco/Fleischmann](https://www.z21.eu/) — Z21-centrale
- [OpenDCC](https://www.opendcc.de/)
- [BiDiB](https://www.bidib.org/)
- [Z21PG](https://pgahtow.de/w/Zentrale_Z21PG/)
- [DCC-EX](https://dcc-ex.com/)
- [OpenRemise](https://github.com/OpenRemise)

</details>

<details>
<summary><strong>Overig</strong></summary>

- [Olimex RP2350B-XL/XXL](https://www.olimex.com/Products/RaspberryPi/PICO/PICO2-XXL/open-source-hardware) — MCU-module

</details>
