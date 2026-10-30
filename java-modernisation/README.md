# Java Modernisation Sample

This example is the companion resource for this [blog post](https://builder.aws.com/content/3LNMhPi6rrAeiiSMDAjHMDaMpHH/modernising-a-legacy-java-ee-app-with-aws-transform-custom-java-25-spring-boot-and-react-in-four-steps). It is intended for educational purposes only. It generates a legacy Jakarta EE 8 (`javax.*`) application that you can use as the input for later modernisation transformations.

| Sample | Purpose |
|---|---|
| [legacy-java-app-generator](legacy-java-app-generator/) | A transformation definition (TD) for [AWS Transform custom](https://aws.amazon.com/transform/custom/) that focuses on Java modernisation. The `SKILL.md` file and a `references/` folder that the agent reads during the run. |

---

## legacy-java-app-generator

This TD builds, in an empty repository, a complete **Inventory & Order Management** application that is outdated for educational purposes. The result compiles, passes its unit tests and runs on WildFly 18–26. It gives you a realistic, reproducible starting point for a modernisation work such as a Java version upgrade, a `javax` → `jakarta` migration, dependency upgrades or a frontend migration.

### What gets generated

A multi-module Maven EAR with JPA entities, stateless EJB services, business rules, JAX-RS APIs and an AngularJS front end.

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


### Suggested follow-up transformations

Once generated, the code base is a good input for AWS-managed transformations such as:

- a Java version upgrade (for example Java 11 → 21);
- a `javax.*` → `jakarta.*` migration (and a move to WildFly 27+);
- a Log4j 1.x → Log4j 2 migration, and other library upgrades (Commons Lang 3, HttpComponents, recent Jackson and H2);
- an AngularJS 1.x → React frontend migration.

Run `atx custom def list` to see the exact transformation names available in your registry. At the end of its run, the TD itself reports the matching registry names.
