# ADR-0003: LLM-Modell & GPU-Sizing

- **Status:** open (Empfehlung dokumentiert, finale Entscheidung nach GPU-Budget-Abstimmung mit Infra)
- **Datum:** 2026-10-24
- **Abhängig von:** ADR-0001, ADR-0002

## Kontext

OnPrem-Deployment erfordert ein self-hosted Open-Source-Modell. GPU-Budget ist derzeit unbekannt. Modellgröße bestimmt Antwortqualität, Hardwarebedarf und max. Durchsatz für 10k potenzielle Nutzer.

Annahme für Lastprofil: 10k MA ≠ 10k gleichzeitige Nutzer. Realistisch ~2–5 % concurrent (200–500 Sessions), davon Bruchteil aktive Inferenz → O(10–100) parallele Generierungen in Spitzen.

## Optionen

| Option | Modell-Beispiele | Hardware | Qualität | Durchsatz |
|---|---|---|---|---|
| Small (~7–9B) | Llama-3.1-8B, Mistral-7B | 1–2× GPU (24–48 GB) | Solide für RAG-Q&A, schwächer bei komplexem Reasoning | Hoch |
| Mid (~70B) FP16 | Llama-3.3-70B, Qwen-2.5-72B | 4–8× A100/H100 | Nahe an proprietären Modellen | Mittel |
| Mid quantisiert | 70B in 4-bit (GPTQ/AWQ) | 2× GPU (80 GB) | Kaum messbarer Qualitätsverlust | Mittel-hoch |
| MoE | Mixtral-8x22B, Qwen3-MoE | 2–4× GPU | Gute Qualität je aktiviertem Experten | Hoch pro GPU |

## Entscheidung (vorläufig)

**PoC: ~70B-Klasse in 4-bit Quantisierung auf 2× GPU (80 GB).** Begründung:
- Deutlich bessere Antwortqualität als 7–8B, besonders bei deutschen Fachtexten und Instruction-Following
- Quantisierung halbiert/viertelt VRAM-Bedarf bei minimalem Qualitätsverlust
- 2 GPUs sind realistisches PoC-Budget; Produktiv-Sizing folgt aus Lasttest

**Produktiv-Sizing-Kriterien** (im ADR festhalten statt Zahl festzulegen):
- p95-Token-Latenz < ~2 s TTFT bei Ziel-Concurrency
- LLM-Serving via vLLM mit Continuous Batching + Queue-Backpressure
- Skalierung horizontal über GPU-Nodes, HPA auf Queue-Länge

## Betrachtete Alternativen

- 8B-Klasse: günstiger, aber für 10k-MA-Rollout mit Qualitätsanspruch riskant ("der Bot liegt oft daneben" killt Adoption schneller als Kosten)
- Managed API trotz OnPrem-Rest: widerspricht ADR-0001, bleibt als Gateway-Fallback konfigurierbar

## Konsequenzen

- Sizing-Workshop mit Infra (verfügbare GPUs, Netzwerk, Strom/Kühlung) als nächster Schritt
- Gateway (ADR-0002) macht Modellwechsel nach Lasttest rein konfigurativ
- Quantisierung als Standard → Modell-Eval (ADR-0012) muss FP16 vs. Q4 vergleichen
