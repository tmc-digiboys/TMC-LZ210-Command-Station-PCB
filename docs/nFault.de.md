[🇬🇧 English](nFault.md) | [🇩🇪 Deutsch](nFault.de.md) | [🇳🇱 Nederlands](nFault.nl.md)

# Umgang mit nFAULTs

Der DRV8874 ist, wie andere moderne integrierte H-Brücken-Treiber, für (induktive) Motoranwendungen ausgelegt und kann beim Einsatz in (kapazitiven) DCC-Modellbahnanwendungen problematisch sein. Die Hauptprobleme stammen von den eingebauten Stromschutzmechanismen, die innerhalb weniger Mikrosekunden reagieren und damit für Modellbahnen zu schnell sind.

Es gibt zwei Arten von Stromschutz: OCP und ITRIP.

## OCP
OCP ist der eingebaute Überstromschutzmechanismus (OverCurrent Protection), der reagiert, wenn der Ausgangsstrom für 3 µs oder länger mehr als 6 (minimal) bis 10 (typisch) Ampere beträgt. Ist der DRV8874 auf Quad-Level 2 konfiguriert (= automatischer Neustart; wie bei der LZ210-TMC), zieht OCP nFAULT auf Low und schaltet das DCC-Ausgangssignal für 2 ms ab. Nach 2 ms erscheint das DCC-Signal wieder, wie in der folgenden Abbildung gezeigt.

<img src="images-nFault/drv8874_01_ocp.png" alt="OCP-Fehler" width="700">

Bleibt die Ursache des OCP-Fehlers bestehen, wird nFAULT nach 2 ms erneut aktiv, sobald der Ausgang wieder eingeschaltet wird, wie in der folgenden Abbildung gezeigt.

<img src="images-nFault/drv8874_02_ocp_repeat.png" alt="Wiederholter OCP-Fehler" width="700">

## ITRIP
ITRIP ist der eingebaute Stromregelmechanismus. Er vergleicht die Spannung über dem Sense-Widerstand (IPROPI) mit VREF. Überschreitet der Ausgangsstrom ITRIP länger als einige Mikrosekunden, wird das DCC-Ausgangssignal abgeschaltet. In Quad-Level 2 (= Cycle-By-Cycle) zieht der DRV8874 nFAULT auf Low, bis zur nächsten Flanke des Eingangssignals. Bei dieser Flanke wird nFAULT freigegeben und der Ausgang folgt wieder dem Eingang, wie in der folgenden Abbildung gezeigt.

<img src="images-nFault/drv8874_03_itrip.png" alt="ITRIP-Fehler" width="700">

Im Gegensatz zu OCP schaltet ITRIP den Ausgang nicht für eine feste Zeit ab, sondern für den Rest des aktuellen Halbbits: höchstens 58 µs bei einem 1-Bit und 100 µs bei einem 0-Bit. Dennoch wird auch in diesem Fall das DCC-Ausgangssignal verfälscht.

Bleibt die Ursache des ITRIP-Fehlers bestehen, wird nFAULT nach jeder Flanke des Eingangssignals erneut aktiv, sobald der Ausgang wieder eingeschaltet wird, also alle 58 oder 100 µs. Dies zeigt die folgende Abbildung.

<img src="images-nFault/drv8874_04_itrip_repeat.png" alt="Wiederholter ITRIP-Fehler" width="700">


## Warum Überstrom?
Auf einer DCC-Modellbahnanlage gibt es mehrere mögliche Ursachen für einen Überstrom:
1. Die Last benötigt mehr Strom, als der DRV8874 liefern kann. Dies kann daran liegen, dass die Lokomotiven und Wagen zu viel Strom aufnehmen. Eine mögliche Abhilfe ist, den Wert des Sense-Widerstands (IPROPI) zu verringern, wodurch die ITRIP-Schwelle steigt.
2. Die Gleisspannung wurde gerade eingeschaltet. Kondensatoren in Lokomotiven und Wagen können beim Laden zunächst einen hohen Einschaltstrom (Inrush) ziehen. Dies kann den Stromschutz vorübergehend auslösen. Sind die Kondensatoren geladen, sinkt der Strom normalerweise wieder auf seinen normalen Wert.
3. Auf dem Gleis liegt ein Kurzschluss vor. In diesem Fall bleibt der Überstrom bestehen, bis der Kurzschluss beseitigt ist. Wichtig ist die Unterscheidung zwischen dauerhaften Kurzschlüssen, wie einer entgleisten Lokomotive, und vorübergehenden Mikrokurzschlüssen, wie kurzzeitigem Kontakt von Spurkränzen am Herzstück von Weichen.

