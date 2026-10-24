---
name: chat-with-your-data
description: Konzeptkontext „Chat with your Data" – Chatbot auf Unternehmensdaten (RAG, ~10k MA, OnPrem). Enthält Aufgabenstellung, getroffene Architektur-Entscheidungen und Konventionen für ADRs und Architekturdiagramm.
allowed-tools:
  - read
  - write
  - edit
  - grep
  - glob
---

# Skill: Chat with your Data – Konzeptkontext

Du arbeitest am Konzept für einen Unternehmens-Chatbot („Chat with your Data"). Das Ergebnis ist eine **konzeptionelle Lösung** – keine vollständige Implementierung. Ziel: klare, nachvollziehbare Architekturentscheidungen.

## Aufgabenstellung (Kurzfassung)

- Chatbot greift auf unternehmenseigene Daten zu, liefert zuverlässige, **halluzinationsarme** Antworten via modernem LLM
- Zielgruppe: potenziell **alle ~10.000 Mitarbeitende**
- Retrieval-Verfahren (RAG) zur Halluzinationsreduktion; OnPrem oder Cloud möglich
- Bewertet werden: Vorgehen, Komponenten (Datenvorverarbeitung, Embeddings, Vektor-DB, Frontend, Rechtevergabe), Tool-Wahl mit Begründung, Architekturdiagramm, Zeitschätzung, Risiken, optional PoC
- Besonders interessant: Strukturierung, Prioritäten, Herausforderungen (Halluzination, Datenqualität, Rechte, Skalierung), spannende Use Cases

## Getroffene Entscheidungen (Baseline)

Diese Entscheidungen wurden mit dem Auftraggeber getroffen und sind in `docs/adr/` dokumentiert. **Sie sind die Baseline** – arbeite konsistent mit ihnen, ändere sie nicht stillschweigend:

| Entscheidung | Stand |
|---|---|
| Deployment: OnPrem auf Kubernetes (volle Datenhoheit, GPU selbst betreiben) | ADR-0001, accepted |
| LLM hinter Gateway-Abstraktion (LiteLLM), austauschbar, user-scoped Cache | ADR-0002, accepted |
| LLM-Größe: offen – Empfehlung ~70B Q4 auf 2 GPUs, Sizing-Kriterien im ADR | ADR-0003, **open** |
| Datenquellen: PDF/Dokumente via RAG + strukturierte Daten via Tools | ADR-0004, accepted |
| Zugriff: API-first (FastAPI, OIDC/Entra ID), Frontend nur Referenz-PoC | ADR-0005, accepted |
| Vektordatenbank: Qdrant self-hosted | ADR-0006, accepted |
| Dokumentberechtigungen: `acl_groups`-Metadaten, serverseitiger Filter beim Retrieval | ADR-0007, accepted |
| Strukturierte Daten: kuratierte MCP-Tools (kein freies Text-to-SQL) | ADR-0008, accepted |
| Embeddings: self-hosted multilingual (bge-m3/e5) hinter demselben Gateway | ADR-0009, accepted |
| Anti-Halluzination: Citations als Pflicht; Reranking/Groundedness/Fallback als Roadmap | ADR-0010, accepted |
| Observability: Langfuse self-hosted + OTel/Grafana; Datenschutz-Pflichten für Traces | ADR-0011, accepted |
| Eval: Golden Dataset + CI-Evals (Ragas/DeepEval) + User-Feedback-Loop | ADR-0012, accepted |
| Versionierung: DVC (Corpus, Pipeline-Artefakte) + MLflow (Model Registry, Eval-Runs); versionierte Qdrant-Collections | ADR-0013, accepted |
| Skalierung: KEDA ScaledObject auf Queue-Länge (vLLM-Prometheus-Metriken), Node-Autoscaling, Warm-Pool vs. Scale-to-Zero | ADR-0014, accepted |

Aktuellen Stand immer zuerst aus `docs/adr/README.md` lesen – einzelne ADRs können sich geändert haben.

## Repo-Struktur & Konventionen

- ADRs: `docs/adr/NNNN-titel.md`, durchnummeriert ab 0001, vereinfachtes MADR-Format
- Architektur: `docs/architecture.md` – Mermaid-Diagramm + Request-Flow + Komponenten- und Risikotabellen + Phasenplanung
- LikeC4: `c4_architecture/architecture.c4` – ein Modell, zoombare Views L1 (Kontext) → L2 (Container) → L3 (Komponenten) → L4 (Deployment/KEDA); Ebenen-Tabelle in `c4_architecture/README.md`
- Arbeitsnotizen/Aufgabenstellung: `Structure.md` (roh, nicht verschönern)
- Sprache aller Docs: **Deutsch**

### ADR-Format (MADR-lite)

```markdown
# ADR-NNNN: Titel

- **Status:** proposed | accepted | open | superseded by ADR-XXXX
- **Datum:** YYYY-MM-DD
- **Abhängig von / Zusammenhang:** ADR-XXXX (falls relevant)

## Kontext
## Entscheidung
## Betrachtete Alternativen   (Tabelle: Alternative | Warum verworfen)
## Konsequenzen               (Positiv / Negativ / Risiken+Mitigation / Pflichten)
```

## Workflow bei typischen Aufgaben

1. **Neue Entscheidung/Frage klären:** Offene Punkte zuerst dem Nutzer stellen (Auswahlfragen mit Optionen + Trade-offs), dann als ADR dokumentieren – nicht einfach selbst festlegen. Nächste freie Nummer verwenden, Index aktualisieren.
2. **Entscheidung ändern:** Alten ADR auf `superseded by ADR-XXXX` setzen, neuen ADR anlegen, Diagramm + Komponenten-Tabelle synchronisieren. Keine ADRs umschreiben – Nachvollziehbarkeit ist der Zweck.
3. **Diagramm aktualisieren:** `docs/architecture.md` (Mermaid) **und** `c4_architecture/architecture.c4` (LikeC4) synchron halten; Mermaid: Label mit Sonderzeichen in `"…"`, `<br/>` für Zeilenumbrüche. ADR-Referenzen in Labels/descriptions pflegen.
4. **Konsistenz wahren:** Jede Komponente im Diagramm sollte einem ADR zugeordnet sein; neue Risiken in die Risikotabelle aufnehmen.

## Implizite Abhängigkeiten beachten

- OnPrem (0001) ⇒ keine SaaS-Komponenten ohne Datenabfluss-Prüfung; Langfuse statt LangSmith etc.
- ACL-Metadaten-Filter (0007) ⇒ Ingestion muss AD-Gruppen-Sync enthalten; Prompt-Cache user-scoped (0002)
- Embedding-Wechsel (0009) ⇒ Index-Versionierung + Re-Indexierung in Qdrant (0006)
- Citations (0010) ⇒ API-Response enthält `citations`; Retrieval-Metadaten bis zur Response tragen (0005)
- Langfuse vs. MLflow (0011/0013): Langfuse = Request-Traces, MLflow = Registry + Eval-Experimente – keine Doppelung einführen
- KEDA (0014) ⇒ Queue-Backpressure am Gateway Pflicht; GPU-Kaltstart ⇒ `minReplicaCount ≥ 1` zu Geschäftszeiten

## Tonalität für die Präsentation

Entscheidungen immer mit „Warum" begründen (Trade-offs, nicht nur Wahl). Risiken offen benennen und Mitigation zeigen – das ist im Interview gefragt, keine Schwäche.
