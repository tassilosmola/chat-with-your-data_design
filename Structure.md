
# Beschreibung
Um ein Konzept der Lösung zu bekommen, benötige ich ADRs die nachvollziehbar machen, warum etwas wie entschieden wurde. 

> **Ergebnis der Konzeption:**
> - 12 ADRs unter [`docs/adr/`](docs/adr/README.md) (Index mit Entscheidungslogik)
> - Architekturdiagramm (Mermaid, präsentationsfertig) unter [`docs/architecture.md`](docs/architecture.md)

## ADRs
- Abwägen Cloud / OnPrem unter berücksichtigung des Datenschutzes → **ADR-0001: OnPrem**
-  

### Anforderungen
- Hallozinationsarm
    - RAG, MCP
- Skalierbar (10000 MA)
- Datenschutz / Compliance
    - Self hosted oder via Vendor API?
    - Personenbezogene Daten
    - Tracing / US SaaS

### Herausforderungen
- Datenschutz ()
- Datenqualität 
    - RAG CI?
    - RAG Validierung? 
- Rechte (Prompt Cache Gefahr personenbezogene Daten leakage)
- Nachvollziehbarkeit (Model, Umgebung, Daten)
- Skalierung




Ich bin gerade in der Konzeption der unten stehenden Aufbabe. Ziel ist es, durch fragen ADRs zu bekommen, die zu dokumentieren, und am Ende ein Architekturdiagramm des Konzepts zur Präsentation zu bekommen. 

# Aufgabe: „Chat with your Data“
## Zielstellung
Entwickeln Sie einen Lösungsansatz für einen Chatbot, der auf unternehmenseigene Daten zugreift und mithilfe eines modernen LLMs verlässliche, halluzinationsarme Antworten liefert. Dieser Bot soll potenziell allen Mitarbeitern (~10.000 Personen) zur Verfügung stehen.
Der Fokus liegt nicht auf einer vollständigen technischen Umsetzung, sondern auf Ihrem Vorgehen, Ihrer Einschätzung, Architekturüberlegungen und Lösungsstrategie. 

## Rahmenbedingungen
Die Datenquelle(n) sind frei wählbar – z.B. strukturierte Tabellen, PDF-Dokumente, Markdown, FAQs oder eigene Beispieldaten.
Der Bot soll mit einem modernen LLM arbeiten (z.B. OpenAI, HuggingFace, Ollama, Azure GPT, etc.), idealerweise unter Einbeziehung eines
Retrieval-Verfahrens (z.B. ) zur Reduktion von Halluzinationen.RAG
Die Lösung kann als OnPrem- oder Cloud-Ansatz gedacht sein, wichtig ist ein sinnvoller Umgang mit Datenschutz, Skalierung und Zugriff.
Es reicht eine konzeptionelle Lösung – z.B. Architekturdiagramm, Beschreibung, Slides, Notizsammlung oder gerne auch ein kleines Notebook.
## Was wir gerne sehen
Einen klaren, nachvollziehbaren Lösungsansatz: Wie würden Sie vorgehen, wenn Sie die Aufgabe im Team umsetzen sollten?
Technische Überlegungen: Welche Komponenten sind nötig (z.B. Datenvorverarbeitung, Embeddings, Vektordatenbank, Frontend,
Rechtevergabe)?
Welche Tools und Frameworks würden Sie nutzen – und warum?
Scope: Architekturdiagramm, Zeitschätzung, mögliche Risiken, PoC einer technischen Implementierung.

## Was uns besonders interessiert
Wie strukturieren Sie die Aufgabe?
Wo setzen Sie Prioritäten?
Welche Herausforderungen sehen Sie (z.B. Halluzination, Datenqualität, Rechte, Skalierung)?
Welche Use Cases sind aus Ihrer Sicht besonders spannend