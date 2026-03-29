# Master Prompt — Go-Live Features
# Direkt in Claude Code eingeben:

Du arbeitest am Comentra-Projekt. Lies zuerst die vollständige Spec:
/root/-comentra-specs/comentra-golive-features-spec.md

Dann verschaffe dir Überblick:
- Zeige aktuellen Verzeichnisbaum: find . -type f -name "*.ts" -o -name "*.tsx" | grep -v node_modules | sort
- Zeige package.json
- Zeige vorhandene Migrations unter supabase/migrations/

Globale Regeln die immer gelten:
1. Redis lokal (redis://redis:6379) — kein Upstash
2. Supabase bleibt die DB — RLS aktiv
3. Next.js App Router — kein Express
4. Keine neue Dependency ohne Begründung
5. Kein Breaking Change ohne Bestätigung
6. Nach jeder Session: Zusammenfassung (Dateien, Neustart ja/nein)
7. Wenn Kapazität oder externe API-Limits relevant werden → Warnung zuerst
8. Warte nach jeder Session auf "weiter"

Starte jetzt mit Session 1 — DATEV Export.
