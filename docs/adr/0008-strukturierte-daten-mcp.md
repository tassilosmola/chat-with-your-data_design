# ADR-0008: Zugriff auf strukturierte Daten – MCP-Tools / API-Layer

- **Status:** accepted
- **Datum:** 2026-10-24
- **Zusammenhang:** ADR-0004 (Datenquellen)

## Kontext

Strukturierte Daten (Tabellen, Kennzahlen, Fachdatenbanken) lassen sich nicht sinnvoll über Vektor-RAG beantworten – Aggregationen und exakte Werte brauchen deterministische Abfragen. Die Frage: frei generiertes SQL vs. kuratierter Zugriff.

## Entscheidung

**Kuratierte Tools über einen MCP-Server (Model Context Protocol):**

- Definierte, read-only Tools mit klarer Signatur, z.B. `get_headcount(department)`, `search_employee(id)`, `get_metric(name, period)`
- Jedes Tool = dokumentierte, testbare Funktion mit eigenem Auth/Scope
- Der Agent im Chat-Backend wählt Tools per Function-Calling; Ergebnisse fließen als Kontext in die Antwort
- MCP als offenes Protokoll → Tools später auch für andere Agenten/Clients nutzbar

## Betrachtete Alternativen

| Alternative | Pro | Warum verworfen |
|---|---|---|
| Text-to-SQL Agent | Maximale Flexibilität | Falsche SQL = falsche Zahlen mit selbstbewusster Antwort; Injection-/Leak-Risiko; Guardrail-Aufwand hoch für PoC |
| Semantischer Layer (Cube/dbt) | Sauberste Governance, Business-Metriken zentral definiert | Zusätzlicher Stack (Metrik-Modellierung) – für v1 zu schwer; spätere Erweiterung unter dem MCP-Layer möglich |
| Direkte REST-APIs im Backend | Einfach | MCP standardisiert Discovery+Schema → entkoppelt Tools vom Backend |

## Konsequenzen

- Tool-Katalog muss mit Fachbereichen kuratiert werden (Welche Fragen sind erlaubt/sinnvoll?)
- Read-only-DB-User + Row-Level-Security für personenbezogene Tabellen
- Tool-Definitionen sind gleichzeitig das Governance-Dokument ("der Bot darf genau das")
- RAG (Dokumente) + Tools (Daten) kombinierbar → Antworten wie "laut Richtlinie X, aktuell N Fälle"
