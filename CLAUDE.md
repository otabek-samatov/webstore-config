# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repository Is

`webstore-config` is the **Git-backed configuration source** for the webstore microservices system.
It is **not** a buildable application — it contains no code, only YAML configuration served at runtime
by the **config-service** (Spring Cloud Config Server, port `8071`, which lives in the main
`webstore` repo). Every other service fetches its configuration from the Config Server on startup.

- **Remote:** https://github.com/otabek-samatov/webstore-config
- **Local clone (this directory):** `C:\Data\Projects\webstore-config`
- **Consumed by:** the config-service, which is pointed at this repo's `config/` directory.

> ⚠️ The Config Server reads from the **Git remote**, not this local working copy. A change only
> takes effect after it is **committed and pushed**. See [Editing workflow](#editing-workflow).

## Repository Layout

```
webstore-config/
├── README.md
├── LICENSE
└── config/
    ├── application.yml          # shared defaults applied to EVERY service
    ├── application-dev.yml      # shared per-profile overlays (DEV / UAT / PROD)
    ├── application-uat.yml
    ├── application-prod.yml
    ├── gateway-service.yml      # per-service overrides (one file per service)
    ├── product-service.yml
    ├── inventory-service.yml
    ├── user-service.yml
    ├── auth-service.yml
    ├── order-service.yml
    ├── order-service-prod.yml   # per-service, per-profile overlay (example)
    └── payment-service.yml
```

All runtime configuration lives under `config/`. There is nothing else to build or run here.

The `-<profile>` files are **environment overlays** (see [Profiles](#profiles)): they are merged only
when a client requests that Spring profile via `SPRING_PROFILES_ACTIVE` (DEV / UAT / PROD). The active
profile is selected in the `webstore` repo's per-environment `.env` files (`SPRING_PROFILE`), not here.

## How Configuration Resolution Works

Spring Cloud Config matches each requesting service's `spring.application.name` to a
`<name>.yml` file in `config/`, then **merges it on top of `application.yml`** (the shared
defaults). When the client sends an active profile (`SPRING_PROFILES_ACTIVE`), the matching
`application-<profile>.yml` and `<service>-<profile>.yml` overlays are merged too.

Property precedence, low→high:
`application.yml` → `application-<profile>.yml` → `<service>.yml` → `<service>-<profile>.yml`
(a more specific / more profile-specific file wins). See [Profiles](#profiles).

- A service's source-tree `application.yml` contains only bootstrap config (`spring.application.name`
  + `config.import: "optional:configserver:,optional:configtree:/run/secrets/"` + the Config Server URI
  `http://localhost:8071`). The `configtree` import turns Docker secret files into properties when the
  service runs in a container (host runs: silent no-op). **Everything else** — DB, Kafka, port,
  schema — is resolved from here at startup. Do not duplicate these values back into a service's
  source tree.

> **Filename = application name:** the file name is the **config-lookup key** (the client's
> bootstrap `spring.application.name`). In every current service the file name and application name
> match — keep it that way; a mismatch silently serves the service only the shared defaults.

## Shared Defaults (`config/application.yml`)

Applied to every service unless overridden:

- **Actuator:** `management.endpoints.web.exposure.include: "*"`
- **Datasource:** single shared PostgreSQL instance (PostgreSQL 18, run via the `webstore` repo's
  `docker-compose.yml`) —
  `jdbc:postgresql://${db.host:localhost}:${db.port:5432}/${db.name:webstore}?currentSchema=${service.schemaName}`,
  driver `org.postgresql.Driver`. Host/port/database carry local-dev defaults inline and are
  overridable per environment via `DB_HOST` / `DB_PORT` / `DB_NAME` env vars (e.g. `DB_HOST=postgres`
  for containerized services). The `${service.schemaName}` placeholder is supplied per-service
  (see below), giving **schema-per-service** isolation inside one shared `webstore` database.
- **Datasource credentials:** `username: ${db_username}` / `password: ${db_password}` — **this repo
  contains no credentials.** Host runs resolve them from `DB_USERNAME` / `DB_PASSWORD` env vars
  (relaxed binding); containerized runs resolve them from Docker secret files named `db_username` /
  `db_password` via each service's `optional:configtree:/run/secrets/` import. The values must match
  the Docker secrets in `webstore/secrets/` (gitignored) that initialize the Postgres container.
- **JPA / Hibernate:** `ddl-auto: validate` (**Flyway is authoritative** for schema — entities must
  match the migrated schema or the service fails to start), `show-sql: true`, `format_sql: true`,
  `PhysicalNamingStrategyStandardImpl`, `PostgreSQLDialect`.
- **Kafka (custom keys — see warning below):**
  - `bootstrap.servers: ${KAFKA_BROKERS:localhost:9092}` (containers set `KAFKA_BROKERS=kafka:19092`,
    the broker's INTERNAL listener)
  - `num.partitions: 3`
  - `replication.factor: 1`   ← sized for a **single-broker** local Kafka
  - `topic.stock.status: stock-status-event`
  - `topic.payment.status: payment-status-event`
  - `topic.order.status: order-status-event`   ← **legacy, no longer read by any service** (the
    payment→order channel moved to `topic.payment.status`)

> ⚠️ **These Kafka keys are custom top-level properties, NOT the standard `spring.kafka.*` keys.**
> They do not auto-configure Spring Boot's Kafka support. Each service that uses Kafka reads them via
> `@Value("${bootstrap.servers}")`, `${num.partitions}`, `${replication.factor}`,
> `${topic.stock.status}`, etc. in its own `KafkaConfig`, and builds its `ConsumerFactory` /
> `ProducerFactory` / `NewTopic` beans by hand. If you rename one of these keys here, you must update
> every consuming `KafkaConfig`.
>
> `replication.factor: 1` and `num.partitions: 3` assume a single local broker. Raising the
> replication factor above the number of running brokers makes `NewTopic` auto-creation fail.

## Per-Service Files (`config/<service>.yml`)

For the **business services**, each file sets only `server.port` and `service.schemaName` (the schema
injected into the shared datasource URL). The infrastructure services carry richer overrides.

`server.port` is written as `${SERVICE_PORT:<default>}` — the inline default (the "Port" column below)
applies to host runs; in Docker each container is passed a uniform `SERVICE_PORT` env var so the port
tracks the single source of truth in `webstore/.env` (Compose maps that repo's per-service
`<NAME>_SERVICE_PORT` value onto `SERVICE_PORT`). See the `webstore` repo's `docker-compose.yml` and
its `.env`. **If you change a default here, change the matching `<NAME>_SERVICE_PORT` in `webstore/.env`
too**, or the host default and the Docker port will diverge.

| File                     | `spring.application.name` | Port (default)          | `service.schemaName` | Notable overrides                                                                 |
|--------------------------|---------------------------|-------------------------|----------------------|-----------------------------------------------------------------------------------|
| `gateway-service.yml`    | (default)                 | `${SERVICE_PORT:8072}`  | —                    | `spring.cloud.gateway.routes` (see below)                                         |
| `product-service.yml`    | (default)                 | `${SERVICE_PORT:8073}`  | `product_schema`     | —                                                                                 |
| `inventory-service.yml`  | (default)                 | `${SERVICE_PORT:8074}`  | `inventory_schema`   | —                                                                                 |
| `user-service.yml`       | (default)                 | `${SERVICE_PORT:8075}`  | `user_schema`        | —                                                                                 |
| `auth-service.yml`       | (default)                 | `${SERVICE_PORT:8076}`  | `auth_schema`        | —                                                                                 |
| `order-service.yml`      | (default)                 | `${SERVICE_PORT:8077}`  | `order_schema`       | `services.inventory.url` / `services.payment.url` — direct REST targets (see below) |
| `payment-service.yml`    | (default)                 | `${SERVICE_PORT:8078}`  | `payment_schema`     | —                                                                                 |

> config-service itself (default port 8071) is **not** configured from this repo — it is the server
> that serves it. Its `server.port: ${SERVICE_PORT:8071}` lives in its **source** `application.yml`
> in the `webstore` repo, driven by the same `SERVICE_PORT` / `.env` mechanism.

### Service URLs — no service discovery

There is **no Eureka / discovery layer**. Every cross-service target is a **direct URL** defined as a
`${<NAME>_SERVICE_URL:http://localhost:<port>}` placeholder: the inline default suits host runs, and
Docker Compose sets the env var to the container DNS name (e.g.
`INVENTORY_SERVICE_URL=http://inventory-service:8074`). In Kubernetes the same env vars will point at
K8s Service names.

- `order-service.yml` → `services.inventory.url`, `services.payment.url` (read via `@Value` by
  order-service's REST client code)
- `gateway-service.yml` → the five route URIs below

### Gateway routes (`gateway-service.yml`)

Each external path is stripped of its prefix via `RewritePath` and forwarded to the direct URL:

| External path   | Route URI placeholder                            |
|-----------------|--------------------------------------------------|
| `/inventory/**` | `${INVENTORY_SERVICE_URL:http://localhost:8074}` |
| `/order/**`     | `${ORDER_SERVICE_URL:http://localhost:8077}`     |
| `/payment/**`   | `${PAYMENT_SERVICE_URL:http://localhost:8078}`   |
| `/product/**`   | `${PRODUCT_SERVICE_URL:http://localhost:8073}`   |
| `/user/**`      | `${USER_SERVICE_URL:http://localhost:8075}`      |

> **auth-service is not routed through the gateway yet.** `auth-service.yml` exists (port 8076,
> `auth_schema`) but there is no `/auth/**` route above and no `AUTH_SERVICE_URL` placeholder —
> it is reachable only directly. Add a sixth route here when it should be exposed externally.

Filter form: `RewritePath=/<prefix>/(?<path>.*), /$\{path}`.

<a id="editing-workflow"></a>## Editing Workflow

1. Edit the relevant file under `C:\Data\Projects\webstore-config\config\`.
2. **Commit and push** to the Git remote — the Config Server reads from Git, so an un-pushed change
   will not take effect.
3. Apply the change to running services by either:
   - restarting the affected service(s), or
   - calling `POST /actuator/refresh` on a service whose beans are `@RefreshScope`.

**Where to put a change:**
- Affects every service, every environment (topic names, DB host default) → `application.yml`.
- Affects one service, every environment (its port, schema, gateway routes, service URLs) → `<service>.yml`.
- Affects every service in one environment (actuator surface, log level, Kafka sizing) → `application-<profile>.yml`.
- Affects one service in one environment → `<service>-<profile>.yml`.

## Conventions & Gotchas

- **Topic property path vs. wire value:** code references the path form `topic.stock.status`; the
  value that actually lands on Kafka is `stock-status-event`. Likewise `topic.order.status` →
  `order-status-event`. Easy to confuse when grepping.
- **`ddl-auto: validate`** means schema drift is fatal at startup. When a service adds a Flyway
  migration, no config change is needed here — but never switch this to `update`/`create` to "fix"
  a validation error; fix the migration instead.
- **No credentials in this repo.** `application.yml` references `${db_username}` / `${db_password}`
  placeholders; the actual values live outside Git (env vars on host runs, Docker secret files in
  containers — see Shared Defaults above). Never commit a literal username/password here — if a value
  must vary non-secretly, add a placeholder with a default or a profile overlay (see below). Note there
  is still no encryption or Vault — secrets stay out of Git entirely, in every profile.
- **Config Server properties beat env vars.** By default Spring Cloud Config property sources override
  the client's system/environment properties. That's why per-environment values are written as inline
  placeholder defaults (`${db.host:localhost}`, `${SERVICE_PORT:8073}`, `${INVENTORY_SERVICE_URL:...}`)
  rather than plain keys (`db.host: localhost`) in this file — a plain repo-defined key would silently
  win over the service's `DB_HOST` / `SERVICE_PORT` / `*_SERVICE_URL` env var. The placeholder form
  leaves the env var free to fill the value and only falls back to the default when it's unset.
- **Profiles are overlays, not full copies.** A `-<profile>.yml` file should contain only the keys that
  **differ** for that environment; everything else inherits from the base file. Don't duplicate an
  entire base file just to change one value.
- **Indentation/whitespace** in these files is hand-maintained YAML — keep two-space indents and
  avoid tabs.

<a id="profiles"></a>## Profiles (DEV / UAT / PROD)

Environment-specific config uses standard Spring Cloud Config profile files:

- **Shared overlays:** `application-dev.yml`, `application-uat.yml`, `application-prod.yml` — merged on
  top of `application.yml` for every service when that profile is active. Current deltas: actuator
  exposure (all in DEV → trimmed in UAT/PROD), SQL log level (`org.hibernate.SQL` debug→warn),
  `format_sql`, and Kafka sizing (`num.partitions` / `replication.factor`).
- **Per-service overlays:** `<service>-<profile>.yml` (e.g. `order-service-prod.yml`) — for a single
  service's env-specific override. Optional; add only when the shared overlay can't express it.

**Selecting the profile:** clients send `SPRING_PROFILES_ACTIVE`. In the `webstore` repo that comes from
the per-environment `.env` file (`SPRING_PROFILE` → forwarded by `docker-compose.yml` as
`SPRING_PROFILES_ACTIVE`); `.env` = DEV, `.env.uat`, `.env.prod`. This repo doesn't choose the profile —
it only supplies the overlays the chosen profile resolves to.

> `replication.factor` in the UAT (2) and PROD (3) overlays assumes a multi-broker Kafka. Against a
> single broker, `NewTopic` auto-creation fails — keep the factor ≤ the running broker count.
