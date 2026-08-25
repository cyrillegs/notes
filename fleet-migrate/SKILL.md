---
name: fleet-migrate
description: "Tabula (schema-per-tenant ERP) only. Use when a tenant-schema migration must reach every tenant database — any PR touching prisma/tenant.prisma or adding a file under prisma/migrations/tenant/. Covers the additive-vs-narrowing timing rule (which decides whether you migrate BEFORE or AFTER the deploy), the NumberSeries backfill check, and the insights_ro grant spot-check. Triggers on \"migrate the fleet\", \"run tenants:migrate\", \"apply the migration to all tenants\", or after merging a milestone that adds tenant tables."
---

# Fleet migration

Tabula is **schema-per-tenant**: one Postgres schema (`t_<slug>`) per customer.
A Prisma migration does NOT reach them by itself. `prisma migrate deploy` runs at
*provisioning* time for a brand-new tenant; every tenant that already exists is
updated only by `scripts/migrate-all-tenants.ts`.

Forget this step and you get the worst kind of bug: **new signups work perfectly
while every existing customer is broken.**

## Step 0 — classify the migration (this decides the timing)

Read the migration SQL and sort it into one of two buckets. Everything else
follows from this.

| Bucket | SQL | When to run |
|---|---|---|
| **Additive** | `CREATE TABLE`, `CREATE INDEX`, `CREATE TYPE`, `ADD COLUMN` (nullable or with a DEFAULT), `INSERT … ON CONFLICT DO NOTHING` | **BEFORE the merge.** Safe — the currently-deployed app simply ignores the new shape. |
| **Narrowing** | `DROP COLUMN`, `DROP TABLE`, `SET NOT NULL`, `RENAME`, type changes, dropping a default | **AFTER the deploy** that stopped using the old shape. |

**Additive migrations should always run first.** The usual race — deploy lands
before the migration and 500s for the gap — simply cannot happen, because the
old code doesn't know the new columns exist. Verified in production on M8 and
M9: prod served 200 under the *old* app version against the *new* schema, mid-migration.

If a PR mixes both, split it.

## Steps

```bash
# 1. Be on the code whose migration you're applying
git checkout main && git pull          # post-merge
# ...or work inside the PR branch / its worktree for a pre-merge additive run

# 2. Regenerate the Prisma clients — skipping this makes typecheck fail confusingly
pnpm prisma:generate

# 3. Preview, then apply
pnpm tenants:migrate --dry-run
pnpm tenants:migrate
```

The script is per-tenant fault-isolated: one failing schema doesn't stop the
rest, and it exits non-zero if any failed. Bookkeeping lives in each schema's
`_tenant_migrations` table.

## Verification — all three, every time

**1. The fleet is actually current.** Re-run the dry run; every line should read
`up to date`, and the count should match the tenant count.

```bash
pnpm tenants:migrate --dry-run 2>&1 | grep -c "up to date"
```

**2. NumberSeries backfill reached OLD tenants** — only if the migration adds a
document type. This is the check that catches the classic failure. Provisioning
seeds series for *new* tenants only, so the migration must `INSERT … ON CONFLICT
("docType") DO NOTHING` for existing ones. Verify on a tenant that was
provisioned *before* this milestone existed:

```ts
const db = getTenantPrisma("<an-old-tenant-slug>");
const s = await db.numberSeries.findMany({ where: { docType: "StockReceipt" } });
console.log(s.length, s[0]?.prefix, s[0]?.current);   // expect exactly 1 row, current=0
```

Exactly one row — two would mean the migration and the provisioning seed both
fired (the seed needs `skipDuplicates: true`).

**3. `insights_ro` can read the new tables, and still cannot write.** The I1
bootstrap set `ALTER DEFAULT PRIVILEGES` per schema so grants carry
automatically — but verify rather than assume:

```ts
await adminPrisma.$queryRawUnsafe(
  `SELECT has_table_privilege('insights_ro', '"t_<slug>"."<NewTable>"', 'SELECT') AS ok`
);  // true
await adminPrisma.$queryRawUnsafe(
  `SELECT has_table_privilege('insights_ro', '"t_<slug>"."<NewTable>"', 'INSERT') AS ok`
);  // MUST be false
```

## Extra check for pre-merge (additive) runs

Not part of the three above — it only applies when you migrated ahead of the
merge. Confirm production still serves under the *old* app version against the
*new* schema:

```bash
curl -sS -o /dev/null -w "%{http_code}\n" https://<slug>.tabula-erp.site/login   # 200
```

## Failure modes actually seen

- **Backfill forgotten** → new tenants submit fine, existing ones throw
  `MissingNumberSeriesError`. Invisible in testing, obvious to a customer.
- **`prisma:generate` skipped** → typecheck errors about a model "not existing on
  PrismaClient" that look like a code bug and aren't.
- **Narrowing migration run early** → live 500s until the deploy catches up.
- **Windows/long paths** when migrating from a worktree — run from the worktree
  root, not a nested path.

## Then

Tick the milestone (see the `tick-milestone` skill) and record in the plan row
*which* verification passed — future readers need to know the backfill was
checked, not assumed.
