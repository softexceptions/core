---
tags: [projekt/aktiv, waermepumpe, warmwasser, energie, messung]
status: aktiv
date: 2026-07-26
updated: 2026-08-23
---

# Brauchwasser-Wärmepumpe

Stillstandsverlust der Brauchwasser-Wärmepumpe im Heizungskeller: Versuchsaufbau,
Auswertung und der geplante Umbau (Siphon, Rückschlagventil, ΔT-Automatik).

Ausgelagert aus [[victron_node_red]] am 23.08.2026 — der Verlust war dort über die
Energiebilanz aufgefallen, ist aber ein Heizungs-, kein Node-RED-Thema.

**Zusammenhänge:**
- Messwert-Herkunft (Zigbee-Dose, Energiebilanz): [[victron_node_red]]
- Datenpfad der Dose, Grafana-Dashboard: [[homelab-monitoring]]
- Bedienungsanleitungs-App für BW-Wärmepumpen: [[bwt]]

> [!warning] Offen: Phase C
> Der Umbau ist noch nicht ausgeführt. Nach dem Umbau muss neu gemessen werden —
> die Auswertungsvorschrift dafür steht am Ende dieser Notiz.

## Versuchsaufbau: BW-WP-Stillstandsverlust eingrenzen — Stufentest Pumpe → Hahn (2026-07-26) ✅ ABGESCHLOSSEN (Auswertung 13.08.2026)