Abhängig von der Ursache sind unterschiedliche Abhilfen möglich. In Fall 1 kann die Hardware angepasst werden, zum Beispiel durch Erhöhen der ITRIP-Schwelle. In Fall 2 können Softwaremaßnahmen wie ein spezielles Inrush-Signal beim Einschalten helfen. In Fall 3 verschwinden Mikrokurzschlüsse von selbst, ein echter Kurzschluss muss jedoch immer erkannt und beseitigt werden. In der Praxis tritt beim DRV8874 häufig eine Kombination aus Fall 1 (hohe mittlere Last) und Fall 2 (kurze Inrush-Last) auf, was die Lösung des Problems erschwert.

## Unterscheidung zwischen OCP- und ITRIP-Fehlern

Bevor Maßnahmen gewählt werden, ist es wichtig festzustellen, ob der Fehler durch OCP oder ITRIP verursacht wurde. Dies lässt sich durch Messen des Abstands zwischen aufeinanderfolgenden nFAULT-Ereignissen feststellen: Erkennt der Prozessor das erste nFAULT-Ereignis, startet er einen Timer und wechselt in einen besonderen Zustand.
1. Innerhalb der nächsten 2 ms treten mehrere neue nFAULT-Ereignisse auf. **Der Fehler wurde durch ITRIP ausgelöst**. Die genaue Anzahl der Ereignisse kann über die Weboberfläche gemeldet werden.
2. Das nächste nFAULT-Ereignis tritt nach etwa 2 ms auf. **Der Fehler wurde durch OCP ausgelöst**. Die Software sollte vermutlich einen Bereich von 1,5 bis 3 ms verwenden, um Zeittoleranzen zu berücksichtigen.
3. Innerhalb der nächsten Millisekunden tritt kein neues nFAULT-Ereignis auf. Das Problem kann durch Mikrokurzschlüsse verursacht worden sein. Es sind keine besonderen Maßnahmen erforderlich, und Statistiken zu diesen Ereignissen können über die Weboberfläche gemeldet werden.

## Maßnahmen bei OCP-Fehlern

Bei einem OCP-Fehler ist es wichtig, zwischen einem Inrush-Problem und einem Kurzschluss auf dem Gleis zu unterscheiden. Ein möglicher Ansatz ist, ein spezielles Inrush-Signal zu senden, das aus einem kurzen EIN-Puls von 2,5 µs besteht, der kürzer ist als die 3 µs, die zum Auslösen von OCP nötig sind, gefolgt von 17,5 µs AUS. Dadurch können sich Kondensatoren aufladen, während verhindert wird, dass OCP nach 2 ms erneut einen nFAULT auslöst.

Dies zeigt die folgende Abbildung. Nachdem der Prozessor das erste nFAULT erkannt hat, startet er einen Timer. Erkennt der Prozessor nach etwa 2 ms ein zweites nFAULT-Ereignis, wechselt er in den Inrush-Modus.

<img src="images-nFault/drv8874_05_inrush.png" alt="Inrush-Muster" width="700">

Wie in der obigen Abbildung zu sehen ist, erscheint das Inrush-Muster nicht sofort, sondern erst, nachdem das aktuelle DCC-Paket vollständig gesendet wurde. Da die Länge eines DCC-Pakets (grob) zwischen 6 und 13 ms liegen kann, ist es möglich, dass noch mehrere nFAULTs auftreten, bevor das Inrush-Muster am Ausgang erscheint.

<img src="images-nFault/drv8874_06_inrush_multi.png" alt="Inrush-Muster" width="700">

Sobald das Inrush-Muster erscheint, werden die Kondensatoren geladen. Für einen 1000-µF-Kondensator kann die erforderliche Ladezeit 10 bis 20 ms betragen. Das Inrush-Muster sollte daher mindestens 20 ms auf dem Gleis anliegen, bevor es eine Wirkung zeigt, danach wird der normale DCC-Betrieb wieder aufgenommen. Treten danach erneut nFAULTs auf, liegt wahrscheinlich ein Kurzschluss vor und das DCC-Signal wird abgeschaltet (siehe die Empfehlung unten).

Es ist jedoch wichtig, dass das Inrush-Muster nicht zu lange dauert. Da die Pulse kürzer sind als die OCP-Entprellzeit, löst OCP nie aus, wodurch die Gefahr besteht, dass der DRV8874 überhitzt. Weil die Pulse sehr kurz sind, kann sich die Wärme möglicherweise nicht schnell genug ausbreiten, um die interne thermische Abschaltung (TSD) auszulösen, und der Chip kann dadurch zerstört werden.

Daher könnte die Software die Spannung über dem Sense-Widerstand (IPROPI) überwachen, um zu prüfen, ob der Strom hoch bleibt oder langsam abnimmt. Bleibt er hoch, liegt wahrscheinlich ein Kurzschluss auf dem Gleis vor und der Prozessor sollte das Ausgangssignal abschalten. Nimmt der Strom ab, wurde OCP wahrscheinlich durch den Einschaltstrom von Kondensatoren ausgelöst, und der Prozessor kann länger im Inrush-Modus bleiben.

