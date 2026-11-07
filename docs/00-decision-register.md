# Decision Register

> Status: **Review** · Updated: 2026-11-07 · Source: analysis of `generic-billing-system-sdd.md` (the **original SDD**)

> **Terminology (2026-11-07).** Tenancy terms follow **DR-112** (*Unified tenant model, B2B2C*). The **tenant** is the business and the isolation unit; the **tenant_user** is its staff; the **tenant_customer** is its customer and holds the wallet; the **tenant_system** is the tenant's service integration; the **platform_user** operates the app. Older entries that say "Client", "Client System", "Customer" or "Tenant" are read through the mapping in DR-112.

The original SDD is strong on *what* and *why*: the money invariants, the wallet lock, memo holds, the outbox, cumulative rating and the descriptive-events rule all come with options, reasons and experiments. It is weaker on *exactly how* where mechanisms meet. Several constraints it states cannot hold together as written (a partitioned table with a unique key that omits the partition key, a pricing model that breaks the "never a negative delta" rule, row-level security that hides the row needed to find the tenant). Guessing wrong at the data and contract layer is expensive to undo once Clients integrate, so these items are decided before any code.

**How to use**

- Each entry has a state: `Proposed` → (Owner reviews) → `Decided` or `Changed` (record the alternative).
- A decided entry is carried into its target documents (the *Write to* line). Architecture-level entries become ADRs in `docs/04-adr/`.
- **⚠ deviates from the original SDD**: the proposal differs from the SDD; the reason is given. The SDD file itself is never edited.
- **🔬 spike**: needs a short experiment (≤ 2 days) before deciding; the `S-xx` task is in master plan Phase 0. **All spikes are deferred** (decision log, 2026-11-07): a 🔬 entry may be decided on its proposal, and its values are tagged `unverified (S-xx)` until the spike runs.
- Every entry below is `Proposed` unless its title says otherwise.

## Decision log

| Date | Decided by | Content | Affected items |
| --- | --- | --- | --- |
| 2026-11-06 | Owner | Documents in English; code, UI strings, logs, API errors and commits in English | Master plan §0.1, every document |
| 2026-11-06 | Owner | Scope: build everything, stages S0–S8 of the original SDD; the SDD's cut order stays as the contingency | Master plan §4 |
| 2026-11-07 | Owner | All nine spikes S-01…S-09 are deferred. P0-02…10 become `Deferred` and do not gate M0. Documents use the proposed values, tagged `unverified (S-xx)`. Each spike runs before the task named in master plan §5, Phase 0 | Master plan §1.3, §4.1, §5, §8, §9; DR-06, 10, 13, 15, 18, 26, 36, 37, 55, 59, 70, 92, 96 |
| 2026-11-07 | Owner | New requirement A: the Finance Admin grants each tenant_system the pricing modes it may use; a Client may in turn give each of its users a subset of those modes. Revoking a tenant's mode only marks it expired and leaves every resource already created intact; revoking a user is immediate, since resources can always be assigned to another user | Group S (DR-104, 105, 107, 108); FR-34, 35, 37; master plan §1.3, §3, §5 |
| 2026-11-07 | Owner | Clarification: there is no admin versus staff split and no named roles; every user of the system is permission-based. A user's "kind" is only where they belong (the platform or one Client), which limits their bits. Customers and tenant_system credentials are unchanged | DR-64, 66, 106, 107, 109; FR-35, 36 |
| 2026-11-07 | Owner | New requirement C: platform and Client are two separate spaces, separate in use cases and not shared, with separate users, permissions and dashboards. A tenant's dashboard is addressed `client1.<domain>` or `<domain>/client1` (both supported if possible); the platform `platform.<domain>` or `<domain>/platform`. The same email may be reused across the platform and Clients | DR-106…111; FR-38, 39; DOC-84…90 |
| 2026-11-07 | Owner | New requirement B: authorization is permission-based, by group, function and action, implemented as a binary mask (bitmask) | DR-106, 109; FR-36; ADR-0026; DOC-83 |
| 2026-11-06 | Owner | While writing documents, Claude may decide small questions itself and record them as delegated DRs (naming, internal defaults). Money semantics, contracts and cost stay with the Owner | Every later DR |
| 2026-11-07 | Owner | Unified tenant model (B2B2C): hierarchy is platform_user → tenant (the business and isolation unit) → tenant_customer (wallet holder) → tenant_system (the business service). Supersedes the Client/Tenant/Customer naming; full model and identifier mapping in DR-112 | DR-112; DR-62…111 |

---

## A. Scope, market and legal

### DR-01 · Which market and currencies does the first release serve?
- **Problem:** Original SDD §22.2 assumes "Vietnam and VND first" from context and leaves it open. Minor-unit handling, invoice wording, PSP choice and retention all depend on it.
- **Decision (proposed):** Vietnam first, VND first. `Currency` in `shared.kernel` is a closed table of ISO 4217 codes with their exponents; the first release ships `VND` (0 decimals) and `USD` (2 decimals), and adding a currency is a code change. An account's currency is immutable. A tenant holding two currencies has two accounts (original SDD §2.3).
- **Consequences:** `USD` is present so that tests exercise a non-zero exponent from the first phase; no USD PSP is required.
- **Write to:** DOC-01, DOC-14, DOC-27.

### DR-02 · How long is financial data retained?
- **Problem:** Original SDD NFR-16 and §22.2 propose 10 years "to confirm with an accountant or lawyer per market".
- **Decision (proposed):** 10 years for journals, postings, wallet movements, charges, invoices and the audit log, as a configuration value per store. The Owner confirms with an accountant before go-live. The complete table is DR-97.
- **Write to:** DOC-01, DOC-20.

### DR-03 · How is tax on an invoice represented? — ⚠ deviates from the original SDD
- **Problem:** Original SDD §2.3 has `invoice_line.tax_amount` with a rate from tenant configuration. §14 says an invoice "never computes or moves" money, §14.3 says paying a postpaid invoice credits `customer_wallet`, and §11.8 expects `balance` to be exactly 0 after payment. If tax were added on top of charges, the customer would pay more than was debited, the wallet would end positive, and the chart of accounts (§6.1) has no `tax_payable` account to hold the difference.
- **Options:** (1) tax-exclusive, with a new `tax_payable` account and journal template: correct, but it is the tax engine the SDD defers; (2) tax-inclusive and informational: prices already include tax, the invoice shows how much of each line is tax, no ledger effect.
- **Decision (proposed):** Option 2. `invoice.total = SUM(invoice_line.amount)`. `tax_amount = round_half_up(amount × rate / (1 + rate))`, computed when the line is written, with the rate from `platform.tenant_billing_profile.tax_rate` (`numeric(5,4)`, default `0`). No posting for tax.
- **Consequences:** A later tax-exclusive engine needs a new template and account; the current model does not block it.
- **Write to:** DOC-33, DOC-15, ADR-0022.

### DR-04 · Is the Billing invoice a legal tax document?
- **Problem:** Original SDD §2.3 defers e-invoicing, and §22.2 says that "if a market requires a registered e-invoice at launch, it moves into scope". The same section assumes Vietnam, where registered e-invoices appear to be mandatory for most business sales (Decree 123/2020/ND-CP; to confirm with an accountant). Leaving this open risks a go-live that is not legally usable.
- **Decision (proposed):** The Billing invoice is a commercial statement. The legal e-invoice is issued outside Billing, by the operator's accounting system, from the ERP batches of §15.4. The PDF carries the line "This document is not a tax invoice." The Owner confirms this with an accountant before go-live. If it must live inside Billing, an `EInvoiceIssuer` port consuming `invoice.issued` is added as new scope.
- **Consequences:** Blocks go-live, not any build phase.
- **Write to:** DOC-01, DOC-33, DOC-36.

### DR-05 · What does "v1.0" mean, and what gates real money?
- **Problem:** The opening of the original SDD delivers the system "in development and staging". §19.1 describes production and S7 includes "staging and production environments". §22.2 proposes going live only after EXP-18. Nothing states what the release tag promises.
- **Decision (proposed):** `v1.0.0` means: stages S0–S8 complete; mini and staging running; `values-prod.yaml` proven by the load and failover experiments; every experiment result recorded. Taking real money is a separate Owner decision that requires all of: EXP-18 passing on staging; DR-01, DR-02, DR-04, DR-70 and DR-91 decided; a verified restore drill (DR-93); a live PSP smoke test.
- **Write to:** DOC-01, DOC-75, DOC-82.

---

## B. Platform and code architecture

### DR-06 · Which exact versions are pinned? — 🔬 spike (S-01)
- **Problem:** Original SDD §4.3 names lines (Spring Boot 4.1.2, Modulith 2.1, Debezium 3.7, PostgreSQL 18, Java 25) and §22.2 leaves pairing to be confirmed. It names no version for Strimzi, CloudNativePG, ClickHouse, KEDA, Keycloak or the libraries around Boot 4. It also says JUnit 5, while a Boot 4 line may manage a newer JUnit.
- **Decision (proposed):**
  - Fixed: Java 25 (Temurin), Spring Boot 4.1.2, Spring Modulith 2.1.x, PostgreSQL 18.x, Debezium 3.7.x, Apache Kafka 4.x at the newest minor the chosen Strimzi supports, PgBouncer ≥ 1.21.
  - Every version lives in `gradle/libs.versions.toml`, Helm values or `portal/package.json`. Renovate proposes updates weekly. A new major version needs a DR.
  - S-01 builds one thin vertical slice with the real stack and records the verified set in DOC-11. It must answer: JUnit line and jqwik compatibility; Error Prone and NullAway on JDK 25; jMolecules ArchUnit rule APIs; Jackson 3 (`tools.jackson`) with OpenAPI generators; Spring Cloud Gateway Server WebMVC for Boot 4.1; ShedLock, Resilience4j and Bucket4j on Boot 4.
- **Consequences:** A library that fails the slice gets a named fallback in DOC-11 or a new DR.
- **Write to:** DOC-11.

### DR-07 · If the application layer has no Spring, who creates its beans?
- **Problem:** Original SDD §5.1 forbids Spring in `application` and shows `@CommandHandler`, a jMolecules marker that Spring does not scan. No layer in the table may hold a `@Configuration`.
- **Decision (proposed):** Add one package per module, `<module>.adapter.config`, which may depend on every layer of its own module and on Spring. It holds `<Module>Configuration` with one explicit `@Bean` per handler and port. There is no component scan over `domain` or `application`.
  ```java
  @Configuration
  class LedgerConfiguration {
      @Bean ReserveBudgetHandler reserveBudget(PricingApi pricing, WalletGate gate, UnitOfWork uow,
                                               IdempotencyGuard guard, WalletRepository wallets) {
          return new ReserveBudgetHandler(pricing, gate, uow, guard, wallets);
      }
  }
  ```
  Role activation sits on `adapter.in` beans only: `@ActiveInRoles({"api"})`, a meta-annotation over a `Condition` reading `billing.role`. `billing.role=all` is allowed only in the fast development loop, and a build test keeps it out of every chart values file.
- **Consequences:** More wiring code, visible and compile-checked. New ArchUnit rule: nothing outside `adapter.config` depends on it.
- **Write to:** DOC-12, ADR-0003.

### DR-08 · What is the exact module map, and how do modules share event types? — ⚠ deviates from the original SDD
- **Problem:** Original SDD §5.1 puts integration event records in each module's `api`. Ledger consumes `payment.succeeded` (§8.5) while Payment calls Ledger for refund holds (§13.3), so `ledger → payment.api` and `payment → ledger.api` form a cycle, which `ApplicationModules.verify()` must reject (§5.5). §5.1 also lists `UnitOfWork` and `Outbox` as ports without saying who owns the shared ones.
- **Decision (proposed):** Event records are generated from `api/events` into `eventing.api.events`, and any module may depend on `eventing.api`. A module's own `api` holds only commands, results and use-case interfaces.

  | Module | Schema | Store | May call (`api` of) |
  | --- | --- | --- | --- |
  | `platform` | `platform` | Billing Ops DB | none |
  | `eventing` | none | none | `platform` |
  | `pricing` | `pricing` | Billing Ops DB | `platform` |
  | `ledger` | `ledger` | Ledger DB | `platform`, `eventing`, `pricing` |
  | `metering` | `metering` | Billing Ops DB, ClickHouse | `platform`, `eventing`, `pricing`, `ledger`, `invoicing` |
  | `invoicing` | `invoicing` | Billing Ops DB | `platform`, `eventing`, `pricing`, `ledger` |
  | `payment` | `payment` | Billing Ops DB | `platform`, `eventing`, `ledger` |
  | `webhook` | `webhook` | Delivery DB | `platform`, `eventing` |
  | `reconciliation` | `reconciliation` | Billing Ops DB | `platform`, `eventing`, `payment`, `ledger` |
  | `audit` | `audit` | Billing Ops DB | `platform`, `eventing` |
  | `scheduling` | none (ShedLock table in `platform`) | Billing Ops DB | job interfaces of every module above |

  `UnitOfWork`, `TenantContext`, `ShardRouter`, `CrossTenantScan`, `TenantLocator` and `PlatformDirectory` are in `platform.api`. `Outbox` is in `eventing.api`. Each `api` package is declared `@NamedInterface("api")`, and each module lists `allowedDependencies` in its `@ApplicationModule`.
- **Consequences:** The allowed-dependency table above is enforced by the build.
- **Write to:** DOC-12, ADR-0003.

### DR-09 · Which aggregates use Spring Data JDBC, and which use explicit SQL?
- **Problem:** Original SDD §5.4 chooses row records and mappers with Spring Data JDBC, with `@Query` or `JdbcClient` for locking. It does not say where the line falls. Composite keys on partitioned tables (DR-13) fit Spring Data JDBC poorly.
- **Decision (proposed):**

  | Access | Used for |
  | --- | --- |
  | `JdbcClient`, explicit SQL | `Wallet`, `Reservation`, `Journal`, `Posting`, `Charge`, `WalletMovement`, `IdempotencyKey`, `Outbox`, `Delivery` and every row claimed with `SKIP LOCKED` |
  | Spring Data JDBC with row records and mappers | `Tenant`, `ClientSystem`, `ClientGrant`, `Product`, `Meter`, `PricePlan` and its versions, `Subscription`, `Invoice`, `Payment`, `Refund`, `WebhookSubscription` |

  Rule: if a statement's lock, condition or row count decides correctness, its SQL is written out.
- **Write to:** DOC-12, ADR-0004.

### DR-10 · How does the OpenAPI pipeline generate code? — 🔬 spike (S-01)
- **Problem:** Original SDD §17 says the REST contract is written first, with server interfaces and the portal client generated from it, but names no generator. §5.5 expects Jackson 3.
- **Decision (proposed):**

  | Output | Tool | Location |
  | --- | --- | --- |
  | Contract | OpenAPI 3.1, split by tag, bundled by Redocly CLI | `api/openapi/` |
  | Server interfaces and DTOs | `openapi-generator` `spring`, `interfaceOnly=true` | `billing/build/generated`, consumed by `adapter.in.rest` |
  | Portal client | `openapi-typescript` + `openapi-fetch` | `portal/src/api/generated` |
  | Demo Client | `openapi-generator` `java`, `restclient` library | `demo-client/build/generated` |
  | Compatibility | `oasdiff breaking` against the last release tag | CI |

  JSON is `snake_case`, as in original SDD §17.2. Fallback if the generator cannot produce Jackson 3 code at S-01: hand-written controllers, with a contract test validating every response against the OpenAPI file.
- **Write to:** DOC-11, DOC-52, DOC-76.

---

## C. Data stores, shards and connections

### DR-11 · Are the logical stores schemas in one database, or separate databases? — ⚠ deviates from the original SDD
- **Problem:** Original SDD §16.1 defines three logical stores "in one PostgreSQL cluster", §19.8 has "2 databases in one instance" for shards, and §8.3 says "one PostgreSQL connector per cluster". A Debezium PostgreSQL connector and its replication slot read a single database. If the stores or shards are separate databases, one connector per cluster cannot work. If they share a database, a store cannot move with its own slot, and a schema-level split is not exercised before it is needed.
- **Decision (proposed):** Each store is a separate database, with **one connector, one slot and one publication per database**.

  | Database | Holds | Production | Mini, staging |
  | --- | --- | --- | --- |
  | `billing_ledger_00` | `ledger` | yes | yes |
  | `billing_ledger_01` | `ledger` | no | yes |
  | `billing_ops` | `platform`, `pricing`, `metering`, `invoicing`, `payment`, `reconciliation`, `audit` | yes | yes |
  | `billing_delivery` | `webhook` | yes | yes |
  | `demo_client` | Demo Client only | no | yes |
  | `keycloak` | Identity Provider | per DR-64 | yes |

- **Consequences:** Production runs 3 connectors, and mini and staging run 4. No cross-store SQL can compile against one connection, which enforces §4.2 from the first day. WAL alerts are per slot.
- **Write to:** DOC-07, DOC-24, DOC-73, ADR-0009, ADR-0016.

### DR-12 · What is the shard hash, and what do public IDs look like?
- **Problem:** Original SDD §16.3 and §19.6 write `hash(account_id) % 1024` without naming the hash. The result is embedded in public IDs forever, so it can never change.
- **Decision (proposed):**
  - `logical_shard = (unsigned big-endian int of the first 4 bytes of SHA-256 over the 16 raw bytes of account_id) % 1024`. SHA-256 is available in every language and independent of UUIDv7's time bits. Test vectors live in `api/events/vectors/shard.json`.
  - Public ID: `<prefix>_<3 lowercase hex digits of the shard>_<32 lowercase hex digits of the UUID>`.

    | Object | Prefix | Example |
    | --- | --- | --- |
    | Reservation | `res` | `res_1a3_01973a2d11117000800000000000c3d4` |
    | Charge | `chg` | `chg_1a3_01973a3122227000800000000000d1e2` |
    | Journal (in history responses) | `jrn` | `jrn_1a3_…` |

  - Accounts, tenants, payments, refunds, invoices, subscriptions, plans and event IDs stay plain UUIDs: the shard is derived from `account_id`, and the other objects live outside Ledger DB.
  - A malformed public ID returns `404 not_found`, the same as a missing one.
- **Write to:** DOC-14, DOC-25, DOC-52, ADR-0015.

### DR-13 · How are `journal`, `reservation` and `charge` partitioned? — ⚠ deviates from the original SDD · 🔬 spike (S-02)
- **Problem:** Original SDD §16.1 partitions `journal` by `HASH (account_id)` and leaves `reservation` and `charge` unpartitioned, although each receives about 8.6 million rows per day (§2.1). Public IDs carry the shard and not the account (§16.3), so `GET /v1/charges/{id}` and the first read of a commit (§6.5) would probe all 64 hash partitions. PostgreSQL also requires a primary key to contain the partition key, so `id` alone cannot be the key. A hash on `account_id` cannot serve shard moves either, because publication row filters allow only simple column expressions (DR-15).
- **Decision (proposed):** Every Ledger table carries `logical_shard smallint`. Tables needing a non-time unique key are range-partitioned on it, into 64 partitions of 16 logical shards; lookups by public ID prune to one partition.

  | Table | Partition | Primary key | Other unique keys |
  | --- | --- | --- | --- |
  | `journal` | `RANGE (logical_shard)`, 64 | `(logical_shard, id)` | `(logical_shard, account_id, source_type, source_id)` |
  | `reservation` | `RANGE (logical_shard)`, 64 | `(logical_shard, id)` | `(logical_shard, account_id, client_id, client_reference)` |
  | `charge` | `RANGE (posted_at)`, monthly | `(posted_at, id)` | `(posted_at, journal_id)` |
  | `posting` | `RANGE (posted_at)`, monthly | `(posted_at, journal_id, line_no)` | none |
  | `wallet_movement` | `RANGE (occurred_at)`, monthly | `(occurred_at, wallet_id, wallet_version)` | none |
  | `wallet`, `billing_account`, `ledger_account` | none | `id` | |

  ```sql
  CREATE TABLE ledger.journal (
      logical_shard smallint    NOT NULL CHECK (logical_shard BETWEEN 0 AND 1023),
      id            uuid        NOT NULL,
      account_id    uuid        NOT NULL,
      tenant_id     uuid        NOT NULL,
      source_type   text        NOT NULL,
      source_id     text        NOT NULL,
      currency      char(3)     NOT NULL,
      posted_at     timestamptz NOT NULL,
      PRIMARY KEY (logical_shard, id),
      UNIQUE (logical_shard, account_id, source_type, source_id)
  ) PARTITION BY RANGE (logical_shard);
  CREATE TABLE ledger.journal_p00 PARTITION OF ledger.journal FOR VALUES FROM (0) TO (16);
  -- ... journal_p63 FROM (1008) TO (1024)
  ```

  Charge lookup by ID uses the time inside the UUIDv7: `WHERE id = :id AND posted_at BETWEEN uuid_extract_timestamp(:id) - interval '1 day' AND uuid_extract_timestamp(:id) + interval '1 day'`.
- **Consequences:** `client_reference` is unique per account rather than per Client, which is harmless because the same reference on two accounts cannot double charge either. Monthly tables can tier to cheap storage (DR-97). S-02 runs this DDL with RLS through PgBouncer on PostgreSQL 18 and measures a hold transaction.
- **Write to:** DOC-14, DOC-25, ADR-0015.

### DR-14 · Where do the shard map and account directory live, and what does `UnitOfWork` look like?
- **Problem:** Original SDD §19.6 mentions "a small configuration table", and §16.1 lists `account_directory` with no columns. Everything needs them on the hot path, including when Billing Ops DB has moved to its own cluster.
- **Decision (proposed):**
  ```sql
  CREATE TABLE platform.shard_map (
      from_shard   smallint NOT NULL,   -- inclusive
      to_shard     smallint NOT NULL,   -- exclusive
      database_key text     NOT NULL,   -- "ledger_00", resolved to a DataSource by configuration
      PRIMARY KEY (from_shard)
  );
  CREATE TABLE platform.account_directory (
      account_id    uuid PRIMARY KEY,
      tenant_id     uuid     NOT NULL,
      logical_shard smallint NOT NULL,
      currency      char(3)  NOT NULL,
      mode          text     NOT NULL CHECK (mode IN ('PREPAID','POSTPAID')),
      status        text     NOT NULL CHECK (status IN ('PROVISIONING','ACTIVE')),
      updated_at    timestamptz NOT NULL DEFAULT now()
  );
  ```
  The map loads at startup and refreshes every 10 s, and each replica keeps its last good copy if the database is unreachable. `account_id → (tenant_id, logical_shard)` is immutable and cached for 1 hour. Mode is cached for 30 s.
  ```java
  public interface UnitOfWork { <T> T in(TxScope scope, Supplier<T> work); }
  public record TxScope(Store store, @Nullable AccountId account, TenantId tenant, ReadMode mode) {}
  public enum ReadMode { PRIMARY, REPLICA }
  ```
- **Write to:** DOC-15, DOC-25.

