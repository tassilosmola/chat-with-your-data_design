# ADR-0012: Evaluationsstrategie – Golden Dataset + CI-Evals + Feedback-Loop

- **Status:** accepted
- **Datum:** 2026-10-24
- **Zusammenhang:** ADR-0010 (Citations → Groundedness-Messung), ADR-0011 (Scores an Traces)

## Kontext

Zwei Qualitätsdimensionen müssen kontinuierlich gesichert werden: **Antwortqualität** (halluziniert das Modell? findet Retrieval das Richtige?) und **Datenqualität** (veraltete/korrupte Dokumente im Index). Beides ändert sich bei jedem Modell-, Prompt- oder Index-Update → Regression muss automatisch sichtbar sein ("RAG-CI").

## Entscheidung

**Zweigleisig:**

1. **Golden Dataset + CI-Evals (Prävention)**
   - Kuratiertes Set von ~50–100 Fragen mit Referenzantworten und erwarteten Quellen, aus echten Fachfragen
   - Automatisierte Evals (z.B. Ragas/DeepEval) in der Pipeline: Metriken = Context-Recall, Faithfulness/Groundedness, Answer-Relevance, ACL-Filter-Korrektheit
   - Eval-Gate vor jedem Release/Index-Rebuild: Scores dürfen nicht unter Baseline fallen

2. **User-Feedback-Loop (Real-World)**
   - Thumbs up/down + optionaler Kommentar über `POST /feedback` (ADR-0005)
   - Negative Feedbacks landen als Kandidaten im Golden Dataset → Dataset wächst mit Realnutzung

## Betrachtete Alternativen

| Alternative | Warum verworfen |
|---|---|
| Nur Feedback-Loop | Reaktiv – Regressionen fallen erst bei Nutzern auf, nicht vor dem Release |
| Nur CI-Evals | Golden Dataset deckt nicht ab, was Nutzer wirklich fragen; ohne Feedback bleibt es statisch |
| Manuelle Stichproben durch Fachbereich | Skaliert nicht auf 10k MA; gut als Ergänzung zum Aufbau des Golden Datasets, nicht als einzige Maßnahme |

## Konsequenzen

- Aufbau des Golden Datasets ist **Arbeitspaket mit Fachbereichen** (nicht rein technisch) → früh einplanen
- CI-Evals brauchen GPU-Zeit (Eval-LLM-Calls) → in Pipeline-Budget einrechnen; Eval-LLM kann kleineres Modell sein
- Datenqualität: Ingestion-Pipeline bekommt eigene Checks (Parse-Erfolg, Chunk-Verteilung, Duplikate) – Qualitätsprobleme werden dort gefangen, nicht erst in der Antwort
