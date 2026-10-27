# AGENTS.md

## Projekt
Konzept-Repo (keine Implementierung): ADRs unter `docs/adr/`, Überblick `docs/architecture.md`, Roadmap `docs/roadmap.md`, LikeC4-Modell `c4_architecture/architecture.c4`. Sprache: Deutsch. Leitlinie: KISS – v1 minimal (ADR-0015), Erweiterungen nur mit Auslöser in der Roadmap.

## Verifikation
- LikeC4-Modell validieren (benötigt Node ≥ 20; WSL hat Node 18):
  `npx -y -p node@22 -p likec4@1 -- likec4 validate c4_architecture/`
  Falls native Bindings fehlen: likec4 mit npm ≥ 10 in ein Temp-Verzeichnis installieren und dessen `.bin/likec4` mit Node 22 ausführen.
  Aus der Windows-Shell (Git Bash, kein npx) bewährt:
  `MSYS_NO_PATHCONV=1 wsl.exe -d Ubuntu --cd /home/tassi/repos/chat-with-your-data_design -- bash -lc "mkdir -p /tmp/lc4 && cd /tmp/lc4 && (test -x node_modules/.bin/likec4 || npx -y -p node@22 -p npm@10 -- npm i likec4@1) && cd - && npx -y -p node@22 -- node /tmp/lc4/node_modules/likec4/bin/likec4.mjs validate c4_architecture/"`
- Dynamic Views: Eltern-Container und deren Komponenten nicht im selben Flow mischen (Warnung „nested actors“).
- Bei ADR-Änderungen Index in `README.md`, `docs/architecture.md` und ADR-Referenzen im C4-Modell konsistent halten.
