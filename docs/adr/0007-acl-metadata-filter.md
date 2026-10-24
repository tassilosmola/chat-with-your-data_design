# ADR-0007: Dokumentberechtigungen – Metadaten-Filter beim Retrieval

- **Status:** accepted
- **Datum:** 2026-10-24
- **Abhängig von:** ADR-0006 (Qdrant), ADR-0005 (Entra ID Auth)
- **Risiko-Adresse:** Prompt-Cache-Leakage personenbezogener Daten

## Kontext

Dokumente haben unterschiedliche Berechtigungsstufen. Der Bot darf Inhalte nur an Nutzer ausgeben, die die Quelle sehen dürfen – sonst wird RAG zur eleganten Datenleckage-Maschine. Die Prüfung muss **vor** dem LLM-Kontext erfolgen, nicht erst in der Antwort.

## Entscheidung

**Document-Level Security via Payload-Metadaten in Qdrant:**

- Bei der Ingestion speichert jeder Chunk `acl_groups: [AD-Gruppen/Entra-IDs]` + `sensitivity` als Payload
- Zur Query-Zeit werden die AD-Gruppen des Nutzers (aus dem OIDC-Token) als **serverseitiger Qdrant-Filter** gesetzt – unberechtigte Chunks verlassen die Vektor-DB gar nicht erst
- Citations in der Antwort verweisen auf Dokumente, die der Nutzer öffnen darf

Ergänzende Maßnahmen gegen den Prompt-Cache-Leakage-Pfad:
- Response-/Prompt-Caching am Gateway mit **user-scoped Cache-Keys** oder deaktiviert für ACL-gefilterte Anfragen (ADR-0002)
- Kein Sharing von LLM-Kontext-Chunks zwischen Sessions unterschiedlicher Nutzer

## Betrachtete Alternativen

| Alternative | Warum verworfen |
|---|---|
| Separate Collections pro Bereich | Keine bereichsübergreifende Suche möglich; Rechte-Änderung = Re-Indexing; Collection-Explosion |
| Post-Retrieval-Check | Treffer werden erst nach dem Retrieval geprüft → Top-k kann mit unberechtigten Treffern "verstopft" sein; Info-Leak über Ranking möglich |
| Zugriffskontrolle nur im Prompt ("sag nichts über X") | Kein echtes Access Control – Prompt-Injection umgehbar |

## Konsequenzen

**Pflichten**
- **AD/Entra-Gruppen-Sync in die Ingestion-Pipeline** (Quelle der Wahrheit: AD + Dokumenten-ACLs). Rechte-Änderungen am Dokument müssen den Index aktualisieren (Inkrement-Sync) → RTO für Rechte-Entzug definieren (z.B. < 15 min)
- Filter-Evaluation ist Teil jeder Query → Payload-Index auf `acl_groups` pflegen
- Audit-Log: welcher Nutzer bekam welche Chunks als Kontext (ADR-0011)

**Restrisiko**
- Rechte-Drift: Quellsystem-ACLs und Index-ACLs divergieren bei Sync-Lag → Monitoring + max. Sync-Latenz als Betriebs-SLO
