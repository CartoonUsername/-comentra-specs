# Comentra — Kampfliste
# Ziel: Jeden Konkurrenten in jedem Bereich strukturell überholen.
# Vorteil: Claude Code ersetzt 6 Monate Teamarbeit in einer Stunde.
# Prinzip: Nicht nachmachen — überholen.

---

## Der Grundgedanke

plentymarkets hat 500 Mitarbeiter.
- ~150 Kundensupport → Claude Code ersetzt das mit KI-gestütztem Self-Service
- ~100 Entwickler → Claude Code baut in 1h was die in 6 Monaten planen
- ~50 QA → TypeScript + automatische Tests
- ~100 Sales/Marketing → besseres Produkt verkauft sich selbst
- ~100 Management/Admin → Multi-Tenant-Architektur braucht kein Team pro Kunde

Comentra braucht kein Team. Es braucht Architektur die skaliert.

---

## Kampfbereich 1: Marktplatz-Anbindungen

### Aktueller Stand
Comentra: 16 Marktplätze
plentymarkets: 300+ Marktplätze
Billbee: 120+ Marktplätze

### Warum Konkurrenten hier gewinnen
Jede neue Anbindung = 2-4 Wochen Entwicklerzeit bei menschlichen Teams.
API-Änderungen = Notfall-Update = Nachtschicht.

### Wie Comentra gewinnt
**Connector-Generator**: Claude Code schreibt neue Connector-Specs in Minuten.
Ziel: 100+ Marktplätze in 30 Tagen.
Priorität: Alle deutschen + europäischen Marktplätze zuerst.

Fehlende Priorität-Connectors:
- Zalando (größter Fashion-Marktplatz DE)
- Kaufland (bereits teilweise)
- Real.de / Kaufland alt
- AboutYou
- Zooplus
- Notebooksbilliger
- Conrad
- Alternate
- Mediamarkt/Saturn (Marketplace)
- Decathlon
- Fnac (FR)
- Bol.com (NL/BE)
- Cdiscount (FR)
- Allegro (PL)
- Emag (RO)
- Ozon (RU)
- Wildberries (RU)
- TikTok Shop (neu, kein Konkurrent hat das gut)
- Pinterest Shopping
- Instagram Shopping
- Google Shopping direkt
- Facebook Marketplace

**Automatische API-Monitoring**: Wenn Marktplatz-API sich ändert →
System erkennt es automatisch → Claude Code generiert Fix → deployed.
Kein menschliches Team nötig.

### Spec-Datei
comentra-connector-generator-spec.md

---

## Kampfbereich 2: KI-Autopilot

### Aktueller Stand
Comentra: KI-Retouren, Regel-Vorschläge, CRM-Personalisierung
Konkurrenten: maximal rudimentäre Automatisierung, kein echter KI-Einsatz

### Warum das der größte Hebel ist
plentymarkets braucht 500 Mitarbeiter weil Händler ständig Fragen haben,
Probleme lösen müssen, Entscheidungen treffen müssen.
Ein echter KI-Autopilot macht das für den Händler.

### Wie Comentra gewinnt — der vollständige Autopilot

**Ebene 1 — Reaktiv (bereits teilweise gebaut)**
- Retoure eingehend → KI entscheidet + Label → fertig
- Bestand niedrig → KI bestellt nach → fertig
- Preis sinkt → KI repriced → fertig

**Ebene 2 — Proaktiv (noch nicht gebaut)**
- KI analysiert täglich Verkaufsdaten → schlägt Sortimentserweiterungen vor
- KI erkennt saisonale Muster → erhöht Lagerbestand vor Peak automatisch
- KI erkennt schlechte Listings → schreibt bessere Texte automatisch
- KI erkennt Kundensegmente → erstellt automatisch Preisgruppen

**Ebene 3 — Strategisch (Langfrist)**
- KI analysiert Marktposition vs Konkurrenten täglich
- KI gibt Empfehlungen: "Produkt X auf Zalando listen → +23% Umsatz"
- KI erstellt monatlichen Business-Report automatisch
- KI-Chat: Händler fragt "Warum sind meine Retouren diese Woche höher?"
  → KI analysiert und antwortet mit konkreten Maßnahmen

### Spec-Datei
comentra-ki-autopilot-spec.md

---

## Kampfbereich 3: Kundensupport ohne Supportteam

### Das Problem bei Konkurrenten
plentymarkets: 150 Support-Mitarbeiter
Billbee: bekannt für guten aber langsamen Support
JTL: Community-Forum als primärer Support

