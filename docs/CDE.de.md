# CDE-Booster-Schnittstelle

Im Gegensatz zu DCC selbst ist die CDE-Booster-Schnittstelle durch keinen RCN- oder NMRA-Standard definiert. Lenz, der Entwickler dieser Schnittstelle, hat zudem nur wenig technische Informationen dazu veröffentlicht. Im Internet finden sich jedoch verschiedene Schaltpläne für die Booster-Seite der Schnittstelle, darunter der [Z21PG Booster](https://pgahtow.de/w/Booster#/media/Datei:Booster_v2.png) und [Paco’s BoosteR-CDE](https://usuaris.tinet.cat/fmco/railcom_en.html#booster).

Die Herkunft des Namens **CDE** ist nicht dokumentiert. In einigen Internetforen wird vermutet, dass die Buchstaben für **C**ommon, **D**ata und **E**rror stehen. Das könnte eine mögliche Erklärung sein, passt jedoch nicht gut zu aktuellen Implementierungen. In modernen Implementierungen sind **C** und **D** zueinander symmetrisch und bilden somit ein Differentialsignal. Eine alternative Möglichkeit ist, dass Lenz die Buchstaben **A** und **B** bereits für die XpressNet-Schnittstelle verwendet hatte. Schließlich werden die RS-485-Signale, auf denen XpressNet basiert, oft als **A** und **B** bezeichnet. Nach dieser Logik könnte **CDE** einfach die nächste Buchstabenfolge sein.

## Das CD-Signal

Die Anschlüsse **C** und **D** führen das DCC-Signal, das vom Booster zur Erzeugung des Gleissignals verwendet wird. Messungen an einem **LZ100**, **LZV100** und **LZV200** zeigen, dass **C** und **D** relativ zur Masse abwechselnd zwischen etwa 0 V und +12 V wechseln.
Die beiden Signale wechseln in entgegengesetzter Phase: Wenn **C** bei 12 V liegt, liegt **D** bei 0 V und umgekehrt. Die Spannungsdifferenz zwischen **C** und **D** wechselt daher zwischen etwa +12 V und −12 V.

Die folgende Oszilloskopabbildung zeigt das **CD**-Signal eines LZV200.

![CD signal LZV200](images/LZV200-CDE.jpeg)

Es ist wahrscheinlich, dass Lenz eine interne H-Brücke verwendet, um dieses **CD**-Signal zu erzeugen. Ohne eine H-Brücke wären zusätzliche Bauteile erforderlich, um sowohl +12 V als auch −12 V relativ zur Masse zu erzeugen. Dies würde einen zusätzlichen DC-Wandler erfordern, was das System unnötig komplex und damit teurer machen würde. Zu beachten ist, dass die H-Brücke nur eine begrenzte Leistung liefern muss.
Selbst bei großen Modelleisenbahnanlagen mit mehreren Verstärkern dürfte der benötigte Strom auf wenige hundert Milliampere begrenzt bleiben.

Es stellt sich die Frage, ob die Gleisspannung selbst oder ein 5-V-Signal nicht ausreichen würde, um das CD-Signal zu erzeugen. Um dies zu klären, muss die Eingangsseite eines modernen Boosters genauer betrachtet werden. Der folgende Schaltplan zeigt die Eingangsseite eines Paco-Boosters.

![CD booster ingang](images/CD-Basic-Schematics.png)

Die Schaltplan enthält zwei Hochgeschwindigkeits-Optokoppler (6N136). Der obere ist aktiv, wenn **C** relativ zu **D** positiv ist; der untere, wenn **C** relativ zu **D** negativ ist. Diese Konfiguration mit zwei Optokopplern ist erforderlich, da drei unterschiedliche Zustände zu berücksichtigen sind: **C** höher als **D**, **C** niedriger als **D** und die RailCom-Cutout, bei der **C** gleich **D** ist.


Entscheidend bei dieser Schaltung ist der in Reihe mit dem **C**-Signal geschaltete 1-kΩ-Widerstand, bei dem es sich in der Regel um einen 250-mW-Widerstand handelt. Bei einer **CD**-Spannung von 12 V fällt über diesem Widerstand ein Spannung von etwa 10,4 V (der Optokoppler hat eine feste Durchlassspannung von V<sub>F</sub> = 1,6 V). Der Widerstand muss also fast 110 mW Verlustleistung aushalten. Bei **CD** = 16 V steigt diese auf über 200 mW. Bei einer Spannung von 18 V oder höher wird der 250-mW-Widerstand überlastet.

Bei einer Spannung von **5 V** fallen am Widerstand nur **3,4 V** ab, und der Strom durch die Optokoppler-LED beträgt somit 3,4 mA. Dies ist zu gering, um die LEDs im Optokoppler bei DCC-Geschwindigkeit zuverlässig zu betreiben.

## Das E-Signal

Das **E**-Signal darf von einem Booster zur Anzeige eines Kurzschlusses verwendet werden. Zu diesem Zweck verwenden sowohl der Lenz- als auch der Paco-Booster einen zusätzlichen Optokoppler (z. B. einen PC817, siehe Abbildung oben). Im Falle eines Kurzschlusses wird dieser Optokoppler aktiviert, sodass Strom von **E** nach **D** fließen kann. Es ist auch möglich, einen Not-Aus-Schalter zwischen **E** und **M** (Masse) anzuschließen, wie in der folgenden Abbildung aus dem Lenz LZV100-Handbuch dargestellt, sodass Strom von **E** nach **M** fließen kann.

![Not Aus](images/CDE-NOTAUS.png)

### Prinzip der Kurzschlusserkennung

Das folgende Schema zeigt eine Schaltung, mit der eine DCC-Zentrale einen Kurzschluss oder ein Not-Aus-Signal erkennen kann.

![Short-Circuit Detection](images/CDE-Basic-Schematics.png)

Wenn der Booster einen Kurzschluss erkennt, aktiviert er den Optokoppler (PC817). Ist die Spannung an **D** höher als die an **E**, verhindert die Diode (z. B. eine 1N4148), dass Strom von **D** nach **E** fließt, sodass die Spannung an **E** unverändert bleibt.
Ist die Spannung an **D** niedriger als die an **E**, kann Strom vom **E**-Anschluss des Boosters fließen. Dieser Strom fließt über Pin 4 (Kollektor) und Pin 3 (Emitter) des Optokopplers und dann zurück durch die Diode zum D-Anschluss. Der Booster zieht daher die Spannung an **E** über den Optokoppler in Richtung **D**.

<B>Die Schaltung liegt daher bei einem Kurzschluss oder Not-Aus auf *Low* und im Normalbetrieb auf *High*.</B>

### Normalbetrieb
Wird kein Kurzschluss gemeldet, funktioniert die Schaltung wie folgt: Wenn das **C**-Signal auf „High“ steht, wird der 47-nF-Kondensator (C701) über den 4,7-kΩ-Widerstand (R703) geladen. Die Zeit, die benötigt wird, um den Kondensator auf seine maximale Spannung aufzuladen, hängt von der Spannung und der genauen Form des DCC-Signals ab, liegt jedoch in der Größenordnung von wenigen Millisekunden. Die Diode 1N4148 (D701) verhindert, dass sich der Kondensator entlädt, wenn das **C**-Signal niedrig ist.

Um eine geeignete Eingangsspannung für einen 3V3-Mikrocontroller zu erhalten, wird die Spannung am Kondensator mithilfe eines Spannungsteilers aus den Widerständen 120 kΩ (R704) und 47 kΩ (R705) auf etwa 28 % der ursprünglichen Spannung reduziert. Bei einer maximalen **C**-Spannung von **12 V** wird der Knoten auf folgende Spannung aufgeladen: *V<sub>node</sub> = (12 − 0,7) × 47 / (120 + 47 + 4,7) ≈ 3,1 V*. Die 3,3-V-Zenerdiode (D702) schützt den Eingang des Mikrocontrollers vor möglichen überhöhten Spannungsspitzen. Der 100-pF-Kondensator (C702) bildet zusammen mit dem Widerstandsteiler einen Tiefpassfilter mit einer Grenzfrequenz von etwa 50 kHz. Dieser Filter unterdrückt hochfrequente Störungen am Eingang des Mikrocontrollers.

Wird anstelle eines 3,3-V-Mikrocontrollers ein 5-V-Mikrocontroller verwendet, muss der 47-kΩ-Widerstand durch einen 68-kΩ-Widerstand und die 3,3-V-Zenerdiode (D702) durch eine 5-V-Zenerdiode ersetzt werden.

### Kurzschluss

Bei einem Kurzschluss oder einem Not-Aus kann sich der 47-nF-Kondensator über den Anschluss **E**, den Optokoppler und die Diode im Booster nach **D** entladen.
Diese Entladung findet nur statt, wenn das **D**-Signal auf Low ist, also jeweils für maximal 58 (DCC 1) oder 100 (DCC 0) µs.

Die Spannung an **E** sinkt auf einen Wert, der der Summe aus der Durchlassspannung der Diode und der Sättigungsspannung des Optokopplers entspricht (*V* <sub>CE(sat)</sub>). Diese liegt typischerweise bei etwa 0,7 bis 1,1 V: etwa 0,6 V für die Diode und 0,1 bis 0,5 V für den Optokoppler, je nachdem, inwieweit der Fototransistor gesättigt ist. Da der Spannungsteiler die Spannung am Kondensator auf etwa 28 % ihres ursprünglichen Wertes reduziert, beträgt die Spannung am Eingang des Mikrocontrollers während eines Kurzschlusses typischerweise nur 0,2 bis 0,4 V.

Im Falle eines NOT-AUS, bei dem **E** direkt mit Masse verbunden wird, wird die Spannung am Eingang des Mikrocontrollers schnell auf 0 V heruntergezogen. Bei 12 V fließt dadurch ein Strom von weniger als 3 mA durch den 4,7-kΩ-Widerstand. Dieser Strom ist klein genug, um eine thermische Überlastung des Widerstands zu verhindern.
