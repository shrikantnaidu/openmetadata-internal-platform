# OpenMetadata Data and Governance Examples

These examples use a fictional retail company called **Acme Retail**. They show the kind of metadata that can be modeled in OpenMetadata after a database or warehouse service has been ingested.

## 1. Example catalog structure

```text
Acme Warehouse
└── analytics
    ├── raw
    │   ├── customers
    │   └── orders
    ├── staging
    │   ├── customers_clean
    │   └── orders_clean
    └── mart
        ├── daily_revenue
        └── customer_lifetime_value
```

A practical setup is:

| Asset | Purpose | Owner | Domain | Tier |
|---|---|---|---|---|
| `raw.customers` | Source customer records | Data Platform | Customer | Tier 3 |
| `raw.orders` | Source order events | Data Platform | Sales | Tier 3 |
| `staging.orders_clean` | Validated and standardized orders | Analytics Engineering | Sales | Tier 2 |
| `mart.daily_revenue` | Daily revenue reporting | Sales Analytics | Finance | Tier 1 |
| `mart.customer_lifetime_value` | Customer value analysis | Marketing Analytics | Customer | Tier 2 |

In the UI, first add the source service and run metadata ingestion. Then add the owners, domains, tiers, descriptions, and tags to the resulting assets.

## 2. Example table and column metadata

Example for `mart.daily_revenue`:

| Column | Description | Example governance metadata |
|---|---|---|
| `revenue_date` | Calendar date for the revenue total | Required; business key |
| `order_count` | Number of completed orders | Non-sensitive; quality metric |
| `gross_revenue` | Revenue before discounts and refunds | Financial; Tier 1 |
| `refund_amount` | Total refunds recorded for the date | Financial; Tier 1 |
| `net_revenue` | `gross_revenue - refund_amount` | Certified business metric |
| `currency_code` | ISO currency code | Required; controlled value |

Use descriptions to explain business meaning, not only the physical data type. Add sample SQL or calculation logic to the description of important derived metrics such as `net_revenue`.

## 3. Example classifications and tags

Create classifications for reusable governance labels:

```text
PII
├── Sensitive
├── Personal
└── Confidential

DataQuality
├── Required
├── Unique
├── FreshnessCritical
└── Certified

Criticality
├── Tier1
├── Tier2
└── Tier3
```

Apply tags such as:

| Tag | Apply to | Meaning |
|---|---|---|
| `PII.Sensitive` | `raw.customers.email`, `raw.customers.phone` | Directly identifying customer data |
| `PII.Personal` | `raw.customers.address` | Personal information requiring handling controls |
| `DataQuality.Required` | `orders_clean.order_id`, `orders_clean.customer_id` | Must not be null |
| `DataQuality.Unique` | `orders_clean.order_id` | Must be unique |
| `DataQuality.Certified` | `mart.daily_revenue.net_revenue` | Approved for trusted reporting |
| `Criticality.Tier1` | `mart.daily_revenue` | Business-critical reporting asset |

A classification describes a category of tags. A tag describes the specific governance property applied to an asset or column.

## 4. Example business glossary

Create glossary terms that business users can understand:

| Term | Definition | Synonyms | Related assets |
|---|---|---|---|
| Customer | A person or organization with at least one registered account | Account holder | `customers`, `customer_lifetime_value` |
| Completed Order | An order that has passed payment and fulfillment requirements | Fulfilled order | `orders_clean`, `daily_revenue` |
| Gross Revenue | Total order value before refunds and discounts | Booked sales | `daily_revenue.gross_revenue` |
| Net Revenue | Gross revenue less refunds and approved adjustments | Realized revenue | `daily_revenue.net_revenue` |
| Data Product | A maintained data asset with an owner, documentation, and quality expectations | Curated dataset | `mart.daily_revenue` |

Assign glossary terms to tables and columns so users can discover assets by business meaning rather than only by physical names.

## 5. Example ownership and domains

Create teams or users such as:

```text
Data Platform
├── platform.owner@acme.example
└── platform.oncall@acme.example

Sales Analytics
├── sales.analytics@acme.example
└── revenue.owner@acme.example

Privacy Office
└── privacy@acme.example
```

Suggested domain ownership:

| Domain | Owner | Example assets |
|---|---|---|
| Customer | Customer Data team | Customer profiles and segmentation |
| Sales | Sales Analytics team | Orders and revenue |
| Finance | Finance Data team | Revenue and financial reporting |
| Privacy | Privacy Office | PII classifications and policies |

Ownership answers who is accountable for an asset. Domains group related assets for discovery and governance; they do not replace ownership.

## 6. Example lineage

Represent the flow from source data to a business report:

```text
raw.orders
    └──> staging.orders_clean
              └──> mart.daily_revenue
                            └──> Executive Revenue Dashboard

raw.customers
    └──> staging.customers_clean
              └──> mart.customer_lifetime_value
                            └──> Customer Value Dashboard
```

Add lineage from the UI or from a supported ingestion connector. Include descriptions at important transformation points, for example:

```text
net_revenue = gross_revenue - refund_amount - approved_adjustments
```

Use lineage during impact analysis. Before changing `raw.orders.order_status`, inspect downstream tables, dashboards, owners, and quality tests.

## 7. Example data quality checks

For `staging.orders_clean`:

| Check | Rule | Expected result |
|---|---|---|
| Row count | Table contains at least one row | `count > 0` |
| Required field | `order_id` is not null | Null count = 0 |
| Uniqueness | `order_id` is unique | Duplicate count = 0 |
| Referential integrity | Every `customer_id` exists in customers | Invalid references = 0 |
| Freshness | Latest `order_timestamp` is less than 2 hours old | Pass within SLA |
| Accepted values | `order_status` is one of `completed`, `cancelled`, `refunded` | Invalid values = 0 |

For `mart.daily_revenue`:

- A row exists for every expected reporting date.
- `net_revenue` is not negative unless refunds exceed sales by an approved exception.
- `currency_code` is populated and valid.
- The table is refreshed before the daily reporting SLA.

Configure these as data quality or profiler tests where supported by the source connector. Give failed checks an owner and an escalation path.

## 8. Example certification workflow

A simple promotion process can be:

```text
Ingested → Documented → Quality Checked → Certified → Deprecated
```

For a table to become certified:

1. Assign an owner and domain.
2. Add a useful description and usage guidance.
3. Document important columns and metrics.
4. Apply sensitivity and criticality tags.
5. Add lineage to upstream sources and downstream dashboards.
6. Configure the required quality checks.
7. Review the checks with the domain owner.
8. Mark the table or metric as certified.

When the source is replaced, quality checks fail repeatedly, or the owner confirms it is no longer used, mark the asset deprecated and document the replacement asset.

## 9. Example access and governance policy ideas

Use roles and policies to separate responsibilities:

| Role | Typical capability |
|---|---|
| Data Consumer | Discover and read catalog metadata |
| Data Steward | Maintain descriptions, tags, glossary links, and quality expectations |
| Domain Owner | Approve certification and governance decisions for a domain |
| Platform Admin | Configure services, users, roles, and system settings |

Example policy rules:

- Only the Privacy Office can approve changes to PII classifications.
- Data Stewards can edit descriptions and tags for assets in their domain.
- Domain Owners can certify or deprecate assets in their domain.
- Platform Admins manage integrations and permissions but should not own business definitions by default.

These are governance conventions; implement the exact permissions using your organization’s OpenMetadata roles and policies.

## 10. Example announcements and incidents

When a breaking change is planned for `mart.daily_revenue`:

```text
Announcement: daily_revenue schema change

The column `refund_amount` will change from decimal(18,2) to decimal(20,4)
on 2026-10-15. Consumers should validate casts and dashboard formatting.
Owner: Sales Analytics
Replacement contact: revenue.owner@acme.example
```

For a quality incident:

```text
Incident: daily_revenue freshness SLA missed

Impact: Executive Revenue Dashboard is delayed.
Start: 08:15 UTC
Owner: Data Platform
Status: Investigating
Related asset: mart.daily_revenue
Related check: FreshnessCritical
```

Keep announcements and incidents linked to the affected assets so consumers can see operational context while browsing the catalog.

## 11. A small end-to-end practice exercise

Use this sequence to practice the main features:

1. Ingest a development database containing `customers` and `orders`.
2. Create the `Customer` and `Sales` domains.
3. Assign the Data Platform team as owner of the source tables.
4. Create the `PII` and `DataQuality` classifications.
5. Tag customer email and phone columns with `PII.Sensitive`.
6. Create glossary terms for Customer, Gross Revenue, and Net Revenue.
7. Add descriptions to `orders_clean` and `daily_revenue`.
8. Add lineage from source tables to the reporting table.
9. Add nullness, uniqueness, and freshness checks.
10. Certify `daily_revenue` only after the checks pass.
11. Search for `Net Revenue` and verify the glossary, tags, owner, lineage, and quality information are all visible.

This exercise produces a small but realistic governed catalog without requiring production data.

## 12. Data handling reminder

Use synthetic or non-sensitive development data for demonstrations. Do not upload real customer PII to a test instance. If sensitive data must be cataloged, document the metadata and apply the appropriate tags without storing raw values in descriptions, sample data, or screenshots.