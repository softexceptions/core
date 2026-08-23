---
title: Wallbox Mieterabrechnung
tags:
  - projekt
  - aktiv
  - energie
  - recht
status: offen
stand: 2026-08-17
date: 2026-08-18
---

# Wallbox-Abrechnung gegenüber dem Mieter

> [!warning] Kernproblem
> Die **openWB Pro ist nach derzeitiger Erkenntnis nicht eichrechtskonform** — nur der
> verbaute Zähler ist MID-konform. Für eine kWh-Abrechnung gegenüber einem **Mieter** wird
> nach MessEG/MessEV jedoch eine eichrechtskonforme Ladeeinrichtung verlangt. Die laufende
> Abrechnung steht damit auf unsicherem Grund.

## Hardware-Befund (2026-08-17, aus `/api/v1/status` der Wallbox)

| Feld | Wert |
|---|---|
| Modell | openWB Pro |
| Hardware-Version | V0R5e |
| Software-Version | 3.2.1 |
| Seriennummer | 823088 |
| IP | 192.168.2.145 |
| Zähler | `eastron` (SDM630) |
| `eichrecht_protocol` | **`none`** |

Abruf: `curl -s http://192.168.2.145/api/v1/status`

## Der Unterschied, um den es geht

| | MID-Zähler | Eichrechtskonforme Ladeeinrichtung |
|---|---|---|
| Zertifiziert ist | die **Messhardware** | die **gesamte Kette** bis zur Rechnung |
| Manipulationssichere Speicherung | nein | ja |
| Signierte Datensätze (OCMF) | nein | ja |
| Nachprüfbar durch Transparenzsoftware | nein | ja (S.A.F.E.) |

Der Eastron SDM630 **misst** genau — am 2026-08-17 gegen evcc verifiziert, Abweichung der
Zählerstände 0,03 kWh. „Genau messen" und „rechtssicher abrechnen" sind aber zwei
verschiedene Anforderungen.

## Belege