### DR-15 · How exactly is a logical shard moved? — 🔬 spike (S-08)
- **Problem:** Original SDD §19.6 says "logical replication of that shard's accounts", then "the router blocks writes". PostgreSQL row filters cannot call `hash()`, routers refresh at different moments, and replicated outbox rows would be published twice by the target's connector.
- **Decision (proposed):**
  1. Create the publication with row filters on the column from DR-13: `CREATE PUBLICATION shard_move_163 FOR TABLE ledger.wallet WHERE (logical_shard = 163), ledger.journal WHERE (logical_shard = 163), …`. `outbox` is excluded.
  2. Subscribe from the target. Wait until lag is under 1 s.
  3. Fence at the arbiter: `UPDATE ledger.wallet SET moved_at = now() WHERE logical_shard = 163`. The wallet lock statement gains `AND moved_at IS NULL`. Zero rows returned raises `ShardMoved`, and the request refreshes the map and retries once on the new database. A stale router therefore cannot write to the source.
  4. Wait until the source connector has published the source outbox.
  5. Wait until replication has applied everything, then drop the subscription.
  6. Clear `moved_at` on the target copies, then flip `platform.shard_map`.
  7. After 7 days, `billing_maint` deletes the source rows.

  Target: the write block between steps 3 and 6 is ≤ 30 s (NFR-18). S-08 rehearses this on k3d between `billing_ledger_00` and `billing_ledger_01`, and EXP-16 proves it under load at P8.
- **Consequences:** A `moved_at` column on `wallet`, unused until the first move. System ledger accounts already share IDs across shards (DR-20).
- **Write to:** DOC-25, DOC-78, ADR-0015.

### DR-16 · How do migrations run, and who maintains partitions?
- **Problem:** Original SDD §19.1 says "Flyway as a pre-deploy job", with one location per module (§5.5). The Ledger location must run on every shard database, and with about 10 partitioned table families someone must create future partitions and drop expired ones. §16 names no mechanism.
- **Decision (proposed):**
  - A Helm `pre-install,pre-upgrade` Job runs the `billing` image with `billing.role=migrate`, connecting directly to `-rw` as `billing_owner`. It applies `platform` first, then the other modules in dependency order, and the `ledger` location on every `billing_ledger_*`. History tables are per schema. File names: `V<yyyyMMddHHmm>__<description>.sql`.
  - A `PartitionMaintenance` job runs daily (DR-85), written in Java over a small table of families. It creates partitions 3 months ahead, drops partitions past retention (DR-97), and runs as `billing_maint`.
  - pg_partman is not used: it would add an extension to the CloudNativePG image for about 10 table families.
  - Alert: the next month's partition is missing with fewer than 14 days to go.
- **Write to:** DOC-19, DOC-20, DOC-37, DOC-73.

### DR-17 · What is the complete database role matrix? — ⚠ deviates from the original SDD
- **Problem:** Original SDD §12.3 defines `billing_app` and `billing_system`, and §19.7 implies a read-only path. It leaves out who owns tables, who drops partitions, what Debezium connects as, how the conformance runner reads, and how ID-addressed requests find their tenant (DR-62).
- **Decision (proposed):**

  | Role | Attributes | Used by | Rights |
  | --- | --- | --- | --- |
  | `billing_owner` | owns objects | Migration job, direct | All DDL |
  | `billing_app` | subject to RLS; `idle_in_transaction_session_timeout = 5s` | `api`, `worker`, `rating`, `webhook-*`, `ingest` | DML per table; no `UPDATE`, `DELETE` on append-only tables (original SDD §16.2) |
  | `billing_system` | `BYPASSRLS` | `CrossTenantScan` only | `SELECT (tenant_id, id, …)` by column; `UPDATE (expiry_enqueued_at)` on `reservation` |
  | `billing_locator` | `BYPASSRLS` | `TenantLocator` only (DR-62) | `SELECT` on locator columns only |
  | `billing_maint` | member of owner | `PartitionMaintenance`, retention deletes | Partition DDL; `DELETE` on `idempotency_key` and moved rows |
  | `billing_readonly` | `BYPASSRLS`, read-only | Invariant checks, conformance runner, `reconcile` reads | `SELECT` on all |
  | `billing_debezium` | `REPLICATION` | Debezium, direct | `SELECT` on `outbox`; heartbeat table |
  | `demo_client` | none | Demo Client | Its own database only |

  ClickHouse users: `usage_writer` (insert), `usage_rating` (select, used by `rating` and `scheduler`), `usage_api` (select on rollups). Kafka users: one per process role, with ACLs from DR-40.
- **Write to:** DOC-19, DOC-41.

### DR-18 · How do the per-role connection pools map to PgBouncer? — ⚠ deviates from the original SDD · 🔬 spike (S-04)
- **Problem:** Original SDD §19.7 gives one server-connection number per role and "PgBouncer × 3". A PgBouncer pool is per (database, user) and per pod, so with several databases and 3 pods the numbers multiply. Several pools share `billing_app`, so one Pooler cannot separate them. The `reconcile` pool is read-only on `-ro`, but Reconciliation must write `reconciliation.run` and `exception`, and digests update state. `ingest` has no pool, but needs reference data (DR-49).
- **Decision (proposed):** One CloudNativePG `Pooler` per pool. `default_pool_size = total / pods`. Totals are per database:

  | Pooler | Service | Role | Pods (prod) | `billing_ledger_*` | `billing_ops` | `billing_delivery` |
  | --- | --- | --- | --- | --- | --- | --- |
  | `pool-api` | `-rw` | `billing_app` | 3 | 60 + 10 reserve | 20 | 5 |
  | `pool-worker` | `-rw` | `billing_app` | 2 | 25 | 15 | none |
  | `pool-rating` | `-rw` | `billing_app` | 2 | 10 | 5 | none |
  | `pool-webhook` | `-rw` | `billing_app` | 2 | none | none | 30 |
  | `pool-ingest` | `-rw` | `billing_app` | 1 | none | 3 | none |
  | `pool-system` | `-rw` | `billing_system` | 2 | 10 | 5 | 3 |
  | `pool-locator` | `-rw` | `billing_locator` | 2 | 5 | 5 | none |
  | `pool-reconcile-rw` | `-rw` | `billing_app` | 1 | none | 3 | none |
  | `pool-reconcile-ro` | `-ro` | `billing_readonly` | 1 | 5 | 5 | none |

  `max_connections = 400` on production. `query_wait_timeout = 2s` on `pool-api`. S-04 proves that one Pooler per pool works with several databases, and measures its memory.
- **Consequences:** About 16 PgBouncer pods in production instead of 3, each light. Mini runs one pod per Pooler.
- **Write to:** DOC-19, DOC-73, DOC-75, ADR-0017.

---

## D. Wallet and Ledger

### DR-19 · Which journal templates are missing? — ⚠ deviates from the original SDD
- **Problem:** Original SDD §6.1 says "a template is code, so an operator cannot invent an unbalanced entry", and lists one manual adjustment direction, crediting the customer. But §7.3 sends a commit after the late window to "a Finance Admin … as an adjustment", which must debit the customer. §15.1 says a configured rule "posts" a known PSP fee, while the chart has no fee account and §6.3 says a journal "touches one customer wallet by construction".
- **Decision (proposed):** Add to the table in original SDD §6.1:

  | Event | Debit | Credit | Source key | Movement kind |
  | --- | --- | --- | --- | --- |
  | Manual adjustment, debit | `customer_wallet` | `adjustment` | `adjustment:<adjustment_id>` | `CHARGE` |
  | Rating credit (DR-45) | `revenue` | `customer_wallet` | `rating:<cursor_id>:<seq>` | `CREDIT` |
  | PSP fee | `psp_fee` | `psp_clearing` | `psp-fee:<psp>:<report_line_id>` | none |

  `POST /v1/admin/adjustments` takes `direction: CREDIT | DEBIT`. New ledger account `psp_fee` (expense, per PSP and currency). A PSP fee journal has `account_id` = the reserved system account `00000000-0000-7000-8000-000000000000`, `logical_shard = 0`, takes no wallet lock and writes no movement. It is the only journal without a customer wallet.
- **Consequences:** The "wallet balance equals postings" check filters on `customer_wallet`, so it is unaffected. The "total debits equal credits" check still covers fee journals.
- **Write to:** DOC-14, DOC-21.

### DR-20 · How are ledger accounts identified across shards?
- **Problem:** `revenue` is per tenant_system and currency, and `psp_clearing` is per PSP and currency (original SDD §6.1). With several physical databases these rows must exist on each. If each generated its own UUID, a shard move or a cross-shard report would collide or mismatch. §16 does not say when the rows are created.
- **Decision (proposed):** `ledger_account.id = UUIDv5(fixed namespace, "<type>:<scope>:<currency>")`, for example `revenue:<client_id>:VND`, `psp_clearing:vnpay:VND`, `adjustment::VND`, `customer_wallet:<account_id>:VND`. The same ID results on every database. Rows are inserted lazily with `ON CONFLICT DO NOTHING` in the posting transaction, guarded by an in-process set of known IDs, so the insert runs only on a cache miss.
- **Write to:** DOC-14, DOC-21, DOC-25.

### DR-21 · Can `available` at a past time really be reconstructed? — ⚠ deviates from the original SDD
- **Problem:** FR-31 and original SDD §6.9 say the movement log answers "what was available at time T". `available = balance − held + credit_limit`, but `wallet_movement` stores only `balance_after` and `held_after`, and `credit_limit` changes by admin action (§17.1). The check in §19.4 also compares `w.version` with the latest row, so any version bump without a movement raises a false critical alert, and §6.8 does not cover TTL-only extends or limit changes.
- **Decision (proposed):**
  - Add `credit_limit_after bigint NOT NULL` to `wallet_movement`.
  - Add movement kind `LIMIT_CHANGE`, with both deltas 0, written when `credit_limit` changes.
  - `wallet.version` increments exactly when a movement row is written.
  - A TTL-only extend writes no movement and no version bump; it changes only `reservation`.
  - A rejected hold updates only `insufficient_notified_at`, with no version bump.
- **Write to:** DOC-14, DOC-21, ADR-0007.

### DR-22 · What are the default standing thresholds, and how is one event type told apart?
- **Problem:** FR-12 makes `low_threshold` configurable with no default. Original SDD §6.7 uses `balance.insufficient` both for entering `INSUFFICIENT` and for a rejected oversized request.
- **Decision (proposed):**
  - `low_threshold` defaults to 0 (never `LOW`) and is set per account.
  - Evaluation order: `available <= 0` → `INSUFFICIENT`; `available < low_threshold` → `LOW`; otherwise `NORMAL`.
  - `balance.insufficient.data.trigger` is `STANDING_CHANGED` or `HOLD_REJECTED`; `requested_amount` is present only for the second.
  - `balance.insufficient_notify_interval` defaults to 5 minutes.
- **Write to:** DOC-21, DOC-54.

### DR-23 · How is the Wallet Gate implemented, and how does it survive autoscaling? — ⚠ deviates from the original SDD
- **Problem:** Original SDD §6.5 sets 5 permits per `api` replica so that 3 replicas hold at most 15 connections (NFR-17), and says permits "come down" as replicas are added. But §19.1 autoscales `api` with an HPA, so at 6 replicas one wallet could hold 30 connections, half the pool. One semaphore per wallet held forever also grows without bound over 10,000 accounts.
- **Decision (proposed):**
  - `wallet_gate.total_permits = 15` per environment. The chart renders `wallet_gate.permits_per_replica = max(1, floor(total_permits / hpa.maxReplicas))`.
  - Production: `api` HPA min 3, max 5, so 3 permits per replica. The SDD's 5 per replica holds only with no autoscaling.
  - A build test over rendered values fails if `permits_per_replica × hpa.maxReplicas > 0.25 × pool-api size`.
  - Implementation: `ConcurrentHashMap<WalletId, Slot>`, where `Slot` holds an `int inUse`, updated inside `compute`; the entry is removed when `inUse` returns to 0; `tryAcquire` never waits.
  ```java
  public interface WalletGate {
      <T> T enter(WalletId wallet, Supplier<T> whenBusy, Supplier<T> work);
  }
  ```
- **Consequences:** Fewer permits per replica at low replica counts: 9 in-flight at 3 replicas, not 15. EXP-02 measures rejections at 50 per second and recalibrates.
- **Write to:** DOC-21, DOC-43, ADR-0005.

### DR-24 · How can the balance invariant keep running on a 10-year ledger? — ⚠ deviates from the original SDD
- **Problem:** The balance check in original SDD §19.4 joins `posting` with no time bound, and the daily "full" run does this for every wallet. At about 630 GB per year (§16.1), with old partitions on cheap storage (§15.2), this becomes impossible well before the 10-year retention ends.
- **Decision (proposed):**
  ```sql
  CREATE TABLE ledger.wallet_checkpoint (
      wallet_id uuid        NOT NULL,
      as_of     timestamptz NOT NULL,   -- first instant of a month, exclusive upper bound
      balance   bigint      NOT NULL,
      PRIMARY KEY (wallet_id, as_of)
  );
  ```
  - The check becomes `wallet.balance = checkpoint.balance + SUM(postings since as_of)`.
  - A monthly job writes each checkpoint from the previous checkpoint plus that month's postings, and alerts instead of writing on any mismatch.
  - "Recently changed" wallets come from `wallet_movement` since the last run.
  - Checks run as `billing_readonly` on `-ro`, and any row is rechecked on the primary before alerting.
  - A yearly job recomputes one full year from postings and compares it with the checkpoints, which is what catches an error frozen into a checkpoint.
  - Check SQL lives in `deploy/sql/invariants/*.sql`, shared with the runner (DR-102).
- **Write to:** DOC-14, DOC-18, DOC-35, DOC-37.

### DR-25 · What happens when mode or credit limit changes?
- **Problem:** Original SDD §17.1 allows `PATCH /v1/admin/accounts/{id}` on mode and limit. Only `cycle_length` and `payment_terms` are "effective from the next cycle". §6.4 defines both modes but no transition, and §14.2 decides the invoice state from the mode.
- **Decision (proposed):**
  - **Mode:** stored as `billing_account.pending_mode`, applied at the next cycle start. `billing_cycle.account_mode` snapshots the mode, so an invoice uses its cycle's mode. Postpaid to prepaid returns `409 account_has_open_invoices` while any invoice is `ISSUED`.
  - **Credit limit:** applied at once under the wallet lock, writing `LIMIT_CHANGE` (DR-21) and recomputing standing. Lowering it below current use is allowed: existing holds stay, and `available` may go negative.
- **Write to:** DOC-21, DOC-33, DOC-53.

### DR-26 · How is the "debits equal credits" trigger built on a partitioned table? — 🔬 spike (S-02)
- **Problem:** Original SDD §6.2 requires a deferred constraint trigger, while `posting` is partitioned monthly (§16.1). It is not stated whether PostgreSQL 18 accepts a constraint trigger on the parent, or what it costs, since it fires at `COMMIT` while the wallet lock is held.
- **Decision (proposed):** One check per journal, on the first line only:
  ```sql
  CREATE FUNCTION ledger.assert_journal_balanced() RETURNS trigger AS $$
  BEGIN
    IF (SELECT SUM(CASE direction WHEN 'D' THEN amount ELSE -amount END)
        FROM ledger.posting
        WHERE posted_at = NEW.posted_at AND journal_id = NEW.journal_id) <> 0 THEN
      RAISE EXCEPTION 'journal % is unbalanced', NEW.journal_id;
    END IF;
    RETURN NULL;
  END $$ LANGUAGE plpgsql;
  -- on the parent if accepted, else created on each partition by PartitionMaintenance
  CREATE CONSTRAINT TRIGGER journal_balanced AFTER INSERT ON ledger.posting_y2026m11
    DEFERRABLE INITIALLY DEFERRED FOR EACH ROW WHEN (NEW.line_no = 1)
    EXECUTE FUNCTION ledger.assert_journal_balanced();
  ```
  S-02 answers parent versus per-partition and measures the cost. A build test fails if any `posting` partition lacks the trigger.
- **Write to:** DOC-14, DOC-21.

---

## E. Balance Reservation

### DR-27 · What are the exact semantics of a reserve request? — ⚠ deviates from the original SDD
- **Problem:** Original SDD §7.1 allows "estimated usage, or an explicit `max_amount`". §7.2 adds a buffer "when the Client gives no `max_amount`". It does not say which wins when both are sent, how a plan is pinned with `max_amount` alone, or what happens when the estimate is 0, while §16.2 has `CHECK (held_amount > 0)` and a free plan must still work.
- **Decision (proposed):**

  | Field | Rule |
  | --- | --- |
  | `account_id`, `client_reference`, `product` | Required |
  | `estimated_usage`, `max_amount` | At least one |
  | `ttl_seconds` | Optional, default `default_ttl`, range 60–86,400; outside the range is `400` |

  Hold amount: `max_amount` when given, with no buffer; otherwise `round_half_up(estimate × (1 + estimate_buffer_pct / 100))`. With `max_amount` alone, the plan is still resolved and pinned (DR-47) but no rating runs. An estimate of 0 creates an `ACTIVE` reservation with `held_amount = 0`, so `CHECK (held_amount >= 0)` replaces `> 0`. No plan, or a plan in another currency: `422 no_applicable_price`.
- **Write to:** DOC-22, DOC-53.

### DR-28 · What does each operation do in each edge state?
- **Problem:** Original SDD §7.2 and §7.4 define the main transitions but not extend past `expires_at`, release after TTL, or a commit when the job is days behind. The body of `extend` is undefined ("optional increase").
- **Decision (proposed):** `POST …/extend` body: `{ "ttl_seconds": 3600, "additional_amount": 20000 }`; both optional, at least one present.

  | State at `db_now` | Commit | Release | Extend |
  | --- | --- | --- | --- |
  | `ACTIVE`, before `expires_at` | `COMMITTED` | `RELEASED` | Applies |
  | `ACTIVE`, past `expires_at`, within window | `LATE_COMMITTED` | `RELEASED` | `409 reservation_not_active` |
  | `ACTIVE`, past window (job far behind) | `409 reservation_closed` | `RELEASED` | `409 reservation_not_active` |
  | `EXPIRED`, within window | `LATE_COMMITTED` | `409 reservation_not_active` | `409 reservation_not_active` |
  | `EXPIRED`, past window | `409 reservation_closed` | `409 reservation_not_active` | `409 reservation_not_active` |
  | `COMMITTED`, `LATE_COMMITTED`, `RELEASED` | `409 reservation_not_active` | same | same |

  The window is inclusive: `db_now <= expires_at + late_commit_window`. An extend past `created_at + max_total_lifetime` returns `422 ttl_exceeds_lifetime`. A rejected increase returns `402` and leaves the TTL unchanged.
- **Write to:** DOC-22, DOC-46, DOC-53.

### DR-29 · How do plan parameters apply on a reservation versus on a stream?
- **Problem:** Original SDD §9.2 defines `minimum_amount` and `maximum_amount` "per cycle" and `included_quantity` per component, while §9.3 says a workload "is priced on its own quantities". A hybrid plan has `FLAT` plus metered components, and nothing says what a reservation does with the flat part.
- **Decision (proposed):**

  | Parameter | On reserve and commit | On streaming usage |
  | --- | --- | --- |
  | `FLAT` component | Ignored; the subscription job charges it | Ignored |
  | `included_quantity` | Subtracted once per reservation | Once per cycle, per cursor |
  | Tiers | Applied to the reservation's own quantity | Applied to the cycle's cumulative quantity |
  | `maximum_amount` | Per reservation, per component | Clamps cumulative amount |
  | `minimum_amount` | Per reservation, per component | Applied only by the final rating of the cycle (DR-52) |
  | `billing_increment` | Rounds up each component's quantity | Rounds up the cumulative quantity |

  A commit usage line whose meter is not in the pinned plan returns `422 unknown_meter`. `charge.rating_snapshot` is JSON naming `plan_version_id`, and per component: `quantity`, `included`, `billed_quantity`, `tiers_applied` and `amount`.
- **Write to:** DOC-22, DOC-27, DOC-29.

### DR-30 · What does a refund hold look like inside `reservation`? — ⚠ deviates from the original SDD
- **Problem:** Original SDD §13.3 places "a `kind = 'REFUND'` hold, with no TTL" with source `refund-hold:<refund_id>`. But `reservation` requires `client_id`, `client_reference` and `plan_version_id` (§16.1, §16.2), its status machine (§7.2) has no refund path, and nothing makes placing the hold idempotent.
- **Decision (proposed):**
  - `client_id` = the reserved system UUID, and `client_reference = 'refund-hold:<refund_id>'`. The unique key from DR-13 makes placing the hold idempotent.
  - `plan_version_id` and `expires_at` are `NULL`, enforced by `CHECK ((kind = 'REFUND') OR (plan_version_id IS NOT NULL AND expires_at IS NOT NULL))`.
  - States: `ACTIVE → RELEASED` (rejected or failed); `ACTIVE → COMMITTED` (refund succeeded, writes movement `REFUND` and journal `refund:<refund_id>`).
  - No `reservation.*` event is published; `refund.*` events describe it. It is not reachable through `/v1/reservations`, and the portal shows it as "Refund pending".
  - Ledger API: `PlaceRefundHold`, `ReleaseRefundHold`, `SettleRefund`.
- **Write to:** DOC-14, DOC-22, DOC-32, ADR-0006.

### DR-31 · Where do the per-Client reservation parameters live?
- **Problem:** Original SDD §7.2 sets TTL, window, overage and buffer "per tenant_system", with no table, endpoint or statement of what a later change does to reservations already open.
- **Decision (proposed):** Columns on `platform.client_system`: `default_ttl_seconds`, `max_total_lifetime_seconds`, `late_commit_window_seconds`, `overage_policy`, `estimate_buffer_pct`, `rate_limits jsonb`. Read through `PlatformDirectory`, cached 30 s. Set on `POST /v1/admin/client-systems` and changed by `PATCH /v1/admin/client-systems/{id}`. A reservation snapshots `late_commit_window_seconds` and `overage_policy` at creation, so a change never alters one already open.
- **Write to:** DOC-15, DOC-22, DOC-43.

---

## F. Idempotency

### DR-32 · Where do idempotency keys live, and who is the caller? — ⚠ deviates from the original SDD
- **Problem:** FR-24 covers "every write command", but original SDD §16.1 has only `ledger.idempotency_key`, keyed `(client_id, key)`. Payments, refunds and subscriptions run in other stores, and one transaction touches one module (§4.2), so they cannot claim a key there. Customers and Finance Admins have no `client_id`, but `POST /v1/payments` and `POST /v1/refunds` require a key.
- **Decision (proposed):** One identical table in each schema with idempotent writes (`ledger`, `payment`, `invoicing`, `pricing`, `webhook`, `platform`, `reconciliation`), through a shared `IdempotencyGuard` in `platform.api`:
  ```sql
  CREATE TABLE <schema>.idempotency_key (
      caller          text        NOT NULL,  -- client:<client_id> | user:<tenant_id>:<sub> | admin:<sub>
      key             text        NOT NULL CHECK (length(key) BETWEEN 1 AND 255),
      request_hash    bytea       NOT NULL,
      response_status smallint,
      response_body   jsonb,
      created_at      timestamptz NOT NULL DEFAULT now(),
      PRIMARY KEY (caller, key)
  );
  ```
  There is no `tenant_id` column and no RLS: a row is read only under the primary key, which contains the caller. Each table has its own nightly cleanup at 7 days.