### Wie Comentra gewinnt — Zero-Human Support

**KI-Wissensdatenbank**
- Alle Docs, alle API-Referenzen, alle Feature-Beschreibungen als Embeddings
- Händler tippt Frage → sofortige Antwort aus der Wissensdatenbank
- Kein Ticket, keine Wartezeit, keine Support-Mitarbeiter nötig
- Ollama bereits im Stack für lokale Embeddings

**Kontextueller In-App-Assistent**
- "AI"-Button bereits in der Navigation (sichtbar in Screenshots)
- Händler ist auf der Bestellungs-Seite → KI kennt den Kontext
- "Warum wurde diese Bestellung nicht synchronisiert?" → KI schaut
  in die Daten und erklärt es direkt

**Automatische Fehler-Erkennung + Selbstheilung**
- System erkennt wenn ein Sync fehlschlägt
- Versucht automatisch zu reparieren
- Wenn nicht → erklärt dem Händler in plain Deutsch was zu tun ist
- Kein "Error 500" — immer eine verständliche Fehlermeldung mit Lösung

**Video-Tutorials automatisch generiert**
- Jede neue Feature-Release → KI generiert automatisch Erklärungs-Text
- Kein Content-Team nötig

### Spec-Datei
comentra-zero-support-spec.md

---

## Kampfbereich 4: Onboarding — von 0 auf produktiv in 15 Minuten

### Aktueller Stand
Billbee: 15 Minuten Setup (ihr größter Vorteil laut eigener Aussage)
JTL: Wochen
plentymarkets: Tage bis Wochen

### Wie Comentra gewinnt — 5 Minuten Setup

**Guided Setup Wizard (5 Schritte)**
1. Firma anlegen (Name, Logo upload) → 30 Sekunden
2. Ersten Marktplatz verbinden (OAuth flow) → 2 Minuten
3. Produkte automatisch importieren (aus Marktplatz) → 1 Minute
4. KI analysiert Daten → erste 3 Regelvorschläge sofort → 1 Klick aktivieren
5. Fertig — Dashboard zeigt echte Daten

**Migrations-Assistent**
- "Ich komme von Billbee" → CSV-Import-Template für Billbee-Export
- "Ich komme von JTL" → JTL-spezifischer Import-Flow
- "Ich komme von plentymarkets" → plenty-Export-Format unterstützt
- Alle historischen Daten in < 10 Minuten importiert

**KI-Erstkonfiguration**
- Nach Daten-Import: KI analysiert Sortiment + Kanäle
- Konfiguriert automatisch: Versandregeln, Repricing-Basis, E-Mail-Templates
- Händler muss nichts manuell konfigurieren — KI tut es

### Spec-Datei
comentra-onboarding-spec.md

---

## Kampfbereich 5: PIM — Produktdaten-Management

### Aktueller Stand
Comentra: Basis-PIM, ASIN-Import
plentymarkets: Vollständiges PIM mit Marktplatz-spezifischen Attributen
Akeneo (PIM-Spezialist): Industriestandard PIM

### Wie Comentra gewinnt — KI-gestütztes PIM

**Automatische Listing-Optimierung**
- KI analysiert Produkttitel → schlägt SEO-optimierten Titel vor
- KI vergleicht eigene Listings mit Top-3 Konkurrenten auf Amazon
- KI schreibt automatisch Bullet Points + Beschreibung
- Keine Content-Agentur nötig

**Marktplatz-spezifische Attribute automatisch**
- Produkt wird angelegt → KI befüllt automatisch alle Pflichtfelder
  für Amazon, eBay, Otto etc.
- Händler muss nie "Pflichtfeld fehlt" Fehler sehen

**Bild-Management**
- Automatische Bildkomprimierung für jeden Marktplatz (Größen, Formate)
- Hintergrund-Entfernung automatisch (KI)
- Compliance-Check: "Dieses Bild erfüllt Amazon-Anforderungen nicht"

**Übersetzungs-Engine**
- Produkt in DE → automatisch in EN, FR, IT, ES, NL, PL übersetzt
- Marktplatz-spezifische Sprache (Amazon.fr erwartet anderen Ton als Amazon.de)

### Spec-Datei
comentra-pim-spec.md

---

## Kampfbereich 6: Analytics & Business Intelligence

### Aktueller Stand
Comentra: Basis-Dashboard, Statistiken
plentymarkets: Dashboards, Reports
Spezialist-Tools: Sellerboard, Datahawk (Amazon-Analytics)

