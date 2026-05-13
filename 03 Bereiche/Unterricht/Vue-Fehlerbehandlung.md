---
tags: [bereich, unterricht, vue, fehlerbehandlung, async]
---

# Fehlerbehandlung in Vue Composables

Unterrichtsbeispiel: Warum jeder `async`-API-Call ein `try/catch/finally` braucht.

## Das Problem: Stilles Versagen

> [!warning] Silent Failure
> Wenn ein `await`-Aufruf fehlschlägt und kein `catch`-Block vorhanden ist, passiert **nichts** — kein Fehler, kein Feedback, der UI bleibt einfach hängen.

**Reales Beispiel aus dem Unterricht:** Bei der HEMS-Gesamtkonferenz (~40 Personen gleichzeitig) hat der "Auswerten"-Button im Quiz nicht reagiert — weil `submit()` keinen `catch`-Block hatte. Kein User wusste, ob er nochmal klicken soll.

## ❌ Vorher — Stilles Versagen

```ts
async function submit() {
  result.value     = await quizService.evaluate(answers.value)
  showResult.value = true
  // Netzwerkfehler? → nichts passiert, Button hängt für immer
}
```

## ✅ Nachher — Drei-Zonen-Pattern

```ts
async function submit() {

  // Zone 1: VOR dem Call — UI sperren
  isSubmitting.value = true
  submitError.value  = null

  try {
    // Zone 2: Der riskante Bereich
    result.value     = await quizService.evaluate(answers.value)
    showResult.value = true

  } catch {
    // Zone 3a: Plan B — User informieren
    submitError.value = 'Auswertung fehlgeschlagen – bitte versuche es nochmal.'

  } finally {
    // Zone 3b: Aufräumen — läuft IMMER, egal ob Erfolg oder Fehler
    isSubmitting.value = false
  }
}
```

## Die drei Zustands-Variablen

| Variable | Typ | Aufgabe |
|---|---|---|
| `isSubmitting` | `boolean` | Sperrt Button → verhindert Doppelklick |
| `submitError` | `string \| null` | Zeigt dem User eine verständliche Fehlermeldung |
| `showResult` | `boolean` | Schaltet zur Ergebnis-Ansicht um |

> [!tip] Merksatz
> `try` = riskanter Bereich · `catch` = Plan B · `finally` = Aufräumen, läuft **immer**

## Faustregel für den Unterricht

**Immer wenn `await` verwendet wird und der User danach auf eine Reaktion wartet → `try/catch/finally` ist Pflicht.**

## Verbundene Notizen

- [[02 Projekte/SchulOrdnungHems|SchulOrdnungHems]] — Projekt, aus dem dieses Beispiel stammt
- [[Unterricht]] — Übergeordneter Unterrichtsbereich
