# Comentra — Master Prompt: Sessions 2–8
# Kopiere diesen Text 1:1 in Claude Code

---

Du arbeitest am Comentra-Projekt. Lies zuerst die vollständige Spec:
/root/-comentra-specs/comentra-claude-code-spec.md

Dann verschaffe dir einen Überblick über die aktuelle Projektstruktur:
- Zeige mir den kompletten Verzeichnisbaum (find . -type f -name "*.ts" -o -name "*.tsx" -o -name "*.js" | grep -v node_modules | grep -v .next | sort)
- Zeige mir package.json
- Zeige mir docker-compose.yml
- Zeige mir die aktuell vorhandenen Dateien in /src/cache/, /src/sync/, /src/cron/ falls vorhanden

Danach arbeite Session für Session durch — von Session 2 bis Session 8.

Regeln die immer gelten:
1. Nach jeder Session: Zusammenfassung (was geändert, welche Dateien, ob docker compose up --build -d nötig)
2. Warte nach jeder Session auf mein "weiter" bevor du die nächste startest
3. Keine neuen Dependencies ohne kurze Begründung
4. Kein Breaking Change an bestehenden API-Contracts oder Supabase-Queries
5. TypeScript überall wo es bereits TypeScript gibt
6. Wenn du eine Datei nicht findest die laut Spec existieren soll — sag es mir, erfinde sie nicht

Starte jetzt mit Session 2.
