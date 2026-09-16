# CDE Booster Interface

Unlike DCC itself, the CDE booster interface is not defined by any RCN or NMRA standard. Lenz, the developer of this interface, has also published very little technical information on the subject. However, various schematics for the booster side of the interface can be found online, including the [Z21PG Booster](https://pgahtow.de/w/Booster#/media/Datei:Booster_v2.png) and [Paco’s BoosteR-CDE](https://usuaris.tinet.cat/fmco/railcom_en.html#booster).

The origin of the name **CDE** is not documented. On some internet forums, it is suggested that the letters stand for **C**ommon, **D**ata and **E**rror. That could be a possible explanation, but it does not fit well with current implementations. In modern implementations, **C** and **D** are symmetrical in relation to each other and thus form a differential signal. An alternative possibility is that Lenz had already used the letters **A** and **B** for the XpressNet interface. After all, the RS-485 signals on which XpressNet is based are often designated as **A** and **B**. Based on that reasoning, **CDE** could simply be the next sequence of letters.

## The CD signal

The **C** and **D** connections carry the DCC signal used by the booster to generate the track signal. Measurements taken on an **LZ100**, **LZV100** and **LZV200** show that **C** and **D** switch alternately between approximately 0 V and +12 V relative to ground. The two signals switch in opposite phase: when **C** is at 12 V, **D** is at 0 V, and vice versa. The voltage difference between **C** and **D** therefore alternates between approximately +12 V and −12 V.

The oscilloscope waveform below shows the **CD** signal of a LZV200.

![CD signal LZV200](images/LZV200-CDE.jpeg)

It is likely that Lenz uses an internal H-bridge to generate this **CD** signal. Without an H-bridge, additional circuitry would be necessary to generate both +12 V and −12 V relative to ground. This would require an additional DC converter, making the system unnecessarily complex and therefore more expensive. Note that the H-bridge only needs to supply a limited amount of power. Even on large model railway layouts with multiple boosters, the required current is expected to remain limited to a few hundred milliamps.

A legitimate question is whether the DCC voltage itself, or a 5 V differential signal, would suffice to generate the **CD** signal. To answer that question, we need to take a closer look at the input side of a modern booster. The schematic below shows the input side of the Paco booster.

![CD booster ingang](images/CD-Basic-Schematics.png)

The schematic contains two high speed optocouplers (6N136). The upper one is active when **C** is positive relative to **D**; the lower one when **C** is negative relative to **D**. This two-optocoupler configuration is necessary because there are three distinct states to consider: **C** higher than **D**, **C** lower than **D**, and the RailCom cut-out, where **C** is equal to **D**.


Crucial to this circuit is the 1 kΩ resistor connected in series with the **C** signal, which is usually a 250 mW resistor. At a **CD** voltage of 12 V, approximately 10.4 V is dropped across this resistor (a fixed forward voltage of V<sub>F</sub> = 1.6 V is dropped across the optocoupler). The resistor must then dissipate nearly 110 mW. At **CD** = 16 V, this rises to over 200 mW. At a voltage of 18 V or higher, a 250 mW resistor will be overloaded.

At a voltage of **5 V**, only **3.4 V** is dropped across the resistor, and the current through the optocoupler LED is therefore 3.4 mA. This is too low for the LEDs in the optocoupler to switch reliably at DCC speed.

## The E-signal

The **E** signal may be used by a booster to indicate a short circuit. To this end, both the Lenz and Paco boosters use an additional optocoupler (such as a PC817, see the diagram above). In the event of a short circuit, this optocoupler is activated, allowing current to flow from **E** to **D**. It is also possible to connect an emergency stop switch between **E** and **M** (Masse = Ground), as illustrated in the figure below taken from the Lenz LZV100 manual, allowing current to flow from **E** to **M**.

![Not Aus](images/CDE-NOTAUS.png)

### Principle of Short-Circuit Detection

The diagram below shows a circuit that enables a DCC command station to detect a short-circuit or emergency stop signal.

![Short-Circuit Detection](images/CDE-Basic-Schematics.png)

When the booster detects a short circuit, it activates the (PC817) optocoupler. If the voltage at **D** is higher than that at **E**, the diode (for example, a 1N4148) prevents current from flowing from **D** to **E**, leaving the voltage at **E** unaffected.
If the voltage at **D** is lower than that at **E**, current can flow from the booster’s **E** connection. That current flows via pin 4 (collector) and pin 3 (emitter) of the optocoupler and then back through the diode to the D connection. The booster therefore pulls **E** towards **D** via the optocoupler.

<B>The circuit therefore has a *low signal* during a short circuit or emergency stop, and a *high signal* during normal operation.</B>

### Normal operation
If no short circuit is reported, the circuit operates as follows. When the **C** signal is high, the 100 nF capacitor (C701) is charged via the 4.7 kΩ resistor (R703). The time required to charge the capacitor to its maximum voltage depends on the voltage and the precise shape of the DCC signal, but is in the order of a few milliseconds. The 1N4148 diode (D701) prevents the capacitor from discharging when the **C** signal is low.

To obtain a suitable input voltage for a 3V3 microcontroller, the voltage across the capacitor is reduced to approximately 25% of the original voltage using a voltage divider comprising 56 kΩ (R704) and 18 kΩ (R705) resistors. At a maximum **C** voltage of **12 V**, the node charges to: *V<sub>node</sub> = (12 − 0,7) × 18 / (56 + 18 + 4,7) ≈ 2,5 V*. The 100 pF capacitor (C702), together with the resistor divider, forms a low-pass filter with a cut-off frequency of around 100 kHz. This filter suppresses any high-frequency interference at the microcontroller’s input.

If a 5 V microcontroller is used instead of a 3.3 V microcontroller, the 18 kΩ resistor must be replaced with a 47 kΩ resistor.

### Short circuit

During a short circuit or emergency stop, the 100 nF capacitor may discharge via the **E** terminal, the optocoupler and the diode in the booster, to **D**. This discharge only occurs when the **D** signal is low, so for a maximum of 58 (DCC 1) or 100 (DCC 0) µs each time.

The voltage at **E** drops to the sum of the diode’s forward voltage and the optocoupler’s saturation voltage (*V*<sub>CE(sat)</sub>). This is typically around 0.7 to 1.1 V: approximately 0.6 V for the diode and 0.1 to 0.5 V for the optocoupler, depending on the degree to which the phototransistor is saturated. As the voltage divider reduces the voltage across the capacitor to approximately 25% of its original value, the voltage at the microcontroller input during a short circuit is typically only 0.2 to 0.4 V.

In the event of an EMERGENCY STOP, where **E** is connected directly to ground, the voltage at the microcontroller input is rapidly pulled down to 0 V.  At 12 V, this causes a current of less than 3 mA to flow through the 4.7 kΩ resistor. This current is low enough to prevent thermal overload of the resistor.
