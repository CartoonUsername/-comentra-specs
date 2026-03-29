# Comentra — Claude Code Spec: Performance Architecture
> Referenz für Claude Code Sessions. Immer mit `/model claude-sonnet-4-6` arbeiten.

---

## Kontext
- Stack: React + Node.js + Supabase (managed) + Redis + Docker + Traefik + PM2
- VPS: 4 CPU / 8–16 GB RAM, Hostinger, 1 Server
- Deployment: `docker compose up --build -d`, kein Auto-Deploy
- Supabase: RLS aktiv, service_role key für API, managed auf supabase.com
- Sync: 16 Marketplace Connectors (eBay, Otto, Amazon etc.), BullMQ + Redis

---

## Prompt-Template

```
Arbeite an der Datei [PFAD].
Kontext: [KURZE BESCHREIBUNG DES MODULS].
Aufgabe: Implementiere [BAUSTEIN] gemäß der Spec in /docs/performance-spec.md.
Keine neuen Dependencies ohne Rückfrage. Kein Breaking Change an bestehenden API-Contracts.
Nach jeder Änderung: kurze Zusammenfassung was geändert wurde und warum.
```

---

## Session 1 — pgBouncer + pg_stat_statements

**Ziel:** DB-Connection-Pooling aktivieren und Query-Profiling einschalten.

**Aufgabe 1 — pgBouncer:**
```
Supabase Dashboard → Settings → Database → Connection Pooling.
Mode: Transaction.
Gib mir den neuen Connection String und erkläre welche ENV-Variable ich in docker-compose.yml
für den API-Service anpassen muss (DATABASE_URL).
Prüfe ob node-postgres (pg) oder @supabase/supabase-js mit Transaction Mode kompatibel ist
und ob prepared statements deaktiviert werden müssen.
```

**Aufgabe 2 — pg_stat_statements:**
```
Aktiviere pg_stat_statements in Supabase (Dashboard → Extensions).
Erstelle eine neue Datei /scripts/db-profiling.sql mit:
1. Query für Top-10 teuerste Queries nach total_exec_time
2. Query für Top-10 häufigste Queries nach calls
3. Query um pg_stat_statements zu resetten
Kommentiere jeden Block auf Deutsch.
```

---

## Session 2 — Redis Key Schema + TTL Tiers

**Ziel:** Redis-Nutzung standardisieren, TTL-Stufen definieren, Pub/Sub Invalidierung.

**Aufgabe:**
```
Erstelle /src/cache/redis-schema.ts mit:
1. Key-Generator-Funktionen für alle Cache-Keys (tenant:config, tenant:products, tenant:inventory)
2. TTL-Konstanten: LIVE_ORDERS=10, INVENTORY=30, PRODUCTS=300, CONFIG=3600, SESSION=86400
3. Eine invalidateTenant(tenantId) Funktion die alle Keys eines Tenants löscht (SCAN + DEL)
4. Publisher-Funktion publishInvalidation(tenantId, type) für Pub/Sub
5. Subscriber-Setup der in allen PM2-Workern den lokalen L1-Cache leert

Bestehende Redis-Instanz läuft bereits. Nutze ioredis.
Kein Breaking Change an bestehenden Cache-Calls — nur Wrapper drüber.
```

---

## Session 3 — LRU L1 Cache + Request Deduplication

**Ziel:** API-Layer vor Redis entlasten, simultane Requests coalescing.

**Aufgabe:**
```
Erstelle /src/cache/l1-cache.ts:
1. node-lru-cache Instanz: max 500 Einträge, TTL 10000ms
2. get/set/del Wrapper mit TypeScript Generics
3. Automatische Invalidierung wenn Redis Pub/Sub Signal empfangen

Erstelle /src/middleware/dedup.ts:
1. In-flight Request Deduplication via async-singleflight
2. Key-Strategie: tenant_id + route + params_hash
3. Nur für GET-Requests und RPC-Calls, nie für Mutations

Integriere beide in die bestehende Express-Middleware-Chain in /src/server.ts.
Zeige mir den Diff für server.ts separat.
```

---

## Session 4 — BullMQ Priority Queues

**Ziel:** Sync-Service in 3 priorisierte Queues aufteilen.

**Aufgabe:**
```
Refactore /src/sync/queue.ts:

Aktueller Stand: [BESCHREIBE AKTUELLE QUEUE-STRUKTUR]

Ziel: 3 BullMQ-Queues mit separaten Worker-Pools:
- orders:high     → concurrency: 5
- inventory:medium → concurrency: 3  
- listings:low    → concurrency: 1

Regeln:
- Bestehende Job-Typen den richtigen Queues zuordnen (Orders → high, etc.)
- Worker-Health-Check pro Queue: wenn Queue > 5000 Jobs → pausiere neue Fetches (Backpressure)
- Exponential Backoff: [1000, 2000, 4000, 8000, 30000]ms, dann DLQ
- Keine Änderung am Job-Payload-Format

Zeige die neue Queue-Konfiguration und die Worker-Initialisierung.
```

---

## Session 5 — Write Coalescing (Inventory)

