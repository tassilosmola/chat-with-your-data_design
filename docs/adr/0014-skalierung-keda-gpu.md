# ADR-0014: Skalierung – KEDA auf Request-Queue für GPU-Workloads

- **Status:** proposed
- **Datum:** 2026-10-25
- **Abhängig von:** ADR-0001 (K8s onPrem), ADR-0002 (Backpressure am Gateway), ADR-0003 (LLM-Sizing/GPUs)

## Kontext

~10.000 Mitarbeitende mit stark bursty Nutzung (Morgens-Peak, Mittags, Feierabend). GPUs sind die knappe, teure Ressource. Klassischer HPA auf CPU/GPU-Utilization ist für LLM-Serving das falsche Signal:

- GPU-Auslastung korreliert schlecht mit Nutzerwartezeit – entscheidend ist die **Warteschlange anstehender Requests**
- vLLM mit Continuous Batching sättigt eine GPU weit, bevor "Utilization" aussagekräftig wird
- Ein zusätzlicher GPU-Pod mit Minuten-Kaltstart (Modell laden) muss **vor** der Lastspitze starten, nicht danach

**OnPrem-Realität:** GPUs sind gekaufte Hardware in einem **festen GPU-Nodepool**. Scale-down spart kein Geld wie in der Cloud – der Nutzen liegt darin, **freie GPU-Kapazität anderen Workloads** zu geben (Eval-Läufe, Re-Indexierung/Embedding-Batches, andere ML-Teams im geteilten Pool) und Lastspitzen innerhalb des Pools abzufangen.

## Entscheidung

**KEDA (Kubernetes Event-driven Autoscaling) mit Queue-Länge als Skalierungssignal:**

- **vLLM-Deployment:** `ScaledObject` mit **Prometheus-Scaler** auf `vllm:num_requests_waiting` → Replicas skalieren anhand der Warteschlange, nicht der Utilization
- **Grenzen:** `minReplicaCount` = Basislast aus Lasttest (Warm-Pool, kein Kaltstart zu Geschäftszeiten); `maxReplicaCount` = Kapazität des GPU-Pools
- **Außerhalb der Geschäftszeiten** (KEDA `Cron`-Trigger): auf Minimum reduzieren → frei werdende GPUs für Eval- und Ingestion-Batches. Scale-to-Zero nur, wenn ein Kaltstart am Morgen per Cron **vorgewärmt** wird.
- **Über der Pool-Grenze** skaliert nichts mehr – dann greift **Backpressure am Gateway** (Queue-Limit, Timeout, `429` + Retry-After, ADR-0002). Lieber "bitte kurz warten" als Timeouts für alle.
- **Kaltstart verkürzen:** Modellgewichte aus internem Mirror auf Node-lokalen Cache (hostPath/PVC) vorladen
- **Node-Ebene:** OnPrem i.d.R. **kein** Cluster-Autoscaler für Bare-Metal-GPU-Nodes → fester Pool, Erweiterung per Kapazitätsplanung. In einer Private Cloud mit GPU-Provisionierung kann ein Cluster-Autoscaler ergänzt werden.
- **Übrige Services** (Chat-API, Gateway, Ingestion-Worker): Standard-HPA bzw. KEDA auf Queue-Tiefe der Ingestion

**PoC-Hinweis:** Mit dem kleinen MoE-Modell (ADR-0003) ist 1× GPU = 1 vLLM-Replica – schon 2 PoC-GPUs erlauben eine 1↔2-Replica-Skalierung. Zusätzlich kann eine GPU per **MIG** (GPU Operator, `mixed`-Strategie) zwischen Serving und Batch-Jobs partitioniert werden. Im PoC werden **Metrik, ScaledObject und Backpressure** aufgebaut und im Lasttest verifiziert; wirksam wird die Skalierung mit dem Produktiv-Pool.

## Betrachtete Alternativen

| Alternative | Warum verworfen |
|---|---|
| Klassischer HPA (CPU/Memory/GPU-Util) | Falsches Signal für GPU-Inferenz (s.o.); skaliert zu spät oder gar nicht |
| Statische Replica-Zahl | Einfachste Lösung und für den PoC ausreichend – aber GPUs sind dauerhaft gebunden, keine Umverteilung an Batch-Workloads, kein Puffer für Peaks |
| Knative/KServe Autoscaling | Eigener Serving-Stack, schwerer; KEDA ist additiv und schlanker |
| Custom Controller | Rad neu erfinden; KEDA deckt Prometheus/Queue/Cron ab |

## Konsequenzen

**Positiv**
- Skalierung am eigentlichen Engpass (Request-Queue) → vorhersagbares Verhalten unter Lastspitzen
- GPU-Pool wird zwischen Live-Serving und Batch-Workloads (Eval, Re-Indexierung) geteilt – zeitlich (Cron) oder, weil das Modell MIG-fähig ist (ADR-0003), **räumlich** als feste Slices mit Hardware-Isolation
- KEDA additiv, kein Vendor-Lock-in

**Negativ / Risiken**
- **Kaltstart-Latenz** bleibt physikalisch (Minuten) → Warm-Pool über `minReplicaCount`, Threshold konservativ (skaliert früh), Cron-Vorwärmen
- Skalierung endet an der Pool-Grenze → Backpressure-Design am Gateway ist Pflicht
- Schwellwerte (Queue-Länge pro Replica) nur per Lasttest bestimmbar (Phase 3)
