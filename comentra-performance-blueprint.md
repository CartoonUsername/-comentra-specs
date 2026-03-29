# Comentra — Performance Blueprint
> 4 CPU / 8–16 GB RAM · 1 VPS · Supabase Managed · Präventiv vor Go-Live

---

## Legende
- ✅ bereits aktiv
- 🔧 zu implementieren
- 🚨 kritische Lücke
- 🟣 einzigartig für Comentra

---

## Layer 1 — Client (React)

### 🔧 Code Splitting
Lazy-load pro Route via `React.lazy()`. Vendor-Chunk von App-Code trennen.
- **Ziel:** Initial-Bundle unter 150kb
- Bei 200k Tenants × Mitarbeiter laden viele gleichzeitig — jedes KB zählt
- Vite: `build.rollupOptions.output.manualChunks` konfigurieren

### ✅ React Query — staleTime
- Produktdaten: `staleTime 5min`
- Inventory: `30s`
- Bestellungen: `10s`
- User sieht nie einen Spinner — Daten kommen aus Cache, Refresh läuft unsichtbar im Hintergrund

### 🔧 Virtual Scroll
`@tanstack/virtual` für alle Listen über 200 Einträge.
- 150k SKUs ohne Virtual Scroll = Browser-Absturz
- DOM rendert maximal die sichtbaren ~20 Zeilen
- Gilt für: Produktliste, Bestellungen, Inventory-Tabelle

### 🔧 Optimistic Updates
Inventory-Änderung sofort in der UI spiegeln, DB-Write im Hintergrund.
- Bei Fehler: automatischer Rollback via React Query `onError`
- User spürt 0ms Latenz statt 200ms Netzwerk-RTT
- Gilt für: Inventory-Felder, Status-Änderungen, Preise

### 🚨 Bundle Audit
`vite-bundle-visualizer` einmalig laufen lassen.
- Häufigste Fallen: `lodash` ungekürzt (70kb), `moment.js` (232kb), ungenutzter Recharts-Import
- **Ziel:** Initial JS unter 150kb
- Befehl: `npx vite-bundle-visualizer`

---

## Layer 2 — Edge (Traefik + Nginx)

### 🔧 Brotli Kompression
In `nginx.conf`:
```nginx
brotli on;
brotli_comp_level 6;
brotli_types text/html text/css application/json application/javascript;
```
- Typisch 70–80% kleiner als ohne Kompression
- Traefik Middleware reicht nicht — muss in Nginx selbst konfiguriert werden

### ✅ Static Cache 1yr
`Cache-Control: public, max-age=31536000, immutable` für alle JS/CSS-Dateien mit Content-Hash.
- Vite macht das automatisch durch Dateinamen-Hash
- Kein einziger Re-Download für bekannte Assets

### ✅ API Cache Headers
GET-Endpunkte: `Cache-Control: private, max-age=30`
- `private` = nur Browser cached, kein shared Cache
- Wichtig bei Tenant-Daten — niemals `public` setzen

### 🔧 Rate Limit per Tenant
Traefik Middleware: 500 req/min pro Tenant-JWT.
- Ein großer Tenant mit 40 Mitarbeitern soll nicht den VPS für alle anderen blockieren
- In Traefik via `middlewares → rateLimit` konfigurierbar
- Identifikation per JWT-Claim `tenant_id`

---

## Layer 3 — API (Node.js · PM2 Cluster · 4 Workers)

### 🔧 In-process LRU Cache (L1)
`node-lru-cache` pro Worker: 500 Einträge, TTL 10s.
- Kosten: ~0.01ms (in-memory)
- Redis: ~1ms
- DB: ~10ms
- Ideal für: `tenant_configs`, `permission_checks`, `channel_credentials`
- Ändern sich selten, werden aber bei jedem Request geprüft

### 🔧 Request Deduplication
Wenn 30 Mitarbeiter eines Tenants gleichzeitig `/products` laden: nur 1 DB-Call.
- Alle anderen warten auf dasselbe Promise (`async-singleflight` oder `p-memoize`)
- Besonders wirksam 7–9 Uhr wenn alle gleichzeitig einloggen
- Key: `tenant:{id}:route:{path}:{params_hash}`

### 🔧 Batch Endpoints
`POST /api/products/batch` mit Array von IDs statt 50 einzelne GET-Requests.
- Frontend schickt `[id1...id50]` → 1 DB-Query mit `WHERE id = ANY($1)`
- Besonders wichtig bei Inventory-Check pro SKU
- Gilt für: Products, Orders, Inventory, Channels

