---
tags: [victron, wirtschaftlichkeit, kosten, waermepumpe, arbitrage]
status: aktiv
date: 2026-07-08
updated: 2026-08-23
---

# Wirtschaftlichkeit

Was die Anlage kostet, was sie spart und welche Entscheidung sich daraus ableitet.
**Was hier steht, gilt.**

> [!abstract] Die drei Ergebnisse
> 1. **Öl vs. Wärmepumpe:** Break-even-JAZ **2,12** (Stand 08/2026, Öl 139,90 €/100 l).
>    So tief, dass die Splits sie praktisch jeden Tag überbieten. Jahresersparnis
>    1.350–1.930 €.
> 2. **Batterie-Arbitrage lohnt nicht** — der Roundtrip frisst die Tag/Nacht-Spreizung.
>    Der Hebel ist der Wärmepumpen-Wechsel, nicht der Speicher.
> 3. **Leerlaufsockel:** 171–253 €/Jahr, davon über 70 % im Winter. Hebbar ist
>    nur ein Bruchteil (20–89 €).

> [!warning] Überholte Zahlen im Juli-Abschnitt
> Der unterste Abschnitt (Wärmewende 07/2026) enthält eine Rechnung, die am
> 13.08. als fehlerhaft erkannt wurde (Brutto-Öleinsparung ohne Gegenrechnung
> des Stroms) und eine überholte Preisbasis (114,56 statt 139,90 €/100 l).
> Er ist als Herleitung erhalten, die Fehler sind im Text markiert.
> **Gültig ist der Abschnitt „Öl vs. Wärmepumpe: Stand 08/2026".**

Verwandt: [[Energiebilanz und Kennzahlen]] (woher die Verlustzahlen kommen) · [[Brauchwasser-Wärmepumpe]] · [[Ausbauplanung]]

## Wärmewende & Speicher-Arbitrage (2026-07-08) — Wirtschaftlichkeit

**Kontext:** Norbert will weg vom Öl, hat zwei Split-Klimageräte (bisher nur Sommer-Kühlbetrieb gelaufen, Winter-Heizbetrieb noch ungetestet) und überlegt Speicher-Arbitrage mit dynamischem Tarif. „Arbitrage" = Gewinn aus Preisdifferenz desselben Guts; hier über die **Zeit** (nachts billig laden, tagsüber teuren Netzbezug vermeiden). Der Speicher ist das Transportmittel, der Wirkungsgrad der „Fahrpreis": lohnt nur, wenn **η_roundtrip > p_nacht / p_tag**.

**Haus-Grundlast (rekonstruiert aus Batterie-Entladung, Netzbezug nachts ~0 → ESS deckt alles):**
- Absolute Grundlast (tiefe Nacht): **~250–300 W AC**; mittlere Nachtlast ~450–550 W; Abendlast (Kochen) ~1–2 kW.
- Konsequenz für den Ausspeicher-η: Grundlast-Stunden ~74–78 %, Abend/höhere Last ~88–91 %. Energie-gewichtet real ~82–86 %, außer in reinen Grundlast-Nächten (dann die gemessenen ~74 %).

**Split-Klima-Verbrauch (Zigbee-Steckdosen, Sommer-Kühlbetrieb 30 d):**
- SK Arbeitszimmer/Küche (`sensor.0xa085e3fffebc4574_power`): 219,2 kWh Lebenszeit, Peak ~750–1000 W (max 1564).
- SK Wohnzimmer (`sensor.0xa085e3fffebd16b8_power`): 99,6 kWh, Peak ~1280–1340 W (max 1896).
- Installierte el. Spitze zusammen ~2,3–2,9 kW. **Winter-Heizbetrieb liegt damit im GUTEN Wechselrichter-η-Bereich** (~88–91 %) — die Heizlast lässt den fixen Teillast-Overhead prozentual verschwinden. Achtung: Kühl-Daten, Heizen bei Kälte zieht anders (COP sinkt, Abtauzyklen) → reale JAZ erst diesen Winter messbar.

