---
tags: [projekt]
status: aktiv
erstellt: 2026-04-27
deployed: true
url: https://ihkmatrix.softexceptions.com
---

# IHK-Bewertungssoftware

Assistenzsystem zur Unterstützung bei der Bewertung von IHK Abschlussprüfungen (Projektbewertung).

## Ziel

Ein robustes, erweiterbares Assistenzsystem entwickeln, das die Bewertung komplexer Abschlussprojekte strukturiert unterstützt und validiert.

## Status

Größtenteils fertig, soll erweiterbar sein.

## 🚧 Aktueller Arbeitsstand — Einstieg für 2026-06-16

**Wo wir stehen (Kurzfassung):** Konsistenz UND Niveau sind gelöst. Der Vision-Umbau ist committet, deployt und gemessen — **Niveau wieder bei 87** (mit einem Grenzfall). Die kurzzeitige Eskalation auf 90 (Spekulation über ungelesene Anhänge) ist behoben. Der Grenzfall wurde diagnostiziert (Kat. 2 Kostenplanung) und daraus eine Deckelungs-Regel abgeleitet. **Korrektur 2026-06-15: Die beiden zuvor als „NICHT committet" geführten Pakete (Grenzfall-UI-Fix + Deckelungs-Regel) sind committet — beide in `0e15993` „Konsistenz UND Niveau". Offen bleibt für diese beiden: Deploy + 3×-Messung (UNGEMESSEN).** Ebenfalls committet (2026-06-15): der Retry-/PDF-Löschungs-Fix in `07ee45a` „*.pdf länger auf Laufwerk" (siehe eigener Block unten). **Working Tree ist clean; alle drei Pakete warten nur noch auf Deploy.**

