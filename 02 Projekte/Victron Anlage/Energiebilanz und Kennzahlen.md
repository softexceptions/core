---
tags: [victron, kennzahlen, wirkungsgrad, verluste, messung]
status: aktiv
date: 2026-07-07
updated: 2026-08-23
---

# Energiebilanz und Kennzahlen

Welche Kennzahl was misst, wie sie zustande kommt und welcher Wert gilt.
**Was hier steht, gilt** — Herleitungen und verworfene Zwischenstände im Journal.

> [!abstract] Die drei Kennzahlen auf einen Blick
> | Kennzahl | Wert | Bedeutung |
> |---|---|---|
> | DC-Roundtrip (Batterie) | ~96,9–98,4 % | Batterie allein, Batrium-Shunt |
> | Wechselrichter-Entladung | ~73 % nachts, ~88–91 % bei Last | stark lastabhängig |
> | **Leerlaufsockel der 3 Multis** | **174–200 W** (offen: evtl. 118 W) | dominiert den Tagesverlust |
>
> Der Nachtwert von 73 % ist **kein Defekt**, sondern der ungünstigste
> Betriebspunkt: fixer Leerlauf bei minimaler Last.

> [!warning] Offene Frage: 118 W oder 174 W?
> Zwei Regressionen widersprechen sich — 174 W (08.–10.08.) gegen 118 W (13.–17.08.).
> Die Differenz verschiebt die Jahreskosten um 82 € und das Standby-Sparpotenzial
> um ein Drittel. Auflösung erst über den Messplan in [[Ausbauplanung]].
> Details: [[2026-08 Journal#⚠️ Zweite Regression widerspricht dem Leerlaufsockel (18.08.2026) — zu klären]]

Verwandt: [[Wirtschaftlichkeit]] (was die Verluste kosten) · [[Zähler und Statistik]] (wie die Zähler gehärtet sind)

## Speicher-Wirkungsgrad: korrekte Kennzahl (2026-07-07)

**Anlass:** `sensor.speicher_wirkungsgrad_monat` (HA-Template `AC-Abgabe_Monat / Batterie-geladen_Monat`) zeigte 184 %. Zwei Fehler: (1) Monats-`utility_meter`-Reset → SoC-Carryover über die Monatsgrenze; (2) **prinzipiell**: Der Zähler `AC-Abgabe` (vebus `InverterToAcIn1 + InverterToAcOut`) zählt **jede** DC→AC-Wandlung — bei DC-gekoppelter MPPT-PV also auch **PV-Durchleitung**, die nie durch die Batterie lief. Deshalb kann der Quotient >100 % werden → es ist kein Wirkungsgrad. (Der Fronius am AC-IN stört nicht, er läuft nie durch die vebus-Wandlung.)

**Korrekte Zerlegung:** `AC-Roundtrip = DC-Roundtrip × Wechselrichter-Entlade-Wirkungsgrad ≈ 96,9 % × ~94 % ≈ 91 %`.
- **DC-Roundtrip** = `sensor.batterie_wirkungsgrad` (entladen/geladen, Lifetime, gleicher Batrium-Shunt) = 96,9 % — war schon sauber.
- **Wechselrichter-Entlade-Wirkungsgrad** — neu gemessen, nur wenn **DC-PV (MPPT) ≈ 0 UND Batterie entlädt**: dann stammt die AC-Abgabe ausschließlich aus der Batterie (keine PV-Durchleitung). Physikalisch eindeutig, praktisch = Nachtstunden.

**Umsetzung (Node-RED, Flow „Victron Speicher-Wirkungsgrad (MQTT)", Live-ID `fa26bca6047459ed`, Variante B):**
- Datenquellen: Rohsignale **direkt vom D-Bus** via `victron-input-*` (autark, projektkonform) — `/History/DischargedEnergy` + `/History/ChargedEnergy` (`com.victronenergy.battery/512`), `/Dc/Pv/Power` + `/Dc/Battery/Power` (`com.victronenergy.system`), je mit nachgeschaltetem `change`-Node für ein eindeutiges Topic. Über MQTT kommt die **gehärtete** Summe `victron/multiplus/ac_out_energy` (AC-Abgabe, DC→AC) — sonst müsste man die vebus-Summe + Härtung duplizieren.
- Function `clspx0func000001` (1 Ausgang → 1 gemeinsamer MQTT-Out, leeres Topic) akkumuliert `accAc`/`accDc` (AC-Abgabe ÷ Batterie-Entladung), **gegatet** auf `Dc/Pv/Power < 30 W` UND `Dc/Battery/Power < −50 W` (entlädt). Tick alle 60 s. **Wichtig:** beide Zuwächse pro Tick mit `>= 0` summieren — NICHT auf `dDc>0` gaten (grober 0,1-kWh-Entladezähler würde die feine AC-Abgabe sonst unterzählen → war der 17-%-Bug). Akku persistiert über retained `victron/battery/_eff_acc`; Reset-Inject `clspx0reset00001` (Topic `reset`).
- Publiziert **erst ab >1 kWh** gesammelter Entlade-Energie → sonst `unknown`.
- **Zwei HA-Sensoren** via MQTT-Discovery:
  - `sensor.victron_speicher_wechselrichter_entlade_wirkungsgrad` (Entlade-Wandlung DC→AC; nachts gemessen, ~80–90 % — Nacht-Last niedrig → Eigenverbrauch drückt, daher unter Datenblatt).
  - `sensor.victron_speicher_ac_roundtrip_inkl_wandlung` — Anzeigename **„PV-Speicher-Nutzgrad (DC → AC)"** = DC-Roundtrip × Entlade-Wandlung, ~91 %. Entity-ID bewusst unverändert (nur Friendly-Name), damit die Gauge auf `victron-sys` nicht bricht.
- **Lade-/Netz-Roundtrip-Sensoren entfernt (2026-07-08):** AC→DC-Laden gibt es zwar (Fronius via MultiPlus bei Schlechtwetter — Cerbo → System-Setup → Ladekontrolle, Ladestrom 5 A → 60 A), ist aber **nicht sauber isolierbar**: tagsüber immer mit DC-gekoppeltem MPPT-Laden vermischt (`charged`-Zähler trennt die Quellen nicht), und ein sauberes MPPT=0-Fenster gäbe es nur beim seltenen Netz-Nachtladen. Statt schlecht zu messen → **Referenzwert** MultiPlus-Ladewandlung ~94 % → Fronius-geladener Roundtrip ≈ 0,94 × 0,97 × ~0,90 ≈ **82 %** (betrifft nur den Schlechtwetter-Anteil).
- Didaktik-Skizze: `docs/speicher-wirkungsgrad.excalidraw` — Lehrdiagramm „was man MISST, was man ABSCHÄTZT": Messgrenze (Batterie-Shunt), Verlust-Klassifikation (gemessen / schon im DC-Roundtrip / upstream MPPT / geschätzt Fronius) + Messtechnik-Lektion (Messgrenze kennen, nur Isolierbares messen, Rest begründen).
- **Reproduzierbarkeit (Generatoren im Repo, 2026-07-08):** Flow-JSON und Skizze werden aus Python-Generatoren erzeugt, nicht von Hand gepflegt:
  - `tools/gen_eff_flow.py` → schreibt `flows/victron-speicher-wirkungsgrad-mqtt.json` **und** `eff_deploy.json` (PUT-Payload, ins CWD). Live-Deploy: `curl -X PUT http://192.168.2.80:1880/flow/fa26bca6047459ed -H 'Content-Type: application/json' --data-binary @eff_deploy.json`. (`eff_deploy.json` ist nur temporär — nicht committen.)
  - `tools/gen_excalidraw.py` → schreibt `docs/speicher-wirkungsgrad.excalidraw`.
  - Also: Änderung immer im Generator machen, neu ausführen, dann deployen — nicht direkt in der JSON editieren.

**Warum nicht als HA-Template-Helper:** Erst versucht, scheiterte an zwei HA-Zicken — der Config-Flow verwirft den `availability`-Key (nicht im Schema), und ein Template-Sensor mit **leerem Render überspringt das Update** (behält den letzten Wert). Eine initial gerenderte 0,0 blieb dadurch kleben. In Node-RED/MQTT gilt „kein Publish = unknown" nativ → sauberer. Template-Helper wieder gelöscht.

**Caveat:** Gemessen wird nur bei niedriger Nacht-Last → spiegelt den Wechselrichter-Wirkungsgrad im typischen Entlade-Betrieb (Abend/Nacht), nicht den Peak bei Volllast.

**Erledigt:** Alter `sensor.speicher_wirkungsgrad_monat` (Fehlkennzahl, 184 %) am 2026-07-07 gelöscht — ersetzt durch den korrekten AC-Roundtrip. Falls ein Dashboard-Card ihn referenzierte, dort auf `sensor.victron_speicher_ac_roundtrip_inkl_wandlung` umstellen.

### Stand & Verifikation (2026-07-08) — ✅ VERIFIZIERT

**Bug gefixt (07.07. abends):** Sensoren zeigten 17 % statt ~90 %. Ursache: grober Entladezähler (`/History/DischargedEnergy`, 0,1-kWh-Stufen) vs. feiner AC-Abgabe-Zähler; Akkumulation gatete auf `dDc>0` → AC nur in Sprung-Minuten gezählt → unterzählt. **Fix:** beide Zuwächse pro Tick `>=0` summieren. Verschmutzter Akku per Reset genullt.

**Vereinfacht (08.07.):** Lade-/Netz-Roundtrip-Sensoren entfernt (AC→DC-Laden nicht isolierbar, s. o.) → es bleiben **zwei** Sensoren. Flow: Function `clspx0func000001` → gemeinsamer MQTT-Out `clspx0out_pub001` (leeres Topic), Tick `clspx0tick00001`, **Reset `clspx0reset00001`**. Persistenz retained `victron/battery/_eff_acc` (JSON accAc/accDc).

**Reset-Befehl:** `curl -X POST http://192.168.2.80:1880/inject/clspx0reset00001` (nullt Akku + löscht die 2 retained Efficiency-Topics).

**Verifikation abgeschlossen (2026-07-08):** Nach der Entlade-Nacht 07.→08.07. haben sich die Sensoren selbst überschrieben und zeigen jetzt:
- `victron_speicher_wechselrichter_entlade_wirkungsgrad` = **72,9 %**
- `victron_speicher_ac_roundtrip_inkl_wandlung` = **71,7 %** (= 72,9 % × 98,4 %, Verrechnung stimmt exakt)
- `batterie_wirkungsgrad` (DC-Roundtrip) = **98,4 %**

**5-Minuten-Gegenprüfung (Nachtfenster 23:00–06:25, PV=0):** Σ `victron_multiplus_ac_abgabe` = 3,62 kWh ÷ Σ `victron_batterie_entladen` = 4,90 kWh = **73,9 %** roh — trifft den Sensor (72,9 %) auf ~1 pp (Rest = Fensterbeginn erst 23:00 + 0,1-kWh-Quantisierung des Entladezählers). **→ Kein Bug, Wert ist real.** Der 17-%-Bug ist damit endgültig erledigt.

**Warum 73 % real sind (kein Defekt):** Reine Teillast-Physik. Verlust 4,90−3,62 = 1,28 kWh über ~7,4 h = **~173 W Dauerverlust** bei nur ~490 W AC-Nutzlast. Der 3×-MultiPlus-II-Verbund (30 kW Nenn) versorgt nachts <2 % seiner Nennlast; der last­unabhängige Eigenverbrauch (~75–90 W, 3 Geräte) + Teillast-Wandlung dominiert prozentual. Bei höherer Last steigt der η stark: ~2 kW → ~91 %, ~5 kW → ~93 % (fixer Overhead bleibt, wird prozentual klein). **Der Nacht-Wert ist der ungünstigste Betriebspunkt, nicht der Alltags-Nutzgrad.** Beantwortet nebenbei die alte Grundlast-Frage: ~173 W mittlerer Verlust > reiner Eigenverbrauch, enthält also Eigenverbrauch + Teillast-Wandlung.

**Gauge-Schwelle offen (Entscheidung von Norbert ausstehend):** `victron-sys`-Gauge „PV-Speicher-Nutzgrad" steht auf grün ≥ 88 → zeigt beim realen ~72-%-Nachtwert dauerhaft rot/gelb. Optionen: (a) Schwelle an Nacht-Realität senken (~70), (b) Sensor erst ab höherer Entladeleistung gaten (näher ans Datenblatt), (c) unverändert lassen (Farbe kosmetisch). Norbert hat das im Kontext der Wärme-/Arbitrage-Diskussion (s. u.) zurückgestellt.

## 📐 Leerlaufverlust: Stand der Messungen (konsolidiert 2026-08-12)

Dieser Abschnitt löst widersprüchliche Angaben an mehreren Stellen der Notiz auf. **Was hier steht, gilt.**

### Die drei Datenpunkte

| Quelle | Wert | Methode | Belastbarkeit |
|---|---|---|---|
| Victron-Datenblatt | **114 W** (3 × 38 W) | Laborwert „zero load power", 25 °C, ohne Last am AC-Out | untere Grenze — im netzparallelen ESS regelt der Multi permanent nach, Search/AES sind wirkungslos |
| Referenznacht 25.07. | **173 W** | Einzelmessung, 7,4 h, 662 W Durchsatz | Größenordnung richtig, aber Sockel und Teillast-Wandlung nicht trennbar |
| **Regression 08.–10.08.** | **174 W** | 72 Stundenpaare, `Verlust = 0,0332 × AC-Abgabe + 0,1736` — der Achsenabschnitt ist per Konstruktion der Wert bei Durchsatz **null** | **maßgeblich** |
| Gegenprobe | **200 W** (151–267) | 12 Stunden mit AC-Abgabe ≈ 0 | bestätigt unabhängig |

### Auflösung

**Der Leerlaufsockel beträgt 174–200 W**, nicht 114 W. Die 173 W der Referenznacht waren bereits der Sockel — die damalige Deutung als „Sockel + Teillast-Wandlung" (Abschnitt „Idee nachts kleiner MultiPlus", 2026-07-26) ist widerlegt. Regression und Gegenprobe messen unabhängig voneinander dasselbe; das Datenblatt ist der zu optimistische Laborwert. Pro Gerät sind das ~60 W statt 38 W.

⚠️ Es bleibt eine **Bilanzgröße** aus der Differenz zweier Messwelten (Batrium/MPPT-Zähler gegen vebus-Zähler). Enthalten sind neben dem Multi-Eigenverbrauch auch Cerbo, BMS, DC-Verkabelung und ein möglicher Kalibrierungsoffset. Auf die letzten ~20 W nicht festlegen.

### Was daraus folgt — korrigierte Werte

| Größe | alt (Notiz 25.07.) | **gültig** |
|---|---|---|
| Standby-Anteil am Tagesverlust | 60 % | **80–92 %** |
| Tageszerlegung (bei 21,4 kWh AC) | 2,74 fix + 1,9 Wandlung | **4,17 fix + 0,7–1,0 Wandlung** |
| Wandlungswirkungsgrad | ~93 % (Durchschnitt) | **96,7 % marginal** (3,3 % je zusätzlicher kWh) |
| Leerlauf pro Jahr | 1.000 kWh ≈ 350 € | **1.520–1.750 kWh** |

Die alte Zerlegung ging rechnerisch auf (2,74 + 1,9 = 4,6 ✓), verteilte die Anteile aber falsch. **Kernaussage bleibt und verstärkt sich sogar:** Der Löwenanteil des „Wandlungsverlusts" ist Bereitschaftsverbrauch, nicht Wandlung — die Wandlung selbst ist mit ~96,7 % ausgezeichnet.

### Weitere vereinheitlichte Angaben

- **Batteriekapazität:** 280 Ah bei 16S-LiFePO4 (51,2 V nominal) = **~14,3 kWh brutto**, bei realer Betriebsspannung ~52,5 V rund 14,7 kWh. Die Angabe „≈ 15 kWh" (Abschnitt Batrium-Zähler) ist die gerundete Obergrenze — beide meinen dasselbe Pack. **Nutzbar bei MinSoC 20 %: ~11,5 kWh**, nach dem Ausbau auf 2 Packs **~23–24 kWh**.
- **Standby-Sparpotenzial im Winter:** Die Angaben „~70 €/Jahr" (2026-07-26) und „~65 €/Jahr" (Winter-Abschnitt) sind dieselbe Größenordnung aus leicht verschiedenen Annahmen. **Verbindlich: 65–70 €/Jahr als obere Grenze**, die durch Batterie-Ausbau und Heizbetrieb weiter sinkt.
- **Die „350 €"** aus der Kostenbezifferung sind der *physikalische* Verlust bei durchgängigem Winterstrompreis, nicht der hebbare Anteil. Mit dem korrigierten Sockel wären es rechnerisch 530–610 €/Jahr — **aber nur ein Bruchteil davon ist überhaupt vermeidbar**, weil die Multis 20 h/Tag arbeiten. Für Entscheidungen ist ausschließlich der hebbare Anteil relevant.

### ⬜ Offene Gegenprobe

Nach dem Batterie-Ausbau die Regression wiederholen. **`P_fix` muss unverändert bei 174–200 W liegen** — am Leerlauf der Multis ändert ein zweites Batteriepack nichts. Kommt ein anderer Wert heraus, war ein Teil des gemessenen „Leerlaufs" in Wahrheit ein Kalibrierungsoffset zwischen den Messwelten.

---