- **Write to:** DOC-14, DOC-15, DOC-16, DOC-23, ADR-0008.

### DR-33 · How is the request hash built, and which endpoints take a key? — ⚠ deviates from the original SDD
- **Problem:** FR-24 says every write command accepts a key. Original SDD §17.3 says required "on every `POST` that changes money". `PATCH` and `DELETE` on subscriptions post prorated charges and use neither word. `POST /v1/usage` does no database write in the request path (§10.1), so it cannot store a key at all. "Request hash" is undefined.
- **Decision (proposed):** `request_hash = SHA-256(method + "\n" + path + "\n" + raw body bytes)`. The path makes one key on two endpoints a `409 idempotency_key_reused`. Raw bytes mean a Client must resend identical bytes.

  | Endpoints | `Idempotency-Key` |
  | --- | --- |
  | Reservation `POST`s; `POST /v1/payments`; `POST /v1/refunds`; admin adjustments, charge reversal, refund approve and reject, invoice void; `POST`, `PATCH`, `DELETE` on subscriptions | Required; missing is `400 validation_failed` |
  | Other admin `POST`s, webhook subscription writes, redeliver, secret rotation | Optional |
  | `POST /v1/estimates`; `POST /v1/usage` and inbound usage (`event_id` deduplicates); PSP callback | Ignored |

  Stored and replayed: `2xx`, `402`, `409`, because they ran under the claim. Not stored: `400`, `404`, `422`, which run before the claim, and `5xx`, which rolled back. FR-24 in DOC-03 is rewritten to match this table.
- **Write to:** DOC-03, DOC-23, DOC-52.

### DR-34 · What does a 10-request commit storm really return? — ⚠ deviates from the original SDD
- **Problem:** Original SDD §11.8 and step 4 of the demo script expect "one charge; nine responses with `Idempotent-Replayed: true`". But §6.5 returns `503 account_busy` at once beyond the gate's permits, so on mini (1 replica) only some of the 10 concurrent requests are even admitted; the rest get `503`. §8.1 also has the in-flight duplicate wait "up to `lock_timeout`", while the sample in §6.3 sets `lock_timeout` only after the key claim.
- **Decision (proposed):**
  - The Demo Client resends every `503` with the same key (original SDD §11.7). The assertion becomes **final** outcomes: one `200` executed, nine `200` with `Idempotent-Replayed: true`, after retries; any number of intermediate `503`s, reported.
  - EXP-04 asserts effects, not first responses.
  - `SET LOCAL lock_timeout` is the first statement of every money transaction, issued by `UnitOfWork` before the key claim.
  - A `lock_timeout` while waiting on the key returns `503 account_busy`.
  - A replay of a finished key takes the same gate path; no read happens before the gate.
- **Consequences:** A replay under load can see `503` before it sees the stored response; every Client already retries `503`.
- **Write to:** DOC-23, DOC-40, DOC-81.

---

## G. Outbox, relay and event contract

### DR-35 · What is the exact outbox table? — ⚠ deviates from the original SDD
- **Problem:** Original SDD §8.2 inserts `id, tenant_id, account_id, event_type, api_version, occurred_at, payload`. But §17.3 propagates `traceparent` through Kafka headers and the outbox has no column for it. The key is `account_id` (§8.3), while `price_plan.published` and `webhook_endpoint.disabled` have no account. §8.4's envelope has no field that says which Client an event belongs to (DR-38). It is unclear whether `payload` is the whole envelope or only `data`.
- **Decision (proposed):**
  ```sql
  CREATE TABLE <schema>.outbox (
      id            uuid        NOT NULL,           -- envelope id, uuidv7()
      occurred_at   timestamptz NOT NULL,           -- db_now of the change
      destination   text        NOT NULL DEFAULT 'events' CHECK (destination IN ('events','audit')),
      partition_key text        NOT NULL,           -- account_id, else tenant_id, else client_id
      tenant_id     uuid,
      account_id    uuid,
      client_id     uuid,
      event_type    text        NOT NULL,
      api_version   text        NOT NULL,
      payload       jsonb       NOT NULL,           -- the whole envelope
      traceparent   text,
      PRIMARY KEY (occurred_at, id)
  ) PARTITION BY RANGE (occurred_at);
  ```
  Envelope fields: `id`, `type`, `api_version`, `occurred_at`, `tenant_id` (nullable), `account_id` (nullable), `client_id` (nullable), `data`.
- **Write to:** DOC-09, DOC-14, DOC-24.

### DR-36 · What is the Debezium connector configuration? — 🔬 spike (S-03)
- **Problem:** Original SDD §8.3 names the Outbox Event Router and `table.field.event.key`, and §22.2 leaves partitioned-outbox behavior to S2. Debezium is core to NFR-07 and shapes the outbox DDL, so waiting for S2 is too late. Heartbeats are not mentioned, and a quiet database such as `billing_delivery` on a calm day would retain WAL behind an idle slot.
- **Decision (proposed):** One connector per database (DR-11):
  ```json
  {
    "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
    "plugin.name": "pgoutput",
    "database.dbname": "billing_ledger_00",
    "slot.name": "dbz_billing_ledger_00",
    "slot.failover": "true",
    "publication.name": "dbz_outbox",
    "publication.autocreate.mode": "disabled",
    "table.include.list": "ledger.outbox",
    "snapshot.mode": "no_data",
    "tombstones.on.delete": "false",
    "heartbeat.interval.ms": "10000",
    "heartbeat.action.query": "UPDATE ledger.debezium_heartbeat SET ts = now()",
    "key.converter": "org.apache.kafka.connect.storage.StringConverter",
    "value.converter": "org.apache.kafka.connect.storage.StringConverter",
    "transforms": "outbox",
    "transforms.outbox.type": "io.debezium.transforms.outbox.EventRouter",
    "transforms.outbox.table.field.event.id": "id",
    "transforms.outbox.table.field.event.key": "partition_key",
    "transforms.outbox.table.field.event.payload": "payload",
    "transforms.outbox.route.by.field": "destination",
    "transforms.outbox.route.topic.replacement": "billing.${routedByValue}.v1",
    "transforms.outbox.table.fields.additional.placement": "event_type:header,traceparent:header,occurred_at:header"
  }
  ```
  Flyway creates the publication: `CREATE PUBLICATION dbz_outbox FOR TABLE ledger.outbox WITH (publish = 'insert', publish_via_partition_root = true);`. S-03 runs this on real versions with a partitioned outbox and confirms order, headers and exact property names.
- **Write to:** DOC-24, DOC-73, ADR-0009.

### DR-37 · What happens to the replication slot when the primary fails over? — ⚠ deviates from the original SDD · 🔬 spike (S-03)
- **Problem:** Original SDD §19.3 treats a lost slot as "replay from the 30-day outbox with a fresh snapshot", and the primary-loss row does not mention Debezium. A logical slot exists only on the primary unless it is synchronized to standbys. If it does not survive failover, then every failover (EXP-14) either loses the events committed since the last relay position or replays 30 days of them, breaking NFR-07 and NFR-08. `max_slot_wal_keep_size` of 200 GB (§19.3) can also invalidate a slot by design.
- **Decision (proposed):**
  1. **Normal path:** failover-enabled slots (PostgreSQL 17+): `slot.failover=true`, `sync_replication_slots = on` and `hot_standby_feedback = on` on standbys, `synchronized_standby_slots` on the primary, configured through CloudNativePG. S-03 kills the primary under load and shows the connector resuming on the new primary with no event lost.
  2. **Lost slot** (`RB-13`): stop the connector; create a new slot; compute `last_good` as the highest `occurred_at` header in Kafka for that database's events, minus 10 minutes; run a Debezium incremental snapshot of `outbox` with `occurred_at >= last_good`; resume. The 30-day blanket replay is only for when `last_good` cannot be determined.
  3. Replayed events are duplicates, and every consumer already dedupes (original SDD §8.5). An older duplicate arriving after a newer event is not an order inversion in EXP-07, because its `id` was already applied.
  4. Fallback if failover slots do not work with the operator: lost-slot procedure as an automated step of failover, accepted as RTO for events only, with a new DR.
- **Write to:** DOC-24, DOC-77, DOC-79, ADR-0009.

### DR-38 · Who may see which event? — ⚠ deviates from the original SDD
- **Problem:** Original SDD §12.1 says "neither tenant_system can read the other's reservations". But `billing.events.v1`, webhook subscriptions (§11.2) and the feed `GET /v1/accounts/{id}/events` (§11.6) all work per account, and a tenant shares one wallet across Clients. So a Client subscribed to `reservation.committed` or reading the feed would see another tenant's reservations and charges. §8.4's envelope requires `tenant_id` and `account_id`, which `price_plan.published` and `webhook_delivery.dead` do not have.
- **Decision (proposed):** Events carry `client_id` when they belong to one Client. This table is the contract, and a contract test walks it.

  | Group | `account_id` | `client_id` | Delivered to | In feed |
  | --- | --- | --- | --- | --- |
  | Reservation, Charge, Subscription, `usage.quarantined` | Yes | Owner | Owner only | Owner only |
  | Balance, Payment, Refund, Invoice, `billing_cycle.closed` | Yes | None | Every Client with a grant on the tenant | Same |
  | `price_plan.published`, `webhook_endpoint.disabled`, `webhook_delivery.dead` | None | Owner | Owner only | No |

  Matching adds: `event.client_id IS NULL AND grant exists` or `event.client_id = subscription.client_id`. Feed adds: `client_id IS NULL OR client_id = :caller`. Invoice event payloads carry totals only, never other tenants' lines. Key for non-account events: `client_id`.
- **Consequences:** Cursor offsets may skip values for one caller, which is still ordered and complete for that caller.
- **Write to:** DOC-09, DOC-26, DOC-30, DOC-31, DOC-54, ADR-0010.

### DR-39 · How is `api_version` versioned and rendered?
- **Problem:** Original SDD §17.3 pins `api_version` per subscription, and §8.2 stores one version in the row. Nothing says how an event written in the new version reaches a subscription pinned to the old one, or what counts as breaking.
- **Decision (proposed):** Date-formatted version, `2026-11-01` first. A subscription pins the current version when created, and changes it with `PATCH`. Events are stored in the current version. `webhook-delivery` applies a chain of `EventDowngrader`s to reach the pinned version, and an old version is supported for 12 months after its successor. The first release has one version and an identity chain. Schemas: `api/events/<api_version>/<event.type>.schema.json`, JSON Schema 2020-12. Adding an optional field is not breaking; anything else needs a new version. The feed always returns the current version.
- **Write to:** DOC-09, DOC-52, DOC-54.

### DR-40 · Which role runs which consumer, and what do the DLQs look like? — ⚠ deviates from the original SDD
- **Problem:** Original SDD §8.5 lists five consumers, §15.2 adds an Audit consumer, §11.6 adds a feed consumer, and §10.5 adds an analytics writer. The role table in §4.4 places only Ledger and Invoicing consumers. `reconcile` is "fixed at 1" with a read-only pool (§19.7) and cannot be a write consumer. DLQ names exist but not their settings. The topic table has no destination for the extra flows in DR-50, DR-53 and DR-83.
- **Decision (proposed):**

  | Consumer group | Role | Topic | Writes to |
  | --- | --- | --- | --- |
  | `ledger-events` | `worker` | `billing.events.v1` | Ledger DB |
  | `ledger-jobs` | `worker` | `billing.jobs.v1` (`reservation.expire`) | Ledger DB |
  | `invoicing-projection` | `worker` | `billing.events.v1` | `invoicing` |
  | `reconciliation-events` | `worker` | `billing.events.v1` | `reconciliation` |
  | `audit-writer` | `worker` | `billing.audit.v1` | `audit` |
  | `metering-exceptions` | `worker` | `usage.exceptions.v1` | `metering` |
  | `rating-jobs` | `rating` | `billing.jobs.v1` (`usage.rate`) | Ledger DB, `metering` |
  | `usage-writer` | `usage-writer` | `usage.raw.v1` | ClickHouse |
  | `metering-analytics` | `usage-writer` | `billing.events.v1` | ClickHouse |
  | `webhook-subscriber` | `webhook-subscriber` | `billing.events.v1` | `webhook` |
  | `ingest-inbound` | `ingest` | `client.inbound.v1` | Kafka |

  New topics: `billing.audit.v1` (key `tenant_id`, or `global`; 32 partitions; 30 days) and `usage.exceptions.v1` (key `client_id`; 12 partitions; 30 days). DLQs: `<topic>.<group>.dlq`, 6 partitions, 30 days, RF as the environment. Error handler: `DefaultErrorHandler` with `DeadLetterPublishingRecoverer`; non-transient errors make 5 attempts, transient errors (`SQLTransientException`, connection failures) back off 1–30 s forever. Topics are Strimzi `KafkaTopic` resources, with auto-create off. Offsets are committed manually after the transaction.
- **Write to:** DOC-09, DOC-37, DOC-44.

### DR-41 · What does a job message look like?
- **Problem:** Original SDD §7.5 and §10.3 publish jobs to `billing.jobs.v1` but never define the message. Two roles read the same topic.
- **Decision (proposed):**
  ```json
  { "job_id": "01973b00-…", "type": "reservation.expire", "scheduled_at": "2026-11-06T09:14:00Z",
    "tenant_id": "…", "account_id": "…", "subject_id": "res_1a3_…", "attempt_hint": 0 }
  ```
  Key `account_id`, header `job_type`. Types: `reservation.expire`, `usage.rate`. Each consumer group filters by `job_type` and commits the rest. Jobs are internal commands, are never exposed to Clients, and sit outside the event catalog, so the descriptive-name test does not apply. `tenant_id` binds the context per message.
- **Write to:** DOC-09, DOC-37.

### DR-42 · What must each event payload contain? — ⚠ deviates from the original SDD
- **Problem:** Original SDD §10.5 reads "the usage lines in `reservation.committed`", but the example in §8.4 has none. §8.4 says payloads are "complete snapshots" and shows only one event.
- **Decision (proposed):** Minimum `data` by group; DOC-54 gives one complete JSON per event type.

  | Group | Fields |
  | --- | --- |
  | Reservation | `reservation_id`, `client_reference`, `status`, `held_amount`, `currency`, `plan_version_id`, `expires_at`; settled also `charged_amount`, `released_amount`, `absorbed_amount`, `charge_id`, `usage_lines[]` (`line_no`, `meter`, `quantity`, `dimensions`) |
  | Balance | `balance`, `held`, `available`, `credit_limit`, `standing`, `previous_standing`, `currency`; `trigger`, `requested_amount` where relevant |
  | Charge | `charge_id`, `kind`, `amount`, `currency`, `source_type`, `source_id`, `service_from`, `service_to`, `plan_version_id`; reversal also `reverses_charge_id` |
  | Payment, Refund | `payment_id` or `refund_id`, `kind`, `amount`, `captured_amount`, `currency`, `status`, `invoice_id`, `reason` |
  | Invoice | `invoice_id`, `number`, `status`, `total`, `amount_paid`, `currency`, `period_from`, `period_to`, `due_at`; never a PDF link |
  | Subscription, cycle | `subscription_id`, `plan_code`, `status`, `cycle_id`, `cycle_start`, `cycle_end`, `fee_amount` |

  No names, emails, card or address data in any payload (original SDD §19.5).
- **Write to:** DOC-09, DOC-54.

---

## H. Pricing

### DR-43 · What is a meter called in the API? — ⚠ deviates from the original SDD
- **Problem:** Original SDD §10.1 sends `"meter": "gpu.a100.gpu_seconds"`, §17.2 sends `"meter": "gpu_seconds"` beside `"product": "gpu.a100"`, and §11.7 lists `llm.small.tokens` and also bare `vm_seconds`. A Client integrating from one section would fail against another.
- **Decision (proposed):** The API always uses the fully qualified name `<product>.<meter>`, in usage events, estimates, reservations and commits. Regex: `^[a-z0-9_]+(\.[a-z0-9_]+)+$`. The reservation keeps `product` because it selects the plan. `meter.code` is unique per `(client_id, code)`, and ingest resolves it to `meter_id` within the caller's namespace (FR-01). The §17.2 example becomes `"meter": "gpu.a100.gpu_seconds"`.
- **Write to:** DOC-15, DOC-27, DOC-53.

### DR-44 · What is the plan schema, and how are prices like "per GPU hour" stored exactly? — ⚠ deviates from the original SDD
- **Problem:** Original SDD §9.1 gives no currency on a plan, though an account has exactly one and a plan in the wrong currency must not rate. §11.8 prices "1,000,000 per GPU hour" on a `gpu_seconds` meter: 277.777… per second does not fit `numeric(30,12)` exactly, contradicting §9.2's "must be exact". Tier bounds at the edges are unspecified, and §14.1 uses a `fee_on_insufficient` field with no home.
- **Decision (proposed):**
  - `price_plan`: `client_id`, `code`, `name`, `currency`, `fee_on_insufficient` (`SKIP` default, `ALLOW`). Unique `(client_id, code)`.
  - `price_plan_version`: `plan_id`, `version`, `status`, `effective_from`.
  - `price_component`:
    ```json
    { "meter": "gpu.a100.gpu_seconds", "charge_model": "PER_UNIT",
      "dimension_filter": {"region": "hn-1"},
      "unit_price": "1000000", "unit_quantity": "3600",
      "tiers": null, "included_quantity": "0", "package_size": null,
      "minimum_amount": null, "maximum_amount": null,
      "billing_increment": "1", "rounding": "HALF_UP", "cycle": null }
    ```
  - `A(q) = unit_price × q / unit_quantity`, computed with exact `BigDecimal` multiplication and one division at the last step, then rounded to the currency's exponent. `unit_quantity` defaults to 1.
  - Tiers are `[{ "up_to": "50000", "unit_price": "10" }, { "up_to": null, "unit_price": "8" }]`. A tier covers `previous_up_to < q ≤ up_to`; the first starts at 0, and the last has `up_to: null`. `q = 0` rates to 0.
  - Publishing validates: bounds ascending, last bound open, exactly one model's fields present, currency known.
  - **Amended by DR-104 and DR-108:** `price_plan` also carries `pricing_mode` (immutable) and `managed_by`; `client_product` and `meter` carry `managed_by`. Publishing also checks the mode rules of DR-104 and that the Client holds that mode (DR-105).
- **Write to:** DOC-15, DOC-27.

### DR-45 · VOLUME pricing makes cumulative amount fall. What happens to the delta? — ⚠ deviates from the original SDD
- **Problem:** Three statements in the original SDD conflict. §9.2 property-tests that "`A` never decreases as `q` grows". But VOLUME is not monotonic: under the §11.8 hybrid plan, `A(50,000) = 500,000` and `A(50,001) = 400,008`. §10.3 says rating "never produces a negative delta" and routes any decrease to a Finance Admin. And §11.8 requires the hybrid cell to end at exactly 480,000, with usage arriving in batches over a minute and rating every 30 s, so rating can land on the high side of the tier boundary. Then the customer is overcharged by 20,000 until a person acts, and the fixture fails at random.
- **Options:** (1) Keep §10.3 and send to Finance: the cell cannot pass reliably, and every VOLUME plan needs manual reversals each cycle. (2) Rate VOLUME only at cycle close: correct, but a prepaid account's exposure grows to a whole cycle (§10.3 accepts one interval). (3) Allow a negative delta when the quantity did not fall: a tier drop is an expected, explainable result, not a metering bug.
- **Decision (proposed):** Option 3.
  - If `Δ < 0` and `Q_n ≥ Q_{n-1}`, it is a tier drop: post a **rating credit** (DR-19) with the same source key rule and a `charge` row of `kind = 'RATING_CREDIT'`, publishing `charge.reversed` with `reason = "tier_repricing"`.
  - If `Δ < 0` and `Q_n < Q_{n-1}`, it can only be a data defect: original SDD §10.3 stands (record `pending_correction` on the cursor, alert, Finance Admin decides).
  - The jqwik law "never decreases" applies to `PER_UNIT`, `GRADUATED`, `PACKAGE` and `FLAT` only. The law for VOLUME: the final amount depends only on the final `q`, not on the batches.
  - Condition 1 of §11.8 reads "the **net** charges of each component equal the expected amount".
- **Consequences:** VOLUME plans may show charges and credits within a cycle. The invoice shows the net per line group. The positive overshoot on a prepaid wallet is bounded by the plan's tier jump.
- **Write to:** DOC-27, DOC-29, DOC-33, DOC-40, ADR-0013.

### DR-46 · How do meter aggregations and dimensions reach the rating query? — ⚠ deviates from the original SDD
- **Problem:** Original SDD §9.1 declares `SUM`, `MAX` and `COUNT` meters, but the rating query in §10.2 always computes `sum(quantity)`. The query groups by `dimension_key`, while the cursor is per component (§10.3), and a component has a dimension filter. Freezing (§10.3) is described only for a sum.
- **Decision (proposed):**
  | Aggregation | Expression | Frozen rule | Event rule |
  | --- | --- | --- | --- |
  | `SUM` | `sum(quantity)` | `frozen + window` | `quantity > 0` |
  | `COUNT` | `count()` | `frozen + window` | `quantity` is 1, or the event is rejected as `invalid_quantity` |
  | `MAX` | `max(quantity)` | `GREATEST(frozen, window)` | `quantity ≥ 0` |

  `dimension_key` is the meter's declared dimensions, sorted by name: `region=hn-1|tier=a`; undeclared attributes are rejected. The query returns one row per `(meter_id, dimension_key)`, and Java assigns each row to the component whose `dimension_filter` matches, with `Q` per cursor as their combination. A row matching no component adds to the `billing_usage_unpriced_quantity` metric and raises an alert. It is not charged.
- **Write to:** DOC-17, DOC-27, DOC-29.

### DR-47 · How is a plan chosen for an account, and how does a draft get previewed? — ⚠ deviates from the original SDD
- **Problem:** Original SDD §9.1's `PriceAssignment` "falls back to the tenant_system's default plan". §14.1 attaches a subscription to a plan's `FLAT` components, but a hybrid plan's usage must rate on the same plan, and nothing links the two. §11.7 treats `storage.basic` and `storage.plus` as products, while `PATCH` changes "plan". §18.1's tier preview calls `POST /v1/estimates`, whose input (§9.3) cannot name a draft version.
- **Decision (proposed):**
  - Resolution for `(account, product)`: active subscription's plan → `price_assignment` → the tenant's `client_product.default_plan_id`. None: `422 no_applicable_price`.
  - Version: at reserve, the latest `PUBLISHED` with `effective_from <= now`; for streaming, the version effective at cycle start (original SDD §9.4).
  - `storage.basic` and `storage.plus` are **plans** of product `storage`. `POST /v1/subscriptions` takes `{ "account_id", "product", "plan_code" }`, and `PATCH` takes `{ "plan_code", "prorate": true }`.
  - `POST /v1/estimates` accepts optional `plan_version_id`, allowed only for a user holding `pricing.price_plan.read` on platform scope (DR-106), and may name a `DRAFT` version.
