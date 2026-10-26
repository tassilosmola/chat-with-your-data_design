# ADR-0004: Datenquellen & Pipeline – v1 Dokumente, v2 strukturierte Daten

- **Status:** proposed
- **Datum:** 2026-10-25
- **Zusammenhang:** ADR-0006 (Vektor-DB), ADR-0008 (MCP-Tools, v2), ADR-0015 (Scope), ADR-0016 (PII-Sanitizing)

## Kontext

Die Aufgabe lässt Datenquellen frei wählbar. Unterschiedliche Quelltypen brauchen unterschiedliche Zugriffspfade – semantische Suche funktioniert für Dokumente, nicht für relationale Daten (Aggregationen, exakte Werte).

Die vorliegenden PoC-Daten (`rag-resources/`) sind **ausschließlich Dokumente**: Tarifvertrag, Manteltarifvertrag, Arbeitszeit- und Reisekostenrichtlinie (HR-Self-Service, ADR-0015).

## Entscheidung

**v1: Ein Quelltyp, eine feste Pipeline.**

- **PDF/Dokumente** → Ingestion (Parsing inkl. Tabellen, **PII-Sanitizing** (ADR-0016), Chunking, ACL-Metadaten) → Embeddings → Qdrant
- Serving als **deterministische Pipeline** statt Agent:
  1. Query-Rewriting mit Chat-Historie (Folgefragen → eigenständige Suchanfrage)
  2. Retrieval mit ACL-Filter (ADR-0007)
  3. Generation mit Kontext + Citations (ADR-0010)
- Umsetzung in **plain Python** (FastAPI + OpenAI-SDK gegen Gateway + `qdrant-client`) – kein Agent-Framework nötig

**v2: Strukturierte Daten als zweiter Pfad** über kuratierte Tools (ADR-0008, MCP). Erst dann wird aus der Pipeline ein Tool-Calling-Ablauf (LLM wählt Retrieval und/oder Tool).

Die Ingestion ist über Konnektoren erweiterbar (SharePoint/Confluence/FAQ), ohne die Kernarchitektur zu ändern.

## Betrachtete Alternativen

| Alternative | Warum verworfen |
|---|---|
| Agent-Orchestrierung (LangChain/LlamaIndex) ab v1 | Bei nur einem Pfad kein Nutzen; nicht-deterministische Pfadwahl erschwert Eval und Debugging; zusätzliche Framework-Abhängigkeit |
| Dokumente + strukturierte Daten in v1 | Keine strukturierten PoC-Daten vorhanden; Tool-Governance (Fachbereiche, Auth-Propagation) verzögert PoC |
| Strukturierte Daten als Dump → Chunking → RAG | Verliert Präzision (Zahlen, Aggregationen halluzinationsanfällig) |
| SharePoint/Confluence in v1 | Konnektor-Aufwand (Auth, Export, Inkremente, ACL-Mapping) verzögert PoC |

## Konsequenzen

- Pipeline-Schritte sind einzeln testbar und in Langfuse als Spans sichtbar (ADR-0011)
- **Tabellen-Parsing** ist kritisch: Reisekosten-/Tarif-Tabellen (Pauschalen, Entgeltgruppen) müssen strukturerhaltend extrahiert werden (z.B. als Markdown-Tabelle im Chunk) – eigener Qualitätscheck in der Ingestion (ADR-0012)
- **Personenbezogene Daten** in den Dokumenten werden nach dem Parsing und vor dem Chunking redigiert; Dokumente mit Findings werden erst nach HR-Freigabe indexiert (ADR-0016)
- **Dokument-Lebenszyklus:** Ingestion erkennt geänderte/gelöschte Dokumente über Content-Hash-Manifest (ADR-0013) und entfernt veraltete Chunks aus dem Index; eine neue Sanitizer-Version löst ebenfalls Re-Ingest aus
- Der Übergang zu v2 ist additiv: Retrieval wird zu einem „Tool“ unter mehreren
