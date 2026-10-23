---
name: atx-td-recreate-legacy-inventory-app
description: Recreates from scratch a deliberately legacy Jakarta EE 8 (javax.*) "Inventory & Order Management" sample application — a Java 11 multi-module Maven EAR (EJB + WAR + EAR) for WildFly 18–26 with JPA/H2, JAX-RS resources that are @Stateless EJBs, deliberately pinned EOL libraries (Log4j 1.2.17, Commons Lang 2.6, Jackson 2.9.10, Commons HttpClient 3.1, JAXB 2.3.0, H2 1.4.197, JUnit 4.11), an AngularJS 1.5.11 / Bootstrap 3.3.7 / jQuery 1.12.4 SPA, JUnit 4 + Mockito unit tests, README and .gitignore. The output is a legacy code base intended as the input of later transformation / upgrade exercises, so it must NOT be modernized.
---

# Recreate the Legacy Jakarta EE 8 Inventory & Order Management App

## Objective
Generate, in an empty repository, a complete **deliberately legacy** "Inventory & Order Management" application, as
described in `references/RECREATE_SPEC.md`, that **compiles, passes its unit tests and runs on WildFly 18–26**. The
code base is the *starting point* for later modernization work: a Java upgrade, a `javax`→`jakarta` migration,
dependency upgrades or a frontend migration. Every outdated choice (Java 11, the `javax.*` namespace, WildFly 18–26,
EOL library versions, AngularJS 1.x, Bower) is **intentional** and must be kept.

The generated code may differ from one run to another. That is fine as long as the app builds, its tests pass, it
runs once deployed, and it keeps the legacy stack. Anything the references do not pin down (wording, Javadoc,
styling, labels, log messages) is the agent's choice.

## Summary
The transformation reads four references:
- `RECREATE_SPEC.md`: what to build.
- `IMPLEMENTATION_DECISIONS.md`: runtime pitfalls found when a reference implementation was deployed on WildFly 18.0.1.Final, plus the gaps the tests depend on.
- `TEST_PLAN.md`: the minimum set of behaviours to unit-test.
- `VENDORED_LIBRARIES.md`: where to download the 5 front-end library files.

It copies the build files and descriptors in `references/templates/` verbatim. It then writes the Java sources
(entities, DTOs, utilities, stateless services with local interfaces, a startup seeder, JAX-RS resources), the unit
tests and the AngularJS SPA. It downloads the vendored libraries, writes `README.md`, and builds with Maven.

The most important runtime fix is already in the `web.xml` template: `resteasy.preferJacksonOverJsonB=true`. Without
it, RESTEasy on WildFly serializes `List` responses with JSON-B, which ignores `@JsonIgnore`, and every list endpoint
fails with `LazyInitializationException`. The build does not catch this, so never modify that template.

## Entry Criteria
1. The target repository is empty, or contains only files this TD may overwrite (`README.md`, `.gitignore`, or a previous partial run of this TD). If unrelated application code is present, STOP and report — do not merge into it.
2. **JDK 11** available (`java -version` reports 11.x). If several JDKs are installed, point `JAVA_HOME` at 11 for every Maven command.
3. **Maven 3.6+** on PATH, with access to Maven Central (or a mirror that serves the pinned versions).
4. Outbound HTTPS to the library CDNs in `references/VENDORED_LIBRARIES.md` (or the listed mirrors).

## Reference Material
| Asset | Location | Use |
|---|---|---|
| Config templates | `references/templates/**` | Verbatim copies of the 4 POMs, `persistence.xml`, `beans.xml`, `ejb-jar.xml`, `log4j.properties`, `inventory-ds.xml`, `web.xml`, `index.html`, `bower.json`, `css/app.css`, `.gitignore` |
| Specification | `references/RECREATE_SPEC.md` | What to build (sections 0–12), including the exact README structure |
| Decisions & pitfalls | `references/IMPLEMENTATION_DECISIONS.md` | Runtime pitfalls P1–P4; decisions D01–D12, F01–F02, T01–T03 |
| Test plan | `references/TEST_PLAN.md` | Minimum behaviours to test, per class |
| Vendored libraries | `references/VENDORED_LIBRARIES.md` | Download URLs, sizes and SHA-256 checksums of the 5 front-end files |

