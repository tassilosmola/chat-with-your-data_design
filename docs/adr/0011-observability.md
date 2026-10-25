# ADR-0011: Observability – Langfuse + bestehendes Monitoring

- **Status:** proposed
- **Datum:** 2026-10-25
- **Abhängig von:** ADR-0001 (OnPrem), ADR-0002 (Gateway)
- **Adressiert Anforderung:** Nachvollziehbarkeit (Modell, Umgebung, Daten)

## Kontext

Zwei Observability-Ebenen: (a) **LLM-spezifisch** – Prompts, Retrievals, Antworten, Token, Latenz je Trace; (b) **Infrastruktur** – Latenz, Fehler, GPU-Auslastung, Queue-Längen. Beide brauchen wir für Debugging, Audit und Nachvollziehbarkeit.

## Entscheidung

- **Langfuse (self-hosted)** für LLM-Traces: jeder Request als Trace mit Spans (Query-Rewrite, Retrieval, Generation), inkl. **Modell-, Prompt- und Index-Version** (ADR-0013) → beantwortet "welches Modell + welche Daten haben diese Antwort erzeugt". Langfuse übernimmt zugleich **Prompt-Versionierung, Datasets und Eval-Scores** (ADR-0012) – kein separates Experiment-Tracking nötig.
- **Bestehendes Plattform-Monitoring (Prometheus/Grafana, OTel)** für Infrastruktur-Metriken – wird **wiederverwendet**, nicht neu aufgebaut. vLLM- und Gateway-Metriken (u.a. `vllm:num_requests_waiting`) werden dort gescrapt und speisen auch KEDA (ADR-0014).
- Trace-Kontext (`trace_id`) wird durchgereicht → Sprung von Infra-Span zu LLM-Trace möglich

## Betrachtete Alternativen

| Alternative | Warum verworfen |
|---|---|
| Nur OTel/Grafana | Keine LLM-Semantik (Prompt/Completion-Paare, Retrieval-Kontext, Eval-Scores) |
| LangSmith | SaaS, Prompts verlassen das Haus → widerspricht ADR-0001 |
| Strukturierte Logs + Audit | Reicht nicht für Prompt-Debug + Eval-Verknüpfung |

## Konsequenzen

**Aufwand (ehrlich bewertet)**
- Langfuse self-hosted (v3) ist **kein Einzel-Container**: benötigt Postgres, ClickHouse, Redis und S3-kompatiblen Object-Storage. Postgres/S3 können mit ADR-0005/-0006 geteilt werden; ClickHouse ist neu. Der Aufwand lohnt sich, weil Langfuse Tracing, Prompt-Management und Eval in einem Werkzeug abdeckt.

**Datenschutz & Mitbestimmung (Pflicht vor Rollout)**
- LLM-Traces und Chat-Verläufe enthalten Prompts mit potenziell personenbezogenen Daten → rollenbasierter Zugriff auf Langfuse, definierte **Retention** (z.B. 30 Tage Volltext, danach nur Metriken), User-IDs pseudonymisiert
- **Datenschutz-Folgenabschätzung (DSFA)** nach Art. 35 DSGVO; Tracing als Verarbeitungstätigkeit dokumentieren
- **Betriebsrat:** Das Logging von Nutzeranfragen ist eine technische Einrichtung, die zur Leistungs-/Verhaltenskontrolle geeignet ist (§ 87 Abs. 1 Nr. 6 BetrVG) → **Betriebsvereinbarung** vor dem Pilot. Früh einbinden – typischer Blocker für den Zeitplan.
