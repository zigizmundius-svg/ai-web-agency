# AI Web Agency — Platform Foundation

## Purpose

Build a multi-tenant SaaS platform for small and medium-sized businesses in Russia. The first pilot is an auto service (GARAGE 27). The platform must let the agency create and maintain client projects while each client manages their own website and business data through a browser-based cabinet.

This repository's `main` branch contains the existing agency website and its visual admin prototype. Platform work is isolated on `platform/mvp-foundation`.

## Product boundaries

### Agency workspace
- Create, configure, suspend and archive client workspaces.
- Assign a business niche/template to a workspace.
- Enable or disable modules for a workspace.
- Manage plans, billing status, access and technical health.
- Support client accounts without entering their passwords.

### Client workspace
- Manage website pages, text, images, contacts, services and prices.
- View and process incoming enquiries.
- Manage staff access within the permissions granted by the agency.
- Use optional modules such as online booking and CRM.
- Export the client's data in a usable format.

### Public website
- Public pages are served independently from the admin application.
- Website forms create records in the same tenant's CRM.
- Public booking and manual admin booking must use the same availability rules.
- A confirmed booking blocks the slot for every other channel.

## MVP scope — GARAGE 27

### Included
1. Agency creates a GARAGE 27 workspace.
2. Agency assigns an owner account.
3. Owner signs in to a web cabinet.
4. Owner edits a controlled set of website fields: business contacts, opening hours, services, prices, descriptions and gallery.
5. Public website displays the saved content.
6. Website enquiry form creates a lead in the correct workspace.
7. Owner can view leads and change their statuses.
8. Agency can see workspace status and provide support.

### Not in the first release
- Online payments and automatic subscription renewals.
- Complex accounting, inventory and warehouse management.
- Multi-branch/franchise support.
- AI content generation inside the cabinet.
- Native mobile apps.
- Complex workflow automation.

Online booking is a planned next module. Its availability model must be designed before implementation so that public and admin bookings cannot create overlapping reservations.

## Architecture principles

1. **Multi-tenancy from the beginning.** Every business-owned record carries a `tenant_id`. Server-side authorization derives tenant access from the authenticated membership, never from a client-supplied tenant identifier alone.
2. **Separate agency and client permissions.** Agency operators and client users have different roles and permission scopes.
3. **Server is authoritative.** UI visibility is not security. Every read and write is checked on the server.
4. **Russian personal-data requirements.** The production database and primary storage for personal data must be selected with Russian data-localization requirements in mind. Hosting location alone does not establish legal compliance.
5. **Portability.** A tenant can export its content, leads and relevant business records. Avoid proprietary data formats where practical.
6. **No secrets in frontend code.** Credentials and privileged keys remain server-side.
7. **Auditability.** Important changes to users, permissions, leads, bookings and workspace settings should be logged.
8. **Progressive delivery.** Ship a small working vertical slice before adding more modules.

## Initial domain model

- `users`: authenticated people.
- `tenants`: client business workspaces.
- `memberships`: user-to-tenant relationship and role.
- `agency_memberships`: agency staff and agency-level permissions.
- `sites`: public website configuration for a tenant.
- `site_pages`: editable page content.
- `services`: tenant services and prices.
- `media_assets`: uploaded images and file metadata.
- `leads`: enquiries from forms and other channels.
- `lead_events`: lead status and activity history.
- `modules`: available platform modules.
- `tenant_modules`: modules enabled for each tenant.
- `audit_log`: security- and business-relevant actions.

Future booking entities:
- `resources`: employees, service bays or other bookable resources.
- `availability_rules`: working hours and exceptions.
- `appointments`: bookings with explicit status and time range.
- `appointment_events`: reschedules, cancellations and status history.

Every tenant-owned table must have tenant-scoped access rules and indexes. Cross-tenant access is denied by default.

## Roles — initial proposal

- `platform_owner`: full platform control.
- `agency_admin`: manage projects, tenants, modules and agency staff within granted scope.
- `agency_manager`: manage assigned client projects, but not platform-wide security or billing settings.
- `tenant_owner`: full control over their own business workspace.
- `tenant_manager`: manage assigned business operations.
- `tenant_staff`: limited access to assigned operational functions.

Role names are a starting point; permissions should be explicit capabilities, not scattered hard-coded role checks.

## Infrastructure decision — pending

Do not connect production personal data to a foreign-hosted database by default. Before choosing a provider, compare Russian hosting/database options against:
- database location and backup location;
- object storage location;
- authentication and email/SMS integration options;
- managed PostgreSQL availability and operational burden;
- access controls, encryption, backups and recovery;
- deployment automation and predictable costs;
- contractual and legal terms relevant to personal-data processing.

GitHub is used for source control only. It is not the production database or file store.

## Delivery sequence

### Phase 0 — foundation
- Confirm deployment and Russian infrastructure.
- Define environments: local/development, staging and production.
- Establish secrets handling and deployment process.
- Define database schema and tenant isolation tests.

### Phase 1 — first working vertical slice
- Authentication and session handling.
- Tenant and membership creation.
- Agency creates a tenant and invites an owner.
- Owner edits GARAGE 27 website content.
- Public website reads published tenant content.
- Enquiry form writes a lead.
- Owner processes leads in the cabinet.

### Phase 2 — operational modules
- Online booking with shared availability logic.
- Staff and resource schedules.
- Notifications.
- Data export and audit history.

### Phase 3 — commercial platform
- Plans and module entitlements.
- Subscription billing.
- Usage limits and support tools.
- Additional industry templates.

## Definition of done for the MVP

The MVP is not considered complete until:
- a client can sign in without agency assistance;
- one tenant cannot read or change another tenant's data;
- a client can update website content without editing code;
- public forms create correctly scoped leads;
- the agency can manage client workspaces;
- backups and a restore procedure exist;
- data export works;
- core flows are tested on desktop and mobile.

## Current status

- [x] Existing agency site preserved on `main`.
- [x] Existing admin HTML prototype inspected; it is a demo UI, not a connected backend.
- [x] Dedicated branch `platform/mvp-foundation` created.
- [x] Initial platform scope and architecture recorded.
- [ ] Select Russian production infrastructure.
- [ ] Implement backend, database and authentication.
- [ ] Connect the first client cabinet and GARAGE 27 website.
