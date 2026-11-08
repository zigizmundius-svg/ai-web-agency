# Reuse Platform Audit — 2026-11-08

## Decision status

**Preferred technical base for the next phase: SimplerDevelopment.**
This is a preliminary adoption decision, not yet a production approval. Do not continue building a parallel Next.js foundation under `platform/app/` until the upstream adoption spike is completed.

The existing agency website, GARAGE 27 demo, and agency-admin prototype remain separate and must not be overwritten.

## Product target to preserve

We are building a Russian-market, multi-tenant website and business-operations platform sold by monthly subscription.

Required product model:
- Agency master console: manage tenants, sites, plans, modules, subscriptions, team access, support, audit trail and system health.
- Client portal: each business manages its own website, content, services, prices, media, leads, CRM and appointments.
- Public tenant sites: separate domains/subdomains, SEO, published content and lead capture.
- Shared source-of-truth calendar: a booking made by a visitor and an appointment entered by staff must reserve the same time/capacity.
- Modular vertical packs: first GARAGE 27 / auto service, then other local-business verticals and sports sections.
- Subscription billing and entitlements controlled by the agency.
- Export and portability of client data.
- Self-hosting on infrastructure selected for Russian personal-data requirements.
- No client should need to edit code.

## Candidate comparison

| Candidate | Strengths | Blocking / material concerns | Assessment |
|---|---|---|---|
| SimplerDevelopment | Apache-2.0; PostgreSQL + Drizzle; Docker Compose; agency admin, client portal, per-tenant sites, block CMS/visual editor, CRM, bookings, automations, billing, S3-compatible media; dedicated tenancy regression suite | Large and complex codebase; Bun runtime/tooling; many Vercel cron routes; some optional external integrations; app-layer tenant filtering rather than Postgres RLS; security hardening required; deployment needs adaptation for a Russian VPS | **Best base to test first** |
| Aglyn | Apache-2.0; strongest visual canvas editor; separate console and tenant runtime; plugins; CRM; booking; multi-tenant org model; Docker self-host path | Self-hosting currently requires Firebase Auth, Firestore, Storage and Realtime Database; vendor-neutral backend is a separate roadmap item; Node 24+; 8 GB Docker memory recommended; two containers plus wildcard DNS; not suitable if Google/Firebase dependency is unacceptable | Strong UI/product reference, not our core |
| SeldonFrame | PostgreSQL; Docker Compose; agency workflow; websites, booking, intake, CRM and AI agents; client workspaces; local-business focus | AGPL-3.0: modified network service must disclose modifications under AGPL unless commercial license is obtained; product is agent/front-office oriented rather than a universal CMS/site platform | Study selected vertical workflows; do not base proprietary platform on it without licensing decision |

## SimplerDevelopment: what was verified in source

### Product surfaces
The architecture describes three audience trees:
- `app/admin/**`: internal agency/staff panel, cross-tenant.
- `app/portal/**`: authenticated tenant/client portal.
- `app/sites/**` and aliases: public websites resolved from request hostname.
- REST v1 and MCP API surfaces for integrations/automation.

The upstream README and architecture describe integrated websites/CMS, CRM, bookings, commerce, campaigns, surveys, e-signatures, workflows, billing, projects/tickets, support, agency onboarding/branding and plugins.

### Tenant resolution and authorization
- A tenant is represented by a client account; a client may have multiple websites.
- User access is resolved through `client_members` membership and/or direct ownership.
- The active-client cookie is only a preference. The resolver intersects it with the user's actual accessible client set; it is not sufficient by itself to grant access.
- Portal authorization resolves the authenticated session and client, then applies role/action and optional service/entitlement checks.
- Tenant-scoped tables carry `clientId` and/or `siteId`; application queries are expected to filter by these identifiers.
- Public sites are resolved by validated host/domain mapping and rendered from published content.
- A dedicated tenancy integration gate exists (`bun test:tenancy`) and is part of upstream CI.

**Important security qualification:** the upstream tenancy design is primarily application-layer scoping, not PostgreSQL Row-Level Security. We must audit every route and query we intend to expose, retain and extend the negative cross-tenant tests, and consider adding database-level defense-in-depth where practical.

### Roles and agency access
- Staff roles include `admin` and `employee`.
- Tenant roles include `owner`, `admin`, `member` and `viewer`.
- The admin layout has a server-side staff session gate. Individual API routes still require correct server-side authorization.
- Staff impersonation exists; it must be audited and logged, and must never influence issuance of durable tenant credentials.

### Booking and calendar
The public booking endpoint:
- validates required visitor fields and requested time;
- enforces minimum notice and maximum advance window;
- validates the requested slot against configured availability;
- checks existing non-cancelled bookings and group capacity;
- supports staff assignment, buffers, group bookings and add-ons in the booking model;
- emits booking-related side effects such as confirmation/host notifications and automation events.

The code includes booking pages, bookings, booking-page members, attendees and capacity helpers. This is a useful foundation for GARAGE 27 and other service businesses.

**Not yet proven:** the exact staff-side manual appointment flow and whether it uses the same conflict/capacity rules as public booking. This must be tested explicitly before we claim the shared calendar requirement is met. If staff-created appointments bypass the public booking engine, we will route both paths through one domain service.

### Data and deployment
- PostgreSQL with Drizzle ORM; pgvector is required for the Company Brain / RAG features.
- Docker Compose starts the app and Postgres, with optional local mail capture, MinIO S3-compatible storage, realtime and agent services.
- Railway deployment documentation exists, but it is not a ready-made Russian VPS deployment guide.
- Upstream production config includes a large list of Vercel Cron schedules. A VPS deployment needs a durable scheduler/worker replacement and an audited inventory of every cron route.
- Optional dependencies include email, AI APIs, Google/Microsoft integrations, Stripe, Redis and S3 storage. They must remain disabled until deliberately configured and reviewed.