## IMPORTANT: Exclusions
1. **Do NOT modernize.** Do not upgrade any dependency, plugin, Java level or JS library. Do not use `jakarta.*`, `java.time`, Spring, Lombok, MapStruct, Bean Validation or JUnit 5. Do not add authentication.
2. **Do NOT add TD artifacts.** Do not add reports, logs, scripts or tooling to the repository. It must contain only the application, `README.md` and `.gitignore`, because it is the input of later transformations.
3. **Do NOT fabricate vendored libraries.** If a download fails on every mirror, stop and report a blocker.
4. **Do NOT commit, publish or deploy** anything.
5. **Do NOT install, start or deploy to an application server.** Do not download WildFly, run `standalone.sh`, copy the EAR into a deployments folder, or start any long-running process. Deploying and checking the app at runtime is the user's job, after the run. The generated `README.md` tells them how (spec section 11). This run's verification ends at `mvn clean install` plus the static checks in step 9.

## Available AWS-managed Transformations
This TD produces a *legacy* code base on purpose. When reporting at the end of the run, call the
`list_available_transformations_from_registry` tool and name, by their exact registry names, the transformations that
fit as **follow-ups**. Examples: a Java version upgrade or a `javax`→`jakarta` migration; a Log4j 1.x migration; and the
Angular-to-React migration, which also handles AngularJS 1.x frontends like this one. Never apply them in this run.

## Implementation Steps

Precedence when sources disagree: the templates (for the files they provide), then an explicit sentence in
`RECREATE_SPEC.md`, then `IMPLEMENTATION_DECISIONS.md`, then `TEST_PLAN.md`.

### 1. Initialization
1. Read all four reference documents completely before writing any file.
2. Create the module skeleton:
   ```bash
   P=inventory-ejb/src/main/java/com/example/inventory
   mkdir -p $P/{entity,dto,util,service,rest} inventory-ejb/src/main/resources/META-INF \
            inventory-ejb/src/test/java/com/example/inventory/{service,rest} \
            inventory-web/src/main/webapp/{WEB-INF,css,lib,app/{product,customer,order,invoice,notification,report}} \
            inventory-ear/src/main/application/META-INF
   ```

### 2. Build files and descriptors (verbatim from templates)
Copy these files byte for byte. Do not reformat or "improve" them.

| Template | Destination |
|---|---|
| `templates/root/pom.xml` | `pom.xml` |
| `templates/root/gitignore.txt` | `.gitignore` |
| `templates/inventory-ejb/pom.xml` | `inventory-ejb/pom.xml` |
| `templates/inventory-ejb/META-INF/{persistence,beans,ejb-jar}.xml` | `inventory-ejb/src/main/resources/META-INF/` |
| `templates/inventory-ejb/log4j.properties` | `inventory-ejb/src/main/resources/log4j.properties` |
| `templates/inventory-web/pom.xml` | `inventory-web/pom.xml` |
| `templates/inventory-web/WEB-INF/web.xml` | `inventory-web/src/main/webapp/WEB-INF/web.xml` |
| `templates/inventory-web/index.html`, `bower.json` | `inventory-web/src/main/webapp/` |
| `templates/inventory-web/css/app.css` | `inventory-web/src/main/webapp/css/app.css` |
| `templates/inventory-ear/pom.xml` | `inventory-ear/pom.xml` |
| `templates/inventory-ear/META-INF/inventory-ds.xml` | `inventory-ear/src/main/application/META-INF/inventory-ds.xml` |

Then run `mvn -B -q validate` from the root. It must succeed before you continue.

### 3. Domain model, DTOs and utilities
1. Write the 3 enums and 6 entities following spec section 3 and decision D12: `CUSTOMER_ORDER` as the `Order` table, the named queries, the helper getters `isLowStock()`, `getFullName()` and `getSubtotal()`, `@JsonIgnore` on the four lazy back-references, and EAGER fetch on `Order.customer` and `Order.lineItems`.
2. Write the 3 DTOs (spec section 4), and `LoggingUtil` and `InvoiceNumberGenerator` (spec section 5).

### 4. EJB services
1. Write the 6 `@Local` interfaces with the signatures in spec section 6.
2. Write the 6 `@Stateless` `<Domain>Service` beans with the business rules in spec section 6 and decisions D01–D09. Inject the persistence context with `@PersistenceContext(unitName = "inventoryPU") private EntityManager em;` and collaborators with `@EJB private`.
3. Write `DataSeederBean`: `@Singleton @Startup`, a `@PostConstruct` seed method, idempotent, with the seed data from spec section 6.
4. Compile: `mvn -B -q -pl inventory-ejb compile`.

### 5. JAX-RS layer
1. Write `InventoryRestApplication` and the 6 `@Stateless` resources, with the paths and status codes in spec section 7 and decisions D10–D11.
2. Apply pitfall **P2**: unwrap `EJBException` to return a 400 in `OrderResource.placeOrder`, `OrderResource.updateStatus` and `InvoiceResource.generateInvoice`.
3. Declare `@Path("/low-stock")` as its own method, separate from `@Path("/{id}")` (pitfall P4).

