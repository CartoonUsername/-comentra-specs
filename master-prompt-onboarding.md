# Master Prompt — Onboarding: 0 auf produktiv in 5 Minuten
# Direkt in Claude Code eingeben:

Du arbeitest am Comentra-Projekt.

Globale Regeln:
1. Redis lokal (redis://redis:6379)
2. Supabase DB — RLS aktiv
3. Next.js App Router
4. Migrations: Format YYYYMMDD_HHMMSS_name.sql, danach supabase db push
5. Kein Breaking Change ohne Bestätigung
6. Nach jeder Session: Zusammenfassung + supabase db push + git push
7. Warte nach jeder Session auf "weiter"

---

Session 1 — Guided Setup Wizard (5 Schritte)

Baue src/app/(onboarding)/setup/page.tsx — läuft AUSSERHALB des App-Routers
(kein Dashboard-Layout, eigene minimale Shell).

5 Schritte, Progress-Bar oben, Weiter/Zurück Navigation:

Schritt 1: Firma
- Firmenname (Pflicht)
- Logo-Upload (optional, Supabase Storage)
- Land (DE/AT/CH, Default DE)
- Währung (EUR, vorausgefüllt)
- "Weiter" → speichert in tenants Tabelle

Schritt 2: Ersten Marktplatz verbinden
- Karten: Amazon, eBay, Otto, Kaufland, Shopify, Zalando
- Klick → öffnet Credential-Formular (bestehende Kanal-Setup Logik nutzen)
- "Überspringen" möglich aber mit Hinweis

Schritt 3: Produkte importieren
- Wenn Amazon verbunden: "ASINs importieren" Button (bestehender ASIN-Import)
- Wenn eBay verbunden: "eBay Listings importieren"
- Wenn nichts verbunden: CSV-Upload
- "Überspringen" möglich

Schritt 4: KI-Analyse
- Ladeanimation: "KI analysiert dein Sortiment..."
- Ruft im Hintergrund auf:
  → ai-suggester.ts → erste 3 Regelvorschläge
  → portfolio-valuation.ts → ersten Produktwert
  → customer segments → erste Segmente
- Zeigt Ergebnis: "Wir haben X Produkte gefunden, Y Regelvorschläge erstellt"

Schritt 5: Fertig
- Konfetti-Animation (CSS only, kein canvas)
- Zusammenfassung: was verbunden ist
- "Zu Comentra" Button → /dashboard

Datenbank:
- onboarding_completed boolean in tenants (falls nicht vorhanden: Migration)
- Wenn onboarding_completed = false → redirect zu /setup
- Nach Schritt 5: onboarding_completed = true

Migrations-Format: 20260331_HHMMSS_onboarding.sql
Nach Migration: supabase db push

Warte auf weiter.

---

Session 2 — Migrations-Assistent (Von Billbee / JTL wechseln)

Baue src/app/(app)/settings/migration/page.tsx

3 Import-Flows:

1. Billbee-Import:
   - Billbee CSV-Export Format (Artikel-Export)
   - Spalten-Mapping: Billbee → Comentra products Tabelle
   - Felder: SKU, EAN, Titel, Beschreibung, Preis, Bestand, Kategorie
   - Validierung + Fehler-Report vor Import
   - Bulk-Insert mit Fortschrittsbalken

2. JTL-Import:
   - JTL-Wawi CSV-Export Format
   - Analoges Mapping

3. Generischer CSV-Import:
   - Spalten manuell zuordnen (Drag & Drop Mapping)
   - Pflichtfelder: SKU oder EAN
   - Preview erste 5 Zeilen vor Import

API: POST /api/settings/migration/upload
     POST /api/settings/migration/preview
     POST /api/settings/migration/execute

Nach Import: KI-Analyse automatisch starten (Session 1 Schritt 4 Logik)

Migrations-Format: 20260331_HHMMSS_migration_assistant.sql falls DB-Änderungen
Nach Migration: supabase db push + git push

Zusammenfassung dann weiter.

---

Session 3 — Seed-Daten für EmsCraft24 Test-Tenant

Erstelle scripts/seed-emscraftdata.ts

Für Tenant 6163af6f-dadf-446b-af24-c0f4498351e5:

50 realistische Bestellungen (letzte 30 Tage):
- Verschiedene Status: pending(10), processing(15), shipped(20), delivered(5)
- Kanäle: Amazon(25), eBay(15), Otto(10)
- Beträge: zwischen 15€ und 280€
- Echte deutsche Kundennamen + Adressen

20 Produkte:
- Home & Living Kategorie (Bilderrahmen, Spiegel, Deko)
- Mit EAN, Beschreibung, Preisen
- Inventory zwischen 5 und 150 Stück
- 5 Produkte mit Produktkarten für Amazon + eBay

5 Kunden (aus den Bestellungen)

3 offene Retouren:
- Verschiedene Gründe: defective, not_as_described, changed_mind
- KI-Klassifikation bereits gesetzt

2 Lieferanten:
- Mit Produkten und Reorder Points
- 1 Nachbestellvorschlag bereits generiert

Danach: npx tsx scripts/seed-emscraftdata.ts ausführen
Bestätigen dass Dashboard Daten zeigt.
git push danach.
