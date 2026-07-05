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
    ├── discovery-service.yml    # per-service overrides (one file per service)
    ├── gateway-service.yml
    ├── product-service.yml
    ├── inventory-service.yml
    ├── user-service.yml
    ├── order-service.yml
    └── payment-service.yml
```

All runtime configuration lives under `config/`. There is nothing else to build or run here.

## How Configuration Resolution Works

Spring Cloud Config matches each requesting service's `spring.application.name` to a
`<name>.yml` file in `config/`, then **merges it on top of `application.yml`** (the shared
defaults). Property precedence: `<service>.yml` overrides `application.yml`.

- A service's source-tree `application.yml` contains only bootstrap config (`spring.application.name`
  + `config.import: "optional:configserver:,optional:configtree:/run/secrets/"` + the Config Server URI
  `http://localhost:8071`). The `configtree` import turns Docker secret files into properties when the
  service runs in a container (host runs: silent no-op). **Everything else** — DB, Kafka, port,
  schema — is resolved from here at startup. Do not duplicate these values back into a service's
  source tree.

> **Filename vs. application name gotcha:** the file name is the **config-lookup key** (the client's
> bootstrap `spring.application.name`), and a served file may then override `spring.application.name`
> to a *different* runtime value. `discovery-service.yml` is fetched under the key
> `discovery-service` but sets `spring.application.name: discovery-server` — so the Eureka server
> registers as **`discovery-server`** while its config file is named `discovery-service.yml`. The
> business services don't do this; their file name and application name match.

## Shared Defaults (`config/application.yml`)

Applied to every service unless overridden:

- **Eureka client:** `defaultZone: ${EUREKA_URI:http://localhost:8070/eureka/}` (containers set
  `EUREKA_URI=http://discovery-service:8070/eureka/`), `preferIpAddress: true`,
  `registerWithEureka: true`, `fetchRegistry: true`
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

| File                     | `spring.application.name` | Port | `service.schemaName` | Notable overrides                                                                 |
|--------------------------|---------------------------|------|----------------------|-----------------------------------------------------------------------------------|
| `discovery-service.yml`  | `discovery-server`        | 8070 | —                    | Eureka **server** (`registerWithEureka: false`, `fetchRegistry: false`)           |
| `gateway-service.yml`    | (default)                 | 8072 | —                    | `spring.cloud.gateway.routes` (see below)                                         |
| `product-service.yml`    | (default)                 | 8073 | `product_schema`     | —                                                                                 |
| `inventory-service.yml`  | (default)                 | 8074 | `inventory_schema`   | —                                                                                 |
| `user-service.yml`       | (default)                 | 8075 | `user_schema`        | —                                                                                 |
| `order-service.yml`      | (default)                 | 8077 | `order_schema`       | —                                                                                 |
| `payment-service.yml`    | (default)                 | 8078 | `payment_schema`     | —                                                                                 |

> config-service itself (port 8071) is **not** configured from this repo — it is the server that
> serves it.

### Gateway routes (`gateway-service.yml`)

Each external path is stripped of its prefix via `RewritePath` and forwarded to the Eureka-named
service:

| External path   | Routed to (`lb://`)      |
|-----------------|--------------------------|
| `/inventory/**` | `lb://inventory-service` |
| `/order/**`     | `lb://order-service`     |
| `/payment/**`   | `lb://payment-service`   |
| `/product/**`   | `lb://product-service`   |
| `/user/**`      | `lb://user-service`      |

Filter form: `RewritePath=/<prefix>/(?<path>.*), /$\{path}`.

<a id="editing-workflow"></a>## Editing Workflow

1. Edit the relevant file under `C:\Data\Projects\webstore-config\config\`.
2. **Commit and push** to the Git remote — the Config Server reads from Git, so an un-pushed change
   will not take effect.
3. Apply the change to running services by either:
   - restarting the affected service(s), or
   - calling `POST /actuator/refresh` on a service whose beans are `@RefreshScope`.

**Where to put a change:**
- Affects every service (Kafka brokers, DB host, Eureka URL, topic names) → `application.yml`.
- Affects one service (its port, schema, gateway routes, Eureka server flags) → `<service>.yml`.

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
  must vary, add a placeholder with a non-secret default instead. Note there is still no encryption,
  Vault, or per-environment profile — this remains a local-dev setup, just without secrets in Git.
- **Config Server properties beat env vars.** By default Spring Cloud Config property sources override
  the client's system/environment properties. That's why the datasource defaults are inline placeholder
  defaults (`${db.host:localhost}`) rather than `db.host: localhost` keys in this file — repo-defined
  keys would silently win over a service's `DB_HOST` env var.
- **No profiles.** There are no `<service>-<profile>.yml` variants today; every environment would
  resolve to the same values. Introduce Spring profiles if environment-specific config is needed.
- **Indentation/whitespace** in these files is hand-maintained YAML — keep two-space indents and
  avoid tabs.
