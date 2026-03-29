# Comentra — Spec: Go-Live Features
> Priorität: Diese 3 Features fehlen für den Go-Live im deutschen Markt.
> Arbeite Session für Session. Warte nach jeder Session auf "weiter".

---

## Globale Regeln (immer gültig)
1. Redis läuft lokal (redis://redis:6379) — kein Upstash
2. Supabase bleibt die Datenbank — RLS aktiv, service_role für API
3. Next.js App Router — kein Express
4. Keine neue Dependency ohne Begründung + Paketgröße
5. Kein Breaking Change ohne explizite Bestätigung
6. Wenn VPS-Kapazität (4 CPU / 8–16GB) nicht reicht → Warnung zuerst
7. Deployment: docker compose up --build -d — kein Auto-Deploy
8. Nach jeder Session: Zusammenfassung (Dateien, Neustart ja/nein)

---

## Session 1 — DATEV / Steuer-Export

### Warum
Deutscher E-Commerce-Markt: Jeder Händler braucht DATEV-kompatible Exporte
für seinen Steuerberater. Ohne das kauft niemand Comentra.

### Was bauen
Erstelle einen DATEV-Export-Service:

1. Neue Supabase-Tabelle (Migration erstellen):
```sql
create table datev_exports (
  id uuid primary key default gen_random_uuid(),
  tenant_id uuid references tenants(id) on delete cascade,
  period_start date not null,
  period_end date not null,
  export_type text not null check (export_type in ('invoices', 'orders', 'full')),
  status text not null default 'pending' check (status in ('pending', 'processing', 'done', 'error')),
  file_url text,
  row_count int,
  created_at timestamptz default now(),
  created_by uuid references auth.users(id)
);
alter table datev_exports enable row level security;
create policy "tenant_isolation" on datev_exports
  using (tenant_id = current_setting('app.current_tenant_id')::uuid);
```

2. Export-Logik unter src/lib/export/datev.ts:
- Liest Orders + Positionen für den gewählten Zeitraum
- Erzeugt CSV im DATEV-Format (Buchungsstapel):
  - Spalten: Umsatz, Soll/Haben, Konto, Gegenkonto, Buchungstext, Belegdatum, Belegnummer
  - Encoding: UTF-8 mit BOM (DATEV-Anforderung)
  - Trennzeichen: Semikolon
- Speichert Datei in Supabase Storage (bucket: datev-exports, private)
- Gibt signed URL zurück (gültig 1h)

3. API-Route: src/app/api/export/datev/route.ts
- POST mit { period_start, period_end, export_type }
- Schreibt datev_exports Eintrag mit status='processing'
- Startet Export async (BullMQ listings:low Queue)
- Gibt export_id zurück

4. API-Route: src/app/api/export/datev/[id]/route.ts
- GET: Polling-Endpunkt — gibt aktuellen Status + file_url zurück

5. UI-Komponente: src/components/export/DatevExportDialog.tsx
- Modal mit: Zeitraum-Picker (von/bis), Export-Typ (Dropdown), Export-Button
- Polling alle 2s bis Status 'done' → automatischer Download
- Zeige Row-Count nach Abschluss

### Kein Breaking Change
Bestehende Export-Routen unter /api/export/* bleiben unverändert.

---

## Session 2 — Versand-Integration (DHL + Sendcloud)

### Warum
Versand ist der tägliche Kernprozess jedes Händlers.
Billbee gewinnt nur weil es Sendcloud und DHL direkt integriert hat.

### Was bauen

1. Neue Supabase-Tabellen (Migration):
```sql
create table shipping_providers (
  id uuid primary key default gen_random_uuid(),
  tenant_id uuid references tenants(id) on delete cascade,
  provider text not null check (provider in ('dhl', 'sendcloud', 'gls', 'dpd')),
  credentials jsonb not null default '{}',
  is_active boolean default true,
  created_at timestamptz default now()
);
alter table shipping_providers enable row level security;
create policy "tenant_isolation" on shipping_providers
  using (tenant_id = current_setting('app.current_tenant_id')::uuid);

create table shipments (
  id uuid primary key default gen_random_uuid(),
  tenant_id uuid references tenants(id) on delete cascade,
  order_id uuid references orders(id) on delete cascade,
  provider text not null,
  tracking_number text,
  label_url text,
  status text default 'pending',
  weight_g int,
  created_at timestamptz default now()
);
alter table shipments enable row level security;
create policy "tenant_isolation" on shipments
  using (tenant_id = current_setting('app.current_tenant_id')::uuid);
```

2. Provider-Abstraktion: src/lib/shipping/base-provider.ts
- Abstrakte Klasse ShippingProvider
- Methoden: createLabel(order, weight), trackShipment(trackingNumber), cancelLabel(labelId)

3. DHL-Integration: src/lib/shipping/providers/dhl.ts
- DHL Paket API v2 (api-sandbox.dhl.com für Test, api-eu.dhl.com für Prod)
- createLabel: POST /shipments → gibt trackingNumber + labelUrl zurück
- Credentials aus shipping_providers.credentials: { apiKey, accountNumber }

4. Sendcloud-Integration: src/lib/shipping/providers/sendcloud.ts
- Sendcloud API v2 (panel.sendcloud.sc/api/v2)
- createLabel: POST /parcels → gibt tracking_number + label zurück
- Credentials: { publicKey, secretKey }

5. API-Routen:
- POST /api/shipping/label — erstellt Label für Order
- GET /api/shipping/[trackingNumber] — Tracking-Status
- GET /api/shipping/providers — listet aktive Provider des Tenants

6. UI: src/components/orders/ShippingButton.tsx
- Button in Order-Detail: "Label erstellen"
- Provider-Auswahl (wenn mehrere aktiv)
- Gewicht-Input (Gramm)
- Nach Erstellung: Tracking-Nummer anzeigen + Label-Download

### Sandbox zuerst
DHL und Sendcloud beide im Sandbox-Modus implementieren.
ENV-Variable SHIPPING_SANDBOX=true schaltet automatisch auf Sandbox-Endpoints.

---

## Session 3 — Repricing Engine (Amazon)

### Warum
Amazon-Händler passen Preise täglich oder stündlich an Konkurrenten an.
Ohne Repricing-Engine verlieren sie systematisch die Buy Box.

### Was bauen

1. Neue Supabase-Tabellen (Migration):
```sql
create table repricing_rules (
  id uuid primary key default gen_random_uuid(),
  tenant_id uuid references tenants(id) on delete cascade,
  channel_id uuid references channels(id) on delete cascade,
  name text not null,
  is_active boolean default true,
  strategy text not null check (strategy in ('match_lowest', 'beat_lowest', 'target_buybox', 'fixed_margin')),
  min_price numeric(10,2) not null,
  max_price numeric(10,2) not null,
  adjustment_value numeric(10,2) default 0.01,
  applies_to text default 'all' check (applies_to in ('all', 'selection')),
  created_at timestamptz default now()
);
alter table repricing_rules enable row level security;
create policy "tenant_isolation" on repricing_rules
  using (tenant_id = current_setting('app.current_tenant_id')::uuid);

create table repricing_log (
  id uuid primary key default gen_random_uuid(),
  tenant_id uuid references tenants(id) on delete cascade,
  rule_id uuid references repricing_rules(id),
  sku_id uuid,
  old_price numeric(10,2),
  new_price numeric(10,2),
  reason text,
  created_at timestamptz default now()
);
```

2. Repricing-Engine: src/lib/repricing/engine.ts
- Strategien:
  - match_lowest: Preis = niedrigster Konkurrent
  - beat_lowest: Preis = niedrigster Konkurrent - adjustment_value
  - target_buybox: Preis optimiert für Buy Box Gewinn (Amazon-spezifisch)
  - fixed_margin: Preis = Einkaufspreis × (1 + margin%)
- Sicherheitscheck: neuer Preis immer zwischen min_price und max_price
- Schreibt jeden Preischange in repricing_log

3. Amazon-Preisabfrage: src/lib/repricing/amazon-competitive.ts
- SP-API: GET /products/pricing/v0/competitivePricing
- Gibt niedrigsten Angebotspreis + Buy Box Preis zurück

4. Cron-Job: src/app/api/cron/repricing/route.ts
- Läuft stündlich (cron/crontab: 0 * * * *)
- Für jeden aktiven Tenant mit aktiver Repricing-Rule:
  - Hole aktuelle Konkurrenzpreise von Amazon
  - Berechne neuen Preis via Engine
  - Wenn Preisänderung > 0.01€: SP-API Preis-Update + repricing_log Eintrag
- withCronLock('cron:repricing', 55, ...) — max 55 Minuten Laufzeit

5. UI: src/app/(dashboard)/repricing/page.tsx
- Tabelle: aktive Rules mit Name, Strategie, Min/Max, letzter Preisaktion
- Button: Rule erstellen / bearbeiten (Modal)
- Repricing-Log Tab: letzte 100 Preisänderungen mit Grund

### Wichtig
- Nur für Amazon implementieren (hat die reichste Competitive Pricing API)
- eBay und Otto können später nachgezogen werden
- SP-API Credentials kommen aus channels-Tabelle (wie bestehende Amazon-Integration)