- **Write to:** DOC-27, DOC-33, DOC-39, DOC-53.

### DR-48 · When is a plan version retired?
- **Problem:** Original SDD §9.4 retires a version when "no open reservation or cycle references it". Checking across 64 partitions and every shard for open references is expensive for a cosmetic state.
- **Decision (proposed):** A daily job retires a version when a later version has been effective longer than `max(max_total_lifetime + late_commit_window)` across Clients (default 14 days), and no `OPEN` cycle in `invoicing` pins it. A commit pinned to a retired version still rates: retiring only hides it from new assignments.
- **Write to:** DOC-27, DOC-37.

---

## I. Usage Metering

### DR-49 · How can `ingest` validate with no database on the usage path? — ⚠ deviates from the original SDD
- **Problem:** Original SDD §10.1 validates "meter, dimensions, account grant, event age". §19.1 says `ingest` has "no database connection on the usage path", and the deployment figure connects it only to Kafka. NFR-11 asks more of ingest (99.95%) than of reservations (99.9%), so it must keep working through a PostgreSQL outage.
- **Decision (proposed):** `ingest` holds an in-memory snapshot of meters, dimensions, grants and the account directory. It loads at startup through `pool-ingest` and refreshes by `updated_at` every 60 s. The request path only reads memory. On refresh failure it keeps serving stale data for up to `ingest.snapshot_max_stale = 1h`, then returns `503`. Readiness requires a first load.
- **Consequences:** A new meter or grant takes up to 60 s to be accepted, and a revoked grant up to 60 s to be refused. A `pool-ingest` is added (DR-18).
- **Write to:** DOC-28, DOC-43.

### DR-50 · What are the exact usage contracts on each channel? — ⚠ deviates from the original SDD
- **Problem:** Original SDD §4.1 normalizes "one usage or client-event contract", but "client event" is defined nowhere. §10.1 returns per-event results, without a schema. On the Kafka channel (§11.1) there is no response at all, so a rejected event vanishes. Kafka ACLs cannot restrict message keys, so one Client holding write access to shared `client.inbound.v1` could send events claiming another `client_id`.
- **Decision (proposed):**
  - **Usage only**: this phase accepts no other client event type.
  - **REST and inbound webhook:** body `{ "events": [ … ] }`, 1–1,000 events. Response `202`: `{ "accepted": 998, "quarantined": 1, "rejected": [{ "index": 3, "event_id": "…", "code": "unknown_meter" }] }`. Codes: `unknown_meter`, `unknown_dimension`, `account_not_granted`, `occurred_at_in_future`, `invalid_quantity`, `invalid_reservation_id`.
  - **Kafka:** one event per message, key `client_id`, header `Billing-Signature` in the scheme of original SDD §11.4 with the tenant's inbound secret. The listener still uses mTLS; the signature is what authenticates the `client_id`. A wrong signature is dropped, counted per principal and alerted.
  - **Rejections on the Kafka channel:** `ingest` writes `{kind: "REJECTED"}` to `usage.exceptions.v1`; `worker` stores them in `metering.usage_rejection`, kept 30 days and read through `GET /v1/usage/rejections`.
  - `quantity` is a JSON string or number parsed to `BigDecimal`, scale ≤ 9. `reservation_id` is a public ID checked for format only: a Client marking its own usage as reserved only forgoes its own revenue.
- **Write to:** DOC-09, DOC-28, DOC-53, DOC-55.

### DR-51 · Can freezing lose an event still in Kafka? — ⚠ deviates from the original SDD
- **Problem:** Original SDD §10.3 says everything older than the horizon "can no longer change", and §10.4 checks age **at ingest**. But §19.3 keeps usage in Kafka for up to 7 days while ClickHouse is down. An event accepted at age 71 h can reach ClickHouse days later, after `frozen_until` has passed it. Rating then never counts it, which silently under-charges and breaks NFR-05 ("0 lost").
- **Decision (proposed):** Freeze boundary `X = now − lateness_horizon − freeze_lag_cap` (cap 60 s), and freezing runs only when the `usage-writer` group's lag on `usage.raw.v1` is below that cap, as read by `rating` through Kafka `AdminClient`. Reasoning: any event not yet stored was ingested within the last 60 s, so it has `occurred_at ≥ now − 60 s − 72 h = X`, which is inside the unfrozen window. Otherwise the run skips freezing and only costs a longer `FINAL` scan. EXP-08 adds a case: stop ClickHouse past the horizon of a boundary event, restart, and assert the exact total.
- **Write to:** DOC-29, DOC-81, ADR-0013.

### DR-52 · What is a cursor's lifecycle, and how do late events cross cycles?
- **Problem:** Original SDD §10.3 has one cursor per account, component and cycle. §10.4 says an event in a cycle whose invoice was already issued "is rated in the currently open cycle" with its true service period. It is unclear whether its tier position is the old cycle's or the new one's, when an old cursor stops, and when `minimum_amount` applies.
- **Decision (proposed):**
  - A cursor is created at the first usage in a cycle. Columns: `id`, `account_id`, `component_id`, `cycle_id`, `plan_version_id`, `seq`, `cum_quantity`, `cum_amount`, `frozen_until`, `frozen_quantity`, `pending_correction`, `status` (`OPEN`, `PAUSED`, `CLOSED`), `unique (account_id, component_id, cycle_id)`.
  - An event belongs to the cycle containing its `occurred_at`, and tiers continue that cycle's cumulative position, so the price is right for when it happened.
  - The cursor stays `OPEN` until `cycle_end + lateness_horizon + freeze_lag_cap`, then a final rating applies `minimum_amount`, and it goes `CLOSED`.
  - Charges from an old cursor after its invoice was issued go on the next invoice with their true service period, which is what §10.4 asks for.
- **Consequences:** On 5-minute test cycles many cursors stay open for 72 hours. Rating selects only cursors with new usage, so they cost nothing while idle.
- **Write to:** DOC-29, DOC-15, DOC-47.

### DR-53 · How do events reach quarantine, and how does a Finance Admin resolve them? — ⚠ deviates from the original SDD
- **Problem:** Original SDD §10.4 stores events over 72 hours old in `metering.usage_quarantine`, while `ingest` has no database on the path (DR-49) and `usage-writer` only reaches ClickHouse. UC-15 resolves quarantined usage, but there is neither endpoint nor rule for what resolving does.
- **Decision (proposed):** `ingest` sends late events to `usage.exceptions.v1` with `kind: "LATE"`. `worker` inserts into `metering.usage_quarantine` (`UNIQUE (client_id, event_id)`, status `PENDING`, `CHARGED` or `DISCARDED`) and publishes `usage.quarantined`. `POST /v1/admin/usage-quarantine/{id}/resolve` takes `{ "action": "CHARGE" | "DISCARD", "reason": "…" }`. `CHARGE` publishes a usage event to `usage.raw.v1` with `event_id = "quarantine:<id>"` and `occurred_at = now`, rated in the open cycle at the current tier; the original time stays on the quarantine row and in `resource_ref`.
- **Write to:** DOC-28, DOC-29, DOC-53, DOC-69.

### DR-54 · What triggers rating, and what is the sanity limit? — ⚠ deviates from the original SDD
- **Problem:** Original SDD §10.3 has the Scheduler run `DISTINCT account_id` on ClickHouse, but the deployment figure connects `scheduler` to PostgreSQL and Kafka only. §22.1 promises "a per-run sanity limit on rating deltas", with no value or mechanism.
- **Decision (proposed):**
  - `scheduler` gets the `usage_rating` ClickHouse user. Query: `SELECT DISTINCT account_id FROM usage.event WHERE ingested_at >= :last_run - :interval`. Interval by mode from `account_directory`: 5 min prepaid, 60 min postpaid (config per environment).
  - Accounts with a cursor due for freezing or final rating also get a job, found in `metering.rating_cursor`.
  - Sanity limit per cursor: if `Δ > max(rating.sanity.abs_limit, 10 × the cursor's previous Δ)`, nothing posts; the cursor goes `PAUSED`, `RatingSanityPaused` alerts, and `POST /v1/admin/rating-cursors/{id}/resume` continues it. `abs_limit` per currency: 100,000,000 VND, 5,000 USD.
- **Write to:** DOC-29, DOC-37, DOC-42, DOC-43.

### DR-55 · How is ClickHouse operated, and how is it read beyond rating? — 🔬 spike (S-05)
- **Problem:** Original SDD §19.1 says "Operator" with no name, and sets no retention on 86 million rows per day. §10.2 says dashboards read "a separate rollup" that is not defined, and `GET /v1/accounts/{id}/usage` needs a source. Batch retries are not deduplicated by design. §10.3 estimates about 13 million rows on a hot account's `FINAL` and defers measuring it to EXP-08 at S4, although ADR-level rating design rests on it.
- **Decision (proposed):**
  - Altinity `clickhouse-operator`. `ReplicatedReplacingMergeTree` as in original SDD §10.2.
  - `TTL occurred_at + INTERVAL 13 MONTH`: explanation beyond that uses the charge's rating snapshot.
  - Rollup: `usage.event_hourly`, a `SummingMergeTree` fed by a materialized view. It counts duplicates, so `GET /v1/accounts/{id}/usage` returns `"is_approximate": true`. It is never used for money.
  - Writer: official Java client, RowBinary, up to 10,000 rows or 1 s; `insert_deduplication_token = <topic>-<partition>-<first>-<last>` so a retried batch writes once.
  - S-05 loads 13 million rows on one sort-key prefix and times the §10.2 query. Pass: p95 ≤ 2 s on the staging node size. If it fails: shorten the horizon or lengthen that account's interval, via a new DR.
- **Write to:** DOC-11, DOC-17, DOC-28, DOC-53.

---

## J. Webhooks and catch-up feed

### DR-56 · How can a delivery be unique on a table partitioned by month? — ⚠ deviates from the original SDD
- **Problem:** Original SDD §16.2 requires `delivery UNIQUE (event_id, subscription_id)`, and §16.1 partitions `delivery` by month. §16.1 itself explains that PostgreSQL requires a unique constraint to include the partition key, so this exact pair cannot exist. If the partition key were the insert time, a replay in a later month would insert a second row and a second delivery. `webhook.event_log` (§11.6) holds about 26 million rows per day for 90 days, and has no partitioning or uniqueness stated at all.
- **Decision (proposed):** Partition by the **event's** time, carried in the envelope and identical on every replay:
  ```sql
  CREATE TABLE webhook.delivery (
      id                uuid        NOT NULL,
      event_id          uuid        NOT NULL,
      event_occurred_at timestamptz NOT NULL,
      subscription_id   uuid        NOT NULL,
      status            text        NOT NULL,
      attempt_count     smallint    NOT NULL DEFAULT 0,
      next_attempt_at   timestamptz NOT NULL,
      lease_until       timestamptz,
      PRIMARY KEY (event_occurred_at, id),
      UNIQUE (event_occurred_at, event_id, subscription_id)
  ) PARTITION BY RANGE (event_occurred_at);
  CREATE INDEX ON webhook.delivery (next_attempt_at) WHERE status IN ('PENDING','RETRYING');
  CREATE INDEX ON webhook.delivery (lease_until) WHERE status = 'DELIVERING';

  CREATE TABLE webhook.event_log (
      occurred_at  timestamptz NOT NULL,
      event_id     uuid        NOT NULL,
      account_id   uuid,
      client_id    uuid,
      kafka_offset bigint      NOT NULL,
      event_type   text        NOT NULL,
      payload      jsonb       NOT NULL,
      PRIMARY KEY (occurred_at, event_id)
  ) PARTITION BY RANGE (occurred_at);
  CREATE INDEX ON webhook.event_log (account_id, kafka_offset);
  ```
  `attempt` is partitioned by `attempted_at`. All three are dropped at 90 days. A redeliver call on an event past 90 days returns `404`.
- **Write to:** DOC-16, DOC-30.

### DR-57 · How exactly does the feed work?
- **Problem:** Original SDD §11.6 says "a consumer" stores events and that the cursor "encodes the topic version and the offset". The response shape, first page, empty page and writer are undefined.
- **Decision (proposed):**
  - The `webhook-subscriber` group writes `event_log` and matching `delivery` rows **in one transaction**, then commits the offset. One consumer, one failure mode.
  - A replayed event hits the primary key and is ignored, so the feed shows it once.
  - Cursor: base64url of `{"v":1,"t":"billing.events.v1","o":123456}`. Omitted `after` starts at the oldest retained row.
  - `limit` default 100, max 500. Response: `{ "data": [...], "next_cursor": "…", "has_more": false }`. An empty page returns the same cursor.
  - Offset older than retained: `410 cursor_expired`. Filter per DR-38. Read from the primary of Delivery DB.
- **Write to:** DOC-16, DOC-31, DOC-53.

### DR-58 · How do the per-endpoint limit and the circuit breaker work across replicas? — ⚠ deviates from the original SDD
- **Problem:** Original SDD §11.5 sets "at most 10 in-flight" per endpoint and a breaker where deliveries "are pushed forward", with 3 `webhook-delivery` replicas. A Resilience4j breaker is per process, so each replica opens separately. Pushing forward means bulk `UPDATE`s on a hot table. The claim query in §11.2 has no endpoint condition, so a dead endpoint's backlog at the front of `next_attempt_at` keeps being claimed ahead of healthy endpoints, breaking NFR-09. No poll interval is stated, although NFR-08 needs first attempt within 5 s.
- **Decision (proposed):**
  - State shared in one table:
    ```sql
    CREATE TABLE webhook.endpoint_state (
        subscription_id uuid PRIMARY KEY,
        consecutive_failures int NOT NULL DEFAULT 0,
        circuit_open_until timestamptz,
        last_success_at timestamptz
    );
    ```
    Updated only on failure, and on the first success after a failure.
  - Each replica caches the open-circuit set for 1 s, and tracks its own in-flight counts with a limit of `ceil(10 / replicas)`.
  - The claim adds `AND d.subscription_id <> ALL(:blocked)`. A blocked endpoint's rows are **not touched**, use no attempt and cost no updates. This replaces "pushed forward".
  - After 60 s one replica claims one row as the trial.
  - Poll every 200 ms when idle. Each replica caps total in-flight at 400.
  - Resilience4j stays for PSP calls only (original SDD §4.3 narrows).
- **Consequences:** Backlogs stay in place and are cheap. EXP-11 measures it against a no-limit baseline.
- **Write to:** DOC-16, DOC-30, DOC-43.

### DR-59 · Which HTTP client can make the SSRF rules true? — 🔬 spike (S-09)
- **Problem:** Original SDD §11.5 requires each delivery to "resolve the host, reject private ranges, connect to the checked address, follow no redirects", which closes DNS rebinding. The JDK `HttpClient` has no per-client resolver hook. No client is named.
- **Decision (proposed):** Apache HttpClient 5 classic on virtual threads, with a custom `DnsResolver` that resolves once and throws when any address is in the blocked set, so the connection uses exactly the checked address. Hostname verification and SNI stay on the original name. Redirects off, connect timeout 3 s, total 10 s, response body read limit 64 KB. Blocked: `0.0.0.0/8`, `10.0.0.0/8`, `100.64.0.0/10`, `127.0.0.0/8`, `169.254.0.0/16`, `172.16.0.0/12`, `192.0.0.0/24`, `192.168.0.0/16`, `198.18.0.0/15`, `224.0.0.0/4`, `240.0.0.0/4`, `::/128`, `::1/128`, `::ffff:0:0/96`, `64:ff9b::/96`, `fc00::/7`, `fe80::/10`, `ff00::/8`. The `webhook.internal_endpoint_allowlist` of §11.5 skips the check for exact hosts only. S-09 proves a rebinding test against a stub DNS and throughput on virtual threads.
- **Write to:** DOC-11, DOC-30, DOC-41.

### DR-60 · How are webhook and inbound secrets stored and issued?
- **Problem:** Original SDD §11.4 sets AES-256-GCM and 24-hour rotation overlap. §11.1 uses "a separate inbound secret", with no table or endpoint to obtain or rotate it.
- **Decision (proposed):**
  ```sql
  CREATE TABLE webhook.secret (
      subscription_id uuid NOT NULL, version int NOT NULL,
      key_version int NOT NULL, nonce bytea NOT NULL, ciphertext bytea NOT NULL,
      created_at timestamptz NOT NULL, expires_at timestamptz,
      PRIMARY KEY (subscription_id, version)
  );
  ```
  `platform.client_inbound_secret` has the same shape. Billing generates secrets: `whsec_` + 32 random bytes in base64url, returned once. AAD for GCM is the owner ID. The key comes from `WEBHOOK_KEK_<version>` in the secret manager. A new secret sets `expires_at = now + 24h` on the old one; a job purges expired ones. `POST /v1/client-systems/me/rotate-inbound-secret` for the Client, and the same under `/v1/admin/client-systems/{id}` for Finance Admins.
- **Write to:** DOC-15, DOC-16, DOC-30, DOC-41.

### DR-61 · What are the delivery headers, and how do dead-delivery events avoid a loop? — ⚠ deviates from the original SDD
- **Problem:** Original SDD §11.3 publishes `webhook_delivery.dead` to a tenant's other endpoints. If those are also failing, each death creates a new event that dies in turn, a loop between two dead endpoints. §11.5 disables an endpoint after 7 days with no success, with the fate of its pending rows unstated, and §11.4 and §17.3 name only some headers.
- **Decision (proposed):**
  - Headers: `Billing-Signature`, `Billing-Event-Id`, `Billing-Event-Type`, `Billing-Delivery-Id`, `Billing-Api-Version`, `traceparent`, `User-Agent: GenericBilling-Webhook/1`.
  - No `webhook_delivery.dead` is published for a delivery whose event type starts with `webhook_`.
  - Auto-disable (hourly job): `last_success_at`, or `created_at` if never, older than 7 days and at least one failed attempt. Its `PENDING` and `RETRYING` rows become `DEAD` with `reason = endpoint_disabled`, without one event per row; one `webhook_endpoint.disabled` is published. New events stop matching. `PATCH {"status":"ACTIVE"}` re-enables it, and old rows need redelivery.
  - Response body: stored in `attempt` up to 1 KB.
- **Write to:** DOC-30, DOC-54.

---

## K. Tenancy and security

### DR-62 · How does an ID-addressed request find its tenant under RLS? — ⚠ deviates from the original SDD
- **Problem:** A real contradiction. Original SDD §12.1 takes a tenant's tenant from "the requested account's tenant". But `POST /v1/reservations/{id}/commit`, `release`, `extend`, `GET /v1/reservations/{id}` and `GET /v1/charges/{id}` name no account, and the ID carries a shard, not a tenant. §6.5 says a commit "first reads its reservation to learn the wallet". §12.3 says that with no tenant set, RLS returns zero rows, so that first read finds nothing. The PSP callback (§13.1) and the same-key replay path have the same problem.
- **Options:** (1) Put the tenant in the public ID: a long-lived leak of identifiers, and a format the contract would be stuck with. (2) A second RLS policy on `tenant_system_id` for tenant_system reads: spreads a second isolation axis across many tables. (3) A narrow locator that bypasses RLS for immutable identifiers only.
- **Decision (proposed):** Option 3.
  ```java
  public interface TenantLocator {
      Optional<Located> locate(LocatorKind kind, UUID id, short logicalShard);
  }
  public record Located(TenantId tenant, CustomerId customer, AccountId account, WalletId wallet, @Nullable TenantSystemId owner) {}
  ```
  - Uses role `billing_locator` (DR-17), with column-level `SELECT` on `reservation`, `charge`, `payment`, `refund`, `invoice`, `subscription`: only `id`, `tenant_id`, `customer_id`, `account_id`, `wallet_id`, `tenant_system_id`. No amounts.
  - Cached in Caffeine for 1 hour (immutable). A reserve fills the cache on its replica.
  - Sequence for a tenant_system: locate → `owner` must equal the caller → enrollment must exist (`PlatformDirectory`, cached 30 s) → bind the tenant → normal RLS transaction. Any failure is `404`.
  - tenant_customers use the tenant and customer from their token, and RLS decides. platform_users: DR-65.
  - An ArchUnit rule limits the port to `adapter.in`. EXP-12 covers it.
- **Consequences:** One extra read per first touch of an ID on a replica. Grant revocation takes effect within 30 s.
- **Write to:** DOC-19, DOC-26, DOC-46, ADR-0014.

### DR-63 · Which tables are exempt from RLS? — ⚠ deviates from the original SDD
- **Problem:** Original SDD §5.5 fails the build for any table with `tenant_id` lacking a forced policy. §12.1's lookup in `platform.account_directory` and `customer_enrollment` must work **before** a tenant is known, and both have `tenant_id`. The rule and the mechanism cannot both hold.
- **Decision (proposed):** Exactly three tables are exempt, named in a fixed list inside the RLS coverage test: `platform.customer`, `platform.account_directory`, `platform.customer_enrollment`. They hold identifiers and no money. Only `PlatformDirectory` reads them (ArchUnit). `platform.customer_billing_profile` has personal data and stays under RLS. Adding to the list needs a DR.
- **Write to:** DOC-19, DOC-26, ADR-0014.

### DR-64 · Which identity provider, with which claims and users?
- **Problem:** Original SDD §4.3 and §22.2 leave the production provider open. It mentions a `tenant_id` claim and a `finance_admin` role, but no token lifetime, mapping from token to `tenant_system`, or how a tenant_customer user comes to exist.
- **Decision (proposed):** Self-hosted Keycloak in every environment, backed by CloudNativePG, with any OIDC provider able to replace it if it issues the same claims.

  | Claim | Value | Used for |
  | --- | --- | --- |
  | `sub` | User or service account | Caller, audit |
  | `azp` | OIDC client | Maps to `tenant_system.idp_client_id` |
  | `tenant_id` | UUID, from a user attribute | Business (tenant) context |
  | `customer_id` | UUID, from a user attribute | Tenant-customer context (within the tenant) |
  | `realm_access.roles` | `demo_operator` (realm `billing`); `billing_user` (platform and tenant realms); `finance_admin` removed, see below | Door only; rights come from permission bits (DR-106) |
  | `aud` | `billing-api` | Validated at gateway and application |

  Access token 5 minutes. JWKS cached 1 hour, and last good keys stay in use on failure (original SDD §19.3). platform_users with finance bits require OTP. tenant_customer users are provisioned by a platform_user in Keycloak in this phase, with no self-signup and one identity per tenant. There are no finer tenant-customer roles.

  **Amended by DR-107:** there are three kinds of realm. `billing` keeps tenant_customers, tenant_systems and `demo_operator`. `billing-platform` holds platform users, and `billing-tenant-<code>` holds each tenant's users, so the same email can exist once per space. The role `finance_admin` is removed; the realm role `billing_user` only opens the door to a row in `platform.platform_user` or `platform.client_user`, and rights come from permission bits (DR-106). Tenant tokens also carry `billing_space` and `billing_tenant_id`; tenant-customer tokens add `billing_customer_id`.
