# Comentra — Channel Inventory Engine

## WICHTIG — REGELN

- Erstelle NUR neue Dateien
- Berühre KEINE bestehenden Dateien
- Keine Änderungen an App.tsx, router, types.ts, package.json, MasterProductCard.tsx
- Wenn du eine bestehende Datei anfassen willst: STOPP und frage mich zuerst
- Zeige mir zuerst welche neuen Dateien du anlegen wirst
- createReservation() MUSS als Supabase RPC Function implementiert werden — kein direktes SELECT FOR UPDATE aus TypeScript

---

## Ziel

Multi-Channel Inventory Engine mit Row Locking, Channel Buffern und automatischer Sync-Logik.
Alles additiv — keine bestehende Datei wird verändert.

---

## Neue Dateien

```
src/
  lib/
    inventoryEngine/
      index.ts
      calculateAvailable.ts
      createReservation.ts       ← ruft Supabase RPC auf
      releaseReservation.ts
      syncInventory.ts
      types.ts
  jobs/
    inventoryCron.ts             ← alle 60 Sekunden, standalone
  components/
    inventory/
      ChannelInventoryCard.tsx   ← standalone, nicht in Router
      InventoryStatusBadge.tsx
supabase/
  migrations/
    20260326_inventory_engine.sql
  functions/
    create_reservation/
      index.ts                   ← Supabase Edge Function mit Row Lock
```

---

## Database Migrations

```sql
alter table product_channel_inventory
  add column if not exists buffer integer default 0,
  add column if not exists reserved integer default 0,
  add column if not exists last_sync_at timestamptz,
  add column if not exists sync_status text default 'pending',
  add column if not exists sync_error text;

create table if not exists inventory_reservations (
  id uuid primary key default gen_random_uuid(),
  tenant_id uuid references tenants(id),
  product_id uuid references products(id),
  channel text not null,
  quantity integer not null,
  order_id text,
  expires_at timestamptz not null,
  created_at timestamptz default now()
);

create table if not exists inventory_events (
  id uuid primary key default gen_random_uuid(),
  tenant_id uuid references tenants(id),
  product_id uuid references products(id),
  channel text,
  type text,
  quantity integer,
  metadata jsonb,
  created_at timestamptz default now()
);

alter table inventory_reservations enable row level security;
alter table inventory_events enable row level security;

create policy "tenant_isolation_reservations" on inventory_reservations
  using (tenant_id = auth.uid());

create policy "tenant_isolation_events" on inventory_events
  using (tenant_id = auth.uid());
```

---

## Supabase RPC Function — create_reservation

MUSS als PostgreSQL Funktion implementiert werden wegen Row Locking:

```sql
create or replace function create_reservation(
  p_tenant_id uuid,
  p_product_id uuid,
  p_channel text,
  p_quantity integer,
  p_order_id text,
  p_expires_at timestamptz
) returns jsonb
language plpgsql
as $$
declare
  v_stock integer;
  v_reserved integer;
  v_buffer integer;
  v_available integer;
  v_reservation_id uuid;
begin
  select stock into v_stock
  from products
  where id = p_product_id and tenant_id = p_tenant_id
  for update;

  select coalesce(sum(quantity), 0) into v_reserved
  from inventory_reservations
  where product_id = p_product_id and expires_at > now();

  select coalesce(buffer, 0) into v_buffer
  from product_channel_inventory
  where product_id = p_product_id and channel = p_channel;

  v_available := v_stock - v_reserved - v_buffer;

  if v_available < p_quantity then
    return jsonb_build_object(
      'success', false,
      'reason', 'insufficient_stock',
      'available', v_available
    );
  end if;

  insert into inventory_reservations (
    tenant_id, product_id, channel, quantity, order_id, expires_at
  ) values (
    p_tenant_id, p_product_id, p_channel, p_quantity, p_order_id, p_expires_at
  ) returning id into v_reservation_id;

  return jsonb_build_object(
    'success', true,
    'reservation_id', v_reservation_id,
    'available_after', v_available - p_quantity
  );
end;
$$;
```

---

## calculateAvailable.ts

```ts
export function calculateAvailable(
  globalStock: number,
  reservations: number,
  buffer: number
): number {
  return Math.max(0, globalStock - reservations - buffer)
}
```

---

## createReservation.ts

Ruft RPC auf — kein direktes SQL:

```ts
export async function createReservation(params: {
  tenant_id: string
  product_id: string
  channel: string
  quantity: number
  order_id: string
  expires_at: string
}): Promise<{ success: boolean; reason?: string; available?: number }>
```

Wenn success: false → Order als stock_conflict markieren.

---

## syncInventory.ts

```ts
export async function syncInventory(product_id: string, tenant_id: string): Promise<void>
```

Ablauf:
1. Bestand und Reservierungen laden
2. calculateAvailable() pro Kanal mit jeweiligem Buffer
3. product_channel_inventory updaten
4. n8n Webhook triggern (URL aus bestehenden Channel Settings lesen)
5. sync_status und last_sync_at updaten
6. Event loggen

---

## inventoryCron.ts (Standalone)

Läuft alle 60 Sekunden:
1. Abgelaufene Reservierungen löschen
2. Bestand pro Produkt neu berechnen
3. Kanäle sync wenn sync_status = pending
4. Events loggen

Nicht automatisch registriert.

---

## ChannelInventoryCard.tsx (Standalone)

Props: productId: string, channel: string

Zeigt:
- Globaler Bestand
- Buffer
- Reserviert
- Verfügbar (grün > 5, gelb 1-5, rot 0)
- Sync Status Badge
- Last Sync

Nicht in Router eingetragen.

---

## Order Flow

```
Order importiert
  → createReservation()
    → success: syncInventory() alle Kanäle
    → fail:    order.status = stock_conflict
```

---

## Safety

- Math.max(0, ...) überall — kein negativer Bestand
- tenant_id in jeder Query
- Row Lock nur über PostgreSQL RPC Function
- Kein direktes FOR UPDATE aus TypeScript

---

## Liefere

1. supabase/migrations/20260326_inventory_engine.sql
2. PostgreSQL RPC Function create_reservation
3. src/lib/inventoryEngine/types.ts
4. src/lib/inventoryEngine/calculateAvailable.ts
5. src/lib/inventoryEngine/createReservation.ts
6. src/lib/inventoryEngine/releaseReservation.ts
7. src/lib/inventoryEngine/syncInventory.ts
8. src/lib/inventoryEngine/index.ts
9. src/jobs/inventoryCron.ts
10. src/components/inventory/ChannelInventoryCard.tsx
11. src/components/inventory/InventoryStatusBadge.tsx

Keine Änderungen an bestehenden Dateien.
Alle Dateien vollständig, kein Platzhalter-Code.