### 6. Unit tests
1. Write one test class per service and per resource, covering every behaviour in `references/TEST_PLAN.md`, applying T01–T03 and the `setField` reflection helper.
2. Run `mvn -B test -pl inventory-ejb` and require `Failures: 0, Errors: 0, Skipped: 0`. Fix production code or tests until that holds. Never delete, skip or weaken tests to get there.

### 7. Frontend SPA
1. Download and checksum the 5 libraries exactly as described in `references/VENDORED_LIBRARIES.md`.
2. Write the JS files and partials with the exact paths `index.html` loads (spec section 9), using the controller names in F01 and the `$http` calls in F02. Declare the literal routes before the parameterized ones (P4). Never use `ng-bind-html`, `$sce.trustAsHtml`, `innerHTML` or `eval`.
3. Make the product and customer detail controllers send only the editable fields (P3).
4. Do not write `css/app.css`: it is the template copied in step 2. Write the partials with the markup it styles (spec section 9, "Markup").

### 8. Documentation and build
1. Write `README.md` with **exactly** the structure in spec section 11, including the "not deployed" note, the "Check the deployment" list and the JSON-B troubleshooting row.
2. Build from the root: `mvn -B clean install`. Require `BUILD SUCCESS`, all tests passing and `inventory-ear/target/inventory.ear`. Redirect the output to a temporary file outside the repository and inspect it with `grep`/`tail`.

### 9. Self-review and report
1. Walk through every item of the spec section 12 checklist and every pitfall P1–P4, and fix any gap. Check them statically (source, descriptors, build output); never deploy to check them (Exclusion 5). Pay particular attention to the "Runnable on WildFly" items. The unit tests mock the `EntityManager`, so a wrong JPQL field name, a wrong `mappedBy` or a reserved-word column passes the build and only fails when the EAR is deployed.
2. Remove any build artifacts or temporary files that `.gitignore` does not cover.
3. End the run with a short report containing:
   - the Maven result and the test tally;
   - any library-mirror substitution;
   - any judgment call not covered by the references;
   - the candidate follow-up transformations (exact registry names);
   - a statement that the app was not deployed, pointing to the README's "Running locally" section.

## Constraints and Guardrails
1. **Pinned versions only:** the POM templates pin every backend and test dependency and every plugin. Frontend: AngularJS 1.5.11, angular-route 1.5.11, jQuery 1.12.4, Bootstrap 3.3.7, as listed in `VENDORED_LIBRARIES.md`.
2. **Namespace:** every EE import is `javax.*`. The only allowed occurrence of `jakarta` is the POM property name `jakarta.ee.version`.
3. **EJB rules:** resources are `@Stateless`, every `@EJB` field is `private`, and service names have no `Bean` suffix (`DataSeederBean` is the only exception).
4. **Serialization:** keep `@JsonIgnore` on `Customer.orders`, `Product.orderLineItems`, `OrderLineItem.order` and `OrderLineItem.product`. No `@JsonIgnoreProperties`.
5. **Security notes:**
   - The app deliberately has no authentication (spec). The README runs it on a local WildFly, which binds to loopback by default. Do not add any instruction to bind it to other interfaces.
   - Test HTTP servers bind to `InetAddress.getLoopbackAddress()`.
   - The `sa`/`sa` H2 credentials in `inventory-ds.xml` are acceptable only because the database is in-memory inside the JVM and never network-exposed.

## Validation / Exit Criteria
The run is complete only when all of the following hold:
1. **Build:** `mvn clean install` on JDK 11 ends with `BUILD SUCCESS` and produces `inventory-ear/target/inventory.ear`.
2. **Tests:** all unit tests pass (`Failures: 0, Errors: 0, Skipped: 0`), with one test class per service and per resource, together covering every behaviour in `TEST_PLAN.md`.
3. **Legacy stack:** zero `jakarta.*` imports; the templated files are unmodified; each of the 5 vendored files either matches its SHA-256 in `VENDORED_LIBRARIES.md` or, when taken from a listed alternative mirror, carries the expected version banner on its first line (and the substitution is reported).
4. **Runnable on WildFly (checked statically, no deployment):** every "Runnable on WildFly" item in spec section 12 holds.
5. **Repository content:** only the application, `README.md` and `.gitignore` — no TD artifacts.
6. **Report:** the final report contains everything listed in step 9.3.
