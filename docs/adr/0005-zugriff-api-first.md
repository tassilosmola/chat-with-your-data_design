# ADR-0005: Zugriffskanal – API-first

- **Status:** accepted
- **Datum:** 2026-10-24

## Kontext

~10.000 Mitarbeitende sollen den Bot potenziell nutzen. Kanaloptionen (Teams-Bot, Web-App, Slack) haben unterschiedliche UI-Kosten, ändern aber nichts an der Kernfunktionalität. Für das Konzept ist die Backend-Qualität entscheidend, nicht das Frontend.

## Entscheidung

**API-first:** Eine REST-API (FastAPI, OpenAPI-Spec) ist das eigentliche Produkt:

- `POST /chat` – Chat-Anfrage mit Session-Kontext, Antwort inkl. Citations
- `GET /chat/{id}` – Verlauf/Nachvollziehbarkeit
- `POST /feedback` – Thumbs/Kommentar (speist Eval, ADR-0012)
- Auth: OIDC/JWT via Entra ID → liefert User-Identität + AD-Gruppen für ACL-Filter (ADR-0007)

Ein minimales Referenz-Frontend (einfache Web-UI oder Teams-Stub) demonstriert den Flow im PoC, ist aber explizit austauschbar.

## Betrachtete Alternativen

| Alternative | Warum verworfen |
|---|---|
| MS Teams Bot als Primärkanal | Sinnvoll für Rollout, aber Teams-SDK/Bot-Framework-Details verdecken im PoC die eigentliche Architekturfrage |
| Eigene vollwertige Web-App | Doppelter Scope; UI-Entscheidungen (Design, Citations-Darstellung, Feedback-UX) lenken von Kernarchitektur ab |

## Konsequenzen

- Kanalentscheidung (Teams vs. Web vs. beides) wird zu einer **späteren, reversiblen** Frontend-Entscheidung – gute Story für die Präsentation
- API-Vertrag muss Citations, Feedback und Auth von Anfang an enthalten → kein nachträgliches Design-Risiko
- SSO über Entra ID wird bereits in der API erzwungen, nicht erst im Frontend
