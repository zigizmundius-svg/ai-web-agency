# MVP Data Model and API Contract

Status: design baseline — provider-neutral, no production credentials or personal data.
Repository: `zigizmundius-svg/ai-web-agency`
Branch: `platform/mvp-foundation`
Pilot: GARAGE 27

## 1. Decisions for the first vertical slice

- Use PostgreSQL as the system of record.
- Build a modular monolith: one backend application, separated into domain modules.
- Keep authentication provider-neutral until hosting and account ownership are decided.
- Keep public website delivery separate from privileged admin operations.
- Derive tenant access from the authenticated user's server-side membership. Never trust a tenant ID supplied by the browser as proof of access.
- Enforce tenant isolation in application authorization and add PostgreSQL Row-Level Security (RLS) as defense in depth where practical.
- Do not store real customer data in development fixtures.

## 2. MVP relational model

All IDs are UUIDs. Timestamps are UTC. Money is stored as integer minor units plus an ISO currency code; never use floating-point values for money.

### Identity and access

**users**
- id, email_normalized (unique), display_name, status, created_at, updated_at

**agency_memberships**
- id, user_id -> users, role, status, created_at
- Unique(user_id)

**tenants**
- id, legal_name, display_name, status, created_at, updated_at

**tenant_memberships**
- id, tenant_id -> tenants, user_id -> users, role, status, created_at
- Unique(tenant_id, user_id)
- Index(user_id, status), Index(tenant_id, status)

Roles are capability bundles, not UI labels. Initial tenant roles: owner, manager, staff. Initial agency roles: owner, admin, manager, support. Permission checks happen on every server operation.

### Projects and public sites

**projects**
- id, tenant_id -> tenants, name, slug, business_type, status, created_at, updated_at
- Unique(tenant_id, slug)
- Index(tenant_id, status)

**sites**
- id, tenant_id -> tenants, project_id -> projects, status, published_version_id (nullable), created_at, updated_at
- Unique(project_id)

**site_content**
- id, tenant_id -> tenants, site_id -> sites, content_key, content_json, updated_by -> users, updated_at
- Unique(site_id, content_key)
- Only allow validated content keys and schemas; do not accept arbitrary executable HTML or scripts.

**services**
- id, tenant_id -> tenants, project_id -> projects, name, description, price_minor (nullable), currency, duration_minutes (nullable), sort_order, is_active, created_at, updated_at
- Index(tenant_id, project_id, is_active)

**media_assets**
- id, tenant_id -> tenants, project_id -> projects, storage_key, original_name, mime_type, size_bytes, visibility, created_by -> users, created_at
- Store files in private object storage by default. Public access must be explicit and limited to published assets.

### CRM

**leads**
- id, tenant_id -> tenants, project_id -> projects, source, name, phone, email, message, status, assigned_to (nullable) -> users, created_at, updated_at
- Index(tenant_id, project_id, status, created_at)
- Restrict access to authorized tenant members and authorized agency support workflows.

**lead_events**
- id, tenant_id -> tenants, lead_id -> leads, actor_user_id (nullable) -> users, event_type, old_value_json, new_value_json, created_at
- Index(tenant_id, lead_id, created_at)

### Platform configuration and audit

**modules**
- id, code (unique), name, description, status

**project_modules**
- id, tenant_id -> tenants, project_id -> projects, module_id -> modules, status, settings_json, enabled_at, disabled_at
- Unique(project_id, module_id)

**audit_events**
- id, actor_user_id (nullable) -> users, tenant_id (nullable) -> tenants, project_id (nullable) -> projects, action, object_type, object_id, outcome, metadata_json, created_at
- Never record passwords, tokens, secrets, full payment details or unnecessary personal data.

## 3. Tenant isolation requirements

1. Every tenant-owned row has a non-null `tenant_id`, or a direct relationship that is checked against the tenant context.
2. The API resolves the current user's active membership on the server for each request.
3. A browser-supplied `tenant_id` may be used only as a requested context selector; the server must verify membership before using it.
4. Reads, updates, deletes, file access and exports all apply the same tenant authorization rule.
5. Public site requests resolve a published site by a verified public hostname/slug, then derive the tenant from that site record.
6. Public form submissions can create leads only for the tenant associated with the resolved public site.
7. Agency access is separately permissioned and audited; it is not an implicit bypass for all staff.
8. Automated tests must attempt cross-tenant reads and writes, including guessed UUIDs and altered route parameters.

## 4. API contract — first release

All admin routes require an authenticated session and server-side authorization. JSON responses use a consistent error shape. The exact framework and URL prefix can be chosen during implementation.

### Session and identity
- `GET /api/me` — current user and permitted workspaces.
- `POST /api/auth/login` — sign in through the selected auth implementation.
- `POST /api/auth/logout` — terminate session.
- `POST /api/auth/recovery` — initiate account recovery.

### Agency
- `GET /api/agency/tenants` — list tenants visible to the agency user.
- `POST /api/agency/tenants` — create tenant.
- `GET /api/agency/tenants/:tenantId` — tenant overview.
- `POST /api/agency/tenants/:tenantId/invitations` — invite tenant owner/member.
- `GET /api/agency/projects` — list projects.
- `POST /api/agency/projects` — create project with tenant, business type and initial configuration.
- `PATCH /api/agency/projects/:projectId` — update permitted project settings.

### Client workspace
- `GET /api/workspace` — current tenant and membership context.
- `GET /api/workspace/site-content` — retrieve editable content.
- `PATCH /api/workspace/site-content` — validate and save allowed content fields.
- `GET /api/workspace/services` — list services.
- `POST /api/workspace/services` — create service.
- `PATCH /api/workspace/services/:serviceId` — update service.
- `GET /api/workspace/leads` — list tenant leads.
- `PATCH /api/workspace/leads/:leadId` — update lead status/assignment.

### Public site
- `GET /api/public/sites/:siteSlug` — retrieve published, public-safe site content.
- `POST /api/public/sites/:siteSlug/leads` — submit a validated enquiry form.
- Public endpoints require rate limiting, input validation, abuse protection and safe error responses.

## 5. First acceptance tests

- User without a membership cannot open a tenant workspace.
- Tenant A user cannot read, update, delete, export or upload against Tenant B IDs.
- Tenant A public form cannot create a lead for Tenant B.
- Tenant A content edits appear on Tenant A's public site only after the defined publish/save behavior.
- A staff role cannot perform owner-only or agency-only actions.
- Disabled or suspended memberships lose access immediately.
- Every sensitive agency action and role change creates an audit event.
- Invalid content keys, oversized payloads and unsafe file types are rejected.
- Lead creation is rate-limited and validated.
- A data export contains only the requesting tenant's records.

## 6. Deliberately deferred

- Online booking and shared availability engine.
- Payment processing and recurring billing.
- Multi-branch businesses.
- Inventory/accounting.
- AI generation inside the client cabinet.
- Production authentication provider, email/SMS vendor and cloud-specific integrations.

These are deferred until the first vertical slice and infrastructure/legal decisions are validated.
