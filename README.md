# Architecture Decision Records – „Chat with your Data"

Alle ADRs liegen unter [`docs/adr/`](docs/adr/) und folgen einem vereinfachten MADR-Format: Kontext → Entscheidung → Alternativen → Konsequenzen.

- Architekturüberblick: [`docs/architecture.md`](docs/architecture.md)
- Roadmap (v1 → v2 → v3+): [`docs/roadmap.md`](docs/roadmap.md)
- Zoombares C4-Modell (LikeC4): [`c4_architecture/`](c4_architecture/README.md)

| ADR | Titel | Status | Kurzentscheidung |
|-----|-------|--------|------------------|
| [0001](docs/adr/0001-deployment-onprem-kubernetes.md) | Deployment-Modell | proposed | OnPrem auf bestehender Kubernetes-Plattform |
| [0002](docs/adr/0002-llm-gateway-abstraktion.md) | LLM-Gateway | proposed | LiteLLM für Quotas, Backpressure, Routing – kein Response-Cache |
| [0003](docs/adr/0003-llm-modell-sizing.md) | LLM-Modell & GPU-Sizing | open | ~70B Q4 / MoE auf 2 GPUs im PoC, Produktiv-Sizing per Lasttest |
| [0004](docs/adr/0004-datenquellen.md) | Datenquellen & Pipeline | proposed | v1: Dokumente, feste RAG-Pipeline ohne Agent; v2: strukturierte Daten |
| [0005](docs/adr/0005-zugriff-api-first.md) | Zugriffskanal | proposed | API-first, Auth in der API (inkl. Group-Overage), Postgres für Verläufe |
| [0006](docs/adr/0006-vektordatenbank-qdrant.md) | Vektordatenbank | proposed | Qdrant wegen nativem Hybrid-Search; pgvector als KISS-Alternative |
| [0007](docs/adr/0007-acl-metadata-filter.md) | Dokumentberechtigungen | proposed | ACL-Filter in Qdrant über Gruppen-IDs, kein Gruppen-Sync |
| [0008](docs/adr/0008-strukturierte-daten-mcp.md) | Strukturierte Daten | proposed (v2) | MCP-Tools je Fachdomäne, On-Behalf-Of |
| [0009](docs/adr/0009-embedding-strategie.md) | Embedding-Strategie | proposed | bge-m3 self-hosted (dense + sparse), via Gateway |
| [0010](docs/adr/0010-anti-halluzination.md) | Halluzinationsminderung | proposed | Citations + "keine Quelle"-Schwelle + Citation-Check; Reranking als nächster Schritt |
| [0011](docs/adr/0011-observability.md) | Observability & Compliance | proposed | Langfuse + bestehendes Monitoring; DSFA + Betriebsvereinbarung |
| [0012](docs/adr/0012-eval-strategie.md) | Evaluationsstrategie | proposed | Deterministische Tests + Golden Dataset (Ragas) + Feedback |
| [0013](docs/adr/0013-modell-datenversionierung.md) | Modell- & Datenversionierung | proposed | Manifest + Collection-Alias + Git-Pinning (statt DVC/MLflow) |
| [0014](docs/adr/0014-skalierung-keda-gpu.md) | Skalierung GPU-Workloads | proposed | KEDA auf Request-Queue im festen GPU-Pool + Backpressure |
| [0015](docs/adr/0015-use-case-scope-v1.md) | Use Case & Scope v1 | proposed | HR-Self-Service, 8 Bausteine, Rest = Roadmap mit Auslöser |
| [0016](docs/adr/0016-pii-sanitizing-ingestion.md) | PII-Sanitizing in der Ingestion | proposed | Presidio-Redaction vor Chunking, Funktionsträger-Allowlist, HR-Freigabe, Lineage `source_hash → sanitized_hash`; deterministische Injection-Checks im selben Gate |

## Entscheidungslogik (Zusammenhang)

```
Scope (0015): HR-Self-Service, minimal
OnPrem (0001)
  └─> Self-hosted LLM + Embeddings (0003, 0009)
        └─> Gateway: Quotas + Backpressure (0002)
              └─> KEDA auf Queue-Länge (0014)
Datenquellen (0004)
  ├─> v1: PDFs → PII-Sanitizing (0016) → Qdrant (0006) mit ACL-Filter (0007)
  └─> v2: Strukturierte Daten → MCP-Tools (0008)
Qualität
  ├─> Citations + "keine Quelle" (0010)
  └─> Eval: deterministisch + Golden Dataset (0012)
Betrieb
  ├─> Langfuse + bestehendes Monitoring, DSFA/Betriebsrat (0011)
  └─> Versionierung leichtgewichtig (0013), Lineage über Sanitizing (0016)
```
