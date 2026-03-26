# Comentra — Produkte Modul: Hauptartikel + Kanal Produktkarten

Implementiere das vollständige Produkte-Modul in der bestehenden Comentra React-App.
Das aktuelle Modul zeigt nur eine einfache Tabelle mit einem Produkt ohne Kanalstruktur.
Ersetze es komplett durch die folgende Architektur.

---

## Ziel

Master-Produkt (Hauptartikel) als zentrale Datenquelle.
Pro Hauptartikel gibt es Kanal-Produktkarten für Amazon, eBay, Shopify, Otto, Kaufland, Etsy.
Jede Karte hat eigene Einstellungen (Overrides), die den Hauptartikel ergänzen.

---

## 1. Datenbankschema (Supabase)

Erstelle folgende Tabellen falls nicht vorhanden:

```sql
-- Hauptartikel
create table products (
  id uuid primary key default gen_random_uuid(),
  tenant_id uuid references tenants(id),
  name text not null,
  sku text,
  ean text,
  brand text,
  description text,
  base_price numeric(10,2),
  cost numeric(10,2),
  weight numeric(8,2),
  dimensions jsonb,
  images text[],
  category text,
  status text default 'active',
  created_at timestamptz default now(),
  updated_at timestamptz default now()
);

-- Kanal-Verknüpfungen
create table product_channels (
  id uuid primary key default gen_random_uuid(),
  product_id uuid references products(id) on delete cascade,
  channel text not null, -- 'amazon' | 'ebay' | 'shopify' | 'otto' | 'kaufland' | 'etsy'
  status text default 'not_listed', -- 'not_listed' | 'active' | 'error' | 'review' | 'paused'
  listing_id text,
  created_at timestamptz default now()
);

-- Kanal-spezifische Einstellungen (Overrides)
create table product_channel_settings (
  id uuid primary key default gen_random_uuid(),
  product_channel_id uuid references product_channels(id) on delete cascade,
  title_override text,
  description_override text,
  brand_override text,
  bullet_points text[],
  keywords text[],
  category_id text,
  attributes jsonb,
  fulfillment_type text,
  shipping_group text,
  handling_time int,
  subtitle text,
  vendor text,
  product_type text,
  tags text[],
  extra jsonb
);

-- Kanal-spezifische Preise
create table product_channel_prices (
  id uuid primary key default gen_random_uuid(),
  product_channel_id uuid references product_channels(id) on delete cascade,
  price numeric(10,2),
  sale_price numeric(10,2),
  min_price numeric(10,2),
  auto_pricing_rule jsonb
);

-- Kanal-spezifischer Bestand
create table product_channel_inventory (
  id uuid primary key default gen_random_uuid(),
  product_channel_id uuid references product_channels(id) on delete cascade,
  available int default 0,
  on_hand int default 0
);
```

RLS aktivieren: Alle Tabellen auf tenant_id filtern.

---

## 2. Typen (types/products.ts)

```ts
export type ChannelType = 'amazon' | 'ebay' | 'shopify' | 'otto' | 'kaufland' | 'etsy';

export type ChannelStatus = 'not_listed' | 'active' | 'error' | 'review' | 'paused';

export interface Product {
  id: string;
  name: string;
  sku?: string;
  ean?: string;
  brand?: string;
  description?: string;
  base_price?: number;
  cost?: number;
  weight?: number;
  images?: string[];
  category?: string;
  status: 'active' | 'inactive' | 'draft';
  channels?: ProductChannel[];
}

export interface ProductChannel {
  id: string;
  product_id: string;
  channel: ChannelType;
  status: ChannelStatus;
  listing_id?: string;
  settings?: ProductChannelSettings;
  price?: ProductChannelPrice;
  inventory?: ProductChannelInventory;
}

export interface ProductChannelSettings {
  title_override?: string;
  description_override?: string;
  bullet_points?: string[];
  keywords?: string[];
  category_id?: string;
  attributes?: Record<string, string>;
  fulfillment_type?: 'fba' | 'fbm' | 'seller';
  shipping_group?: string;
  handling_time?: number;
  subtitle?: string;
  tags?: string[];
  extra?: Record<string, unknown>;
}

export interface ProductChannelPrice {
  price?: number;
  sale_price?: number;
  min_price?: number;
  auto_pricing_rule?: unknown;
}

export interface ProductChannelInventory {
  available: number;
  on_hand: number;
}
```

---

## 3. Komponenten-Struktur

Erstelle folgende Dateien:

```
src/
  pages/
    ProductsPage.tsx
  components/products/
    MasterProductCard.tsx
    ChannelGrid.tsx
    ChannelProductCard.tsx
    ChannelSettingsDrawer.tsx
    ProductEditModal.tsx
    AddChannelModal.tsx
    tabs/
      AmazonSettings.tsx
      EbaySettings.tsx
      ShopifySettings.tsx
      OttoSettings.tsx
      KauflandSettings.tsx
      EtsySettings.tsx
  hooks/
    useProducts.ts
    useProductChannels.ts
```

---

## 4. ProductsPage.tsx

- Topbar: "Produkte" Titel + "+ Neues Produkt" Button
- Suchfeld (Name oder SKU)
- Filter: Alle / Aktiv / Inaktiv / Entwurf
- Liste von MasterProductCards
- Bulk-Aktionen wenn mehrere ausgewählt

