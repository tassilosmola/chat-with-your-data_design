# ADR-0002: LLM-Gateway als Abstraktionsschicht

- **Status:** proposed
- **Datum:** 2026-10-25
- **Abhängig von:** ADR-0001 (OnPrem)

## Kontext

Bei ~10.000 potenziellen Nutzern ist die GPU die knappe Ressource. Ohne zentrale Steuerung kann ein einzelner Nutzer/Client (Skript, Massenabfrage) die Kapazität für alle blockieren. Zusätzlich entwickelt sich das LLM-Feld schnell – heute optimale Open-Source-Modelle sind in 6–12 Monaten überholt.

Hinweis: Reiner **Modellwechsel** wäre auch ohne Gateway möglich – vLLM ist OpenAI-kompatibel, ein konfigurierbares `base_url` genügt. Das Gateway wird daher **nicht primär für Wechselbarkeit** eingeführt, sondern für Fairness und Lastschutz.

## Entscheidung

**Ein schlankes LLM-Gateway (LiteLLM Proxy) als einheitliche Schnittstelle** zwischen Chat-API/Ingestion und allen Modell-Endpunkten (Chat-LLM + Embeddings).

Das Gateway kapselt in v1 genau vier Aufgaben:
1. **Quotas & Rate-Limits pro User/Client** (Fairness bei 10k MA, Schutz vor Missbrauch)
2. **Backpressure:** Queue-Limit + Timeout + saubere `429`-Antworten (Voraussetzung für ADR-0014)
3. **Routing** auf Modell-Endpunkte (primär self-hosted vLLM), Modellwahl per Config
4. **Request-Logging** an Langfuse (ADR-0011)

**Bewusst nicht in v1:** Response-/Prompt-Caching. RAG-Antworten hängen vom ACL-gefilterten Kontext des jeweiligen Nutzers ab – ein Cache bringt wenig Trefferquote, aber ein reales Leakage-Risiko über Nutzergrenzen (ADR-0007).

## Betrachtete Alternativen

| Alternative | Warum verworfen |
|---|---|
| Direkte vLLM-Anbindung (OpenAI-SDK + `base_url`) | Modellwechsel ginge auch so – aber keine nutzerbezogenen Quotas und kein zentraler Backpressure-Punkt |
| Nur Framework-Abstraktion (LangChain) | Deckt Code-Ebene ab, nicht Limits/Backpressure zur Laufzeit; zusätzlich Framework-Lock-in |
| Kong AI Gateway | Funktional ähnlich, aber schwerer; lohnt nur, wenn Kong ohnehin im Haus ist |

## Konsequenzen

**Positiv**
- Faire Verteilung der GPU-Kapazität, Schutz vor Lastspitzen einzelner Clients
- Einheitliches Interface auch für Embeddings (ADR-0009)
- Modellwechsel nach Lasttest/Eval konfigurativ (ADR-0003)

**Negativ**
- Zusätzliche Komponente im kritischen Pfad → stateless, mind. 2 Replicas
- Latenz-Overhead (gering, ~ms-Bereich)
