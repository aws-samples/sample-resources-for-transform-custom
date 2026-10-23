# Java Modernisation Sample

A transformation definition (TD) for [AWS Transform custom](https://aws.amazon.com/transform/custom/) that focuses on Java modernisation. The TD is a `SKILL.md` file plus a `references/` folder that the agent reads during the run.

| Sample | Purpose |
|---|---|
| [legacy-java-app-generator](legacy-java-app-generator/) | Generates a deliberately legacy Jakarta EE 8 (`javax.*`) application to use as the input of later modernisation transformations. |

---

## legacy-java-app-generator

Registry name (from the `SKILL.md` front matter): `atx-td-recreate-legacy-inventory-app`.

This TD builds, in an empty repository, a complete **Inventory & Order Management** application that is outdated on purpose. The result compiles, passes its unit tests and runs on WildFly 18–26. It gives you a realistic, reproducible starting point for modernisation work such as a Java version upgrade, a `javax` → `jakarta` migration, dependency upgrades or a frontend migration.

The TD must **not** modernise anything. Every legacy choice below is intentional.

### What gets generated

A multi-module Maven EAR (`com.example.inventory:inventory-management-parent:1.0.0-SNAPSHOT`):

```
inventory-management-parent/   (root pom, dependency and plugin management)
├── inventory-ejb/     EJB module: JPA entities, DTOs, @Stateless services, JAX-RS resources, unit tests
├── inventory-web/     WAR module: AngularJS 1.x SPA (no Java source, no build step)
└── inventory-ear/     EAR module: packaging + WildFly datasource descriptor (inventory-ds.xml)
```

- **Domain model**: 6 JPA entities (`Product`, `Customer`, `Order` mapped to `CUSTOMER_ORDER`, `OrderLineItem`, `Invoice`, `NotificationLog`) and 3 enums, on an in-memory H2 database with `create-drop` schema generation.
- **Services**: 6 `@Stateless` EJBs with `@Local` interfaces (`ProductService`, `CustomerService`, `OrderService`, `InvoiceService`, `NotificationService`, `ReportingService`), plus a `@Singleton @Startup` `DataSeederBean` that seeds 5 products and 3 customers.
- **Business rules**: order placement decrements stock, cancellation restocks, invoices are idempotent with 8% tax, notifications are persisted and sent through a simulated webhook, and reporting computes a sales summary and low-stock alerts.
- **REST API**: 6 JAX-RS resources under `/inventory-web/api` (`/products`, `/customers`, `/orders`, `/invoices`, `/notifications`, `/reports`). The resources are themselves `@Stateless` EJBs.
- **Frontend**: an AngularJS 1.5.11 SPA with `ngRoute`, Bootstrap 3.3.7 and jQuery 1.12.4, covering a dashboard, products, customers, orders, invoices and notifications.
- **Tests**: JUnit 4 + Mockito unit tests, one class per service and per resource (about 69 tests).
- **Docs**: a project `README.md` with build, deploy, smoke-test and troubleshooting steps, and a `.gitignore`.

### Deliberate legacy stack

| Area | Pinned choice |
|---|---|
| Language / platform | Java 11, Jakarta EE 8 with the `javax.*` namespace |
| Runtime | WildFly 18–26 (not 27+, which moved to `jakarta.*`) |
| Logging | Log4j 1.2.17 |
| Utilities | Apache Commons Lang 2.6 |
| JSON | Jackson 2.9.10 |
| HTTP client | Apache Commons HttpClient 3.1 |
| XML binding | JAXB API / RI 2.3.0 |
| Database | H2 1.4.197 (in-memory) |
| Tests | JUnit 4.11, Mockito 3.12.4 |
| Frontend | AngularJS 1.5.11, jQuery 1.12.4, Bootstrap 3.3.7, Bower (`bower.json`) |

### How the TD is organised

```
legacy-java-app-generator/
├── SKILL.md                              The TD: objective, entry criteria, exclusions, steps, exit criteria
└── references/
    ├── RECREATE_SPEC.md                  What to build (sections 0–12), including the generated README structure
    ├── IMPLEMENTATION_DECISIONS.md       Runtime pitfalls P1–P4 and decisions D01–D12, F01–F02, T01–T03
    ├── TEST_PLAN.md                      Minimum behaviours to unit-test, per class
    ├── VENDORED_LIBRARIES.md             URLs, sizes and SHA-256 checksums of the 5 front-end files
    └── templates/                        Files copied verbatim: 4 POMs, EE descriptors, web.xml,
                                          index.html, bower.json, css/app.css, .gitignore
```

When sources disagree, the precedence is: templates, then `RECREATE_SPEC.md`, then `IMPLEMENTATION_DECISIONS.md`, then `TEST_PLAN.md`.

The TD runs in 9 steps: read the references, copy the templates, write the domain model and utilities, write the EJB services, write the JAX-RS layer, write and pass the unit tests, build the SPA (downloading and checksumming the vendored libraries), write the README and run `mvn clean install`, then self-review against the acceptance checklist and report.

### Runtime pitfalls built into the references

These issues pass the build and the unit tests but fail once the EAR is deployed. The references already encode the fixes, which is why the templates must stay unmodified.

| Pitfall | Effect | Fix |
|---|---|---|
| P1: RESTEasy picks JSON-B for `List` responses | `@JsonIgnore` is ignored and list endpoints return `500` with `LazyInitializationException` | `web.xml` template sets `resteasy.preferJacksonOverJsonB=true` |
| P2: exceptions wrapped at the EJB boundary | `catch (IllegalArgumentException)` never fires in the container | Resources also unwrap `EJBException` before returning `400` |
| P3: read-only JSON properties sent back | Saving a product or customer fails on unknown properties (`lowStock`, `fullName`) | Detail controllers send only the editable fields |
| P4: route ordering | `/low-stock` or `/new` gets matched as an id | Literal paths are declared separately and before parameterised ones |

### Prerequisites

- An empty target repository (or one containing only files the TD may overwrite).
- JDK 11 (point `JAVA_HOME` at it if several JDKs are installed).
- Maven 3.6+ with access to Maven Central.
- Outbound HTTPS to the CDNs listed in `references/VENDORED_LIBRARIES.md`.
- The [AWS Transform CLI](https://docs.aws.amazon.com/transform/latest/userguide/custom-get-started.html) (`atx`) installed and configured.

### Running the TD

Publish the TD to your registry, then run it against an empty folder. See the [command reference](https://docs.aws.amazon.com/transform/latest/userguide/custom-command-reference.html) for all options.

```bash
# Publish the TD from this folder
atx custom def publish \
  -n recreate-legacy-java-inventory-app \
  --sd ./legacy-java-app-generator \
  --description "Recreates a deliberately legacy Jakarta EE 8 (javax) Inventory and Order Management EAR app for WildFly 18-26 as input for modernization exercises"

# Run it in an empty target repository
mkdir ./inventory-legacy-app && cd ./inventory-legacy-app && git init
atx custom def exec \
  -n recreate-legacy-java-inventory-app \
  -p . \
  -c "mvn clean install"
```

The run ends after `mvn clean install` and static checks. It does **not** install WildFly or deploy the EAR. The generated project `README.md` explains how to install WildFly 18.0.1.Final locally, deploy `inventory-ear/target/inventory.ear`, open `http://localhost:8080/inventory-web/` and smoke-test the API.

### Guardrails

- No modernisation: no dependency, plugin, Java level or JS library upgrades; no `jakarta.*`, `java.time`, Spring, Lombok, MapStruct, Bean Validation or JUnit 5.
- No TD artifacts in the output: only the application, `README.md` and `.gitignore`.
- Vendored libraries are never fabricated. If every mirror fails, the run stops and reports a blocker.
- No commit, publish or deploy during the run.
- The app has no authentication by design. It runs on a local WildFly bound to loopback, test HTTP servers bind to the loopback address, and the `sa`/`sa` H2 credentials are only acceptable because the database is in-memory and never network-exposed. Do not expose the deployed app beyond your machine.

### Suggested follow-up transformations

Once generated, the code base is a good input for AWS-managed transformations such as:

- a Java version upgrade (for example Java 11 → 21);
- a `javax.*` → `jakarta.*` migration (and a move to WildFly 27+);
- a Log4j 1.x → Log4j 2 migration, and other library upgrades (Commons Lang 3, HttpComponents, recent Jackson and H2);
- an AngularJS 1.x → React frontend migration.

Run `atx custom def list` to see the exact transformation names available in your registry. At the end of its run, the TD itself reports the matching registry names.
