# ADR-0001: Deployment-Modell – OnPrem auf Kubernetes

- **Status:** accepted
- **Datum:** 2026-10-24
- **Kontext:** ~10.000 Mitarbeitende, unternehmenseigene teils personenbezogene Daten, Datenschutz/Compliance als Rahmenbedingung

## Kontext

Der Bot verarbeitet sensible Unternehmensdaten und ggf. personenbezogene Daten in Prompts und Antworten. Die Wahl des Deployment-Modells bestimmt Datenhoheit, Betriebsaufwand und Time-to-Market.

Optionen:
- **Azure Cloud (EU-Region):** Managed Services (Azure OpenAI, AI Search), EU Data Boundary, schneller Start. Aber: Vertrauen in US-Hyperscaler nötig, Tracing/Telemetrie teilweise außerhalb der eigenen Kontrolle, laufende Token-Kosten bei 10k Nutzern.
- **OnPrem (Kubernetes):** Volle Datenhoheit, keine Datenabflüsse, aber GPU-Betrieb und Modell-Hosting in Eigenverantwortung.
- **Hybrid:** Daten/Embeddings onPrem, nur Prompts zum Cloud-LLM. Kompromiss, aber Personenbezug in Prompts schwer garantiert auszuschließen.

## Entscheidung

**OnPrem auf Kubernetes.** Alle Komponenten (LLM-Serving, Vektor-DB, Ingestion, Observability) laufen im eigenen Cluster. Bestehende K8s-Infrastruktur wird vorausgesetzt; GPU-Nodes werden als neue Node-Pool-Kategorie ergänzt.

## Betrachtete Alternativen

| Alternative | Warum verworfen |
|---|---|
| Azure Cloud (EU) | Datenhoheit + Compliance-Bedenken bei personenbezogenen Daten trotz EU Boundary; Vendor-Abhängigkeit |
| Hybrid | Komplexe Datenflusskontrolle (Prompts können Personenbezug enthalten); zwei Betriebsmodelle |
| Reine Managed-SaaS-Kette | Widerspricht Datenschutzanforderung fundamental |

## Konsequenzen

**Positiv**
- Volle Kontrolle über Datenflüsse – kein Drittlandtransfer, keine Vendor-Telemetrie
- Keine laufenden Token-/API-Kosten → planbare Kosten bei 10k Nutzern
- Skalierung über K8s (HPA auf GPU-Nodes, Queue-basiertes Scaling für LLM-Requests)

**Negativ / Risiken**
- GPU-Hardware muss beschafft und betrieben werden (Kapazitätsplanung → ADR-0003)
- Modell-Qualität liegt in eigener Verantwortung; kein "einfach GPT-4o anbinden"
- Höherer initialer Betriebsaufwand (Serving-Stack, Monitoring, Updates)

**Mitigation**
- LLM-Gateway (ADR-0002) hält Tür offen, später cloudbasierte Modelle für unkritische Workloads anzubinden
- PoC-Phase mit kleiner GPU-Ausstattung validieren, dann skalieren