**✅ 2026-06-15: Retry-/PDF-Löschungs-Fix (committet in `07ee45a` „*.pdf länger auf Laufwerk", Deploy offen).** Anlass: Beim Bewerten trat ein `Error code: 500 / api_error / Internal server error` auf — ein **transienter Fehler auf Anthropics Seite** (request_id belegt Ankunft), kein Code-Bug. Beim Nachvollziehen der Fehlerkette zwei nicht offensichtliche Schwächen gefunden: **(1)** Die vorhandene Celery-Retry-Mechanik (`max_retries=2`, 30 s Delay) lief bei genau solchen transienten Fehlern **ins Leere**, weil das `finally` in `run_evaluation` das Prüflings-PDF bei *jedem* Durchlauf löschte — der Retry 30 s später fand die Datei nicht mehr. **(2)** Selbst mit PDF hätte der Retry sofort an der Statusmaschine zerbrochen: Der erste Fehlversuch setzt den Job per `job.fail()` auf `FAILED`, `start_extraction()` verlangt aber per `_assert_status` `PENDING` → ValueError, der die echte Ursache (den 500er) in der `error_message` überschrieb. **Fix (3 Teile, untrennbar):** (a) neue Domain-Methode `EvaluationJob.reset_for_retry()` (Status → `PENDING`, `error_message` → None); (b) `run_evaluation` setzt beim Wiedereinstieg zurück und löscht das PDF **nicht** mehr (`finally` + `import os` raus, `job.fail` bleibt); (c) neuer Helper `_delete_job_pdf(job_id)` in `evaluation_tasks.py` löscht das PDF nur am **endgültigen** Ausgang (Erfolg, `SoftTimeLimitExceeded`, `MaxRetriesExceededError`) — **nicht** im `self.retry`-Pfad. Geänderte Dateien: `domain/models/evaluation_job.py`, `application/services/evaluation_orchestration_service.py`, `infrastructure/celery/evaluation_tasks.py`, `tests/domain/test_evaluation_models.py` (+1 Test). Tests **53/53** grün. **Trade-off (DSGVO-relevant):** Das PDF bleibt nun zwischen Retries kurz liegen (max ~1 Min); bei hartem Worker-Kill (SIGKILL) gar nicht automatisch gelöscht — war aber auch vorher so (`finally` läuft bei SIGKILL nicht). Natürlicher Aufräumort dafür: der geplante **Watchdog (Härtung 3)** sollte verwaiste PDFs mit abräumen. Commit-Vorschlag: `fix: PDF-Löschung in den Celery-Task verschieben, damit Retries greifen`.

**✅ 2026-06-14/15: Vision-Umbau — committet, deployt, gemessen → 87 bestätigt.** Anhang-/Screenshot-Seiten ohne extrahierbaren Text werden zu PNG gerendert (`pypdfium2`, lazy import) und als Bild-Blöcke an den Extractor gegeben — Claude liest ihren Inhalt per Vision. Der frühere `pdf_hinweis` („ungelesene Seiten nicht als fehlend werten" = Spekulation zugunsten) ist ersatzlos gestrichen. Geänderte Dateien: `pdf_extractor.py` (`image_page_indices` + `render_pages_to_png`), `claude_client.py` (`_build_user_content` → multimodale Message, `images`-Param), `extractor_agent.py` + `IDocumentExtractor` (`image_pages`-Param, Bildseiten-Block im Systemprompt), `evaluation_orchestration_service.py` (rendert Bildseiten, `pdf_hinweis` raus), `category_evaluator_agent.py` (toten Hinweis-Block entfernt), `config.py` (`extractor_read_image_pages`, `_image_render_scale=2.0`, `_max_image_pages=20`), `requirements.txt` (`pypdfium2==5.9.0`). **Messung nach Deploy: 87, ein Grenzfall** — Diagnose („Schüler ist gut" vs. „zu großzügig") damit zugunsten ersterer entschieden, der gelesene Anhang-Inhalt stützt die Bewertung.

> **🔴 Datenschutz — jetzt produktiv, Klärung offen:** Bildinhalte laufen NICHT durch Presidio (kann nur Text schwärzen), bewusst auf die Anhang-Bildseiten begrenzt (Textseiten weiter anonymisiert). Da der Umbau **bereits deployt** ist, ist die Frage nicht mehr „vorher", sondern akut: prüfen, ob das Senden un-anonymisierter Bildseiten vom Consent/DSGVO-Konzept gedeckt ist (`Erweiterung_Dsvgo.odt`); ggf. Einwilligung anpassen.

**✅ 2026-06-15: Grenzfall-Begründung im Frontend sichtbar gemacht (committet in `0e15993`, Deploy offen).** Problem: Das Grenzfall-Badge zeigte „bitte prüfen", aber nicht *worauf* sich der Grenzfall bezieht. Ursache: Das `reasoning`-Feld (KI-Entscheidungsprotokoll A-C, benennt die verworfene Alternative) wird zwar erzeugt und in der DB gespeichert, war aber nie im `CategoryEvaluationDto` — also nie im Frontend. Fix (4 Dateien): `evaluation_dto.py` (+`reasoning`, gemappt aus `ai_reasoning`), `evaluation_result.py` (Kommentar korrigiert), `EvaluationResult.ts` (Feld + `fromDto` + `hasReasoning()`), `CategoryRow.vue` (violetter Block „Warum Grenzfall? – Abwägung der KI", sichtbar bei `grenzfall`). Rein Anzeige-Logik, keine DB-Migration, **rückwirkend** für Bestandsbewertungen sichtbar (reasoning lag schon in der DB). Tests 52/52 grün, Frontend-Build (inkl. `vue-tsc`) grün. Zwischenweg ohne Deploy: `scripts/keicher_messung.py` druckt die Grenzfall-Begründungen bereits aus der DB. Commit-Vorschlag: `feat: Grenzfall-Begründung im Ergebnis anzeigen (reasoning ins DTO + Frontend)`.

**✅ 2026-06-15: Deckelungs-Regel für fehlende Pflicht-Elemente (committet in `0e15993`, Deploy + Messung offen → NICHT gemessen).** Diagnose des aktuellen Grenzfalls (Kat. 2): Personal/Sachmittel/Termine/Ablauf top, nur Kostenplanung schwach (Stunden, keine Geldbeträge). Das Modell schwankte nicht über die Kosten (klar: nur ableitbar), sondern über die **Aggregation** — die IHK-Rubrik presst alle 5 Aspekte in eine Stufe und hatte keine Regel dafür, ob ein schwacher Pflicht-Aspekt die ganze Kategorie deckelt (Streuung 96/86/58 bei genau dieser Kategorie). Das ist der ganz am Anfang genannte strukturelle Befund, jetzt im Realfall. Fix: **DECKELUNGS-REGEL** in `systemprompt.md` Schritt C — ein nicht EXPLIZIT belegtes Pflicht-Element (FEHLT/NUR ABLEITBAR) deckelt die Kategorie auf **höchstens befriedigend (80-67)**, unabhängig von den übrigen Aspekten (Norberts Deckel-Entscheidung: befriedigend, nicht ausreichend). Kat. 2 zusätzlich konkretisiert: Kostenplanung braucht Geldbeträge (Stundensatz × Stunden, Euro; 0 € gilt, wenn benannt). Geänderte Dateien: `systemprompt.md`, `kategorien/02-ressourcen-und-ablaufplanung.md`. Template-`format()` + Rubrik-Laden verifiziert, Tests 52/52 grün. Commit-Vorschlag: `feat: Deckelungs-Regel — fehlendes Pflicht-Element deckelt Kategorie auf befriedigend`.

> **⚠️ Die Deckelungs-Regel wirkt GENERELL über alle 6 Kategorien**, nicht nur Kat. 2. Bei der Messung auf Niveau-Verschiebungen in ALLEN Kategorien achten (z.B. Kat. 1, wenn Einstieg/Ausstieg fehlt). Falls zu hart: Obergrenze nachjustieren oder „Pflicht-Element" enger fassen.

**🟡 GEPLANT (2026-06-15, Umsetzung MORGEN): Echten Claude-Code-Skill für die Bewertungs-Orientierung anlegen.** Architektur-Klärung mit Norbert — „Skill" meinte bisher zwei verschiedene Dinge, die getrennt gehören:
- **(1) Die Bewertungs-Pipeline** (`systemprompt.md` + `kategorien/`, vom `rubric_loader` deterministisch in JEDEN Call geladen) ist ein **Single-Call-Workflow, kein Agent**. Sie bleibt Prompt-Template. Ein echter Anthropic-Skill (progressive disclosure / Managed-Agents-Skill) wäre hier das FALSCHE Werkzeug — würde Determinismus, Parallelität (6 Kategorien) und Kostenkontrolle gegen Agenten-Magie tauschen, für eine Aufgabe, die gar kein Agent ist. **Nicht umbauen.**
- **(2) Die Orientierung, die Claude beim Arbeiten AM Bewertungssystem leitet** — DAS ist der echte Skill-Anwendungsfall (das meinte Norbert mit „der Skill ist für dich").

> **Umsetzungsplan morgen:** Echten Claude-Code-Skill `.claude/skills/ihk-bewertung/SKILL.md` anlegen — Frontmatter (`name`, `description: "Beim Arbeiten an der IHK-Bewertung: Scoring-Regeln ändern, Grenzfälle, Rubriken, Mess-Methodik"`); Body = die **Geltenden Bewertungsgrundsätze** (stehen aktuell als Abschnitt in `backend/skills/bewertung/SKILL.md`) + verbindliche Methodik (einzeln einbauen + 3× messen) + klarer Verweis „operative Wahrheit steht im `systemprompt.md`". Die Grundsätze NUR an EINE Stelle ziehen (den echten Skill), keine Doppelung. `backend/skills/bewertung/` bleibt Produktions-Config (Name ist historisch). Der echte Skill orientiert Claude beim ENTWICKELN — er läuft NICHT in der Produktions-Bewertung mit. Die Auto-Memory-Krücke kann dann entfallen.

**Nächste Schritte:**
1. **Deploy:** Alle drei Pakete sind committet (Grenzfall-UI-Fix + Deckelungs-Regel in `0e15993`, Retry-/PDF-Fix in `07ee45a`), Working Tree clean — **noch nicht deployt**. Gemeinsam via `sync.sh` ausrollen (Frontend-Rebuild, keine Migration; vorher Job-Check: `SELECT count(*) FROM evaluation_jobs WHERE status NOT IN ('completed','failed');` muss 0 sein). Hinweis: Der SKILL.md-Grundsätze-Abschnitt wandert ggf. in den Claude-Code-Skill (3) — erst danach final.
2. **Deckelungs-Regel 3× messen** (Keicher, `keicher_messung.py`). Erwartung: Kat. 2 fällt von 58/Grenzfall auf eindeutige 74; Gesamtniveau evtl. leicht unter 87, falls weitere Pflicht-Elemente in anderen Kategorien greifen — genau beobachten.
3. **Claude-Code-Skill anlegen** (siehe gelben Plan oben) — Norberts ausdrücklicher Wunsch, morgen umsetzen.
4. **DSGVO-Deckung der Bildseiten klären** (siehe rote Notiz oben) — bleibt höchste Priorität, da bereits produktiv.
5. Reserve (aktuell NICHT nötig): der **Mittelwert-Anker** selbst (nur 6 mögliche Kategorie-Werte) bleibt unangetastet — die Deckelung adressiert nur die Teilerfüllungs-Aggregation, nicht die Diskretisierung.

---

### Verlauf (chronologisch, 2026-06-12/13)

**✅ MESSUNG ABGESCHLOSSEN 2026-06-12 abends — voller Erfolg:**
- Keicher (31 Seiten) 3× bewertet: **87 / 87 / 88** (Baseline 11.06: 67 / 72 / 66)
- **Streuung 6 → 1 Punkt**; Baseline kippten 5 von 6 Kategorien, jetzt 4 von 6 punktidentisch, in Lauf 3 zwei kompensierende Ein-Stufen-Flips (Portfolio 86→96, Kundendoku 96→86)
- Kundendoku 40–58 → 86–96. **Produktions-Beleg für den 15K-Bug:** Alle drei Baseline-Begründungen sagten wörtlich „Anhang A.5 nicht einsehbar/nicht enthalten" — das Modell hat das fehlende Einlesen selbst dokumentiert.
- A/B/C-Protokoll steht sauber mit Zitaten in allen neuen reasoning-Feldern
- Beobachtung: Lauf 3 nutzte den Bildseiten-Hinweis als Abstufungsgrund (96→86 wegen „7 von 31 Seiten nicht lesbar") — Schärfung als Maßnahme in SKILL.md notiert
- Unterwegs behoben (je committet + deployt): Zahlen-Artefakte aus json-repair in strengths (Lauf-3-Crash) + fehlende Pflichtfelder → jetzt Validierung + 1× Retry im CategoryEvaluatorAgent (Tests 38/38)
- Mess-Skript für künftige Läufe: `scripts/keicher_messung.py` (per scp auf Server, umgeht SQL_ASCII-Problem)

**✅ 2026-06-13: Grenzfall-Flag + Feldreihenfolge umgesetzt (lokal, NICHT committet/deployt).** Bewusst gebündelt — nach 87/87/88 ist die 3×-Messung nur noch Regressions-Check (Einzeleffekte unterhalb der Auflösungsgrenze). Umfang: GRENZFALL-REGEL + neue Feldreihenfolge (`reasoning` → `grade_label` → `score` → `grenzfall`) in `systemprompt.md`; Flag durch die ganze Pipeline (Agent-Validierung mit bool-Normalisierung → Domain → Repository inkl. Bestandsdaten-Default → DTO → Frontend-Badge „Grenzfall – bitte prüfen" in CategoryRow.vue, violett). Manuelle Score-Korrektur setzt das Flag zurück. Tests 44/44, Frontend-Build grün. Commit-Vorschlag: `feat: Grenzfall-Flag und reasoning-zuerst im Bewertungs-Output`.

**✅ 2026-06-12 spät: Grenzfall-Flag deployt + Regressions-Messung gelaufen — Niveau-Shift entdeckt:**
- Keicher 3×: **94 / 93 / 91** (vorher 87/87/88) — kreuzt die 92er-Grenze („sehr gut")
- Ursache identifiziert: Neue Feldreihenfolge macht das A/B/C-Protokoll voll bindend; Schritt B wertet **Eigenaussagen des Prüflings als EXPLIZIT-Beleg** („Vollständig: Alle Funktionen dokumentiert" → Bestnote). Vorher dämpfte das holistische Gesamturteil (Beleg: Lauf 21:12 kam mit denselben Zitaten zu 86 statt 96). 4 Kategorien geschlossen 86→96; Ausgangssituation zugleich 86→74/74/84 — Protokoll wirkt in beide Richtungen, keine pauschale Milde.
- Grenzfall-Flag: feuert treffsicher (beide Fälle echte Grenzfälle, z.B. Ressourcen 86: Personalkosten ohne Stundensatz → „transparent" nicht erfüllt), aber inkonsistent zwischen Läufen — ergänzt Wiederholungsmessung, ersetzt sie nicht.
- Nebenbefund: erneut json-repair-Schnitt (abgeschnittenes reasoning in Lauf 22:08) → Tool-Use-Umbau bleibt auf der Liste.

**✅ 2026-06-13: BELEG-REGEL umgesetzt (lokal, NICHT committet/deployt).** Norberts Prüfer-Urteil: Eigenaussagen des Prüflings sind überhaupt KEIN Bewertungskriterium; 87/87/88 war realistisch. Regel in `systemprompt.md` (STUFEN-DEFINITIONEN + Verweis in Schritt B): Selbsteinschätzungen werden vollständig ignoriert, bewertet wird nur nachprüfbarer Inhalt. Tests 45/45. Commit-Vorschlag: `feat: Beleg-Regel — Eigenaussagen des Prüflings sind kein Bewertungskriterium`.

**🔴 2026-06-13: Beleg-Regel allein wirkt NICHT (Messung: weiter 93) → Wäsche-Befund + Extractor-Fix:**
- Diagnose per Begründungs-Vergleich: Der **Extractor fasst zusammen statt zu zitieren** — dabei geht verloren, ob ein Satz Eigenaussage des Prüflings oder echter Inhalt ist („Herkunfts-Wäsche"). Der Evaluator kann die Beleg-Regel ohne diese Information nicht anwenden; er zitierte nach dem Deploy dieselben beschreibenden Bullets wie vorher. Gegenprobe: Ausgangssituation (echter Text im Abschnitt) wird sauber kritisch bewertet (76, Grenzfall, „Einstieg/Ausstieg FEHLT").
- **Fix (lokal, NICHT committet/deployt):** Extractor-Systemprompt mit OBERSTER REGEL „wörtlich extrahieren, niemals zusammenfassen" (Auslassungen mit […] erlaubt; Begründung im Prompt: Prüfer müssen Behauptung von Inhalt unterscheiden können), Kundendoku-Anweisung auf echten Anhang-Inhalt statt beschreibendes Kapitel, `max_tokens` 8192 → 16384 (sichere Grenze ohne Streaming; Haiku 4.5 könnte 64K), Warnung bei leeren Abschnitten. Tests 45/45. Commit-Vorschlag: `fix: Extractor extrahiert wörtlich statt zusammenzufassen (Herkunfts-Wäsche)`.
- Architektur-Lektion (auch in SKILL.md): Der Extractor war kein neutraler Durchleiter, sondern ein unauditierter Vor-Richter — seine Paraphrasen wurden downstream als Fakten zitiert. Bestätigt die SKILL.md-Warnung „Extraktionsfehler = gemeinsamer Single Point of Failure".

**Nächste Schritte:**
1. Commit + Deploy (`sync.sh`, vorher Job-Check)
2. Keicher 3× messen — Erwartung: Niveau sinkt Richtung ~87, weil die Beleg-Regel jetzt greifen kann. Prüfen: (a) zitieren die Kat.-6-Begründungen echten Handbuch-Inhalt (Syntax-Beispiele) statt Beschreibungs-Bullets, (b) Mehrkosten im Kosten-Log (größere Extractor-Outputs), (c) worker.log auf „Abschnitt fehlt oder ist leer"-Warnungen
3. Danach ggf. Grenzfall-Regel asymmetrisch schärfen / Bildseiten-Hinweis / Begriffe operationalisieren (SKILL.md)

---

**Erledigter Plan von 2026-06-12 (Referenz):**

1. **Commit** der drei lokalen Arbeitspakete (Working Tree ist NICHT clean!):
   - `feat: Timeouts für Celery-Tasks und Anthropic-Client (Härtung 2)`
   - `refactor: Bewertungsrubriken als editierbaren Skill externalisieren` (+ Stufen-Operationalisierung)
   - `fix: Dokumente vollständig einlesen statt nur erste 15.000 Zeichen`
   (zusammen oder getrennt — Norberts Entscheidung; Tests aktuell 28/28 grün)
2. **Deploy:** `bash sync.sh` aus Norberts interaktiver Shell — vorher prüfen, dass kein Job läuft (`SELECT count(*) FROM evaluation_jobs WHERE status NOT IN ('completed','failed');` muss 0 sein)
3. **Konsistenz-Messung:** dieselbe Test-PDF **3×** bewerten.
   - ⚠️ **Korrektur 2026-06-12:** Die Baseline 66/72/67 wurde mit der Dokumentation **Keicher** gemessen, NICHT mit Jotzo (frühere Fassung dieser Notiz hatte das verwechselt). Jotzo war nur das Analyse-/Fixture-PDF für den 15K-Kürzungsbefund. Für einen sauberen Varianz-Vergleich muss wieder die **Keicher-PDF** verwendet werden — sie liegt nicht im Repo (Server löscht PDFs nach Bewertung), Norbert muss die Originaldatei bereithalten.
   - Varianz vergleichen mit Baseline 66/72/67 (nur Varianz-Vergleich gültig — das absolute Niveau verschiebt sich, weil das System Kundendoku + Anhänge jetzt erstmals wirklich liest, vermutlich nach oben, v.a. Kategorie 6)
   - `reasoning`-Feld der Kategorie 6 prüfen: Steht dort das A/B/C-Protokoll über den echten Inhalt der Kundendokumentation?
   - Misst den **Kombinationseffekt** von 3 ungemessenen Änderungen: Stufen-Operationalisierung + vollständiges Einlesen + Bildseiten-Hinweis
4. **Nach der Messung (beschlossen 2026-06-12):** zwei Skill-Verbesserungen **einzeln** einbauen und jeweils erneut 3× messen: (1) JSON-Feldreihenfolge `reasoning` vor `score`, (2) Grenzfall-Flag in Schritt C + Ausgabeformat. Weitere priorisierte Ideen (Begriffs-Operationalisierung, Kategorie-5-Epistemik, Beleg-Verifizierung im Code) stehen in der SKILL.md-Maßnahmenliste.

---

**Eventloop-Bug behoben — deployt und auf dem Server per Log VERIFIZIERT (2026-06-11).**

**Danach: Härtung 2 (Timeouts) lokal umgesetzt — noch NICHT committet, noch NICHT deployt.** Details siehe Checkliste unter „Nächste Schritte". Geänderte Dateien: `config.py` (4 neue Settings), `claude_client.py` (timeout/max_retries), `celery_app.py` (Time-Limits), `evaluation_tasks.py` (SoftTimeLimitExceeded-Handling + `_mark_job_failed`-Helper). Commit-Vorschlag: `feat: Timeouts für Celery-Tasks und Anthropic-Client (Härtung 2)`.

**2026-06-12: 🔴 Gravierender Pipeline-Befund + Fix (lokal, NICHT committet/deployt):** Der ExtractorAgent sah nur die **ersten 15.000 Zeichen** des Dokuments — beim 42-Seiten-Beispiel-PDF (Jotzo, 58.424 Zeichen) nur Seite 1–6 = 26 %. Kapitel 7.2 Kundendokumentation (S. 15) und die angehängte Kundendoku (S. 32 ff.) wurden **nie gelesen** — erklärt Norberts Beobachtung (nur erwähnt → mal hohe, ausführlich → mal niedrige Punkte: Bewertung beruhte auf Inhaltsverzeichnis-Fragmenten) und relativiert die Varianz-Baseline 66/72/67. **Fix (4 Punkte):** (1) Kürzung 15K → Schutzgrenze 150K Zeichen (`extractor_max_input_chars`), (2) `volltext` wird im Code gesetzt statt vom Modell geechot (war durch max_tokens=8192 ohnehin verlustbehaftet), (3) Evaluator-Budgets 4K/3K → 12K/8K (`evaluator_section_max_chars`/`evaluator_context_max_chars`), (4) Bildseiten-Erkennung: Seiten <200 Zeichen (Screenshots im Anhang, beim Beispiel 11 von 42) werden gezählt und als Hinweis an die Evaluatoren gegeben („nicht lesbar ≠ fehlend"). Mehrkosten: wenige Cent pro Bewertung. Neuer Test mit Beispiel-PDF als Fixture. Tests 28/28 grün. Commit-Vorschlag: `fix: Dokumente vollständig einlesen statt nur erste 15.000 Zeichen`.

**2026-06-12: Stufen-Begriffe operationalisiert** (`systemprompt.md`: STUFEN-DEFINITIONEN + ENTSCHEIDUNGSPROTOKOLL A/B/C — Kerntest „wörtlich im Text vs. nur ableitbar" für die Grenze erkennbar/erschließbar). Zweitprüfer-Agent als Konzept in SKILL.md dokumentiert (blinde Zweitbewertung, Dissens-Anzeige statt Mittelung), Umsetzung bewusst zurückgestellt bis die Operationalisierung gemessen ist. **Nach Deploy: dieselbe Test-PDF (Keicher!) 3× bewerten, Streuung gegen Baseline 66/72/67 vergleichen.** Tests 26/26 grün.

**2026-06-11: Bewertungsskill angelegt (lokal, NICHT committet/deployt).** Anlass: Konsistenz-Test ergab 66/72/67 Punkte für dieselbe Dokumentation (**Keicher**) — Streuung kreuzt die Notengrenze 66/67. Analyse: Punktanker + temperature=0 waren schon aktiv; Varianz kommt von der **Stufen-Entscheidung** („erkennbar" 74 vs. „erschließbar" 58 kippt; bei Durchführung mit 30 % = 4,8 Punkte im Gesamt). Umsetzung: Rubriken + Systemprompt aus `category_evaluator_agent.py` nach **`backend/skills/bewertung/`** externalisiert (SKILL.md + systemprompt.md + kategorien/*.md), neuer `rubric_loader.py` lädt sie zur Laufzeit mit strenger Validierung. **Äquivalenz per Diff gegen Git-HEAD bewiesen: Prompts byte-identisch** → Umstellung verhaltensneutral. 8 neue Tests (u.a. Gewichtungs-Sync Skill ↔ Domain `weight_percent()`). Tests gesamt 26/26 grün. Arbeitsstand + Maßnahmen-Ideen fürs Konsistenz-Problem stehen **in der SKILL.md selbst** (gemeinsames Arbeitsdokument). Commit-Vorschlag: `refactor: Bewertungsrubriken als editierbaren Skill externalisieren`.

**2026-06-11 erledigt:**
- `session.py`: neue Factory `create_engine_and_sessionmaker()` — Engine-Konfiguration an einer Stelle; modulglobale Engine (für FastAPI, persistenter uvicorn-Loop) wird daraus erzeugt.
- `evaluation_tasks.py`: `_run()` **und** `_mark_failed()` erzeugen jetzt eine task-lokale Engine im eigenen Loop, mit `await engine.dispose()` im `finally`. Damit werden auch fehlgeschlagene Jobs zuverlässig auf `failed` gesetzt (vorher blieben sie auf `pending`, weil `_mark_failed` an derselben kaputten Engine scheiterte).
- Tests: 18/18 grün. Von Norbert committet und per `sync.sh` deployt (~21:07 Uhr).
- **Verifikation mit perfektem Vorher/Nachher-Beleg im worker.log:**
  - *Vorher* (19:04, alter Code): Folge-Task auf ForkPoolWorker-1 crashte 8 ms nach Start mit „attached to a different loop", lief erst im Retry.
  - *Nachher* (21:19, neuer Code): Dritter Job landete als Folge-Task auf ForkPoolWorker-1 — exakt dieselbe Konstellation — und lief fehlerfrei durch (21:19:52 → 21:22:21, kein Loop-Fehler, kein Retry). Davor auch zwei parallele Jobs auf Worker-1/2 sauber abgeschlossen.

**Offene Korrektur in dieser Notiz:** Server-Zugriff läuft per **Passwort-Auth** (nicht SSH-Key) — deshalb können weder Claude-Tool-Umgebung noch das `!`-Präfix in Claude Code auf den Server; Befehle muss Norbert im eigenen Terminal ausführen. Optional: `ssh-copy-id root@192.168.2.83` würde das beheben.

**Stand 2026-06-07 (Vorarbeit):**
- Hängenden Job „Weidlich Luca" diagnostiziert + aufgeräumt (verwaisten `pending`-Duplikat per DELETE-Endpoint gelöscht; gültige Bewertung lag bereits vor).
- Ursache **per Log eindeutig belegt**: Eventloop-Bug, **nicht** OOM (OOM-These verworfen).
- SpaCy-Worker-Singleton umgesetzt, getestet und deployt (`5b1ebc0`) — Effizienz-Verbesserung, nicht der Bugfix.

# Link

[IHK-Bewertung](https://ihkmatrix.softexceptions.com)
## Nächste Schritte

- [ ] Architektur mit SOLID-Prinzipien überprüfen
- [ ] Mit KI-Agenten erweitern?
- [ ] Zusammenarbeit mit IHK klären
- [ ] Feedback von Lehrenden einholen
- [x] **Echter Bugfix (Variante B) — ERLEDIGT 2026-06-11:** task-lokale Engine in `_run()` + `_mark_failed()` mit `dispose()` im `finally`. Deployt und per worker.log verifiziert (Folge-Task im selben Kind läuft jetzt fehlerfrei; vor dem Deploy crashte dasselbe Muster um 19:04 noch).
- [x] **Härtung Punkt 2 — lokal umgesetzt 2026-06-11 (Commit + Deploy offen):** `task_soft_time_limit=900s` / `task_time_limit=960s` (konfigurierbar via .env, `config.py`), Anthropic-Client mit `timeout=120s` + `max_retries=2` (SDK-Default-Timeout wäre 600s). `SoftTimeLimitExceeded` wird im Task explizit behandelt: kein Retry, Job direkt auf `failed`. `_mark_failed` zu modulweitem Helper `_mark_job_failed()` refaktoriert. **Restlücke:** Greift nur das Hard-Limit (Kind in C-Code blockiert, SIGKILL), läuft `_mark_job_failed` nicht → Job bleibt in aktivem Status. Das deckt erst der Watchdog (Härtung 3) ab.
- [ ] **Härtung Punkt 3:** Watchdog (Celery-Beat), der Jobs in aktivem Status mit altem `updated_at` automatisch auf `failed` setzt → Selbstheilung statt manuellem Aufräumen
- [ ] Kosmetik: doppelte Log-Zeile beim Worker-Start (doppelt registrierter Logging-Handler, Celery-Logger + Root-Propagation) glätten

## Notizen

Potenzial für Erweiterung durch MCP-Integrationen oder Agent-basierte Analysen.

---

## Betrieb & Infrastruktur

- **Deployment:** nativ auf Proxmox-LXC (kein Docker), systemd + lokales PostgreSQL + Redis
- **Server:** `root@192.168.2.83` (LXC „IHKBewertung"), ~3,4 GB RAM (Erhöhung auf 8 GB optional — war als Reaktion auf die *verworfene* OOM-These gedacht; nur Reserve, **nicht** der Fix)
- **Pfade:** Code `/opt/ihk-bewertungssystem/backend`, Uploads `/opt/ihk-bewertungssystem/uploads`, Worker-Log `/var/log/ihk-bewertungssystem/worker.log`
- **Dienste:** `ihk-api` (uvicorn, `--workers 2`, Port 8000), `ihk-worker` (Celery, `--concurrency=2 --queues=evaluation`), nginx (Port 9080), PostgreSQL, Redis
- **Deploy:** `bash sync.sh` vom lokalen Arbeitsrechner (rsync + pip + npm build + Dienst-Restart). Server-Login per **Passwort** (kein Key) — **Claude-Tool-Umgebung und `!`-Präfix können nicht auf den Server** (kein TTY für den Passwort-Prompt); Server-Befehle führt Norbert im eigenen Terminal aus. Optional behebbar via `ssh-copy-id root@192.168.2.83`.
- **DB-Zugriff:** `runuser -u postgres -- psql -d ihk_evaluator -c "…"`

### Stolperfallen
- **DB-Encoding ist `SQL_ASCII`** (nicht UTF-8, Befund 2026-06-12): **Jede** serverseitige JSON-Verarbeitung (`::jsonb`, `->>`, auch `json_array_elements` & Co.) schlägt auf Dokumenten mit Umlaut-Escapes fehl („unsupported Unicode escape sequence") — die Funktionen de-escapen `ä` intern, was SQL_ASCII nicht darstellen kann. Einziger Workaround: JSON-Spalte als roher Text holen (`spalte::text`, keine Verarbeitung) und per Pipe in `python3 -c "… json.loads …"` clientseitig parsen. Langfristig sauber: DB nach UTF-8 migrieren (dump + restore in neu angelegte UTF-8-DB).
- **Worker-Logs gehen in die Datei** (`--logfile=…/worker.log`), **nicht** nach `journalctl`. Bei Worker-Log-Suche immer die Datei nehmen.
- **Zeitzone:** DB speichert UTC (`datetime.utcnow()`), Frontend zeigt Europe/Berlin (UTC+2). „14:25 im Frontend" = „12:25 in der DB".
- **Vor Deploy/Restart:** sicherstellen, dass **kein Job läuft** — `systemctl restart ihk-worker` killt laufende Bewertungen (→ landen sonst auf `pending`, siehe unten). Check: `SELECT count(*) FROM evaluation_jobs WHERE status NOT IN ('completed','failed');` muss `0` sein.
- **`SYSTEMD_PAGER=cat`** setzen, damit `systemctl`/`journalctl` keinen Pager öffnen (sonst „Konsole hängt" → mit `q` raus).

---

## Runbook: Job hängt „in der Warteschlange" (Status `pending`)

**Symptom:** Job bleibt im Frontend ewig in der Warteschlange. In der DB: `status = pending`, `error_message = NULL` — der Job hat nie den ersten Statuswechsel erreicht.

**Root Cause (Fall vom 2026-06-07, „Weidlich Luca") — per Log belegt:** asyncio-Eventloop-Bug, **kein** OOM. Fehler im worker.log: `got Future ... attached to a different loop`. Die SQLAlchemy-Async-Engine wird in `session.py` **modulglobal beim Import** erzeugt; jeder Celery-Task ruft aber `asyncio.run(_run())` → **neuer Eventloop pro Task**. Der asyncpg-Pool bindet sich an den ersten Loop; jeder Folge-Task im selben Worker-Kind läuft in einem neuen Loop → sofortiger Fehler (~1 s, kein Speicherdruck).

Warum `pending` statt `failed`: Nach 3 Fehlversuchen (initial + `max_retries=2`) läuft `_mark_failed()` — das nutzt **dieselbe kaputte Engine im wieder neuen Loop** und scheitert ebenfalls → `failed` wird nie geschrieben. Warum manche Jobs durchlaufen: Der **erste** Task pro frisch geforktem Kind bindet die Engine an seinen Loop und läuft; nur **Folge-Tasks** im selben Kind scheitern (intermittierend, lastabhängig).

> ⚠️ **Verworfene OOM-These:** Ursprünglich vermutet (knapper RAM × SpaCy-Modell pro Task). Durch das Log widerlegt. Lektion: Die Indizien (pending, `error_message` NULL, knapper RAM) passten *scheinbar* — erst das Worker-Log brachte die Wahrheit. **RAM-Erhöhung und der SpaCy-Singleton lösen diesen Bug NICHT.**

**Warum erst nach ~50 Auswertungen? (intermittierend, nicht deterministisch)**
Zustands-/timing-abhängiger Bug (Heisenbug), kein deterministischer — sonst hätte er bei Auswertung Nr. 1 zugeschlagen. Drei Bedingungen müssen **zusammen** zutreffen:
1. Dasselbe Worker-Kind bearbeitet **nacheinander** (NICHT gleichzeitig!) mehr als einen Task — Loop-1 aus Task A, später Loop-2 aus Task B im selben Prozess.
2. Eine DB-Connection aus Task A **überlebt im Pool** bis Task B (wird nicht idle recycelt).
3. Task B **verwendet diese gepoolte Connection wieder** statt eine frische zu öffnen → sie hängt an Loop-1, läuft in Loop-2 → 💥.

**Last ist Begünstiger, nicht Auslöser:** Gleichzeitigkeit ist nicht nötig. Entscheidend ist, dass Task B **dicht genug** auf Task A folgt, damit die Connection (Bedingung 2) überlebt. Bei den ~50 vorherigen (verteilte Last) wurde die Connection zwischen Tasks meist recycelt → frische Connection → kein Fehler. In der Prüfungslast kamen Auswertungen dicht hintereinander → Connection überlebte → Bug. Theoretisch reicht **eine** Person, die zwei Auswertungen kurz nacheinander startet und beide ins selbe Kind fallen. Beleg im Log: ForkPoolWorker-1 scheiterte sofort an Lucas Job → dieses Kind hatte **vorher schon** eine (längst fertige) Auswertung bearbeitet; Luca war der Folge-Task.

**Diagnose-Befehle:**
```bash
# 1. Echten Status + Zeitstempel + Job-ID
runuser -u postgres -- psql -d ihk_evaluator -c "SELECT id, status, error_message, updated_at FROM evaluation_jobs WHERE participant_name ILIKE '%NAME%' ORDER BY updated_at;"
# 2. ENTSCHEIDEND: Was hat der Worker mit dem Job gemacht? (Job-ID-Kurzform aus Schritt 1)
cd /var/log/ihk-bewertungssystem
grep <JOB_ID_KURZ> worker.log
#   → "attached to a different loop"      = der Eventloop-Bug (siehe Root Cause) — der bisher beobachtete Fall
#   → "Starte Bewertung" ohne Folgezeile  = Kind-Tod (dann Schritt 3 prüfen)
# 3. Nur falls KEIN Loop-Fehler: nach hartem Tod suchen
grep -iE "WorkerLost|signal 9|exited prematurely|MemoryError" worker.log
# 4. Liegt das Eingabe-PDF noch da? (übrig = Job kam nie sauber durch; sonst löscht finally es)
ls -la /opt/ihk-bewertungssystem/uploads/
```

**Reparatur:**
- Existiert bereits eine **erfolgreiche Bewertung desselben Prüflings** (Neu-Upload): verwaisten Job löschen — `curl -i -X DELETE http://127.0.0.1:8000/api/evaluations/<JOB_ID>` (entfernt Job + Eingabe-PDF, Status 204).
- Sonst, wenn PDF noch da + Status sauber `pending`: Task neu einreihen (`run_evaluation_task.apply_async(args=[job_id], queue="evaluation")` im venv mit `PYTHONPATH=src`).

**Echter Fix — Variante B, deployt + verifiziert 2026-06-11:** `_run()` und `_mark_failed()` (evaluation_tasks.py) erzeugen eine task-lokale Async-Engine via `create_engine_and_sessionmaker()` (neue Factory in `session.py`) und schließen sie im `finally` mit `await engine.dispose()`. Dieses Runbook ist damit nur noch für Altfälle/andere Crash-Typen relevant.

**Bereits deployt (2026-06-07), aber NICHT der Fix für diesen Bug:** SpaCy/Presidio als prozessweiter Worker-Singleton (`get_shared_presidio_engines()` in `pii_anonymizer.py` + `worker_process_init`-Hook in `celery_app.py`). Legitime Effizienz-/RAM-Verbesserung (Modell einmal pro Worker-Kind statt pro Task), lässt den Eventloop-Bug aber unberührt. Bleibt drin.

> Ergänzend sinnvoll (Härtung 2+3): `task_time_limit` gegen echte Hänger + Watchdog, der aktive Jobs mit altem `updated_at` auf `failed` setzt → Selbstheilung, falls je ein anderer Crash-Typ auftritt.
