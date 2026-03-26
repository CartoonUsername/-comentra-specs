# Claude Code Prompt — Comentra Order-Sync Service

Baue einen Amazon SP-API Order-Sync Service für die Comentra SaaS-Plattform.
Laufumgebung: Hostinger VPS, Node.js + PM2, kein n8n.

Ziel: Amazon.de Bestellungen eines Tenants automatisch alle 5 Minuten 
in Supabase einspielen. Der Service ist pro Tenant instanziierbar —
jeder Tenant hat eigene Amazon Credentials und eine eigene TENANT_ID.

## Stack
- Node.js (ES Modules)
- amazon-sp-api (npm package)
- @supabase/supabase-js
- node-cron für Polling
- dotenv für Credentials
- PM2 ecosystem.config.cjs für Autostart

## Supabase Tabellen (exakte Struktur verwenden)

orders:
  id, tenant_id, channel_id, customer_id, channel_order_id (text, Amazon Order ID),
  status, payment_status, currency, subtotal, shipping_cost, tax_amount,
  discount_amount, total, shipping_address (jsonb), billing_address (jsonb),
  notes, tags (array), metadata (jsonb), ordered_at, paid_at, created_at, updated_at

order_items:
  id, tenant_id, order_id, variant_id, listing_id, name, sku, qty,
  unit_price, tax_rate, total, status, purchase_price, discount_amount, created_at

## .env Variablen (Struktur vorgeben, keine echten Werte)
SUPABASE_URL=
SUPABASE_SERVICE_KEY=
TENANT_ID=
CHANNEL_ID=
AMAZON_REFRESH_TOKEN=
AMAZON_CLIENT_ID=
AMAZON_CLIENT_SECRET=
AMAZON_MARKETPLACE_ID=A1PA6795UKMFR9

## Logik

1. Cron alle 5 Minuten: Amazon SP-API getOrders() aufrufen
   - Filter: LastUpdatedAfter = last_sync timestamp (in sync_state.json)
   - OrderStatuses: Unshipped, PartiallyShipped, Shipped, Canceled

2. Pro Order:
   - Prüfe ob channel_order_id bereits existiert (skip wenn ja, update wenn Status geändert)
   - Map Amazon Status → Comentra Status:
     Unshipped → "confirmed"
     Shipped → "shipped"
     Canceled → "cancelled"
   - Map PaymentStatus:
     "PaymentComplete" → "paid"
     sonst → "pending"
   - Alle Queries immer mit tenant_id gefiltert (RLS-sicher)
   - Upsert in orders Tabelle

3. Pro OrderItem (getOrderItems()):
   - Upsert in order_items (match auf order_id + sku)
   - tax_rate = 19 als Default

4. Fehlerbehandlung:
   - Try/catch pro Order (eine fehlerhafte Order stoppt nicht den Rest)
   - Konsolenlog mit Timestamp pro Sync-Lauf
   - Bei SP-API Rate Limit (429): exponential backoff

5. sync_state.json speichert last_sync Timestamp pro Tenant

## Dateistruktur
comentra-order-sync/
  index.js              (Cron + Orchestrierung)
  amazon.js             (SP-API Wrapper)
  supabase.js           (Upsert Logik, immer tenant_id mitgeben)
  mapper.js             (Amazon → Comentra Mapping)
  sync_state.json       (automatisch erstellt)
  .env.example
  ecosystem.config.cjs
  package.json

Kommentare auf Deutsch. Kein TypeScript. Kein Express.
Multi-Tenant-Gedanke: tenant_id ist immer Pflichtfeld bei allen 
Supabase-Operationen, damit spätere Tenants sauber isoliert sind.
Am Ende: Anleitung für PM2 Start auf Hostinger VPS.
