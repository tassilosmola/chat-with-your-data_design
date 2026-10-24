# ADR-0009: Embedding-Strategie – self-hosted multilingual, via Gateway

- **Status:** accepted
- **Datum:** 2026-10-24
- **Abhängig von:** ADR-0001 (OnPrem), ADR-0002 (Gateway), ADR-0006 (Qdrant)

## Kontext

Unternehmensdokumente sind überwiegend deutsch/teils englisch → multilingual fähiges Embedding-Modell nötig. Mit der OnPrem-Entscheidung dürfen Dokumentinhalte auch beim Embedding nicht an externe APIs gehen.

## Entscheidung

**Self-hosted multilingual Embedding-Modell** (Kandidaten: `BAAI/bge-m3`, `intfloat/multilingual-e5-large`) als eigener Inferenz-Service, **hinter demselben LLM-Gateway** wie das Chat-Modell:

- Einheitliche OpenAI-kompatible `/embeddings`-Schnittstelle
- Provider/Modell konfigurativ wechselbar → konsistente Architektur (ADR-0002)
- bge-m3 unterstützt zusätzlich sparse Vektoren → optionaler Hybrid-Search-Pfad in Qdrant ohne neue Infrastruktur

## Betrachtete Alternativen

| Alternative | Warum verworfen |
|---|---|
| OpenAI/Azure Embedding-API | Dokumentinhalte verlassen das Haus → widerspricht ADR-0001/Datenschutz |
| Direkt in Ingestion-Pipeline eingebunden (kein Service) | Query-Embedding zur Laufzeit braucht sowieso einen Service → besser einmalig hinter Gateway |

## Konsequenzen

- **Index-Versionierung Pflicht:** Embedding-Modellwechsel = Vektordimension/Semantik ändert sich → neue Qdrant-Collection + Re-Indexierung, alte Version parallel für A/B und Rollback
- Eval-Pipeline (ADR-0012) sollte Embedding-Qualität auf deutschen Fachtexten messen, nicht nur Chat-LLM
- GPU-Bedarf für Embedding-Service ist klein (CPU möglich, GPU empfohlen für Ingestion-Durchsatz)
