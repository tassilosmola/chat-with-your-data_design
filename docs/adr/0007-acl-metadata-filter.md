# ADR-0007: Dokumentberechtigungen – Metadaten-Filter beim Retrieval

- **Status:** proposed
- **Datum:** 2026-10-25
- **Abhängig von:** ADR-0006 (Qdrant), ADR-0005 (Entra ID Auth)

## Kontext

Dokumente haben unterschiedliche Berechtigungsstufen. Der Bot darf Inhalte nur an Nutzer ausgeben, die die Quelle sehen dürfen – sonst wird RAG zur eleganten Datenleckage-Maschine. Die Prüfung muss **vor** dem LLM-Kontext erfolgen, nicht erst in der Antwort.

## Entscheidung

**Document-Level Security via Payload-Metadaten in Qdrant:**

- **Ingestion:** Jeder Chunk erhält `acl_groups: [Entra-Group-Object-IDs]` + `sensitivity` aus den ACLs des Quelldokuments
- **Query-Zeit:** Die Gruppen des Nutzers (Token bzw. Graph bei Overage, ADR-0005) werden als **serverseitiger Qdrant-Filter** gesetzt – unberechtigte Chunks verlassen die Vektor-DB nicht
- Citations verweisen nur auf Dokumente, die der Nutzer öffnen darf

**Vereinfachung: kein Gruppen-Mitgliedschafts-Sync.** Weil Chunks Gruppen-*IDs* speichern und die Mitgliedschaft des Users zur Query-Zeit aus Entra kommt, wirken Änderungen an Gruppenmitgliedschaften sofort (spätestens nach Token-/Cache-Ablauf). Synchronisiert werden müssen nur die **Dokument-ACLs** (welche Gruppe darf welches Dokument sehen).

**Woher kommen die Dokument-ACLs?**
- v1 (PoC-Daten): Tarif-, Mantel-, Arbeitszeit- und Reisekostenregelungen sind für **alle Mitarbeitenden** freigegeben → eine Gruppe „alle MA“. Der Mechanismus ist trotzdem vollständig gebaut und getestet.
- Später (SharePoint/Confluence): ACLs werden vom Konnektor aus dem Quellsystem gelesen und auf Entra-Group-IDs gemappt.

**Kein Response-Caching** in v1 (ADR-0002) – damit entfällt der Cache-Leakage-Pfad vollständig. Kein Sharing von Kontext-Chunks zwischen Sessions.

## Betrachtete Alternativen

| Alternative | Warum verworfen |
|---|---|
| Separate Collections pro Bereich | Keine bereichsübergreifende Suche; Rechte-Änderung = Re-Indexing; Collection-Explosion |
| Post-Retrieval-Check | Top-k kann mit unberechtigten Treffern "verstopft" sein; Info-Leak über Ranking möglich |
| Zugriffskontrolle nur im Prompt ("sag nichts über X") | Kein echtes Access Control – per Prompt-Injection umgehbar |
| Sync der Gruppenmitgliedschaften in die Ingestion | Unnötig bei ID-basierten ACLs; erzeugt Sync-Lag bei Rechte-Entzug |

## Konsequenzen

**Pflichten**
- Inkrementeller Sync der **Dokument-ACLs** → max. Latenz für Rechte-Entzug am Dokument als SLO (z.B. < 15 min)
- Payload-Index auf `acl_groups`
- Audit-Log: welcher Nutzer bekam welche Chunks als Kontext (ADR-0011)
- **Deterministische ACL-Tests** in CI: Testnutzer ohne Berechtigung darf definierte Chunks nie erhalten (ADR-0012)

**Restrisiko**
- Rechte-Drift: Quellsystem-ACLs und Index divergieren bei Sync-Lag → Monitoring der Sync-Latenz
