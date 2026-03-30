# Comentra — UI-Fixes Punkte 2–5
# Nacheinander in Claude Code eingeben. Nach jedem Punkt auf "weiter" warten.

---

## Punkt 2 — Sidebar Farbe vereinheitlichen

Schau dir die linke Sidebar an:
- In src/app/(app)/wms/page.tsx oder dem Lager-Screen ist die Sidebar dunkel/schwarz
- In allen anderen Screens ist sie hell

Finde heraus warum die Sidebar im Lager-Screen eine andere Farbe hat.
Vereinheitliche sie auf den gleichen Stil wie alle anderen Screens.
Kein Breaking Change am Layout — nur die Farbe/Theme angleichen.
Zeige mir welche Datei du änderst und warum.
Danach: docker compose up --build -d nur wenn nötig.

---

## Punkt 3 — Dashboard "No data" Bug

Im Dashboard-Widget "Channels" steht "No data" obwohl ein Amazon-Kanal
verbunden ist (Bessam AMZ, letzter Sync 26.03.2026).

1. Finde das Channels-Widget im Dashboard (wahrscheinlich dashboard-tabs.tsx
   oder ein eigenes Widget)
2. Debugge warum die Kanal-Daten nicht angezeigt werden
3. Prüfe ob die Supabase-Query die richtigen Daten zurückgibt
4. Fixe den Bug
5. Zeige mir den Fix und erkläre was falsch war

---

## Punkt 4 — Kanal-Setup Flow vervollständigen

Das Formular "Kanal hinzufügen" hat nur: Name + Dropdown + "Sofort aktivieren".
Die API-Credentials fehlen komplett.

Erweitere den Kanal-Setup Flow:

Wenn Marktplatz = Amazon gewählt:
- Seller ID (Text-Input, Pflichtfeld)
- Marketplace ID (Dropdown: DE=A1PA6795UKMFR9, FR=A13V1IB3VIYZZH, IT=APJ6JRA9NG5V4, ES=A1RKKUPIHCS9HS, UK=A1F83G8C2ARO7P)
- Access Key (Text-Input, Pflichtfeld)
- Secret Key (Password-Input, Pflichtfeld)
- Role ARN (Text-Input, optional)

Wenn Marktplatz = eBay gewählt:
- App ID (Text-Input, Pflichtfeld)
- Cert ID (Text-Input, Pflichtfeld)
- Dev ID (Text-Input, Pflichtfeld)
- User Token (Text-Input, Pflichtfeld)

Wenn Marktplatz = Otto gewählt:
- Client ID (Text-Input, Pflichtfeld)
- Client Secret (Password-Input, Pflichtfeld)

Credentials werden verschlüsselt in channels.credentials (jsonb) gespeichert —
das ist bereits so in der DB angelegt.

UI: Zeige die Credential-Felder dynamisch je nach gewähltem Marktplatz.
Füge einen "Verbindung testen" Button hinzu der einen Test-API-Call macht
und grünes/rotes Feedback zeigt.

Kein Breaking Change an bestehenden Kanälen — nur das Formular erweitern.

---

## Punkt 5 — Onboarding Flow

Neue Tenants sehen überall leere Screens. Das muss sich ändern.

Baue einen Onboarding-Flow:

1. Neue Datei: src/components/onboarding/OnboardingBanner.tsx
   - Zeigt sich wenn: tenant hat 0 Kanäle ODER 0 Produkte ODER 0 Bestellungen
   - Banner oben auf dem Dashboard mit Fortschrittsanzeige:
     ✅ Account erstellt
     ⬜ Ersten Kanal verbinden → Link zu /channels
     ⬜ Produkte importieren → Link zu /products
     ⬜ Fertig — loslegen!
   - Verschwindet automatisch wenn alle Schritte abgehakt sind
   - Kann manuell geschlossen werden (localStorage: onboarding_dismissed)

2. Neue Datei: src/components/onboarding/EmptyStateCard.tsx
   - Wiederverwendbare Komponente für leere Listen
   - Props: icon, title, description, ctaText, ctaHref
   - Ersetze alle "Keine X gefunden" Texte durch EmptyStateCard mit
     konkreten Handlungsaufforderungen:
     - Bestellungen leer: "Verbinde deinen ersten Marktplatz um Bestellungen zu sehen"
     - Lieferanten leer: "Füge deinen ersten Lieferanten hinzu"
     - Retouren leer: "Keine Retouren — alles im grünen Bereich 🎉"
     - Kanäle leer: "Verbinde Amazon, eBay oder Otto um zu starten"

3. Integriere OnboardingBanner in src/app/(app)/dashboard/page.tsx

Kein Breaking Change. Nur neue Komponenten + Integration in Dashboard.
Neustart nötig? Nur wenn Build-Änderungen.
