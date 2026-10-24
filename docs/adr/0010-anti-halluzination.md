# ADR-0010: Halluzinationsminderung – Citations als Pflicht

- **Status:** accepted
- **Datum:** 2026-10-24
- **Zusammenhang:** ADR-0012 (Eval misst Groundedness)

## Kontext

Halluzinationen sind die zentrale Vertrauens-Herausforderung. Es gibt mehrere komplementäre Maßnahmen mit unterschiedlichen Kosten (Latenz, Komplexität, zweiter LLM-Call).

## Entscheidung

**Pflicht (v1):**
- **RAG über die gesamte Pipeline** – Antworten nur aus retrieved Kontext + Tool-Ergebnissen
- **Citations/Quellenangaben in jeder Antwort:** Dokument + Chunk/Seite bzw. Tool-Name. System-Prompt instruiert: "Antworte nur aus dem Kontext; wenn nichts gefunden → sag es." Damit ist Halluzination für den Nutzer **überprüfbar**, auch wenn sie nicht zu 100 % verhindert wird.

**Optional (Roadmap, im ADR dokumentiert statt jetzt gebaut):**
- Cross-Encoder-Reranking (z.B. bge-reranker) → bessere Top-k-Qualität, ~moderater Latenzzuschlag
- Groundedness-Check (zweiter LLM-Call validiert Antwort gegen Kontext) → hohe Kosten/Latenz, nur für kritische Use-Cases
- Confidence-Threshold/Fallback "keine Quelle gefunden" → kann später via Langfuse-Metriken kalibriert werden

## Betrachtete Alternativen

- Alle Maßnahmen sofort → Latenz- und Komplexitätskosten unverhältnismäßig für PoC; Citations liefern den größten Vertrauens-Effekt pro Aufwand

## Konsequenzen

- API-Vertrag (ADR-0005) enthält `citations: [{doc_id, title, page, chunk}]` in jeder Response
- Retrieval-Metadaten (doc_id, acl, sensitivity) müssen durch die Pipeline bis zur Response getragen werden
- Eval-Metriken (ADR-0012) messen Context-Recall + Groundedness → entscheidet später datenbasiert, ob Reranking/Groundedness-Check nötig wird
