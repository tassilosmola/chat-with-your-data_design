# ADR-0006: Vektordatenbank – Qdrant

- **Status:** proposed
- **Datum:** 2026-10-25
- **Abhängig von:** ADR-0001 (OnPrem), ADR-0007 (ACL-Filter), ADR-0009 (bge-m3)

## Kontext

Die Vektor-DB muss (a) onPrem auf K8s laufen, (b) performantes Payload-/Metadaten-Filtering für ACLs können, (c) mit mittleren Datenmengen (100k–1M Chunks) klarkommen und (d) Hybrid-Search (sparse + dense) unterstützen – deutsche Fachbegriffe, Paragraphen-Nummern und Entgeltgruppen („EG 7“, „§ 5 MTV“) werden von rein semantischer Suche schlecht getroffen.

## Entscheidung

**Qdrant self-hosted im K8s-Cluster.**

Ausschlaggebend ist **natives Hybrid-Search** mit den Dense- *und* Sparse-Vektoren aus bge-m3 (ADR-0009) in einer Query, kombiniert mit Payload-Filter auf `acl_groups` – ohne zusätzliche Volltext-Infrastruktur.

## Betrachtete Alternativen

| Alternative | Pro | Warum verworfen |
|---|---|---|
| pgvector (Postgres) | **KISS-Variante**: Postgres wird ohnehin gebraucht (ADR-0005, Langfuse); bei 100k–1M Chunks performant genug; Filter + HNSW ausreichend | Hybrid nur über separates `tsvector`-Ranking + eigene Fusion; bge-m3-Sparse nicht nativ nutzbar |
| OpenSearch | Hybrid BM25+Vector out-of-the-box; oft schon vorhanden | Schwereres Betriebsmodell (JVM, Cluster-Mgmt) – nur sinnvoll, wenn bereits betrieben |
| Weaviate | Hybrid Search eingebaut | Ähnliche Capability wie Qdrant; Qdrant hat schlankere Ops |
| Azure AI Search | Managed, Hybrid, Security-Trimming | Cloud-only → widerspricht ADR-0001 |

**Bewusster Trade-off:** Qdrant kostet einen zusätzlichen Service gegenüber pgvector. Zeigt die Eval (ADR-0012), dass Dense-only ausreicht, ist pgvector eine legitime Vereinfachung.

## Konsequenzen

**Positiv**
- Payload-Index auf ACL-Metadaten → serverseitige Filterung ohne Post-Processing (ADR-0007)
- Hybrid-Search ohne weitere Infrastruktur
- Guter K8s-Support (Helm Chart, Snapshots)

**Negativ / Pflichten**
- Backup/Disaster-Recovery selbst organisieren (Snapshots auf Object-Storage)
- Replikation für HA planen (in v1: 1 Replica + Snapshot genügt; der Index ist aus den Quellen jederzeit rekonstruierbar)
- Index-Versionierung über Collections (ADR-0013)