**Öl vs. Wärmepumpe — Wärmepreis pro kWh (die faire Kennzahl):**
- Öl: 3.500 l/Jahr, Angebot 114,56 €/100 l = **4.009,65 €**. Heizöl ~10 kWh/l → 35.000 kWh Brennstoff; bei Kessel 85 % = 29.750 kWh Nutzwärme → **~13,5 ct/kWh Wärme** (Spanne 12,1 bei Brennwert / 14,3 bei 80 %).
- Split-WP: Wärmepreis = Strompreis ÷ JAZ. Bei Normaltarif **0,3494 €/kWh** (Netzbetrieb Hirschberg): JAZ 3,0 → 11,6 ct; **JAZ 3,5 → 10,0 ct**; JAZ 4,0 → 8,7 ct.
- **Kernbefund: Schon mit dem heutigen Normaltarif OHNE jeden Trick schlägt die Split-WP (JAZ ≥ 3,5 → ~10 ct) das Öl (~13,5 ct).** Ersparnis pro kWh-Preis (bei gleichbleibender Wärmemenge): JAZ 3,0 −545 €, JAZ 3,5 −1.040 €, JAZ 4,0 −1.411 €. ~~**Realistische Gesamtersparnis aber höher (~2.000–2.800 €/Jahr, s. Erkenntnis 3), weil die Splits mehr Öl-Menge ersetzen als die reine Preisdifferenz-Rechnung annimmt** — die kWh-Preis-Ersparnis ist nur der Effekt auf den ersetzten Anteil.~~
  > ❌ **Rechenfehler, korrigiert 13.08.2026.** Die „2.000–2.800 €" sind die **Brutto-Öl-Einsparung ohne Gegenrechnung des Stroms**, den die Splits für dieselbe Wärme verbrauchen. Beispiel 70 % Deckung: Öl sinkt um 2.807 €, aber es kommen 5.950 kWh Strom (2.079 €) dazu → **netto ~730 €**, nicht 2.800 €. Die Preisdifferenz-Rechnung eine Zeile darüber (−545/−1.040/−1.411 €) ist dagegen **richtig**. ⚠️ Auch die Preisbasis ist überholt (114,56 €/100 l). → aktueller Stand im Abschnitt **„Öl vs. Wärmepumpe: Stand 08/2026"** am Dateiende.

