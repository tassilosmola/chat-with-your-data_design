# ADR-0013: Modell- & Datenversionierung – DVC + MLflow

- **Status:** accepted
- **Datum:** 2026-10-24
- **Abhängig von:** ADR-0001 (OnPrem), ADR-0006 (Qdrant), ADR-0011 (Langfuse – Abgrenzung unten), ADR-0012 (Eval)
- **Adressiert Anforderung:** Nachvollziehbarkeit („Modell, Umgebung, Daten") + Reproduzierbarkeit der RAG-Pipeline

## Kontext

Eine Antwort des Bots hängt von mehreren versionierbaren Artefakten ab:

- **Daten:** Dokument-Corpus, Chunking-Parameter, geretrievte Chunks, Qdrant-Index-Stand
- **Modelle:** Chat-LLM-Revision, Embedding-Modell-Revision (Embedding-Wechsel = neuer Index, ADR-0009)
- **Code/Config:** Prompt-Templates, Retrieval-Parameter (top_k, Filter), Pipeline-Version

Für Halluzinations-Analysen, Eval-Regressionen (ADR-0012) und Rollbacks muss beantwortbar sein: *„Welche Datenversion + welches Modell hat diese Antwort erzeugt?"* – nicht nur auf Trace-Ebene, sondern reproduzierbar.

## Entscheidung

**DVC für Daten, MLflow für Modelle & Experimente:**

- **DVC** versioniert: Dokument-Corpus-Snapshots, Ingestion-Artefakte (Parse-Ergebnisse, Chunk-Manifeste mit `doc_id` + Content-Hash), Chunking-/Retrieval-Config. DVC-Remote auf internem Object-Storage (S3/MinIO onPrem). Git referenziert via `.dvc`-Pointer – Pipeline-Läufe sind reproduzierbar und diffbar.
- **Index-Versionierung:** Qdrant-Collections versioniert (`rag_v1`, `rag_v2`, …). Rebuild = neue Collection, alte bleibt für Rollback/A-B-Vergleich; Index-Version wird in jeden Langfuse-Trace geschrieben.
- **MLflow Model Registry:** Einträge für Chat-LLM und Embedding-Modell mit gepinnten HF-Revisions; Gateway-Config referenziert Registry-Version → Modellwechsel = Registry-Promotion (staging → production).
- **MLflow Tracking:** Eval-Runs aus ADR-0012 landen als Experiment-Runs (Metriken, Artefakte, verknüpfte Daten-/Modellversion). CI-Gate liest Scores aus MLflow.

**Abgrenzung zu Langfuse (ADR-0011):** Langfuse = Observability auf Request-Ebene (einzelner Trace). MLflow = Registry + Experiment-Ebene (Versionen, Eval-Verläufe). Keine funktionale Doppelung, aber bewusste Schnittstelle: Trace referenziert Modell- + Index-Version, Eval-Runs aggregieren Trace-Scores.

## Betrachtete Alternativen

| Alternative | Warum verworfen |
|---|---|
| Leichtgewichtig (Manifest + Collection-Versioning + HF-Pinning) | Hätte für PoC gereicht; aber Corpus-Reproduzierbarkeit + Eval-Linieage ohne DVC/MLflow deutlich schwächer |
| Nur MLflow | Registry ja, aber Datenversionierung ungelöst – RAG-Antworten hängen mehr vom Corpus als vom Modell ab |
| lakeFS / andere Datenversions-Tools | lakeFS zielt auf Data-Lake-Branches; DVC passt besser zu Pipeline-Artefakten + Git-Workflow |

## Konsequenzen

**Positiv**
- Vollständige Reproduzierbarkeit: Corpus-Stand + Pipeline-Version + Modell-Revision + Index-Version
- Rollback-Story für die Präsentation: „Modell-Update schlechtere Evals → Registry-Rollback + alte Collection bleibt live"
- Eval-Regression wird an Daten-/Modell-Versionen messbar (nicht nur zeitlich)

**Negativ / Pflichten**
- Zwei zusätzliche Plattformen: MLflow (Backend-Store Postgres + Artifact-Store MinIO) + DVC-Remote – beide onPrem betreibbar
- DVC-Disziplin nötig: Ingestion-Outputs müssen getrackt werden; Content-Hashes deduplizieren Doc-Updates
- Mehr Komponenten = mehr Betriebs-/Backup-Aufwand (in Phase 2 einplanen)
