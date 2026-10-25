# ADR-0015: Use Case & Scope v1 – HR-Self-Service mit minimaler Architektur

- **Status:** proposed
- **Datum:** 2026-10-25
- **Zusammenhang:** alle ADRs; Roadmap v2: [`docs/roadmap.md`](../roadmap.md)

## Kontext

Die Aufgabe fragt nach Priorisierung und spannenden Use Cases. Ohne festen ersten Use Case wächst die Architektur in alle Richtungen ("könnte man auch noch brauchen"). Ein Team von 2–3 Personen muss in wenigen Wochen einen belastbaren PoC liefern, der trotzdem auf 10k MA skaliert.

### Use-Case-Bewertung

| Use Case | Fragevolumen | Datenbasis klar | Risiko | Messbarer Nutzen | Priorität |
|---|---|---|---|---|---|
| **HR-Self-Service** (Tarif, Arbeitszeit, Urlaub, Reisekosten) | sehr hoch, betrifft alle MA | ja – PoC-Daten vorhanden | niedrig (allgemein freigegebene Regelungen) | HR-Ticket-Reduktion | **v1** |
| IT-Helpdesk / Onboarding | hoch | Wiki/KB, oft verstreut | niedrig | Ticket-Reduktion | v2 |
| Persönliche Auskünfte (Resturlaub, eigene Reisekosten) | hoch | strukturierte HR-Systeme | mittel (personenbezogen) | hoch | v2 (MCP, ADR-0008) |
| Richtlinien/Compliance-Fragen bereichsübergreifend | mittel | viele Quellen, unterschiedliche ACLs | mittel | Zeitersparnis | v2+ |

**Warum HR-Self-Service zuerst:** Jede/r der 10k MA hat Fragen dazu ("Wie viele Urlaubstage bei 4-Tage-Woche?", "Welche Pauschale bei 10 h Dienstreise?", "Welche Entgeltgruppe nach Stufenaufstieg?"). Die Dokumente sind dicht, tabellenlastig und juristisch formuliert – ein realistischer Härtetest für Parsing, Retrieval und Citations, bei geringem Berechtigungsrisiko.

## Entscheidung

**v1-Scope: 8 Bausteine, alles andere ist Roadmap mit Auslöser.**

| # | Baustein | ADR |
|---|---|---|
| 1 | Referenz-Web-UI | 0005 |
| 2 | Chat-API (FastAPI, feste RAG-Pipeline, JWT-Validierung) | 0004, 0005 |
| 3 | LLM-Gateway (LiteLLM: Quotas, Backpressure, Routing) | 0002 |
| 4 | LLM-Serving (vLLM) + KEDA-Scaling auf Queue-Länge | 0003, 0014 |
| 5 | Embedding-Service (bge-m3) | 0009 |
| 6 | Qdrant (Hybrid + ACL-Filter, versionierte Collections) | 0006, 0007, 0013 |
| 7 | Postgres (Chat-Verläufe, Feedback, Ingestion-Manifest) | 0005, 0013 |
| 8 | Langfuse (Traces, Prompts, Datasets, Eval-Scores) | 0011, 0012 |

**Wiederverwendet (keine eigene Komponente):** Entra ID, Ingress-Controller, Prometheus/Grafana, Object-Storage, CI/CD.

**Nicht in v1 (siehe Roadmap):** MCP-Tools / strukturierte Daten, Reranking, Groundedness-Check, weitere Konnektoren (SharePoint/Confluence), Teams-Kanal, MLflow/DVC.

## Konsequenzen

- Architektur ist auf einer Folie erklärbar; jeder Baustein hat genau eine Aufgabe
- Erweiterungen sind **additiv** (neues Tool, neuer Konnektor, Reranker-Stufe), keine Umbauten
- Fachbereich HR ist Owner für Golden Dataset (ADR-0012) und Content-Qualität
- Betriebsrat + Datenschutz (ADR-0011) werden **parallel zum PoC** eingebunden, nicht erst vor Rollout
