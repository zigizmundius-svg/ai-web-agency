# Russian Infrastructure Review — November 2026

## Decision context

The platform targets businesses in Russia and will process customer enquiries and, later, appointment data. The architecture therefore assumes that Russian citizens' personal data must be collected, systematized, accumulated, stored, updated and retrieved using databases located in Russia, subject to the exceptions and other conditions in applicable law. A provider's marketing statement or a Russian server location does not by itself make the operator compliant.

## Providers reviewed

### Selectel

**Verified product capabilities**
- Managed PostgreSQL is offered as a managed cloud database service.
- Selectel states that it operates data centers in Russia, including Moscow, Saint Petersburg/Leningrad region and a partner data center in Novosibirsk.
- Managed PostgreSQL product information lists SLA 99.95%, 24/7 support, backups, and API/Terraform management.
- S3 documentation says the service is designed for object storage, is available in multiple regions/pools, and by default complies with 152-FZ. Selectel distinguishes its infrastructure responsibilities from the customer's responsibility for bucket permissions, integration configuration and legal compliance.
- S3 supports the Amazon S3 API. The documentation describes multi-AZ pool ru-6 with synchronous replicas across three physically isolated data centers; other pools have different resilience properties.

**Strengths for this project**
- A reasonably complete combination of cloud compute, managed PostgreSQL and S3 storage.
- Russian data-center footprint and explicit product documentation around 152-FZ.
- API/Terraform options support repeatable deployment.

**Questions before production**
- Confirm the exact Managed PostgreSQL region/pool and the location of automated backups.
- Confirm the S3 pool selected for tenant media and backups.
- Price the smallest practical production configuration, including compute, database, storage, traffic, backups and support.
- Confirm account registration, contract/payment method and whether the account can be operated by the project owner.

### Yandex Cloud

**Verified product capabilities**
- Managed PostgreSQL supports versions 14–18 and publishes Russia-region pricing.
- Managed PostgreSQL pricing includes compute, storage, backups and outbound traffic; the first 100 GB of monthly outbound traffic is not charged according to current documentation.
- Yandex Object Storage is S3-compatible. Its documentation says data is stored across geographically distributed data centers in Russia.
- Yandex Cloud documents regional separation and states that user data is stored within the selected region.

**Strengths for this project**
- Broad, mature cloud ecosystem and managed database service.
- Clear documentation for PostgreSQL and object storage.
- Potentially convenient integration between application hosting, database, storage, logging and monitoring.

**Questions before production**
- Confirm account onboarding and payment eligibility for the actual contracting person/entity.
- Confirm the exact Russian region for every service, including backups and logs.
- Calculate the smallest practical deployment using the current calculator.

### VK Cloud

**Verified product capabilities**
- Cloud Databases offers managed PostgreSQL, among other engines, with single, master-replica and cluster deployment patterns.
- VK Cloud describes backup configuration, monitoring and scaling options for managed databases.
- VK Cloud's Object Storage is S3-compatible; its product material states that data centers are in Russia and describes 152-FZ-related attestation for its cloud storage offering.
- VK Cloud describes two availability zones in Russia and a platform SLA of 99.95%.

**Strengths for this project**
- Managed PostgreSQL and S3-compatible storage are both available.
- Russian cloud infrastructure and managed database operations.
- Could be cost-effective depending on the chosen database configuration.

**Questions before production**
- Confirm exact database and backup locations, as well as the chosen Object Storage region.
- Confirm the precise scope of the cited 152-FZ attestation for the selected service and deployment.
- Price the smallest practical configuration and verify account/payment onboarding.

## Preliminary recommendation

Keep the application portable and PostgreSQL-based. Do not bind the code to a provider-specific database API.

**First provider to price and validate: Selectel.** It is a practical initial candidate because managed PostgreSQL, S3, compute and API/Terraform workflows are documented together, with Russian regions and explicit product-level 152-FZ statements. This is a candidate, not yet a final production decision.

**Second candidate: Yandex Cloud.** It is a strong alternative if its onboarding and payment process is workable for the project owner and the full deployment cost is competitive.

**Third candidate: VK Cloud.** Keep it in the shortlist, especially if its managed database and storage quote is materially better.

No production database or personal data should be created until the account, region, backup location, contract and costs have been checked.

## Recommended deployment shape

- Application: a modular monolith (one deployable application with clear internal modules), not microservices.
- Database: PostgreSQL, initially one managed instance; tenant isolation enforced in the application and database policies/tests.
- File storage: S3-compatible object storage, private by default; public delivery only for explicitly published website assets.
- Backups: database backups plus a separate export/backup policy for tenant files; test restoration before production.
- Source code: GitHub.
- Environments: development/staging and production must be separate.
- Secrets: provider secret manager or server-side environment variables; never commit credentials.

## Legal and operational caveat

The platform owner and each client must establish the correct personal-data roles and contractual arrangements for their actual processing model. Before accepting real personal data, review the operator's obligations, notices/consents, privacy policy, processing поручение/contract, security measures, incident handling, retention/deletion and any cross-border transfer. Infrastructure selection is only one part of that work.

## Sources

- Selectel Managed PostgreSQL: https://selectel.ru/services/cloud/managed-databases/postgresql/
- Selectel Managed Databases documentation: https://docs.selectel.ru/managed-databases/
- Selectel S3 product and 152-FZ scope: https://docs.selectel.ru/s3/about/about-s3/
- Selectel S3 replication pools: https://docs.selectel.ru/en/s3/about/about-s3/
- Yandex Managed PostgreSQL: https://yandex.cloud/ru/docs/managed-postgresql/
- Yandex PostgreSQL pricing: https://yandex.cloud/ru/docs/managed-postgresql/pricing
- Yandex Object Storage data location: https://yandex.cloud/ru/docs/storage/qa
- VK Cloud Cloud Databases: https://land.vkcloud-static.ru/databases/
- VK Cloud Object Storage: https://land.vkcloud-static.ru/storage/
- VK Cloud technical characteristics: https://land.vkcloud-static.ru/properties/