**Ziel:** Die 242 W Dauerverlust der Brauchwasser-Wärmepumpe (hergeleitet in [[victron_node_red#5. BW-WP: 54 % des Stroms halten nur die Temperatur]]) auf ihre Ursache zurückführen. Kandidaten: (a) Zirkulationspumpe läuft mehr als angenommen, (b) Schwerkraftzirkulation durch die stillstehende Zirkulationsleitung, (c) Einrohrzirkulation im blanken Warmwasser-Vorlauf bzw. Speicherdämmung.

**Design — von Norbert vorgegeben, besser als mein ursprünglicher Vorschlag.** Ich hatte „Hahn zu + Pumpe stromlos" in einem Schritt vorgeschlagen; das hätte einen Gesamteffekt gemessen, ohne die Anteile zu trennen. Norberts Staffelung isoliert schrittweise:

```
  Basis  -> Phase A   =  echter Pumpenanteil
  A      -> Phase B   =  Schwerkraftzirkulation durch den Kreis
  Rest bei B          =  Einrohrzirkulation Vorlauf + Speicherdämmung
```

Drei Größen aus zwei Eingriffen. Der erwartete Nullbefund in Phase A ist kein verschwendeter Aufwand, sondern der **Test der Annahme** „bedarfsgesteuerte Pumpe läuft vernachlässigbar" — die war bisher nur Norberts Aussage, nicht gemessen.

### Basislinie (16 Tage, 09.–25.07.2026)
`sensor.0xa085e3fffeb7c870_energy`, Tages-`change`: **Mittel 3,19 kWh/Tag, σ = 0,35 kWh**, Spanne 2,61–3,70 (Ausreißer 5,61 am 23.07. ausgeschlossen).

Bei 3 Messtagen je Phase: Standardfehler = 0,35/√3 = **0,202 kWh/Tag** → **nachweisbar ab 0,40 kWh/Tag** Unterschied (= 21 % des Stillstandsverlusts von 1,93 kWh/Tag el.).

### Vorhersagen — VOR der Messung festgelegt (Falsifizierbarkeit)

**Phase A — nur Pumpe stromlos, Absperrhahn offen**
- Modell sagt: **3,1–3,2 kWh/Tag, praktisch unverändert.**
- Begründung: Die 242 W wurden im Nachtfenster 23:57–05:57 gemessen, in dem die bedarfsgesteuerte Pumpe ohnehin steht. Eine stromlose Pumpe ist zudem **kein geschlossenes Ventil** — das Gehäuse bleibt hydraulisch offen, Schwerkraftzirkulation läuft weiter.
- **Fällt der Wert unter 2,79 kWh/Tag → Modell widerlegt**, die Pumpe läuft deutlich mehr als „bedarfsgesteuert" nahelegt.

**Phase B — zusätzlich Absperrhahn in der Zirkulationsleitung zu**
```
  Kreis ist die ganze Ursache  ->  1,26 kWh/Tag
  Kreis ist die halbe Ursache  ->  2,23 kWh/Tag
  Kreis irrelevant             ->  3,19 kWh/Tag
```
- Bleibt es bei ~3,2 kWh/Tag: weder Pumpe noch Kreis. Dann Einrohrzirkulation im blanken Vorlauf oder Speicherdämmung — **kein Ventil hilft**, nur Dämmung + Schwanenhals (nach unten gezogene Rohrschleife) direkt am Speicherausgang.
- Ein Tageswert **unter 2,5 kWh** läge außerhalb von allem in 16 Tagen Gemessenen → eindeutiges Signal.

### Störgröße (verfälscht BEIDE Phasen nach oben)
Ohne Zirkulation dauert warmes Wasser an entfernten Zapfstellen länger → längeres Laufenlassen → mehr Warmwasserbedarf. Größenordnung: 30 s × 5 Zapfungen × 6 l/min = 15 l/Tag bei 40 K ≈ **0,23 kWh/Tag elektrisch = 1,2 Standardfehler**. Maskiert kleine echte Effekte.

**Gegenmaßnahmen:** (1) Familie bitten, das Zapfverhalten nicht zu ändern. (2) Zusätzlich das **Nachtfenster 23:57–05:57** auswerten — dort zapft niemand, die Störgröße greift nicht. Referenzwerte aus der Basislinie: 0,081 kWh/h (Nacht auf 26.07.) und 0,087 kWh/h (Nacht auf 23.07.).

### Messprotokoll
- **Primärgröße:** Tages-`change` von `sensor.0xa085e3fffeb7c870_energy` (Energiezähler = robust gegen Zigbee-Meldelücken).
- **NICHT** den Gauge „Momentanleistung" auf `kg-heizungskeller` verwenden — der hängt an der lückenhaften Leistungsmeldung und kann stundenlang veraltet sein (siehe Stolperfalle im Abschnitt oben).
- **Sekundärgröße:** Nachtfenster-Rate, nur verwertbar wenn die Zigbee-Erfassung an dem Tag lückenlos war — Validierung durch Rekonstruktion `Σ(Laufzeit × mittlere Leistung)` gegen den Tages-kWh-Wert (Abweichung < ~2 % → belastbar).
- 3 auswertbare Tage je Phase, normale Anwesenheit. Keine Gäste, keine Abwesenheit, keine Wäsche-Sonderaktionen.
- **Umschaltzeitpunkte notieren.** HA bucketet Tagesstatistik Mitternacht–Mitternacht lokal → spätabends umschalten, dann sind die Buckets sauber getrennt.
- Pumpe fest verdrahtet? Deaktivierung über Schalter/Sicherung genügt, keine Klemmarbeit an der festen Installation nötig.
- Phase B: Hahn zu **und** Pumpe weiter stromlos — der Hahn macht die thermische Arbeit, die Stromlosigkeit schützt die Pumpe vor Anlauf gegen geschlossenes Ventil.
- Legionellen: für 3+3 Tage irrelevant (Speicher bleibt heiß). Erst bei dauerhafter Stilllegung ein Thema → dann Leitung richtig abtrennen/entleeren oder mit Schwerkraftbremse betreiben, Sache des Installateurs.

### Status

**Phase A gestartet: 2026-07-26, 01:41 CEST** (Pumpe stromlos, Absperrhahn offen).

- Auswertbare Tage: **26. / 27. / 28.07.** Der 26.07. ist ab 01:41 Phase A — die ersten 1h41 fallen nicht ins Gewicht, weil eine bedarfsgesteuerte Pumpe zu dieser Uhrzeit ohnehin nicht läuft. 27. und 28.07. sind vollständig sauber.
- **Umschalten auf Phase B: spätabends am 28.07.** (nach 23:00, vor Mitternacht) → Phase B = 29. / 30. / 31.07.
- AC-Lasten-Snapshot zum Umschaltzeitpunkt (Referenz, ersetzt keine Messung): Netz L1 92 / L2 −3 / L3 −102 = −13 W; AC-Lasten L1 205 / L2 143 / L3 0 = **348 W**. Liegt im normalen Nachtband (Minimum 342, 25-%-Quantil 387, Median 506 W) — kein Sofort-Effekt erkennbar, war so vorhergesagt.

- [x] Phase A gestartet am: **26.07.2026 01:41**  Tageswerte: **1,40** (Teiltag) / **0,80** / **0,75** kWh
- [x] Phase B gestartet am: **28.07.2026 spätabends**  Tageswerte: **0,60** / **0,91** / **0,54** kWh
- [x] Auswertung gegen Basislinie → **Ergebnis siehe „BW-WP-Stufentest: Auswertung" am Dateiende (13.08.2026)**

### Randnotiz (offen, betrifft den Versuch NICHT)
Beim Snapshot stand `Consumption/L3/Power` auf exakt **0 W**, während Netz L3 bei −102 W lag. Verdacht: Der GX klemmt die Phasen-Consumption bei 0 ab, wenn die Rechnung negativ würde. Nachprüfen ließ es sich nicht — zwei aufeinanderfolgende `dbus-send`-Aufrufe sind nicht simultan, die MultiPlus-AcIn-Werte hatten sich zwischen den Reads von ~−150 auf ~−220 W je Phase verschoben. **Konsequenz falls bestätigt:** Bei viel PV auf einer Phase wären die AC-Lasten nach oben verzerrt (negative Phasenanteile fielen weg). Betrifft die Summenaussage aus dem Abschnitt oben nicht — die wurde zweimal in verschiedenen Betriebszuständen gegengerechnet und ging auf. Für eine saubere Klärung bräuchte es einen einzelnen Aufruf, der `/Ac` und den vebus gleichzeitig liest (z. B. ein kleines Skript über eine dauerhafte D-Bus-Verbindung statt Einzel-`dbus-send`).


## 🔬 BW-WP-Stufentest: Auswertung (13.08.2026) — Modell widerlegt, Ursache gefunden

Auswertung des am 26.07. gestarteten Versuchs (Aufbau, Vorhersagen und Messprotokoll siehe Abschnitt oben). Primärgröße: Tages-`change` von `sensor.0xa085e3fffeb7c870_energy`.

### Messwerte

| Phase | Tageswerte | Mittel |
|---|---|---|
| Basislinie (10.–25.07., ohne Ausreißer 23.07.) | 15 Tage | **3,19 kWh/Tag** (σ 0,36) |
| **A** — Pumpe stromlos, Hahn offen (26.–28.07.) | 1,40* / 0,80 / 0,75 | **0,78** (*26.07. Teiltag, ausgeklammert) |
| **B** — zusätzlich Hahn zu (29.–31.07.) | 0,60 / 0,91 / 0,54 | **0,68** |
| Nachlauf (01.–12.08., Zustand unverändert) | 12 Tage | **0,52** |

Die aus HA rekonstruierte Basislinie (3,19 / σ 0,36) trifft den im Versuchsaufbau notierten Wert exakt — die Datengrundlage ist konsistent.

### ❌ Vorhersage falsifiziert

**Erwartet war für Phase A: 3,1–3,2 kWh/Tag („praktisch unverändert"), Falsifikationsschwelle < 2,79.**
**Gemessen: 0,78 kWh/Tag — 6,6 σ unter der Basislinie.**

Die Annahme „die bedarfsgesteuerte Pumpe läuft im Nachtfenster ohnehin nicht, eine stromlose Pumpe ist kein geschlossenes Ventil" ist damit klar widerlegt. **Die Pumpe lief faktisch durch**, unabhängig davon, was ihre Steuerung nominell tut. Der vorab festgelegte Falsifikationstest hat genau das geleistet, wofür er gedacht war.

### Zerlegung — das Ziel des Versuchs

> ⚠️ **Ursache inzwischen gefunden (13.08.2026):** Der „Pumpenanteil" unten ist **nicht** der Preis der Zirkulation, sondern die Folge eines **Installationsfehlers — die Pumpe fördert verkehrt herum**. → Abschnitt „Verkehrte Pumpenrichtung" am Dateiende. Die Prozentzahlen bleiben als Messung gültig, die Zuordnung „Pumpe/Zirkulation ist schuld" nicht.

```
Pumpenanteil          Basis − A  =  2,42 kWh/Tag   (75,7 %)
Schwerkraft / Kreis   A − B      =  0,09 kWh/Tag   ( 2,9 %)
Rest (Dämmung+Zapf)   B          =  0,68 kWh/Tag   (21,4 %)
```

**Der gestaffelte Aufbau hat sich ausgezahlt.** Ein einzelner Schritt „Hahn zu + Pumpe aus" hätte nur einen Gesamteffekt von 2,5 kWh gezeigt, ohne die Anteile zu trennen — und hätte zur falschen Konsequenz geführt (Dämmung/Schwanenhals statt Pumpensteuerung). Die Schwerkraftzirkulation, ursprünglich als Hauptverdächtiger gehandelt, ist mit 2,9 % praktisch bedeutungslos.

**Physikalische Einordnung:** Die 2,42 kWh/Tag sind überwiegend *nicht* Pumpenstrom (eine Zirkulationspumpe zieht 10–30 W ≙ max. 0,72 kWh/Tag), sondern die **in der Leitung abgegebene Wärme**, die die BW-WP nachliefern muss: 2,42 kWh el. × COP 3 ≈ 7 kWh Wärme/Tag ≈ **300 W Dauerverlust**. ⬜ **Offen:** Hängt die Zirkulationspumpe am selben Zigbee-Zähler wie die BW-WP? Falls ja, enthält der Wert auch den Pumpenstrom.

**Ersparnis im gemessenen Zustand:** 2,67 kWh/Tag = **975 kWh/Jahr ≈ 341 €** (Netzbezug 34,94 ct) bzw. 72 € bei reiner PV-Deckung.

### Entscheidung: Siphon + Rückschlagventil + ΔT-Automatik

Umgesetzt wird: **Siphon (Schwanenhals) und Rückschlagventil in den Zirkulationskreis**, dazu eine **Automatikschaltung, die über die Temperaturdifferenz zwischen Zirkulationsrücklauf und Heißwasser bedarfsgesteuert schaltet**. Danach Wiederinbetriebnahme.

⚠️ **Wichtig für die Einstellung — hier liegt der eigentliche Hebel:** Die alte Pumpe galt ebenfalls als „bedarfsgesteuert" und lief trotzdem durch. Eine ΔT-Steuerung mit **enger** Hysterese hält die Leitung dauerwarm und verhält sich am Ende wie Dauerlauf. Der Spareffekt hängt fast vollständig davon ab, **wie weit die Leitung auskühlen darf**:

| ΔT-Schwelle | Wirkung |
|---|---|
| eng (~5 K) | Leitung bleibt warm, häufiges Takten → wenig Ersparnis |
| **großzügig (15–20 K)** | Leitung kühlt zwischendurch ab → deutliche Ersparnis, kurze Wartezeit |

→ **Großzügig anfangen**, nur bei echtem Alltagsproblem enger stellen. Der Siphon unterstützt das, weil ohne Schwerkraftzirkulation die Auskühlung langsamer erfolgt und die Pumpe seltener anspringt.

### Geprüft und verworfen: Zirkulation dauerhaft absperren

| | Ersparnis brutto | Nebenkosten | netto |
|---|---|---|---|
| Dauerhaft ab (Absperrventile) | 330 € | −25 € Wasser (15 l/Tag), −29 € Nachlaufwärme | **276 €** |
| ΔT-Steuerung großzügig (Schätzung 1,2 kWh/Tag) | 254 € | — | **254 €** |
| ΔT-Steuerung eng (Schätzung 2,0 kWh/Tag) | 152 € | — | 152 € |

**Nur 23 €/Jahr Unterschied** zwischen Dauerabschaltung und gut eingestellter Automatik — die Einstellung der Schwelle (100 € zwischen großzügig und eng) wiegt schwerer als die Grundsatzfrage.

⚠️ **Hygienisch ist Absperren die schlechteste Variante:** Eine abgesperrte, wassergefüllte Leitung bleibt am Trinkwassernetz und wird zum **Totstrang** — stehendes Wasser bei Raumtemperatur (20–30 °C) liegt mitten im Legionellen-Wachstumsbereich (25–45 °C). DVGW W 551 / VDI 6023 verlangen für nicht mehr genutzte Leitungen **fachgerechtes Trennen und Entleeren**, nicht bloßes Absperren. Für 23 €/Jahr nicht vertretbar.

### ⬜ Phase C — nach dem Umbau messen

**Vor der Wiederinbetriebnahme:** Die Leitung stand seit 26.07. rund 3 Wochen still (Totstrang-Situation). Einmal **längere Zeit bei voller Speichertemperatur durchspülen** (> 60 °C an der Entnahmestelle), erst danach die Automatik scharf schalten.

Erwartungswerte für die Woche nach dem Umbau — **Untergrenze ist 0,68 kWh/Tag** (echter Zapfbedarf + Speicherdämmung, darunter geht es physikalisch nicht):

| Messung | Interpretation |
|---|---|
| 1,0–1,5 kWh/Tag | **Zielbereich** — Großteil der Ersparnis gesichert, voller Komfort und Hygiene |
| ~2,0 kWh/Tag | Schwelle zu eng → ΔT großzügiger stellen |
| ~3,0 kWh/Tag | Automatik wirkungslos, Zustand wie vorher → Steuerung prüfen (Sensorposition? Hysterese?) |

---

## 🔧 Verkehrte Pumpenrichtung — die eigentliche Ursache (13.08.2026)

**Befund:** Die Zirkulationspumpe fördert **in die falsche Richtung**. Sie zieht das Wasser nicht von der Verbrauchsstelle zurück in den Speicher, sondern **aus dem Speicher heraus zur Verbrauchsstelle**.

### Der Mechanismus: thermischer Kurzschluss durch den Speicher

```
richtig:   Speicherkopf (heiß) → Vorlauf → Zapfstelle → Rücklauf → Speichermitte (kühler)
falsch:    Speichermitte (lau) → Rücklauf → Zapfstelle → Vorlauf → Speicherkopf (heiß)
                                                                    ↑ kühlt genau die Zone ab,
                                                                      die die WP gerade erwärmt hat
```

Bei korrekter Richtung wird das **abgekühlte** Wasser dort eingespeist, wo der Speicher ohnehin kühler ist — die Schichtung bleibt erhalten. Verkehrt herum wird lauwarmes Wasser permanent **in die heiße Zone** gepumpt: Die Schichtung wird zerstört, der Speicherkopf kühlt ab, die WP heizt nach, und die Pumpe verteilt die frische Wärme sofort wieder nach unten. Ein Kreislauf, der sich selbst am Laufen hält.

**Damit ist die offene Frage aus der Auswertung beantwortet, warum die „bedarfsgesteuerte" Pumpe durchlief:** Hängt die Steuerung an einem Fühler, der durch die verkehrte Strömung dauerhaft zu kühles Wasser sieht, meldet sie permanent Bedarf. Die Steuerung hat nicht versagt — sie hat auf ein falsches Signal korrekt reagiert.

### Neubewertung der Auswertung

Die gemessenen 2,42 kWh/Tag „Pumpenanteil" sind damit **Fehlerfolge, nicht Zirkulationskosten**. Eine korrekt installierte Zirkulation hätte nie so viel gekostet. Auch das Scheitern der Vorhersage erklärt sich neu: Das Modell war nicht falsch gedacht — im System steckte ein Fehler, den keine der drei Hypothesen (Pumpenlaufzeit / Schwerkraft / Dämmung) auf dem Zettel hatte.

**Korrigierte Erwartung für Phase C: 0,7–1,0 kWh/Tag** statt der zuvor genannten 1,0–1,5 — der Durchmischungsanteil entfällt mit der richtigen Fließrichtung vollständig.

### Jahreskosten in absoluten Zahlen (Jahresmittel inkl. Winterkorrektur, Mischpreis 21 ct)

| Zustand | kWh/Tag | kWh/Jahr | €/Jahr |
|---|---|---|---|
| **A** bisher — Pumpe verkehrt, Dauerlauf | 3,93 | 1.434 | **301 €** |
| **C** repariert — richtige Richtung + ΔT-Automatik *(Erwartung)* | ~1,10 | ~400 | **~84 €** |
| **B** Zirkulation ganz aus *(Zustand seit 26.07.)* | 0,75 | 274 | **57 €** |

```
A → C   Reparatur (Richtung + Steuerung)     ~217 €/Jahr
C → B   zusätzlich komplett abschalten         ~27 €/Jahr
A → B   beides zusammen                      ~244 €/Jahr
```

**Rund neun Zehntel der Ersparnis kommen vom behobenen Installationsfehler**, nicht vom Verzicht auf die Zirkulation. Die Zirkulation selbst kostet nach der Reparatur nur noch ~27 €/Jahr — dafür sofort warmes Wasser und kein Legionellenthema. Damit ist die Absperr-Diskussion endgültig erledigt.

Aufschlüsselung des Ausgangszustands: **Zapfbedarf 274 kWh/Jahr (19 %) · Zirkulationsverlust 1.161 kWh/Jahr (81 %)** — vier Fünftel des Verbrauchs waren Verlust.

⚠️ **Belastbarkeit:** Der Zähler steht bei 57,75 kWh gesamt, die Messhistorie umfasst 35 Tage (Sensor läuft seit 09.07.2026). Die Jahreszahlen sind eine Hochrechnung um Faktor ~25. Solide ist der Tagesmittelwert der Basislinie (17 Tage, σ 11 %); geschätzt sind die Winterfaktoren (Kaltwasser 15→8 °C ≈ 1,18 · COP im kalten Keller ≈ 1,20 · Leitungsverlust ≈ 1,23), die aus 1.164 kWh die 1.434 machen. **Frühere Angaben von „341 €/Jahr" waren zu hoch** — sie rechneten Sommerwerte mit vollem Netzpreis.

### Drei Punkte für den Umbau

1. **Rückschlagventil in Fließrichtung einbauen** — es verhindert dann nicht nur Schwerkraftzirkulation, sondern erzwingt dauerhaft die richtige Richtung. Der Fehler kann nicht zurückkommen.
2. **Fühlerposition der ΔT-Automatik prüfen** — der Fühler muss das **zurückkehrende, abgekühlte** Wasser kurz vor Speichereintritt messen. Sitzt er auf der falschen Seite, entsteht wieder ein Signal, das nicht zum Bedarf passt (genau der Effekt, der die alte Steuerung zum Dauerläufer machte).
3. **Nach dem Umbau die Richtung verifizieren** — im Betrieb beide Leitungen anfassen: Der Vorlauf muss deutlich wärmer sein als der Zirkulationsrücklauf. Sind beide gleich heiß, stimmt die Richtung immer noch nicht.

### Weitere Präzisierungen zur Auswertung (13.08.2026)

- **Urlaubstage aus der Nachlaufphase herausrechnen:** Der 02.08. (0,05) und der 06.08. (0,15) waren Abwesenheit. Bereinigt ergibt sich für 29.07.–12.08. **0,62 kWh/Tag (σ 0,15 = 25 %)** statt 0,55 (σ 0,23 = 42 %). → **Für Phase C reichen damit fünf saubere Messtage**, nicht sieben.
- **Störgröße im Alltag bestätigt:** Bei geschlossener Zirkulation steigt der Verbrauch bei Wasserentnahme spürbar, weil erst kaltes Vorlaufwasser ablaufen muss. Die im Versuchsaufbau geschätzten ~0,23 kWh/Tag sind damit real. **Konsequenz: Die Untergrenze für Phase C ist nicht 0,68, sondern ~0,39 kWh/Tag** (0,62 − 0,23 Störgröße = echter Zapfbedarf + Speicherdämmung).
- **Auswertungsvorschrift für Phase C:** Sobald die Zirkulation wieder läuft, entfällt die Störgröße. Ein naiver Vergleich gegen 0,62 würde die Zirkulation zu günstig aussehen lassen — **korrekt ist Phase C minus 0,39**. Beispiel: Phase C misst 1,20 → Zirkulation kostet real 0,81 kWh/Tag, nicht 0,58. *(Für die Geldrechnung bleibt dagegen die Differenz der Gesamtverbräuche maßgeblich, da die Störgröße real in Strom und Wasser bezahlt wird.)*

