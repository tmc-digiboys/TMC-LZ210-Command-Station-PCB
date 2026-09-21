
[🇬🇧 English](README.md) | [🇩🇪 Deutsch](README.de.md) | [🇳🇱 Nederlands](README.nl.md)

# DCC Command Station LZ210-TMC

Diese modulare DCC-Zentrale verfügt über einen Ethernet- und einen USB-Anschluss sowie eine (teilweise) Unterstützung von XpressNet und Z21. Die Platine bietet Schnittstellen für LocoNet-T- und XpressNet-Handregler sowie weitere Geräte und unterstützt CDE und LocoNet-B. Über den RS-Bus können Rückmeldemodule angeschlossen werden.

![DCC Command Station](docs/images/DCC-CommandStation-LZ210-TMC.png)


## Warum diese Platine
Der [Twentse Modelspoorweg Club (TMC)](https://twentsemodelspoorweg.club) modernisiert seine Elektronik und Software für die digitale Anlagensteuerung. Für jeden seiner Bahnhöfe (Hengelo, Enschede, Oldenzaal, Almelo) wird eine eigene, neue DCC-Zentrale benötigt, um sämtliche Weichen und Signale zu steuern und Rückmeldungen zur Gleisbesetzung zu empfangen. Obwohl fertige DCC-Zentralen selbstverständlich zu kaufen sind, ist der Selbstbau nicht nur eine schöne Herausforderung, sondern auch eine mögliche Kostenersparnis.

Ausgangspunkt ist ein modularer Entwurf, ähnlich der [Z21PG](https://pgahtow.de/w/Zentrale_Z21PG/de). Zunächst wurde geprüft, ob sich dieser Entwurf einfach anpassen oder nachbauen ließe. Schnell zeigte sich jedoch, dass ein Umstieg auf einen moderneren Prozessor und fertige Module zu besseren Ergebnissen und einem niedrigeren Preis führen würde.

Ziel ist die Entwicklung einer Zentrale, die schrittweise erweitert werden kann und die **Clubmitglieder sowie andere Interessierte einfach anpassen und nachbauen können**. Deshalb wurde bewusst eine leicht zu lötende, preiswerte THT-Platine (Through-Hole Technology) gewählt, und bei den kritischen Bauteilen wurde auf fertige Module gesetzt, die problemlos im Internet erhältlich sind.

Als Prozessor wurde der RP2350B gewählt, vor allem wegen seines PIO-Peripheriegeräts, mit dem sich ein vollständig jitterfreies DCC-Signal erzeugen lässt. Es handelt sich um einen preiswerten und modernen Dual-Core-Prozessor, der mit 125 MHz getaktet ist. Siehe [Prozessorwahl](docs/processor.de.md) für einen Vergleich mit ESP32, DxCore und STM32.

Als Name wurde LZ210-TMC gewählt, eine Zusammenziehung aus Z21 (Roco, Z21-Protokoll, LocoNet) und LZ100 (Lenz, XpressNet, RS-Bus-Schnittstelle und CDE-Boosteranschluss).


## Schaltplan
Der vollständige Schaltplan (alle hierarchischen Sheets) befindet sich in [Schematic.pdf](docs/Schematic.pdf).

Der Schaltplan ist in folgende Funktionsblöcke unterteilt (auf den Namen klicken für weitere Details):

<details>
<summary><strong>Spannungsversorgung</strong></summary>

Die Zentrale muss mit einer Gleichspannung versorgt werden, die der gewünschten Gleisspannung entspricht. In der Praxis kann ein Schaltnetzteil oder ein Laptop-Netzteil verwendet werden, das eine Spannung zwischen 14 und 20 Volt sowie 3 bis 4 Ampere liefern kann.

⚠️ **Achtung:** Nicht jedes Schaltnetzteil ist für diesen Zweck geeignet oder sicher. Praktischer Hinweis: Wählen Sie ein Netzteil, wie es auch bei anderen DCC-Zentralen verwendet wird, zum Beispiel das [Roco 10851](https://www.roco.cc/rde/10851-schaltnetzteil-54-watt.html) oder das [YaMoRC YD7460-18](https://yamorc.de/products/?singleproduct=13843).

Die externe Versorgungsspannung gelangt über eine 3-Ampere-eFuse und eine sogenannte "ideale Diode", die die Schaltung vor Verpolung schützt, zu VCC und zu zwei Step-Down-Konverter-Modulen.

VCC versorgt die H-Brücken für die Gleisspannung und das Programmiergleis. VCC führt außerdem zum Prozessor, sodass über einen Spannungsteiler und einen ADC-Eingang geprüft werden kann, ob die externe Versorgungsspannung den richtigen Wert hat.

Der 12V-Step-Down-Konverter versorgt LocoNet, XpressNet, die CDE-Boosterschnittstelle und die RS-Bus-Rückmeldung mit Strom. Auch die 12V-Spannung wird zur Kontrolle über einen Spannungsteiler an einen ADC-Eingang weitergeleitet.

Die Wahl des 5V-Step-Down-Konverters bedarf einer Erklärung, da Prozessor und Peripherie eigentlich 3V3 erwarten. Der Grund für die Wahl von 5V liegt darin, dass auf dem Olimex-Modul RP2350 Pico2-XL bereits ein eigener Step-Down-Chip verbaut ist. An den 3V3-Ausgang dieser Modul darf *kein* externer 3V3-Wandler angeschlossen werden (siehe unten). Der Ausgang des Olimex-Moduls liefert genügend Leistung, um die übrige Peripherie mit 3V3 zu versorgen. Die Diode zwischen dem 5V-Step-Down-Konverter und dem Olimex-Modul RP2350 Pico2-XL verhindert Schäden, falls zusätzlich zur externen Versorgung auch über den USB-Anschluss Strom eingespeist wird.


><details>
><summary>Erklärung: Warum kein externes 3V3 am Olimex-Modul?</summary>
>
> Der TPS62A02A-Chip auf dem Olimex-Modul besitzt eine aktive Ausgangsentladung ("active output discharge"): Sobald der TPS62A02A abgeschaltet wird (zum Beispiel indem 3V3_EN auf GND gelegt wird), schaltet sich ein interner FET ein, um die in der externen Spule noch gespeicherte Energie aktiv zu entladen. Wird in diesem Zustand dennoch extern 3,3 V an den Ausgang des TPS62A02A angelegt, bleibt der interne FET dauerhaft leitend und zieht mehrere zehn mA aus der externen Versorgung. Dadurch kann sich das SOT-563-Gehäuse überhitzen. Dieses Verhalten wird auch im [E2E-Forum von TI](https://e2e.ti.com/support/power-management-group/power-management/f/power-management-forum/1305135/tps62a01-active-output-discharge) beschrieben.
>
> Daher die Entscheidung, nicht den 3V3-Ausgang, sondern den VSYS-Eingang zu speisen. Dadurch bleibt der onboard verbaute TPS62A02A stets aktiv, unabhängig davon, ob die Versorgung über den 5V-Step-Down-Konverter oder über USB erfolgt. Der TPS62A02A kann genügend Leistung liefern, um auch die übrige Peripherie zu versorgen.
>
> Dieses Verhalten unterscheidet sich vom originalen Raspberry Pi Pico2: Dieser verwendet den RT6150B als Step-Down-Wandler, der diese aktive Ausgangsentladung nicht kennt. Beim Pico2 ist das Anlegen von GND an 3V3_EN und das anschließende externe Anlegen von 3,3 V an den Ausgang tatsächlich die von Raspberry Pi selbst dokumentierte Methode. Für andere RP2350B-Module gilt: Immer zuerst prüfen, welcher Step-Down-Wandler verbaut ist, da sich das Verhalten je nach Board unterscheiden kann.
>
> </details>

</details>


<details>
<summary><strong>Prozessor</strong></summary>

Als Prozessor wurde ein Raspberry Pi Pico-Prozessor gewählt; Details siehe [Prozessorwahl](docs/processor.de.md). Ein wichtiger Vorteil des Raspberry Pi Pico ist, dass er über ein Programmable-IO-(PIO)-Peripheriegerät verfügt, einen spezialisierten Co-Prozessor, mit dem sich sehr präzise getaktete Ausgangssignale erzeugen lassen.

Innerhalb der Pico-Serie gibt es zwei Varianten: den RP2040 mit zwei PIOs und den RP2350 mit drei PIOs. Da wir drei PIOs benötigen, fiel die Wahl auf den RP2350. Die erste PIO wird für das DCC-Signal verwendet, die zweite für den RS-Bus-Master und die dritte für den XpressNet-RS-485-Anschluss.

Der RP2350 wiederum gibt es in zwei Varianten: den "einfachen" RP2350 mit 30 General-Purpose-IO-Pins (GPIO) und den RP2350B, der 18 mehr besitzt. Für eine frühe Testversion der Zentrale wurde der "einfache" RP2350 verwendet; da wir jedoch mehr IO-Pins benötigten, fiel die endgültige Wahl auf den RP2350B.

Mehrere Anbieter (etwa Olimex, Waveshare und WeAct) bieten unterschiedliche RP2350B-Module an. Wir haben uns für das Modul RP2350B-X(X)L von Olimex entschieden; die Begründung dafür findet sich weiter unten im Abschnitt Nachbau.

><details>
><summary>Erklärung: Sind alternative Module möglich?</summary>
>
> Werden weniger Pins und weniger PIOs benötigt — etwa weil kein RS-Bus-Master oder kein XpressNet erforderlich ist —, können auch RP2040-Module verwendet werden. In diesem Fall müssten das Platinenlayout und gegebenenfalls die Spannungsversorgung angepasst werden.
>
> </details>

><details>
><summary>Erklärung: Warum eine PIO für XpressNet?</summary>
>
> Bei den meisten Prozessoren ist es möglich, für das Senden und Empfangen von XpressNet-Nachrichten ein UART zu verwenden. Das funktioniert, weil das UART bei vielen Prozessoren sowohl 8 als auch 9 Datenbits unterstützt. XpressNet erfordert jedoch 9 Datenbits (wobei das neunte Bit angibt, ob es sich um ein Adress- oder ein Datenbyte handelt). Im Vergleich zu anderen Prozessoren ist das UART des Raspberry Pi Pico relativ eingeschränkt und unterstützt nur 8 Datenbits. Durch ein kleines PIO-Programm ist es dennoch möglich, auch auf dem Raspberry Pi Pico mit 9 Datenbits zu arbeiten und damit XpressNet zu unterstützen.

> Mit einigen Tricks wäre es prinzipiell möglich, das UART des Raspberry Pi Pico dennoch für XpressNet zu nutzen. Da es jedoch nur zwei UARTs gibt, wurde davon abgesehen.
>
> </details>

</details>

<details>
<summary><strong>Ethernet</strong></summary>

Der Ethernet-Schaltplan enthält nur das WIZnet-Modul W5500. Alle Pins sind mit dem Prozessor verbunden.

Die älteren (und oft teureren) Module W5100 und W5100S funktionieren wahrscheinlich ebenfalls, dies wurde jedoch nicht getestet.

</details>

<details>
<summary><strong>Gleisausgang</strong></summary>

Das DCC-Signal wird durch ein DRV8874-Modul verstärkt, bestückt mit demselben Treiber-IC, das auch im DCC-EX-Projekt verwendet wird.

Der Schaltplan enthält nur wenige Bauteile, da das Modul selbst bereits eine Reihe von Widerständen enthält, die für unsere Zentrale brauchbare Einstellungen ergeben.

Standardmäßig ist I<sub>TRIP</sub> auf etwa 3 Ampere eingestellt. Falls gewünscht, kann ein anderer Wert gewählt werden, indem der 2,49-kΩ-Widerstand auf dem Modul entfernt und stattdessen R601 mit einem anderen Wert bestückt wird.

Standardmäßig ist **PMODE** mit 3V3 verbunden, wodurch der PWM-Modus aktiviert ist. Falls gewünscht, kann über den "PWM (PH / EN) Solder Jumper" auf der Unterseite der Platine **PMODE** stattdessen mit GND verbunden werden. Dazu die bestehende Kupferverbindung zu 3V3 entfernen, um einen Kurzschluss zu vermeiden, und die andere Lötverbindung zu GND herstellen.

![CD booster input](docs/images/PCB-Bottom-Jumpers.png)

Standardmäßig ist das Modul auf Quad-Level 2 eingestellt; dies kann über einen zweiten "Solder Jumper" auf der Unterseite der Platine geändert werden, indem **IMODE** mit GND verbunden wird. Siehe das DRV8874-Datenblatt für die Unterschiede zwischen den Modi.

Auf der Platine ist kein Platz für eine Snubber-Schaltung und/oder eine Spule vorgesehen. Falls gewünscht, können diese nach dem Steckverbinder in der Verdrahtung zum Gleis platziert werden.

Ausführliche Hintergrundinformationen zur Wahl und zu den Einschränkungen dieses Treiber-ICs sind auf einer eigenen Seite beschrieben. Dort wird ausführlich auf die Einschaltstrom-Probleme eingegangen, die dieser und ähnliche Chips bei kapazitiven Lasten haben können. Siehe: [DRV8874-Endstufe](docs/DRV8874.de.md)

</details>

<details>
<summary><strong>Programmiergleis</strong></summary>

Auf der Platine kann dasselbe DRV8874-H-Brücken-Modul wie für den normalen Gleisausgang verwendet werden. Der Vorteil der Wiederverwendung von Modulen ist eine einheitliche Stückliste und ein einfacherer Nachbau.

Besser ist es jedoch, statt eines DRV8874-Moduls ein DRV8876-Modul zu verwenden. Diese Module können etwas weniger Leistung liefern, was für ein Programmiergleis gerade ein Vorteil ist, und sie sind zudem etwas günstiger.

Ein wichtiger technischer Unterschied zwischen den Modulen DRV8874 und DRV8876 besteht darin, dass Letzteres einen höheren Wert für den Stromspiegel-Verstärkungsfaktor **AIPROPI** aufweist, nämlich 1000 (statt 450). Dadurch beträgt die Spannung, die der ADC-Eingang des Prozessors bei einem (60 mA) DCC-ACK-Signal sieht, 150 mV (statt 67 mV). Dies ermöglicht eine zuverlässigere Messung des DCC-ACK-Signals.

Wie beim normalen Gleisausgang ist es möglich, den standardmäßig auf dem Modul verbauten 2,49-kΩ-Widerstand zu entfernen und durch R801 zu ersetzen. Ein guter Wert für R801 ist 6,8 kΩ. Mit diesem Wert wird der maximale Strom I<sub>TRIP</sub>, der während des Programmierens geliefert werden kann, auf 500 mA begrenzt. RCN-216 empfiehlt beim Einschalten eine (nicht verpflichtende) Begrenzung auf 250 mA ±20 %, erlaubt für fest eingestellte Strombegrenzer aber auch Werte bis zu 1 A; 500 mA passt gut in diesen Bereich.

Ein zusätzlicher Vorteil des Ersetzens des 2,49-kΩ-Widerstands durch einen 6,8-kΩ-Widerstand ist, dass die Spannung während eines DCC-ACK auf deutlich über 400 mV erhöht wird, wodurch sie am ADC-Eingang des Prozessors besser messbar ist.

</details>

<details>
<summary><strong>CDE-Boosterschnittstelle</strong></summary>

Auf der Platine wurde derselbe Typ H-Brücken-Modul wie für das Programmiergleis gewählt (DRV8876 oder DRV8874). Technisch gesehen könnte auch ein DRV8871-Modul von AliExpress verwendet werden. Die Abmessungen dieses Moduls passen jedoch nicht auf die aktuelle Platine.

Für den CDE-Ausgang gelten die bekannten Einschaltstrom-Probleme der DRV887x-ICs nicht. Die Last ist nicht kapazitiv, und für das CD-Signal wird nicht mehr als einige hundert Milliampere benötigt. Der interne Kurzschlussschutz des DRV887x spricht beim Einschalten nicht an, es sei denn, der Ausgang ist tatsächlich kurzgeschlossen. Der DRV887x ist für diesen Zweck zwar überdimensioniert, kann aber grundsätzlich problemlos eingesetzt werden.

Ausführliche Informationen zur CDE-Schnittstelle sind auf einer eigenen Seite beschrieben: [CDE-Schnittstelle](docs/CDE.de.md).

</details>

<details>
<summary><strong>LocoNet</strong></summary>

Der Schaltplan für den LocoNet-Teil ist Standard und an vielen Stellen im Internet beschrieben. Allerdings wurden die Werte einiger Bauteile angepasst, sodass das Ganze auch mit 3V3 funktioniert.

Neben der LocoNet-T-Schnittstelle ist auch eine LocoNet-B-Schnittstelle vorhanden, die an den äußeren Pins des Steckers ein RailSync-Signal für Booster liefert. Die Erzeugung des RailSync-Signals ist auf verschiedene Arten möglich.

- Bis einschließlich V2.2 wurde eine Schaltung verwendet, die aus dem Internet kopiert wurde (https://www.fucik.name/masinky/NanoL/, https://oshwlab.com/fiorinid/z21pg-by-df-pro). Diese Schaltungen nutzen einen speziellen Treiber-IC; im Schaltplan werden mehrere Optionen aufgeführt. Von den genannten Optionen verfügt nur der UCC27425-IC über einen ENABLE-Eingang, wodurch das RailSync-Signal einfach deaktiviert werden kann. An den Ausgängen dieses Treibers sind Widerstände mit 33 Ω / 2 Watt (oder besser: 22 Ω / 5 Watt) angeschlossen. Einerseits dienen diese Widerstände als Kurzschlussschutz (deshalb müssen sie 2 bis 5 Watt dissipieren können). Andererseits bilden diese Widerstände zusammen mit den 4,7-nF-Kondensatoren ein Tiefpassfilter, wodurch Störungen reduziert werden. Die P6KE15A sind ESD-Dioden, die den LocoNet-Treiber-IC schützen. Der Nachteil dieser Schaltung ist, dass sie nicht mehr als 100 mA liefern kann, wodurch die maximale Anzahl der anschließbaren Booster je nach Booster-Typ auf etwa 10 begrenzt ist.

- Ab Version 2.3 wird für RailSync das CD(E)-Signal (Booster) verwendet. Dadurch werden nicht nur weniger Bauteile benötigt (was die Kosten senkt), sondern es kann auch eine viel größere Anzahl von Boostern angeschlossen werden. Außerdem verfügen nun alle angeschlossenen Booster über genau dasselbe Signal, unabhängig davon, ob sie über CDE oder über LocoNet-T angeschlossen sind.

</details>

<details>
<summary><strong>XpressNet</strong></summary>

Die XpressNet-Schnittstelle ist einfach und Standard. Auf der [OpenDCC-Seite](https://www.opendcc.de/elektronik/opendcc/xpressnet_hw.html) findet sich eine Grundschaltung, die von vielen nachgebaut wurde. Die einzige Änderung besteht darin, dass die 1K5-Widerstände durch 560 Ω ersetzt wurden, da das Treiber-IC nicht mit 5 V, sondern mit 3V3 versorgt wird. Der 8K2-Widerstand (R504) sorgt dafür, dass der Treiber im RX-Modus startet.

Häufig wird als Treiber ein MAX(3)485-IC verwendet. Eine bessere Wahl ist der MAX(3)483, also die Slew-Rate-limitierte Version dieses Treibers, da hochfrequente Störungen auf der RS-485-Leitung dadurch begrenzt werden. Leider ist dieses IC deutlich teurer. Die oben genannte OpenDCC-Seite gibt weitere Details zu möglichen alternativen Treiber-ICs.

</details>

<details>
<summary><strong>RS-Bus-Rückmeldung</strong></summary>

Der Schaltplan für den RS-Bus wurde von der [Der-Moba-Website](https://www.der-moba.de/index.php/RS-Rückmeldebus) übernommen.
Der dort für das Senden genannte Transistor wurde jedoch durch einen MOSFET (IRLZ44) ersetzt, wodurch eine etwas höhere Last möglich wird.

</details>


## Nachbau
Für den Nachbau der Zentrale ist es am einfachsten, die aktuelle Platine fertigen zu lassen, indem die Datei [production/TMC-Centrale.zip](production/TMC-Centrale.zip) an ein Unternehmen wie JLCPCB gesendet wird. Diese Datei enthält alle (Gerber-)Dateien, die der Hersteller benötigt. Im Sommer 2026 kostete die Fertigung von 5 Platinen inklusive Versand etwa 25 €.

Der Entwurf ist Open Source. Nachbau, Anpassung und Weitergabe werden unter den Bedingungen der [Lizenz](LICENSE) ausdrücklich begrüßt, unter Angabe dieser Quelle.

Die vollständige Bauteileliste befindet sich hier: [production/bom.csv](production/bom.csv).

Um den Nachbau so einfach wie möglich zu gestalten, wurde, wo immer möglich, auf fertige Module gesetzt, wie sie online etwa bei AliExpress und anderen Anbietern erhältlich sind. Im Folgenden eine Übersicht der wichtigsten Module sowie der zur Platine passenden Steckverbinder:


<details>
<summary><strong>RP2350B (MCU-Modul)</strong></summary>

Als Prozessor wurde das Modul [Olimex RP2350B-XL](https://www.olimex.com/Products/RaspberryPi/PICO/PICO2-XXL/open-source-hardware) gewählt. Olimex garantiert, dass die Produktion dieser Module bis mindestens 2035 fortgeführt wird, und hat alle KiCad-Entwurfsdateien sowie Fertigungsunterlagen offen auf GitHub veröffentlicht.

Von diesem Modul gibt es zwei Varianten: das RP2350B-XL (5 €) und das RP2350B-XXL (9 €). Der Unterschied besteht darin, dass die XXL-Variante einige zusätzliche Fähigkeiten besitzt, die für diese Zentrale nicht benötigt werden. Obwohl im Schaltplan auf die XXL verwiesen wird, wird aus Preisgründen die XL-Variante bevorzugt. Beide Varianten passen jedoch auf die Platine.

Die Module können direkt bei [Olimex](https://www.olimex.com/Products/RaspberryPi/PICO/PICO2-XXL/open-source-hardware) gekauft werden, aber auch über Distributoren wie [TME](https://www.tme.eu).

</details>

<details>
<summary><strong>DRV8874 / DRV8876 (Motortreiber)</strong></summary>

Für den DCC-, den Programmiergleis- und den CDE-Ausgang werden fertige DRV8874-/DRV8876-Module verwendet.

Pololu brachte vor einigen Jahren Module für den [DRV8874](https://www.pololu.com/product/4035) und den [DRV8876](https://www.pololu.com/product/4037) auf den Markt. Seit Frühjahr 2026 sind (günstigere) Varianten auch bei AliExpress erhältlich. Das Pololu-Modul enthält als zusätzlichen Schutz einen verpolungsschützenden MOSFET, den die AliExpress-Module nicht besitzen. Dieser zusätzliche Schutz wird für diese Zentrale nicht benötigt.

![AliExpress DRV8876](docs/images2/DRV8876.png)

</details>

<details>
<summary><strong>Step-Down-Modul (5V/12V)</strong></summary>

Die Abmessungen auf der Platine sind auf die bekannten MP1584-Step-Down-Module abgestimmt, die von mehreren Anbietern auf AliExpress angeboten werden. Es werden zwei Module benötigt. Das erste liefert die 5V-Versorgungsspannung für den Prozessor (Anmerkung: Das Olimex-Prozessormodul selbst wandelt diese 5 V wieder in 3V3 um), das zweite Modul liefert 12 V. Anstelle von Modulen mit fester 5V-/12V-Ausgangsspannung können auch Module mit einstellbarer Ausgangsspannung verwendet werden.

Andere Module als das MP1584 funktionieren grundsätzlich ebenfalls, passen jedoch nicht auf die Platine.

![AliExpress MP1584](docs/images2/MP1584.png)

</details>

<details>
<summary><strong>Ideal-Diode-Modul</strong></summary>

Als Verpolungsschutz wird ein XL74610-Modul verwendet, wie es bei AliExpress und anderen Anbietern erhältlich ist.

![AliExpress XL74610](docs/images2/XL74610.png)

</details>

<details>
<summary><strong>W5500 Ethernet</strong></summary>

Für die Ethernet-Verbindung wird ein W5500-Modul verwendet, wie es bei AliExpress und anderen Anbietern erhältlich ist.

![AliExpress W5500](docs/images2/W5500.png)

</details>

<details>
<summary><strong>Steckverbinder</strong></summary>

Die Platine ist für folgende Steckverbinder ausgelegt:

><details>
><summary>LocoNet: 6P6C (RJ12)</summary>
>
> Es gibt verschiedene Marken und Typen von 6P6C-Steckverbindern auf dem Markt, mit unterschiedlichen Footprints. Die Platine ist für Steckverbinder mit einem Footprint ausgelegt, der dem der WayConn-MJEA-Steckverbinder entspricht. Diese haben ihre elektrischen Anschlüsse auf der Unterseite (Platinenseite).
> ![AliExpress 6P6C](docs/images2/6P6C.png)
>
> </details>
><details>
><summary>Hohlstecker (Barrel Jack)</summary>
>
> Für den Spannungsversorgungsanschluss wird ein 5,5x2,1-mm-Hohlstecker verwendet.
> ![AliExpress BarrelJack](docs/images2/BarrelJack.png)
>
> </details>
><details>
><summary>Klemmleisten (Terminal Blocks)</summary>
>
> Für den DCC-Gleisausgang wird eine PhoenixContact MSTBA 2,5/2-G mit einem Pinabstand von 5,00 mm verwendet. Klemmleisten mit einem Pinabstand von 5,08 mm passen jedoch ebenfalls.
>
> Für den Programmierausgang wird eine Klemmleiste Phoenix Contact MC 1.5/2-G-3.81 verwendet.
>
> Für den CDE-Ausgang wird eine Klemmleiste Phoenix Contact MC 1.5/3-G-3.81 verwendet.
>
> Für den RS-Bus-Eingang wird eine Klemmleiste Phoenix Contact MC 1.5/2-G-3.81 verwendet.
>
> Für den XpressNet-Ausgang wird eine Klemmleiste Phoenix Contact MC 1.5/4-G-3.81 verwendet. Auf der Vorderseite ist ein 5-poliger DIN-Steckverbinder montiert.
>
> Varianten dieser Steckverbinder werden von mehreren Herstellern angeboten, unter anderem von PTR/Hartmann.
>
> ![AliExpress DIN](docs/images2/DIN.png)
>
> </details>

</details>


## Software

Die Firmware für diese Zentrale befindet sich in einem eigenen GitHub-Repository (https://github.com/tmc-digiboys/TMC-LZ210-Command-Station) und wird daher hier nicht weiter besprochen.


## Vergleich mit anderen Zentralen

Diese Zentrale steht nicht für sich allein, sondern in einer langen Tradition von Open-Source-DCC-Zentralen:

<details>
<summary><strong>OpenDCC</strong></summary>

Eine der ersten Open-Source-DCC-Zentralen ist die [OpenDCC Z1](https://www.opendcc.de/elektronik/opendcc/opendcc.html). Diese Zentrale wurde vor zwanzig Jahren von Wolfgang Kufer entworfen.
- **Prozessor.** OpenDCC läuft auf einem 8-Bit-Atmel-AVR (ATmega32 oder ATmega644P) mit 16 MHz. Diese Zentrale verwendet einen Dual-Core-RP2350 mit 125 MHz und einem PIO-Peripheriegerät und ist dadurch um Größenordnungen leistungsfähiger.
- **Endstufe.** OpenDCC verwendet eine feste H-Brücke (STM L6206), theoretisch für 2,8 A pro Ausgang ausgelegt, auf der Platine thermisch jedoch auf etwa 1,5 A Dauerstrom begrenzt. Diese Zentrale verwendet ein DRV8874-Modul (theoretisch 6 A), dessen Strom auf dem Modul selbst auf etwa 2,9 A begrenzt wird.
- **Rückmeldung.** OpenDCC unterstützt S88, mit einer Erweiterung für die Weichenstellungs-Rückmeldung. Diese Zentrale verwendet RS-Bus und LocoNet.
- **Anschluss an einen PC.** OpenDCC kommuniziert über RS232/USB mit einem PC, mit XpressNet oder P50X als übergeordnetes Protokoll; eine Netzwerk- oder WLAN-Verbindung besitzt OpenDCC selbst nicht. Diese Zentrale verfügt neben USB über einen eigenen Ethernet-Anschluss und unterstützt XpressNet und Z21 als übergeordnetes Protokoll.
- **Booster.** OpenDCC besitzt keine eigene Schnittstelle für externe Booster wie CDE oder LocoNet-B.
- **Handregler.** Wie diese Zentrale unterstützt auch OpenDCC XpressNet-Handregler (etwa die Roco Multimaus). LocoNet-Handregler unterstützt OpenDCC jedoch nicht.

Die OpenDCC-Z1-Zentrale kann als Vorläufer der Open-Source-DCC-Zentralen betrachtet werden. Viele ihrer Ideen, Schaltungen und Software wurden (in angepasster Form) von anderen übernommen und werden bis heute verwendet. Die Z1 heute noch nachzubauen erscheint jedoch keine gute Idee: Moderne Mikrocontroller sind sowohl günstiger als auch um ein Vielfaches leistungsfähiger als die 8-Bit-ATmega-Generation, auf der die OpenDCC Z1 seinerzeit basierte.
</details>


<details>
<summary><strong>BiDiB</strong></summary>

[BiDiB](https://www.bidib.org/) kann als Nachfolger der OpenDCC-Z1-Zentrale betrachtet werden. BiDiB wird seit 2010 — erneut — von Wolfgang Kufer entwickelt und unter anderem von Fichtelbahn in Hardware umgesetzt.
- **Architektur.** BiDiB ist in erster Linie ein Bus, an den mehrere intelligente Komponenten (wie GBMboost/GBM16T/OpenSwitch/IF2) angeschlossen werden können. Es handelt sich nicht um eine Zentrale im herkömmlichen Sinn, mit eingebautem Booster und Schnittstellen wie XpressNet, LocoNet und RS-Bus.
- **Prozessor.** BiDiB-Implementierungen basieren auf dem Atmel-XMEGA-Prozessor. Obwohl dies einer der leistungsfähigsten 8/16-Bit-/32-MHz-Prozessoren ist, handelt es sich zugleich um einen recht alten Prozessor, der im Vergleich zum Dual-Core-RP2350/125-MHz-Prozessor dieser Zentrale ziemlich teuer ist.
- **Nachbau.** BiDiB-Platinen sind für SMD-Bauteile ausgelegt. Obwohl die BiDiB-Entwürfe fortschrittlicher und von höherer Qualität sind als die dieser (THT-)Zentrale, sind sie auch schwerer nachzubauen, anzupassen und zu erweitern.
- **RailCom.** BiDiB glänzt beim Empfangen, Analysieren und Verarbeiten von RailCom-Rückmeldungen. Diese Zentrale bietet hierfür keine Unterstützung.

BiDiB bietet ein vollständiges Ökosystem mit Schwerpunkt auf intelligenten Komponenten und umfangreicher RailCom-Verarbeitung. Diese Zentrale ist in erster Linie eine traditionelle Zentrale wie die von Roco und Lenz, die über etablierte Protokolle mit Geräten verschiedener Hersteller kommuniziert.

</details>


<details>
<summary><strong>Z21PG</strong></summary>

[Z21PG](https://pgahtow.de/w/Zentrale_Z21PG/) ist ein Arduino-basiertes Selbstbauprojekt von Philipp Gahtow für einen kostengünstigen Nachbau der Roco Z21.
- **Einfacher Nachbau.** Philipp Gahtow selbst veröffentlicht vor allem Software (Arduino-Sketches) und einzelne Schaltpläne für mehrere Hardwarevarianten (Arduino MEGA, ESP32, ...). Im Gegensatz zu dieser Zentrale gibt es keinen offiziellen Z21PG-(KiCad-)Platinenentwurf. Für die Z21PG existieren jedoch einige von Dritten erstellte Platinenentwürfe. Beide Zentralen sind einfach nachzubauen.
- **Prozessor und DCC-Signal.** Die am häufigsten nachgebaute Z21PG-Variante läuft auf einem (veralteten) ATmega2560 (16 MHz), wobei auch der ESP32 als Möglichkeit genannt wird. Das DCC-Signal wird per Software über Timer-Interrupts erzeugt, nicht über ein dediziertes Hardware-Peripheriegerät. Diese Zentrale verwendet einen (modernen) Dual-Core-RP2350 mit 125 MHz und einem PIO-Peripheriegerät, mehr als zehnmal so schnell und mit jitterfreiem DCC-Signal.
- **Schnittstellen.** Die Z21PG bietet vergleichbare Anschlüsse wie die Roco-Z21-Zentrale: LAN, X-Bus (XpressNet), L-Bus (LocoNet-T), Boosteranschluss, Gleis- und Programmierausgang. Diese Zentrale verfügt zusätzlich über einen RS-Bus-Anschluss, und ihre Booster-Anschlüsse sind kompatibel mit CDE und LocoNet-B.

Von allen hier besprochenen Projekten hat die Z21PG die meisten Gemeinsamkeiten mit dieser Zentrale. Beide sind modular aufgebaut und relativ einfach und günstig nachzubauen. Die Z21PG orientiert sich in erster Linie an der Roco Z21, während sich diese Zentrale zusätzlich auch an den Lenz-Zentralen orientiert. Die Z21PG ist ein etwas älterer Entwurf; diese Zentrale verwendet modernere Bauteile.
</details>


<details>
<summary><strong>DCC-EX</strong></summary>

Eine derzeit populäre Selbstbau-Zentrale ist [DCC-EX](https://dcc-ex.com/). Es gibt funktionale Gemeinsamkeiten (unter anderem die DRV8874-Endstufe), aber auch einige wesentliche Unterschiede:
- **Verfügbarkeit.** DCC-EX bietet sowohl werkseitig fertig montierte Platinen (EX-CSB1, ab ca. 75–115 € zzgl. MwSt./Versand) als auch Selbstbau auf Platinen im Arduino-Formfaktor. Diese Zentrale ist für den Selbstbau vorgesehen, und die Bauteilkosten sind niedriger (ca. 40 €).
- **Anschluss an PC/Netzwerk.** Diese Zentrale verwendet Ethernet (XpressNet und Z21); DCC-EX verwendet WLAN (mit Unterstützung für JMRI, WiThrottle und Engine Driver).
- **Schnittstellen für Handregler.** Diese Zentrale besitzt native (RS485) XpressNet- und LocoNet-Schnittstellen, an die Handregler von Lenz, Roco und anderen angeschlossen werden können. DCC-EX richtet sich vor allem an WLAN-Fahrregler und JMRI.
- **Schnittstellen für Rückmeldung.** Diese Zentrale besitzt native RS-Bus- und LocoNet-Schnittstellen. DCC-EX besitzt keine solchen Schnittstellen.
- **Booster-Schnittstellen.** Diese Zentrale unterstützt externe Booster über LocoNet-B und die CDE-Schnittstelle. DCC-EX bietet keine CDE-/LocoNet-B-Unterstützung, verfügt jedoch mit RailSync über einen Mechanismus, mit dem externe Booster angeschlossen werden können.
- **Gemischter DCC-/DC-Betrieb.** DCC-EX kann jeden Ausgang zwischen DCC und DC-PWM umschalten, sodass auch klassische Gleichstrom-Lokomotiven fahren können. Diese Zentrale unterstützt ausschließlich DCC.

Der deutlichste Unterschied liegt im Ökosystem, in das die jeweilige Zentrale eingebettet ist. DCC-EX orientiert sich stark an der amerikanischen Hobbyszene: WLAN-Fahrregler, JMRI und, für Zubehör, das LCN-Konzept (Layout Control Node), bei dem preiswerte Arduino-Nanos mit nRF24L01-Funkmodulen Weichen, Signale und Beleuchtung drahtlos ansteuern. Diese Zentrale richtet sich dagegen stärker an traditionell europäische Nutzer: XpressNet- und LocoNet-Handregler, CDE-/LocoNet-B-Booster und RS-Bus-Rückmeldung. Für Clubs, die bereits mit Lenz-, Roco- oder vergleichbarer Ausrüstung arbeiten, ist das ein direkterer Anschluss als der WLAN-/JMRI-/LCN-Weg von DCC-EX.
</details>


<details>
<summary><strong>OpenRemise</strong></summary>

[OpenRemise](https://github.com/OpenRemise) richtet sich ausdrücklich auf **Benutzerfreundlichkeit ohne Löt- oder Elektronikkenntnisse**: Das System ist nahezu Plug-and-Play und wird nach der Installation vollständig über eine Weboberfläche auf Smartphone, Tablet oder PC bedient. Damit steht es philosophisch fast im Gegensatz zu dieser Zentrale.
- **Hardware.** OpenRemise richtet sich vor allem an Nutzer, die eine fertige Platine möchten; die Platine besteht überwiegend aus SMD-Bauteilen und lässt sich nicht einfach von Hand löten. Diese Zentrale richtet sich vor allem an Personen, die selbst löten und gegebenenfalls anpassen möchten.
- **Prozessor.** OpenRemise basiert, ebenso wie diese Zentrale (RP2350), auf einem modernen, leistungsfähigen Prozessor (ESP32). Für jitterfreie DCC-Signalerzeugung verwendet OpenRemise, ebenso wie diese Zentrale (PIO), ein dediziertes Peripheriegerät (RMT).
- **Endstufe.** OpenRemise besitzt eine auf diskreten MOSFETs basierende Endstufe, die der in dieser Zentrale verwendeten DRV8874-Endstufe überlegen ist.
- **Zielgruppe.** OpenRemise glänzt beim Aktualisieren von Decoder-Firmware, kann aber auch als vollständige Zentrale eingesetzt werden. Diese Zentrale ist eher als traditionelle DCC-Zentrale gedacht und verfügt über mehr Standard-Schnittstellen (XpressNet, LocoNet, CDE).
- **Netzwerk.** OpenRemise unterstützt WLAN und setzt auf webbasierte Bedienung. Diese Zentrale unterstützt Ethernet und setzt auf PC-Steuerung über das XpressNet/Z21 Protokoll.

OpenRemise ist in erster Linie ein fertiger SMD-Platinenentwurf mit zugehöriger Firmware. Diese Zentrale richtet sich vor allem an Hobbyisten, die eine preiswerte Zentrale wollen, die einfach nachzubauen und anzupassen ist. Dadurch musste diese Zentrale einige Kompromisse eingehen (etwa bei der DCC-Endstufe), die OpenRemise nicht eingehen musste.
</details>


## Status und Zukunftspläne
Die Zentrale ist seit Sommer 2026 beim TMC im Einsatz, für die Ansteuerung von Weichen und Signalen sowie für die Gleisbesetztmeldung. Obwohl die meisten Funktionen getestet wurden und sich als gut erwiesen haben, gibt es keine Garantie, dass alles wie vorgesehen funktioniert.

Es gibt Pläne, von der Zentrale auch eine SMD-Version zu erstellen, wobei diese Pläne noch nicht endgültig sind.

Auf der Platine ist Platz für einen Erweiterungsanschluss vorgesehen, an dem unter anderem SPI und I2C anliegen. Auf einer Erweiterungsplatine könnten zusätzliche Funktionen realisiert werden, wie ein Display, RailCom-Rückmeldung, S88N oder der CAN-Bus.

Grundsätzlich sollte es auch möglich sein, die Zentrale mit einer WLAN-Schnittstelle auszustatten. Da Ethernet jedoch zuverlässiger ist, haben wir auf WLAN verzichtet.

## Referenzen

<details>
<summary><strong>Details zu diesem Entwurf</strong></summary>

- [Prozessorwahl](docs/processor.de.md) — Vergleich des RP2350 mit ESP32, DxCore und STM32
- [DRV8874-Endstufe](docs/DRV8874.de.md) — Hintergrund zur Wahl des Treiber-ICs und zur Einschaltstrom-Problematik
- [CDE-Schnittstelle](docs/CDE.de.md) — Hintergrund zur externen Boosterschnittstelle

</details>

<details>
<summary><strong>Protokolle und Schaltungen</strong></summary>

- [RailCommunity (RCN)](https://railcommunity.de) — DCC-Normen
- [XpressNet-Spezifikation](https://www.lenz-elektronik.de/media/0a/d1/4e/1760451943/XpressNet_V40_2.pdf?ts=1760451943) — XpressNet Version 4.0 (Lenz)
- [XpressNet-Schaltung](https://www.opendcc.de/elektronik/opendcc/xpressnet_hw.html) — Grundschaltung (OpenDCC)
- [LocoNet](https://www.digitrax.com/static/apps/cms/media/documents/tech_notes/loconet-personal-edition-1-0.pdf) — Personal Use Edition (Digitrax)
- [RS-Bus](https://www.der-moba.de/index.php/RS-Rückmeldebus) — Grundschaltung (Der Moba)
- [Paco's Official Web Site](https://usuaris.tinet.cat/fmco/home_en.htm) — Grundschaltungen für XpressNet, LocoNet, CDE und Booster

</details>

<details>
<summary><strong>Kommerzielle Zentralen und vergleichbare Selbstbauprojekte</strong></summary>

- [Lenz](https://www.digital-plus.de/) — Zentralen LZV100/LZV200 (XpressNet, CDE und RS-Bus)
- [Roco/Fleischmann](https://www.z21.eu/) — Z21-Zentrale
- [OpenDCC](https://www.opendcc.de/)
- [BiDiB](https://www.bidib.org/)
- [Z21PG](https://pgahtow.de/w/Zentrale_Z21PG/)
- [DCC-EX](https://dcc-ex.com/)
- [OpenRemise](https://github.com/OpenRemise)

</details>

<details>
<summary><strong>Sonstiges</strong></summary>

- [Olimex RP2350B-XL/XXL](https://www.olimex.com/Products/RaspberryPi/PICO/PICO2-XXL/open-source-hardware) — MCU-Modul

</details>