---

## 5. MasterProductCard.tsx

Darstellung als horizontale Karte:

```
[Bild] [Name / SKU / Kategorie]  [Varianten]  [Bestand]  [Kanäle x/6]  [Status]  [Aktionen]
```

- Klick auf Karte: Expand-Bereich öffnet sich inline mit ChannelGrid
- Expand-Animation: smooth max-height transition
- Aktionen-Menü (drei Punkte): Bearbeiten, Varianten, Archivieren, Löschen
- Badge "MASTER ARTIKEL" in der oberen rechten Ecke
- Kanal-Indikator zeigt farbige Dots für verbundene Kanäle

---

## 6. ChannelGrid.tsx

- 3-spaltig Desktop, 2-spaltig Tablet, 1-spaltig Mobile
- Zeigt alle 6 Kanäle: Amazon, eBay, Shopify, Otto, Kaufland, Etsy
- Nicht gelistete Kanäle: grau mit "+ Kanal hinzufügen"
- Aktive Kanäle: farbige Darstellung mit echten Daten

---

## 7. ChannelProductCard.tsx

Jede Karte zeigt:
- Channel-Icon (SVG oder Buchstaben-Avatar in Kanalfarbe)
- Kanalname
- Titel (override oder Hauptartikel-Name)
- Preis
- Bestand (available / on-hand)
- Status-Badge: nicht gelistet | aktiv | fehler | in prüfung | pausiert

Status-Farben:
- aktiv → grün
- nicht gelistet → grau
- fehler → rot
- in prüfung → gelb/orange
- pausiert → blau

Hover-Aktionen:
- Bearbeiten (öffnet ChannelSettingsDrawer)
- Listen / Deaktivieren
- Neu synchronisieren

Klick auf Karte → ChannelSettingsDrawer öffnen

---

## 8. ChannelSettingsDrawer.tsx

- Slide-in von rechts (400px breit, full height)
- Header: "[Kanalname] — [Produktname]" + Schließen-Button
- Status-Badge + Listing-ID falls vorhanden
- Dynamisch: rendert je nach channel prop die passende Settings-Komponente
- Speichern-Button: upsert in product_channel_settings + product_channel_prices
- Aktions-Buttons: Auf Kanal listen / Deaktivieren / Synchronisieren

---

## 9. AmazonSettings.tsx

Tabs: Listing | Preis | Versand | Varianten | SEO | Erweitert

**Listing-Tab:**
- Titel (max 200 Zeichen, Zeichenzähler)
- Bullet Points (5 Felder, je max 500 Zeichen)
- Beschreibung (textarea)
- Brand, Hersteller
- Suchbegriffe (tag input)

**Preis-Tab:**
- Amazon Preis
- Sale Preis
- Automatische Preisregel (toggle + Regel-Editor)

**Versand-Tab:**
- FBA / FBM Toggle
- Versandgruppe (select)
- Bearbeitungszeit (Tage)

---

## 10. EbaySettings.tsx

Tabs: Listing | Preis | Versand | Kategorie | Artikelmerkmale | Erweitert

**Listing-Tab:**
- Titel (max 80 Zeichen)
- Untertitel (max 55 Zeichen)
- Beschreibung

**Preis-Tab:**
- eBay Preis
- Mindestpreis
- Auto Preisregel

**Versand-Tab:**
- Versandprofil (select)
- Kostenloser Versand (toggle)
- Bearbeitungszeit

**Artikelmerkmale-Tab:**
- Marke, Material, Farbe, Größe

---

## 11. ShopifySettings.tsx

- Titel Override
- Beschreibung Override
- Tags (tag input)
- Vendor, Produkt Typ
- Preis Override
- Bestand Override

---

## 12. useProducts.ts Hook

```ts
export function useProducts() {
  // fetchProducts(): lädt alle Produkte mit channels per JOIN
  // createProduct(data)
  // updateProduct(id, data)
  // deleteProduct(id)
  // addChannel(productId, channel)
  // removeChannel(channelId)
  // saveChannelSettings(channelId, settings, price)
  // syncChannel(channelId): triggert n8n webhook
}
```

---

## 13. Design-System

- Inter Font
- CSS-Variablen für Farben (kein hardcoded hex)
- Kanalfarben:
  - Amazon: #FF9900
  - eBay: #E53238
  - Shopify: #96BF48
  - Otto: #F00020
  - Kaufland: #E4000F
  - Etsy: #F56400
- Keine Emojis in der UI
- Professionelles, nicht-AI-artiges Design

---

## 14. Wichtige Regeln

1. Kein Daten-Duplikat: Override nur wenn explizit gesetzt
2. Fallback: Kein Override → Hauptartikel-Daten anzeigen (visuell als "Standard" markiert)
3. Alle 6 Kanal-Karten immer anzeigen (auch wenn nicht gelistet)
4. Supabase Realtime für Bestand-Updates aktivieren
5. Fehlerbehandlung: Fehlerstatus in Karte anzeigen

---

## Liefere:

1. Alle Komponenten-Dateien (vollständig, kein Platzhalter)
2. SQL-Migrations-Dateien
3. Type-Definitionen
4. useProducts + useProductChannels Hooks
5. Routing-Update falls nötig

Starte mit DB-Schema → Typen → Hooks → Komponenten von außen nach innen.
