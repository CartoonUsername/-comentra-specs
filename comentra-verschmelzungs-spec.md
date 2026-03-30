# Comentra — Spec: Verschmelzungs-Features
# Prinzip: Kein Feature wird isoliert gebaut.
# Jedes Feature verschmilzt mit dem was bereits existiert.
# KI, Cache, BullMQ, PWA, Retouren, CRM, Repricing — alles fließt zusammen.

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

## Session 1 — KI-Regel-Engine (Workflow-Builder)

### Wie Billbee es macht
Händler öffnet Regeleditor, tippt manuell:
"WENN Kanal = Amazon UND Betrag > 100 DANN Versandart = DHL Express"
Starres If-Then System. Neue Händler wissen nicht welche Regeln sie brauchen.
Ergebnis: 80% der Händler nutzen die Regel-Engine nie weil Lernkurve zu hoch.

### Wie Comentra es macht — verschmolzen
Die KI analysiert was der Tenant bereits tut (Bestellhistorie, Versandwahl,
Retourenmuster) und schlägt fertige Regeln vor.
"Ich sehe dass du Amazon-Bestellungen über 80€ immer mit DHL Express versendest.
Soll ich das automatisieren?" → Ein Klick → Regel aktiv.

Dazu: Regeln greifen in ALLE bestehenden Module ein:
- Retouren-Modul: "Wenn Retourengrund = defekt → automatisch Ersatz bestellen"
- Repricing: "Wenn Marge unter 15% → Repricing-Regel deaktivieren"
- Lieferanten: "Wenn Bestand unter Reorder Point → sofort PO erstellen"
- CRM: "Wenn Kunde hat 5+ Bestellungen → VIP-Tag + andere E-Mail-Template"
- Bundles: "Wenn Komponente ausverkauft → Bundle deaktivieren" (bereits gebaut,
  jetzt als Regel konfigurierbar)

### Was bauen

1. Migration: supabase/migrations/20260410_rules_engine.sql
```sql
create table workflow_rules (
  id uuid primary key default gen_random_uuid(),
  tenant_id uuid references tenants(id) on delete cascade,
  name text not null,
  description text,
  is_active boolean default true,
  trigger_event text not null check (trigger_event in (
    'order_created', 'order_status_changed', 'order_shipped',
    'inventory_updated', 'inventory_low', 'return_created',
    'return_classified', 'shipment_created', 'invoice_created',
    'customer_tagged', 'bundle_stock_changed', 'repricing_applied',
    'batch_expiring', 'purchase_order_created'
  )),
  conditions jsonb not null default '[]',
  actions jsonb not null default '[]',
  ai_suggested boolean default false,
  run_count int default 0,
  last_run_at timestamptz,
  created_at timestamptz default now()
);

create table workflow_rule_log (
  id uuid primary key default gen_random_uuid(),
  tenant_id uuid references tenants(id) on delete cascade,
  rule_id uuid references workflow_rules(id) on delete cascade,
  trigger_event text,
  trigger_data jsonb,
  actions_executed jsonb,
  success boolean default true,
  error_message text,
  executed_at timestamptz default now()
);

create index idx_rules_tenant_event on workflow_rules(tenant_id, trigger_event)
  where is_active = true;

alter table workflow_rules enable row level security;
alter table workflow_rule_log enable row level security;
create policy "tenant_isolation" on workflow_rules
  using (tenant_id = current_setting('app.current_tenant_id')::uuid);
create policy "tenant_isolation" on workflow_rule_log
  using (tenant_id = current_setting('app.current_tenant_id')::uuid);
```

2. Rule-Engine Core: src/lib/rules/rule-engine.ts
   - evaluate(tenantId, event, data): Lädt alle aktiven Regeln für Event
   - evaluateConditions(conditions, data): Prüft alle Bedingungen
   - executeActions(actions, data): Führt alle Aktionen aus
   
   Verfügbare Bedingungen:
   - order.total > X / < X / = X
   - order.channel = "amazon" / "ebay" / "otto"
   - order.country = "DE" / "AT" / "CH"
   - inventory.quantity < X
   - customer.order_count > X
   - customer.tags contains "VIP"
   - return.reason_category = "defective"
   - batch.days_until_expiry < X
   
   Verfügbare Aktionen (integriert in bestehende Module):
   - set_shipping_method(method) → bestehende Versand-API
   - send_email(template_id) → bestehende CRM E-Mail-Engine
   - tag_customer(tag) → Kunden-Tabelle
   - create_purchase_order(supplier_id) → bestehende PO-Engine
   - pause_repricing_rule(rule_id) → bestehende Repricing-Engine
   - send_notification(message) → bestehende Notification
   - add_order_note(text)
   - change_order_status(status)
   - deactivate_bundle(bundle_id) → bestehende Bundle-Engine
   - webhook(url, payload) → externe Systeme

