# Architecture Decision Records – „Chat with your Data"

Alle ADRs folgen einem vereinfachten MADR-Format: Kontext → Entscheidung → Alternativen → Konsequenzen.

| ADR | Titel | Status | Kurzentscheidung |
|-----|-------|--------|------------------|
| [0001](0001-deployment-onprem-kubernetes.md) | Deployment-Modell | accepted | OnPrem auf Kubernetes |
| [0002](0002-llm-gateway-abstraktion.md) | LLM-Abstraktion | accepted | LLM-Gateway (LiteLLM) als einheitliche Schnittstelle |
| [0003](0003-llm-modell-sizing.md) | LLM-Modell & GPU-Sizing | open | Sizing-Optionen abwägen, Gateway erlaubt späteren Wechsel |
| [0004](0004-datenquellen.md) | Datenquellen | accepted | PDF/Dokumente via RAG + strukturierte Daten via Tools |
| [0005](0005-zugriff-api-first.md) | Zugriffskanal | accepted | API-first, Frontend als Referenz-PoC |
| [0006](0006-vektordatenbank-qdrant.md) | Vektordatenbank | accepted | Qdrant self-hosted |
| [0007](0007-acl-metadata-filter.md) | Dokumentberechtigungen | accepted | Metadaten-Filter beim Retrieval |
| [0008](0008-strukturierte-daten-mcp.md) | Zugriff strukturierte Daten | accepted | MCP-Tools / kuratierter API-Layer |
| [0009](0009-embedding-strategie.md) | Embedding-Strategie | accepted | Self-hosted multilingual, via Gateway |
| [0010](0010-anti-halluzination.md) | Halluzinationsminderung | accepted | Citations als Pflicht, Reranking/Groundedness als Option |
| [0011](0011-observability.md) | Observability & Tracing | accepted | Langfuse self-hosted + OpenTelemetry/Grafana |
| [0012](0012-eval-strategie.md) | Evaluationsstrategie | accepted | Golden Dataset + CI-Evals + User-Feedback-Loop |

## Entscheidungslogik (Zusammenhang)

```
OnPrem (0001)
  └─> Self-hosted LLM + Embeddings (0003, 0009)
        └─> Gateway-Abstraktion für Wechselbarkeit (0002)
Datenquellen (0004)
  ├─> PDFs → Qdrant (0006) mit ACL-Metadaten (0007)
  └─> Strukturierte Daten → MCP-Tools (0008)
Qualität
  ├─> Citations (0010)
  └─> Eval-Pipeline (0012)
Betrieb
  └─> Langfuse + OTel (0011)
```
