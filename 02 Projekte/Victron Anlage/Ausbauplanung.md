---
tags: [victron, planung, ausbau, batterie, notstrom]
status: aktiv
date: 2026-07-07
updated: 2026-08-23
---

# Ausbauplanung

Vorhaben, die noch nicht umgesetzt sind — Whole-home-Backup, Batterie-Ausbau,
Winter-Messkampagne, zurückgestellte Flow-Erweiterungen.

> [!abstract] Stand der Vorhaben
> | Vorhaben | Stand | Blockiert durch |
> |---|---|---|
> | Batterie 15 → 30 kWh | beschlossen | — |
> | Whole-home-Backup (AC-Out) | geplant | Elektrofachkraft, Verteilerumbau |
> | Fronius auf AC-Out | ⛔ nicht möglich | braucht 60–100 kWh Batterie |
> | Multi-Standby-Automatik | zurückgestellt | Winter-Messung nötig, Nutzen fraglich |
> | Hochlast-Gauge | zurückgestellt bis Herbst | zu wenig Hochlast-Fenster im Sommer |

> [!todo] Wenn der Batterie-Ausbau kommt
> - [ ] Gesamtkapazität im WatchMon Core neu einstellen — sonst rechnen **alle
>       SoC-Automatiken falsch**
> - [ ] Ladedeckel-Automatik auf kWh-Restbedarf statt SoC-Prozent umstellen
> - [ ] Verlustgerade neu regressieren (`P_fix` muss gleich bleiben — tut es das
>       nicht, war ein Teil des „Leerlaufs" ein Messoffset)

Verwandt: [[Energiebilanz und Kennzahlen]] · [[Wirtschaftlichkeit]] · [[Wartung und Betrieb]]

## Planung: Whole-home-Backup & Batterie-Ausbau (2026-07-07)

**Ziel:** Notstromfähigkeit für das ganze Haus — Hauslasten über den **AC-Out** der MultiPlus führen (aktuell hängt alles am AC-IN → kein Backup).

### Ist-Stand / Topologie (Auslegungsdaten)
- **3× MultiPlus-II 48/10000** (3-Phasen, EIN VE.Bus-System): je **8.000 W** Dauerleistung @25 °C → **~24 kW** Insel gesamt (bei 40 °C ~19–20 kW); **Transfer-Relais 100 A/Phase**; Wechselrichter-Ausgangsstrom 36 A/Phase.
- **Fronius Symo 17.5-3-M am AC-IN** → netzabhängig, **im Inselbetrieb NICHT verfügbar**.
- **2× SmartSolar MPPT 250/70** (~5 kWp, DC an der Batterie) → einzige PV-Quelle im Notstrom.
- **Batterie:** aktuell ~15 kWh (280 Ah/48 V), **im Ausbau auf ~30 kWh** (2. Pack à 15 kWh, parallel).
- **BMS:** **1× WatchMon Core + 2× CellMate K9** = EIN BMS für die Gesamtbatterie → ein CAN-BMS am Cerbo (DVCC sauber), ein UDP-Datenstrom fürs Monitoring. (Frühere Fehlannahme „zwei getrennte K7" verworfen.)

### Bewertung
- **Leistung reicht locker:** Die 3× 10000 tragen das ganze Haus inkl. 11-kW-Wallbox. Durchgangsstrom (100 A/Phase) liegt über jedem üblichen Hausanschluss, Inselleistung 24 kW ist komfortabel. → Hürden „Durchgangsstrom" und „Inselleistung" **entfallen**.
- **Fronius-Umzug auf AC-Out (für PV im Notstrom): aktuell NICHT möglich.** Grund = Batterie-Aufnahmefähigkeit: 17,5 kW ÷ 48 V ≈ **365 A** Ladestrom bei plötzlichem Lastabwurf; sicher bei 0,2–0,3 C → **~1.200–1.800 Ah ≈ 60–100 kWh** Batterie nötig. 30 kWh = ~0,65 C → **noch zu klein**. Schwelle für Umzug: **~60–100 kWh**.
- **Notstrom-Konzept jetzt:** Whole-home über AC-Out, **Fronius bleibt AC-IN**, PV im Notstrom nur ~5 kWp (MPPT), Autonomie = Batteriekapazität (~30 kWh) + MPPT-Nachladung. → **Lastabwurf für Wallbox/Großverbraucher** im Inselbetrieb einplanen (sonst Batterie in <1 h leer).

### Batterie-Ausbaupfad
- Jetzt +15 kWh → 30 kWh: verdoppelte Notstrom-Autonomie, niedrigere C-Raten, längere Lebensdauer.
- Erweiterung = **weitere CellMate K9 am selben WatchMon Core** (bleibt EIN BMS). Nach dem Ausbau: **Gesamtkapazität im Core neu einstellen** (SoC/Coulomb), **Shunt misst Gesamtstrom** (Position!), gleicher SoC/Spannung beim Parallelverbinden, **Sicherung pro Pack**, Balancing über alle Zellen.
- Ab ~60–100 kWh wird der Fronius-Umzug auf AC-Out möglich → volle 22,5 kWp PV auch im Notstrom.

### Offene Punkte für die Elektrofachkraft / Fachplanung
- Verteilerumbau: Netz → AC-In → AC-Out → Hausverteilung, mit **Wartungs-Bypass** (Haus direkt aufs Netz bei Multi-Wartung).
- **N-PE-Erdungsrelais-Konzept** für FI-Schutz im Inselbetrieb (kritisch! internes Erdungsrelais der MultiPlus, sauberes TN-System).
- **Lastmanagement/Lastabwurf** für Wallbox + Großverbraucher im Notstrom.
- Symmetrische Lastverteilung auf die 3 Phasen; Hausanschluss-Absicherung bestätigen (bei 100 A Transfer unkritisch).

### Monitoring-Konsequenz
Bleibt einfach: **ein WatchMon Core = ein UDP-Datenstrom** → Batrium-Alarm-Flow + HA-Sensoren unverändert, nur Kapazitätsbezug (15→30 kWh) anpassen, wo hinterlegt.

**Quellen:** [MultiPlus-II Datenblatt (48/10000: 8000 W, Transfer 100 A)](https://www.victronenergy.com/upload/documents/Datasheet-MultiPlus-II-inverter-charger-EN-.pdf) · [External Transfer Switch Applikation](https://www.victronenergy.com/upload/documents/MultiPlus-II_External_Transfer_Switch_application/179843-MultiPlus-II_external_transfer_switch_application-pdf-en.pdf) · [Batrium Victron/DVCC Wiki](https://wiki.batrium.com/integration/victron)

## Geplant (zurückgestellt bis Herbst): Hochlast-Gauge + Fix/Variabel-Verlustmodell (2026-07-10)

**Anlass:** E3DC-Vergleich („>95 %" = Datenblatt-Peak vs. unsere gemessenen 73,6 % Nacht-Roundtrip). Befund: kein Technik-Nachteil, sondern andere Kennzahl — realer E3DC-Nachtbetrieb liegt laut PV-Forum ebenfalls bei ~80 % und schlechter. Einziger struktureller Nachteil unserer Anlage: Überdimensionierung (3× 10 kVA fürs Whole-home-Backup → ~173 W Fixverlust nachts, davon Eigenverbrauch ~2 kWh/Tag ≈ 83 W).

**Beschlossenes Design (noch NICHT umgesetzt — Hochsommer, Hochlast-Fenster zu selten):**
1. **Hochlast-Gauge:** zweites Akkumulator-Paar (`accAcHi`/`accDcHi`) im Wirkungsgrad-Flow, Gate wie gehabt (PV < 30 W, entlädt) **plus Entladeleistung > 1,5 kW**. Dritter Discovery-Sensor „Entlade-Wirkungsgrad Hochlast" + Gauge in `victron-sys`. Erwartung ~88–91 %. Publikation erst ab > 1 kWh (sonst unknown). **Umsetzung im Generator `tools/gen_eff_flow.py`, nie direkt im Flow-JSON!**
2. **Datenlage:** Sommer dünn (dunkel erst ~22 Uhr; nur späte Spülmaschine/Trockner). Ab Herbst füllt es sich täglich (Kochen bei Dunkelheit + **Split-Klimas im Heizbetrieb 1–3 kW aus dem Akku**) — liefert genau den η am Heizlast-Betriebspunkt für die Wärmepreis-Rechnung. Schwelle bewusst 1,5 kW (datenblatt-vergleichbar); Alternative 1,2 kW (BW-WP+Grundlast fiele rein, aber näher an Mittellast) verworfen. E-Auto aus dem Akku wäre zwar Messfutter, ist aber unerwünschtes Szenario (doppelter Roundtrip, 15 kWh in ~3 h leer).
3. **Fix/Variabel-Verlustmodell statt „Eigenverbrauch herausrechnen":** Bereinigen um angenommene 2 kWh/Tag ergäbe nachts ~84–86 % Wandlungs-η, wäre aber Modell statt Messung (Annahme-Konstante!) und für Wirtschaftlichkeit falsch (Eigenverbrauch ist real bezahlt). Stattdessen: Verlust(P) = P_fix + k·P. **Die zwei Bins liefern zwei Punkte der Verlustgeraden → P_fix und k empirisch lösbar** — dann ist der Eigenverbrauch gemessen statt geschätzt, und der Roundtrip für jedes Durchsatz-Szenario (30-kWh-Ausbau, Winterheizung, Arbitrage) berechenbar. Kern-Einsicht: η ist Funktion des Durchsatzes; 2 kWh Fix bei 5 kWh Nacht-Durchsatz = 29 % Overhead → 73 %; bei 15–20 kWh Winter-Durchsatz ~9 % → 85–88 % von allein.
4. Referenz-Nacht zum Nachrechnen: 4,90 kWh DC-Entladung, 3,62 kWh AC über 7,4 h → 73,9 %; bereinigt 3,62/(4,90−0,61) = 84,4 %.

**Trigger für Umsetzung:** Herbstbeginn / erste Heiztage, oder wenn Norbert es früher will. Aufwand klein (Generator + Deploy + 1 Gauge). Damit erledigt sich auch die offene **Gauge-Schwellen-Frage** (grün ≥ 88): Das bestehende Gauge bleibt der ehrliche Grundlast-Wert, das neue zeigt den vergleichbaren Hochlast-Wert.

## 📌 Für den Winter: Entscheidungsgrundlage Multi-Standby (Stand 2026-08-12)

> [!ZUSAMMENFASSUNG] Kurzfassung
> Der Leerlauf der drei Multis ist **gemessen 174–200 W** (≈ 4,9 kWh/Tag) und **saisonunabhängig**. Im Sommer ist Abschalten **funktional ausgeschlossen**, im Winter wird es **wirtschaftlich interessant** — aber die Datengrundlage dafür fehlt noch. Vor einer Entscheidung: die unten beschriebene Kennzahl über den Winter mitzählen lassen.

### Gemessene Basis (Regression über 72 Stunden, 08.–10.08.2026)

`Verlust [kWh/h] = 0,0332 × AC-Abgabe + 0,1736`

| Kennzahl | Wert | Methode |
|---|---|---|
| **Fixverlust (Leerlauf)** | **174 W** | Achsenabschnitt der Regression |
| Gegenprobe | **200 W** (151–267) | 12 Stunden mit AC-Abgabe ≈ 0 |
| Marginaler Wandlungsverlust | 3,3 % → **η ≈ 96,8 %** | Steigung |
| Aufteilung Tagesverlust (5,19 kWh) | **80 % Leerlauf / 20 % Wandlung** | |
| Durchschnittliche AC-Leistung | 890 W | 21,35 kWh / 24 h |
| **Auslastung** | **3,0 %** von 3 × 10 kVA | |

Pro Gerät sind das ~60 W statt 35 W laut Datenblatt. Die Differenz ist plausibel (aktiver ESS-Regelbetrieb, mitgezählte DC-Verbraucher wie Cerbo und BMS, Verkabelung). ⚠️ Es ist eine **Bilanzgröße** aus der Differenz zweier Messwelten — ein systematischer Kalibrierungs-Offset zwischen Batrium/MPPT- und vebus-Zählern würde hier ebenfalls als „Leerlauf" erscheinen. Auf die letzten ~20 W nicht festlegen.

### Warum der Sommer entschieden ist

- **Die MPPTs sind DC-gekoppelt** (~26 kWh/Tag = 22 % der PV). Ihr einziger Weg ins Hausnetz führt durch die Multis. Ohne sie könnten sie nur die Batterie laden (7,5 kWh) und würden danach abregeln → **~19 kWh/Tag Ertragsverlust**. Der Fronius (~94 kWh/Tag, 78 %) läuft dagegen AC-seitig an den Multis vorbei.
- Die Multis liefern **20,0 h/Tag** AC (83 % der Zeit), Durchsatz 20,2 kWh/Tag. Untätig sind sie nur **4,0 h/Tag — und zwar morgens 7–10 Uhr**, nicht nachts: Dann deckt der Fronius den Verbrauch und die MPPTs laden die Batterie direkt über DC.
- Abschalten in genau diesen 4 Stunden spart 0,82 kWh/Tag = 18 % des Leerlaufs — im Sommer bewertet mit 7,38 ct sind das **6 Cent pro Tag**. Lohnt nicht.

### Was im Winter anders ist

Batterie: **280 Ah ≈ 14,3 kWh brutto, bei MinimumSocLimit 20 % rund 11,5 kWh nutzbar.**

- Der MPPT-Ertrag bricht bei Ost/West und tiefem Sonnenstand stark ein → kaum noch DC-Durchleitung.
- Die Batterie wird oft nicht mehr voll und erreicht abends die 20 %-Grenze. **Ab da stehen die Multis wirklich untätig**, während das Haus am Netz hängt — genau die Stunden, die im Sommer fehlen.
- Der Leerlauf bleibt bei 4,9 kWh/Tag, **aber der Preis kehrt sich um: 7,38 ct → 34,94 ct (Faktor 4,7)** — 1,71 €/Tag statt 0,36 €.

**Modellrechnung (NICHT gemessen, Annahme: halber Tag „Batterie leer, keine PV"):**
`12 h × 205 W = 2,5 kWh/Tag → per Mode 2 hebbar ~60–70 % → ~0,56 €/Tag → ~65 €/Jahr`
Deckt sich mit der früheren Schätzung von ~70 €. Bestätigt zugleich, warum die 350 € aus der reinen Fixverlustrechnung zu hoch gegriffen waren: Der Großteil des Leerlaufs fällt an, **während die Multis gebraucht werden**.

### ⬜ Offen: die Kennzahl, die im Winter zu messen ist

Datenlage: **Es gibt keine Winterdaten** — die HA-Langzeitstatistik beginnt im Juli 2026 (auch `sensor.fronius_pv_energie` hat nur Juli/August). Die obige Rechnung ist deshalb ein Modell, keine Messung.

Zu erheben ist **nicht** der Gesamtverlust, sondern der **vermeidbare Anteil**: Verlust, der anfällt, während die Anlage nichts leisten kann — also **SoC ≤ 25 % UND PV ≈ 0**. Ein kleiner Zusatz-Flow im LXC (Generator-Stil wie die übrigen), der genau diese Stunden akkumuliert, würde bis Januar aus der Schätzung eine Zahl machen. *Noch nicht gebaut — bewusst zurückgestellt.*

### Randbedingungen für eine spätere Standby-Automatik

1. **Ausschließlich Mode 2 (Charger Only) schreiben.** Mode 4 (Off) öffnet das Transferrelais → AC-Out spannungslos. Relevant spätestens nach einem Whole-home-Umbau.
2. **Rechtzeitig zurückschalten, bevor morgens PV kommt** — sonst kostet die Automatik DC-Ertrag statt Geld zu sparen.
3. Bedingung saisonal bzw. am Ertrag der Vortage aufhängen, **nicht** an Tag/Nacht: Nachts arbeiten die Multis (Batterieentladung), sie stehen nur, wenn die Batterie leer ist.

### ⚠️ Ergänzung zum Winter-Abschnitt: Batterie-Ausbau auf ~30 kWh ändert die Rechnung

Der oben stehende Abschnitt rechnet mit **11,5 kWh nutzbar** (280 Ah, MinSoC 20 %). Mit dem beschlossenen Ausbau auf ~30 kWh brutto (2. Pack à 15 kWh parallel, weitere CellMate K9 am selben WatchMon Core) sind es **~24 kWh nutzbar** — die dortigen Zahlen sind damit überholt.

**1. Das Sparpotenzial sinkt, statt zu steigen.** Die Modellrechnung von ~65 €/Jahr unterstellt, dass die Batterie im Winter etwa den halben Tag leer ist und die Multis untätig stehen. Mit doppeltem Speicher überbrückt sie längere Dunkelphasen und erreicht die 20 %-Grenze **seltener** — genau die Stunden, um die es beim Standby geht, werden weniger. **Die 65 €/Jahr sind damit eine obere Grenze, die nach dem Ausbau weiter fällt.** Eine Standby-Automatik lohnt sich danach noch weniger als ohnehin schon.

**2. Der Wirkungsgrad steigt von allein.** Der Leerlauf (4,2 kWh/Tag) bleibt konstant und verteilt sich auf mehr Durchsatz. Mit der gemessenen Verlustgeraden `Verlust = 0,0332 × AC + 4,17`:

| Szenario | AC-Durchsatz | Verlust | Wirkungsgrad |
|---|---|---|---|
| heute (Sommer) | 21,4 kWh/Tag | 4,9 kWh | **80,4 %** |
| E-Auto nachts aus dem Speicher | ~41 kWh/Tag | 5,5 kWh | **88,2 %** |

Der Verlust steigt nur um 0,6 kWh, obwohl fast doppelt so viel Energie durchgeht — es wächst ja nur der variable Anteil von 3,3 %. ⚠️ **Wichtig: Der Durchsatz steigt NICHT automatisch mit der Kapazität.** Im Sommer werden heute nur 7,6 von 11,5 nutzbaren kWh entladen — der Nachtbedarf des Hauses ist der Engpass, nicht der Speicher. Zusätzlichen Durchsatz bringt vor allem **die Nachtladung des E-Autos aus der Batterie** (Auto tagsüber unterwegs, lädt abends aus dem Speicher statt aus dem Netz). Ohne dieses Nutzungsmuster bleibt die zweite Batteriehälfte im Sommer weitgehend ungenutzt.

**3. Korrektur zur früheren Modellannahme (Abschnitt „Fix/Variabel-Verlustmodell"):** Dort war `P_fix` mit **~2 kWh/Tag** angesetzt. Gemessen sind es **4,17–4,8 kWh/Tag**, also mehr als das Doppelte. Die dortige Wirkungsgrad-Prognose (85–88 % bei hohem Durchsatz) trifft trotzdem ungefähr zu — aber nur, weil der angenommene Durchsatz hoch war, nicht weil P_fix stimmte.

**4. Die Multi-Dimensionierung ist damit gerechtfertigt.** Mit 24 kWh nutzbar wird die 11-kW-Nachtladung praktikabel: 2,2 Stunden Volllast statt 1,0, das sind ~24 kWh ins Auto (140–160 km), bei entspannten 0,41 C statt 0,82 C. Drei 5000er wären bei 3,67 kW/Phase plus Hauslast am Limit gewesen — die 10000er sind für dieses Szenario die richtige Wahl. Der Preis der Bereitschaft (110–190 €/Jahr Mehr-Leerlauf gegenüber 5000ern) kauft damit eine real nutzbare Funktion, nicht nur Reserve.

**To-dos, die mit dem Ausbau akut werden (aus den Abschnitten oben):**
- [ ] **Gesamtkapazität im WatchMon Core neu einstellen** (SoC/Coulomb) — sonst rechnet der SoC auf der alten Kapazität weiter und **alle SoC-basierten Automatiken laufen falsch**.
- [ ] **Ladedeckel-Automatik anpassen**: Schwelle „SoC < 60 %" ist für 15 kWh gerechnet (= 6 kWh Restbedarf); bei 30 kWh entspricht dasselbe Ziel nur noch 30 %. Besser gleich **in kWh-Restbedarf statt SoC-Prozent** umstellen.
- [ ] Nach dem Ausbau die Verlustgerade **neu regressieren** (`Verlust vs. AC-Abgabe`, Stundenwerte) — P_fix sollte unverändert bleiben; tut es das nicht, war ein Teil des „Leerlaufs" in Wahrheit ein Messoffset.

### 🔥 Ergänzung 2: Heizen mit den Split-Klimas verändert das Winterbild grundlegend

Erfasst sind aktuell zwei Einheiten mit eigenem Zigbee-Zähler: `sensor.0xa085e3fffebc4574_energy` (**SK_Arbeitszimmer_Küche**, 298 kWh) und `sensor.0xa085e3fffebd16b8_energy` (**SK_Wohnzimmer**, 144 kWh). Geplant ist, im Winter **vermehrt damit zu heizen**.

**Damit ist die Standby-Überlegung endgültig vom Tisch.** Der Winterabend- und Nachtverbrauch steigt genau in den Stunden, in denen die Multis nach der Modellrechnung untätig herumstehen sollten. Sie arbeiten dann — der „vermeidbare Leerlauf", um den es ging, verschwindet weitgehend von selbst. **Die 65 €/Jahr aus dem Modell oben sind damit endgültig eine theoretische Obergrenze.**

**Der Wirkungsgrad wandert in den prognostizierten Bereich.** Bei 15–20 kWh zusätzlichem Tagesbedarf im Winter liegt der Durchsatz genau dort, wo die frühere Notiz 85–88 % erwartet hat — diesmal aus einem realen Nutzungsmuster statt aus einer Annahme. Zusammen mit dem 30-kWh-Speicher ist das die Konstellation, für die die Anlage ausgelegt wurde.

**Wirtschaftlichkeit gegen die Ölheizung** (Referenz aus dem Abschnitt oben: **13,5 ct/kWh Wärme**, 3.500 l/Jahr bei Kessel 85 %), gerechnet mit Netzstrom 34,94 ct:

| COP | Wärmekosten | vs. Öl (13,5 ct) |
|---|---|---|
| 3,5 (mild, ~10 °C) | 10,0 ct | −26 % |
| 3,0 (~0 °C) | 11,6 ct | −14 % |
| **2,6** | **13,5 ct** | **Break-even** |
| 2,0 (~−10 °C) | 17,5 ct | +30 % |

**Break-even bei COP 2,6** — den unterschreiten moderne Split-Geräte erst bei deutlichem Frost. Mit PV- oder Batteriestrom (Opportunitätskosten 7,38 ct statt 34,94 ct) ist die Klima praktisch immer im Vorteil. ⚠️ Nicht vergessen: Im Heizbetrieb ziehen Split-Geräte **deutlich mehr als im Kühlbetrieb** (Abtauzyklen, größere Temperaturspreizung) — die heute gemessenen 442 W (Kühlen, beide Räume) sind keine Referenz für den Winter.

**To-do: den realen Jahres-COP messen statt schätzen.** Beide Geräte haben Stromzähler. Über den Winter den SK-Stromverbrauch gegen den **eingesparten** Ölverbrauch stellen (Liter × 10 kWh × 0,85 Kesselwirkungsgrad = ersetzte Nutzwärme) → daraus ergibt sich der tatsächliche Arbeitsjahres-COP der Anlage. Das ist belastbarer als jedes Datenblatt und beantwortet endgültig, wie weit sich der Umstieg trägt. Sinnvollerweise **vor** der nächsten Ölbestellung auswerten.

---