Die Messung der Spannung über dem Sense-Widerstand ist nicht trivial. Die Pulse sind sehr kurz, und die Form des Signals am ADC-Eingang hängt davon ab, ob der im DRV8874-Datenblatt vorgeschlagene 10-nF-Kondensator bestückt ist. Die folgende Abbildung veranschaulicht dies für Stromspitzen von 10 A (idealisiert: die interne IPROPI-Klemmung begrenzt die tatsächliche Spannung). Ohne den Kondensator sind die Spitzen hoch, dauern aber weniger als 1 µs. Der ADC kann sie nur im Free-Running-Modus erfassen, und nicht jeder Puls wird erfasst: Bei 20 Samples mit 500 kS/s (40 µs) sollte der Prozessor den höchsten gemessenen Wert auswählen. Mit dem Kondensator beträgt die maximale Spannung nur etwa 0,4 V und fällt mit einer Zeitkonstante von etwa 13 µs ab.

<img src="images-nFault/drv8874_07_ipropi.png" alt="V(IPROPI) mit und ohne 10-nF-Kondensator" width="700">

In der Praxis kann eine Kombination aus hoher Dauerlast und einem kurzen kapazitiven Einschaltstrom auftreten. Beträgt die Dauerlast zum Beispiel 2 A und wird eine neue Lokomotive mit einem 1000-µF-Kondensator auf das Gleis gesetzt, kann OCP ausgelöst werden. In diesem Fall reicht das Aktivieren des Inrush-Musters möglicherweise nicht aus, um den Kondensator zu laden und gleichzeitig weiterhin 2 A zu liefern. Dadurch verhält sich das System, als läge ein Kurzschluss auf dem Gleis vor.

## Maßnahmen bei ITRIP-Fehlern

Auch wenn nFAULT durch ITRIP ausgelöst wird, kann es eine Kombination von Ursachen geben.
Eine mögliche Ursache ist, dass Kondensatoren in Lokomotiven einen hohen Inrush-Strom verursachen, der Widerstand von Verkabelung und Gleisen diesen Strom jedoch unter den OCP-Pegel begrenzt. Wie in den ITRIP-Abbildungen oben zu sehen ist, wird das DCC-Ausgangssignal nach etwa 3 µs abgeschaltet, alle 58 oder 100 µs. Sofern keine andere nennenswerte Last auf dem Gleis liegt, kann ein 1000-µF-Kondensator nach einigen zehn ms geladen sein. In diesem Fall verschwinden die nFAULTs von selbst. Auf einer realen Anlage ist jedoch eine Kombination aus kapazitiver und ohmscher Last wahrscheinlicher, was zu ähnlichen Problemen wie oben beschrieben führt, sodass die nFAULTs möglicherweise nie verschwinden.

Bei ITRIP-Fehlern kann das Erzeugen eines speziellen Inrush-Signals von gewissem Nutzen sein. Ohne das Inrush-Muster beträgt das Tastverhältnis etwa 2 % bis 4 %, je nachdem, ob ein 1-Bit (58 µs) oder ein 0-Bit (100 µs) gesendet wird. Mit dem Inrush-Muster steigt das Tastverhältnis auf 12,5 %, sodass Kondensatoren schneller geladen werden. Sobald die nFAULTs verschwinden, sollte der Prozessor den Inrush-Modus verlassen und zum normalen DCC-Betrieb zurückkehren. Verschwinden die nFAULTs nicht, ist keine Softwarelösung möglich. Dieser Fall sollte in die Weboberfläche gemeldet werden und das DCC-Signal soll abgeschaltet werden.

Dauerhafte ITRIP-Fehler lassen sich nur durch Änderungen an der Hardware beheben. Es gibt drei Möglichkeiten:
1. VREF erhöhen. In der Praxis ist dies meist nicht möglich, weil VREF oft bereits an die maximale Spannung angepasst ist, die der ADC verarbeiten kann (auf dem LZ210-TMC-Board beträgt VREF 3 V).
2. Den Sense-Widerstand (IPROPI) verringern und dadurch den maximalen Strom erhöhen, bevor ITRIP anspricht. Die Pololu- und AliExpress-Module haben einen Sense-Widerstand von 2,49 kΩ, was bei einem VREF von 3 V zu einem ITRIP-Wert von knapp 2,7 A führt. Dieser Wert ist ziemlich niedrig, aber nachvollziehbar, da diese Module auch mit einem VREF von 5 V betrieben werden können, was 4,4 A ergibt. Überwacht die CPU ohnehin den mittleren Stromverbrauch und greift ein, wenn die mittlere Verlustleistung 2,5 bis 3 W überschreitet, ist es ratsam, den ITRIP-Wert auf etwa 6 A zu erhöhen. Bei einem VREF von 3 V kann der Sense-Widerstand durch einen 1-kΩ-Widerstand ersetzt oder ein zusätzlicher 1,8-kΩ-Widerstand parallel zum 2,49-kΩ-Sense-Widerstand auf dem Modul angebracht werden.
3. Einen Kondensator parallel zum Sense-Widerstand (IPROPI) anbringen. Das Datenblatt schlägt einen 10-nF-Kondensator vor, um Transienten zu filtern, damit die Stromregelung nicht vorzeitig anspricht. Als Nebeneffekt verzögert er ITRIP und glättet die Spannung, wodurch sich der ADC-Wert leichter auslesen lässt.

