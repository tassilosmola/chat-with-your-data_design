# ADR-0012: Evaluationsstrategie – Golden Dataset + CI-Evals + Feedback-Loop

- **Status:** proposed
- **Datum:** 2026-10-25
- **Zusammenhang:** ADR-0010 (Citations → Groundedness-Messung), ADR-0011 (Langfuse Datasets/Scores), ADR-0016 (PII-Verifikation)

## Kontext

Zwei Qualitätsdimensionen müssen kontinuierlich gesichert werden: **Antwortqualität** (halluziniert das Modell? findet Retrieval das Richtige?) und **Datenqualität** (veraltete/korrupte Dokumente im Index). Beides ändert sich bei jedem Modell-, Prompt- oder Index-Update → Regression muss automatisch sichtbar sein ("RAG-CI").

## Entscheidung

**Dreigleisig – deterministisch, LLM-basiert, real-world:**

1. **Deterministische Tests (CI, jeder Build)**
   - **ACL-Korrektheit:** Testnutzer mit/ohne Gruppen → definierte Chunks dürfen nie bzw. müssen zurückkommen. Sicherheitseigenschaft = harter Test, keine LLM-Metrik.
   - Citation-Check (ADR-0010), "Keine Quelle"-Verhalten bei Fragen außerhalb des Corpus
   - Ingestion-Checks: Parse-Erfolg, Tabellen erkannt, Chunk-Längenverteilung, Duplikate
   - **PII-Sanitizing (ADR-0016):** Recall-Gate auf annotiertem PII-Testset (100 % für strukturierte IDs), Allowlist-Tests für Funktionsträger, Canary-Tokens dürfen nach Ingestion in keinem Qdrant-Payload auftauchen; synthetische Dokumente mit versteckten/sichtbaren Prompt-Injections müssen erkannt werden, die PoC-PDFs dürfen keinen Fehlalarm erzeugen
   - **Adversariale PII-Fragen** im Golden Dataset, deterministisch per Regex gegen bekannte PII-Werte geprüft (kein LLM-Judge)

2. **Golden Dataset + LLM-Evals (vor Release/Index-Rebuild)**
   - Kuratiertes Set von ~50–100 Fragen mit Referenzantworten und erwarteten Quellen, gemeinsam mit HR erstellt (ADR-0015)
   - Metriken via Ragas: Context-Recall, Faithfulness/Groundedness, Answer-Relevance
   - Dataset und Scores liegen in **Langfuse Datasets** (ADR-0011); Ergebnis ist an Modell-/Prompt-/Index-Version geknüpft (ADR-0013)
   - Eval-Gate: Scores dürfen nicht unter Baseline fallen

3. **User-Feedback-Loop (Real-World)**
   - Thumbs up/down + optionaler Kommentar über `POST /feedback` (ADR-0005)
   - Negative Feedbacks werden gesichtet und landen als Kandidaten im Golden Dataset

## Betrachtete Alternativen

| Alternative | Warum verworfen |
|---|---|
| Nur Feedback-Loop | Reaktiv – Regressionen fallen erst bei Nutzern auf |
| Nur LLM-Evals | Sicherheitseigenschaften (ACL) dürfen nicht von einem probabilistischen Judge abhängen |
| Manuelle Stichproben durch Fachbereich | Skaliert nicht; gut zum Aufbau des Golden Datasets |
| MLflow für Eval-Tracking | Zweites Werkzeug neben Langfuse ohne Mehrwert (ADR-0013) |

## Konsequenzen

- Aufbau des Golden Datasets ist **Arbeitspaket mit dem Fachbereich** (nicht rein technisch) → früh einplanen
- LLM-as-Judge braucht GPU-Zeit; Judge-Modell sollte **nicht schwächer** als das Antwortmodell sein (sonst unzuverlässige Scores) → Eval-Läufe außerhalb der Geschäftszeiten auf dem Produktionsmodell
- Qualitätsprobleme der Daten werden in der Ingestion gefangen, nicht erst in der Antwort
