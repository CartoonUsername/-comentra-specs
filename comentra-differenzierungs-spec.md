# Comentra — Spec: Differenzierungs-Features
> Diese 3 Features existieren bei Billbee, JTL, plentymarkets nicht in dieser Form.
> Sie sind Verkaufsargumente — nicht nur Funktionen.

---

## Globale Regeln (immer gültig)
1. Redis lokal (redis://redis:6379)
2. Supabase bleibt die DB — RLS aktiv
3. Next.js App Router
4. Keine neue Dependency ohne Begründung
5. Kein Breaking Change ohne Bestätigung
6. Nach jeder Session: Zusammenfassung (Dateien, Neustart ja/nein)
7. Warte nach jeder Session auf "weiter"

---

## Session 1 — KI-Retourenmanagement

### Warum das ein Verkaufsargument ist
Billbee: Retoure = manueller Prozess. Mitarbeiter öffnet Paket, entscheidet,
tippt Status ein. Bei 100 Retouren/Tag = 1 halber Arbeitstag verloren.

Comentra: Retoure kommt rein → KI klassifiziert Grund, entscheidet automatisch
ob Erstattung, Umtausch oder Ablehnung → Mitarbeiter bestätigt nur noch.
Verkaufsargument: "Retouren auf Autopilot — 80% weniger manueller Aufwand."

### Was bauen

1. Migration: supabase/migrations/20260404_returns.sql
```sql
create table returns (
  id uuid primary key default gen_random_uuid(),
  tenant_id uuid references tenants(id) on delete cascade,
  order_id uuid references orders(id),
  channel text not null,
  reason_raw text,
  reason_category text check (reason_category in (
    'defective', 'wrong_item', 'not_as_described', 'changed_mind',
    'damaged_shipping', 'too_late', 'other'
  )),
  ai_decision text check (ai_decision in ('refund', 'exchange', 'reject', 'manual_review')),
  ai_confidence numeric(3,2),
  ai_reasoning text,
  final_decision text check (final_decision in ('refund', 'exchange', 'reject')),
  decided_by uuid references auth.users(id),
  status text default 'pending' check (status in (
    'pending', 'ai_classified', 'awaiting_approval', 'approved', 'rejected', 'completed'
  )),
  refund_amount numeric(10,2),
  created_at timestamptz default now(),
  resolved_at timestamptz
);
alter table returns enable row level security;
create policy "tenant_isolation" on returns
  using (tenant_id = current_setting('app.current_tenant_id')::uuid);

create index idx_returns_tenant_status on returns(tenant_id, status);
create index idx_returns_tenant_created on returns(tenant_id, created_at desc);
```

2. KI-Klassifikation: src/lib/returns/ai-classifier.ts
- Nutze Anthropic SDK (bereits im Projekt: @anthropic-ai/sdk)
- Prompt-Struktur:
  ```
  System: Du bist ein Retouren-Spezialist für E-Commerce.
  Klassifiziere diese Retoure und entscheide über die beste Lösung.
  Antworte NUR als JSON: { category, decision, confidence, reasoning }
  
  Eingabe: { reason_raw, order_value, days_since_purchase,
             product_category, return_history_count }
  ```
- Model: claude-haiku-4-5-20251001 (schnell + günstig für Batch-Klassifikation)
- Confidence < 0.7 → decision = 'manual_review'
- Speichere Ergebnis in returns-Tabelle

3. Auto-Approval-Engine: src/lib/returns/auto-approve.ts
- Regeln (konfigurierbar pro Tenant):
  - confidence >= 0.9 + decision = 'refund' + order_value < 50€ → auto-approve
  - confidence >= 0.95 + decision = 'reject' → auto-reject mit Begründung
  - Alles andere → awaiting_approval (Mitarbeiter entscheidet)
- Schreibt final_decision + resolved_at

4. API-Routen:
- POST /api/returns — neue Retoure erfassen (triggert KI-Klassifikation via BullMQ)
- GET /api/returns — Liste mit Filter (status, channel, date)
- PATCH /api/returns/[id]/decide — Mitarbeiter überschreibt KI-Entscheidung

5. BullMQ Job: returns:classify (in orders:high Queue)
- Läuft nach POST /api/returns
- Ruft ai-classifier.ts auf
- Ruft auto-approve.ts auf
- Sendet optional E-Mail an Kunde (via bestehenden E-Mail-Service)

6. UI: src/app/(app)/returns/page.tsx
- 3-Spalten-Kanban: Pending → KI-Vorschlag → Entschieden
- Pro Karte: Bestellnummer, Grund, KI-Entscheidung mit Confidence-Badge (grün/gelb/rot)
- One-Click Approve / Reject / Override
- Filter: Channel, Status, Zeitraum

7. Dashboard-Widget: src/components/dashboard/ReturnsWidget.tsx
- Heute: X Retouren eingegangen, Y auto-approved, Z warten auf Review
- Auto-Approval-Rate in % (Verkaufsargument für Demo)

### Verkaufsargument in der UI sichtbar machen
Zeige prominent auf der Returns-Seite:
"KI hat heute X von Y Retouren automatisch bearbeitet — du hast Z Stunden gespart."

---

## Session 2 — Intelligentes Lieferantenmanagement

### Warum das ein Verkaufsargument ist
Billbee/JTL: Lieferantenliste = Excel in der App. Kein Händler weiß wann er
nachbestellen muss bis das Regal leer ist.

Comentra: System berechnet automatisch wann welcher Artikel beim welchem
Lieferanten bestellt werden muss — basierend auf Verkaufsgeschwindigkeit,
Lieferzeit und einstellbarem Sicherheitsbestand.
Verkaufsargument: "Nie wieder ausverkauft, nie wieder zu viel auf Lager."

### Was bauen

1. Migration: supabase/migrations/20260405_suppliers.sql
```sql
create table suppliers (
  id uuid primary key default gen_random_uuid(),
  tenant_id uuid references tenants(id) on delete cascade,
  name text not null,
  email text,
  phone text,
  lead_time_days int default 7,
  minimum_order_value numeric(10,2) default 0,
  currency text default 'EUR',
  notes text,
  is_active boolean default true,
  created_at timestamptz default now()
);

create table supplier_products (
  id uuid primary key default gen_random_uuid(),
  tenant_id uuid references tenants(id) on delete cascade,
  supplier_id uuid references suppliers(id) on delete cascade,
  sku_id uuid,
  supplier_sku text,
  purchase_price numeric(10,2),
  min_order_qty int default 1,
  reorder_point int,
  reorder_qty int,
  safety_stock int default 0,
  is_preferred boolean default false
);

create table purchase_orders (
  id uuid primary key default gen_random_uuid(),
  tenant_id uuid references tenants(id) on delete cascade,
  supplier_id uuid references suppliers(id),
  status text default 'draft' check (status in (
    'draft', 'sent', 'confirmed', 'partial', 'received', 'cancelled'
  )),
  expected_delivery date,
  total_value numeric(10,2),
  notes text,
  created_at timestamptz default now(),
  sent_at timestamptz,
  received_at timestamptz
);

create table purchase_order_items (
  id uuid primary key default gen_random_uuid(),
  purchase_order_id uuid references purchase_orders(id) on delete cascade,
  tenant_id uuid references tenants(id) on delete cascade,
  sku_id uuid,
  quantity_ordered int not null,
  quantity_received int default 0,
  purchase_price numeric(10,2)
);

alter table suppliers enable row level security;
alter table supplier_products enable row level security;
alter table purchase_orders enable row level security;
alter table purchase_order_items enable row level security;

create policy "tenant_isolation" on suppliers using (tenant_id = current_setting('app.current_tenant_id')::uuid);
create policy "tenant_isolation" on supplier_products using (tenant_id = current_setting('app.current_tenant_id')::uuid);
create policy "tenant_isolation" on purchase_orders using (tenant_id = current_setting('app.current_tenant_id')::uuid);
create policy "tenant_isolation" on purchase_order_items using (tenant_id = current_setting('app.current_tenant_id')::uuid);
```

2. Nachbestellungs-Engine: src/lib/suppliers/reorder-engine.ts
- Berechnet pro SKU:
  - Verkaufsgeschwindigkeit: Ø Units/Tag der letzten 30 Tage (aus orders)
  - Days of Stock: aktueller Bestand ÷ Verkaufsgeschwindigkeit
  - Reorder Point: (Lieferzeit × Verkaufsgeschwindigkeit) + Safety Stock
  - Empfohlene Bestellmenge: Reorder Qty aus supplier_products
- Gibt Liste zurück: { sku_id, days_of_stock, should_reorder, suggested_qty, supplier }

3. Auto-Draft Purchase Orders: src/lib/suppliers/po-generator.ts
- Wenn should_reorder = true → erstellt draft Purchase Order automatisch
- Gruppiert nach Lieferant (1 PO pro Lieferant)
- Prüft minimum_order_value — fügt fehlende Items hinzu wenn nötig

4. Cron: src/app/api/cron/reorder-check/route.ts
- Täglich 08:00 Uhr (nach Pre-warming)
- Für alle aktiven Tenants mit supplier_products
- Ruft reorder-engine.ts auf
- Erstellt Draft-POs via po-generator.ts
- Sendet Zusammenfassung per E-Mail: "3 Artikel müssen nachbestellt werden"

5. API-Routen:
- GET/POST /api/suppliers — Lieferantenverwaltung
- GET/POST /api/suppliers/[id]/products — Lieferantenprodukte
- GET/POST /api/purchase-orders — Bestellungen
- PATCH /api/purchase-orders/[id]/send — PO per E-Mail an Lieferant senden
- PATCH /api/purchase-orders/[id]/receive — Wareneingang buchen (updated Inventory)

6. UI: src/app/(app)/suppliers/page.tsx
- Tab 1: Lieferantenliste
- Tab 2: Nachbestellvorschläge (heute generiert) — mit 1-Click "Alle bestellen"
- Tab 3: Offene Bestellungen (Kanban: Draft → Sent → Confirmed → Received)

7. Inventory-Integration
- Wenn PO status → 'received': automatisch Inventory erhöhen
- Nutze bestehenden Inventory-Update-Flow + Write Coalescer

---

## Session 3 — Progressive Web App (PWA)

### Warum das ein Verkaufsargument ist
Alle Konkurrenten: nur Desktop-Browser.
Lagerarbeiter haben keinen Desktop — sie haben ein Smartphone oder Scanner.

Comentra PWA: Installierbar auf jedem Smartphone, funktioniert offline,
hat Barcode-Scanner via Kamera. Kein App Store, kein Update-Prozess.
Verkaufsargument: "Comentra läuft auf jedem Gerät — auch im Lager ohne WLAN."

### Was bauen

1. PWA-Grundkonfiguration:
- next-pwa installieren (oder manuell service-worker.js — prüfe was besser passt)
- public/manifest.json:
  ```json
  {
    "name": "Comentra",
    "short_name": "Comentra",
    "start_url": "/",
    "display": "standalone",
    "background_color": "#F7F5F0",
    "theme_color": "#C9521A",
    "icons": [...]
  }
  ```
- Service Worker: Cache-Strategie
  - Static Assets: Cache First (1 Woche)
  - API GET-Requests: Network First mit Fallback auf Cache (offline-fähig)
  - API POST/PATCH: Queue (Background Sync wenn wieder online)

2. Offline-fähige Kern-Seiten:
- /dashboard — letzte bekannte Daten aus Cache
- /inventory — Bestandsabfrage offline möglich
- /orders — letzte 50 Orders offline lesbar

3. Offline-Indicator: src/components/ui/OfflineBar.tsx
- Zeigt Banner wenn offline: "Offline — Änderungen werden synchronisiert sobald Verbindung besteht"
- Verschwindet automatisch wenn wieder online

4. Barcode-Scanner: src/components/scanner/BarcodeScanner.tsx
- Nutze @zxing/library (bereits im Projekt laut Session 8)
- Öffnet Kamera, scannt EAN/QR in Echtzeit
- Verwendung:
  - Inventory-Seite: SKU scannen → Bestand anzeigen / anpassen
  - Wareneingang: PO scannen → Menge eingeben → Inventory buchen

5. Mobile-optimierte Seiten:
- src/app/(app)/mobile/inventory/page.tsx — vereinfachte Inventur-Ansicht
  - Großer Scan-Button mittig
  - Nach Scan: Produktname + aktueller Bestand + Input für neue Menge
  - Confirm-Button → PATCH /api/inventory
- src/app/(app)/mobile/receiving/page.tsx — Wareneingang
  - PO auswählen (Dropdown)
  - Artikel scannen oder manuell abhaken
  - "Wareneingang buchen" Button

6. Install-Prompt: src/components/pwa/InstallPrompt.tsx
- Zeige einmalig nach Login: "Comentra als App installieren?"
- "Auf Homescreen" Button → triggert beforeinstallprompt
- Speichere Dismissed-State in localStorage

7. Push Notifications (optional, nur wenn Zeit):
- Neue Retoure eingegangen → Push an zuständigen Mitarbeiter
- Nachbestellvorschlag generiert → Push an Einkäufer
- Nutze Web Push API + VAPID Keys (in .env)

### Wichtig
PWA funktioniert nur über HTTPS — Traefik macht das bereits.
Teste auf echtem Smartphone (nicht nur Browser DevTools).
Android Chrome hat beste PWA-Unterstützung — iOS Safari hat Einschränkungen
bei Background Sync, dokumentiere das in einem Kommentar.
