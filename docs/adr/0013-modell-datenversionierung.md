# ADR-0013: Modell- & Datenversionierung – leichtgewichtig über Manifest, Collections und Git

- **Status:** proposed (ersetzt frühere Fassung „DVC + MLflow“)
- **Datum:** 2026-10-25
- **Abhängig von:** ADR-0006 (Qdrant), ADR-0009 (Embeddings), ADR-0011 (Langfuse), ADR-0012 (Eval), ADR-0016 (PII-Sanitizing)
- **Adressiert Anforderung:** Nachvollziehbarkeit („Modell, Umgebung, Daten") + Reproduzierbarkeit

## Kontext

Eine Antwort des Bots hängt von mehreren versionierbaren Artefakten ab:

- **Daten:** Dokument-Corpus, Chunking-Parameter, Qdrant-Index-Stand
- **Modelle:** Chat-LLM-Revision, Embedding-Modell-Revision (Embedding-Wechsel = neuer Index, ADR-0009)
- **Code/Config:** Prompt-Templates, Retrieval-Parameter (top_k, Schwellwert), Pipeline-Version

Für Halluzinations-Analysen, Eval-Regressionen und Rollbacks muss beantwortbar sein: *„Welche Datenversion + welches Modell hat diese Antwort erzeugt?"*

Wichtig für die Werkzeugwahl: **Wir trainieren keine eigenen Modelle.** Modelle sind gepinnte Open-Weight-Revisionen; die Quellsysteme bleiben die Wahrheit über den Corpus.

## Entscheidung

**Vier leichtgewichtige Bausteine statt zusätzlicher Plattformen:**

| Artefakt | Versionierung | Wo |
|---|---|---|
| Corpus / Chunks | **Ingestion-Manifest** je Lauf (append-only): `doc_id`, Quelle, `source_hash`, `sanitizer_version`, `sanitized_hash`, PII-Findings ohne Klartext, Freigabe-Status, ACL, Chunk-IDs + Chunk-Hashes, Pipeline-Version | Object-Storage (S3/MinIO) bzw. Postgres |
| Index | **Versionierte Qdrant-Collections** (`rag_v1`, `rag_v2`, …) + Alias `rag_current` | Qdrant |
| Modelle | **Gepinnte HF-Revision (Commit-Hash)** in der Gateway-/Serving-Config; Gewichte im internen Mirror | Git (Config-Repo) |
| Prompts & Retrieval-Parameter | Prompt-Versionen in Langfuse bzw. Git; Parameter in der Config | Langfuse / Git |

- **Jeder Trace** trägt `model_rev`, `embedding_rev`, `index_version`, `prompt_version` (ADR-0011)
- **Jeder Eval-Lauf** (ADR-0012) ist an dieselben Versionen geknüpft
- **Rollback:** Alias `rag_current` auf vorherige Collection umhängen bzw. Config-Commit revertieren – Sekunden, kein Rebuild
- **Inkrementelle Ingestion:** Content-Hash im Manifest erkennt neue, geänderte und gelöschte Dokumente (ADR-0004)
- **Lineage bei PII-Sanitizing (ADR-0016):** Das Original wird nie verändert. Die Kette `source_hash → (sanitizer_version) → sanitized_hash → chunk_hash` belegt sowohl die verarbeitete Quellversion als auch den tatsächlich indexierten Inhalt. Der Sanitizer ist deterministisch → im Audit nachrechenbar. Re-Ingest bei geändertem `source_hash` **oder** neuer `sanitizer_version` (Policy-Änderung = Rebuild in neue Collection + Alias-Switch)

## Betrachtete Alternativen

| Alternative | Warum verworfen |
|---|---|
| **DVC + MLflow** (frühere Fassung) | Zwei zusätzliche Plattformen (MLflow mit Postgres + Artifact-Store, DVC-Remote) für Probleme, die wir nicht haben: keine eigenen Trainingsläufe, Corpus-Wahrheit liegt in den Quellsystemen. Eval-Tracking deckt Langfuse ab. |
| Nur MLflow | Registry für gepinnte Fremdmodelle bringt gegenüber einem Git-Commit keinen Mehrwert |
| lakeFS | Zielt auf Data-Lake-Branches, nicht auf RAG-Pipelines |

**Auslöser für Neubewertung (Roadmap):** Eigenes Fine-Tuning (LLM, Embedding oder Reranker) oder regulatorische Pflicht zur bitgenauen Corpus-Reproduktion → dann MLflow (Registry/Experimente) bzw. DVC (Corpus-Snapshots) ergänzen.

## Konsequenzen

**Positiv**
- Vollständige Nachvollziehbarkeit pro Antwort ohne zusätzliche Plattform
- Schneller Rollback über Collection-Alias
- Eval-Regressionen an Versionen messbar, nicht nur zeitlich

**Negativ / Pflichten**
- Disziplin: Versions-Felder in Traces und Eval-Läufen sind Pflichtfelder (per Code erzwingen)
- Corpus-Stand ist über Manifest + Hashes nachvollziehbar, aber nicht bitgenau rekonstruierbar, wenn Quellsysteme Dokumente löschen → für v1 akzeptiert
- Alte Collections nach Retention (z.B. 2 Versionen) aufräumen