- **Write to:** DOC-26, DOC-41, DOC-72, ADR-0023.

### DR-65 · How does a Finance Admin name a tenant, and how do cross-tenant queues work? — ⚠ deviates from the original SDD
- **Problem:** Original SDD §3.1 says "every cross-tenant read names a tenant", and §12.1 gives "one tenant at a time". But §18.1 needs a refund approval queue and exceptions list across tenants. `CrossTenantScan` is limited to job candidates (§12.3).
- **Decision (proposed):**
  - Admin calls on an object find the tenant through `TenantLocator`; calls listing one tenant's data take the header `Billing-Tenant-Id`. A missing header on such a call is `400`.
  - Queues (`GET /v1/admin/refunds?status=REQUESTED`, quarantine, exceptions): `CrossTenantScan` returns `(tenant_id, id)` candidates, and each row is then read in its own tenant context. Page size ≤ 50. So each row's read still names its tenant, and the rule stands.
  - One audit entry per call with the tenant list.
  - The finance overview shows counts only.
  - ArchUnit allows `CrossTenantScan` in job handlers and these named query handlers.
- **Write to:** DOC-26, DOC-35, DOC-41.

### DR-66 · What is the endpoint × role matrix?
- **Problem:** Original SDD §17.1 gives only "`/v1/admin/*` needs `finance_admin`". §11.7 says a Client never holds Customer rights, and §17 says "everything the portal does, a Client could do with suitable permissions". Nobody can implement authorization from that.
- **Decision (proposed):** TS = tenant_system, TC = tenant_customer, P = platform user (a "Finance Admin" is a platform user holding the finance bits). Scope is enforced as DR-38 and DR-62.

  | Endpoints | TS | TC | P |
  | --- | --- | --- | --- |
  | Estimates | own | own tenant | any, draft preview |
  | Reservations (all) | own | read own tenant | read |
  | Usage `POST` | own | no | no |
  | Account read, transactions, movements, usage | granted | own tenant | named tenant |
  | Feed | granted, filtered | no | named tenant |
  | Charges read | own | own tenant | any |
  | Payments, Refunds `POST` | no | own tenant | refunds only, named tenant |
  | Payments, Refunds read | granted | own tenant | any |
  | Invoices read | granted, totals only | own tenant | any |
  | Subscriptions | own | read own tenant | read |
  | Products read | own | own tenant | any |
  | Webhook subscriptions, deliveries | own | no | read any, redeliver any |
  | `/v1/admin/*` | no | no | yes |

  `/v1/psp/*/callback` and `/v1/inbound/*` have no token and are authenticated by signature only.

  **Amended by DR-106 and DR-109:** the P column is no longer a role. It means "a platform_user", and each `/v1/admin/*` endpoint names one permission code (`group.function.action`) the user must hold. A new column K ("a tenant_user") covers `/v1/tenant/*`, and a token in the wrong space is `403 wrong_space`. TS and TC are unchanged. The path is the outer boundary; the permission bit is the inner check.
- **Write to:** DOC-41, DOC-53.

### DR-67 · Where is each rate limit enforced? — ⚠ deviates from the original SDD
- **Problem:** Original SDD §12.4 places "per client and per tenant" buckets at the gateway, but a tenant's token has no tenant, and the tenant is known only after the account is looked up (§12.1). "2,000 events per second per client" cannot be counted at a gateway that sees requests, not events. "Reservations per account" needs the account inside the body or behind an ID.
- **Decision (proposed):**

  | Limit | Default | Where | Key | Over the limit |
  | --- | --- | --- | --- | --- |
  | Client API calls | 500/s | Gateway | `azp` | `429` |
  | Customer API calls | 100/s | Gateway | `tenant_id` claim | `429` |
  | Inbound webhook, no token | 200/s | Gateway | `{client_id}` in path | `429` |
  | PSP callback | 50/s | Gateway | Source IP allow-list per PSP | `429` or `403` |
  | Reservations per account | 100/s | `api`, before the gate | `account_id` | `429` |
  | Usage events | 2,000/s | `ingest` handler, counting events | `client_id` | `429` for that batch |

  All Bucket4j, in memory, with capacity divided by replica count from Helm values. Overrides per Client in `client_system.rate_limits`, read at 30 s. Load experiments set overrides explicitly (DR-100).
- **Write to:** DOC-38, DOC-41, DOC-43.

### DR-68 · How is TLS done inside the cluster, and which secret manager is used?
- **Problem:** Original SDD §19.5 says "TLS at the edge and within the cluster", and §19.1 "an external secret manager", both without a mechanism. §19.8 uses plain development values on mini.
- **Decision (proposed):**
  - cert-manager with an internal CA issues service certificates; apps serve on 8443.
  - PostgreSQL and PgBouncer use CloudNativePG-managed TLS with `sslmode=verify-full`. Kafka's internal listener uses TLS and SCRAM, and the external listener mTLS. ClickHouse uses TLS.
  - External Secrets Operator syncs from the provider's manager (DR-91) on staging and production. Mini uses plain Kubernetes Secrets.
  - `tls.internal.enabled` is `false` on mini for gateway-to-app and ClickHouse only, to save resources; the mTLS Kafka listener stays on, because the Demo Client exercises it.
  - A service mesh is not used.
- **Write to:** DOC-41, DOC-73, DOC-74, DOC-75.

### DR-69 · How do refund limits and dual control work? — ⚠ deviates from the original SDD
- **Problem:** FR-19 and original SDD §13.3 use `refund_auto_approve_limit`. §19.5 requires "a second approver" for high-value adjustments and refunds, naming `adjustment_dual_control_limit`. But the refund state machine has no second-approval state, adjustments are one call with no state at all, and no values are given.
- **Decision (proposed):**

  | Setting | Default (VND) | Effect |
  | --- | --- | --- |
  | `refund_auto_approve_limit` | 0 | Every refund needs a Finance Admin |
  | `refund_dual_control_limit` | 50,000,000 | At or above this, two different admins |
  | `adjustment_dual_control_limit` | 10,000,000 | At or above this, two different admins |

  Refund state `PENDING_SECOND_APPROVAL` (DR-72). New table `ledger.adjustment` (`id`, `status` = `PENDING`, `POSTED`, `REJECTED`, `requested_by`, `approved_by`, `direction`, `amount`, `reason`). Below the limit, the adjustment posts in the request; at or above, it waits, and `POST /v1/admin/adjustments/{id}/approve` posts it. The approver must differ from the requester, a rule checked in the handler and also applying to refunds. Values per currency.
- **Write to:** DOC-14, DOC-32, DOC-41, DOC-67.

---

## L. Payment and Refund

### DR-70 · Which PSP is first? — 🔬 spike (S-07)
- **Problem:** Original SDD §13.2 and §22.2 leave it open: "Not chosen. Verify it against section 13.2". Without one, S5's "first real PSP" and the reconciliation report format cannot be designed. The SDD's state machine also assumes a server-side create call, and some PSPs only build a redirect URL.
- **Decision (proposed):** S-07 scores two Vietnam-capable candidates, to be named by the Owner (VNPAY and PayOS are the suggested starting list), against the five capabilities of §13.2 using their sandboxes and published documents. The first PSP is the one passing all five. `FakePsp` carries everything until then.
  ```java
  public interface PaymentGateway {
      CreateResult create(CreatePayment cmd);                 // cmd.reference() = payment_id
      VerifiedEvent verifyCallback(byte[] rawBody, Map<String, String> headers);
      PaymentStatus query(PaymentReference reference);
      RefundResult refund(RefundCommand cmd);                 // cmd.reference() = refund_id
      RefundStatus queryRefund(RefundReference reference);
      SettlementReport settlementReport(LocalDate businessDate);
  }
  ```
  If no candidate passes everything: a PSP with redirect-only create goes `CREATED → PENDING` when the URL is built, and has no `UNKNOWN`. A PSP without a report reconciles from daily queries, and NFR-14's scope narrows in writing.
- **Consequences:** Blocks P6, not earlier phases.
- **Write to:** DOC-32, DOC-34, ADR-0024.

### DR-71 · What happens at the edges of the payment state machine? — ⚠ deviates from the original SDD
- **Problem:** Original SDD §13.1 leaves gaps that lose or mis-state money. (a) A process dying after committing `CREATED`, before calling the PSP, leaves a row the poller ignores, since it "resolves `UNKNOWN`, expires `PENDING`". (b) "PSP query says … not found" gives `FAILED`, which is terminal, but a slow PSP can report success later, while the arbiter `UPDATE` excludes `FAILED`, so **real money arrives and is not credited**. §13.1 credits a late `EXPIRED` for exactly this reason. (c) A callback amount differing from the row's amount has no rule. (d) The payment window has no value. (e) The `POST /v1/payments` body is undefined.
- **Decision (proposed):**
  - Request: `{ "account_id", "kind": "TOP_UP" | "INVOICE", "amount", "invoice_id"?, "return_url" }`. `return_url` is checked against an allow-list. Limits `payment.min_amount` and `payment.max_amount` per currency.
  - (a) The poller treats `CREATED` older than 60 s as `UNKNOWN`.
  - (b) `UNKNOWN → FAILED` on "not found" only after 3 answers spanning at least 15 minutes. `FAILED → SUCCEEDED` is allowed on a verified callback or query, publishing `payment.succeeded` and raising a `LATE_SUCCESS_AFTER_FAILED` exception for review. The arbiter's status list gains `'FAILED'`.
  - (c) New column `captured_amount`. The wallet is credited with what the PSP verifiably captured, and a mismatch raises `AMOUNT_MISMATCH`. A currency mismatch changes no state and raises an exception. `refunded_amount` is capped by `captured_amount`.
  - (d) `payment.window_seconds` defaults to 1,800, per PSP.
  - (e) A callback for an unknown reference stores `psp_event` with `payment_id = NULL`, raises `UNKNOWN_PAYMENT` and returns `2xx`.
- **Write to:** DOC-15, DOC-32, DOC-49, ADR-0024.

### DR-72 · Which refund states are missing? — ⚠ deviates from the original SDD
- **Problem:** Original SDD §13.3 enters at `REQUESTED: hold placed on the wallet`, but also says the hold is placed by a separate Ledger transaction after Payment commits, with a resume job scanning "`REQUESTED` refunds with no recorded hold". So `REQUESTED` means both "hold placed" and "maybe not yet". A hold failure "compensates", but there is no state for it. `APPROVED → PROCESSING` has no timeout outcome, although a refund call can time out exactly as a payment can, and neither the worker nor any refund polling appears in §15.3.
- **Decision (proposed):**
  ```mermaid
  stateDiagram-v2
      [*] --> HOLD_PENDING: refund row written, refunded_amount raised
      HOLD_PENDING --> REQUESTED: Ledger hold placed
      HOLD_PENDING --> REFUSED: insufficient available, refunded_amount restored
      REQUESTED --> APPROVED: auto, or admin below dual-control limit
      REQUESTED --> PENDING_SECOND_APPROVAL: first admin, at or above the limit
      PENDING_SECOND_APPROVAL --> APPROVED: second, different admin
      REQUESTED --> REJECTED: admin, hold released
      PENDING_SECOND_APPROVAL --> REJECTED: admin, hold released
      APPROVED --> PROCESSING: state committed, then PSP called
      PROCESSING --> UNKNOWN: timeout
      UNKNOWN --> PROCESSING: PSP query in progress
      PROCESSING --> SUCCEEDED: verified result
      UNKNOWN --> SUCCEEDED: verified result
      PROCESSING --> FAILED: PSP refused, hold released
      UNKNOWN --> FAILED: PSP refused or not found after grace
  ```
  - `POST /v1/refunds` runs the first two steps synchronously and returns `201` or `409 insufficient_balance`. `refund.requested` is published on entering `REQUESTED`, not before.
  - Jobs (DR-85): resume `HOLD_PENDING`; send `APPROVED` to the PSP; poll `PROCESSING` and `UNKNOWN`. Each minute.
  - The PSP reference is the refund ID, so a resend is safe (§13.2).
- **Write to:** DOC-15, DOC-32, DOC-49.

### DR-73 · Can a postpaid account top up, and what happens on overpayment? — ⚠ deviates from the original SDD
- **Problem:** FR-13 lets "a customer" top up, while original SDD §6.4 funds postpaid by invoice payment. A postpaid top-up would raise `balance` but leave the invoice `ISSUED` and later `overdue`, even though the customer has paid. Two concurrent payments on one invoice can together exceed it (§14.2 only accumulates `amount_paid`).
- **Decision (proposed):**
  - `POST /v1/payments` with `TOP_UP` on a postpaid account returns `422 top_up_not_allowed`. `INVOICE` on a prepaid account returns the same error, since a statement is `PAID` on issue.
  - An invoice payment's amount must be at most the remaining amount at creation, default the remaining amount.
  - Overpayment from a race: `invoice.amount_paid` stops at `total`. The surplus stays in the wallet as a positive balance, reported as `overpaid_amount`.
- **Write to:** DOC-32, DOC-33.

### DR-74 · How are reversals modeled, including partial ones? — ⚠ deviates from the original SDD
- **Problem:** Original SDD §6.1 keys a reversal `reversal:<journal_id>`, allowing exactly one per journal. But §14.1 reverses a prorated part of a fee on downgrade, and an admin may reverse in parts. `invoice_line UNIQUE (charge_id)` and `invoice_charge UNIQUE (charge_id)` need the reversal to have its own charge row. `charge` is append-only, so a running "amount reversed" cannot live on the original row. The API takes no amount.
- **Decision (proposed):**
  - `charge.kind`: `RESERVATION`, `RATING`, `FEE`, `PRORATION`, `REVERSAL`, `RATING_CREDIT`, `ADJUSTMENT`. Credit kinds are stored with a positive `amount`; the kind gives the sign.
  - `POST /v1/admin/charges/{id}/reverse` body: `{ "amount"?: 1000, "reason": "…" }`, default the remaining amount.
  - Source key: `reversal:<charge_id>:<reversal_id>`, with `reversal_id` from the request's idempotent row.
  - Under the wallet lock, the handler sums existing reversals by `reverses_charge_id` and returns `409 reversal_exceeds_charge` beyond the original amount.
  - Invoicing treats `REVERSAL` and `RATING_CREDIT` as negative lines.
- **Write to:** DOC-14, DOC-21, DOC-33.

---

## M. Invoicing and Subscription

### DR-75 · Who owns account settings, and how is an account created across stores? — ⚠ deviates from the original SDD
- **Problem:** `POST /v1/admin/accounts` (original SDD §17.1) sets mode, `credit_limit`, `cycle_length` and `payment_terms` together, but §16 places `billing_account` and `wallet` in Ledger DB and `billing_cycle` in Ops DB, and no transaction may cross stores (§4.2). `cycle_length` has no owning table, and no module may read another's schema.
- **Decision (proposed):**

  | Data | Owner |
  | --- | --- |
  | `tenant_id`, `logical_shard`, currency, `mode` (cache), status | `platform.account_directory` |
  | Mode, `pending_mode`, currency, wallet, `credit_limit`, `low_threshold` | `ledger.billing_account`, `ledger.wallet` |
  | `cycle_length`, `payment_terms`, `anchor_at`, pending values | `invoicing.account_settings` |

  Saga with a client-supplied or generated `account_id`:
  1. Ops transaction: directory `PROVISIONING` and settings.
  2. Ledger transaction on the shard: account, wallet, customer ledger account, with `ON CONFLICT DO NOTHING`.
  3. Ops transaction: directory `ACTIVE`, first `billing_cycle`.

  A job resumes any `PROVISIONING` row older than 1 minute. Every other operation on a `PROVISIONING` account returns `404`. `PATCH` routes each field to its owner.
- **Write to:** DOC-14, DOC-15, DOC-33, DOC-51.

### DR-76 · How is a cycle calculated?
- **Problem:** Original SDD §14.1 says "monthly by default, anchored at the account's creation day". It does not say in which time zone a day starts, how the 31st behaves, or what type `cycle_length` is. §16.3 says all times are UTC, but Vietnamese customers expect a local month.
- **Decision (proposed):**
  - `cycle_length` and `payment_terms` are ISO 8601 durations: `P1M`, `P1D`, `PT5M`. Floors come from configuration (original SDD §19.8).
  - `anchor_at` is local midnight of the creation day in `tenant.billing_timezone`, default `Asia/Ho_Chi_Minh`.
  - Calendar cycles compute from the anchor, not from the previous end: anchored on Jan 31 gives Feb 28, then Mar 31.
  - Sub-day cycles: `anchor_at + n × length`.
  - Rows store UTC bounds. A change of length starts at the next cycle start.
  - Property tests cover month ends, leap years and time zones (original SDD §11.8).
- **Write to:** DOC-15, DOC-33.

### DR-77 · What is the subscription state machine, and which source keys does proration use? — ⚠ deviates from the original SDD
- **Problem:** The original SDD names `subscription.activated`, `.fee_unpaid` and `.canceled`, but gives no state machine, although each entity with a status needs one. §14.1 keys fees `cycle:<cycle_id>:<subscription_id>`, and uses the same key for a mid-cycle first fee. A mid-cycle *change* needs a second journal in the same cycle, and two changes need a third, which that key cannot express. `DELETE` has no money rule. §11.8's formula needs `cycle_end` and second precision that no response returns.
- **Decision (proposed):**
  ```mermaid
  stateDiagram-v2
      [*] --> ACTIVE: created, fee posted
      [*] --> FEE_UNPAID: created, fee skipped
      FEE_UNPAID --> ACTIVE: fee posted
      ACTIVE --> FEE_UNPAID: next cycle fee skipped
      ACTIVE --> CANCELED: DELETE
      FEE_UNPAID --> CANCELED: DELETE
  ```
  - Keys: fee `cycle:<cycle_id>:<subscription_id>`; change `subscription-change:<change_id>`, with a new row in `invoicing.subscription_change`.
  - Upgrade charges `prorate(new − old, t)` with `kind = PRORATION`. Downgrade reverses `prorate(old − new, t)` of the current fee (DR-74). `prorate = false` applies at the next cycle.
  - Times are the database clock truncated to whole seconds, used in the formula and returned as is: `prorate(f, t) = round_half_up(f × (cycle_end_s − t_s) / (cycle_end_s − cycle_start_s))`.
  - Responses include `activated_at` or `changed_at`, and `current_cycle: { id, start_at, end_at }`.
  - `DELETE` is immediate: future fees stop, with no automatic credit. A Finance Admin may reverse by hand.
- **Write to:** DOC-15, DOC-33, DOC-40, DOC-50.

### DR-78 · When exactly does an unpaid fee retry? — ⚠ deviates from the original SDD
- **Problem:** Original SDD §14.1 retries "when `balance.restored` arrives". But §6.7 publishes that only on entering `NORMAL` from `LOW` or `INSUFFICIENT`. A wallet at 300,000 available and `NORMAL`, owing a 500,000 fee, that tops up by 300,000 stays `NORMAL` and never fires it; a top-up leaving it `LOW` fires `balance.low`. The fee then stays unpaid forever despite funds. The fee check is also outside the lock, so it can race. §22.2 also leaves the first-fee proration open.
- **Decision (proposed):**
  - Retry on `balance.credited` or `balance.restored`, plus hourly for any `FEE_UNPAID` in its own cycle.
  - `PostCharge` takes `requireAvailable: true`, evaluated by the aggregate under the wallet lock. It returns `InsufficientAvailable`, posts nothing, and changes nothing.
  - The amount stays as computed at cycle start, with no time reduction for lateness.
  - `subscription.fee_unpaid` is published once per cycle per subscription.
  - At cycle end, an unposted fee is dropped: no back-billing. The Client decides what that means.
  - First fee of a mid-cycle subscription: prorated, as original SDD §14.1 proposes (Owner to confirm).
- **Write to:** DOC-33, DOC-40, DOC-50.

### DR-79 · What exactly does the completeness check compare? — ⚠ deviates from the original SDD
- **Problem:** Original SDD §14.1 sums "uninvoiced charges in the projection, for service periods that start before `cycle_end`", and asks Ledger "for the total for the same scope". Ledger has no idea what is invoiced, so it cannot compute that scope.
- **Decision (proposed):** Compare what each side has seen in a closed time window, using `posted_at`:
  1. `watermark = now − 1 minute`, `from` = the previous close's watermark for this account (the account's creation at first).
  2. `LedgerApi.chargeTotals(account, from, watermark)` returns `(count, net_amount)` from `charge`, with index `(account_id, posted_at)`.
  3. Compare with the same window in `invoice_charge`. If they differ, the projection is behind: retry in 1 minute, alert after 15.
  4. If they match, in one transaction: select uninvoiced rows with `service_from < cycle_end`, write the invoice and lines, mark them, and store `watermark` on the cycle.
- **Consequences:** A charge posted after the watermark falls to the next invoice, which original SDD §14.1 already accepts.
- **Write to:** DOC-33, DOC-50.

### DR-80 · What are the remaining invoice rules? — ⚠ deviates from the original SDD
- **Problem:** Original SDD §14.2 leaves undefined: a cycle with no charges; a zero or negative total (DR-45, DR-74); the number format; voiding a partly paid invoice. §16.2 has `invoice_line UNIQUE (charge_id)` "for non-void invoices", which a partial index cannot express, because the status is on another table, and "lines are immutable once issued".
- **Decision (proposed):**
  - No charges in the cycle: no invoice and no number. `billing_cycle.closed` has `invoice_id: null`.
  - Total ≤ 0: `DRAFT → PAID` on either mode, and a negative total stays in the wallet.
  - Number: `INV-<YYYY>-<6-digit per-tenant sequence>`. The counter never resets, so it is gapless per tenant. A voided invoice keeps its number.
  - Void only with `amount_paid = 0`; otherwise `409 invoice_has_payments`, and the fix is a reversal.
  - `invoice_line` has `UNIQUE (invoice_id, charge_id)`, and is never updated. "One live invoice per charge" is enforced in `invoice_charge`: `invoice_id` is nullable, void sets it back to `NULL`, and the invoice-write statement is `UPDATE … WHERE invoice_id IS NULL`.
  - `invoice.overdue` is postpaid only.
- **Write to:** DOC-15, DOC-33.

