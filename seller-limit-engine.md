# Comentra — Seller Limit Engine

## WICHTIG — REGELN

- Erstelle NUR neue Dateien
- Berühre KEINE bestehenden Dateien
- Keine Änderungen an App.tsx, router, types.ts, package.json
- Keine globale Middleware
- Wenn du eine bestehende Datei anfassen willst: STOPP und frage mich zuerst
- Zeige mir nach dem Scan der Projektstruktur welche neuen Dateien du anlegen wirst

---

## Ziel

Engine im Hintergrund die automatisch Limits pro Tenant berechnet, überwacht und skaliert.
Keine Änderung an bestehender Architektur. Alles additiv.

---

## Neue Dateien anlegen

```
src/
  lib/
    limitEngine/
      index.ts              ← öffentliche API der Engine
      calculateTrustScore.ts
      checkTenantLimit.ts
      autoScaler.ts
      limitGuard.ts         ← Middleware-Funktion (nicht eingehängt, nur exportiert)
      types.ts              ← eigene Types, nicht in bestehende types.ts
  jobs/
    recalculateLimits.ts    ← Cron Job, standalone
  components/
    settings/
      LimitsSettings.tsx    ← UI Komponente, standalone (nicht in Router eingetragen)
supabase/
  migrations/
    20260326_tenant_limits.sql
```

---

## Database Schema

```sql
create table tenant_limits (
  tenant_id uuid primary key references tenants(id),
  max_active_listings integer,
  max_monthly_orders integer,
  max_monthly_revenue numeric,
  max_products_total integer,
  max_api_requests_per_minute integer,
  max_marketplace_connections integer,
  trust_score numeric default 0,
  risk_score numeric default 0,
  auto_scaling_enabled boolean default true,
  last_recalculated_at timestamptz,
  created_at timestamptz default now(),
  updated_at timestamptz default now()
);

create table tenant_limit_usage (
  tenant_id uuid primary key references tenants(id),
  active_listings integer default 0,
  monthly_orders integer default 0,
  monthly_revenue numeric default 0,
  products_total integer default 0,
  api_requests_current integer default 0,
  period_start timestamptz,
  period_end timestamptz
);

create table tenant_limit_events (
  id uuid primary key default gen_random_uuid(),
  tenant_id uuid references tenants(id),
  type text,
  metric text,
  value numeric,
  reason text,
  metadata jsonb,
  created_at timestamptz default now()
);
```

RLS auf allen Tabellen. Tenant sieht nur eigene Rows.

---

## calculateTrustScore.ts

```ts
export function calculateTrustScore(input: {
  account_age_days: number
  orders_last_30d: number
  revenue_last_30d: number
  sync_error_rate: number
  api_error_rate: number
  refund_rate: number
  cancellation_rate: number
}): number {
  const raw =
    input.account_age_days * 0.1 +
    input.orders_last_30d * 0.3 +
    input.revenue_last_30d * 0.2 -
    input.sync_error_rate * 0.2 -
    input.refund_rate * 0.2
  return Math.min(100, Math.max(0, raw))
}
```

---

## checkTenantLimit.ts

```ts
export async function checkTenantLimit(
  tenant_id: string,
  metric: string,
  value: number
): Promise<{ allowed: boolean; reason: string; remaining: number }>
```

- Liest aus tenant_limits + tenant_limit_usage
- Gibt allowed: false zurück wenn Limit erreicht
- Loggt Event in tenant_limit_events

---

## autoScaler.ts

Prüft pro Tenant:
- Bei 80% Nutzung → Warning Event loggen
- Bei 90% → Scaling prüfen
- Bei 100% → blocked Event loggen

Auto-Erhöhung wenn:
- trust_score > 70
- error_rate < 2%

```
max_products_total *= 1.5
max_monthly_orders *= 1.3
```

---

## limitGuard.ts

Exportiert nur eine Funktion — nicht global eingehängt:

```ts
export async function limitGuard(tenant_id: string, metric: string) {
  const result = await checkTenantLimit(tenant_id, metric, 1)
  if (!result.allowed) {
    throw new LimitExceededError(result.reason)
  }
}
```

Nutzung später manuell pro Route einhängen wenn gewünscht.

---

## recalculateLimits.ts (Cron Job)

Standalone-Datei, stündlich ausführen:
1. Alle Tenants laden
2. Trust Score neu berechnen
3. Auto Scaler laufen lassen
4. Usage-Periode zurücksetzen wenn abgelaufen
5. last_recalculated_at updaten

Nicht automatisch registriert — manuell in bestehenden Job-Runner einhängen wenn bereit.

---

## LimitsSettings.tsx (UI)

Standalone React Komponente:
- Props: tenantId: string
- Zeigt Usage Bars pro Metrik
- Farben: grün < 80%, gelb 80-95%, rot > 95%
- Zeigt Trust Score + Risk Score
- Toggle für auto_scaling_enabled

Nicht in Router eingetragen — manuell importieren wenn gewünscht.

---

## SaaS Tiers

```ts
export const TIER_DEFAULTS = {
  starter: {
    max_products_total: 1000,
    max_monthly_orders: 500,
    max_marketplace_connections: 2,
    auto_scaling_enabled: false
  },
  growth: {
    max_products_total: 10000,
    max_monthly_orders: 5000,
    max_marketplace_connections: 5,
    auto_scaling_enabled: true
  },
  enterprise: {
    max_products_total: null,
    max_monthly_orders: null,
    max_marketplace_connections: null,
    auto_scaling_enabled: true
  }
}
```

---

## Liefere

1. SQL Migration
2. src/lib/limitEngine/types.ts
3. src/lib/limitEngine/calculateTrustScore.ts
4. src/lib/limitEngine/checkTenantLimit.ts
5. src/lib/limitEngine/autoScaler.ts
6. src/lib/limitEngine/limitGuard.ts
7. src/lib/limitEngine/index.ts
8. src/jobs/recalculateLimits.ts
9. src/components/settings/LimitsSettings.tsx

Keine Änderungen an bestehenden Dateien.
Alle Dateien vollständig, kein Platzhalter-Code.
