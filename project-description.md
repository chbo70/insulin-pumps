# Closed-Loop Blood-Glucose Control via an Artificial Pancreas based on the Bergman Minimal Model

## 1. Fundierte Projektbeschreibung (Pitch für das Gespräch)

**Klinische Motivation (Pain-Point):**
Bei Patienten mit Typ-1-Diabetes ist die endogene Insulinproduktion der $\beta$-Zellen im Pankreas gestört oder inexistent. Dies führt zu einem Verlust der Blutzuckerregulation. Das klinische Ziel ist es, den Blutzuckerspiegel nach Störgrößen (Mahlzeiten / "Glucose Spikes") wieder auf einen gesunden Basalwert (z. B. 80-100 mg/dL) zurückzuführen, um akute (Hypoglykämie/Koma) und chronische (Organschäden durch Hyperglykämie) Komplikationen zu verhindern.

**Projektziel (Objective):**
Entwicklung und Simulation einer künstlichen Bauchspeicheldrüse (Artificial Pancreas). Das System besteht aus:
1. **Plant Model:** Das etablierte physiologische *Bergman Minimal Model*, welches die Dynamik zwischen Blutzucker, Plasmainsulin und der verzögerten Insulinwirkung im Gewebe abbildet.
2. **Digital Controller:** Ein zeitdiskreter PID-Regler, der als Insulinpumpe agiert. Er berechnet basierend auf fehlerbehafteten (quantisierten) Sensordaten die exogene Insulinzufuhr.

**Quantitative Analyse (Gemäß Vorlesungs-Anforderungen):**
Das Projekt evaluiert die Robustheit des Closed-Loop-Systems hinsichtlich *Disturbance Rejection* (Ausregelung eines simulierten Kohlenhydrat-Spikes). Dabei werden transiente Kriterien (Rise Time, Settling Time, Steady-State Error) quantifiziert sowie die Auswirkungen von typischen Hardware-Limitationen moderner Continuous Glucose Monitors (CGM) untersucht: Abtastzeiten ($T_s$) und ADC-Quantisierungsfehler (Sensorauflösung).

---

## 2. Das mathematische Modell (Die ODEs)

Die Basis bilden die Herleitungen von Bergman et al. (1979) und die retrospektive Zusammenfassung (Bergman, 2021). Das klassische Modell wurde um die für die Regelungstechnik essenziellen externen Einflüsse (Störgröße und Stellgröße) erweitert.

Das System wird durch drei gekoppelte gewöhnliche Differentialgleichungen (ODEs) beschrieben:

### 2.1 Glukose-Kinetik (Blood Glucose Dynamics)
Diese Gleichung beschreibt die zeitliche Änderung der Blutzuckerkonzentration $G(t)$. Sie hängt vom Glukoseabbau durch das Insulin im "Remote Compartment" $X(t)$, der basalen Glukoseproduktion der Leber und der externen Störgröße (Mahlzeit) ab.

$$\frac{dG(t)}{dt} = -(p_1 + X(t))G(t) + p_1 G_b + D(t)$$

* $G(t)$: Blutzuckerkonzentration [mg/dL]
* $G_b$: Basaler Blutzuckerspiegel (Steady-State, ca. 81-100 mg/dL)
* $p_1$: Glukose-Effektivität (rate of insulin-independent glucose utilization) [min$^{-1}$]
* **$D(t)$:** Die Störgröße (Disturbance) – repräsentiert den Glukose-Einstrom durch eine Mahlzeit ("Spike").

### 2.2 Insulinwirkung im Gewebe (Insulin Action / Remote Compartment)
Bergman (1979) fand heraus, dass Insulin nicht instantan im Blutplasma wirkt, sondern verzögert im Interstitium ("Remote Compartment"). Diese Verzögerung ist der kritischste Teil für das Regler-Design.

$$\frac{dX(t)}{dt} = -p_2 X(t) + p_3(I(t) - I_b)$$

* $X(t)$: Insulinwirkung auf die Glukoseaufnahme (proportional zur Insulinkonzentration im Interstitium) [min$^{-1}$]
* $I(t)$: Plasmainsulinkonzentration [$\mu$U/mL]
* $I_b$: Basales Plasmainsulin [$\mu$U/mL]
* $p_2$: Abbaurate der Insulinwirkung [min$^{-1}$]
* $p_3$: Transportrate von Insulin aus dem Blutplasma in das "Remote Compartment" [min$^{-2}$ per $\mu$U/mL]

### 2.3 Plasmainsulin-Kinetik (Plasma Insulin Dynamics)
Diese Gleichung beschreibt, wie sich das Insulin im Blutplasma verändert. Hier greift unser Regler (die Pumpe) ein.

$$\frac{dI(t)}{dt} = -n(I(t) - I_b) + \frac{U(t)}{V_I}$$

* $n$: Fraktionale Clearance-Rate des Insulins aus dem Blutplasma [min$^{-1}$]
* $V_I$: Insulin-Verteilungsvolumen [L]
* **$U(t)$:** Die Stellgröße (Control Input) – die exogene Insulinzufuhr durch unsere Pumpe (gesteuert durch den PID-Regler) [$\mu$U/min].

---

## 3. Implementierungsplan (Bezug zu Hausübung 7)

1. **Die `plant_model` Funktion:** Die Funktion `bergman_model(t, y, U_pump, D_meal)` definiert, welche den Array `[dGdt, dXdt, dIdt]` für den Solver zurückgibt.
2. **Die Simulations-Schleife:** Es wird eine Zeit-Schleife mit einer definierten Abtastzeit (z. B. $T_s = 5$ Minuten) implementiert, die den realen Messintervallen eines CGM-Sensors entspricht.
3. **Der diskrete Regler:** In der Schleife wird das Integral des Fehlers berechnet (`error_int += error_v * dt_control`). Der PID-Algorithmus ermittelt die Insulin-Dosis $U(t)$, welche konstant für das nächste Intervall (Zero-Order Hold) an den `solve_ivp` Solver übergeben wird.
4. **Störgröße & Hardware-Limits:** Ein `if`-Statement simuliert den Mahlzeiten-Spike $D(t)$. Zusätzlich wird eine Quantisierungsfunktion vorgeschaltet, um ADC-Auflösungsfehler (z. B. Messungen nur in 1 mg/dL oder 5 mg/dL Schritten) abzubilden.