3. KI-Regelvorschläge: src/lib/rules/ai-suggester.ts
   - Nutze claude-haiku-4-5-20251001 (bereits installiert)
   - Analysiert letzte 30 Tage Bestellhistorie des Tenants
   - Erkennt manuelle Muster: "DHL Express bei > 80€" etc.
   - Gibt 3-5 Regelvorschläge zurück mit Konfidenz-Score
   - Cron täglich 07:00: neue Vorschläge generieren, in DB speichern

4. Rule-Trigger Integration — in ALLE bestehenden Hooks einbauen:
   - src/app/api/orders/route.ts → trigger('order_created', order)
   - src/app/api/inventory/route.ts → trigger('inventory_updated', item)
   - src/lib/returns/auto-approve.ts → trigger('return_classified', return)
   - src/app/api/shipments/route.ts → trigger('shipment_created', shipment)
   - src/lib/inventory/fifo-engine.ts → trigger('batch_expiring', batch)
   - BullMQ: Rule-Execution in orders:high Queue (nicht blockierend)

5. API-Routen:
   - GET/POST /api/rules — Regel CRUD
   - GET /api/rules/suggestions — KI-Vorschläge
   - POST /api/rules/[id]/test — Test-Run mit Beispieldaten
   - GET /api/rules/log — Ausführungsprotokoll

6. UI: src/app/(app)/rules/page.tsx
   - Tab 1: Aktive Regeln — Liste mit Run-Count, letzter Ausführung, Toggle
   - Tab 2: KI-Vorschläge — Cards mit "Aktivieren" Button, Konfidenz-Badge
   - Tab 3: Regel erstellen — visueller Builder:
     Trigger auswählen (Dropdown) →
     Bedingungen (+ Button, AND/OR) →
     Aktionen (+ Button, mehrere möglich) →
     Test-Run Button →
     Speichern
   - Tab 4: Ausführungsprotokoll — letzte 100 Ausführungen mit Status
   - Navigation: Link zu /rules in Sidebar

### Verkaufsargument
"Billbee hat eine Regel-Engine die du selbst konfigurieren musst.
Comentra zeigt dir welche Regeln du brauchst — basierend darauf
was du bereits tust. Ein Klick und es läuft."

---

## Session 2 — Buchhaltungsintegration: Echtzeit statt Export

### Wie Billbee es macht
Export-Button → CSV/DATEV-Datei → manuell in sevDesk/lexoffice importieren.
Täglich oder wöchentlich manueller Schritt.

### Wie Comentra es macht — verschmolzen
Jede Rechnung die im Rechnungs-Modul (bereits gebaut) erstellt wird →
geht **automatisch in Echtzeit** zu lexoffice/sevDesk.
Kein Export. Kein Button. Kein Mensch.

Dazu: Zahlungseingang in lexoffice/sevDesk → automatisch Rechnung in
Comentra als "bezahlt" markiert → Mahnwesen stoppt automatisch.
Bidirektionale Echtzeit-Sync statt Einweg-Export.

### Was bauen

1. Migration: supabase/migrations/20260411_accounting.sql
```sql
create table accounting_integrations (
  id uuid primary key default gen_random_uuid(),
  tenant_id uuid references tenants(id) on delete cascade,
  provider text not null check (provider in ('lexoffice', 'sevdesk', 'fastbill', 'datev')),
  api_key text,
  is_active boolean default true,
  sync_invoices boolean default true,
  sync_customers boolean default true,
  sync_payments boolean default true,
  last_sync_at timestamptz,
  created_at timestamptz default now()
);

create table accounting_sync_log (
  id uuid primary key default gen_random_uuid(),
  tenant_id uuid references tenants(id) on delete cascade,
  integration_id uuid references accounting_integrations(id),
  entity_type text check (entity_type in ('invoice', 'customer', 'payment')),
  entity_id uuid,
  direction text check (direction in ('push', 'pull')),
  status text check (status in ('success', 'error')),
  error_message text,
  synced_at timestamptz default now()
);

alter table accounting_integrations enable row level security;
alter table accounting_sync_log enable row level security;
create policy "tenant_isolation" on accounting_integrations
  using (tenant_id = current_setting('app.current_tenant_id')::uuid);
create policy "tenant_isolation" on accounting_sync_log
  using (tenant_id = current_setting('app.current_tenant_id')::uuid);
```

2. Provider-Abstraktion: src/lib/accounting/base-provider.ts
   - Abstrakte Klasse AccountingProvider
   - Methoden: pushInvoice(invoice), pushCustomer(customer),
     pullPayments(), testConnection()

