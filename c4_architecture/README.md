# C4-Architektur (LikeC4)

Zoombares Architekturmodell für „Chat with your Data" in [LikeC4](https://likec4.dev) DSL.
Ein einziges Modell (`architecture.c4`), alle Ebenen sind **Views auf dasselbe Modell** –
Änderungen an einer Stelle aktualisieren alle Diagramme.

## Ebenen / Flughöhen

| View | Ebene | Inhalt | Zoom-Einstieg |
|---|---|---|---|
| `index` | **L1 · Systemkontext** | Akteur (10k MA), System, externe Systeme (Entra ID, Quellen, Fach-DBs, Modellquelle) | Klick auf *Chat with your Data* → L2 |
| `l2_overview` | **L2 · Überblick** | Reduziert auf den Live-Request-Pfad: Web-UI, Ingress, Chat API, Gateway, vLLM, Embedding, Qdrant, MCP + externe Systeme | **Default-Zoom aus L1**; Klick auf *Chat API / Gateway / Qdrant* → L3 |
| `l2_container` | **L2 · Container** | Alle Container, **gruppiert nach Funktionsbereichen** (Zugriff, Chat-Backend, Modell-Serving, Retrieval & Daten, Ingestion, Observability, Versionierung, Skalierung) | Klick auf *Chat API / Ingestion / Gateway / Qdrant* → L3 |
| `l2_llm_abstraktion` | **L2 · LLM-Abstraktionsschicht** | Gezielte Sicht auf das LLM-Gateway: **Konsumenten** (Chat API = Serving, Ingestion = Embeddings) → Gateway (aufgeklappt mit Komponenten) → **Backends** (vLLM, Embedding) + Governance (MLflow, Langfuse) | Klick auf *LLM-Gateway* → `l3_gateway` |
| `l3_api` | **L3 · Komponenten** | Chat API: Auth-MW → Agent → Retrieval(ACL) / Tool-Client → Citations → Feedback | – |
| `l3_ingest` | **L3 · Komponenten** | Ingestion: Konnektoren → Parser → Chunker → ACL-Enrichment → Embed → Index-Writer | – |
| `l3_gateway` | **L3 · Komponenten** | LLM-Gateway: Router, Rate-Limit/Backpressure, user-scoped Cache, Trace-Exporter | – |
| `l3_qdrant` | **L3 · Komponenten** | Qdrant-Interna: versionierte Collections, `acl_groups` Payload-Index | – |
| `l4_deployment` | **L4 · Deployment** | OnPrem DC → K8s Cluster → CPU- vs. GPU-Nodepool, Pods, Operatoren | Klick auf *GPU Nodepool* → `l4_scaling` |
| `l4_scaling` | **L4 · Skalierung** | KEDA ScaledObject (Queue-Länge `num_requests_waiting`) → vLLM-Replicas; Cluster-Autoscaler → GPU-Nodes; Warm-Pool vs. Scale-to-Zero | – |
| `flow_rag_ingestion` | **Flow · Ingestion** | *Dynamic View* – nummerierte Schrittfolge: Dokumente → Parser → Chunker → ACL-Enrichment (AD-Gruppen) → Embeddings via Gateway → Upsert in `rag_vN` → DVC-Artefakte | – |
| `flow_rag_serving` | **Flow · Serving** | *Dynamic View* – nummerierte Schrittfolge: Anfrage → OIDC → Agent → parallel Retrieval (ACL-Filter) + Tool-Calls → LLM via Gateway → Citations → Antwort + Trace | – |

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

Jeder Container trägt im `description` seine ADR-Referenz (ADR-0001 … ADR-0014).
Tags: `#gpu` = GPU-abhängig, `#pii` = verarbeitet personenbezogene Daten.
Beziehungsstile: punktiert = Telemetrie/Sync, gestrichelt orange = Skalierung.