### DR-81 · How is the invoice PDF produced, and where do the customer's details come from?
- **Problem:** Original SDD §14.2 says "PDF stored", and §19.3 has it retry independently, so it cannot be in the issuing transaction. No library, template or customer name, address or tax code exists in the data model.
- **Decision (proposed):**
  - `platform.tenant_billing_profile`: `tenant_id`, `legal_name`, `tax_code`, `address`, `billing_email`, `tax_rate`. Under RLS. Seller details come from configuration.
  - A job renders each invoice with `pdf_object_key IS NULL`, within a minute of issue, using OpenPDF and an embedded Noto Sans font for Vietnamese text. The PDF is in English first (DR-90).
  - Object: `billing-invoices/<tenant_id>/<number>.pdf`.
  - `GET /v1/invoices/{id}/pdf` returns a 5-minute presigned URL, or `409 pdf_not_ready`.
- **Write to:** DOC-15, DOC-33, DOC-11.

---

## N. Reconciliation, audit, jobs and ERP

### DR-82 · What are the reconciliation mechanics?
- **Problem:** Original SDD §15.1 compares three sources daily, and §19.7 gives it a read-only pool. It does not say which day boundary applies, or how a payment settled near midnight avoids a false "Billing `SUCCEEDED`; PSP has no record", the case that pages someone immediately. A fee rule is "configured" with no table.
- **Decision (proposed):**
  - One `reconciliation.run` per `(psp, currency, business_date)`, unique, with the business date in the PSP's report time zone. It fetches the report at 06:00 that time zone on the next day.
  - It reads through `payment.api` and `ledger.api` query ports on the replica, and writes through `pool-reconcile-rw`.
  - A payment counts in a run only if `settled_at < report_cutoff − reconciliation.settlement_tolerance` (default 2 hours). A later one falls to the next run, staying within NFR-14's 24 hours plus the tolerance.
  - `reconciliation.exception`: `id`, `run_id`, `type`, `severity`, `status` (`OPEN`, `RESOLVED`, `IGNORED`), `tenant_id` nullable, `payment_id` nullable, `psp_reference`, `amount_expected`, `amount_actual`, `owner`, `reason`. Types include `LOST_CALLBACK`, `UNKNOWN_PAYMENT`, `FALSE_SUCCESS`, `AMOUNT_MISMATCH`, `MISSING_POSTINGS`, `LATE_SUCCESS_AFTER_FAILED`.
  - `reconciliation.fee_rule`: `psp`, `currency`, `percent`, `fixed`. When a difference equals the rule's result within 1 minor unit, the fee posts (DR-19) and no exception opens.
- **Write to:** DOC-15, DOC-34, DOC-68.

### DR-83 · What does the audit log actually store, and how does it travel? — ⚠ deviates from the original SDD
- **Problem:** FR-27 and original SDD §15.2 record "every state change on money objects" in `audit.log` through each module's outbox. At sustained load that is tens of millions of rows per day, more than `posting`, retained 10 years in Billing Ops DB, described as "lower write volume" (§16.1). No sizing or partitioning is given. Sharing `billing.events.v1` would also send before-and-after values past every webhook subscriber, and it is absent from the catalog (§8.4) and consumer list (§8.5).
- **Options:** (1) Full copy of every transition: duplicates the append-only logs at great cost. (2) Audit only what is not already append-only.
- **Decision (proposed):** Option 2.

  | Change | Evidence |
  | --- | --- |
  | Hold, commit, release, expiry, charge, credit | `wallet_movement`, `journal`, `posting`, `charge` (already append-only and digested); no `audit.log` row |
  | Reservation status | The `reservation` row and its movement; a transition with no money change is still visible in `reservation` and its outbox event |
  | Payment, refund, invoice, subscription transitions; every admin action; credit limit, mode, plan publish, grants, secret rotation | `audit.log` row |
  | Pricing-mode grant and expiry, user creation, permission and mode changes, deactivation, resource reassignment (DR-105, 107, 108) | `audit.log` row, with the permission mask before and after rendered as codes |

  An "explain this object" query unions both. Audit events have `destination = 'audit'` and go to `billing.audit.v1` (DR-35, DR-36, DR-40). `audit.log` is partitioned monthly by `recorded_at`, with an estimated ≤ 1 million rows per day. FR-27 is restated in DOC-03.
- **Write to:** DOC-03, DOC-15, DOC-35, ADR-0025.

### DR-84 · How are the daily digests defined exactly? — ⚠ deviates from the original SDD
- **Problem:** Original SDD §15.2 says "a SHA-256 digest over that day's journals, postings, wallet movements and audit rows, chained to the previous day's". It gives no row format, day definition or scope. A chain per physical database breaks when a logical shard moves (§19.6). A transaction with `posted_at` before midnight may commit after the digest starts. Audit rows arrive by Kafka, so their event time is unusable.
- **Decision (proposed):**
  - Day = UTC day. The job runs at 01:00 UTC, past the 5-second `idle_in_transaction_session_timeout`.
  - Ledger tables are digested per **logical shard**; `audit.log` by `recorded_at` as one chain.
  - Row form: UTF-8 lines of fixed columns joined by `|`, sorted by primary key, UUIDs lowercase, timestamps in microseconds as RFC 3339 UTC, integers plain.
  - Manifest: `billing-digests/<yyyy>/<mm>/<dd>/manifest.json`, with per-chain `{chain, previous_digest, row_counts, digest}`. `digest = SHA-256(previous_digest || lines)`.
  - Object lock: `COMPLIANCE` in production for 10 years, `GOVERNANCE` on staging, none on mini.
  - Verification: last 7 days nightly, last 90 days weekly, everything on demand. Runs on `-ro`. A mismatch is a critical alert (`RB-01`).
- **Write to:** DOC-35, DOC-20, DOC-37.

### DR-85 · What is the complete job catalog? — ⚠ deviates from the original SDD
- **Problem:** Original SDD §15.3 lists 8 jobs, but other sections need more: refund resume and polling (§13.3), plan retirement (§9.4), webhook auto-disable (§11.5), secret purge (§11.4), partition maintenance, rollups (§6.3), account saga resume, ERP export and PDF rendering. Without one list, some of them will never be built.
- **Decision (proposed):** One table in DOC-37:

  | Job | Interval (prod) | Role | Idempotent because |
  | --- | --- | --- | --- |
  | Reservation expiry sweep | 30 s per database | `scheduler` | Status guard |
  | Usage rating | 5 and 60 min | `scheduler` | Source key, cursor |
  | Cycle start and close | 1 min | `worker` via `scheduler` | Source key, conditional status |
  | Fee retry | 1 h and on events | `worker` | Source key |
  | Overdue invoices | 1 h | `worker` | `overdue_notified_at` |
  | Payment polling | 1 min | `worker` | Conditional status |
  | Refund resume, send, poll | 1 min | `worker` | Unique hold, PSP reference, conditional status |
  | Account saga resume | 1 min | `worker` | `ON CONFLICT` |
  | Invoice PDF render | 1 min | `worker` | Object key from number |
  | Webhook auto-disable; secret purge | 1 h | `webhook-delivery` | Conditional status |
  | Plan version retirement | Daily | `worker` | Conditional status |
  | Idempotency cleanup | Nightly | `worker` | Delete twice is no-op |
  | Partition maintenance | Daily | `reconcile` | `IF NOT EXISTS` |
  | Daily rollup | Daily | `reconcile` | Upsert by date |
  | Wallet checkpoint | Monthly | `reconcile` | Primary key |
  | Reconciliation | Daily | `reconcile` | Unique per run |
  | Digests and verification | Daily | `reconcile` | Object per day |
  | ERP export | Daily | `reconcile` | Sequence, control total |
  | Invariant checks | 15 min, daily | `reconcile` | Read-only |

  Each job fires from `scheduling` under ShedLock, with a metric `billing_job_last_success_timestamp{job}` and one alert for staleness.
- **Write to:** DOC-37, DOC-42.

### DR-86 · What does the ERP export look like with no ERP chosen?
- **Problem:** Original SDD §15.4 and §22.2: "generic files until an ERP is named"; no columns given.
- **Decision (proposed):** Daily batch under `billing-exports/erp/<yyyy-mm-dd>/<seq>/`: `journals.csv` (`journal_id`, `posted_at`, `source_type`, `source_id`, `line_no`, `ledger_account_code`, `direction`, `amount`, `currency`), `invoices.csv` (`number`, `tenant_id`, `issued_at`, `status`, `total`, `tax_amount`, `currency`), and `manifest.json` (`sequence`, `row_counts`, `control_total_debit`, `control_total_credit`, `sha256` per file). The `ErpExporter` port's default adapter writes to object storage only. SFTP and HTTP adapters are built when an ERP is named. The journal file uses the books' fixed account mapping, with positive and negative customer balances split into customer credit and receivables (§6.4).
- **Write to:** DOC-36.

---

## O. API

### DR-87 · Which endpoints does the design need but the catalog omits? — ⚠ deviates from the original SDD
- **Problem:** Original SDD §17.1 cannot support its own screens and flows. There is no account list, though the overview needs one. No way to see active holds (§18.1), per-Client spending (§18.1), quarantined usage (UC-15), a refund queue, or subscription reads. No way to assign a plan or add a plan version. Seed jobs and the runner (§11.7, §11.8) need `cycle_end` and existence checks.
- **Decision (proposed):** Add:

  | Group | Endpoint | Purpose |
  | --- | --- | --- |
  | Account | `GET /v1/accounts` | Accounts of the caller's tenant, or granted ones |
  | Account | `GET /v1/accounts/{id}/reservations?status=ACTIVE` | Active holds, including refund holds |
  | Account | `GET /v1/accounts/{id}/spending?from&to` | Per-Client totals, from the replica |
  | Account | `GET /v1/admin/accounts?tenant_id` | Admin account list |
  | Usage | `GET /v1/usage/rejections` | Kafka channel rejections (DR-50) |
  | Usage | `GET /v1/admin/usage-quarantine`, `POST …/{id}/resolve` | UC-15 (DR-53) |
  | Rating | `POST /v1/admin/rating-cursors/{id}/resume` | DR-54 |
  | Refund | `GET /v1/refunds`, `GET /v1/refunds/{id}`, `GET /v1/admin/refunds` | Track and queue |
  | Adjustment | `GET /v1/admin/adjustments`, `POST …/{id}/approve`, `POST …/{id}/reject` | DR-69 |
  | Charge | `GET /v1/accounts/{id}/charges` | Charges by account |
  | Invoice | `GET /v1/invoices/{id}/pdf` | DR-81 |
  | Subscription | `GET /v1/subscriptions`, `GET /v1/subscriptions/{id}` | Reads, with `current_cycle` |
  | Pricing | `GET /v1/admin/price-plans`, `GET …/{id}`, `POST …/{id}/versions`, `PATCH …/versions/{v}` (draft only), `DELETE …/versions/{v}` (draft only) | Draft editor |
  | Pricing | `POST /v1/admin/price-assignments`, `DELETE …/{id}` | `PriceAssignment` |
  | Platform | `GET /v1/admin/tenants`, `PATCH /v1/admin/tenants/{id}`, `GET /v1/admin/client-systems`, `PATCH …/{id}`, `POST …/{id}/rotate-inbound-secret`, `DELETE /v1/admin/client-grants/{id}` | Administration |
  | Platform | `POST /v1/client-systems/me/rotate-inbound-secret` | DR-60 |
  | Reconciliation | `GET /v1/admin/reconciliation/runs` | Run results |
  | Finance | `GET /v1/admin/overview` | Counts for the overview screen |

  A natural key makes admin creation idempotent for the seed job: `tenant.code`, `client_system.code`, `(account.tenant_id, account.code)`, `plan.code`. A duplicate returns `409` with the existing `id`.

  **Added by DR-109:** pricing-mode grants, platform and tenant_users, permission catalogs and the `/v1/console/*` family.
- **Write to:** DOC-53, DOC-39.

### DR-88 · What are the remaining API conventions?
- **Problem:** Original SDD §17.3 sets main rules but leaves list shape, filters, validation detail and the problem-type base URL open.
- **Decision (proposed):**
  - List: `{ "data": [...], "next_cursor": "…" | null, "has_more": bool }`.
  - Time filters: `from` inclusive, `to` exclusive.
  - `400` extension: `errors: [{ "field": "estimated_usage[0].quantity", "code": "must_be_positive" }]`.
  - Problem `type`: `https://docs.billing.example/problems/<code>`, with the host configurable per environment.
  - Every error carries `code` and `trace_id`.
  - Enums upper-case, JSON `snake_case`, paths kebab-case.
  - Money as JSON integers; quantities as strings.
  - `Retry-After` in seconds on `429` and `503`.
  - Each list endpoint states its sort; default newest first, and feed oldest first.
- **Write to:** DOC-52.

---

## P. Web Portal

### DR-89 · What is the rest of the portal stack?
- **Problem:** Original SDD §4.3 and §18.2 name React, TypeScript, Vite, TanStack Query and i18next, but no router, component library, form handling, OIDC library, runtime configuration or hosting.
- **Decision (proposed):**

  | Concern | Choice | Reason |
  | --- | --- | --- |
  | Router | TanStack Router | Typed search params for filters |
  | Components | shadcn/ui with Tailwind CSS and Radix | Accessible primitives, owned source |
  | Tables, forms | TanStack Table; React Hook Form + Zod | Fits the query stack |
  | OIDC | `oidc-client-ts` | PKCE, in-memory tokens |
  | Silent refresh | Refresh token rotation; on reload, `prompt=none` redirect | Third-party cookies make iframe refresh unreliable |
  | API client | DR-10 | |
  | Tests | Vitest, Testing Library, Playwright | |
  | Runtime config | `/config.js` from a ConfigMap | Same image in every environment |
  | Hosting | `nginxinc/nginx-unprivileged`, behind the ingress | Original SDD §19.1 "Portal static files" |
  | Security | CSP with no inline scripts; CORS at the gateway for the portal origin only | |

  **Amended by DR-110 and DR-111:** the portal becomes three front-ends in one repository (platform dashboard, tenant dashboard, tenant-customer portal) built separately and served by space; runtime config and `<base href>` are rendered per request so one image serves both host and path addressing.
- **Write to:** DOC-11, DOC-56, DOC-57.

### DR-90 · What is the exact screen list and language? — ⚠ deviates from the original SDD
- **Problem:** Original SDD §18's flow figure has "Finance overview" and "Payment result", which the screen table omits. FR-01 has Finance Admins register tenant_systems and grants, but no screen does this, while the cut order expects admin screens may fall to "a CLI" that is not defined. §22.2 leaves the end-user language open.
- **Decision (proposed):** 12 screens: Customer: Account overview, Top up (with Payment result), Transactions, Invoices, Refunds. Finance Admin: Finance overview, Price plans, Refund approvals, Reconciliation, Tenants and limits (with quarantine), tenant_systems and grants (added), Webhook deliveries. An account selector appears in the Customer header. English only at first, with all strings through i18next. The cut fallback is `make admin-cli`, a thin `scripts/billing-admin` script over the admin API, written only if the cut happens. **Amended by DR-111:** 18 screens in three front-ends. The platform dashboard gains "Platform users and permissions" and extends "tenant_systems and grants" with pricing modes, tenant_users and realm; a tenant dashboard (Overview, Users and permissions, Price plans, Products and meters, Resources and reassignment) is added.
- **Write to:** DOC-56, DOC-59, DOC-70.

---

## Q. Operations and deployment

### DR-91 · Which cloud, Kubernetes provider and staging budget?
- **Problem:** Original SDD §22.2 leaves all of these to the Owner, and the mini profile covers development. Staging is first needed at P8, and production at go-live.
- **Decision (proposed):** Working assumption, for the Owner to confirm before P8: managed Kubernetes in a Singapore region (EKS `ap-southeast-1`), self-managed operators as in the SDD, and S3 and Secrets Manager from the same provider. Staging: 3 nodes of 4 vCPU and 16 GB, on-demand, stopped outside rehearsals, cost cap USD 400 per month, as a planned number. Any Vietnamese data-localization duty is confirmed with counsel before go-live, and it can force a local provider. Charts stay provider-neutral: storage class, ingress class and secret backend are values. Comparing managed PostgreSQL, Kafka and ClickHouse is deferred until the first production price is known.
- **Consequences:** Blocks P8 only.
- **Write to:** DOC-74, DOC-75, DOC-01.

### DR-92 · What does the synchronous replication setting really do? — ⚠ deviates from the original SDD · 🔬 spike (S-04)
- **Problem:** Original SDD §19.1 sets `method: any, number: 1` with "1 sync standby, 1 replica", and §22.2 asks the Owner about `dataDurability`. Under `ANY 1`, either standby can acknowledge, so there is no dedicated sync standby. Then "standby slow" in §19.3 and NFR-17's "with the standby slowed" (EXP-02) do not slow commits at all unless **both** are slow. On staging (one standby) it does. §19.8's production table repeats "sync standby, replica" as distinct roles.
- **Decision (proposed):**
  - Keep `method: any`, `number: 1`, `dataDurability: required`. Writes stop rather than risk an unreplicated commit (the safer proposal of §22.2; Owner to confirm).
  - Documents state: production has two quorum standbys, either can serve as acknowledgment and for `-ro` reads.
  - The "slow standby" failure row applies when both are slow, or on one-standby environments.
  - EXP-02 slows both standbys on production-size runs, and reports that separately from a single slow standby.
  - S-04 verifies this with CloudNativePG on real versions.
- **Write to:** DOC-10, DOC-75, DOC-81, ADR-0019.

### DR-93 · What are the backup and restore rules?
- **Problem:** Original SDD §19 covers failover but has no backup section. NFR-12 covers primary loss, not loss of a cluster or a destructive mistake. Usage in ClickHouse is revenue data with 7 days of Kafka behind it.
- **Decision (proposed):**

  | Store | Method | Retention | RPO on cluster loss | Restore target |
  | --- | --- | --- | --- | --- |
  | PostgreSQL | CloudNativePG object-store backups: weekly base, continuous WAL | 30 days PITR | ≤ 5 minutes | ≤ 4 hours |
  | ClickHouse | `clickhouse-backup` to object storage, daily | 30 days | 24 hours, closed by replaying Kafka | ≤ 8 hours |
  | Kafka | None; rebuilt, sources are the outbox and Clients | | | |
  | Object storage | Versioning, object lock | Per DR-97 | | |
  | Keycloak | As PostgreSQL | | | |

  After any PITR: stop relays, run all invariant checks, and replay from `last_good` (DR-37). A restore drill on staging runs each quarter and before go-live, with the time recorded. Single-region only in this phase.
- **Write to:** DOC-77, DOC-79.

### DR-94 · Where do metrics, logs and alerts go, and how is the SLO measured?
- **Problem:** Original SDD §4.3 names the stack, and §19.2 the alerts. It has no alert destination, no retention, and NFR-11 has "SLO monitors" with no definition, though a `503 account_busy` on one hot wallet should not count against everyone's availability.
- **Decision (proposed):**
  - `kube-prometheus-stack`, Loki, Tempo, and an OpenTelemetry Collector. Retention: metrics 30 days, logs 14, traces 7. Mini: Prometheus and Grafana only.
  - Alertmanager: Medium and High to one team chat webhook (Slack or Telegram, Owner picks); Critical also to a phone-paging service (Owner picks), and mini sends nothing.
  - SLI: `1 − (5xx except account_busy) / all requests` at the gateway, per route group; `POST /v1/reservations*` for 99.9% and `POST /v1/usage` for 99.95%. A blackbox probe with a synthetic account runs each minute, for when traffic is quiet.
  - Multi-window burn-rate alerts: 14.4× over 1 h, 6× over 6 h.
- **Write to:** DOC-42, DOC-10.

### DR-95 · Which CI system, runner, branches and supply-chain tools?
- **Problem:** Original SDD §19.8 runs the mini profile at each merge, needing about 8 vCPU and 16 GB, which standard hosted runners may lack. §19.5 says "dependency scanning and image signing". This repository has branches `dev` and `main`, while the SDD speaks only of "the main branch".
- **Decision (proposed):**
  - GitHub Actions in a private repository. Build and unit jobs on standard runners. The k3d job on a larger runner or a self-hosted machine with 8 vCPU and 16 GB, as S-06 measures.
  - Branches: short-lived `feat/*` → PR to `dev` (build, unit, module, architecture tests); `dev` → PR to `main` (adds k3d smoke and, from P9, the matrix); tags `vX.Y.Z` on `main`.
  - Registry: GHCR, tags `sha-<12 hex>` and the release tag.
  - Security: Trivy for images, `dependency-review` plus OWASP Dependency-Check on dependencies, cosign keyless signing, CycloneDX SBOM.
  - Format: Spotless with google-java-format, ESLint and Prettier. Renovate for updates.
- **Write to:** DOC-76, DOC-11.

### DR-96 · How are the charts laid out, and what does each role get? — 🔬 spike (S-06)
- **Problem:** Original SDD §19.1 lists replicas without resources, and §19.8 estimates 8 vCPU and 16 GB in total as an open point. Nothing says how one artifact maps to nine Deployments or how operators are installed.
- **Decision (proposed):** `deploy/helm/billing` with one `Deployment` template rendered from `.Values.roles.<role>`. `deploy/helm/platform` installs CloudNativePG, Strimzi, the ClickHouse operator, cert-manager, KEDA and observability through `helmfile` in `deploy/k3d`. Starting resources (mini, planned):

  | Component | Memory |
  | --- | --- |
  | `billing`, each of 9 roles | 512 MiB |
  | `gateway`, `portal` | 256 MiB, 32 MiB |
  | PostgreSQL, 9 PgBouncer pods | 1 GiB, 16 MiB each |
  | Kafka, Kafka Connect | 1.5 GiB, 768 MiB |
  | ClickHouse, Keeper | 2 GiB, 256 MiB |
  | MinIO, Keycloak | 256 MiB, 768 MiB |
  | Prometheus, Grafana, operators | 1 GiB, 512 MiB |
  | Demo Client (P9) | 384 MiB |

  Total about 14 GiB. S-06 measures this on 8 vCPU and 16 GB and records real numbers; EXP-17 keeps them honest. JVM: ZGC off on mini, `-XX:MaxRAMPercentage=70`. Production uses requests equal to limits on the money path (original SDD §19.1), with sizes from EXP-03.
- **Write to:** DOC-10, DOC-73, DOC-74, DOC-75, ADR-0018.

