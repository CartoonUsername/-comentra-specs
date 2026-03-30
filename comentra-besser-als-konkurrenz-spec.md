# Comentra — Spec: 4 Features besser als Konkurrenz
> Jedes Feature wird nicht nur gebaut — es wird so gebaut dass es im Demo
> sofort gewinnt. "Besser" ist definiert bevor der erste Code geschrieben wird.

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

## Session 1 — Retourenlabels: vollautomatische Kette

### Wie Billbee es macht
Mitarbeiter öffnet Retoure → klickt "Label erstellen" → wählt Carrier →
druckt PDF → schickt manuell an Kunden. 4 manuelle Schritte.

### Wie Comentra es macht (besser)
Retoure eingehend → KI klassifiziert (bereits gebaut) → Auto-Approve →
**Label wird automatisch generiert** → **E-Mail geht automatisch an Kunden** →
Status springt auf "Label versendet". 0 manuelle Schritte.

Mitarbeiter sieht nur noch: "✅ Label automatisch versendet an kunde@email.de"

### Was bauen

1. Label-Generator: src/lib/returns/label-generator.ts
   - Liest Bestellung → ermittelt Kanal (Amazon/eBay/Otto/DHL/Sendcloud)
   - Amazon: Nutzt SP-API /returns/label Endpunkt → Base64 PDF
   - eBay: eBay Returns API → Label URL
   - Alle anderen: DHL oder Sendcloud über bestehende
     src/lib/shipping/providers/ (bereits gebaut in Session 2)
   - Gibt zurück: { label_url, tracking_number, carrier, expires_at }

2. E-Mail-Versand: src/lib/returns/return-email.ts
   - Template auf Deutsch:
     Betreff: "Ihr Rücksendeetikett für Bestellung #{{order_number}}"
     Inhalt: Grund der Annahme, Anleitung, Label als PDF-Anhang,
     Tracking-Nummer, Frist (14 Tage)
   - Nutze bestehenden E-Mail-Service im Projekt

3. Automatische Trigger-Chain: src/lib/returns/auto-approve.ts ERWEITERN
   - Nach auto_decision = 'refund' oder 'exchange':
     → label-generator.ts aufrufen
     → return-email.ts aufrufen
     → returns.status = 'label_sent'
     → returns.label_url + tracking_number speichern

4. Neue Spalten in returns (Migration):
   ```sql
   alter table returns
     add column label_url text,
     add column tracking_number text,
     add column label_sent_at timestamptz,
     add column label_carrier text;
   ```

5. UI-Update: src/app/(app)/returns/page.tsx
   - Neue Spalte "Label" in der Tabelle
   - Status "Label versendet" mit Tracking-Nummer und Download-Link
   - Manuelle "Label erstellen" Button für Manual-Review Fälle
   - Dashboard-Widget zeigt: "Heute X Labels automatisch versendet"

### Verkaufsargument
"Bei Billbee braucht dein Mitarbeiter 4 Klicks pro Retoure.
Bei Comentra: 0 Klicks — das System macht es alleine."

---

## Session 2 — Bundles & Stücklisten: intelligent & live

### Wie JTL/plentymarkets es macht
Statisches Bundle: A + B = Bundle C. Wenn A ausverkauft ist, merkt das
niemand bis der erste Überverkauf passiert.

### Wie Comentra es macht (besser)
Dynamisches Bundle: Wenn Komponente A unter Mindestbestand fällt →
Bundle wird **automatisch auf allen Kanälen deaktiviert** → Händler
bekommt Push-Notification → Bundle reaktiviert sich automatisch wenn
A wieder auf Lager ist.

Dazu: eingebauter **Margen-Rechner** der zeigt ob Bundle profitabler
ist als Einzelverkauf (kein Konkurrent hat das).

### Was bauen

1. Migration: supabase/migrations/20260404_bundles.sql
```sql
create table bundles (
  id uuid primary key default gen_random_uuid(),
  tenant_id uuid references tenants(id) on delete cascade,
  name text not null,
  sku text not null unique,
  sale_price numeric(10,2),
  is_active boolean default true,
  auto_deactivate boolean default true,
  created_at timestamptz default now()
);

create table bundle_components (
  id uuid primary key default gen_random_uuid(),
  bundle_id uuid references bundles(id) on delete cascade,
  tenant_id uuid references tenants(id) on delete cascade,
  sku_id uuid not null,
  quantity int not null default 1
);

alter table bundles enable row level security;
alter table bundle_components enable row level security;
create policy "tenant_isolation" on bundles
  using (tenant_id = current_setting('app.current_tenant_id')::uuid);
create policy "tenant_isolation" on bundle_components
  using (tenant_id = current_setting('app.current_tenant_id')::uuid);
```

