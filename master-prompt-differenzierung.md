# Master Prompt — Diferenzierungs-Features
# Direkt in Claude Code eingeben:

Du arbeitest am Comentra-Projekt. Lies zuerst:
/root/-comentra-specs/comentra-differenzierungs-spec.md

Verschaffe dir Überblick:
- find . -type f -name "*.ts" -o -name "*.tsx" | grep -v node_modules | sort
- Zeige package.json
- Zeige supabase/migrations/ (letzte 5 Dateien)

Globale Regeln:
1. Redis lokal (redis://redis:6379)
2. Supabase DB — RLS aktiv, service_role für API
3. Next.js App Router
4. Keine neue Dependency ohne Begründung + Paketgröße
5. Kein Breaking Change ohne Bestätigung
6. Wenn Kapazität oder API-Limits relevant → Warnung zuerst
7. Nach jeder Session: Zusammenfassung (Dateien, Neustart ja/nein)
8. Warte nach jeder Session auf "weiter"

Wichtig für Session 1 — KI-Klassifikation:
- Anthropic SDK (@anthropic-ai/sdk) ist bereits installiert
- Nutze claude-haiku-4-5-20251001 (günstig + schnell für Batch)
- ANTHROPIC_API_KEY ist bereits in .env gesetzt

Starte mit Session 1 — KI-Retourenmanagement.