3. lexoffice-Integration: src/lib/accounting/providers/lexoffice.ts
   - API: api.lexoffice.io/v1
   - pushInvoice: POST /invoices (konvertiert Comentra-Format → lexoffice)
   - pushCustomer: POST /contacts
   - pullPayments: GET /voucherlist?voucherType=invoice&voucherStatus=paid
   - Webhook-Empfang: wenn lexoffice Zahlung bucht → Comentra Rechnung bezahlt

4. sevDesk-Integration: src/lib/accounting/providers/sevdesk.ts
   - API: my.sevdesk.de/api/v1
   - Gleiche Methoden wie lexoffice, anderes API-Format
   - sevDesk sendet Webhooks bei Zahlungseingang

5. Echtzeit-Trigger — in bestehende Module einbauen:
   - src/app/api/invoices/route.ts → nach Invoice-Create:
     pushInvoice() zu allen aktiven Integrationen (BullMQ listings:low)
   - Neuer Webhook-Endpunkt: POST /api/webhooks/accounting/[provider]
     → empfängt Zahlungsbestätigung → markiert Invoice als bezahlt
     → stoppt Mahnwesen → triggert Rule-Engine (invoice_paid Event)

6. Mahnwesen — verschmolzen mit CRM E-Mail-Engine:
   - Neues CRM-Template: 'payment_reminder_1', 'payment_reminder_2', 'payment_reminder_3'
   - Neue Regel in Rule-Engine: "Wenn Rechnung 7 Tage überfällig → Mahnstufe 1"
   - Wenn Zahlung eingeht (via Webhook) → Mahnkette automatisch gestoppt
   - Kein separates Mahnwesen-Modul nötig — Rule-Engine + CRM macht es

7. API-Routen:
   - GET/POST /api/accounting/integrations
   - POST /api/accounting/integrations/[id]/test
   - POST /api/accounting/integrations/[id]/sync — manueller Full-Sync
   - POST /api/webhooks/accounting/lexoffice
   - POST /api/webhooks/accounting/sevdesk

8. UI: src/app/(app)/settings/accounting/page.tsx
   - Provider-Karten: lexoffice, sevDesk, FastBill, DATEV
   - Pro Provider: API-Key eingeben, Verbindung testen, Sync-Optionen
   - Sync-Log: letzte 50 Synchronisierungen mit Status
   - Im Rechnungs-Modul: kleines Icon zeigt ob Rechnung zu lexoffice/sevDesk gesynct

### Verkaufsargument
"Billbee exportiert einmal täglich eine Datei die du manuell importierst.
Comentra synct jede Rechnung in Echtzeit — und wenn dein Steuerberater
die Zahlung in lexoffice bucht, ist Comentra sofort informiert."

---

## Session 3 — Zahlungsabgleich & Preisgruppen: verschmolzen

### Zahlungsabgleich — verschmolzen mit Buchhaltung (Session 2)
Durch den Webhook aus lexoffice/sevDesk ist Zahlungsabgleich bereits
automatisch gebaut. Zusätzlich: direkter Bankabgleich für Händler
ohne Buchhaltungssoftware.

### Was bauen

1. Preisgruppen: supabase/migrations/20260412_price_groups.sql
```sql
create table price_groups (
  id uuid primary key default gen_random_uuid(),
  tenant_id uuid references tenants(id) on delete cascade,
  name text not null,
  discount_percent numeric(5,2) default 0,
  markup_percent numeric(5,2) default 0,
  applies_to_channels text[] default '{}',
  created_at timestamptz default now()
);

create table customer_price_groups (
  customer_id uuid,
  price_group_id uuid references price_groups(id),
  tenant_id uuid references tenants(id) on delete cascade,
  primary key (customer_id, price_group_id)
);

alter table price_groups enable row level security;
create policy "tenant_isolation" on price_groups
  using (tenant_id = current_setting('app.current_tenant_id')::uuid);
```

2. Preisgruppen-Engine: src/lib/pricing/price-group-engine.ts
   - getEffectivePrice(customerId, skuId, basePrice): Wendet Rabatt an
   - Integriert in Rechnungserstellung: automatisch richtiger Preis
   - Integriert in Repricing: Preisgruppen-Rabatt vor Marktplatz-Preis
   - Verschmolzen mit Rule-Engine: "Wenn Kunde 10+ Bestellungen → VIP-Gruppe"

3. Direkter Zahlungsabgleich (für Händler ohne lexoffice/sevDesk):
   - Upload: Kontoauszug als CSV (CAMT.053 Format, deutsche Banken)
   - src/lib/accounting/bank-reconciliation.ts:
     Parsed CAMT.053 → matched Zahlungen mit offenen Rechnungen
     (Betrag + Verwendungszweck-Matching)
   - Ungematchte Zahlungen: manuelles Zuordnen per Drag & Drop
   - Verschmolzen mit Mahnwesen: gematche Zahlung → Mahnung automatisch gestoppt