### 🔧 Circuit Breaker
`opossum` library, pro Marketplace-Connector.
- Wenn eBay API 5× in Folge 503 zurückgibt: Circuit öffnet für 30s
- Keine weiteren Requests in dieser Zeit
- Verhindert Retry-Storm der alle 16 Connectors lahmlegt
- States: CLOSED → OPEN → HALF-OPEN

### 🚨 pgBouncer — KRITISCH
**Supabase Dashboard → Settings → Database → Connection Pooling → Transaction Mode aktivieren.**
- Ohne Pooling: 4 Workers × 20 Connections = 80 offene Postgres-Connections im Normalbetrieb
- Mit 200k Tenants im Peak: DB-Crash durch Connection-Limit
- Transaction Mode: Connection wird nur während aktiver Query gehalten
- **Sofort erledigen — 2 Klicks im Dashboard**

---

## Layer 4 — Cache (Redis · bereits aktiv)

### 🔧 Key Schema standardisieren
```
tenant:{id}:products:{hash}
tenant:{id}:inventory:{sku_id}
tenant:{id}:config
tenant:{id}:session:{user_id}
```
- Immer Tenant-prefixed
- Erlaubt atomares Löschen aller Daten eines Tenants: `SCAN tenant:123:*` + `DEL`
- Bei Tenant-Kündigung: vollständiger Cache-Purge in einem Befehl

### 🔧 TTL-Stufen definieren
| Datentyp | TTL |
|---|---|
| Live-Orders | 10s |
| Inventory | 30s |
| Produktdaten | 5min |
| Tenant-Config | 1h |
| Session-Data | 24h |

Nie unbegrenzt — RLS-Policies können sich ändern.

### 🟣 Cache Pre-warming 06:50 Uhr
**Cron täglich 06:50:** Top-500 aktivste Tenants (sortiert nach `last_login`) vorab in Redis laden.
- Inventory + Produkte werden warm geladen
- Peak startet um 07:00 Uhr mit 0 Cold-Starts
- Kein anderes Multi-Tenant-SaaS macht das, weil Consumer-SaaS keinen vorhersehbaren B2B-Peak hat
- Query für Top-500: `SELECT tenant_id FROM tenants ORDER BY last_login DESC LIMIT 500`

### 🔧 Pub/Sub Cache Invalidierung
Wenn Inventory geupdated: `PUBLISH tenant:123:inv:changed`.
- Alle 4 API-Worker subscriben und löschen ihren lokalen L1-Cache sofort
- Cross-Worker-Konsistenz ohne DB-Poll
- Latenz der Invalidierung: <1ms statt nächster TTL-Ablauf

---

## Layer 5 — Database (Supabase Managed · RLS aktiv)

### ✅ Composite Indexes (tenant_id aktiv)
Nächste Schritte:
```sql
-- Orders
CREATE INDEX idx_orders_tenant_channel ON orders(tenant_id, channel_id);
CREATE INDEX idx_orders_tenant_status  ON orders(tenant_id, status);

-- Soft-Delete: nur aktive Zeilen indexieren
CREATE INDEX idx_products_active ON products(tenant_id, sku_id)
  WHERE deleted_at IS NULL;
```

### 🔧 Materialized Views
```sql
CREATE MATERIALIZED VIEW mv_inventory_summary AS
  SELECT tenant_id, sku_id, channel_id, SUM(qty) as total_qty
  FROM inventory_reservations
  GROUP BY tenant_id, sku_id, channel_id;

-- Refresh jede 60s via Supabase cron (pg_cron)
SELECT cron.schedule('refresh-inv-summary', '* * * * *',
  'REFRESH MATERIALIZED VIEW CONCURRENTLY mv_inventory_summary');
```
- Dashboard-Aggregationen kommen nie mehr aus Live-Queries

### ✅ RLS + set_config Pattern
API nutzt `service_role` + setzt manuell Tenant-Context:
```sql
SELECT set_config('app.current_tenant_id', $1, true);
```
RLS-Policy:
```sql
WHERE tenant_id = current_setting('app.current_tenant_id')::uuid
```
Schneller als JWT-basiertes RLS, weil kein JWT-Parse pro Row.

