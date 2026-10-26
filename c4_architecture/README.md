# C4-Architektur (LikeC4)

Zoombares Architekturmodell für „Chat with your Data" in [LikeC4](https://likec4.dev) DSL.
Ein einziges Modell (`architecture.c4`), alle Ebenen sind **Views auf dasselbe Modell** –
Änderungen an einer Stelle aktualisieren alle Diagramme.

## Ebenen / Flughöhen

| View | Ebene | Inhalt | Zoom-Einstieg |
|---|---|---|---|
| `index` | **L1 · Systemkontext** | Akteure (10k MA, HR Content-Owner), System, externe Systeme (Entra ID, Dokumentquellen, Modell-Mirror, Plattform-Monitoring; Fach-DBs gestrichelt = v2) | Klick auf *Chat with your Data* → L2 |
| `l2_overview` | **L2 · Überblick v1** | Live-Request-Pfad: Web-UI, Ingress, Chat API, Gateway, vLLM, Embedding, Qdrant, Postgres | **Default-Zoom aus L1**; Klick auf *Chat API / Gateway / Qdrant* → L3 |
| `l2_container` | **L2 · Container v1** | Alle v1-Container, **gruppiert** (Zugriff, Chat-Backend, Modell-Serving, Retrieval & Persistenz, Ingestion, Observability & Qualität) | Klick auf *Chat API / Ingestion / Gateway / Qdrant* → L3 |
| `l2_llm_abstraktion` | **L2 · LLM-Abstraktionsschicht** | Konsumenten (Chat API, Ingestion) → Gateway (Quotas/Backpressure, Router, Tracing) → Backends (vLLM, Embedding) + Langfuse | Klick auf *LLM-Gateway* → `l3_gateway` |
| `l2_roadmap_v2` | **L2 · Roadmap v2** | v1-Kern + MCP-Server und Fach-DBs (gestrichelt) als zusätzlicher Konnektor | – |
| `l3_api` | **L3 · Komponenten** | Chat API: Auth-MW → RAG-Pipeline → Retrieval (ACL) → Citations → Feedback; Redaction-Review (HR); Tool-Client (v2) | – |
| `l3_ingest` | **L3 · Komponenten** | Ingestion: Konnektoren → Parser → PII-Sanitizer → Residual-Check → Chunker → ACL-Enrichment → Embed → Index-Writer | – |
| `l3_gateway` | **L3 · Komponenten** | LLM-Gateway: Quotas/Backpressure, Provider-Router, Trace-Exporter | – |
| `l3_qdrant` | **L3 · Komponenten** | Qdrant-Interna: versionierte Collections + Alias, `acl_groups` Payload-Index | – |
| `l4_deployment` | **L4 · Deployment** | OnPrem DC → K8s Cluster → CPU- vs. fester GPU-Nodepool, KEDA, Plattform-Monitoring | Klick auf *GPU Nodepool* → `l4_scaling` |
| `l4_scaling` | **L4 · Skalierung** | Prometheus (`num_requests_waiting`) → KEDA → vLLM-Replicas im festen Pool; Gateway-Backpressure; Warm-Pool / Cron | – |
| `flow_rag_ingestion` | **Flow · Ingestion** | *Dynamic View*: Dokumente + ACLs → Parser → PII-Sanitizer → Residual-Check (Status/HR-Freigabe) → Chunker → ACL-Enrichment → Embeddings via Gateway → Upsert in `rag_vN` → Manifest (Lineage-Hashes) | – |
| `flow_rag_serving` | **Flow · Serving** | *Dynamic View*: Anfrage → JWT-Prüfung → Query-Rewrite → Retrieval (ACL-Filter) → LLM via Gateway → geprüfte Citations → Verlauf + Trace | – |

## Ansehen

**VS Code:** Extension `LikeC4` installieren → `architecture.c4` öffnen → Preview.
Elemente mit eigener Scoped View sind anklickbar (Zoom). Zurück per Breadcrumb/Esc.

**Interaktiver Devserver (empfohlen für Präsentation):**

```bash
npx likec4 start c4_architecture/
# Browser: http://localhost:5173 – zoomen per Klick, Views in der Sidebar
```

**Playground (ohne Installation):** Inhalt von `architecture.c4` in
<https://playground.likec4.dev> einfügen.

**Export für Slides:**

```bash
npx likec4 export png c4_architecture/ --outdir c4_architecture/export
# oder: likec4 export svg / likec4 build → statische HTML-Seite
```

## Mapping Modell ↔ ADRs

Jeder Container trägt im `description` seine ADR-Referenz (ADR-0001 … ADR-0016).
Tags: `#gpu` = GPU-abhängig, `#pii` = verarbeitet personenbezogene Daten, `#v2` = Roadmap (gestrichelt, transparent).
Beziehungsstile: punktiert = Telemetrie, gestrichelt orange = Skalierung, gestrichelt grau = v2 (Roadmap).
