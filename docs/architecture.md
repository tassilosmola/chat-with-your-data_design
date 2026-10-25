# Architektur – „Chat with your Data"

v1 ist ein HR-Self-Service-Bot (ADR-0015) für ~10k Mitarbeitende, OnPrem auf der bestehenden Kubernetes-Plattform (ADR-0001). Er besteht aus **8 Bausteinen**. Dokumente laufen über RAG mit ACL-Filter und Citations-Pflicht. Strukturierte Daten über MCP-Tools sind als v2 vorbereitet ([Roadmap](roadmap.md)).


## Request-Flow (eine Chat-Anfrage)

1. **Auth:** Nutzer → Ingress (TLS) → Chat-API validiert das Entra-Token; Gruppen-IDs aus dem Token bzw. per Graph bei Overage
2. **Query-Rewriting:** Folgefrage + Historie → eigenständige Suchanfrage
3. **Retrieval:** Hybrid-Query in Qdrant mit **serverseitigem ACL-Filter** → nur berechtigte Chunks. Unter Mindest-Score: feste Antwort „keine Quelle gefunden“, ohne LLM-Call
4. **Generation:** Kontext + System-Prompt ("nur aus Kontext, jede Aussage zitieren") → Gateway (Quota-Prüfung) → vLLM
5. **Antwort:** Text + **Citations** (Dokument, Seite). Citation-Check stellt sicher, dass nur Chunks aus dem Kontext zitiert werden
6. **Nachvollziehbarkeit:** Trace in Langfuse mit Modell-, Prompt- und Index-Version; Verlauf in Postgres
7. **Qualität:** Feedback → Golden Dataset; deterministische Tests + Ragas-Evals als Release-Gate

## Komponenten-Übersicht

| # | Komponente | Technologie | ADR |
|---|---|---|---|
| 1 | Referenz-UI | einfache Web-UI (austauschbar) | 0005 |
| 2 | Chat-API | FastAPI, OpenAI-SDK, qdrant-client – kein Agent-Framework | 0004, 0005 |
| 3 | LLM-Gateway | LiteLLM Proxy | 0002 |
| 4 | LLM-Serving + Skalierung | vLLM, ~70B Q4 (Sizing offen); KEDA auf Queue-Länge | 0003, 0014 |
| 5 | Embeddings | bge-m3, self-hosted | 0009 |
| 6 | Vektor-DB | Qdrant (Hybrid, ACL-Payload-Filter, Collection-Alias) | 0006, 0007, 0013 |
| 7 | Persistenz | Postgres (Verläufe, Feedback, Manifest) | 0005, 0013 |
| 8 | LLM-Observability & Eval | Langfuse (Traces, Prompts, Datasets), Ragas in CI | 0011, 0012 |
| – | Plattform (wiederverwendet) | Entra ID, Ingress, Prometheus/Grafana, CI/CD | 0001, 0011 |
| v2 | Strukturierte Daten | MCP-Server je Fachdomäne, On-Behalf-Of | 0008 |

## Zentrale Risiken & Mitigation

| Risiko | Mitigation | ADR |
|---|---|---|
| Unberechtigter Zugriff über RAG | ACL-Filter vor dem LLM, Gruppen-IDs statt Sync, deterministische ACL-Tests in CI | 0007, 0012 |
| Personenbezogene Daten in Traces/Verläufen | Kein Response-Cache, Retention, Pseudonymisierung, rollenbasierter Langfuse-Zugriff | 0002, 0011 |
| Betriebsrat/DSFA blockiert Rollout | Früh einbinden, Betriebsvereinbarung parallel zum PoC | 0011, 0015 |
| Halluzination trotz RAG | Citations-Pflicht, „keine Quelle“-Schwelle, Citation-Check, Eval-Gate; nächster Schritt Reranking | 0010, 0012 |
| Tabellen (Pauschalen, Entgeltgruppen) falsch geparst | Strukturerhaltendes Parsing, Ingestion-Checks, Golden-Dataset-Fragen zu Tabellenwerten | 0004, 0012 |
| Indirekte Prompt-Injection über Dokumente | Kontext als Daten markiert, keine Tools/Aktionen in v1 | 0010 |
| Modellqualität onPrem < Cloud-LLM | ~70B-Klasse, Eval misst objektiv, Gateway erlaubt Modellwechsel | 0002, 0003 |
| GPU-Kapazität bei Lastspitzen | KEDA auf Queue-Länge im festen Pool, Warm-Pool, Backpressure (429); Lasttest vor Rollout | 0003, 0014 |
| „Welche Daten/welches Modell?“ nicht beantwortbar | Versionsfelder als Pflicht in jedem Trace; Collection-Alias für Rollback | 0013 |

## Grobe Phasenplanung (Orientierung)

| Phase | Inhalt | Grob-Schätzung |
|---|---|---|
| 0 – Setup | GPU-Nodepool, vLLM, Gateway, Qdrant, Langfuse auf bestehender Plattform; Kick-off mit Datenschutz + Betriebsrat | 1–2 Wochen |
| 1 – PoC | Ingestion der HR-PDFs (inkl. Tabellen), RAG-Pipeline, Chat-API, Referenz-UI, Citations, erste 30 Golden-Fragen mit HR | 3–4 Wochen |
| 2 – Härtung | ACL-Filter + Tests, Group-Overage, Eval-Gate in CI, KEDA + Backpressure, Retention | 2–3 Wochen |
| 3 – Pilot | HR-Pilot (~100–300 MA), Feedback-Loop, Lasttest, Schwellwerte kalibrieren | 4–6 Wochen |
| 4 – Rollout | Stufenweiser Ausbau auf 10k MA, Kanalentscheidung (Teams/Web), danach v2 nach [Roadmap](roadmap.md) | fortlaufend |

*Annahme: 2–3 Personen, bestehende K8s-Plattform, HR liefert Golden-Dataset-Fragen, GPU-Hardware verfügbar.*