4. API-Routen:
   - GET/POST /api/price-groups
   - POST /api/accounting/bank-reconciliation/upload
   - GET /api/accounting/bank-reconciliation/unmatched

5. UI:
   - src/app/(app)/settings/price-groups/page.tsx
     Preisgruppen erstellen, Kunden zuweisen (auch per Rule-Engine automatisch)
   - src/app/(app)/accounting/reconciliation/page.tsx
     Kontoauszug hochladen, gematchte/ungematchte Zahlungen, Drag & Drop

---

## Session 4 — Öffentliche REST-API

### Warum wichtig
Billbee hat eine API mit 100k Calls/Monat inklusive.
Enterprise-Kunden wollen Comentra in ihre eigenen Systeme integrieren:
SAP, eigene BI-Tools, Shopware-Plugins, eigene Apps.

### Verschmolzen
Die API nutzt dieselbe Auth, denselben Cache, dieselbe Rate-Limiting-
Infrastruktur die bereits für interne Routen gebaut wurde.
Per-Tenant Token-Bucket (bereits in src/lib/sync/rate-limiter.ts) wird
erweitert um API-Key-basiertes Rate Limiting.

### Was bauen

1. Migration: supabase/migrations/20260413_public_api.sql
```sql
create table api_keys (
  id uuid primary key default gen_random_uuid(),
  tenant_id uuid references tenants(id) on delete cascade,
  name text not null,
  key_hash text not null unique,
  key_prefix text not null,
  scopes text[] default '{"orders:read","products:read"}',
  calls_per_hour int default 1000,
  calls_today int default 0,
  total_calls bigint default 0,
  last_used_at timestamptz,
  expires_at timestamptz,
  is_active boolean default true,
  created_at timestamptz default now()
);

create table api_request_log (
  id uuid primary key default gen_random_uuid(),
  tenant_id uuid,
  api_key_id uuid references api_keys(id),
  method text,
  path text,
  status_code int,
  response_time_ms int,
  created_at timestamptz default now()
);

alter table api_keys enable row level security;
create policy "tenant_isolation" on api_keys
  using (tenant_id = current_setting('app.current_tenant_id')::uuid);
```

2. API-Key Middleware: src/middleware/api-key-auth.ts
   - Liest Authorization: Bearer ck_xxx Header
   - Hash-Vergleich (SHA-256, key nie im Klartext gespeichert)
   - Rate Limiting: Token-Bucket aus bestehender rate-limiter.ts erweitern
   - Setzt tenant_id context für RLS

3. Öffentliche API-Routen unter /api/v1/:
   - GET /api/v1/orders — Bestellungen (Filter: status, channel, date)
   - GET /api/v1/orders/[id] — Bestellung Details
   - PATCH /api/v1/orders/[id] — Status ändern
   - GET /api/v1/products — Produkte
   - GET /api/v1/products/[id]
   - GET /api/v1/inventory — Lagerbestand
   - PATCH /api/v1/inventory/[sku] — Bestand anpassen
   - GET /api/v1/shipments — Sendungen
   - POST /api/v1/shipments — Sendung erstellen
   - GET /api/v1/returns — Retouren
   - GET /api/v1/customers — Kunden
   - Alle Routen: Pagination (limit/offset), Filtering, Sorting

4. Webhooks (outbound):
   - Tenant kann Webhook-URL registrieren
   - Events: order.created, order.updated, shipment.created, return.created
   - Delivery: BullMQ listings:low Queue, 3 Retry-Versuche
   - Signature: HMAC-SHA256 Header für Webhook-Validierung (bereits in Security-Spec)

5. API-Dokumentation: src/app/api-docs/page.tsx
   - Interaktive Swagger/OpenAPI Dokumentation
   - Nutze swagger-ui-react (prüfe ob bereits installiert, sonst begründen)
   - Zeigt alle Endpunkte mit Beispiel-Requests und Responses

6. UI: src/app/(app)/settings/api/page.tsx
   - API-Key erstellen: Name, Scopes auswählen, Rate Limit setzen
   - Key wird einmalig im Klartext angezeigt (danach nie wieder)
   - Verwendungsstatistiken: Calls heute, diese Woche, Top-Endpunkte
   - Webhook-URLs verwalten

### Verkaufsargument
"Billbee hat eine API. Comentra hat eine API die denselben
Rate-Limiting-Schutz nutzt wie alles andere — und Webhooks die
dein System in Echtzeit informieren statt dass du pollen musst."