### Wie Comentra gewinnt — BI das Entscheidungen trifft

**Echtzeitanalyse die Handlungen vorschlägt**
Nicht: "Dein Umsatz ist gesunken"
Sondern: "Dein Umsatz ist 23% gesunken weil Produkt X auf Amazon seit
3 Tagen nicht die Buy Box hat. Klick hier um das zu beheben."

**Margen-Analyse pro SKU, pro Kanal, pro Tag**
- Einstandspreis + alle Kosten (Marktplatz-Gebühren, Versand, Retouren)
- Echter Gewinn pro Bestellung — nicht nur Umsatz
- Welche Produkte auf welchem Kanal profitabel sind

**Predictive Analytics**
- "In 14 Tagen läuft Produkt X aus" → jetzt nachbestellen
- "Nächsten Monat ist Saisonpeak für diese Kategorie" → Lager aufstocken
- "Dieser Kunde kauft alle 60 Tage" → E-Mail in 55 Tagen

**Wettbewerber-Monitoring**
- Preise der Top-5 Konkurrenten auf Amazon täglich tracken
- Wenn Konkurrent ausverkauft → automatisch Preis erhöhen (bereits Repricing)
- Neue Konkurrenten erkennen

### Spec-Datei
comentra-bi-spec.md

---

## Kampfbereich 7: Enterprise & Compliance

### Aktueller Stand
Comentra: Basis-Auth, RLS, kein SSO
plentymarkets/weclapp: SSO, SAML, granulare Rollen

### Wie Comentra gewinnt

**SSO / SAML** → Okta, Azure AD, Google Workspace
**Granulare Rollen** → nicht 3-4 Rollen sondern beliebig konfigurierbar
**Vollständiges Audit Log** → jede Aktion, jeder User, jede Änderung
**DSGVO-Tooling** → Daten-Export per Klick, Lösch-Anfragen automatisch
**Multi-Standort** → ein Tenant, mehrere Lager, mehrere Teams
**White-Label** → Agenturen können Comentra unter eigenem Brand verkaufen

### Spec-Datei
comentra-enterprise-spec.md

---

## Kampfbereich 8: Integrationen & Ökosystem

### Aktueller Stand
Comentra: n8n, Stripe, Resend, Shipcloud, lexoffice, sevDesk
plentymarkets: 300+ offizielle Plugins

### Wie Comentra gewinnt — offenes Ökosystem

**Plugin-System**
- Öffentliche API bereits gebaut (Session 4)
- Plugin-Marketplace: Drittanbieter können Integrationen bauen
- Jede Integration automatisch dokumentiert

**Tiefe Shopware/Shopify/WooCommerce Integration**
- Bidirektional: nicht nur Bestellungen importieren
  sondern auch Bestände + Preise zurückschreiben
- Shopware 6 Plugin
- Shopify App Store Listing

**ERP-Anbindung**
- SAP Business One → REST API bereits gebaut
- Sage → Connector
- Lexware → Connector

### Spec-Datei
comentra-ecosystem-spec.md

---

## Priorisierung — Reihenfolge der Specs

| Priorität | Bereich | Warum zuerst | Zeit-Schätzung |
|---|---|---|---|
| 🔥 1 | Connector-Generator | Breite — 100 Marktplätze | 2 Sessions |
| 🔥 2 | KI-Autopilot Ebene 2+3 | Größter Differenziator | 3 Sessions |
| 🔥 3 | PIM mit KI | Täglich genutzt | 2 Sessions |
| 🔥 4 | Onboarding 5min | Erster Eindruck | 1 Session |
| 🔥 5 | Analytics/BI | Kaufentscheidung | 2 Sessions |
| 🔥 6 | Enterprise | Großkunden | 2 Sessions |
| 🔥 7 | Zero-Support | Kein Team nötig | 2 Sessions |
| 🔥 8 | Ökosystem | Langfristig | 3 Sessions |

---

## Die Waffe: Claude Code als Multiplikator

plentymarkets plant 6 Monate für ein Feature:
1. Monat 1: Anforderungsanalyse
2. Monat 2: Design + Architektur
3. Monat 3-4: Entwicklung
4. Monat 5: Testing
5. Monat 6: Deployment + Doku

Claude Code macht das in 1 Stunde.

Das bedeutet: Comentra kann in einem Monat implementieren was
plentymarkets in 5 Jahren gebaut hat.

Das ist kein Vorteil. Das ist ein anderes Spiel.
