# Comentra — People & Team Module
# Claude Code Prompt
# ─────────────────────────────────────────────

Build the complete People & Team module for Comentra.
Entry point: src/pages/People.tsx
Sub-components: src/components/people/

Design reference (study the layout language, card density, and component style):
https://dribbble.com/shots/24690997-Dashboard-UI-components-for-the-online-banking-platform

This is not a Slack clone. This is the complete internal operating system
for every person inside a Comentra tenant (Unternehmen).

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
LAYOUT
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Three-column layout:

LEFT SIDEBAR (220px)
  - Company name + Logo
  - Navigation:
      People (alle Mitarbeiter)
      Channels (#lager, #allgemein, #bestellungen)
      Direct Messages (nach Mitarbeiter sortiert)
      Aufgaben (meine + alle)
      Anrufe (Verlauf)
  - Online-Status aller Mitarbeiter (grüner Dot)
  - Eigenes Profil unten mit Status-Wechsel

CENTER (flex: 1)
  - Wechselt je nach aktiver Ansicht:
      → Mitarbeiterliste / Organigramm
      → Chat-Verlauf (Channel oder DM)
      → Aufgaben-Board
      → Anruf-Interface
      → Mitarbeiter-Profil

RIGHT PANEL (300px)
  - Kontextpanel — zeigt immer das verknüpfte Objekt:
      Wenn Chat offen: Mitarbeiter-Kurzprofil + geteilte Dateien
      Wenn Aufgabe offen: verknüpfte Bestellung / Lager-Event
      Wenn Profil offen: Aktivitäts-Feed dieser Person


━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
MODULE 1 — MITARBEITER-PROFIL
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Jeder Mitarbeiter hat eine vollständige Profilseite:

HEADER
  - Avatar (Initialen-Circle, farbkodiert nach Abteilung)
  - Name, Rolle, Abteilung
  - Online-Status: Online / Abwesend / Beschäftigt / Offline
  - Schnellaktionen direkt im Header:
      [Nachricht schreiben]
      [Anrufen]
      [Aufgabe zuweisen]
      [Aktivität ansehen]

KONTAKTDATEN
  - E-Mail (klickbar)
  - Telefon (klickbar → startet Anruf)
  - Standort / Schicht
  - Eintrittsdatum

ZUSTÄNDIGKEITEN
  - Abteilung: Lager / Kundenservice / Einkauf / Finanzen
  - Zugewiesene Kanäle
  - Aktuelle offene Aufgaben (mit Status-Badge)

VERKNÜPFTE OBJEKTE (das Kernstück)
  - Letzte Bestellungen die dieser Mitarbeiter bearbeitet hat
  - Letzte Lager-Events (Pick, Pack, Restock)
  - Offene Retouren die ihm zugewiesen sind
  - Klick auf jedes Objekt → öffnet direkt die jeweilige Seite

AKTIVITÄTS-FEED
  - Chronologischer Feed aller Aktionen dieser Person
  - Icons je nach Typ: Pick / Pack / Chat / Aufgabe / Anruf
  - Filterbar: Heute / Diese Woche / Alles


━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
MODULE 2 — CHAT
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Channels (#lager, #bestellungen, #allgemein, #retouren)
Direct Messages (1:1 zwischen Mitarbeitern)

NACHRICHTEN-FEATURES
  - Text
  - @mentions (öffnet Mitarbeiter-Kurzprofil on hover)
  - Objekt-Verknüpfung: Tippe /order #4821 → fügt klickbare
    Bestellkarte ein
  - Tippe /sku SKU-991 → fügt Produktkarte ein
  - Tippe /task → erstellt direkt eine Aufgabe aus dem Chat
  - Reaktionen (Emoji)
  - Antwort auf Nachricht (Thread)

NACHRICHTENKARTE (verknüpftes Objekt im Chat)
  - Mini-Card mit Icon + Titel + Status
  - Klick → öffnet Objekt in neuem Panel rechts
  - Beispiel: "Schau mal kurz" + [Order #4821 · Active · 3/5 Items]

UNGELESENE BADGES auf allen Channels und DMs


━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
MODULE 3 — ANRUFE
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Anruf-Button auf jedem Mitarbeiter-Profil und in DMs.

ANRUF-UI (Modal oder Inline-Overlay)
  - Avatar + Name + "Verbinde..."
  - Aktiver Anruf: Timer + [Stummschalten] [Lautsprecher] [Auflegen]
  - Während des Anrufs: Chat bleibt sichtbar im Hintergrund

ANRUF-VERLAUF
  - Liste aller vergangenen Anrufe
  - Eingehend / Ausgehend / Verpasst (farbkodiert)
  - Dauer
  - Klick → direkt zurückrufen

HINWEIS FÜR IMPLEMENTIERUNG:
  UI komplett bauen. Für echte Anruf-Funktion später
  WebRTC oder Daily.co integrieren. Jetzt: UI-only mit
  simuliertem State.


━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
MODULE 4 — AUFGABEN
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Board-Ansicht (Kanban) + Listen-Ansicht umschaltbar.

AUFGABEN-FELDER
  - Titel
  - Beschreibung
  - Zugewiesen an (Mitarbeiter-Picker)
  - Priorität: Hoch / Mittel / Niedrig
  - Status: Offen / In Arbeit / Review / Erledigt
  - Fälligkeitsdatum
  - Verknüpftes Objekt: Bestellung / Retour / SKU / Lager-Event

ERSTELLEN
  - Quick-Add per [+ Aufgabe] Button
  - Oder direkt aus dem Chat per /task Command

MEINE AUFGABEN
  - Gefilterte Ansicht nur für den eingeloggten User
  - Sortiert nach Priorität + Fälligkeit


━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
MODULE 5 — ORGANIGRAMM
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Visuelle Unternehmensstruktur als Tree-Layout.

  [Geschäftsführer]
       │
  ┌────┴────┐────────┐
[Lager]  [Einkauf]  [Kundenservice]
  │
[Picker] [Packer] [Restock]

- Jeder Node: Avatar + Name + Rolle + Online-Dot
- Klick auf Node → öffnet Mitarbeiter-Profil im rechten Panel
- Abteilungen farbkodiert (Lager = Orange, Einkauf = Blau, etc.)


━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
VERKNÜPFUNGS-SYSTEM (Kernlogik)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Alles in Comentra ist miteinander verknüpft:

  Chat-Nachricht → enthält Order #4821
    → Klick öffnet Bestelldetails in Bestellungen-Seite

  Aufgabe → verknüpft mit Retour RET-0042
    → Klick öffnet Retouren-Seite mit dieser Retour

  Mitarbeiter-Profil → zeigt letzte Pick-Events
    → Klick öffnet Lager-Seite mit diesem Event

  Aktivitäts-Feed → zeigt Anruf mit Rainer K.
    → Klick öffnet Anruf-Verlauf + Chat-Verlauf mit Rainer K.

Implementierung:
  Nutze React Router für alle Links.
  Alle Objekt-Karten haben einheitliches Format:
  { type: 'order'|'sku'|'return'|'task'|'event', id: string, 
    title: string, status: string }
  Eine universelle <LinkedObjectCard /> Komponente rendert alle.


━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
DESIGN RULES
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Font:        Syne (UI) + JetBrains Mono (IDs, Codes, Timestamps)
Background:  #F7F5F0 (Seite), #FFFFFF (Cards/Panel)
Akzent:      #C9521A (Orange) — einziger Farbakzent
Border:      #E2DDD4 (1px, überall konsistent)
Text:        #1A1714 (primary), #6B6560 (secondary), #A09890 (hint)

Status-Farben (nur für Status-Dots und Badges):
  Online:      #1A7A4A (Grün)
  Abwesend:    #8A5A0A (Amber)  
  Beschäftigt: #B52020 (Rot)
  Offline:     #A09890 (Grau)

Avatare: Initialen-Circle, Farbe nach Abteilung
  Lager:         Orange-Tint
  Einkauf:       Blau-Tint
  Kundenservice: Grün-Tint
  Finanzen:      Lila-Tint

VERBOTEN:
  - Kein Tailwind-Generic-Look
  - Keine Slack/Teams Kopie
  - Keine Gradients
  - Keine Stock-Icons die nach Bootstrap aussehen
  - Kein schwarzes Dark Theme


━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
MOCK DATA (für alle Komponenten)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Mitarbeiter:
  - Bessam N. / Geschäftsführer / Online
  - Rainer K. / Lager – Picker / Online
  - Lisa M.   / Lager – Picker / Abwesend
  - Tom B.    / Lager – Packer / Online
  - Mirko D.  / Lager – Restock / Beschäftigt
  - Sara H.   / Kundenservice / Online

Channels:
  #allgemein (12 ungelesen)
  #lager (3 ungelesen)
  #bestellungen
  #retouren

Aufgaben (Beispiele):
  "Retoure RET-0042 prüfen" → Mirko D. → Hoch → verknüpft mit RET-0042
  "SKU-991 nachbestellen"   → Bessam N. → Hoch → verknüpft mit SKU-991
  "Amazon Listing #4821"    → Sara H.   → Mittel

Anruf-Verlauf:
  Rainer K. → ausgehend → 4min 32s → heute 09:14
  Lisa M.   → verpasst  → —        → heute 08:55
  Tom B.    → eingehend → 1min 12s → gestern 17:40


━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
FILE STRUCTURE
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

src/pages/People.tsx                    ← Haupt-Seite
src/components/people/
  MitarbeiterProfil.tsx                 ← Vollprofil-Ansicht
  MitarbeiterKarte.tsx                  ← Mini-Card für Listen
  ChatView.tsx                          ← Channel + DM Chat
  ChatNachricht.tsx                     ← Einzelne Nachricht
  LinkedObjectCard.tsx                  ← Universelle Objekt-Karte
  AnrufOverlay.tsx                      ← Anruf-Modal
  AnrufVerlauf.tsx                      ← Anruf-Liste
  AufgabenBoard.tsx                     ← Kanban Board
  AufgabenKarte.tsx                     ← Einzelne Aufgabe
  OrgChart.tsx                          ← Organigramm Tree
  AktivitaetsFeed.tsx                   ← Feed-Komponente
  StatusDot.tsx                         ← Online-Status Indicator
src/types/people.ts                     ← Alle TypeScript Types
src/data/peopleMock.ts                  ← Mock Daten


━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
WICHTIG
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

- Keine bestehenden Dateien anfassen
- Nur neue Dateien erstellen
- Alle Komponenten TypeScript strict
- Keine any types
- Bestehende Supabase-Hooks nicht berühren
- Production ready — kein TODO-Code
