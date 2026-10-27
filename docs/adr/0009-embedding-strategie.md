# ADR-0009: Embedding-Strategie – self-hosted multilingual, via Gateway

- **Status:** proposed
- **Datum:** 2026-10-25
- **Abhängig von:** ADR-0001 (OnPrem), ADR-0002 (Gateway), ADR-0006 (Qdrant)

## Kontext

Unternehmensdokumente sind überwiegend deutsch/teils englisch → multilingual fähiges Embedding-Modell nötig. Mit der OnPrem-Entscheidung dürfen Dokumentinhalte auch beim Embedding nicht an externe APIs gehen.

## Entscheidung

**Self-hosted multilingual Embedding-Modell** (Kandidaten: `BAAI/bge-m3`, `intfloat/multilingual-e5-large`) als eigener Inferenz-Service, **hinter demselben LLM-Gateway** wie das Chat-Modell:

- Einheitliche OpenAI-kompatible `/embeddings`-Schnittstelle
- Modell konfigurativ wechselbar (ADR-0002)
- bge-m3 liefert zusätzlich Sparse-Vektoren → Hybrid-Search in Qdrant ohne neue Infrastruktur (ADR-0006)

## Betrachtete Alternativen

| Alternative | Warum verworfen |
|---|---|
| OpenAI/Azure Embedding-API | Dokumentinhalte verlassen das Haus → widerspricht ADR-0001/Datenschutz |
| Direkt in Ingestion-Pipeline eingebunden (kein Service) | Query-Embedding zur Laufzeit braucht sowieso einen Service → einmalig hinter Gateway |

## Konsequenzen

- **Index-Versionierung Pflicht:** Embedding-Modellwechsel = neue Qdrant-Collection + Re-Indexierung, alte Version parallel für A/B und Rollback (ADR-0013)
- Eval (ADR-0012) misst Retrieval-Qualität (Context-Recall) auf deutschen Fachtexten, nicht nur das Chat-LLM
- GPU-Bedarf klein (CPU möglich, GPU empfohlen für Ingestion-Durchsatz)
- Ingestion-Batches laufen über ein eigenes Gateway-Kontingent, damit sie Live-Queries nicht verdrängen