Zur Veranschaulichung zeigt die folgende Abbildung (VREF = 3 V, R_sense = 1,35 kΩ, ITRIP = 4,9 A) die Wirkung des 10-nF-Kondensators bei einem mittleren Strom von 3 A und 5 A. Wie auf der rechten Seite der Abbildung zu sehen ist (der 5-A-Fall, also Überlast), spricht ITRIP ohne den 10-nF-Kondensator bereits nach wenigen µs an, mit dem 10-nF-Kondensator dagegen deutlich später. Das Messen der Spannung über dem Sense-Widerstand mit dem ADC ist ohne Kondensator recht schwierig, mit Kondensator jedoch machbar, sofern der ADC im Free-Running-Modus läuft und die höchsten Werte erfasst.

<img src="images-nFault/drv8874_08_cap_compare.png" alt="V(IPROPI) mit und ohne 10-nF-Kondensator" width="1000">

## Empfehlung

Der beste Ansatz ist, ITRIP-bezogene nFAULTs so weit wie möglich zu vermeiden. Auf dem LZ210-TMC-Board lässt sich dies erreichen, indem der Sense-Widerstand (IPROPI) auf 1 kΩ gesetzt wird oder indem ein zusätzlicher 1,8-kΩ-Widerstand parallel zum 2,49-kΩ-Sense-Widerstand auf dem Modul angebracht wird. In beiden Fällen liegt der ITRIP-Wert über 6 A. Der 10-nF-Kondensator kann bestückt werden, ist aber nicht zwingend. Der ADC kann den Strom dann zuverlässig bis etwa 5 bis 6 A messen, und die Software sollte die Verlustleistung im DRV8874 abschätzen (I² × RDS(on), über die Zeit gemittelt, um die thermische Trägheit des Chips nachzubilden) und bei Bedarf das DCC-Signal abschalten, um Überhitzung zu verhindern.

Tritt ein nFAULT auf, unabhängig davon, ob er durch OCP oder ITRIP verursacht wurde, wechselt der Prozessor in einen Fehlerzustand und der ADC wird nicht mehr gelesen. Während der ersten 1,5 ms werden neue nFAULTs ignoriert, da sie durch einen Mikrokurzschluss verursacht sein können. Ab 1,5 ms überwacht der Prozessor nFAULT wieder (Hinweis: Der nFAULT-Pegel kann vom ersten nFAULT noch Low sein). Wurde der nFAULT durch OCP verursacht, wird der nächste nFAULT 500 µs nach Wiederaufnahme der Überwachung erwartet (da die Sperrzeit 2 ms beträgt); wurde der nFAULT durch ITRIP verursacht, wird der nächste nFAULT innerhalb weniger hundert µs erwartet. Tritt in den folgenden 20 ms (konfigurierbar) kein neuer nFAULT auf, war die Ursache ein Mikrokurzschluss, und der Prozessor kehrt zum Normalbetrieb zurück. Treten ein oder mehrere neue nFAULTs auf, wird der Inrush-Modus für 35 ms aktiviert (konfigurierbar bis 100 ms). Da das Inrush-Muster erst auf dem Gleis erscheint, nachdem das aktuelle DCC-Paket vollständig gesendet wurde (was etwa 13 ms dauern kann), liegt das Muster mindestens 20 ms auf dem Gleis an. Alle zusätzlichen nFAULTs, die in dieser Zeit auftreten, werden ignoriert.

Nachdem das Inrush-Muster gesendet wurde, werden wieder normale DCC-Pakete gesendet. Der Prozessor überwacht nFAULTs weitere 5 ms (konfigurierbar, aber mindestens die OCP-Sperrzeit von 2 ms). Tritt kein nFAULT auf, kehrt der Prozessor zum normalen DCC-Betrieb zurück. Treten neue nFAULTs auf, liegt wahrscheinlich ein Kurzschluss vor: Das DCC-Signal muss abgeschaltet und nach beispielsweise 1 s (konfigurierbar) erneut versucht werden.

<img src="images-nFault/drv8874_09_state_machine.png" alt="Zustandsdiagramm" width="1000">
