# Comentra — PIM Spec v2: Hauptartikel + Kanal-Karten
# Architektur-Prinzip: Ein Master, viele Overlays
# Kein Formular-Chaos — ein Produkt, aufgefächerte Kanal-Karten

---

## Globale Regeln
1. Redis lokal (redis://redis:6379)
2. Supabase DB — RLS aktiv
3. Next.js App Router
4. Keine neue Dependency ohne Begründung
5. Kein Breaking Change ohne Bestätigung
6. Nach jeder Session: Zusammenfassung (Dateien, Neustart ja/nein)
7. Warte nach jeder Session auf "weiter"

---

## Das Kernprinzip — verstehen bevor coden

```
Hauptartikel (Master)
├── titel_de: "Bilderrahmen Eiche 30x40cm"
├── beschreibung_de: "Hochwertiger Bilderrahmen..."
├── ean: "4251234567890"
├── gewicht_g: 450
├── bilder: [bild1.jpg, bild2.jpg, bild3.jpg]
└── maße: 30x40x2cm

                    ↓ Kanal-Karten (Overlays / Deltas)

Amazon-Karte           eBay-Karte             Otto-Karte
├── titel: OVERRIDE    ├── titel: OVERRIDE     ├── titel: OVERRIDE
│   "Bilderrahmen..."  │   "Bilderrahmen Eiche" │   (leer = Master)
├── bullets: [...]     ├── item_specifics: {...} ├── selling_points: [...]
├── keywords_backend   ├── kategorie_id: 12345  ├── produktlinie: "..."
├── a_plus: {...}      ├── payment_policy: "..." ├── gpsr: {...}
└── kategorie: "..."   └── (kein A+)            └── moin: "..."

Felder die NICHT überschrieben sind → kommen vom Hauptartikel
Felder die überschrieben sind → zeigt OVERRIDE Badge
```

**Was das für den Händler bedeutet:**
- Titel einmal eingeben → geht an alle Kanäle als Basis
- Nur was bei Amazon anders sein muss → in der Amazon-Karte überschreiben
- Bild einmal hochladen → alle Kanäle nutzen es (außer Kanal hat eigene Bilder)
- EAN einmal → überall
- Gewicht einmal → überall

**Was das für die Architektur bedeutet:**
- Hauptartikel = `products` Tabelle (bereits vorhanden)
- Kanal-Karte = `product_channel_cards` Tabelle mit NULLABLE Feldern
- NULL = "kommt vom Master"
- Wert = "überschreibe Master für diesen Kanal"
- Beim Upload: Merge von Master + Overlay → marktplatz-spezifisches Format

---

## Session 1 — Datenbank: Hauptartikel erweitern + Kanal-Karten

### Migration: supabase/migrations/20260420_hauptartikel_kanal_karten.sql

```sql
-- Hauptartikel: bestehende products Tabelle erweitern
-- Füge fehlende Master-Felder hinzu falls nicht vorhanden
alter table products
  add column if not exists beschreibung_de text,
  add column if not exists beschreibung_en text,
  add column if not exists beschreibung_fr text,
  add column if not exists beschreibung_it text,
  add column if not exists beschreibung_es text,
  add column if not exists beschreibung_nl text,
  add column if not exists beschreibung_pl text,
  add column if not exists gewicht_g int,
  add column if not exists laenge_cm numeric(8,2),
  add column if not exists breite_cm numeric(8,2),
  add column if not exists hoehe_cm numeric(8,2),
  add column if not exists ean text,
  add column if not exists marke text,
  add column if not exists hersteller text,
  add column if not exists hersteller_adresse text,
  add column if not exists hersteller_email text,
  add column if not exists hersteller_url text,
  add column if not exists material text,
  add column if not exists farbe text,
  add column if not exists zielgruppe text,
  add column if not exists steuersatz numeric(4,2) default 19.00,
  add column if not exists master_keywords text[],
  add column if not exists bilder_urls text[];

-- Kanal-Karten: Delta-Overlays über den Hauptartikel
-- Alle Felder NULLABLE — NULL = "nimm Wert vom Hauptartikel"
create table if not exists product_channel_cards (
  id uuid primary key default gen_random_uuid(),
  tenant_id uuid references tenants(id) on delete cascade,
  product_id uuid not null,
  channel_id uuid references channels(id) on delete cascade,
  marketplace text not null,

  -- === OVERLAY FELDER (NULL = Master-Wert verwenden) ===

  -- Gemeinsam überschreibbar
  titel text,           -- NULL = products.title verwenden
  beschreibung text,    -- NULL = products.beschreibung_de verwenden
  bilder_urls text[],   -- NULL = products.bilder_urls verwenden
  marke text,           -- NULL = products.marke verwenden
  ean text,             -- NULL = products.ean verwenden

  -- === MARKTPLATZ-SPEZIFISCHE FELDER (jsonb) ===
  -- Pflichtfelder die der Hauptartikel nicht kennt
  marketplace_fields jsonb not null default '{}',

  -- === STATUS & COMPLIANCE ===
  status text default 'draft' check (status in (
    'draft',        -- in Bearbeitung
    'ready',        -- Compliance OK, bereit zum Upload
    'uploading',    -- wird gerade hochgeladen
    'live',         -- live auf Marktplatz
    'error',        -- Upload fehlgeschlagen
    'suppressed',   -- vom Marktplatz unterdrückt
    'inactive'      -- deaktiviert
  )),
  compliance_score int default 0,    -- 0-100
  compliance_issues jsonb default '[]', -- [{field, severity, message, fix}]
  last_upload_at timestamptz,
  last_upload_error text,
  marketplace_listing_id text,       -- externe ID (ASIN, eBay Item ID etc.)

  -- === KI TRACKING ===
  ai_generated boolean default false,
  ai_last_run_at timestamptz,
  ai_model text,

  -- === TOP-10 INSPIRATION CACHE ===
  inspiration_fetched_at timestamptz,
  inspiration_data jsonb,            -- gecachte Top-10 Ergebnisse

  created_at timestamptz default now(),
  updated_at timestamptz default now(),

  unique(tenant_id, product_id, channel_id)
);

-- Indices
create index if not exists idx_cards_tenant_product
  on product_channel_cards(tenant_id, product_id);
create index if not exists idx_cards_tenant_status
  on product_channel_cards(tenant_id, status);
create index if not exists idx_cards_marketplace
  on product_channel_cards(tenant_id, marketplace);

-- RLS
alter table product_channel_cards enable row level security;
create policy "tenant_isolation" on product_channel_cards
  using (tenant_id = current_setting('app.current_tenant_id')::uuid);

-- Funktion: Resolved Card (Master + Overlay verschmelzen)
create or replace function get_resolved_card(
  p_product_id uuid,
  p_channel_id uuid,
  p_tenant_id uuid
) returns jsonb language plpgsql as $$
declare
  v_master jsonb;
  v_card jsonb;
  v_resolved jsonb;
begin
  -- Master-Daten holen
  select to_jsonb(p) into v_master
  from products p
  where p.id = p_product_id and p.tenant_id = p_tenant_id;

  -- Kanal-Karte holen
  select to_jsonb(c) into v_card
  from product_channel_cards c
  where c.product_id = p_product_id
    and c.channel_id = p_channel_id
    and c.tenant_id = p_tenant_id;

  if v_card is null then
    return v_master;
  end if;

  -- Merge: Kanal-Wert überschreibt Master wenn nicht NULL
  v_resolved := v_master || jsonb_strip_nulls(v_card);

  return v_resolved;
end;
$$;
```

---

## Session 2 — Merge-Engine + Marktplatz-Schemas

### src/lib/pim/merge-engine.ts

```typescript
// Löst Master + Overlay auf → marktplatz-fertiges Objekt
export async function resolveCard(
  productId: string,
  channelId: string,
  tenantId: string
): Promise<ResolvedCard>

// Merged Felder:
// titel: card.titel ?? product.title
// beschreibung: card.beschreibung ?? product.beschreibung_de
// bilder: card.bilder_urls ?? product.bilder_urls
// ean: card.ean ?? product.ean
// marke: card.marke ?? product.marke
// + alle marketplace_fields
```

### src/lib/pim/marketplace-schemas/ — Pflichtfelder pro Marktplatz

**amazon-schema.ts**
```typescript
export const AMAZON_REQUIRED = [
  { field: 'titel', maxLength: 200, rule: 'keywords_first' },
  { field: 'bullets', count: 5, maxLength: 250, rule: 'customer_intent' },
  { field: 'backend_keywords', maxBytes: 250, rule: 'no_repeats' },
  { field: 'marketplace_fields.product_type', source: 'sp-api' },
  { field: 'marketplace_fields.browse_nodes', minCount: 1 },
  { field: 'ean', format: 'gtin' },
  // 2026 Pflichtfelder:
  { field: 'marketplace_fields.epr_registration_id', condition: 'eu_marketplace' },
]
export const AMAZON_TITLE_RULE = '{marke} {produktname} {hauptmerkmal} {größe/farbe}'
export const AMAZON_MAX_VARIATIONS = 3 // max 3 Variations-Attribute
```

**otto-schema.ts**
```typescript
export const OTTO_REQUIRED = [
  { field: 'ean', format: 'gtin' },
  { field: 'marke', note: 'muss bei OTTO registriert sein' },
  { field: 'marketplace_fields.product_reference', immutable: true },
  { field: 'marketplace_fields.category' },
  { field: 'marketplace_fields.selling_points', count: 5 },
  // GPSR 2026 - EU Pflicht:
  { field: 'hersteller', required: true },
  { field: 'hersteller_adresse', required: true },
  { field: 'hersteller_email', required: true },
  { field: 'hersteller_url', required: true },
]
export const OTTO_TITLE_RULE = '{kategorie} {marke} {modellname} {titelattribute}'
```

**ebay-schema.ts**
```typescript
export const EBAY_REQUIRED = [
  { field: 'titel', maxLength: 80 },
  { field: 'marketplace_fields.item_specifics.Marke' },
  { field: 'marketplace_fields.item_specifics.EAN' },
  { field: 'marketplace_fields.category_id' },
  { field: 'marketplace_fields.payment_policy_id', note: 'Business Policy' },
  { field: 'marketplace_fields.return_policy_id', note: 'Business Policy' },
  { field: 'marketplace_fields.shipping_policy_id', note: 'Business Policy' },
]
```

**kaufland-schema.ts, zalando-schema.ts, shopify-schema.ts** — analog

### src/lib/pim/compliance-checker.ts
```typescript
// Führt alle Schema-Checks durch
// Gibt Score 0-100 + Issues-Liste zurück
export async function checkCompliance(
  resolvedCard: ResolvedCard,
  marketplace: string
): Promise<ComplianceResult>

// Score-Berechnung:
// 100 = alle Pflichtfelder gefüllt + alle Regeln erfüllt
// -10 pro critical issue
// -5 pro warning
// Upload gesperrt wenn score < 70
```

---

## Session 3 — KI-Generator + Top-10 Inspiration

### src/lib/pim/ai-card-generator.ts

```typescript
// Generiert Kanal-Karte aus Hauptartikel + Top-10 Inspiration
export async function generateChannelCard(
  productId: string,
  channelId: string,
  marketplace: string,
  tenantId: string
): Promise<Partial<ProductChannelCard>>

// Prozess:
// 1. Hauptartikel laden
// 2. Top-10 der Kategorie laden (Cache)
// 3. Marktplatz-Schema laden
// 4. Prompt für claude-haiku-4-5-20251001:
//    "Du bist ein {marketplace} Listing-Experte für den deutschen Markt.
//     Hauptartikel: {master}
//     Top-10 Wettbewerber: {top10}
//     Schema-Anforderungen: {schema}
//     Generiere: titel, bullets/selling_points, beschreibung, keywords
//     Format: JSON, nur die Felder die sich vom Hauptartikel unterscheiden"
// 5. Nur Deltas speichern — nicht den ganzen Master überschreiben
```

### src/lib/pim/top10-fetcher.ts

```typescript
// Top-10 Bestseller für Kategorie × Marktplatz
export async function fetchTop10(
  marketplace: string,
  category: string,
  keyword: string,
  tenantId: string
): Promise<Top10Result[]>

// Cache-Strategie:
// Redis L1: 30 Minuten
// DB (inspiration_data): 6 Stunden
// Danach: neu fetchen

// Pro Marktplatz:
// Amazon: SP-API /catalog/2022-04-01/items + /products/pricing
// eBay: Finding API findItemsByCategory?sortOrder=BestMatch
// Otto: Products API sortBy=RELEVANCE
// Kaufland: Catalog API
// Zalando: Partner API
// Shopify: Google Shopping via Merchant Center API
```

---

## Session 4 — Upload-Engine (Master + Overlay → Marktplatz-Format)

### src/lib/pim/upload-engine.ts

```typescript
// Lädt aufgelöste Karte in den Marktplatz hoch
export async function uploadCard(
  cardId: string,
  tenantId: string
): Promise<UploadResult>

// Prozess:
// 1. resolveCard() → merged Objekt
// 2. checkCompliance() → muss >= 70 sein
// 3. formatForMarketplace(merged, marketplace) → marktplatz-spezifisches Format
// 4. Über bestehenden Connector hochladen:
//    Amazon: SP-API Listings API
//    eBay: bestehende ebay/inventory.ts
//    Otto: bestehende otto/products.ts
//    Kaufland: bestehende kaufland/client.ts
//    Zalando: zalando/definition.ts
//    Shopify: woocommerce/definition.ts analog
// 5. marketplace_listing_id speichern
// 6. Status → 'live'

// Format-Mapper pro Marktplatz:
// src/lib/pim/formatters/amazon-formatter.ts
// src/lib/pim/formatters/otto-formatter.ts
// src/lib/pim/formatters/ebay-formatter.ts
// etc.
```

---

## Session 5 — API Routen

```
GET  /api/products/[id]/cards         — alle Kanal-Karten für Produkt
GET  /api/products/[id]/cards/[ch]    — einzelne Karte (resolved)
POST /api/products/[id]/cards/[ch]    — Karte erstellen/aktualisieren (nur Delta)
DELETE /api/products/[id]/cards/[ch]  — Karte löschen (zurück zu Master)

POST /api/products/[id]/cards/[ch]/ai-generate   — KI befüllt Karte
GET  /api/products/[id]/cards/[ch]/inspiration   — Top-10 holen
POST /api/products/[id]/cards/[ch]/compliance    — Compliance prüfen
POST /api/products/[id]/cards/[ch]/upload        — Hochladen

POST /api/products/[id]/cards/bulk-generate — alle Karten mit KI befüllen
POST /api/products/[id]/cards/bulk-upload   — alle ready Karten hochladen
```

---

## Session 6 — UI: Produktdetail mit Kanal-Karten

### Konzept (UI kommt erst vor Launch — aber Logik jetzt)

Das UI wird später gebaut, aber die Daten-Struktur muss jetzt stimmen.

**Ansicht: Produktdetail**
```
[Hauptartikel: Bilderrahmen Eiche 30x40cm]
Titel: Bilderrahmen Eiche 30x40cm    EAN: 4251234567890
Beschreibung: Hochwertiger...         Gewicht: 450g
Bilder: [▪][▪][▪]                    Marke: EigenMarke

Kanal-Karten:
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│ 🟡 Amazon DE    │  │ 🟢 eBay DE      │  │ 🔴 Otto         │
│ Score: 85/100   │  │ Score: 92/100   │  │ Score: 45/100   │
│ LIVE            │  │ LIVE            │  │ FEHLER          │
│                 │  │                 │  │ GPSR fehlt      │
│ [Bearbeiten]    │  │ [Bearbeiten]    │  │ [Reparieren]    │
│ [KI Update]     │  │ [KI Update]     │  │ [KI befüllen]   │
└─────────────────┘  └─────────────────┘  └─────────────────┘

┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│ ➕ Kaufland     │  │ ➕ Zalando      │  │ ➕ Shopify      │
│ Noch keine Karte│  │ Noch keine Karte│  │ Noch keine Karte│
│ [Karte anlegen] │  │ [Karte anlegen] │  │ [Karte anlegen] │
└─────────────────┘  └─────────────────┘  └─────────────────┘
```

**Karte öffnen (Bearbeiten):**
```
Linke Seite: Formular
- Nur die Felder die ANDERS sind als Master → mit "OVERRIDE" Badge
- Alle Pflichtfelder des Marktplatzes die der Master nicht hat
- "Zurücksetzen auf Master" Button pro Feld

Rechte Seite: Top-10 Inspiration
- 10 Karten mit Titel, Bullets, Score
- Klick → "In Karte übernehmen" (einzelne Felder oder alles)

Unten: Compliance-Check
- Score-Balken
- Rote Issues: Upload gesperrt
- Gelbe Warnungen: Upload erlaubt aber suboptimal
- Upload-Button (grau wenn score < 70)
```

### Was jetzt gebaut wird (Session 6):
- src/app/api/products/[id]/cards/ — alle Routen aus Session 5
- Integration in bestehende Produkt-Seite: Tab "Marktplätze" hinzufügen
- Keine UI — nur die Daten-Logik + API
- UI kommt wenn Backend komplett ist (Masterplan)
