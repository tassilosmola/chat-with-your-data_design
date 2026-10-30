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
| Mid quantisiert | ~70B in 4-bit (AWQ/GPTQ) | 1–2× GPU 80 GB | Geringer Qualitätsverlust (per Eval zu prüfen) | Mittel-hoch |
| MoE klein (~20–30B gesamt, ~3B aktiv) | z.B. Qwen3-30B-A3B, gpt-oss-20b | 1× GPU (24–48 GB), **MIG-Slice-fähig** | Gut für RAG-Q&A, begrenztes Reasoning | Sehr hoch (v.a. Prefill) |
| MoE groß (≥100B, wenige aktiv) | z.B. gpt-oss-120b | 1–2× GPU 80 GB | Deutlich besser, höherer Durchsatz pro GPU als dicht | Hoch |

## Entscheidung (vorläufig)

**PoC: kleine MoE-Klasse (~20–30B gesamt, ~3B aktive Parameter) in 4-bit auf 1× GPU (24–48 GB).** Begründung:

- Die Last ist **Prefill-dominiert** (s. Lastabschätzung) – MoE mit wenigen aktiven Parametern liefert genau dort den höchsten Durchsatz pro GPU
- VRAM-Footprint (~20 GB quantisiert) macht das Replica billig: **1 GPU statt 2**, kein Tensor-Parallelism, einfacheres Deployment (KISS)
- Das Modell passt in einen **MIG-Slice** (z.B. `2g.20gb`/`3g.40gb` auf 80-GB-Karten, max. 7 Instanzen/GPU via GPU Operator) → der GPU-Pool kann zwischen Serving und Batch-Workloads (Eval, Re-Indexierung) **partitioniert** werden statt nur zeitlich umgeschichtet (ADR-0014)
- Qualitätsrisiko bei deutschen Fachtexten (Tarif-/Vertragssprache, Instruction-Following) wird **eval-gated** (ADR-0012): Das MoE muss das Golden-Dataset-Gate bestehen – sonst Upgrade-Pfad auf die 70B-Klasse konfigurativ über das Gateway (ADR-0002)

**Produktiv-Sizing-Kriterien** (statt fester Zahl):
- p95 Time-to-First-Token < ~2 s bei Ziel-Concurrency
- LLM-Serving via vLLM mit Continuous Batching + Prefix-Caching
- Skalierung horizontal über GPU-Replicas (1× GPU bzw. 1 Slice pro Replica), gesteuert über Queue-Länge (ADR-0014)

## Betrachtete Alternativen

- **70B-Klasse in 4-bit auf 2× GPU (80 GB):** bessere Qualität, aber doppelter VRAM-Preis pro Replica, Tensor-Parallelism nötig und nicht MIG-fähig – bleibt **Upgrade-Pfad**, falls das MoE das Eval-Gate nicht besteht oder der Lasttest höhere Qualitätsanforderungen zeigt
- Small-Klasse (dicht): ähnlicher Footprint wie das kleine MoE, aber bei gleicher Hardware schwächer in Qualität und Prefill-Durchsatz
- Managed API trotz OnPrem: widerspricht ADR-0001, bleibt als Gateway-Route konfigurierbar

## Konsequenzen

- Sizing-Workshop mit Infra (verfügbare GPUs, Netzwerk, Strom/Kühlung) als nächster Schritt
- Gateway (ADR-0002) macht Modellwechsel nach Lasttest konfigurativ; ein Wechsel MoE → 70B ändert das Pod-Sizing (1× → 2× GPU, Tensor-Parallelism) → Deployment-Templates dafür vorsehen
- **Eval-Vergleich MoE vs. 70B-Klasse** auf dem Golden Dataset (ADR-0012) ist Entscheidungsgate vor Rollout, nicht erst nice-to-have
- Quantisierung als Standard → Eval (ADR-0012) vergleicht quantisiert vs. unquantisiert auf dem Golden Dataset
- MIG-Partitionierung des GPU-Pools via NVIDIA GPU Operator wird in ADR-0014 als Option für geteilte Pools festgehalten
