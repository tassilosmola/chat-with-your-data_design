# ADR-0004: Datenquellen – PDFs + strukturierte Daten

- **Status:** accepted
- **Datum:** 2026-10-24
- **Zusammenhang:** ADR-0006 (Vektor-DB), ADR-0008 (MCP-Tools)

## Kontext

Die Aufgabe lässt Datenquellen frei wählbar. Entscheidend für die Architektur: unterschiedliche Quelltypen brauchen unterschiedliche Zugriffspfade – semantische Suche funktioniert für Dokumente, nicht für relationale Daten.

## Entscheidung

**Zwei Quelltypen, zwei Zugriffspfade:**

1. **PDF/Dokumente** (Richtlinien, Handbücher, Konzepte) → Ingestion-Pipeline → Chunking → Embeddings → Qdrant → RAG
2. **Strukturierte Daten** (Tabellen/Fachdatenbanken) → kuratierte MCP-Tools → deterministischer, read-only Zugriff

Das Orchestrator-Backend (Agent) entscheidet pro Anfrage, welcher Pfad – oder beide kombiniert – genutzt wird.

Die Connector-Architektur ist bewusst erweiterbar (SharePoint/Confluence/FAQ später), ohne die Kernarchitektur zu ändern: neue Quelle = neuer Ingestion-Konnektor bzw. neues MCP-Tool.

## Betrachtete Alternativen

| Alternative | Warum verworfen |
|---|---|
| Nur Dokumente | Strukturierte Daten über Dump→Chunking→RAG verliert Präzision (Zahlen, Aggregationen halluzinationsanfällig) |
| Nur strukturierte Daten | Deckt den typischen "Chat with your Data"-Use-Case (Dokumentenwissen) nicht ab |
| SharePoint/Confluence in v1 | Konnektor-Aufwand (Auth, Export, Inkremente) verzögert PoC; Architektur hält Option offen |

## Konsequenzen

- Ingestion und Tool-Zugriff sind getrennte Pipelines mit eigener Qualitätssicherung (ADR-0012)
- Agent-Orchestrierung wird benötigt (LangChain/LlamaIndex o.ä.) → Komplexität höher als "reines" RAG
- Fachlich spannend für die Präsentation: zeigt Verständnis, dass "Daten" ≠ "Dokumente"
