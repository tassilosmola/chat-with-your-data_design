# ADR-0006: Vektordatenbank – Qdrant

- **Status:** accepted
- **Datum:** 2026-10-24
- **Abhängig von:** ADR-0001 (OnPrem), ADR-0007 (ACL-Filter)

## Kontext

Die Vektor-DB muss (a) onPrem auf K8s laufen, (b) performantes Payload-/Metadaten-Filtering für ACLs können, (c) mit mittleren Datenmengen (100k–1M Chunks) klarkommen und (d) hybrid erweiterbar sein (sparse + dense).

## Entscheidung

**Qdrant self-hosted im K8s-Cluster.**

## Betrachtete Alternativen

| Alternative | Pro | Warum verworfen |
|---|---|---|
| pgvector (Postgres) | Kein neuer Service; bekanntes Betriebsmodell | Filtering/Skalierung bei wachsender Chunk-Zahl schwächer; Postgres wird Single Point für zwei Workloads |
| OpenSearch | Hybrid BM25+Vector out-of-the-box; oft schon vorhanden | Schwereres Betriebsmodell (JVM, Cluster-Mgmt); Overkill wenn nur Vektorsuche+Filter gebraucht wird |
| Weaviate | Hybrid Search eingebaut, gute Doku | Ähnliche Capability wie Qdrant; Qdrant hat schlankere Ops + sehr gutes Payload-Filtering |
| Azure AI Search | Managed, Hybrid, Security-Trimming | Cloud-only → widerspricht ADR-0001 |

## Konsequenzen

**Positiv**
- Dedizierter, schlanker Service mit gutem K8s-Support (Helm Chart, Snapshots)
- Payload-Index auf ACL-Metadaten → serverseitige Filterung ohne Post-Processing (ADR-0007)
- Sparse Vectors (BM25-ähnlich via z.B. FastEmbed) als späterer Hybrid-Pfad ohne DB-Wechsel

**Negativ / Pflichten**
- Backup/Disaster-Recovery selbst organisieren (Snapshots auf PVC/Object-Storage)
- Shard-/Replikations-Setup für HA muss geplant werden
- Index-Versionierung nötig, da Embedding-Modell-Wechsel Re-Indexierung erfordert (ADR-0009)
