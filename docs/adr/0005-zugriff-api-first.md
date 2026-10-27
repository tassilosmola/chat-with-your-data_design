# ADR-0005: Zugriffskanal – API-first

- **Status:** proposed
- **Datum:** 2026-10-25

## Kontext

~10.000 Mitarbeitende sollen den Bot potenziell nutzen. Kanaloptionen (Teams-Bot, Web-App, Slack) haben unterschiedliche UI-Kosten, ändern aber nichts an der Kernfunktionalität. Für das Konzept ist die Backend-Qualität entscheidend, nicht das Frontend.

## Entscheidung

**API-first:** Eine REST-API (FastAPI, OpenAPI-Spec) ist das eigentliche Produkt:

- `POST /chat` – Chat-Anfrage mit Session-Kontext, Antwort inkl. Citations (Streaming via SSE)
- `GET /chat/{id}` – Verlauf/Nachvollziehbarkeit
- `POST /feedback` – Thumbs/Kommentar (speist Eval, ADR-0012)

**Auth:**
- OIDC/JWT via Entra ID; **die Chat-API validiert das Token** (Signatur, Audience, Ablauf) – der Ingress terminiert nur TLS. Eine Prüfstelle, kein doppeltes Auth-Setup.
- User-Identität + Gruppen-Object-IDs aus dem Token → ACL-Filter (ADR-0007)
- **Group-Overage:** Ab ~200 Gruppen liefert Entra keinen `groups`-Claim mehr. Dann holt die API die Gruppen einmalig über Microsoft Graph (`/me/transitiveMemberOf`) und cached sie **pro User** kurzzeitig (z.B. 10 min). Alternativ: Token auf App-zugewiesene Gruppen beschränken.

**Persistenz:** Chat-Verläufe und Feedback liegen in **Postgres** (bestehende Plattform oder eigene Instanz) – mit definierter Retention (ADR-0011).

Ein minimales Referenz-Frontend (einfache Web-UI) demonstriert den Flow im PoC, ist aber explizit austauschbar.

## Betrachtete Alternativen

| Alternative | Warum verworfen |
|---|---|
| MS Teams Bot als Primärkanal | Sinnvoll für Rollout, aber Teams-SDK/Bot-Framework-Details verdecken im PoC die eigentliche Architekturfrage |
| Eigene vollwertige Web-App | Doppelter Scope; UI-Entscheidungen lenken von Kernarchitektur ab |
| OIDC-Validierung im Ingress | Ingress kennt keine Gruppen-Semantik/Overage; zwei Auth-Stellen erhöhen Fehlerrisiko |

## Konsequenzen

- Kanalentscheidung (Teams vs. Web vs. beides) wird zu einer **späteren, reversiblen** Frontend-Entscheidung
- API-Vertrag enthält Citations, Feedback und Auth von Anfang an → kein nachträgliches Design-Risiko
- Postgres als zusätzlicher (meist bereits vorhandener) Datenspeicher; Chat-Verläufe sind personenbezogen → Retention + Löschkonzept