2. Bundle-Engine: src/lib/bundles/bundle-engine.ts
   - calculateBundleStock(bundleId): Min-Bestand aller Komponenten
     (Bundle kann nur so oft verkauft werden wie die knappste Komponente)
   - calculateBundleMargin(bundleId): Vergleich sale_price vs
     Summe der Einzelpreise → Marge in % und €
   - checkAndDeactivate(tenantId): Wenn Komponente unter Mindestbestand:
     Bundle deaktivieren + alle Channel-Listings updaten
   - checkAndReactivate(tenantId): Wenn alle Komponenten wieder ok:
     Bundle automatisch reaktivieren

3. Inventory-Hook: Erweitere bestehenden Inventory-Update-Flow
   - Nach jedem Inventory-Update: bundle-engine.checkAndDeactivate()
   - Schreibt Bundle-Status-Änderungen in repricing_log (Wiederverwendung)

4. API-Routen:
   - GET/POST /api/bundles — Bundle CRUD
   - GET /api/bundles/[id]/margin — Margen-Berechnung live
   - GET /api/bundles/[id]/stock — verfügbarer Bestand

5. UI: src/app/(app)/bundles/page.tsx
   - Bundle-Liste mit: Name, Komponenten-Count, aktueller Bestand,
     Marge in %, Status (Aktiv/Auto-deaktiviert/Manuell pausiert)
   - Bundle erstellen: Komponenten per Drag & Drop zusammenstellen,
     Mengen eingeben, Sale-Price eingeben
   - Live-Vorschau: Margen-Rechner aktualisiert sich beim Tippen
   - Auto-Deactivate Toggle pro Bundle

6. Navigation: Link zu /bundles in der Sidebar ergänzen

### Verkaufsargument
"JTL zeigt dir ein Bundle. Comentra sagt dir ob das Bundle
gerade verfügbar ist, wie viel du daran verdienst —
und deaktiviert es automatisch wenn eine Komponente ausgeht."

---

## Session 3 — CRM E-Mails: intelligent & automatisch

### Wie Billbee es macht
Statische Templates mit {{Variablen}}. Alle Kunden bekommen
die gleiche E-Mail zur gleichen Zeit.

### Wie Comentra es macht (besser)
**KI-personalisierte E-Mails** die sich nach Kanal, Bestellwert und
Kundenhistorie unterscheiden. Dazu: **Smart Timing** — E-Mails werden
nicht um 3 Uhr nachts verschickt, sondern zur optimalen Uhrzeit.
Plus: **automatische Sprache** basierend auf Marktplatz
(Amazon.fr → Französisch, Amazon.de → Deutsch).

### Was bauen

1. Migration: supabase/migrations/20260405_crm_emails.sql
```sql
create table email_templates (
  id uuid primary key default gen_random_uuid(),
  tenant_id uuid references tenants(id) on delete cascade,
  trigger_event text not null check (trigger_event in (
    'order_confirmed', 'order_shipped', 'order_delivered',
    'return_approved', 'return_label_sent', 'review_request',
    'reorder_reminder', 'bundle_back_in_stock'
  )),
  channel text default 'all',
  subject text not null,
  body_html text not null,
  is_active boolean default true,
  send_delay_hours int default 0,
  smart_timing boolean default true,
  created_at timestamptz default now()
);

create table email_log (
  id uuid primary key default gen_random_uuid(),
  tenant_id uuid references tenants(id) on delete cascade,
  order_id uuid,
  template_id uuid references email_templates(id),
  recipient_email text,
  subject text,
  status text default 'queued' check (status in ('queued','sent','failed')),
  scheduled_for timestamptz,
  sent_at timestamptz,
  created_at timestamptz default now()
);

alter table email_templates enable row level security;
alter table email_log enable row level security;
create policy "tenant_isolation" on email_templates
  using (tenant_id = current_setting('app.current_tenant_id')::uuid);
create policy "tenant_isolation" on email_log
  using (tenant_id = current_setting('app.current_tenant_id')::uuid);
```

2. Smart-Timing-Engine: src/lib/crm/smart-timing.ts
   - Erlaubte Sendezeiten: 08:00–20:00 Uhr Empfänger-Zeitzone
   - Wenn Trigger außerhalb: nächster erlaubter Slot
   - Optimale Zeiten per Typ:
     order_confirmed → sofort (immer)
     order_shipped → sofort (immer)
     review_request → +7 Tage nach Lieferung, 10:00 Uhr
     reorder_reminder → +30 Tage, 09:00 Uhr

3. KI-Personalisierung: src/lib/crm/ai-personalizer.ts
   - Nutze claude-haiku-4-5-20251001 (günstig, bereits installiert)
   - Input: Template + { order_value, customer_name, items,
     channel, previous_orders_count }
   - Output: personalisierter Subject + Body
   - Nur für premium Templates (opt-in per Template-Einstellung)
   - Sprache: automatisch nach Marketplace-Land

4. E-Mail-Dispatcher: src/lib/crm/dispatcher.ts
   - Liest email_log WHERE status='queued' AND scheduled_for <= now()
   - Sendet via bestehendem E-Mail-Service
   - Schreibt status='sent' + sent_at

5. Cron: src/app/api/cron/email-dispatch/route.ts
   - Alle 5 Minuten (*/5 * * * *)
   - Ruft dispatcher.ts auf
   - withCronLock('cron:email-dispatch', 4, ...)

