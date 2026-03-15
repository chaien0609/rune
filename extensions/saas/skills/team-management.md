---
name: "team-management"
pack: "@rune/saas"
description: "Organization, team, and member permissions — RBAC hierarchy, invite flow with expiry, permission checking at API and UI layers, and audit trail for permission changes."
model: sonnet
tools: [Read, Edit, Write, Grep, Glob, Bash]
---

# team-management

Organization, team, and member permissions — RBAC hierarchy, invite flow with expiry, permission checking at API and UI layers, and audit trail for permission changes.

#### Role Hierarchy

```
Owner (1 per org)
  └── Admin (multiple)
        └── Member (default role)
              └── Viewer (read-only)
```

Org-level roles apply across all teams. Team-level roles can be more restrictive (e.g., org Member can be team Admin for a specific team).

#### Permission Matrix

| Action | Owner | Admin | Member | Viewer |
|---|---|---|---|---|
| Delete organization | ✅ | ❌ | ❌ | ❌ |
| Manage billing | ✅ | ✅ | ❌ | ❌ |
| Invite members | ✅ | ✅ | ❌ | ❌ |
| Create teams | ✅ | ✅ | ❌ | ❌ |
| Create projects | ✅ | ✅ | ✅ | ❌ |
| View projects | ✅ | ✅ | ✅ | ✅ |
| Manage team members | ✅ | ✅ (own teams) | ❌ | ❌ |

#### Workflow

**Step 1 — Design org/team schema**
Model: `Organization → Team → Membership (userId, orgId, teamId?, role)`. Org-level membership has `teamId = null`. Team-level membership scopes the role to a specific team. Use a single `Membership` table with nullable `teamId` rather than separate `OrgMember` and `TeamMember` tables.

**Step 2 — Implement RBAC middleware**
Create a `requirePermission(action)` middleware that reads `req.user.id` + `req.tenantId`, loads the user's role for that org, and checks against a permission map. Fail fast: return 403 immediately if permission not found. Never trust client-provided role claims.

**Step 3 — Build invite flow**
Invite: generate a signed token (`crypto.randomBytes(32).hex`), store with `{ email, orgId, role, invitedBy, expiresAt: +7d }`, send email with link. Accept: verify token not expired, not already accepted, create Membership record, mark invite as accepted. Resend: invalidate old token, create new one with fresh expiry. Pending invites visible to admins in settings.

**Step 4 — Add permission UI gates**
In React: `<CanAccess action="invite_members"><InviteButton /></CanAccess>` — hides UI elements the user can't use. Also disable + tooltip pattern: show the button but disable it with "Upgrade to invite members" tooltip (better UX than hiding, helps users understand what's possible). Enforce the same check in the API — UI gates are cosmetic only.

**Step 5 — Emit audit trail**
Every permission change, role assignment, invite, and removal MUST log to an `AuditLog` table: `{ orgId, actorId, targetId, action, before, after, ip, userAgent, timestamp }`. Surface the last 100 entries in the org settings Security tab. Retain for 90 days minimum (compliance requirement for SOC2).

#### Example

```typescript
// Prisma schema — org, team, membership
model Organization {
  id        String       @id @default(cuid())
  name      String
  slug      String       @unique
  members   Membership[]
  teams     Team[]
}

model Team {
  id      String       @id @default(cuid())
  orgId   String
  name    String
  org     Organization  @relation(fields: [orgId], references: [id])
  members Membership[]
}

model Membership {
  id        String       @id @default(cuid())
  userId    String
  orgId     String
  teamId    String?      // null = org-level role
  role      Role
  user      User         @relation(fields: [userId], references: [id])
  org       Organization @relation(fields: [orgId], references: [id])
  team      Team?        @relation(fields: [teamId], references: [id])

  @@unique([userId, orgId, teamId]) // one role per user per scope
}

enum Role { OWNER ADMIN MEMBER VIEWER }

// Permission map
const PERMISSIONS = {
  delete_org:      ['OWNER'],
  manage_billing:  ['OWNER', 'ADMIN'],
  invite_members:  ['OWNER', 'ADMIN'],
  create_projects: ['OWNER', 'ADMIN', 'MEMBER'],
  view_projects:   ['OWNER', 'ADMIN', 'MEMBER', 'VIEWER'],
} as const;
type Action = keyof typeof PERMISSIONS;

// RBAC middleware — never trust client-provided role
const requirePermission = (action: Action) => async (req: Request, res: Response, next: NextFunction) => {
  const membership = await prisma.membership.findFirst({
    where: { userId: req.user!.id, orgId: req.tenantId!, teamId: null },
  });
  if (!membership || !(PERMISSIONS[action] as readonly string[]).includes(membership.role)) {
    return res.status(403).json({ error: { code: 'FORBIDDEN', action } });
  }
  req.userRole = membership.role;
  next();
};

// React permission hook
function usePermission(action: Action): boolean {
  const { membership } = useOrg();
  if (!membership) return false;
  return (PERMISSIONS[action] as readonly string[]).includes(membership.role);
}

// Invite flow
const createInvite = async (orgId: string, email: string, role: Role, invitedBy: string) => {
  const token = crypto.randomBytes(32).toString('hex');
  await prisma.invite.create({
    data: { orgId, email, role, invitedBy, token, expiresAt: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000) },
  });
  await emailQueue.add('invite', { email, token, orgId });
  return token;
};

const acceptInvite = async (token: string, userId: string) => {
  const invite = await prisma.invite.findUnique({ where: { token } });
  if (!invite || invite.acceptedAt || invite.expiresAt < new Date()) {
    throw new Error('Invalid or expired invite');
  }
  await prisma.$transaction([
    prisma.membership.create({ data: { userId, orgId: invite.orgId, role: invite.role } }),
    prisma.invite.update({ where: { token }, data: { acceptedAt: new Date() } }),
    prisma.auditLog.create({ data: { orgId: invite.orgId, actorId: userId, action: 'MEMBER_JOINED', targetId: userId } }),
  ]);
};
```

**Sharp edges for team-management:**
- **Permission escalation**: an Admin inviting another Admin is fine, but an Admin promoting themselves to Owner must be blocked. Rule: you can only assign roles lower than your own.
- **Cross-org data leak**: when loading team resources, always filter by `orgId`. A user who belongs to two orgs must never see org B's data when acting in org A's context.
- **Invite token reuse**: after an invite is accepted, mark it accepted immediately in the same transaction as membership creation. Race condition: two tabs accepting the same invite → use `@@unique` on membership + catch unique constraint error.
- **Owner removal**: prevent the last Owner from being removed or downgraded. Always require at least one Owner per org. Check before processing the role change.
