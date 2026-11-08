# Technical Stack Decision — MVP

Status: proposed baseline for implementation
Repository: `zigizmundius-svg/ai-web-agency`
Branch: `platform/mvp-foundation`
Decision date: 2026-11-08

## Executive decision

Build the first platform release as a **TypeScript modular monolith using Next.js**, backed by **PostgreSQL**. Package the application with **Docker** so it can be deployed to a Russian VPS/VDS or cloud compute service without redesigning the application.

Do not introduce microservices, Kubernetes, Redis, a message broker or a complex multi-app monorepo in the first release.

## Proposed stack

| Layer | Choice | Reason |
|---|---|---|
| Language | TypeScript | One language across UI and server; safer refactoring |
| Web framework | Next.js App Router | Public pages, client cabinet, agency console and server endpoints in one application |
| UI | React + Tailwind CSS | Responsive admin interfaces and reusable components |
| Component base | Accessible component primitives (e.g. shadcn/ui) | Consistent UI without locking into a hosted service |
| Validation | Zod | Validate browser input and server payloads |
| Database | PostgreSQL | Mature relational database; portable between Russian providers |
| Database access | Drizzle ORM | Typed SQL-oriented schema/migrations and low provider coupling |
| Authentication | Server-side session authentication; select library after a short security review | Avoid custom password/session cryptography |
| File storage | S3-compatible object storage | Portable media storage; private by default |
| Runtime | Node.js LTS in Docker | Portable deployment to VPS/VDS and cloud compute |
| Reverse proxy / TLS | Caddy or provider-managed ingress | HTTPS and domain routing |
| Tests | Vitest + Playwright | Domain/API tests and real browser journeys |
| CI | GitHub Actions | Automated lint, typecheck, tests and build |

Exact library versions must be selected at scaffold time from currently supported stable releases and committed in the lockfile. Do not use floating dependency versions in production.

## Application structure

Keep the existing public agency website at the repository root unchanged during the first implementation stage.

Create the new platform application under `platform/app/` so that:
- the existing root website is preserved;
- the platform can be developed and tested independently;
- the Vercel project serving the current agency website is not accidentally converted or broken;
- the same application can later be deployed as a Docker container to Russian infrastructure.

Initial internal modules:

- `auth`: identity, sessions, recovery.
- `agency`: agency users, roles, tenant and project administration.
- `tenants`: client workspaces and memberships.
- `sites`: site configuration, content and publication.
- `services`: client services and pricing.
- `crm`: leads and lead activity.
- `media`: file metadata and S3 adapter.
- `audit`: security and business event log.

Keep domain logic out of UI components. Route handlers/server actions call application services; application services enforce authorization and call repositories.

## Environment model

- Local development: Docker Compose for PostgreSQL and the app; synthetic fixture data only.
- Preview/staging: isolated database and storage bucket; no production customer data.
- Production: Russian-region application hosting, database and object storage after provider and legal review.

Secrets are supplied through environment variables or a secret manager. Never commit `.env` files, passwords, API keys or database URLs.

## Deployment portability

The application must:
- build as a Docker image;
- expose a health-check endpoint;
- use environment-based configuration;
- run database migrations as an explicit release step;
- support graceful shutdown;
- log structured events without exposing secrets or unnecessary personal data.

Do not make the application depend on Vercel-only services. Vercel may continue to host previews and the existing marketing site, but production customer data must not be placed in a foreign-hosted service by default.

## Security baseline

- Secure, HTTP-only, same-site cookies for sessions.
- Password hashing and session handling delegated to a reviewed authentication library.
- Server-side authorization on every protected operation.
- Tenant context derived from verified membership.
- PostgreSQL constraints and tenant-scoped queries; consider RLS as defense in depth.
- Zod validation at every external input boundary.
- Rate limiting for authentication and public forms.
- CSRF/XSS protections appropriate to the chosen framework and session model.
- Private object storage and short-lived signed upload/download URLs.
- Audit important access, permission and data-changing operations.
- Automated cross-tenant authorization tests.

## Implementation sequence

1. Scaffold the isolated Next.js application in `platform/app/`, leaving root files untouched.
2. Add linting, formatting, type checking, test runner and CI checks.
3. Add local PostgreSQL via Docker Compose and initial Drizzle migrations.
4. Implement identity and role/membership model.
5. Implement tenant isolation and automated negative tests before building business screens.
6. Implement agency tenant/project creation.
7. Implement client cabinet content editing.
8. Connect the public GARAGE 27 site to published tenant content.
9. Implement lead capture and CRM workflow.
10. Prepare Russian-provider deployment only after the provider, account owner, regions, backups and legal responsibilities are confirmed.

## Explicitly not decided yet

- VPS/VDS provider and server size.
- Managed versus self-hosted PostgreSQL.
- Authentication library and email delivery provider.
- S3 provider and exact region.
- Production domain and DNS.
- Billing/payment provider.
- Final personal-data operator/processor contract model.

These choices do not block a local development scaffold, but they do block production deployment and real personal-data processing.
