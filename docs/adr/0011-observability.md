# ADR-0011: Observability – Langfuse + OpenTelemetry

- **Status:** accepted
- **Datum:** 2026-10-24
- **Abhängig von:** ADR-0001 (OnPrem), ADR-0002 (Gateway)
- **Adressiert Anforderung:** Nachvollziehbarkeit (Modell, Umgebung, Daten)

## Kontext

Zwei Observability-Ebenen: (a) **LLM-spezifisch** – Prompts, Retrievals, Antworten, Token-Kosten, Latenz je Trace; (b) **Infrastruktur** – Latenz, Fehler, GPU-Auslastung, Queue-Längen. Beide brauchen wir für Debugging, Audit und Nachvollziehbarkeit.

## Entscheidung

**Kombination aus beiden:**

- **Langfuse (self-hosted)** für LLM-Traces: jeder Request als Trace mit Spans (Retrieval, Tool-Calls, Generation), inkl. Modellversion, Prompt-Version, Index-Version → beantwortet "welches Modell + welche Daten haben diese Antwort erzeugt"
- **OpenTelemetry + Grafana/Prometheus** für Infrastruktur-Metriken und verteiltes Tracing über Gateway ↔ Backend ↔ Qdrant ↔ MCP
- Trace-Kontext (trace_id) wird durchgegeben → Sprung von Infra-Span zu LLM-Trace möglich

## Betrachtete Alternativen

| Alternative | Warum verworfen |
|---|---|
| Nur OTel/Grafana | Generisch, aber keine LLM-Semantik (Prompt/Completion-Paare, Kosten, Eval-Scores) |
| LangSmith | SaaS, Prompts verlassen das Haus → widerspricht ADR-0001 (self-hosted Alternative existiert: Langfuse) |
| Minimal: strukturierte Logs + Audit | Reicht nicht für Prompt-Debug + Eval-Verknüpfung; Langfuse self-hosted kostet nur eine zusätzliche K8s-Workload |

## Konsequenzen

**Positiv**
- Vollständige Nachvollziehbarkeit: Trace → Prompt + Modellversion + geretrieve Chunks + Tool-Calls
- Eval-Scores (ADR-0012) landen direkt am Trace → Qualitätskorrelation sichtbar

**Datenschutz-Pflichten (wichtig!)**
- LLM-Traces enthalten Prompts und ggf. personenbezogene Daten → Langfuse-Zugriff rollenbasiert einschränken, Retention definieren, User-IDs pseudonymisieren
- Tracing selbst ist ein Verarbeitungsschritt → in die DSVGO-/Compliance-Doku aufnehmen
