# ADR-0014: Skalierung – KEDA für GPU-Workloads

- **Status:** accepted
- **Datum:** 2026-10-24
- **Abhängig von:** ADR-0001 (K8s onPrem), ADR-0003 (LLM-Sizing/GPUs)

## Kontext

~10.000 Mitarbeitende mit stark bursty Nutzung (Morgens-Peak, Mittags, Feierabend). GPUs sind die knappe, teure Ressource. Klassischer HPA auf CPU/GPU-Utilization ist für LLM-Serving das falsche Signal:

- GPU-Auslastung korreliert schlecht mit Nutzerwartezeit – entscheidend ist die **Warteschlange anstehender Requests**
- vLLM mit Continuous Batching sättigt eine GPU weit, bevor "Utilization" aussagekräftig wird
- Ein zusätzlicher GPU-Pod mit Minuten-Kaltstart (Modell laden) muss **vor** der Lastspitze starten, nicht danach

## Entscheidung

**KEDA (Kubernetes Event-driven Autoscaling) als Skalierungsmechanismus:**

- **vLLM-Deployment:** `ScaledObject` mit **Prometheus-Scaler** auf `vllm:num_requests_waiting` (bzw. Pending-Requests am LLM-Gateway, ADR-0002) → Replicas skalieren anhand der Queue-Länge, nicht der Utilization
- **Cooldown/Min-Replikas:** `minReplicaCount ≥ 1` zu Geschäftszeiten (Kaltstart = Modell laden, Minuten). Optional `minReplicaCount: 0` über Nacht/Wochenende via KEDA `Cron`-Trigger – bewusster Trade-off zwischen GPU-Einsparung und Kaltstart-Latenz
- **Node-Ebene:** Cluster-Autoscaler provisioniert GPU-Nodes, sobald Pods pending hängen. Kaltstart-Mitigation: Modellgewichte auf Node-Cache (hostPath/NFS) vorgelegt oder InitContainer mit internem Mirror – sonst addiert sich Node-Provisionierung + Model-Download
- **Restliche Services:** Chat-API, Gateway, Ingestion-Worker ebenfalls KEDA/HPA; Ingestion-Worker skalieren auf Upload-/Queue-Tiefe

## Betrachtete Alternativen

| Alternative | Warum verworfen |
|---|---|
| Klassischer HPA (CPU/Memory) | Falsches Signal für GPU-Inferenz (s.o.); skaliert zu spät oder gar nicht |
| Statische Replica-Zahl | GPUs dauerhaft vorgehalten → teuer bei bursty Last; kein Scale-down |
| Knative/Kserve Autoscaling | Eigener Serving-Stack, schwerer; KEDA ist additiv und schlanker für "reines" Replica-Scaling |
| Custom Controller | Rad neu erfinden; KEDA-Scaler-Ökosystem deckt Prometheus/Queue/Cron ab |

## Konsequenzen

**Positiv**
- Skalierung am eigentlichen Engpass (Request-Queue) → prognostizierbares Verhalten unter Lastspitzen
- Scale-to-Zero-Option außerhalb Geschäftszeiten spart signifikant GPU-Stunden bei nur 8h-Nutzungsfenster
- KEDA additiv, kein Vendor-Lock-in, Standard-Ökosystem

**Negativ / Risiken**
- **Kaltstart-Latenz** bleibt physikalisch: neuer vLLM-Pod braucht Minuten (Model-Load) → Mitigation: Warm-Pool (`minReplicaCount ≥ 1`), Node-Pre-Provisioning, Threshold konservativ setzen (skaliert früh)
- Queue-basiertes Scaling erfordert Backpressure-Design am Gateway (Request-Timeout + Queue-Limit + saubere 429/Retry-Antworten)
- Zwei Skalierungsebenen (Pod via KEDA, Node via Autoscaler) müssen aufeinander abgestimmt werden → Lasttest vor Rollout (Phase 3)
