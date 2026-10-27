# Roadmap – „Chat with your Data"

Grundsatz: v1 ist bewusst minimal (ADR-0015). Jede Erweiterung hat einen **messbaren Auslöser** – gebaut wird, wenn Daten oder Use Cases es verlangen, nicht vorsorglich. Alle Erweiterungen sind **additiv**; die v1-Architektur hält die Anschlusspunkte offen.

## v1 – HR-Self-Service (PoC → Pilot)

| Baustein | ADR |
|---|---|
| Web-UI, Chat-API (feste RAG-Pipeline), JWT/Entra ID | 0004, 0005 |
| LLM-Gateway (Quotas, Backpressure), vLLM, KEDA auf Queue-Länge | 0002, 0003, 0014 |
| bge-m3 Embeddings, Qdrant Hybrid + ACL-Filter | 0006, 0007, 0009 |
| Citations, "keine Quelle"-Schwelle, Citation-Check | 0010 |
| Postgres (Verlauf, Feedback, Manifest), Langfuse (Traces, Evals) | 0005, 0011, 0012, 0013 |
| PII-Sanitizing in der Ingestion (Presidio, Allowlist, Residual-Check, HR-Freigabe) + deterministische Injection-Checks | 0016 |

## v2 – Strukturierte Daten & mehr Quellen

| Erweiterung | Auslöser | Anschlusspunkt in v1 | ADR |
|---|---|---|---|
| **MCP-Tools für strukturierte Daten** (z.B. Resturlaub, Reisekosten-Pauschalen, offene Stellen) | Erster priorisierter Use Case mit strukturierten Daten + Fachbereich als Tool-Owner | Chat-API ruft Gateway mit Function-Calling auf; Retrieval wird ein Tool unter mehreren; Citations unterstützen `tool:<name>` | 0008 |
| OAuth On-Behalf-Of zu Fachsystemen | Mit erstem MCP-Tool auf personenbezogene Daten | User-Token liegt in der Chat-API bereits vor | 0008 |
| Cross-Encoder-Reranking | Context-Recall im Eval unter Ziel | zusätzliche Stufe zwischen Retrieval und Generation | 0010 |
| Konnektoren SharePoint/Confluence | Zweiter Use Case (z.B. IT-Helpdesk) | Konnektor-Schnittstelle der Ingestion; ACL-Mapping auf Entra-Group-IDs | 0004, 0007 |
| MS Teams als Kanal | Pilot erfolgreich, Rollout an alle MA | API-first – Teams-Bot ist nur ein weiterer Client | 0005 |
| LLM-basierter PII-Detektor als zweite Stufe | PERSON-Recall im PII-Testset unter Ziel oder unstrukturierte Quellen (SharePoint/Confluence) | zusätzlicher Detektor im Residual-Check, via Gateway | 0016 |
| PII-Output-Guard im Serving | Quellen ohne HR-Freigabeprozess | zusätzliche Stufe zwischen Generation und Citation-Assembler | 0016 |
| Prompt-Injection-Klassifikator (z.B. Prompt Guard) | Nicht kuratierte Quellen oder erste MCP-Tools | zusätzlicher Detektor im Ingestion-Gate (Status `needs_review`) | 0010, 0016 |

## v3+ – Nach Bedarf

| Erweiterung | Auslöser | ADR |
|---|---|---|
| Groundedness-Check (zweiter LLM-Call) | Kritischer Use Case (z.B. Rechts-/Compliance-Auskünfte) | 0010 |
| Semantischer Layer (Cube/dbt) unter den MCP-Tools | Viele Kennzahlen-Tools mit gemeinsamer Metrik-Definition | 0008 |
| MLflow / DVC | Eigenes Fine-Tuning (LLM/Embedding/Reranker) oder Pflicht zur bitgenauen Corpus-Reproduktion | 0013 |
| Cluster-Autoscaler für GPU-Nodes | Private Cloud mit dynamischer GPU-Provisionierung | 0014 |
| Cloud-LLM-Route für unkritische Workloads | Qualitäts-/Kapazitätslücke onPrem + Freigabe Datenschutz | 0001, 0002 |