### DR-97 · What is the retention of every table, topic and bucket?
- **Problem:** The original SDD gives retention for some objects (§8.2, §8.3, §11.6, §15.2, §16.1), but none for `reservation`, `charge`, `metering` or `reconciliation`. `reservation` receives about 8.6 million rows per day and cannot tier by time, because its unique key forces a shard partition (DR-13).
- **Decision (proposed):**

  | Object | Hot | After |
  | --- | --- | --- |
  | `journal` | 10 years in place | Delete after legal review |
  | `posting`, `wallet_movement`, `charge` | 24 months | Tablespace `cold`, stays queryable, to 10 years |
  | `reservation` | 13 months after terminal state | Moved to `reservation_archive` (monthly), to 10 years; `client_reference` uniqueness then ends, while the journal key still guards money |
  | `idempotency_key` | 7 days | Deleted |
  | `outbox` | 30 days | Partition dropped |
  | `delivery`, `attempt`, `event_log` | 90 days | Partition dropped |
  | `audit.log` | 24 months | Tablespace `cold`, to 10 years |
  | `usage_rejection` | 30 days | Deleted |
  | `usage_quarantine`, `reconciliation.*`, `invoice*`, `payment*` | 10 years | In place |
  | ClickHouse `usage.event`, rollups | 13 months, 25 months | TTL |
  | Kafka topics, DLQs | Per DR-40 and original SDD §8.3 | Deleted |
  | Buckets `billing-invoices`, `billing-digests`, `billing-exports` | 10 years, object lock on the first two | |

  Tablespaces use a cheaper storage class; CloudNativePG tablespaces are configured from P8.
- **Write to:** DOC-20, DOC-37.

### DR-98 · Which personal data exists, and how is it handled?
- **Problem:** Original SDD §19.5 limits event data, but there is no inventory of personal data. `audit.log` keeps IP addresses for 10 years, and tenants may leave.
- **Decision (proposed):**

  | Data | Where | Treatment |
  | --- | --- | --- |
  | Name, email, login | Keycloak | Not copied to Billing except `sub` |
  | Legal name, tax code, address, billing email | `tenant_billing_profile`, invoice PDF | RLS; the PDF is kept for retention |
  | IP address | `audit.log` | Truncated to /24 and /48 after 90 days |
  | PSP callback body | `psp_event` | Card data never present; 13 months |
  | Logs | Loki | No bodies, tokens or names (original SDD §19.2) |

  Tenant offboarding: no delete flow in this phase; a Finance Admin blanks the profile, and financial rows stay for legal retention. Vietnam's personal data decree (Decree 13/2023/ND-CP) is checked with counsel before go-live.
- **Write to:** DOC-20, DOC-41.

---

## R. Testing, demo and process

### DR-99 · Which work must move between stages for the plan to run? — ⚠ deviates from the original SDD
- **Problem:** The stage table in original SDD §21 has ordering conflicts.
  - S1 includes expiry, which goes through `billing.jobs.v1` (§7.5), while Kafka arrives at S2.
  - S1 runs EXP-05 killing `worker`, which has nothing to consume without Kafka.
  - S0 claims EXP-17, whose smoke suite covers reserve, commit and webhook delivery, which do not exist until S1 and S2.
  - S1 needs accounts and funding to reserve against, but platform admin endpoints arrive at S2, and top-ups at S5.
  - Audit starts at S7, although FR-27 covers changes made from S1, and a log started late cannot be backfilled.
  - S1 runs EXP-02, which §19.8 says runs at production size, on an environment that S7 creates.
- **Decision (proposed):**

  | Move | From → to | Reason |
  | --- | --- | --- |
  | `billing.jobs.v1` topic and job consumer | S2 → P2 | Expiry needs it; Kafka is already in the mini profile from P1 |
  | Outbox table and writes | S2 → P2; relay stays at P3 | Reservations must write events from their first commit |
  | Platform admin endpoints and manual adjustment | S2, S5 → P2 | Funding for EXP-01 and later tests |
  | Audit: writes from P2; consumer and log at P3; digests stay at P8 | S7 → P2, P3 | Cannot be backfilled |
  | EXP-17 staged: `mini-up` only at P1; reserve and commit at P2; webhook at P3 | S0 → P1–P3 | Each stage smoke-tests what exists |
  | EXP-02 at P2: on a rented performance machine or cluster, marked "provisional"; final at P8 | S1 → P2, P8 | Early warning for the lock, and honest numbers later |
  | EXP-12 reruns whenever a phase adds tenant tables | S1 → P2…P7 | Coverage grows with the schema |
  | `FixedPriceEngine` behind `PricingApi` until P4 | Implied | Keeps P2 independent |

- **Write to:** DOC-81, master plan §1.3.

### DR-100 · What do the load numbers mean, and how are they measured? — ⚠ deviates from the original SDD
- **Problem:** NFR-02 says "500 reservations per second", and original SDD §2.1 derives 8.6 million per day and 17 million postings per day, so a reservation means a hold **and** its settlement. Yet §19.7 sizes "about 5 active" connections at 500 per second, implying 500 transactions. The default Client limit of 500 calls per second (§12.4) and the 2,000 events per second per Client would throttle EXP-03 and EXP-08 run from one Client. There is no statement of where latency is measured or how many runs count.
- **Decision (proposed):**
  - 500 reservations per second means 500 holds and 500 settlements per second, 1,000 write transactions, with a mix of 90% commit, 7% release and 3% expiry.
  - Latency is measured at the Gatling injector, in the same region, through the gateway.
  - Runs use a Client with explicit `rate_limits` overrides, documented in the protocol.
  - Each experiment runs 3 times from a clean state; the median is reported, with the worst noted.
  - Environment recorded in each result: node sizes, versions, git SHA.
  - Numbers in documents say **planned** or **measured**.
  - `pool-api` size is reviewed against EXP-03, with planned active connections near 8 rather than 5.
- **Write to:** DOC-10, DOC-81.

### DR-101 · Where do the baselines and sabotage cases live?
- **Problem:** Original SDD §20.2 compares against baselines the production code must not contain: read-check-write (EXP-01), guard disabled (EXP-04), polling relay (EXP-07), no per-endpoint limit (EXP-11). EXP-15 injects "one violation per rule on a test branch". §11.7 forbids test branches in product code.
- **Decision (proposed):** Gradle source set `experiment` in `billing`, depending on `main` and never packaged: alternative adapters for each baseline, selected by Spring profile in the experiment image only. A build test asserts that the production image has no `experiment` classes. EXP-15 keeps one patch per rule in `billing/src/experiment/violations/*.patch`; a CI job applies each to a clean checkout and asserts that `./gradlew check` fails.
- **Write to:** DOC-80, DOC-81.

### DR-102 · What are the remaining Demo Client and runner details?
- **Problem:** Original SDD §11.8 lets the runner run invariant checks with read-only database access while having "no dependency on `billing`", so the SQL needs a neutral home. Fixture format and staging self-reach (§22.2) are open.
- **Decision (proposed):**
  - Check SQL in `deploy/sql/invariants/*.sql`: copied into the `billing` jar at build, and read directly by the runner.
  - Runner access: as a Kubernetes Job in the cluster using `billing_readonly`, on mini and staging.
  - Fixtures: YAML under `demo-client/conformance/fixtures/`, one per cell, with the hand-calculated values as literal numbers or the named `prorate` formula.
  - Demo tenant's database migrations: Flyway in its own project.
  - The staging Demo Client has its own DNS name under the staging ingress. If the provider cannot hairpin, a split-horizon DNS entry points to the ingress service, which still passes SSRF checks only if that address is public; otherwise a new DR.
- **Write to:** DOC-39, DOC-40, DOC-74.

### DR-103 · What process rules apply?
- **Problem:** The original SDD does not cover commit style, repository visibility or who approves documents.
- **Decision (proposed):**
  - Languages per the decision log.
  - Conventional Commits.
  - Private repository.
  - PR template linking the task, DR and DOC.
  - The Owner approves documents, except delegated small questions recorded as `Claude (delegated by Owner)`.
  - Flyway naming as in DR-16.
  - Configuration keys under `billing.<module>.<name>`, with environment variables in upper-case with underscores.
- **Write to:** Master plan §7, DOC-43, DOC-76.

---

## S. Access by pricing mode and permissions

Added on 2026-11-07 from three Owner requirements (decision log): A, entitlement by pricing mode with expiry and per-user modes; B, permission bitmask; C, two separate spaces. The original SDD has three fixed roles (Client, Customer, Finance Admin) and no notion of users who belong to a Client, of entitlement to a pricing mode, or of fine-grained rights.

