[🇬🇧 English](processor.md) | [🇩🇪 Deutsch](processor.de.md) | [🇳🇱 Nederlands](processor.nl.md)

# Prozessorauswahl

Modulares Design ist ein wichtiges Kriterium für den Bau unserer TMC-DCC-Zentrale.
 Die [Z21PG](https://pgahtow.de/w/Zentrale_Z21PG/de)-Zentrale von Philipp Gahtow war dabei für uns ein leuchtendes Beispiel. Diese Zentrale ist modular aufgebaut und unterstützt mehrere Prozessoren, wie beispielsweise den ATMega2560, den ATMega328 und den ESP32.

Der ATMega2560 scheint dabei mit Abstand die beste Unterstützung zu haben und am häufigsten eingesetzt zu werden.
Da der ATMega2560 jedoch ein relativ teurer und alter 8-Bit-Prozessor ist, hatten wir den Wunsch, ihn durch einen moderneren und billigeren Prozessor zu ersetzen.

Bei der Z21PG-Zentrale werden die verschiedenen Prozessoren über „Bit-Banging“ von der DCCInterfaceMaster-Bibliothek (https://github.com/Digital-MoBa/DCCInterfaceMaster) angesteuert. Dabei werden die Ausgangspins des Prozessors über sogenannte Interrupt-Service-Routinen (ISRs) gesetzt.

Obwohl diese Methode den Vorteil hat, dass nur wenig prozessorspezifischer Code benötigt wird, ist unsere Erfahrung dennoch, dass obengenannte Bibliothek aufgrund der großen Anzahl von #ifdef-Konstruktionen relativ schwer zu verstehen und daher schwer anzupassen ist. Daher haben wir zunächst den „Bit-Banging“-Code dieser Bibliothek neu strukturiert und überarbeitet und den ATMega- sowie den ESP32-Code entsprechend angepasst.
Anschließend haben wir den Code für eine Reihe neuer Prozessoren erweitert, darunter den STM32, den RP2040 und den DxCore.

Ein Nachteil des „Bit-Banging“ ist, dass es im DCC-Signal zu einem Jitter von mehreren Mikrosekunden kommen kann. Nachdem wir den „Bit-Banging“-Code der ursprünglichen Bibliothek überarbeitet hatten, wagten wir uns an ein weiteres Experiment: Wie einfach bzw. schwierig würde es sein, den Code von Grund auf komplett neu zu schreiben, diesmal unter Nutzung der modernen Peripheriegeräte, die die aktuelle Prozessorgeneration mit sich bringt. In diesem Schritt haben wir die Zustandsmaschine, die den Kern des „Bit-Banging“-Codes bildet, durch einen Ansatz ersetzt, bei dem das gesamte DCC-Paket im Voraus berechnet wird, einschließlich des exakten Timings aller Bits. Wir haben Implementierungen für DxCore-, STM32-, ESP32- und RP2040/2350-Prozessoren erstellt. Im Laufe dieses Experiments haben wir einige Erfahrungen gesammelt, bei denen wir (für uns) überraschende Ergebnisse feststellen konnten. Diese Erkenntnisse werden im Folgenden beschrieben. Wir beginnen mit dem Prozessor, den wir für die Erzeugung von DCC weniger geeignet halten, und schließen mit dem Prozessor ab, der uns am besten gefallen hat

## ESP32
Wir sind etwas enttäuscht vom ESP32. Auf dem Papier ist der sogenannte RMT-Timer ideal für die Erzeugung von DCC-Signalen und wird daher auch in Projekten wie DCC-EX und OpenRemise verwendet. Innerhalb eines DCC-Pakets liefert der RMT ein hervorragendes DCC-Signal, und es tritt kein Jitter auf. Das Problem tritt jedoch zwischen den DCC-Paketen auf. Durch die Kombination aus FreeRTOS, der relativ ressourcenintensiven ESP-IDF-Hardware-Abstraktionsschicht (HAL) und den (was DCC betrifft) unzureichenden IDF v5-RMT-Treibern (also der aktuellen Version) entsteht jedes Mal eine erhebliche Verzögerung bei einer Neukonfiguration des RMT für das nächste Paket. Diese Verzögerung beträgt in der Regel etwa 20 bis 30 Mikrosekunden und ist zudem nicht konstant. Messungen zeigen, dass der Jitter im Durchschnitt etwa 2 Mikrosekunden beträgt, jedoch bis zu 7 Mikrosekunden oder mehr ansteigen kann, wobei in einigen Internetforen sogar Spitzenwerte von über 10 Mikrosekunden gemeldet werden, insbesondere bei WLAN-Aktivität.
Da dieser Jitter sowohl zwischen DCC-Paketen als auch beim RailCom-Cutout auftritt, entspricht das Signal nicht immer den DCC-Spezifikationen.

Daher ist der ESP32 unserer Meinung nach am wenigsten geeignet, um DCC-Signale zu erzeugen.

## DxCore
Die DxCore-Familie bietet eine deutlich bessere performance, wie beispielsweise der AVR64DA48, der als moderner Nachfolger der klassischen Arduino-Prozessoren (ATMega 328, 2560) angesehen werden kann.

Wir verwenden dabei TCA0 für das DCC-Signal und TCD0 (das meist ungenutzt bleibt) für den RailCom-Cutout. Da diese Prozessoren kein DMA unterstützen, wird nach jedem DCC-Bit (116/200 µs) ein Interrupt ausgelöst, in dem die Timer-Register für das nächste Bit gesetzt werden. In der Praxis ergibt dies ein äußerst stabiles DCC-Signal ohne Jitter zwischen den Paketen. Der RailCom-Cutout-Timer TCD0 wird in der ISR zu Beginn jedes Pakets erneut gestartet, wodurch es – abhängig von anderen Interrupts – zu einem gewissen Jitter im Cutout kommen kann.
Versuche, den Cutout-Timer vollständig hardwaremäßig an den DCC-Timer zu koppeln (beispielsweise über das Ereignissystem), waren bei uns nicht von Erfolg gekrönt. Dennoch ist das DCC-Signal selbst jitterfrei, auch wenn dies aufgrund der vielen Interrupts mit einer relativ hohen CPU-Auslastung einhergeht.

## STM32
Die STM32-Serie besteht aus modernen 32-Bit-Mikrocontrollern und umfasst eine große Vielfalt an Varianten. Es gibt verschiedene Familien, wie die C-, F-, G- und H-Familien, und innerhalb jeder Familie gibt es wiederum eine Reihe von Varianten (wie die F1 und F4). Obwohl alle STM32-M Prozessoren eine gemeinsame Basis haben, unterscheiden sich die Varianten erheblich voneinander, beispielsweise hinsichtlich der DMA-Architektur, des Caching, der Speicherstruktur und der verfügbaren Peripheriegeräte. In der Praxis bedeutet dies, dass (Arduino-)Code nicht ohne Weiteres zwischen verschiedenen STM32-Varianten austauschbar ist und dass oft Anpassungen für die jeweilige Variante erforderlich sind. Wir haben Implementierungen für die F4- und H7-Serien erstellt, ohne FreeRTOS zu verwenden.

Wir haben Timer 3 und DMA zur Erzeugung des DCC-Signals und Timer 4 für den RailCom-Cutout verwendet.
Das gesamte DCC-Paket wird vorab in einen DMA-Puffer geladen, sodass die Signalgenerierung vollständig über die Hardware erfolgt. Lediglich am Ende jedes DCC-Pakets ist ein Interrupt erforderlich, um den DMA-Kanal neu zu konfigurieren, ergänzt durch Timer-4-Interrupts am Anfang und am Ende des Cutouts. Da pro Paket Dutzende Mikrosekunden für diese Neukonfiguration zur Verfügung stehen, tritt in der Praxis kein Jitter zwischen aufeinanderfolgenden DCC-Paketen auf. Für die RailCom-Cutout-Funktion ist theoretisch ein gewisses Maß an Jitter möglich, doch in der Praxis erwies sich dies als nicht messbar, auch weil die Interrupt-Priorität einstellbar ist und optimiert werden kann.

Das Ergebnis ist ein sehr stabiles und sauberes DCC-Signal bei geringer CPU-Auslastung, da pro Paket nur eine begrenzte Anzahl (3) von Interrupts erforderlich ist.


## RP2040/RP2350
Die größte Überraschung für uns waren der RP2040 und der RP2350, die vom Raspberry Pi Pico bekannt sind.
Original-Raspberry-Pi-Pico-Boards kosten etwa 5 Euro, und kleine Boards sind bereits ab etwa 1,5 Euro erhältlich.
Die RP-Familie besteht aus nur zwei Varianten, die softwaretechnisch weitgehend kompatibel sind und über eine interessante Peripherie verfügen: den PIO (Programmable IO). Dabei handelt es sich im Grunde um einen kleinen, spezialisierten Coprozessor, mit dem sehr präzise getaktete Ausgangssignale erzeugt werden können. In Kombination mit DMA kann das gesamte DCC-Signal, einschließlich RailCom-Cutout, vollständig von einem einzigen PIO (in dem zwei sehr einfache Zustandsmaschinen verwendet werden) erzeugt werden. Unsere RP-Implementierung liefert ein absolut stabiles DCC-Signal ohne jeglichen Jitter zwischen den Paketen oder beim Cutout. Zudem ist die Implementierung relativ einfach und die CPU-Auslastung minimal: nur ein Interrupt pro Paket. Mit einer Taktrate von 125 bis 200 MHz und einer Dual-Core-Architektur bietet der RP2040 zudem sehr viel Rechenleistung.

Für die Erzeugung von DCC-Signalen ist der RP2040 unserer Meinung nach daher der am besten geeignete Prozessor. Ein Nachteil ist jedoch die relativ mäßige Qualität des ADC, was diesen Prozessor beispielsweise für Belegtmelder weniger geeignet macht.

## Fazit
Während der ESP32 aufgrund seiner nicht optimierten RMT-Treiberimplementierung und Jitter-Problemen enttäuscht, erzeugen sowohl DxCore als auch STM32 ein gutes DCC-Signal, wenn auch mit unterschiedlichen Kompromissen.

Der RP2040/2350 sticht jedoch hervor: mit der besten DCC-Signalqualität, der geringsten CPU-Auslastung, niedrigen Kosten und einer relativ einfachen Implementierung. Deshalb haben wir uns in unserer Zentrale für den RP2040/2350 entschieden.
