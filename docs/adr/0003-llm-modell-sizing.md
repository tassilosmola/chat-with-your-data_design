# ADR-0003: LLM-Modell & GPU-Sizing

- **Status:** open (Empfehlung dokumentiert, finale Entscheidung nach GPU-Budget-Abstimmung mit Infra)
- **Datum:** 2026-10-25
- **Abhängig von:** ADR-0001, ADR-0002

## Kontext

OnPrem-Deployment erfordert ein self-hosted Open-Weight-Modell. GPU-Budget ist derzeit unbekannt. Modellgröße bestimmt Antwortqualität, Hardwarebedarf und max. Durchsatz für 10k potenzielle Nutzer.

### Lastabschätzung (Überschlag, im Lasttest zu validieren)

| Größe | Annahme | Wert |
|---|---|---|
| Gleichzeitig aktive Nutzer (Peak) | 2–5 % von 10k | 200–500 |
| Anfragerate | 1 Frage / ~2 min pro aktivem Nutzer | ~2–4 Requests/s |
| Kontext pro Request (Prefill) | System-Prompt + ~5–8 Chunks + Historie | ~3–5k Tokens |
| Antwortlänge (Decode) | | ~200–400 Tokens |
| **Benötigter Durchsatz** | | **~10–20k Prefill-Tok/s, ~1–1,5k Decode-Tok/s** |

Erkenntnis: Bei RAG dominiert der **Prefill** (langer Kontext), nicht die Antwortgenerierung. Prefix-Caching in vLLM (gemeinsamer System-Prompt) und eine begrenzte Chunk-Anzahl sind daher die günstigsten Hebel.

## Optionen

| Option | Modell-Klasse (aktuelle Open-Weight-Generation, z.B. Qwen, Llama, Mistral, gpt-oss) | Hardware | Qualität | Durchsatz |
|---|---|---|---|---|
| Small (~7–14B) | dichte Modelle | 1× GPU (24–48 GB) | Solide für RAG-Q&A, schwächer bei komplexem Reasoning | Hoch |
| Mid (~70B) FP16/FP8 | dichte Modelle | 4–8× GPU 80 GB | Nahe an proprietären Modellen | Mittel |
| Mid quantisiert | ~70B in 4-bit (AWQ/GPTQ) | 2× GPU 80 GB | Geringer Qualitätsverlust (per Eval zu prüfen) | Mittel-hoch |
| MoE | Mixture-of-Experts, wenige aktive Parameter | 1–4× GPU 80 GB | Gute Qualität bei hohem Durchsatz pro GPU | Hoch |

## Entscheidung (vorläufig)

**PoC: ~70B-Klasse in 4-bit (oder vergleichbares MoE-Modell) auf 2× GPU (80 GB).** Begründung:
- Deutlich bessere Antwortqualität als die Small-Klasse, besonders bei deutschen Fachtexten (Tarif-/Vertragssprache) und Instruction-Following („antworte nur aus dem Kontext“)
- 2 GPUs sind realistisches PoC-Budget; Produktiv-Sizing folgt aus Lasttest

**Produktiv-Sizing-Kriterien** (statt fester Zahl):
- p95 Time-to-First-Token < ~2 s bei Ziel-Concurrency
- LLM-Serving via vLLM mit Continuous Batching + Prefix-Caching
- Skalierung horizontal über GPU-Replicas, gesteuert über Queue-Länge (ADR-0014)

## Betrachtete Alternativen

- Small-Klasse: günstiger, aber für 10k-MA-Rollout mit Qualitätsanspruch riskant ("der Bot liegt oft daneben" killt Adoption schneller als Kosten)
- Managed API trotz OnPrem: widerspricht ADR-0001, bleibt als Gateway-Route konfigurierbar

## Konsequenzen

- Sizing-Workshop mit Infra (verfügbare GPUs, Netzwerk, Strom/Kühlung) als nächster Schritt
- Gateway (ADR-0002) macht Modellwechsel nach Lasttest konfigurativ
- Quantisierung als Standard → Eval (ADR-0012) vergleicht quantisiert vs. unquantisiert auf dem Golden Dataset