6. Trigger-Integration:
   - order_confirmed: in bestehende Order-Sync Routen einbauen
   - order_shipped: in Sendungen-Status-Update einbauen
   - return_label_sent: in return-email.ts aus Session 1

7. UI: src/app/(app)/crm/page.tsx
   - Tab 1: Templates — Liste + Editor (Subject, Body mit {{Variablen}},
     Trigger-Event, Smart Timing Toggle, KI-Personalisierung Toggle)
   - Tab 2: E-Mail-Log — letzte 100 gesendete E-Mails mit Status
   - Tab 3: Statistiken — Öffnungsrate (wenn möglich), Volumen pro Tag
   - Navigation: Link zu /crm in Sidebar

### Verkaufsargument
"Billbee schickt die gleiche E-Mail an alle zur gleichen Zeit.
Comentra schickt die richtige E-Mail, zur richtigen Zeit,
in der richtigen Sprache — automatisch."

---

## Session 4 — Seriennummern & Chargen: scan-basiert

### Wie JTL es macht
Manuelle Eingabe im Desktop-Tool. Kein Lagerarbeiter macht das
freiwillig. In der Praxis: niemand pflegt es.

### Wie Comentra es macht (besser)
**Scan-basiert über PWA** (bereits gebaut) — Lagerarbeiter scannt
Barcode beim Wareneingang, Seriennummer wird automatisch erfasst.
Beim Versand: System schlägt automatisch die älteste Charge vor
(FIFO) und warnt wenn MHD in 30 Tagen abläuft.

### Was bauen

1. Migration: supabase/migrations/20260406_serial_batches.sql
```sql
create table batches (
  id uuid primary key default gen_random_uuid(),
  tenant_id uuid references tenants(id) on delete cascade,
  sku_id uuid not null,
  batch_number text not null,
  best_before_date date,
  quantity_received int not null,
  quantity_remaining int not null,
  received_at timestamptz default now(),
  purchase_order_id uuid references purchase_orders(id),
  created_at timestamptz default now()
);

create table serial_numbers (
  id uuid primary key default gen_random_uuid(),
  tenant_id uuid references tenants(id) on delete cascade,
  sku_id uuid not null,
  serial_number text not null,
  batch_id uuid references batches(id),
  status text default 'in_stock' check (status in (
    'in_stock', 'reserved', 'shipped', 'returned', 'defective'
  )),
  order_id uuid references orders(id),
  received_at timestamptz default now(),
  shipped_at timestamptz
);

create index idx_batches_tenant_sku on batches(tenant_id, sku_id);
create index idx_batches_best_before on batches(best_before_date)
  where best_before_date is not null;
create index idx_serials_tenant_sku on serial_numbers(tenant_id, sku_id, status);

alter table batches enable row level security;
alter table serial_numbers enable row level security;
create policy "tenant_isolation" on batches
  using (tenant_id = current_setting('app.current_tenant_id')::uuid);
create policy "tenant_isolation" on serial_numbers
  using (tenant_id = current_setting('app.current_tenant_id')::uuid);
```

2. FIFO-Engine: src/lib/inventory/fifo-engine.ts
   - getNextBatch(tenantId, skuId, qty): gibt älteste verfügbare
     Charge zurück (ORDER BY received_at ASC)
   - reserveBatch(batchId, qty, orderId): reserviert Menge
   - checkExpiringBatches(tenantId): Chargen die in 30 Tagen ablaufen

3. MHD-Alert Cron: src/app/api/cron/mhd-alert/route.ts
   - Täglich 07:30 Uhr
   - Findet alle Chargen mit best_before_date <= now() + 30 days
   - Erstellt Notification + optional E-Mail an Tenant-Admin

4. PWA-Erweiterung: src/app/(app)/mobile/receiving/page.tsx ERWEITERN
   - Nach Scan einer SKU: Option "Chargennummer scannen/eingeben"
   - Option "MHD eingeben" (Datepicker)
   - Option "Seriennummern scannen" (Loop bis Menge erreicht)
   - Speichert in batches + serial_numbers

5. Versand-Integration: Bei Order-Pick im WMS
   - FIFO-Engine schlägt automatisch Charge vor
   - Warnung wenn Charge in <30 Tagen abläuft
   - Nach Versand: serial_numbers.status = 'shipped'

6. UI: src/app/(app)/inventory/batches/page.tsx
   - Tabelle: SKU, Charge, MHD, Bestand, Alter
   - Rot markiert: MHD < 30 Tage
   - Filter: Nur ablaufende, Nur bestimmte SKU
   - Export als CSV für DATEV/Steuerberater

### Verkaufsargument
"JTL braucht manuelle Eingabe am Desktop — in der Praxis macht das
niemand. Comentra scannt beim Wareneingang automatisch —
dein Lagerarbeiter braucht 3 Sekunden pro Artikel."

---

## Reihenfolge
Session 1 → 2 → 3 → 4
Nach jeder Session: "weiter"
Migrations immer mit supabase db push anwenden.
