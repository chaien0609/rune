---
name: "multi-tenant"
pack: "@rune/saas"
description: "Multi-tenancy patterns — database isolation strategies, tenant context middleware, data partitioning, cross-tenant query prevention, tenant-aware background jobs, and GDPR data export."
model: sonnet
tools: [Read, Edit, Write, Grep, Glob, Bash]
---

# multi-tenant

Multi-tenancy patterns — database isolation strategies, tenant context middleware, data partitioning, cross-tenant query prevention, tenant-aware background jobs, and GDPR data export.

#### Isolation Strategy Comparison

| Strategy | Cost | Isolation | Migration Difficulty | When to Use |
|---|---|---|---|---|
| Shared DB, tenant column | Low | Weak (app-enforced) | Easy | Early-stage, <1000 tenants |
| Shared DB + PostgreSQL RLS | Low | Strong (DB-enforced) | Easy | Best default for most SaaS |
| Schema-per-tenant | Medium | Strong | Medium | When tenants need schema customization |
| DB-per-tenant | High | Perfect | Hard | Enterprise, compliance (HIPAA, SOC2) |

#### Workflow

**Step 1 — Detect current isolation strategy**
Use Grep to find tenant-related code: `tenantId`, `organizationId`, `workspaceId`, `x-tenant-id` header, RLS policies, schema-per-tenant patterns, database switching logic. Read the database schema and middleware to classify the isolation strategy in use.

**Step 2 — Audit isolation boundaries**
Check for: queries without tenant filter (data leak risk), missing tenant context in middleware, no RLS policies on shared tables, admin endpoints that bypass tenant isolation, background jobs processing cross-tenant data without scoping. Flag each with severity.

**Step 3 — Emit tenant-safe patterns**
Based on detected strategy, emit: tenant middleware (extract from JWT/header, set on request context), RLS policies for shared-schema approach, scoped repository pattern that injects tenant filter on every query, and tenant-aware test fixtures.

**Step 4 — Tenant-aware background jobs**
Every background job MUST carry `tenantId`. Use BullMQ job data to pass tenant context, then initialize a scoped repository inside the job processor. Never process tenant data in a job without an explicit `tenantId` guard.

**Step 5 — Tenant data export (GDPR portability)**
Implement `/api/tenants/:id/export` that collects all data rows belonging to a tenant across all tables, serializes to JSON or CSV, and streams the result as a download. Log the export event in the audit trail with timestamp and requesting user.

#### Example

```typescript
// Tenant middleware — extract from JWT, inject into request context
const tenantMiddleware = async (req: Request, res: Response, next: NextFunction) => {
  const tenantId = req.user?.tenantId ?? req.headers['x-tenant-id'] as string;
  if (!tenantId) return res.status(403).json({ error: { code: 'TENANT_REQUIRED', message: 'Tenant context missing' } });
  req.tenantId = tenantId;
  next();
};

// Scoped repository — every query automatically filtered by tenant
class ScopedRepository<T extends { tenantId: string }> {
  constructor(private model: PrismaModel<T>, private tenantId: string) {}

  async findMany(where: Partial<Omit<T, 'tenantId'>> = {}) {
    return this.model.findMany({ where: { ...where, tenantId: this.tenantId } });
  }

  async create(data: Omit<T, 'tenantId' | 'id' | 'createdAt' | 'updatedAt'>) {
    return this.model.create({ data: { ...data, tenantId: this.tenantId } as any });
  }
}

// PostgreSQL RLS — DB-enforced isolation, safest approach
-- Enable RLS on every shared table
ALTER TABLE projects ENABLE ROW LEVEL SECURITY;

-- Set tenant context before query (from app middleware)
SET LOCAL app.tenant_id = '550e8400-e29b-41d4-a716-446655440000';

-- Policy reads from session variable — automatic for all queries
CREATE POLICY tenant_isolation ON projects
  USING (tenant_id = current_setting('app.tenant_id')::uuid);

-- Set in Prisma $executeRaw before each query block:
-- await prisma.$executeRaw`SELECT set_config('app.tenant_id', ${tenantId}, true)`;

// BullMQ — tenant-aware background job
const emailQueue = new Queue('emails');

// Producer: always pass tenantId in job data
await emailQueue.add('send-invoice', { tenantId, invoiceId, recipientEmail });

// Consumer: initialize scoped context from job data
const worker = new Worker('emails', async (job) => {
  const { tenantId, invoiceId } = job.data;
  const invoices = new ScopedRepository(prisma.invoice, tenantId);
  const invoice = await invoices.findMany({ id: invoiceId });
  // process...
});

// GDPR export — stream all tenant data
app.get('/api/tenants/:id/export', requireOwner, async (req, res) => {
  const { id: tenantId } = req.params;
  const [projects, members, invoices] = await Promise.all([
    prisma.project.findMany({ where: { tenantId } }),
    prisma.member.findMany({ where: { tenantId } }),
    prisma.invoice.findMany({ where: { tenantId } }),
  ]);
  await prisma.auditLog.create({ data: { tenantId, action: 'DATA_EXPORT', actorId: req.user.id } });
  res.setHeader('Content-Disposition', `attachment; filename="export-${tenantId}.json"`);
  res.json({ exportedAt: new Date(), projects, members, invoices });
});
```
