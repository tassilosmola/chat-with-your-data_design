# ADR-0008: Zugriff auf strukturierte Daten – MCP-Tools (v2)

- **Status:** proposed – **Roadmap v2** (siehe [Roadmap](../roadmap.md))
- **Datum:** 2026-10-25
- **Zusammenhang:** ADR-0004 (Datenquellen), ADR-0015 (Scope)

## Kontext

Strukturierte Daten (Tabellen, Kennzahlen, Fachdatenbanken) lassen sich nicht sinnvoll über Vektor-RAG beantworten – Aggregationen und exakte Werte brauchen deterministische Abfragen. Die Frage: frei generiertes SQL vs. kuratierter Zugriff.

v1 enthält keine strukturierten Daten (ADR-0004). Dieses ADR legt fest, **wie** der zweite Datenpfad angebunden wird, damit die v1-Architektur ihn nicht verbaut.

## Entscheidung

**Kuratierte, read-only Tools über MCP-Server (Model Context Protocol) als Konnektor-Standard für v2.**

- Definierte Tools mit klarer Signatur, z.B. `get_vacation_balance()` (eigener Resturlaub), `get_travel_allowance(country, hours)` (Pauschalen-Tabelle), `get_open_positions(department)`
- Jedes Tool = dokumentierte, testbare Funktion mit eigenem Scope
- Die Chat-API wählt Tools per Function-Calling; Ergebnisse fließen als Kontext mit Citation (`source: tool:<name>`) in die Antwort
- Ein MCP-Server pro Fachdomäne (HR, Finance, …) → Fachbereiche können eigene Tools liefern, ohne die Chat-API zu ändern

**Identitäts-Propagation (Pflicht):** Tools laufen **im Kontext des anfragenden Nutzers**, nicht mit einem technischen Super-User. Die Chat-API reicht das User-Token per OAuth **On-Behalf-Of** (Entra ID) an den MCP-Server weiter; die Fach-DB setzt Row-Level-Security auf Basis dieser Identität durch. „Eigener Resturlaub“ heißt technisch: der Nutzer sieht nur seine eigene Zeile.

**Warum MCP statt In-Process-Function-Calling?** Für *ein* Backend wäre In-Process einfacher. MCP lohnt sich, sobald (a) mehrere Fachbereiche Tools liefern und (b) dieselben Tools auch anderen Clients/Agenten (z.B. Copilot-Integrationen, IDE-Agents) zur Verfügung stehen sollen. Beides ist für v2 erwartbar – daher MCP als Standard, aber erst mit dem ersten Tool-Use-Case.

## Betrachtete Alternativen

| Alternative | Pro | Warum verworfen |
|---|---|---|
| Text-to-SQL Agent | Maximale Flexibilität | Falsches SQL = falsche Zahlen mit selbstbewusster Antwort; Injection-/Leak-Risiko; hoher Guardrail-Aufwand |
| Semantischer Layer (Cube/dbt) | Business-Metriken zentral definiert | Zusätzlicher Stack; später **unter** dem MCP-Layer ergänzbar |
| In-Process-Tools in der Chat-API | Einfachste Lösung | Koppelt Fachlogik an die Chat-API; keine Wiederverwendung durch andere Clients |

## Konsequenzen

- Auslöser für v2: erster priorisierter Use Case mit strukturierten Daten + Fachbereich als Tool-Owner
- Tool-Katalog mit Fachbereichen kuratieren – Tool-Definitionen sind gleichzeitig das Governance-Dokument ("der Bot darf genau das")
- Read-only-Zugriff + Row-Level-Security; OBO-Flow muss mit Entra/IAM abgestimmt werden
- Mit Tools wird die feste Pipeline (ADR-0004) zum Tool-Calling-Ablauf – Eval (ADR-0012) erweitert sich um Tool-Auswahl-Korrektheit
- RAG + Tools kombinierbar → Antworten wie "laut Reisekostenrichtlinie § 4 gilt X; Ihre Pauschale für 10 h Inland beträgt Y €"