**Ziel:** Simultane Inventory-Updates von mehreren Marketplaces zusammenfassen.

**Aufgabe:**
```
Erstelle /src/sync/write-coalescer.ts:

Problem: SKU-123 wird gleichzeitig von eBay, Otto und Amazon geupdated
→ 3 simultane DB-Writes auf dieselbe Zeile → Konflikte

Lösung:
1. Buffer pro SKU-Key (tenant_id + sku_id): 100ms Fenster
2. Wenn neues Update kommt: Timer resetten, Wert überschreiben
3. Nach 100ms ohne neues Update: 1 einziger DB-Write
4. TypeScript, keine externen Dependencies nötig (nur setTimeout/clearTimeout)
5. Integriere in den bestehenden Inventory-Sync unter /src/sync/inventory-connector.ts

Zeige mir den vollständigen Code und den Diff für inventory-connector.ts.
```

---

## Session 6 — Cache Pre-warming (06:50 Uhr)

**Ziel:** Top-500 Tenants täglich vor dem Peak vorladen.

**Aufgabe:**
```
Erstelle /src/cron/pre-warm.ts:

1. Cron-Schedule: täglich 06:50 Uhr (node-cron oder pg_cron via Supabase)
2. Query: SELECT tenant_id FROM tenants ORDER BY last_login DESC LIMIT 500
3. Für jeden Tenant parallel (Promise.allSettled, maximal 50 concurrent):
   - Tenant-Config laden und in Redis cachen (TTL: 3600s)
   - Aktive Produkte laden und in Redis cachen (TTL: 300s)  
   - Aktuellen Inventory-Stand laden und in Redis cachen (TTL: 30s)
4. Logging: wie viele Tenants erfolgreich vorgeladen, Gesamtdauer
5. Redis-Lock (SETNX) damit der Job nicht doppelt läuft

Nutze bestehende Cache-Funktionen aus /src/cache/redis-schema.ts (Session 2).
```

---

## Session 7 — Per-Tenant Marketplace Rate Limiting

**Ziel:** Quota-Schutz pro Tenant × Connector.

**Aufgabe:**
```
Erstelle /src/sync/rate-limiter.ts:

Token-Bucket Implementierung in Redis:
- Key: ratelimit:tenant:{tenantId}:connector:{connectorId}
- Limits per Connector (konfigurierbar):
  ebay:    5000 calls/day
  otto:    3000 calls/day
  amazon:  900 calls/hour (SP-API limit)
  kaufland: 1000 calls/day
- Refill: täglich 00:00 Uhr per Cron für daily limits, stündlich für hourly
- Funktionen: checkLimit(tenantId, connector) → boolean, consumeToken(tenantId, connector)
- Wenn Limit erreicht: Job in listings:low Queue verschieben statt droppen

Integriere checkLimit in den Connector-Base-Class unter /src/sync/base-connector.ts.
Zeige den Diff für base-connector.ts.
```

---

## Session 8 — Frontend Bundle + Virtual Scroll

**Ziel:** Initial-Bundle unter 150kb, große Listen performant rendern.

**Aufgabe 1 — Bundle:**
```
Analysiere /package.json und /vite.config.ts.
Erstelle eine optimierte vite.config.ts mit:
1. manualChunks: vendor (react, react-dom), ui (shadcn/radix), data (tanstack-query, supabase-js)
2. Entferne moment.js falls vorhanden → date-fns als Ersatz
3. Prüfe ob lodash als volle Library importiert wird → lodash-es mit tree-shaking
4. Zeige erwartete Bundle-Größen nach der Optimierung
```

**Aufgabe 2 — Virtual Scroll:**
```
Baue eine wiederverwendbare VirtualTable-Komponente unter /src/components/ui/VirtualTable.tsx:
1. Basiert auf @tanstack/virtual (bereits im Stack prüfen, sonst installieren)
2. Props: data[], columns[], rowHeight=40, overscan=10
3. Unterstützt: Sortierung, Selektion (Checkbox), Inline-Edit
4. Ersetze die bestehende Produkt-Tabelle in /src/pages/Products.tsx als ersten Use Case
5. Zeige Vorher/Nachher DOM-Node-Count bei 1000 Zeilen
```

---

## Monitoring — Was nach dem Launch gemessen werden muss

Erstelle `/src/monitoring/metrics.ts` mit diesen Messungen:

```typescript
// Diese Metriken nach Go-Live täglich prüfen:
// 1. Redis Hit Rate: sollte >85% sein
// 2. DB Connection Count: sollte <50 sein (mit pgBouncer)
// 3. API P95 Response Time: sollte <200ms sein
// 4. Queue Depth orders:high: sollte <100 sein um 07:00 Uhr
// 5. Cache Pre-warm Success Rate: sollte 100% sein täglich
```

---

## Nicht anfassen (Breaking Risk)

- `supabase.ts` Haupt-Client — nur lesen, nie modifizieren
- RLS-Policies direkt — immer via Migration, nie via Dashboard
- PM2-Konfiguration — immer `ecosystem.config.js` nutzen
- Docker-Netzwerk-Namen — Traefik-Labels hängen davon ab
