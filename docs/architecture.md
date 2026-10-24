# Architektur – „Chat with your Data"

Alles läuft OnPrem im Kubernetes-Cluster (ADR-0001). Zwei Datenpfade: Dokumente über RAG (Qdrant), strukturierte Daten über kuratierte MCP-Tools. LLM und Embeddings hängen hinter einem Gateway und sind austauschbar.

## Architekturdiagramm

```mermaid
flowchart TB
    subgraph Zugriff["Zugriff (ADR-0005)"]
        MA["~10k Mitarbeitende"]
        Web["Referenz Web-UI (PoC)"]
        Clients["Teams / weitere Clients (später)"]
        MA --> Web
        MA --> Clients
    end

    subgraph Edge["Edge"]
        Ingress["K8s Ingress / API-GW"]
        Entra["Entra ID / AD<br/>OIDC · Gruppen · ACL-Sync"]
        Ingress <-->|"JWT + AD-Gruppen"| Entra
    end

    subgraph Core["Chat Backend"]
        Agent["Chat API + Agent<br/>FastAPI · LangChain<br/>/chat · /feedback"]
        GW["LLM-Gateway (ADR-0002)<br/>LiteLLM · Rate-Limits<br/>user-scoped Cache"]
        LLM["LLM Serving (ADR-0003)<br/>vLLM · ~70B Q4 · GPU"]
        Emb["Embedding-Service (ADR-0009)<br/>bge-m3 multilingual"]
        Agent --> GW
        GW --> LLM
        GW --> Emb
    end

    subgraph Data["Retrieval & Daten"]
        Qdrant[("Qdrant (ADR-0006)<br/>Vektoren + ACL-Metadaten")]
        MCP["MCP-Server (ADR-0008)<br/>kuratierte read-only Tools"]
        DBs[("Fach-DBs<br/>read-only · Row-Level-Security")]
        MCP --> DBs
    end

    subgraph Ingest["Ingestion-Pipeline (ADR-0004)"]
        Conn["Konnektoren<br/>PDF / Dokumente"]
        Pipe["Parser · Chunking<br/>ACL-Enrichment"]
        Conn --> Pipe
    end

    subgraph Ops["Observability & Qualität (ADR-0011, -0012)"]
        LF["Langfuse (self-hosted)<br/>LLM-Traces · Kosten · Prompts"]
        OTel["OpenTelemetry + Grafana<br/>Latenz · Fehler · GPU"]
        Eval["Eval-Pipeline<br/>Golden Dataset · Ragas CI-Gate"]
        FB["Feedback-Loop<br/>thumbs · Kommentare"]
    end

    Web --> Ingress
    Clients --> Ingress
    Ingress --> Agent

    Agent -->|"Query + User-Gruppen<br/>serverseitiger ACL-Filter (ADR-0007)"| Qdrant
    Agent -->|"Function-Calling"| MCP
    Agent -->|"Traces: Prompt · Retrieval · Antwort"| LF
    Agent -.-> OTel

    Pipe -->|"embed"| Emb
    Pipe -->|"Chunks + acl_groups"| Qdrant
    Entra -.->|"Gruppen-Sync"| Pipe

    Agent --> FB
    FB -->|"neue Eval-Cases"| Eval
    Eval -.->|"Gate vor Release"| Agent
```

## Request-Flow (eine Chat-Anfrage)

1. **Auth:** Nutzer → Ingress → OIDC-Token (Entra ID) mit AD-Gruppen
2. **Orchestrierung:** Agent entscheidet – RAG-Pfad, Tool-Calls via MCP, oder beides
3. **Retrieval:** Qdrant-Query mit **serverseitigem ACL-Filter** (User-Gruppen vs. `acl_groups`-Payload) → nur berechtigte Chunks
4. **Generation:** Kontext + System-Prompt ("nur aus Kontext antworten") → LLM-Gateway → vLLM
5. **Antwort:** Text + **Citations** (doc_id, Seite) → Nutzer kann jede Aussage prüfen
6. **Nachvollziehbarkeit:** kompletter Trace in Langfuse (Modellversion, Chunks, Latenz, Kosten)
7. **Qualität:** Feedback fließt ins Golden Dataset; CI-Evals verhindern Regressionen

## Komponenten-Übersicht

| Komponente | Technologie | ADR |
|---|---|---|
| Deployment | Kubernetes onPrem, GPU-Nodepool | 0001 |
| Chat-Backend / Agent | FastAPI, LangChain | 0005, 0004 |
| LLM-Gateway | LiteLLM Proxy | 0002 |
| LLM-Serving | vLLM, ~70B Q4 (Sizing offen) | 0003 |
| Embeddings | bge-m3 / multilingual-e5, self-hosted | 0009 |
| Vektor-DB | Qdrant (ACL-Payload-Filter) | 0006, 0007 |
| Strukturierte Daten | MCP-Server, kuratierte Tools | 0008 |
| Auth | Entra ID / AD, OIDC | 0005, 0007 |
| LLM-Observability | Langfuse (self-hosted) | 0011 |
| Infra-Observability | OpenTelemetry + Grafana | 0011 |
| Eval | Golden Dataset, Ragas/DeepEval, Feedback-API | 0012 |
| Halluzinationsschutz | RAG + Citations (Reranking/Groundedness als Roadmap) | 0010 |

## Zentrale Risiken & Mitigation

| Risiko | Mitigation | ADR |
|---|---|---|
| Personenbezogene Daten leaken über Prompt-Cache/Traces | User-scoped Cache-Keys, Langfuse-Zugriff + Retention, Pseudonymisierung | 0002, 0011 |
| Rechte-Drift (Quell-ACLs vs. Index) | Inkrementeller ACL-Sync, max. Sync-Latenz als SLO, Audit-Log | 0007 |
| Halluzination trotz RAG | Citations-Pflicht; Eval-Gate (Groundedness); Roadmap: Reranking/Groundedness-Check | 0010, 0012 |
| Modellqualität onPrem < Cloud-LLM | ~70B-Klasse + Quantisierung; Gateway erlaubt Wechsel; Eval misst objektiv | 0002, 0003 |
| GPU-Kapazität für 10k MA | Continuous Batching, Queue-Backpressure, horizontale Skalierung; Lasttest vor Rollout | 0003 |

## Grobe Phasenplanung (Orientierung)

| Phase | Inhalt | Grob-Schätzung |
|---|---|---|
| 0 – Setup | GPU-Nodes, Qdrant, Gateway, Langfuse im Cluster | 1–2 Wochen |
| 1 – PoC | Ingestion (PDFs), RAG-Basis, Chat-API, Referenz-UI, Citations | 3–4 Wochen |
| 2 – Härtung | ACL-Filter + AD-Sync, MCP-Tools, Golden Dataset + CI-Evals | 3–4 Wochen |
| 3 – Pilot | Fachbereich-Pilot (~100 MA), Feedback-Loop, Lasttest | 4–6 Wochen |
| 4 – Rollout | Stufenweiser Ausbau, Kanal-Entscheidung (Teams/Web), Betriebsübergabe | fortlaufend |

*Annahme: 2–3 Personen, bestehende K8s-Plattform, Fachbereiche liefern Golden-Dataset-Fragen.*