### 🚨 pg_stat_statements aktivieren
**Supabase Dashboard → Extensions → pg_stat_statements → Enable**
```sql
SELECT query, total_exec_time, calls, mean_exec_time
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;
```
- Zeigt die 10 teuersten Queries nach Gesamtzeit
- Einmalig laufen lassen, Top-3 Queries optimieren = größter DB-Gewinn
- **Sofort erledigen**

### 🔧 Table Partitioning vorbereiten
`products` + `orders` nach `tenant_id` range-partitionieren wenn >5M Zeilen.
- Jetzt Schema so anlegen dass Migration später möglich ist
- Keine direkten Foreign Keys auf unpartitionierte Tabellen
- PostgreSQL pruned Partitionen automatisch → Queries nur auf relevante Partition

### 🔧 Read Replica
Supabase Pro Plan: Read Replica aktivieren.
- Alle SELECT-Queries (Reports, Dashboard, Exports) → Replica
- Writes → Primary
- Verdoppelt effektiven Read-Durchsatz ohne neuen Server
- Connection String: `SUPABASE_REPLICA_URL` als separate ENV-Variable

---

## Layer 6 — Sync Service (PM2 · 16 Connectors)

### 🔧 Priority Queues (BullMQ)
3 Queues mit separaten Worker-Pools:
```
orders:high      → 5 Worker  (Geld — niemals warten lassen)
inventory:medium → 3 Worker  (Bestand — wichtig aber nicht sofort)
listings:low     → 1 Worker  (Kann warten)
```
BullMQ mit Redis bereits im Stack — nur Queues aufteilen.

### 🟣 Per-Tenant Marketplace Rate Limiting
Token-Bucket pro `tenant_id × connector_id` in Redis.
```
key: ratelimit:tenant:{id}:ebay
bucket: 5000 calls / day
refill: täglich 00:00 Uhr
```
- eBay: 5000 calls/day per App-Key
- Ohne Isolation: 1 aktiver Tenant verbrennt das Limit aller anderen
- Kein anderes Multi-Tenant-SaaS löst das auf dieser Granularität

### 🟣 Write Coalescing für Inventory
100ms Buffer pro SKU:
```
SKU-123 Update von eBay   → t=0ms   buffer start
SKU-123 Update von Otto   → t=40ms  buffer reset
SKU-123 Update von Amazon → t=80ms  buffer reset
→ t=180ms: 1 einziger DB-Write mit letztem Wert
```
- Verhindert 3 simultane UPDATE-Konflikte auf derselben Zeile
- Implementierung: `setTimeout` pro SKU-Key, bei neuem Update clearen + neu setzen

### 🔧 Backpressure
Wenn BullMQ-Queue > 5000 pending Jobs: Sync-Service pausiert neue Marketplace-Fetches.
- Verhindert Memory-Überlauf auf dem VPS während 7–9 Uhr Peak
- Wenn alle 200k Tenants gleichzeitig Orders pullen: Queue begrenzt den Druck auf die DB

### ✅ Dead Letter Queue + exponential Backoff
DLQ bereits aktiv. Sicherstellen:
```
Retry 1: +1s
Retry 2: +2s
Retry 3: +4s
Retry 4: +8s
Retry 5: +30s
→ DLQ
```
- Kein flat retry — verhindert Retry-Storm nach Marketplace-Ausfall
- Nach DLQ: Alert + manuelle Review

### ✅ Cron Deduplication
Sicherstellen: Redis-Lock mit `TTL = Cron-Interval`.
```
SETNX lock:cron:{job_name} 1
EXPIRE lock:cron:{job_name} {interval_seconds}
```
- Wenn PM2 2 Worker hat und beide den Cron feuern → Lock verhindert Doppelausführung

---

## Prioritäten — Sofort (vor allem anderen)

| Priorität | Baustein | Aufwand | Impact |
|---|---|---|---|
| 🚨 1 | pgBouncer aktivieren | 2 Klicks | DB crasht ohne das |
| 🚨 2 | pg_stat_statements | 1 Klick | Zeigt echte Bottlenecks |
| 🔧 3 | Virtual Scroll | 2–4h | Browser crasht bei 150k SKUs |
| 🔧 4 | Bundle Audit | 30min | Initial Load halbieren |
| 🔧 5 | BullMQ Priority Queues | 1 Tag | Sync-Chaos im Peak |
| 🟣 6 | Cache Pre-warming | 4h | 0 Cold-Starts um 07:00 |
| 🟣 7 | Write Coalescing | 4h | Inventory-Konflikte |
| 🟣 8 | Per-Tenant Rate Limit | 1 Tag | Quota-Schutz |