### Known hardening items from inspected upstream docs/config
1. The architecture states tenant isolation is enforced in application code; do not assume RLS protects the database.
2. JWT sessions have a documented revocation delay until token expiry.
3. The security overview documents rate-limiter fallback behavior when Redis is unavailable.
4. The security overview lists plaintext-at-rest gaps for certain integration refresh tokens and a CRM enrichment key as hardening targets.
5. Admin API routes and all cross-tenant operations need route-by-route guard review.
6. Cron endpoints, webhook signatures, file uploads, host routing, impersonation and secret encryption need a deployment security pass.
7. AI/RAG, email, SMS and third-party integrations must be optional and disabled in the first Russian pilot unless their data handling is approved.

## Adaptation required for our product

### Preserve from upstream
- Multi-tenant data model and membership/role model (after security audit).
- Admin, portal and public-site route surfaces.
- Block-based CMS / visual editor and site publishing.
- CRM contacts, companies, leads, deals and activities.
- Booking model, availability and capacity helpers.
- S3-compatible media abstraction.
- Subscription/entitlement concepts and audit patterns.
- Existing unit, integration, tenancy and E2E test infrastructure.

### Build or change for our platform
1. **Russian deployment profile:** Docker Compose / VPS, PostgreSQL in the selected Russian region, backups, restore drills, monitoring, TLS, object storage and log retention.
2. **Replace Vercel Cron:** durable scheduler/worker suitable for a long-running VPS process; preserve idempotency and retry semantics for every scheduled job.
3. **GARAGE 27 vertical pack:** auto-service-specific site sections, service catalog, vehicle details, lead forms, CRM pipeline, appointment types, staff/resource calendars and notification templates.
4. **Shared calendar invariant:** staff-created and visitor-created appointments must call the same availability/capacity/conflict service and write to the same source of truth.
5. **Agency subscription model:** our own plans, limits, add-ons, grace periods, suspension and billing-provider adapter; do not hardwire Stripe as the only option.
6. **Client portability:** documented exports for contacts, leads, deals, appointments, site content, media references and audit history.
7. **Russian localization:** Russian UI, ruble prices, local timezone defaults, phone validation, local email/SMS/payment adapters and localized legal notices.
8. **Privacy/compliance:** data inventory, purposes, consent/legal basis, retention/deletion, operator/processor responsibilities, access logs, incident response and a separate 152-FZ review before production data.
9. **Agency UX modernization:** keep our existing agency-admin prototype as a design reference; map its Clients / Projects / Templates / Modules / Plans / Analytics / Support / Team / Audit sections onto the upstream console instead of blindly replacing the interface.
10. **Client experience:** simple client dashboard focused on website, leads, calendar, services/prices, media and settings; hide irrelevant upstream modules by role and entitlement.

## Adoption sequence

### Phase A — Safe technical spike
- Pin an upstream commit/tag; do not track a moving `main` in production.
- Obtain a complete local checkout and run the documented self-host setup in an isolated environment.
- Record exact build/start requirements and all external services.
- Run lint, typecheck, unit, tenancy and critical E2E gates.
- Verify admin login, tenant creation, client invite, site creation, content edit/publish, public lead capture, CRM workflow, visitor booking, staff appointment creation and cross-tenant denial.
- Test backup and restore of PostgreSQL and media.
- Produce a Russian VPS deployment profile with no Vercel runtime dependency.

### Phase B — Adoption decision
Proceed only if:
- clean install/build passes;
- tenant isolation negative tests pass;
- booking conflict tests pass for both public and staff flows;
- data and media can be backed up and restored;
- cron/worker replacement is viable;
- no critical dependency prevents hosting and processing personal data in the selected Russian region;
- license and third-party dependency inventory is acceptable.

### Phase C — Custom implementation
- Use the upstream system as the base rather than maintaining a second parallel app.
- Preserve upstream attribution and license files.
- Keep custom verticals, adapters and UI changes isolated where possible to simplify upstream updates.
- First deliver GARAGE 27; then package reusable vertical modules.

## Current project safety rules
- Work only on `platform/mvp-foundation` until an adoption branch is deliberately established.
- Do not modify `main`, the current production website, or the GARAGE 27 demo.
- Do not create paid infrastructure or upload real customer/personal data during the spike.
- The previous proposed standalone Next.js scaffold under `platform/app/` is paused pending the upstream adoption test.
- Do not claim the upstream project has been built or run locally until that has actually been done.

## Primary upstream references
- SimplerDevelopment: https://github.com/SimplerDevelopment/SimplerDevelopment
- Simpler architecture: https://github.com/SimplerDevelopment/SimplerDevelopment/blob/main/docs/agents/architecture-for-agents.md
- Simpler self-host guide: https://github.com/SimplerDevelopment/SimplerDevelopment/blob/main/docs/getting-started.md
- Simpler Railway guide: https://github.com/SimplerDevelopment/SimplerDevelopment/blob/main/docs/deploy/railway.md
- Aglyn: https://github.com/aglyn/aglyn
- Aglyn self-hosting constraints: https://github.com/aglyn/aglyn/blob/main/docs/SELF_HOSTING.md
- SeldonFrame: https://github.com/seldonframe/seldonframe
- SeldonFrame license: https://github.com/seldonframe/seldonframe/blob/main/LICENSING.md