Vocabulary: a **user** is one person who signs in to operate the system, and everything they may do is a set of permission bits; there are no roles. The system has two **spaces** with separate use cases: the **platform** (the operator of Billing; its users are **platform users**) and each **tenant** (a business that uses Billing; its users are **tenant_users**). The spaces share code but not identities, catalogs, endpoints, tables or dashboards. **tenant_customers** (a tenant's customers who top up and view invoices) and machine **tenant_system** credentials live inside a tenant and keep their fixed access (DR-66). A **pricing mode** is one of the six modes of original SDD §9.1. Identifier mapping between the old and new names is in DR-112.

### DR-104 · How is a plan tied to a pricing mode? — ⚠ deviates from the original SDD
- **Problem:** Original SDD §9.1 and §11.8 say the six pricing modes are "one model" and "not an enum", so no row says which mode a plan belongs to. Entitlement by mode (DR-105) needs something to check against.
- **Options:** (1) Infer the mode from the components: the modes overlap (a GPU time meter is both time-based and workload-based), so inference is ambiguous. (2) A label on the product: a product such as `platform.pro` can have a usage plan and a hybrid plan. (3) A declared label on the plan, with structural checks at publish.
- **Decision (proposed):** Option 3.
  - `pricing.price_plan.pricing_mode`: `TIME_BASED`, `USAGE_BASED`, `WORKLOAD_BASED`, `EVENT_BASED`, `SUBSCRIPTION`, `HYBRID`. Set at creation and **immutable**; a different mode is a new plan. Not nullable.
  - Mode bits for masks (DR-106): `TIME_BASED` 0, `USAGE_BASED` 1, `WORKLOAD_BASED` 2, `EVENT_BASED` 3, `SUBSCRIPTION` 4, `HYBRID` 5. The order is part of the contract and never changes.
  - Publishing a version checks necessary conditions, so a mislabel cannot pass silently:

    | Mode | Publish check |
    | --- | --- |
    | `SUBSCRIPTION` | Every component is `FLAT` with a `cycle` |
    | `HYBRID` | At least one `FLAT` component with a `cycle` **and** at least one metered component |
    | `TIME_BASED` | No `FLAT`-with-cycle component; every metered component is on a meter whose unit is seconds and has `billing_increment` |
    | `USAGE_BASED` | No `FLAT`-with-cycle component; every metered component is on a `SUM` meter |
    | `WORKLOAD_BASED` | No `FLAT`-with-cycle component; at least one component has a non-empty `dimension_filter` |
    | `EVENT_BASED` | No `FLAT`-with-cycle component; every metered component is on a `COUNT` meter |

    A failure is `422 pricing_mode_mismatch` naming the component.
  - Rating never reads `pricing_mode`: it changes who may create or publish plans, not how money is computed. The one-model rule of §9.1 holds.
  - EXP-18's enum-coverage test (original SDD §11.8) now also reads `pricing_mode` from `api/openapi` and requires a cell for each value. This replaces the "not an enum" remark and makes the six-mode check mechanical.
- **Consequences:** Existing plans in a seed or migration must be labeled; the Demo tenant seeds one label per cell. A hybrid plan consumes only the `HYBRID` entitlement, not `SUBSCRIPTION` and `USAGE_BASED`.
- **Write to:** DOC-15, DOC-27, DOC-40, DOC-53.

### DR-105 · How does a tenant_system get a pricing mode, and what does expiry do?
- **Problem:** Requirement A: a Finance Admin permits each tenant_system certain pricing modes. Later withdrawal must not delete or break what the tenant already created. Billing also publishes only facts and never stops a resource (FR-30), and open reservations must still settle (NFR-01).
- **Decision (proposed):**
  - Table `platform.client_pricing_mode` (no `tenant_id`, no money; Ops DB):
    ```sql
    id uuid PRIMARY KEY, client_id uuid NOT NULL, pricing_mode text NOT NULL,
    starts_at timestamptz NOT NULL DEFAULT now(), expires_at timestamptz,   -- NULL: no end
    granted_by uuid NOT NULL, expired_by uuid, expire_reason text,
    CHECK (pricing_mode IN ('TIME_BASED','USAGE_BASED','WORKLOAD_BASED','EVENT_BASED','SUBSCRIPTION','HYBRID')),
    EXCLUDE USING gist (client_id WITH =, pricing_mode WITH =, tstzrange(starts_at, expires_at) WITH &&)
    ```
    A mode is **active** at time `t` when `starts_at <= t AND (expires_at IS NULL OR expires_at > t)`. A trigger lets only `expires_at`, `expired_by` and `expire_reason` change after insert; a row is never deleted.
  - **Revoking is expiring.** `POST /v1/admin/client-systems/{id}/pricing-modes/{mode}/expire` sets `expires_at = now()` (or a future time, to schedule) and needs a `reason`. A later grant of the same mode inserts a new row, so history stays.
  - `POST /v1/admin/client-systems` takes an optional `pricing_modes` list (default empty: the tenant can sell nothing until granted). The seed job grants all six to `demo-rental`.
  - **What expiry blocks and what it leaves alone.** A mode that is not active stops *extending* the tenant's footprint in that mode and nothing else:

    | Blocked while the mode is not active (`403 pricing_mode_not_granted`) | Never blocked |
    | --- | --- |
    | Create a plan of the mode; create or publish a version of such a plan | Estimate, reserve, extend, commit, release, expiry and late commit on an existing plan |
    | Create a price assignment to such a plan; set it as a product's default plan | Usage ingestion, rating, fees of running subscriptions, cycle close, invoices, payments, refunds |
    | `POST /v1/subscriptions` on such a plan, and `PATCH` to such a plan | Subscription `DELETE` and `PATCH` to a plan of an active mode |
    | | Reads; editing a `DRAFT`; deleting a `DRAFT`; deleting an assignment |

    So every plan, version, product, meter, subscription, reservation, charge and invoice created earlier stays exactly as it was, and a running service keeps being billed. Winding down is never blocked.
  - The check is **never cached**: one indexed read on the Ops DB primary per blocked-type call. These calls are rare, and an expired mode is refused on the next call, not within 30 s.
  - Invariant: after any change, every `PUBLISHED` version of a tenant is on a plan whose mode was active when the version was published. A build test cannot prove this over time, so the publish handler checks it and a nightly job counts published versions whose mode had no active row at `published_at` (alert on any).
  - Notification: audit row only. A webhook event for expiry is Later; events are scoped by account (DR-38) and a mode belongs to a tenant.
- **Consequences:** The tenant's runtime never sees a failure from expiry. A tenant who wants to stop *selling* in a mode has no more than this mark; ending already-running services is the tenant's own decision (FR-30). The open question for the Owner: should new reservations on an existing plan also be refused after expiry? Proposed answer is no, for the reasons above.
- **Write to:** DOC-15, DOC-27, DOC-33, DOC-41, DOC-51 (FL-34), DOC-53, DOC-83.

### DR-106 · How are permissions modeled, and how is the bitmask laid out? — ⚠ deviates from the original SDD
- **Problem:** Requirement B: rights are given by group, function and action, as a permission set implemented with a binary mask. Original SDD §17 and §12 have role checks only (`finance_admin`), and DR-66 is a role × endpoint table. A role name cannot express "may publish plans but not delete them". Requirement C (DR-110, DR-111) adds that the platform and a tenant are two **separate spaces** with separate use cases, so their permissions must not share one list.
- **Options:** (1) Roles with a hard-coded endpoint list: what exists now, too coarse. (2) A permission table with one row per (user, permission): readable, but each check is a join and a set of rows to cache. (3) One bit per permission in a fixed-width mask on the user: a check is one AND.
- **Decision (proposed):** Option 3, with **two catalogs**.
  - **Three levels.** A **permission** is `<group>.<function>.<action>`, such as `pricing.price_plan.publish`. Each catalog lives in code, not in a table.
  - **Two catalogs, no shared bits.** `PlatformPermission` (held by platform users) and `ClientPermission` (held by tenant_users) are two enums with two bit spaces, two lock files and two columns. A platform user can never hold a tenant bit, and the reverse, because the types differ: there is no `BOTH` and no scope mask to forget.
  - **Bit layout.** Per catalog, a global index, one bit per permission, **assigned once and never reused or renumbered**. Width is fixed at **256 bits**; version 1 uses about 59 (platform) and 22 (client), planned. Storage: PostgreSQL `bit(256)`, Java `long[4]`. Bit `i` is `set_bit(mask, i, 1)` / `get_bit(mask, i)` in SQL and bit `i` of the long array in Java; a conversion test proves both agree.
  - **Lock files.** `platform-permission-catalog.lock` and `client-permission-catalog.lock` (code → bit, `ACTIVE` or `RETIRED`) are committed. A build test fails when a code's bit changes, a bit is reused, a code disappears, or a new code is missing from its file. A retired permission keeps its bit forever.
  - **Platform catalog (planned; final table with bit numbers in DOC-83):**

    | Group | Function: actions |
    | --- | --- |
    | `tenancy` | `tenant`: read, create, update · `account`: read, create, set_mode, set_limit · `client_system`: read, create, update, rotate_secret · `client_grant`: read, create, revoke |
    | `entitlement` | `pricing_mode`: read, grant, expire |
    | `finance` | `refund`: read, approve, reject · `adjustment`: read, create, approve, reject · `reversal`: create · `reconciliation`: read, resolve · `usage_quarantine`: read, resolve · `rating_cursor`: resume · `invoice`: read, void |
    | `ops` | `webhook_delivery`: read, redeliver · `overview`: read |
    | `client_support` | On a **named tenant's** catalog: `price_plan`: read, create, update, delete, publish · `price_assignment`: read, create, delete · `product`: read, create, update · `meter`: read, create, update |
    | `users` | `platform_user`: read, create, set_permissions, deactivate · `client_user`: read, create, set_permissions, set_modes, deactivate · `client_resource`: reassign |

  - **tenant catalog (planned):**

    | Group | Function: actions |
    | --- | --- |
    | `pricing` | `price_plan`: read, create, update, delete, publish · `price_assignment`: read, create, delete |
    | `catalog` | `product`: read, create, update · `meter`: read, create, update |
    | `access` | `user`: read, create, set_permissions, set_modes, deactivate · `resource`: reassign, manage_any |
    | `profile` | `client`: read (the tenant's own profile and its pricing modes with expiry) |

    On both sides, `create` on `price_plan` covers a new plan and a new version of an existing plan; `update` is `PATCH` of a `DRAFT` version only; `delete` is deleting a `DRAFT` version (DR-87). Published versions are never edited or deleted.
  - **Extending actions** are the ones DR-105 blocks when the mode is not active: `price_plan.create`, `price_plan.publish`, `price_assignment.create`, setting a product's `default_plan_id`, on either side, and subscription create or change-to (tenant_system, no user bit). Only these consult the mode.
  - **Wire form.** A request names permissions as codes, with wildcards at group or function level: `["pricing.price_plan.*", "catalog.*", "access.user.read"]`. The server expands wildcards into bits **at that moment** and stores only bits, so a permission added later is not silently granted by an old `catalog.*`. A response shows the code list and the mask in hex.
  - **No roles and no presets.** There is no `admin` or `staff` kind of user and no named bundle of rights. A user is exactly its permission bits (plus its modes, DR-107). Older entries that say "Finance Admin" mean *a platform user holding the permission for that action*, not a role.
  - **Evaluation.** One function, `PermissionService.require(actor, permission, @Nullable resource)`; no controller or handler tests a role name or a user kind. The caller's **space** is fixed first, by the token's issuer (DR-107, DR-110), and a platform token on a tenant endpoint, or the reverse, is `403 wrong_space` before any bit is read. Then, for user `U` and permission `p` of U's own catalog:
    ```
    allowed(U, p, R) =
          U.status = ACTIVE
      AND get_bit(U.permissions, bit(p)) = 1
      AND ( p is not an extending action
            OR modeBit(R.plan.pricing_mode) ∈ effective_modes(U, R.client) )   -- DR-105, DR-107
      AND ( U is a platform user OR R is null
            OR ( R.client_id = U.client_id
                 AND ( R.managed_by = U.id
                       OR get_bit(U.permissions, bit(access.resource.manage_any)) = 1 ) ) )  -- DR-108
    ```
    A denial is `403 permission_denied` with `required` set to the code, so a developer sees which bit is missing. A mode denial is `403 pricing_mode_not_granted`; a manager denial is `403 not_resource_manager`.
  - **No escalation.** Inside one space, a user may set on another user only bits that the setter holds (`403 grant_exceeds_own_permissions`), and never edits their own mask (`403 cannot_edit_self`). Across spaces the platform is the root of trust: a platform user holding `users.client_user.set_permissions` may set any tenant bit on a tenant_user, and one holding `users.platform_user.set_permissions` is limited to their own platform bits. A tenant_user can never reach a platform bit.
  - **Machine tenants and tenant_customers** keep their fixed access (DR-66, DR-38) and are not in the bitmask system.
  - **Audit.** Every change of a mask writes `audit.log` with before and after as lists of codes (DR-83).
- **Consequences:** Adding a capability is one enum line, one lock-file line and one test row. A search by bit answers "who can approve refunds". Dual control (DR-69) is independent: approver ≠ requester holds even when both hold the bit.
- **Write to:** DOC-26, DOC-41, DOC-53, DOC-83, ADR-0026.

### DR-107 · How are users stored and identified, and how can the same email exist in several spaces?
- **Problem:** Requirement C: platform users and tenant_users are separate populations with separate use cases, and the same email may belong to a platform user and to users of several tenants. One table with a nullable `client_id` and a globally unique email cannot give that. Keycloak by default also allows one account per email per realm.
- **Options:** (1) One realm, unique emails: the same person cannot work for two tenants. (2) One realm with duplicate emails allowed and a synthetic username `<email>@<client>`: works, but sign-in by email is lost and the login page cannot be per space. (3) One realm per space: the same email is a different account in each realm, with a login page per space.
- **Decision (proposed):** Option 3, with separate tables.
  - **Realms.** `billing-platform` for platform users, `billing-client-<code>` for each tenant, and the existing `billing` realm for tenant_customers, tenant_systems and `demo_operator` (DR-64). A tenant's realm is created in the tenant-creation saga (FL-25) by a service account `billing-realm-provisioner` that holds only realm-management rights. At most about 20 tenants (original SDD §2.1) gives at most 22 realms. Fallback if realms prove heavy in operation: option 2 with a new DR.
  - **Tables** (Ops DB, no `tenant_id`, so outside the RLS rule of DR-63; two more rows in DOC-19's matrix):
    ```sql
    -- platform.platform_user
    id uuid PRIMARY KEY, idp_subject text NOT NULL UNIQUE, email text NOT NULL,
    display_name text NOT NULL, permissions bit(256) NOT NULL,
    status text NOT NULL DEFAULT 'ACTIVE',                 -- ACTIVE | ENDED
    created_by uuid, created_at timestamptz NOT NULL, ended_at timestamptz, ended_by uuid
    CREATE UNIQUE INDEX ON platform.platform_user (lower(email)) WHERE status = 'ACTIVE';

    -- platform.client_user
    id uuid PRIMARY KEY, client_id uuid NOT NULL, idp_subject text NOT NULL UNIQUE,
    email text NOT NULL, display_name text NOT NULL,
    permissions bit(256) NOT NULL, mode_mask smallint NOT NULL DEFAULT 0,
    status text NOT NULL DEFAULT 'ACTIVE',
    created_by uuid, created_at timestamptz NOT NULL, ended_at timestamptz, ended_by uuid
    CREATE UNIQUE INDEX ON platform.client_user (client_id, lower(email)) WHERE status = 'ACTIVE';
    ```
  - **Email is unique only inside one space and only among `ACTIVE` rows.** The same email may exist once in the platform and once in each tenant; an `ENDED` user's email can be reused by a new user in the same space. Email is a sign-in and display attribute: authorization, audit and `managed_by` use the user `id`, never the email, so reuse can never transfer rights or history.
  - **Token to user.** Each realm has its own issuer. The gateway keeps an `issuer → space` map (platform realm; each `client_system.idp_realm`) and a JWKS cache per issuer (1 hour, last good keys kept, as DR-64). The user row is looked up by `idp_subject` in the table of the issuer's space. A token whose issuer is unknown, or whose `sub` has no `ACTIVE` row, gets `403`.
  - **Claims.** Platform tokens carry `billing_space=platform`. tenant tokens carry `billing_space=client` and `billing_client_id`, a hard-coded mapper in each tenant realm, which must equal the `client_id` of the realm's issuer and of the user row. Any mismatch is `403`. Realm role `billing_user` only opens the door; no code reads a role for rights. `finance_admin` of DR-64 is removed.
  - **Modes per tenant_user.** `mode_mask` holds up to six bits (DR-104 order) and means "the modes this person may work in". A setter may set only bits inside the **tenant's currently active modes** and inside the setter's own effective modes (`422 mode_not_granted_to_client`; `403 grant_exceeds_own_permissions`). A platform user has no `mode_mask`; acting on a tenant's catalog is limited by that tenant's active modes alone (DR-105).
  - **Effective modes** `effective_modes(U, client) = active_client_modes(client) AND U.mode_mask` for a tenant_user; for a platform user, `active_client_modes(client)`. When a tenant's mode expires, `mode_mask` stays as set, so a re-grant restores each person's setting.
  - **Two independent gates** for a mode-bound action: the function bit says *what* the person may do; the effective mode says *in which pricing mode*. A user with one and not the other is refused.
  - **Creating users.** A platform user holding `users.platform_user.create` creates platform users; one holding `users.client_user.create` creates users of any tenant, including the first. A tenant_user holding `access.user.create` creates users of their own tenant only. The Keycloak account is created in the right realm by the provisioner, with OTP: required for every platform user and for any tenant_user holding an `access.*` bit; no self-signup. The first platform user comes from a one-time `billing.role=bootstrap` command that sets every platform bit on a named `idp_subject` and refuses to run once any user holds `users.platform_user.set_permissions`; mini's realm import and seed job use it.
  - **Reads may be cached 30 s; writes and revocation checks never are.** Every mutating call, and every check that decides a revocation, reads the row from the primary (DR-108).
  - There is no `role` column anywhere.
- **Consequences:** A person who works for two tenants has two separate accounts with the same email, signing in at each tenant's own page. The cost is one realm per tenant and a provisioning saga step. The last holder of `access.user.set_permissions` in a tenant can be deactivated; a platform user holding `users.client_user.set_permissions` repairs it.
- **Write to:** DOC-15, DOC-19, DOC-26, DOC-41, DOC-51 (FL-25, FL-35), DOC-53, DOC-83, ADR-0023, ADR-0026.

### DR-108 · What does revoking a user do, and who owns the resources they made?
- **Problem:** Requirement A: revoking a **tenant** never deletes what it created (DR-105). Revoking a **user** can happen at once, because resources can always be given to another user. That needs a definition of "manages", of what a revoke leaves behind, and of how resources are found and handed over.
- **Decision (proposed):**
  - **Owner and manager are different things.** Each resource belongs to its tenant (`client_id`, never changes). A **manager** is the tenant_user answerable for it day to day: nullable `managed_by uuid` (a `client_user.id`) on `pricing.price_plan`, `platform.client_product` and `meter`. A price assignment follows its plan's manager. The creator becomes the manager when a tenant_user creates it; resources made by a platform user, the seed job or a tenant_system have `managed_by = NULL`. The creator also appears in `audit.log`, which `managed_by` never overwrites.
  - **Who may change a resource.** A tenant_user: the function bit, the effective mode when the action is extending, and (`managed_by = U.id` **or** `access.resource.manage_any`). A platform user: the `client_support` bit and the tenant's active mode; no manager rule. Reading needs only the `read` bit.
  - **Revoking a user:** `POST /v1/console/users/{id}/deactivate` (or `/v1/admin/client-users/{id}/deactivate`, `/v1/admin/platform-users/{id}/deactivate`) sets `status = ENDED`, `ended_at = now()`. It takes effect **on the next call**: every check reads the primary, and the access token (5 minutes) proves identity only. The row is kept for audit and never deleted. A tenant_user's deactivation may carry `reassign_to`, which moves every resource they manage in one transaction.
  - **Narrowing** (`PUT …/permissions`, `PUT …/pricing-modes`) is equally immediate and changes no resource. A person who loses a bit or a mode keeps their `managed_by` links but cannot act through them.
  - **Resources needing a manager** have `managed_by IS NULL`, or an `ENDED` manager, or a manager who no longer holds the `update` bit or the resource's mode. `GET /v1/console/resources?needs_manager=true` lists them; any tenant_user with `access.resource.manage_any` can edit them meanwhile, so nothing is ever stuck.
  - **Reassigning:** `POST /v1/console/resources/reassign` with `{ "from": "<user>" | null, "resources": [ids] | "all", "to": "<user>" }`, needing `access.resource.reassign`; platform users use `POST /v1/admin/client-systems/{id}/resources/reassign` with `users.client_resource.reassign`. The target must be an `ACTIVE` user of the same tenant, hold the `update` bit of each function, and have the mode in `effective_modes` for mode-bound resources, else `422 assignee_lacks_permission` naming the missing item. One audit row per call lists the ids.
  - **Resources keep working throughout.** A plan with no active manager still rates, estimates and bills; only its editing waits for a manager. Revoking a user changes no price, plan version, subscription, reservation, charge or invoice.
  - **What a tenant-level expiry does to users:** nothing to their rows (DR-107). Their managed resources stay linked.
- **Consequences:** The two revocations differ on purpose. A tenant's mode is a commercial entitlement over an organization's footprint: it only stops new growth. A user is an identity: it is cut at once, and continuity of the resource is solved by handing it over.
- **Write to:** DOC-15, DOC-26, DOC-27, DOC-51 (FL-36), DOC-53, DOC-83.

### DR-109 · What are the endpoints, and what changes in the endpoint × role matrix?
- **Problem:** DR-66 and DR-87 know only tenant, tenant_customer and Finance Admin. The new capabilities need endpoints for grants, users, permissions, reassignment and the catalogs, and every `/v1/admin/*` and `/v1/console/*` endpoint needs a permission code. Platform and tenant use cases must stay separate (Requirement C).
- **Decision (proposed):**
  - **Two endpoint families, one per space.** `/v1/admin/*` is for platform users only and `/v1/console/*` for tenant_users only. The door is the token's space (DR-107): a platform token on `/v1/console/*`, or a tenant token on `/v1/admin/*`, is `403 wrong_space`. A platform user who works on one tenant's catalog or people uses `/v1/admin/client-systems/{id}/…`, never `/v1/console/*`. Matrix (P = platform user, K = tenant_user; C and U as in DR-66):

    | Endpoints | C | U | P | K |
    | --- | --- | --- | --- | --- |
    | `/v1/admin/*` | no | no | with the bit | no |
    | `/v1/console/*` | no | no | no | with the bit, own tenant only |
    | `/v1/me` | no | no | read | read |
    | `/v1/permissions/catalog` | no | no | platform catalog | tenant catalog |
    | Everything else | unchanged | unchanged | unchanged | **no** |

    A tenant_user has no access to accounts, wallets, charges, invoices or tenant data: console endpoints work on the tenant's catalog, people and profile only, so a tenant_user needs no grant to a tenant and no RLS context.
  - **New endpoints** (added to DR-87):

    | Space | Endpoint | Permission |
    | --- | --- | --- |
    | Platform | `GET /v1/admin/client-systems/{id}/pricing-modes` | `entitlement.pricing_mode.read` |
    | Platform | `POST /v1/admin/client-systems/{id}/pricing-modes` (grant) | `entitlement.pricing_mode.grant` |
    | Platform | `POST /v1/admin/client-systems/{id}/pricing-modes/{mode}/expire` | `entitlement.pricing_mode.expire` |
    | Platform | `GET/POST /v1/admin/platform-users`, `GET/PUT …/permissions`, `POST …/deactivate` | `users.platform_user.*` |
    | Platform | `GET/POST /v1/admin/client-users` (filter `client_id`), `GET /v1/admin/client-users/{id}`, `PUT …/permissions`, `PUT …/pricing-modes`, `POST …/deactivate` | `users.client_user.*` |
    | Platform | `POST /v1/admin/client-systems/{id}/resources/reassign` | `users.client_resource.reassign` |
    | Platform | `/v1/admin/client-systems/{id}/price-plans…`, `price-assignments…`, `products…`, `meters…` (the `/v1/admin/price-plans` of DR-87 with the tenant in the path) | `client_support.*` |
    | tenant | `GET /v1/console/profile` (own tenant, modes with expiry) | `profile.client.read` |
    | tenant | `GET/POST /v1/console/users`, `GET /v1/console/users/{id}`, `PUT …/permissions`, `PUT …/pricing-modes`, `POST …/deactivate` | `access.user.*` |
    | tenant | `GET /v1/console/resources?managed_by&needs_manager&type`, `POST /v1/console/resources/reassign` | `access.resource.*` |
    | tenant | `/v1/console/price-plans…`, `price-assignments…` | `pricing.*` |
    | tenant | `/v1/console/products…`, `meters…` | `catalog.*` |
    | Both | `GET /v1/me` (own permissions as codes, effective modes, status, space) | any user |
    | Both | `GET /v1/permissions/catalog` (groups, functions, actions, bits) of the caller's catalog | any user |

    The two families call **one handler** for plans, assignments, products and meters; only the actor, the tenant filter and the permission code differ, so there is no second pricing code path. The two families share code and nothing else: separate paths, tokens, permissions, tables and dashboards.
  - **New error codes:** `403 wrong_space`, `403 permission_denied`, `403 pricing_mode_not_granted`, `403 not_resource_manager`, `403 grant_exceeds_own_permissions`, `403 cannot_edit_self`, `422 pricing_mode_mismatch`, `422 mode_not_granted_to_client`, `422 assignee_lacks_permission`, `409 email_in_use` (same email already `ACTIVE` in this space).
  - **Authorization test matrix.** One generated test per (endpoint × permission bit): bit set passes, bit cleared returns `403 permission_denied`. A second generated set per extending action × six modes × (tenant mode active / expired) × (user mode set / cleared). A third: every space-crossing combination (platform token on a tenant endpoint and the reverse, tenant 1's token on tenant 2's data) is refused. A build test requires that every `/v1/admin/*` and `/v1/console/*` operation in `api/openapi` carries an `x-permission` code that exists in the matching catalog.
  - **Rate limit.** Console calls are keyed by `billing_client_id` at the gateway, with the tenant_customer limit of DR-67; admin calls by `sub`.
- **Write to:** DOC-41, DOC-52, DOC-53, DOC-70, DOC-83.

### DR-110 · How are the platform and each tenant addressed: by host, by path, or both?
- **Problem:** Requirement C: a tenant's dashboard lives at `client1.<domain>` or `<domain>/client1`, supporting both if possible, and the platform at `platform.<domain>` or `<domain>/platform`. The original SDD has one portal and no addressing by space.
- **Decision (proposed):**
  - **Spaces and their addresses** (`{base}` is the configured domain):

    | Space | By host | By path |
    | --- | --- | --- |
    | Platform | `platform.{base}` | `{base}/platform` |
    | tenant `code` | `{code}.{base}` | `{base}/{code}` |
    | tenant-customer portal (proposed, for the Owner to confirm) | `portal.{base}` | `{base}/portal` |

  - **Both forms are supported, chosen by configuration:** `billing.routing.mode` = `HOST`, `PATH` or `BOTH`, default `BOTH`. Recommended: `HOST` in staging and production, with `PATH` kept as a fallback when a wildcard DNS or certificate is not available, and `PATH` on mini (a single `localhost` with no wildcard).
  - **One build, both modes.** Assets use relative URLs, and the nginx entrypoint renders `<base href>` and `/config.js` per request from the resolved space, so the router receives its base path at run time (same image everywhere, as DR-89).
  - **URL never grants access.** The API is not per space: it stays at `{api-host}/v1/…`. The caller's space comes from the token's issuer only (DR-107). The URL decides which dashboard bundle is served and which realm's login page opens. A token for tenant 2 used on tenant 1's page is refused at login and signs the user out; the API treats it as tenant 2's token whatever the page says.
  - **tenant `code`.** The existing natural key of `client_system`, now also a URL segment and a realm name. Regex `^[a-z][a-z0-9-]{1,28}[a-z0-9]$`, unique, **immutable**. Reserved and refused: `platform`, `portal`, `api`, `auth`, `admin`, `console`, `www`, `static`, `assets`, `docs`, `status`, `login`, `app`, `billing`. An unknown code returns the same `404` page as an unknown route.
  - **Ingress.** Host rules `platform.{base}`, `portal.{base}`, `*.{base}`; path rules `/platform`, `/portal`, `/{code}`; one wildcard certificate by cert-manager DNS-01 for staging and production.
  - **OIDC.** The dashboard OIDC client in each realm registers the redirect URIs of every enabled form (`https://{code}.{base}/*`, `https://{base}/{code}/*`); the provisioning saga sets them and a job re-registers them when `billing.routing.mode` changes. Silent refresh and `prompt=none` follow DR-89.
  - **CORS** comes from the same table: the gateway allows the exact origin of each enabled form of each space and nothing else.
  - **Isolation caveat.** Path mode serves every space from one browser origin, so a tenant's dashboard and the platform dashboard share storage and same-origin scripts. Tokens are held in memory only (DR-89) and all scripts are ours, so the risk is accepted for `PATH`, and it is why `HOST` is recommended for production. A CSP forbidding inline scripts applies in both.
- **Consequences:** Adding a tenant adds a realm and an OIDC client, and no DNS record under wildcard DNS. A tenant cannot be renamed without a new code and a new realm.
- **Write to:** DOC-38, DOC-43, DOC-56, DOC-72, DOC-73, DOC-74, DOC-75, DOC-84.

### DR-111 · How are the platform, tenant and tenant_customer front-ends separated?
- **Problem:** Requirement C: at least the platform dashboard and the tenant dashboard must be clearly separate. DR-89 and DR-90 describe one portal with role routing, and tenant_users have no screens.
- **Decision (proposed):**
  - **Three front-ends, one repository.** `apps/platform-dashboard`, `apps/tenant-dashboard`, `apps/tenant-customer-portal`, with shared packages `packages/ui` (design system, DR-89), `packages/api-client` (generated, DR-10), `packages/auth` (the `oidc-client-ts` wrapper) and `packages/pricing-editor` (the plan editor used by both dashboards). Code is shared; use cases, navigation, URL maps and bundles are not.
  - **Separate builds and separate sign-in.** Each app has its own Vite entry and bundle, its own OIDC client and, for the tenant dashboard, its own realm per tenant (DR-107). One `portal` image contains the three builds, and nginx serves one by the resolved space (DR-110).
  - **Isolation checked in CI.** A dependency rule fails the build if `tenant-dashboard` or `tenant-customer-portal` imports a platform-only module, or if any of their bundles contains a `/v1/admin` path; and the same for the reverse. The generated client is tree-shaken per app from `x-audience` tags.
  - **Screens.**

    | Front-end | Screens |
    | --- | --- |
    | Platform dashboard | Finance overview, Price plans (any tenant's, through `client_support`), Refund approvals, Reconciliation, Tenants and limits (with quarantine), tenant_systems and grants (now with pricing modes, tenant_users and realm), Platform users and permissions (added), Webhook deliveries |
    | tenant dashboard (added) | Overview (own modes with expiry, user count, resources needing a manager), Users and permissions, Price plans, Products and meters, Resources and reassignment |
    | tenant-customer portal | Account overview, Top up with Payment result, Transactions, Invoices, Refunds |

    This is 18 screens, up from 12 in DR-90. The tenant dashboard shows no tenant, account or money data.
  - **Navigation comes from `GET /v1/me`.** Menu items and buttons are hidden when the permission bit is missing, as a courtesy only; the server decides. A plan editor on an expired mode shows a banner and disables the extending actions (DR-105), and the server still refuses them.
  - **Look.** Each front-end shows its space in the header ("Platform", the tenant's display name, or the customer's tenant), so a user working in two spaces cannot confuse them.
  - **Language:** English only at first, with all strings through i18next (DR-90).
- **Consequences:** tenant_users get a real UI in this phase, which adds five screens to P8. The cut fallback (`make admin-cli`) covers platform screens only; the tenant dashboard is cut last among the admin screens since tenants have no other way to manage their users.
- **Write to:** DOC-11, DOC-56, DOC-57, DOC-59, DOC-84…90.

### DR-112 · What is the unified tenant model (B2B2C)?
- **Problem:** The original SDD names the business a "Client System" and the paying organization a "Customer (Tenant)". "Tenant" names the payer, not the business, and "Client" and "Client System" are used for two different things, so the roles are hard to follow.
- **Decision (proposed):** One hierarchy, with tenants as the isolation unit.

  | Term | Who | Scope | Holds wallet |
  | --- | --- | --- | --- |
  | `platform_user` | Operator of the billing app | Global, cross-tenant | No |
  | `tenant` | A business (e.g. a GPU-rental company). **The isolation unit.** | One tenant | No |
  | `tenant_user` | Staff of the business | Inside its tenant | No |
  | `tenant_customer` | The business's customer (the renter). **Holds billing account + wallet.** | Inside its tenant | Yes |
  | `tenant_system` | The business's service / machine integration | Inside its tenant | No |
  | System (jobs) | Internal automatic jobs | Internal | No |

  Locked sub-decisions: (1) the wallet belongs to `tenant_customer`; the tenant is the merchant. (2) One `tenant_customer` identity per tenant; email is unique per tenant among `ACTIVE` rows. (3) Isolation is by the business `tenant_id`; `customer_id` narrows access to one customer inside a tenant.

  Terminology mapping (old → new):

  | Old | New |
  | --- | --- |
  | Client (organization / space) | `tenant` |
  | Client System (service row, `idp_client_id`, products, meters) | `tenant_system` |
  | Client user | `tenant_user` |
  | Customer (Tenant) / Customer | `tenant_customer` |
  | `client_id` (provider) | `tenant_id` |
  | `tenant_id` (payer) | `customer_id` |
  | `account.tenant_id` (payer) | `account.customer_id` + new `account.tenant_id` |
  | `platform.client_system` | `platform.tenant` |
  | `platform.tenant` (payer org) | `platform.customer` |
  | `platform.client_user` | `platform.tenant_user` |
  | (new) | `platform.tenant_customer` |
  | `platform.client_grant` | `platform.customer_enrollment` (one active per customer) |
  | `tenant_billing_profile` | `customer_billing_profile` |
  | event envelope `client_id` | `tenant_id` (envelope: `tenant_id`, `customer_id`, `account_id`) |
  | `Billing-Tenant-Id` (named the payer) | `Billing-Tenant-Id` (names the business) |
  | realm `billing-client-<code>` | `billing-tenant-<code>` |
  | realm `billing` (customers) | `billing-customer-<code>` (one per tenant) |
  | `ClientPermission` | `TenantPermission` |

  **Client vs Client System.** The old docs use "Client System" for two nested things: the provider organization (space, realm, users) maps to `tenant`; the provider's service row (`client_system`, machine credentials, products, meters) maps to `tenant_system`.

  **The Demo is a whole tenant.** SDD §11.7: the Demo "plays a small rental company named `demo-rental` … as any Client does", and "to Billing it is a row in `client_system` with a grant". So `demo-rental` is a `tenant`, its registration is a `tenant_system`, and the seeded prepaid/postpaid accounts are `tenant_customer`s; it is not merely a `tenant_system`.
- **Consequences:** RLS keys on the business `tenant_id`; every previously payer-scoped table also carries the business `tenant_id`; `billing_account`'s natural key is `(tenant_id, customer_id, code)`; `client_grant` is replaced by `customer_enrollment`; the three front-ends are the platform dashboard, tenant dashboard and tenant-customer portal. Until the prose below is migrated, older entries that say "Client", "Client System", "Customer" or "Tenant" are read through this table.
- **Open points:** grant semantics assume a customer belongs to exactly one tenant; the tenant paying the platform for its own subscription is out of scope; customer-portal hosting is decided in DOC-84; the event-contract change from `client_id` to `tenant_id` must be versioned once live.
- **Write to:** DOC-26, DOC-83, DOC-06.


---

## Summary by impact

| Level | Items | Why decide early |
| --- | --- | --- |
| Blocks P1 | DR-01, 06–18, 62–64, 68, 88, 92, 94–96, 99–101, 103 | Fix the stores, schemas, partitioning, ID format, code layout, roles, pooling, charts and CI. All are expensive to change later, and the ID format is part of the public contract |
| Blocks P2 | DR-19–35, 65–67, 74, 75, 83, 85, 87, 105–107, 109, 110 | Fix the money semantics: templates, movement log, gate, invariant checks, reservation edge states, idempotency, tenancy on ID-addressed calls. Also the permission bitmask, user table and mode grants: every later admin endpoint is written against them, and bit numbers cannot change once data exists |
| Blocks P3 | DR-36–42, 56–61 | Fix the outbox, relay, event visibility, delivery table and SSRF client. Event contracts freeze here |
| Blocks P4 | DR-43–48, 104, 108 | Fix the plan schema, meter names, VOLUME behavior and plan resolution; the plan's pricing mode and resource managers |
| Blocks P5 | DR-49–55 | Fix ingest validation, channel contracts, freezing and ClickHouse operations |
| Blocks P6 | DR-69–73 | Fix the PSP choice and the edges of payment and refund state machines |
| Blocks P7 | DR-03, 76–81 | Fix tax representation, cycle calendar, subscriptions and invoice rules |
| Blocks P8 | DR-02, 05, 82, 84, 86, 89–91, 93, 97, 98, 111 | Fix reconciliation, digests, portal stack, provider, backups and retention |
| Blocks P9 | DR-102 | Fix remaining Demo Client and runner details |
| Blocks go-live only | DR-04 (and the confirmations in DR-01, 02, 05, 70, 91, 98) | Legal and market confirmation from the Owner |
