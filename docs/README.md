# Project documentation: Generic Billing System

Original design document: [`../generic-billing-system-sdd.md`](../generic-billing-system-sdd.md) (the **original SDD**). Documents are written in English; see master plan §0.1.

## Start here

1. [00-master-plan.md](00-master-plan.md): the master plan. Gaps in the SDD, every document to write with its gate, every task by phase, and traceability.
2. [00-decision-register.md](00-decision-register.md): the decision register. Every place where the SDD does not say exactly how, each with one proposed option. Read this first if you are deciding; start with its "Summary by impact".

Reading order for newcomers: master plan §1 → §4 → the current phase in §5 → the documents that phase references.

## Documents

Paths are relative to `docs/`. A path becomes a link when its document is written.

| DOC | Document | Status |
| --- | --- | --- |
| DOC-01 | Vision and scope (`01-product/vision-and-scope.md`) | Not written |
| DOC-02 | Personas and journeys (`01-product/personas-and-journeys.md`) | Not written |
| DOC-03 | Requirements (`01-product/requirements.md`) | Not written |
| DOC-04 | Use cases (`01-product/use-cases.md`) | Not written |
| DOC-05 | Feature catalog (`01-product/feature-catalog.md`) | Not written |
| DOC-06 | Glossary (`02-glossary.md`) | Not written |
| DOC-07 | System context and containers (`03-architecture/system-context-and-containers.md`) | Not written |
| DOC-08 | Data flows (`03-architecture/data-flows.md`) | Not written |
| DOC-09 | Messaging contracts (`03-architecture/messaging-contracts.md`) | Not written |
| DOC-10 | Quality attributes (`03-architecture/quality-attributes.md`) | Not written |
| DOC-11 | Tech stack and versions (`03-architecture/tech-stack-and-versions.md`) | Not written |
| DOC-12 | Code architecture (`03-architecture/code-architecture.md`) | Not written |
| DOC-13 | ADR index (`04-adr/README.md`) | Not written |
| DOC-14 | Ledger model (`05-data/ledger-model.md`) | Not written |
| DOC-15 | Ops model (`05-data/ops-model.md`) | Not written |
| DOC-16 | Delivery model (`05-data/delivery-model.md`) | Not written |
| DOC-17 | Usage store (`05-data/usage-store.md`) | Not written |
| DOC-18 | Data quality rules (`05-data/data-quality-rules.md`) | Not written |
| DOC-19 | DB roles and grants (`05-data/db-roles-and-grants.md`) | Not written |
| DOC-20 | Data lifecycle (`05-data/data-lifecycle.md`) | Not written |
| DOC-21 | Wallet and Ledger (`06-design/wallet-and-ledger.md`) | Not written |
| DOC-22 | Balance Reservation (`06-design/balance-reservation.md`) | Not written |
| DOC-23 | Idempotency (`06-design/idempotency.md`) | Not written |
| DOC-24 | Outbox and relay (`06-design/outbox-and-relay.md`) | Not written |
| DOC-25 | Shard routing (`06-design/shard-routing.md`) | Not written |
| DOC-26 | Multi-tenancy (`06-design/multi-tenancy.md`) | Not written |
| DOC-27 | Pricing Engine (`06-design/pricing-engine.md`) | Not written |
| DOC-28 | Usage Ingestion (`06-design/usage-ingestion.md`) | Not written |
| DOC-29 | Usage Rating (`06-design/usage-rating.md`) | Not written |
| DOC-30 | Webhook Dispatcher (`06-design/webhook-dispatcher.md`) | Not written |
| DOC-31 | Catch-up feed (`06-design/catch-up-feed.md`) | Not written |
| DOC-32 | Payment and Refund (`06-design/payment-and-refund.md`) | Not written |
| DOC-33 | Invoicing and Subscription (`06-design/invoicing-and-subscription.md`) | Not written |
| DOC-34 | Reconciliation (`06-design/reconciliation.md`) | Not written |
| DOC-35 | Audit and digests (`06-design/audit-and-digests.md`) | Not written |
| DOC-36 | ERP export (`06-design/erp-export.md`) | Not written |
| DOC-37 | Scheduler and jobs (`06-design/scheduler-and-jobs.md`) | Not written |
| DOC-38 | API Gateway (`06-design/api-gateway.md`) | Not written |
| DOC-39 | Demo Client (`06-design/demo-client.md`) | Not written |
| DOC-40 | Coverage matrix (`06-design/coverage-matrix.md`) | Not written |
| DOC-41 | Security (`06-design/security.md`) | Not written |
| DOC-42 | Observability (`06-design/observability.md`) | Not written |
| DOC-43 | Configuration reference (`06-design/configuration-reference.md`) | Not written |
| DOC-44 | Error handling (`06-design/error-handling.md`) | Not written |
| DOC-45 | Flows index (`06-design/flows/README.md`) | Not written |
| DOC-46 | Flows: budget control (`06-design/flows/budget-control.md`) | Not written |
| DOC-47 | Flows: usage and rating (`06-design/flows/usage-and-rating.md`) | Not written |
| DOC-48 | Flows: integration (`06-design/flows/integration.md`) | Not written |
| DOC-49 | Flows: payments and refunds (`06-design/flows/payments-and-refunds.md`) | Not written |
| DOC-50 | Flows: invoicing (`06-design/flows/invoicing.md`) | Not written |
| DOC-51 | Flows: administration and operations (`06-design/flows/administration.md`) | Not written |
| DOC-52 | API guidelines (`07-api/api-guidelines.md`) | Not written |
| DOC-53 | API endpoints (`07-api/api-endpoints.md`) | Not written |
| DOC-54 | Webhook events (`07-api/webhook-events.md`) | Not written |
| DOC-55 | Client integration guide (`07-api/client-integration-guide.md`) | Not written |
| DOC-56 | UX principles and IA (`08-ux-ui/ux-principles-and-ia.md`) | Not written |
| DOC-57 | Design system (`08-ux-ui/design-system.md`) | Not written |
| DOC-58 | UI states and copy (`08-ux-ui/ui-states-and-copy.md`) | Not written |
| DOC-59 | Screens index (`08-ux-ui/screens/README.md`) | Not written |
| DOC-60 | Screen: Account overview (`08-ux-ui/screens/account-overview.md`) | Not written |
| DOC-61 | Screen: Top up and payment result (`08-ux-ui/screens/top-up.md`) | Not written |
| DOC-62 | Screen: Transactions (`08-ux-ui/screens/transactions.md`) | Not written |
| DOC-63 | Screen: Invoices (`08-ux-ui/screens/invoices.md`) | Not written |
| DOC-64 | Screen: Refunds (`08-ux-ui/screens/refunds.md`) | Not written |
| DOC-65 | Screen: Finance overview (`08-ux-ui/screens/finance-overview.md`) | Not written |
| DOC-66 | Screen: Price plans (`08-ux-ui/screens/price-plans.md`) | Not written |
| DOC-67 | Screen: Refund approvals (`08-ux-ui/screens/refund-approvals.md`) | Not written |
| DOC-68 | Screen: Reconciliation (`08-ux-ui/screens/reconciliation.md`) | Not written |
| DOC-69 | Screen: Tenants and limits (`08-ux-ui/screens/tenants-and-limits.md`) | Not written |
| DOC-70 | Screen: Client Systems and grants (`08-ux-ui/screens/client-systems.md`) | Not written |
| DOC-71 | Screen: Webhook deliveries (`08-ux-ui/screens/webhook-deliveries.md`) | Not written |
| DOC-72 | Local development (`09-operations/local-dev.md`) | Not written |
| DOC-73 | Deploy: mini (`09-operations/deploy-mini.md`) | Not written |
| DOC-74 | Deploy: staging (`09-operations/deploy-staging.md`) | Not written |
| DOC-75 | Deploy: production (`09-operations/deploy-production.md`) | Not written |
| DOC-76 | CI/CD (`09-operations/ci-cd.md`) | Not written |
| DOC-77 | Backup and restore (`09-operations/backup-restore.md`) | Not written |
| DOC-78 | Scaling playbook (`09-operations/scaling-playbook.md`) | Not written |
| DOC-79 | Runbooks index (`09-operations/runbooks/README.md`) | Not written |
| DOC-80 | Test strategy (`10-testing/test-strategy.md`) | Not written |
| DOC-81 | Experiments index (`10-testing/experiments/README.md`) | Not written |
| DOC-82 | Demo script (`10-testing/demo-script.md`) | Not written |

ADRs (`04-adr/NNNN-*.md`), runbooks (`RB-xx`, under DOC-79) and experiments (`EXP-xx`, under DOC-81) are listed in master plan §3.2.

## Structure

| Folder | Content |
| --- | --- |
| `01-product/` | Vision, personas, requirements, use cases, features |
| `02-glossary.md` | The single definition of every term |
| `03-architecture/` | Containers, data flows, messaging contracts, quality attributes, stack, code architecture |
| `04-adr/` | Architecture decisions |
| `05-data/` | DDL per store, data quality, roles and grants, lifecycle |
| `06-design/` | One document per module, cross-cutting documents, and `flows/` |
| `07-api/` | API guidelines, endpoints, webhook events, Client integration guide |
| `08-ux-ui/` | UX principles, design system, copy, and `screens/` |
| `09-operations/` | Local development, deployment per environment, CI/CD, backup, scaling, and `runbooks/` |
| `10-testing/` | Test strategy, `experiments/`, demo script |

## Conventions

- The first lines of every file give its status (`Draft | Review | Approved | Superseded`) and update date.
- Diagrams are Mermaid.
- Identifiers follow master plan §0.
- A behavior change edits the docs in the same PR. A decision change gets a new ADR that supersedes the old one.