**Drei Erkenntnisse:**
1. **Der große Hebel ist der WP-Wechsel (Faktor JAZ), NICHT die Arbitrage.** Die Batterie-Arbitrage bringt in der Hochpreisregion wenig: Roundtrip 0,83 = 20 % Aufschlag frisst die geringe Tag/Nacht-Spreizung fast auf (Nacht 25 ct über Speicher = 8,6 ct Wärme ≈ direkter Normaltarif 10 ct). Lohnt erst ab Nachttarif Richtung 20 ct.
2. **Für die WP schlägt thermische Speicherung die Batterie:** direkt im billigen dynamischen Fenster heizen (Gebäudemasse/Puffer, ~100 % „Roundtrip") statt Umweg über die 83-%-Batterie. Batterie → PV-Eigenverbrauch + nicht-verschiebbare Lasten.
3. **Mengen-Deckung — Leistung ≠ Jahresarbeit (korrigiert 2026-07-08):** Die frühere „35–52 %"-Zahl war eine reine Leistung-×-Volllaststunden-Rechnung und damit zu pessimistisch. Entscheidend ist die **Jahres-Wärmearbeit** (Heizgradtag-Verteilung): Die milde Übergangszeit + Durchschnitts-Wintertage dominieren mengenmäßig, die wenigen Frost-Tage tragen wenig zur Gesamtmenge bei. Nach **Bivalenz-Prinzip** deckt eine WP mit ~50 % der Spitzenlast typisch **75–85 % der Jahres-Heizarbeit**; der Ölkessel bleibt nur Spitzenlast an den kältesten Tagen. Öl-Rest realistisch: 70 % Deckung → ~1.050 l (−2.807 €), 80 % → ~700 l (−3.208 €).

**Rahmenbedingungen:** Dynamischer Tarif noch nicht möglich (Smartmeter fehlt → dann Modul 3 / dynamische Netztarife); dynamische Preise in der Region generell hoch. Netzbetreiber: Netzbetrieb Hirschberg. Bezug 0,3494 €/kWh, Einspeisung 0,07382 €/kWh.

**Warmwasser bereits elektrifiziert (2026-07-08 bestätigt):** Die **Brauchwasser-Wärmepumpe läuft schon** → Warmwasser hängt nicht mehr am Öl. Damit greift der **Sommer-Effekt** bereits: Öl-WW im Sommer war besonders verschwenderisch (Kessel taktet nur für WW, Nutzungsgrad fällt von ~85 % auf ~55–65 % durch Bereitschafts-/Stillstandsverluste; grob ~320–510 l/Jahr = 370–580 € allein fürs WW). Der Ölkessel kann im Sommer komplett aus (Sommerabschaltung). **Offen/zu klären:** Enthielt der 3.500-l-Wert noch WW (dann ist der Sommer-Anteil eine bereits realisierte Einsparung) oder ist er schon reiner Heizbedarf? → per Tankablesung nach Saison bzw. Beobachtung, ob diesen Sommer überhaupt noch Öl fließt.

**Offene Messung (diesen Winter selbst schließbar):**
- **Reale JAZ der Splits im Heizbetrieb** — wichtigste Zahl, die ganze Rechnung hängt daran (Zigbee-Steckdosen `sensor.0xa085e3fffebc4574_power` / `..._d16b8_power` messen Strom bereits; Wärme grob über Öl-Minderverbrauch gegenrechnen). BW-WP-Verbrauch erfasst Norbert ebenfalls in Kürze — idealer Zeitverschiebungs-Kandidat (WW-Speicher träge → nachts/mittags im günstigen Fenster heizen).

## 🛢️ Öl vs. Wärmepumpe: Stand 08/2026 (konsolidiert, ersetzt die Rechnung von 07/2026)

**Anlass:** Ölpreis deutlich gestiegen + Rechenfehler in der Juli-Analyse gefunden. **Was hier steht, gilt.**

### Neue Preisbasis

| | 07/2026 | **08/2026** | Δ |
|---|---|---|---|
| Heizöl | 114,56 €/100 l | **139,90 €/100 l** | **+22,1 %** |
| Wärmepreis (10 kWh/l, Kessel 85 %) | 13,48 ct/kWh | **16,46 ct/kWh** | |
| 3.500 l/Jahr | 4.009,65 € | **4.896,50 €** | **+887 €** |
| **Break-even-JAZ** (Strom 34,94 ct) | 2,59 | **2,12** | |

**Die Break-even-JAZ ist die entscheidende Zahl.** Sie sagt, ab welcher Arbeitszahl die Split-WP billiger heizt als der Ölkessel. Bei 2,12 liegt sie so tief, dass moderne Inverter-Splits sie **an praktisch jedem Tag des Jahres** überbieten — auch bei −15 °C (dort noch COP ~1,8–2,2). **Damit ist das Argument „die letzten 20 % Wärmearbeit sind unwirtschaftlich" hinfällig**; es galt nur beim alten Ölpreis (Break-even 2,59).

### Szenarien bei 139,90 €/100 l (29.750 kWh Nutzwärme/Jahr)

| Szenario | Kosten/Jahr | Ersparnis |
|---|---|---|
| **A** — weiter nur Öl | 4.897 € | — |
| **B** — Vollverzicht, JAZ 2,5 | 4.158 € (11.900 kWh) | **+739 €** |
| **B** — Vollverzicht, JAZ 3,0 | 3.465 € (9.917 kWh) | **+1.432 €** |
| **B** — Vollverzicht, JAZ 3,5 | 2.970 € (8.500 kWh) | **+1.927 €** |
| **C** — bivalent 70 % (JAZ 3,5) | 3.548 € (2.079 € Strom + 1.469 € Öl) | +1.349 € |
| **C** — bivalent 80 % (JAZ 3,5) | 3.355 € (2.376 € Strom + 979 € Öl) | +1.541 € |

**Wirtschaftlich liegen Vollverzicht und Bivalenz nah beieinander** (1.350–1.930 €) — die Entscheidung fällt daher nicht am Geld, sondern an Technik und Komfort. ⚠️ **Nicht in der Tabelle, aber relevant: Bei echtem Vollverzicht entfallen die Kesselfixkosten** (Wartung, Schornsteinfeger, Tankprüfung) — grob **200–350 €/Jahr**. Damit zieht Szenario B an B/C vorbei.

**Strategisch entscheidend:** Die Stromkosten (2.970–4.158 €) sind **unabhängig vom Ölpreis**. Jede weitere Ölpreissteigerung verbessert die WP-Bilanz automatisch, ohne dass sich an der Anlage etwas ändert — der PV-Eigenverbrauch drückt sie zusätzlich.

### Was Vollverzicht technisch noch fehlt (unverändert gegenüber 07/2026)

Der Ölpreis ändert die **Wirtschaftlichkeit**, nicht die **Physik**:

1. **Leistung an Frosttagen.** Normheizlast grob 12–15 kW (aus 29.750 kWh Jahreswärme); die zwei vorhandenen Splits liefern bei Kälte ~4–5 kW thermisch. Lücke bleibt.
2. **Verteilung.** Splits heizen nur ihren Raum — Schlafzimmer, Bad, Flur haben keine Einheit.

**→ Der Weg zu null Öl führt über weitere Split-Einheiten**, die beide Probleme gleichzeitig lösen und die JAZ heben (jedes Gerät arbeitet entspannter). Bei 1.350–1.930 € Jahresersparnis amortisiert sich eine zusätzliche Einheit (~1.500–2.500 € installiert) **binnen ein bis zwei Jahren**.

### Vorgehen

- **Öl nur noch in Teilmengen kaufen**, nicht 3.500 l am Stück — kein Kapital in einem Brennstoff binden, der eventuell nicht mehr gebraucht wird; Reserve für Extremtage bleibt.
- **Diesen Winter die reale JAZ messen** (Zigbee-Zähler an beiden Splits vorhanden, Wärme über Öl-Minderverbrauch gegenrechnen). Sie entscheidet zwischen +739 € und +1.927 € — die Spanne ist größer als jeder andere Unsicherheitsfaktor.
- **Parallel den Deckungsgrad protokollieren**: An wie vielen Tagen bleibt der Kessel aus? Das beantwortet die Leistungsfrage praktisch statt rechnerisch und zeigt, wie viele Zusatzgeräte wirklich nötig sind.

---

## 💶 Leerlaufsockel in Euro: Saisonrechnung (18.08.2026)

Ergänzt den Abschnitt *„📌 Für den Winter: Entscheidungsgrundlage Multi-Standby"*. **Saisonaufteilung 8 Monate Sommer / 4 Monate Winter — Setzung Norbert**, plausibel für diese Anlagengröße.

> [!WARNING] Frühere Zahlen richtig lesen
> Die zuvor genannten „76 €" und „361 €" waren **keine** Sommer-/Winteranteile, sondern zwei Jahres-**Extremfälle**: der Sockel durchgängig zu Einspeisevergütung bzw. durchgängig zu Bezugspreis bewertet. Die Realität liegt dazwischen — erst die Saisonaufteilung macht daraus eine Jahreszahl.

### Belegt: Im Sommer kostet der Sockel nur Einspeisevergütung

| Monat | Netzbezug | Einspeisung |
|---|---|---|
| Juli 2026 | **3,89 kWh** | 2.447 kWh |
| August 2026 (bis 18.) | **2,36 kWh** | 1.596 kWh |

Netzbezug praktisch null → der Leerlauf frisst ausschließlich PV, die sonst eingespeist worden wäre. Bewertung mit 7,382 ct ist für diese Monate **gemessen, nicht geschätzt**. ⚠️ Der Gridmeter-Sensor existiert erst seit Juli 2026, für den Winter fehlen Messdaten.

### Jahreskosten (243 d × 7,382 ct + 122 d × 34,94 ct)

| Sockel | kWh/Jahr | Sommer | Winter | **Jahr** |
|---|---|---|---|---|
| **118 W** (Regression 13.–17.08.) | 1.034 | 50,80 € | 120,73 € | **171 €** |
| **174 W** (Vault-Regression 08.–10.08.) | 1.524 | 74,91 € | 178,04 € | **253 €** |

Die Spanne von 82 € ist exakt die offene Sockelfrage (110 vs. 175 W, siehe Abschnitt „Leerlaufverlust: Stand der Messungen" oben). **Über 70 % der Kosten fallen in einem Drittel des Jahres an** — der Winter dominiert, weil der Bezugspreis das 4,7-fache der Einspeisevergütung ist.

### Hebbar ist nur der Winteranteil — und dort nur die echten Standby-Stunden

| Standby-Stunden/Tag (Winter) | bei 118 W | bei 174 W |
|---|---|---|
| 4 h | 20 € | 30 € |
| 8 h | 40 € | 59 € |
| **12 h** | 60 € | **89 €** |

Die bisher notierten **65–70 €/Jahr** entsprechen grob **12 Standby-Stunden/Tag beim hohen Sockelwert** — also der optimistischen Ecke beider Unsicherheiten. Realistisch ist eher die Mitte der Tabelle.

### ⬜ Messplan für den kommenden Winter

Beide offenen Fragen brauchen dieselben Daten und lassen sich in einem Aufwasch klären:

1. **Sockelhöhe:** Stunden mit Durchsatz ≈ 0 sammeln und direkt mitteln, statt aus Lastdaten zu extrapolieren.
2. **Hebbare Stunden:** Stunden zählen, in denen AC-Abgabe ≈ 0 **und gleichzeitig Netzbezug** läuft — das ist die Zeit, in der die Batterie leer ist, das Haus am Netz hängt und der Multi nutzlos im Leerlauf steht. Genau diese Stunden sind das Sparpotenzial.

Erst danach ist die Standby-Entscheidung belastbar. Vorher gilt: Im Sommer lohnt Abschalten ohnehin nicht (kostet nur Einspeisevergütung).

---

