# ADR-0002: LLM-Gateway als Abstraktionsschicht

- **Status:** accepted
- **Datum:** 2026-10-24
- **Abhängig von:** ADR-0001 (OnPrem)

## Kontext

Das LLM-Feld entwickelt sich schnell – heute optimale Open-Source-Modelle sind in 6–12 Monaten überholt. Eine direkte Anbindung an einen einzelnen Provider/ein Framework erzeugt Vendor- bzw. Modell-Lock-in. Zudem brauchen wir einen zentralen Punkt für Rate-Limiting, Caching, Kosten-/Kapazitätstracking und Prompt-Governance.

## Entscheidung

**Ein LLM-Gateway (z.B. LiteLLM Proxy oder Kong AI Gateway) als einheitliche Schnittstelle** zwischen Chat-Backend und allen Modell-Endpunkten (Chat-LLM + Embeddings).

Das Gateway kapselt:
- Provider-Auswahl und Routing (primär: self-hosted via vLLM)
- Rate-Limits, Retries, Fallbacks
- Request-/Response-Logging an Langfuse (ADR-0011)
- Cache-Keys **user-scoped** (Leakage-Schutz, siehe ADR-0007)

## Betrachtete Alternativen

| Alternative | Warum verworfen |
|---|---|
| Direkte vLLM/SDK-Anbindung in der App | Modellwechsel = Codeänderung; kein zentraler Governance-Punkt |
| Nur Framework-Abstraktion (LangChain) | Deckt Code-Ebene ab, nicht Routing/Limits/Fallbacks zur Laufzeit; zusätzlich Framework-Lock-in |

## Konsequenzen

**Positiv**
- Modellwechsel rein konfigurativ → Sizing-Entscheidung (ADR-0003) reversibel
- Einheitliches Interface auch für Embeddings (ADR-0009)
- Zentraler Punkt für Tracing, Rate-Limits, Audit
- Fallback-Routen möglich (z.B. Sekundärmodell bei Ausfall)

**Negativ**
- Zusätzliche Komponente im kritischen Pfad → muss HA-fähig deployed werden
- Latenz-Overhead (gering, ~ms-Bereich)

**Risiko & Mitigation**
- Prompt-Cache kann personenbezogene Daten über Nutzergrenzen leaken → Cache-Keys strikt pro User bzw. Caching für personalisierte Antworten deaktivieren (konfigurierbar am Gateway).