**openWB-Maintainer Kevin Wieland** (GitHub Discussion #3479, DKV-Anbindung):

> „Eichrechtskonform ist ein (deutscher) Mechanismus der ontop zu geeichten Zählern kommt und
> dir sicherstellt das du bei MID geeichtem Zähler xy von kWh Zählerstand 7428 bis 7439 (also
> 11kW) geladen hast."

Aus derselben Diskussion: Für eine reguläre öffentliche Wallbox wäre Eichrechtskonformität
nötig, *„this is not currently provided by openWB"*. Die formale Baumusterprüfbescheinigung
wurde wegen Kosten und begrenzter Nachfrage nicht angegangen. **Praxisbeleg:** DKV akzeptiert
die openWB nicht — dort genügt MID ausdrücklich nicht.

**Produktseite der Pro** nennt zur Messtechnik nichts; die Varianten betreffen Kabellänge,
Kabelhalter und RFID. Eine Eichrechts- oder „Pro+"-Variante existiert dort nicht. Das `none`
ist demnach kein deaktiviertes Feature, sondern der Normalzustand dieser Hardware.

> [!tip] Warum die Verwechslung so verbreitet ist
> In Foren und Testberichten steht oft, die openWB habe „einen MID-geeichten Zähler, der jede
> kWh eichrechtlich konform misst". Der Zähler *misst* geeicht — die Anlage ist damit nicht
> eichrechtskonform. Selbst Suchmaschinen geben zu der Frage widersprüchliche Antworten.

## Rechtlicher Rahmen (keine Rechtsberatung)

- Abrechnung nach exakten kWh **gegenüber Dritten** verlangt eine eichrechtskonforme
  Ladeeinrichtung. **Vermietung an Mieter wird ausdrücklich als Anwendungsfall genannt.**
- Die oft zitierte Ausnahme, bei der MID praktisch akzeptiert wird, betrifft die **Erstattung
  durch den eigenen Arbeitgeber** bei Dienstwagen — nicht die Mieterabrechnung.
- Offene juristische Nuance: Gilt die Konstellation als „Abgabe von Ladestrom im
  geschäftlichen Verkehr" (Ladepunkt-Eichrecht) oder als Weiterverteilung von Stromkosten im
  Mietverhältnis (Zähler-Eichrecht wie bei Wohnungszählern)? Diese Grenze verläuft nicht
  trivial und entscheidet über den Lösungsweg.

## Handlungsoptionen

1. **Pauschale statt kWh-Abrechnung** — ohne Abrechnung nach Messwert ist das Messwesen nicht
   betroffen. Muss vertraglich geregelt sein; bei schwankendem Ladeverhalten für eine Seite
   unfair.
2. **Separater geeichter Zähler im Zählerschrank** — der klassische Weg, wenn der Mieter Strom
   bezieht wie über einen Wohnungszähler. Mit Elektrofachkraft und ggf. Eichbehörde klären.
3. **Eichrechtskonforme Wallbox** für diesen Ladepunkt beschaffen.
4. **Nachrüstung** — laut Recherche nicht vorhanden, aber beim Hersteller gegenprüfen.

## Offene Punkte

- [ ] **Support-Mail an `support@openwb.de`** verschickt? (Text unten, verfasst 2026-08-17)
- [ ] Antwort erhalten und eingeordnet — besonders darauf achten, ob zwischen MID und
      Eichrecht klar getrennt wird
- [ ] Rechtliche Klärung: Eichbehörde des Landes oder mietrechtliche Auskunft
- [ ] Entscheidung über Handlungsoption und Umstellung der laufenden Abrechnung
- [ ] Prüfen, ob bereits erfolgte Abrechnungen angreifbar sind

## Support-Anfrage (Stand 2026-08-17, noch nicht verschickt)

Betreff: `Eichrechtskonformität openWB Pro (SN 823088) – Abrechnung gegenüber Mieter`

Kernfragen der Mail:

1. Ist das Gerät eichrechtskonform im Sinne MessEG/MessEV?
2. Falls ja: Bescheinigung (Baumusterprüfbescheinigung/Konformitätserklärung) zusenden
3. Ist der Eastron SDM630 MID-konform, liegt eine Bescheinigung vor?
4. **Wofür steht `eichrecht_protocol`?** Gibt es Varianten/Firmware, bei denen ein signiertes
   Protokoll (OCMF) ausgegeben wird? ← die entscheidende Frage: Hardwarelimit oder
   Konfigurationszustand?
5. Nachrüstoption vorhanden? Falls nein: Welche Lösung empfiehlt openWB für die Abrechnung
   gegenüber einem Mieter?

Bitte um **schriftliche** Antwort, begründet mit der Rechtssicherheit der laufenden
Abrechnung — auch dann, wenn das Ergebnis lautet, dass die Pro dafür nicht vorgesehen ist.

## Nebenbefund: evcc-Zahlen taugen nicht für die Abrechnung

Bei der Analyse fiel auf, dass die evcc-Ladehistorie von Mai bis August 2026 rund **9 % zu
niedrig** war (Ursache: zwei parallel laufende evcc-Instanzen auf derselben Wallbox, seit
2026-08-17 behoben). Für die Abrechnung war das nur deshalb unkritisch, weil die openWB-Werte
messtechnisch korrekt sind. **Für Abrechnungszwecke immer die openWB-Zählerstände nehmen, nicht
die evcc-Session-Historie.** Details in [[homelab-ansible]].

## Verwandte Notizen

- [[homelab-ansible]] — evcc-Playbooks, Doppel-Instanz-Analyse, Zählervergleich
- [[homelab-infrastruktur]] — Container, IPs, DHCP-Bereich der UDM Pro
- [[Victron Anlage]] — PV-/Speicheranlage, aus der der Ladestrom überwiegend kommt

## Quellen

- [DKV-Anbindung zur Dienstwagen-Abrechnung mit openWB — Discussion #3479](https://github.com/openWB/core/discussions/3479)
- [openWB Pro — Produktseite](https://openwb.de/shop/?product=openwb-pro)
- [Geeichter Zähler — openWB Forum](https://forum.openwb.de/viewtopic.php?t=1905)
- [MID oder eichrechtskonforme Wallbox — Dymbol Elektrotechnik](https://dymbol.de/blog/mid-eichrechtskonforme-wallbox-erklaert.html)
- [Eichrechtskonformität vs. MID — energieloesung.de](https://www.energieloesung.de/magazin/Eichrechtskonformitaet-vs.-MID-das-solltest-du-wissen/)
- [Wallbox mit Abrechnungssystem: Rechtssicher laden — emobility.energy](https://www.emobility.energy/e-auto-magazin/wallbox-abrechnungssystem-dienstwagen-eichrecht)
