# Build a Legacy Jakarta EE 8 "Inventory & Order Management" Sample App

You are recreating a **deliberately legacy** Java enterprise application from scratch. The app is the input of later modernization exercises, so every "outdated" choice below is **intentional**. Do not modernize, upgrade, or "fix" any dependency, API, or pattern.

The goal is an app that **compiles, passes its tests and runs on WildFly**. Anything this spec does not pin down (wording, Javadoc, styling, labels, column lengths, log messages) is free: choose something sensible.

## 0. Non-negotiable constraints

- **Java 11** only (`maven.compiler.source`/`target = 11`). Do not use a newer language level.
- **Jakarta EE 8 with the `javax.*` namespace** (NOT `jakarta.*`). Every EE import must be `javax.ejb.*`, `javax.persistence.*`, `javax.ws.rs.*`, `javax.annotation.*`, `javax.xml.bind.*`.
- Target runtime is **WildFly 18–26** (any release still on the `javax.*` namespace). Do NOT target WildFly 27+.
- Keep the deliberately **EOL / outdated dependencies** at the exact versions in the templates. Do not bump them.
- Packaging is a **multi-module Maven EAR** (ejb + war + ear). Do not collapse it into a single module or convert to Spring Boot.
- Do not add authentication, Bean Validation, Lombok, MapStruct, `java.time`, or any framework not listed.
- Build must succeed with `mvn clean install` on JDK 11, producing `inventory-ear/target/inventory.ear`.

## 1. Project identity & layout

Maven coordinates: `com.example.inventory:inventory-management-parent:1.0.0-SNAPSHOT`, packaging `pom`.

```
inventory-management-parent/      (root pom, packaging=pom, dependencyManagement + pluginManagement)
├── inventory-ejb/     packaging=ejb   entities, enums, DTOs, service beans, local interfaces, JAX-RS resources, tests
├── inventory-web/     packaging=war   AngularJS 1.x SPA, no Java source
└── inventory-ear/     packaging=ear   bundles ejb + war, ships a WildFly datasource descriptor
```

Java base package: `com.example.inventory`.

## 2. Dependencies, plugins and module POMs

**All four POMs are provided verbatim in `references/templates/`. Copy them; do not rewrite them.** They pin these versions:

| Dependency | GroupId:ArtifactId | Version | Scope | Used for |
|---|---|---|---|---|
| Java EE 8 API | `javax:javaee-api` | `8.0.1` | provided | Whole EE API surface |
| Log4j 1.x | `log4j:log4j` | `1.2.17` | compile | Logging (`LoggingUtil`) |
| Commons Lang 2.x | `commons-lang:commons-lang` | `2.6` | compile | `Validate`, `DateFormatUtils`, `RandomStringUtils` |
| Jackson databind/core/annotations | `com.fasterxml.jackson.core:*` | `2.9.10` | compile | Invoice audit JSON, `@JsonIgnore` |
| Commons HttpClient | `commons-httpclient:commons-httpclient` | `3.1` | compile | Notification webhook POST |
| JAXB API / impl | `javax.xml.bind:jaxb-api`, `com.sun.xml.bind:jaxb-impl` | `2.3.0` | compile | `@XmlRootElement` |
| H2 | `com.h2database:h2` | `1.4.197` | compile | In-memory DB |
| JUnit | `junit:junit` | `4.11` | test | Unit tests |
| Mockito | `org.mockito:mockito-core` | `3.12.4` | test | Mocks |
| Jersey common | `org.glassfish.jersey.core:jersey-common` | `2.29.1` | test | Lets `javax.ws.rs.core.Response` be built in unit tests outside a container |

## 3. Domain model (JPA entities, package `…​.entity`)

All entities: `implements Serializable`, `@Entity @Table(name=…) @XmlRootElement`, `IDENTITY` id generation. Money is `BigDecimal`; timestamps are `java.util.Date` with `@Temporal(TIMESTAMP)` (legacy on purpose). Map `Order` to the table `CUSTOMER_ORDER`, because `ORDER` is a reserved word and breaks schema generation. Avoid other SQL reserved words as table or column names. Entities are plain JavaBeans with a no-arg constructor and getters/setters; no `@JsonIgnoreProperties`.

Named queries the services rely on: `Product.findAll` (ORDER BY name), `Product.findBySku`, `Product.findLowStock` (`quantityOnHand <= reorderThreshold`), `Customer.findAll`, `Customer.findByEmail`, `Order.findAll` (ORDER BY orderDate DESC), `Order.findByCustomer`, `Order.findByStatus`, `Invoice.findAll`, `Invoice.findByOrder`, `NotificationLog.findAll`. Find-by-id goes through `em.find`; other queries may be inline `em.createQuery`.

Entities, their key fields, and relationships:

- **Product**: `sku` (unique), `name`, `description`, `unitPrice`, `quantityOnHand`, `reorderThreshold`; `@OneToMany(mappedBy="product", cascade=ALL, orphanRemoval=true)` to its `OrderLineItem`s. Helper `isLowStock()` = `quantityOnHand <= reorderThreshold`.
- **Customer**: `firstName`, `lastName`, `email` (unique), `phone`, `shippingAddress`; `@OneToMany(mappedBy="customer", cascade=ALL)` to its `Order`s. Helper `getFullName()`.
- **Order**: `@ManyToOne(fetch=EAGER)` to `Customer`; `orderDate`, `status` (`OrderStatus`, `@Enumerated(STRING)`), `totalAmount`; `@OneToMany(mappedBy="order", cascade=ALL, orphanRemoval=true, fetch=EAGER)` to `OrderLineItem`.
- **OrderLineItem**: `@ManyToOne(fetch=LAZY)` to `Order` and to `Product`; `quantity`, `unitPriceAtOrder`. Helper `getSubtotal()` = `unitPriceAtOrder * quantity` (null-safe → ZERO).
- **Invoice**: `@OneToOne` to `Order` (unique); `invoiceNumber` (unique), `issueDate`, `subtotal`, `taxAmount`, `totalAmount`, `paymentStatus` (`InvoicePaymentStatus`, `@Enumerated(STRING)`).
- **NotificationLog**: `channel` (`NotificationChannel`, `@Enumerated(STRING)`), `recipient`, `subject`, `body`, `sentDate`, `successful` (boolean).

**Enums** (`…​.entity`): `OrderStatus` = `NEW, CONFIRMED, SHIPPED, DELIVERED, CANCELLED`; `InvoicePaymentStatus` = `PENDING, PAID, OVERDUE, VOID`; `NotificationChannel` = `EMAIL, SMS, SYSTEM_LOG`.

**JSON serialization rule (required for the app to run):** mark the lazy back-references `@JsonIgnore` (`Customer.orders`, `Product.orderLineItems`, `OrderLineItem.order`, `OrderLineItem.product`). Keep the associations the frontend needs EAGER and serialized: `Order.customer`, `Order.lineItems`, `Invoice.order`. This avoids `LazyInitializationException` after the transaction closes. It only works if Jackson serializes every response, which the `web.xml` template guarantees (see §8).

## 4. DTOs (package `…​.dto`, all `@XmlRootElement`, `Serializable`)

- **OrderRequestDto**: inbound body for `POST /orders`, shape `{ customerId, lineItems: [ { productId, quantity } ] }`. A `Long customerId` plus a `List<OrderLineItemRequestDto>`, where `OrderLineItemRequestDto` is a `public static` nested class `{ Long productId; int quantity; }` (also `Serializable`). The SPA and the README smoke test send this shape.
- **SalesSummaryDto**: output of `GET /reports/sales-summary`: `long totalOrders`, `BigDecimal totalRevenue`, `BigDecimal averageOrderValue`, `long pendingInvoices`, `long lowStockProductCount`.
- **LowStockAlertDto**: `Long productId`, `String sku`, `String productName`, `int quantityOnHand`, `int reorderThreshold`; a no-arg and an all-args constructor.

## 5. Utilities (package `…​.util`)

- **LoggingUtil**: `final` class, private ctor, static `Logger getLogger(Class<?>)` returning `org.apache.log4j.Logger.getLogger(clazz)`. Every bean logs through it, so Log4j 1.x usage is visible to modernization tooling.
- **InvoiceNumberGenerator**: `final`, private ctor, static `String generate(Date issueDate)` → `"INV-" + DateFormatUtils.format(issueDate, "yyyyMMdd") + "-" + RandomStringUtils.randomNumeric(6)`, using `org.apache.commons.lang.time.DateFormatUtils` and `org.apache.commons.lang.RandomStringUtils` (Commons Lang **2.x**).

## 6. EJB services (package `…​.service`)

One `@Stateless` bean per domain, named **`<Domain>Service`** (no `Bean` suffix): `ProductService`, `CustomerService`, `OrderService`, `InvoiceService`, `NotificationService`, `ReportingService`. Each implements an `@Local` interface `<Domain>ServiceLocal`. Inject the persistence context with `@PersistenceContext(unitName = "inventoryPU") private EntityManager em` (the unit name must match `persistence.xml`) and collaborators with `@EJB` on **`private`** fields (tests inject them by reflection).

Local interface signatures (the resources and tests call these):

| Interface | Methods |
|---|---|
| `ProductServiceLocal` | `Product createProduct(Product)`, `Product updateProduct(Long id, Product)`, `void deleteProduct(Long id)`, `Product findById(Long)`, `Product findBySku(String)`, `List<Product> findAll()`, `List<Product> findLowStock()`, `void adjustStock(Long productId, int delta)` |
| `CustomerServiceLocal` | `Customer createCustomer(Customer)`, `Customer updateCustomer(Long id, Customer)`, `void deleteCustomer(Long id)`, `Customer findById(Long)`, `Customer findByEmail(String)`, `List<Customer> findAll()` |
| `OrderServiceLocal` | `Order placeOrder(OrderRequestDto)`, `Order updateStatus(Long orderId, OrderStatus newStatus)`, `void cancelOrder(Long orderId)`, `Order findById(Long)`, `List<Order> findAll()`, `List<Order> findByCustomer(Long customerId)`, `List<Order> findByStatus(OrderStatus)` |
| `InvoiceServiceLocal` | `Invoice generateInvoice(Long orderId)`, `Invoice markPaid(Long invoiceId)`, `Invoice findById(Long)`, `Invoice findByOrder(Long orderId)`, `List<Invoice> findAll()`, `List<Invoice> findByPaymentStatus(InvoicePaymentStatus)` |
| `NotificationServiceLocal` | `NotificationLog send(NotificationChannel, String recipient, String subject, String body)`, `List<NotificationLog> findAll()` |
| `ReportingServiceLocal` | `SalesSummaryDto getSalesSummary()`, `List<LowStockAlertDto> getLowStockAlerts()`, `void dispatchLowStockAlerts()` |

Plain CRUD/finder methods follow from the REST endpoints in §7: implement them the obvious way (`NoResultException` → null for single-result lookups; Commons Lang `Validate` on create). Business rules:

- **Order placement** (`placeOrder`): reject a request with no customer or no line items (`IllegalArgumentException`); new order starts `status = NEW`; each line captures `unitPriceAtOrder = product.unitPrice`; order total = sum of line subtotals; **decrement product stock** by each quantity via `adjustStock(productId, delta)`, which throws `IllegalStateException` if stock would go negative; send an order-confirmation notification.
- **Order status / cancel**: `updateStatus` → on `SHIPPED` send a shipment notification; `cancelOrder` → **restock** every line (`+quantity`) and set `status = CANCELLED`.
- **Invoicing** (`generateInvoice`): **idempotent**. If an invoice already exists for the order, return it. Otherwise: `invoiceNumber` from `InvoiceNumberGenerator`, `subtotal = order.totalAmount`, **`tax = subtotal × 0.08`**, `total = subtotal + tax`, `paymentStatus = PENDING`; then log a JSON "audit snapshot" of the invoice written with a Jackson `ObjectMapper`. `markPaid` → `paymentStatus = PAID`.
- **Notifications** (`send`): persist a `NotificationLog` for every attempt with a `successful` flag. `SYSTEM_LOG` just logs (success). `EMAIL`/`SMS` simulate an outbound webhook POST using **Commons HttpClient 3.1** (`org.apache.commons.httpclient.*`) to a placeholder URL; failures are caught and recorded as `successful = false` (the endpoint is normally unavailable; that is expected and must never break the calling operation).
- **Reporting**: `getSalesSummary` computes totalOrders, totalRevenue (sum of order totals), averageOrderValue (`totalRevenue / count`, `HALF_UP`, scale 2; ZERO if none), pendingInvoices (count `PENDING`), lowStockProductCount. `getLowStockAlerts` projects low-stock products to `LowStockAlertDto`. `dispatchLowStockAlerts` sends one `EMAIL` alert to `inventory-ops@example.com` summarizing them (no-op if none).

**DataSeederBean** (the only class with a `Bean` suffix): `@Singleton @Startup` with a `@PostConstruct` method. Injects `ProductServiceLocal` and `CustomerServiceLocal` (`@EJB private`) and creates the seed data through them. Idempotent: skip a product or customer whose sku or email already exists. Seed data (the README smoke test relies on it):
  - Products: `SKU-1001 "Wireless Mouse"` 19.99 qty 150 thr 20; `SKU-1002 "Mechanical Keyboard"` 79.99 qty 60 thr 10; `SKU-1003 "27-inch Monitor"` 249.99 qty 8 thr 10; `SKU-1004 "USB-C Dock"` 89.99 qty 45 thr 15; `SKU-1005 "Laptop Stand"` 34.99 qty 5 thr 15. (SKU-1003 and SKU-1005 start below threshold so low-stock reporting has data.)
  - Customers: Alice Nguyen `alice.nguyen@example.com`; Bob Martinez `bob.martinez@example.com`; Chloe Dubois `chloe.dubois@example.com`, with placeholder phones and addresses.

## 7. JAX-RS REST layer (package `…​.rest`)

- **InventoryRestApplication**: `extends javax.ws.rs.core.Application`, annotated `@ApplicationPath("/api")`, empty body.
- Every resource class is annotated `@Stateless @Path(...) @Produces(APPLICATION_JSON) @Consumes(APPLICATION_JSON)` and injects its service via `@EJB` on a **`private`** field typed with the local interface. **The resources must be EJBs (`@Stateless`)** so the container injects their `@EJB` fields; a plain POJO resource throws `NullPointerException` at runtime.
- Every resource method returns `javax.ws.rs.core.Response`.

Endpoints (base path `/api`, under context root `/inventory-web` → full prefix `http://localhost:8080/inventory-web/api`). The SPA (§9) and the README smoke test (§11) call these paths:

| Resource | Method + path | Behavior |
|---|---|---|
| **ProductResource** `/products` | `GET /` | list all |
| | `GET /{id}` | one (404 if missing) |
| | `GET /low-stock` | list low stock |
| | `POST /` | create → 201 |
| | `PUT /{id}` | update (404 if missing) |
| | `DELETE /{id}` | delete → 204 |
| **CustomerResource** `/customers` | `GET /`, `GET /{id}`, `POST /`, `PUT /{id}`, `DELETE /{id}` | CRUD, same shape as products (no low-stock) |
| **OrderResource** `/orders` | `GET /` (`?customerId=` or `?status=` optional filters) | list/filter |
| | `GET /{id}` | one (404) |
| | `POST /` | place order from `OrderRequestDto` → 201, or 400 with message on `IllegalArgumentException` |
| | `PUT /{id}/status?value=` | update status (404 if missing), or 400 with message on `IllegalArgumentException` (e.g. missing `value`) |
| | `PUT /{id}/cancel` | cancel → 204 |
| **InvoiceResource** `/invoices` | `GET /` (`?status=` optional) | list/filter |
| | `GET /{id}` | one (404) |
| | `POST /generate/{orderId}` | generate → 201, 400 on bad arg |
| | `PUT /{id}/mark-paid` | mark paid (404) |
| **NotificationResource** `/notifications` | `GET /` | list log |
| | `POST /test` | send a SYSTEM_LOG test notification → 201 |
| **ReportResource** `/reports` | `GET /sales-summary` | SalesSummaryDto |
| | `GET /low-stock-alerts` | List<LowStockAlertDto> |
| | `POST /low-stock-alerts/dispatch` | dispatch → 202 Accepted |

Use `QueryParam` enums directly (JAX-RS converts `?status=PENDING` to the enum).

## 8. EE descriptors & config

**All descriptors are provided verbatim in `references/templates/`. Copy them; do not rewrite them.** Values the Java code must match:

- `persistence.xml`: JTA persistence unit **`inventoryPU`** on **`java:/InventoryDS`**, `hibernate.hbm2ddl.auto=create-drop`, H2 dialect, no `<class>` list (entities are found by scanning).
- `inventory-ds.xml` (lands in the EAR's `META-INF/`): deployable WildFly datasource `java:/InventoryDS` on in-memory H2 (`jdbc:h2:mem:inventorydb;DB_CLOSE_DELAY=-1;DB_CLOSE_ON_EXIT=FALSE`), using the H2 module that ships with WildFly. No `standalone.xml` edits are needed.
- `web.xml`: sets the context-param `resteasy.preferJacksonOverJsonB=true`. **Why it is mandatory:** WildFly 18–26 ships both a Jackson and a JSON-B (Yasson) provider for RESTEasy. Without this flag, RESTEasy serializes single-entity responses with Jackson but picks **JSON-B for `List` responses**. JSON-B ignores Jackson's `@JsonIgnore`, walks the lazy back-references after the transaction has closed, and every list endpoint fails with `500` (`RESTEASY008205: JSON Binding serialization error … LazyInitializationException`). RESTEasy reads the flag from the web module's servlet context even though the `Application` class lives in the EJB module. Do **not** work around it with `@JsonbTransient`, a `jboss-deployment-structure.xml` exclusion, or by making the associations EAGER.
- `beans.xml`, `ejb-jar.xml`, `log4j.properties`: minimal; copy as provided.

## 9. Frontend: AngularJS 1.x SPA (`inventory-web/src/main/webapp`)

No build step, no npm. `index.html`, `bower.json` and `css/app.css` are provided in `references/templates/`; copy them. The 5 vendored library files come from `references/VENDORED_LIBRARIES.md`.

**Files.** `index.html` loads every app file by path, so these exact names are required:
- `app/app.module.js`, `app/app.config.js`, `app/app.constants.js`
- `app/<feature>/<feature>.service.js` for `product`, `customer`, `order`, `invoice`, `notification`, plus `app/report/report.service.js`
- controllers: `product/product-list.controller.js`, `product/product-detail.controller.js`, `customer/customer-list.controller.js`, `customer/customer-detail.controller.js`, `order/order-list.controller.js`, `order/order-new.controller.js`, `invoice/invoice-list.controller.js`, `notification/notification-list.controller.js`, `report/dashboard.controller.js`
- one `.html` partial next to each controller, with the same base name (`report/dashboard.html` for the dashboard)
- `css/app.css`: the template, unmodified. Do not add other stylesheets or inline `style` attributes.

**Wiring.**
- `app.module.js`: `angular.module('inventoryApp', ['ngRoute']);` (`index.html` uses `ng-app="inventoryApp"`).
- `app.constants.js`: `.constant('API_BASE_URL', 'api')`, a relative URL resolved against `<base href="/inventory-web/">`.
- `app.config.js`: `$locationProvider.hashPrefix('!')` (the navbar links are `#!/…`); routes `/dashboard`, `/products`, `/products/new`, `/products/:productId`, `/customers`, `/customers/new`, `/customers/:customerId`, `/orders`, `/orders/new`, `/invoices`, `/notifications`, `.otherwise({ redirectTo: '/dashboard' })`. Each route uses `controllerAs: 'vm'`.
- Use explicit DI annotation arrays (`['$http', 'API_BASE_URL', Fn]`) and wrap each file in an IIFE with `'use strict'`.
- Six `$http` services (`ProductService`, `CustomerService`, `OrderService`, `InvoiceService`, `NotificationService`, `ReportService`) wrap the endpoints in §7 and return the `$http` promises.

**Pages (behaviour, not layout).** Each page shows a loading state and an error message when a call fails.
- **Dashboard**: the four sales-summary KPIs, the low-stock alerts table, and a "dispatch alerts" action.
- **Products / Customers**: list with a delete action (confirmed with `window.confirm`), a link to create, and a detail page to create or edit. Highlight low-stock products.
- **Orders**: list with "mark shipped" and "cancel" actions (cancel confirmed); a "new order" page to pick a customer and one or more product + quantity lines.
- **Invoices**: list, a "generate invoice for order id" action, and a "mark paid" action on pending invoices.
- **Notifications**: list, and a "send test notification" action.

**Markup.** `css/app.css` styles stock Bootstrap 3 classes and a few AngularJS attributes, with no custom classes. Write the partials so its selectors match:
- Page title: one `<h2>` per partial; sub-sections use `<h3>`.
- Dashboard KPIs: one `<div class="row">` holding four `<div class="col-md-3"><div class="panel panel-default">`, each with a `<div class="panel-heading">` (label) and a `<div class="panel-body">` (value).
- Tables: `<table class="table table-striped">` with `<thead>` and `<tbody>`. Mark low-stock rows with the `danger` class: `class="danger"` on the dashboard alert rows, `ng-class="{'danger': product.lowStock}"` on the product list.
- Loading state: `<div ng-if="vm.loading">…</div>`. Empty state: `<p ng-if="!vm.loading && vm.<items>.length === 0">…</p>` (the dashboard may omit `!vm.loading`).
- Messages: `<div class="alert alert-danger" ng-if="vm.error">` for errors, `alert alert-success` for confirmations.
- Buttons: `btn btn-primary` for create/save, `btn btn-default` for cancel/back, `btn btn-sm btn-danger` for row delete/cancel actions, `btn btn-warning` for "dispatch alerts".
- Forms: `form-group` + `form-control`; inline filters and actions (e.g. "generate invoice for order id") use `<form class="form-inline">`.

Rely on AngularJS interpolation auto-escaping. Do NOT use `ng-bind-html`, `$sce.trustAsHtml`, `innerHTML`, or `eval`.

## 10. Tests (inventory-ejb `src/test/java`, JUnit 4 + Mockito)

Unit tests only, no running container. Write JUnit **4** tests (`@Test`, `@Before`, `org.junit.Assert`). The minimum set of behaviours, one test class per service and per resource, is in `references/TEST_PLAN.md`. `jersey-common` (test scope) is on the classpath so `Response` can be instantiated.

## 11. README.md and .gitignore (project root)

### README.md

Generate a root `README.md` with **exactly** the following structure, headings and content (wording may be lightly adapted, but every section, table, command, and note below must be present, in this order):

1. **`# Inventory & Order Management System (Sample App)`** — intro: a sample Jakarta EE 8 / EJB inventory and order management system, built for deployment to **WildFly**, with an **AngularJS 1.x** single-page frontend. Second paragraph: deliberately built with Java 11 and the pre-Jakarta-9 `javax.*` namespace, and deliberately pins several **deprecated / end-of-life (EOL) third-party libraries** on both the backend and frontend, so that modernization tooling (e.g. **AWS Transform Continuous Modernisation**) can detect and process them.
2. **`## Architecture`** — "Single **EAR** deployable containing two modules", then bullets for `inventory-ejb` (Stateless Session Beans `javax.ejb.*`, JPA entities `javax.persistence.*`, JAX-RS `javax.ws.rs.*` endpoints backed by embedded H2; the JAX-RS resources are themselves `@Stateless` EJBs so the container injects their `@EJB` dependencies), `inventory-web` (AngularJS 1.x SPA WAR consuming the JAX-RS APIs), `inventory-ear` (bundles both plus the deployable `META-INF/inventory-ds.xml`).
   - **`### Backend services`** — table `Service | EJB | REST base path`: Product / Inventory → `ProductService` → `/api/products`; Customer → `CustomerService` → `/api/customers`; Order → `OrderService` → `/api/orders`; Billing / Invoice → `InvoiceService` → `/api/invoices`; Notification → `NotificationService` → `/api/notifications`; Reporting / Alerts → `ReportingService` → `/api/reports`. Followed by a sentence on the `@Singleton @Startup` `DataSeederBean` seeding products and customers.
   - **`### Frontend`** — AngularJS 1.5.x SPA under `inventory-web/src/main/webapp`, `ngRoute` routing, modules Dashboard, Products, Customers, Orders, Invoices, Notifications; vendored libs under `webapp/lib/`, also described in `webapp/bower.json` (Bower was the standard AngularJS-era package manager).
   - **`### Persistence`** — persistence unit `inventoryPU` is JTA-based on `java:/InventoryDS`, defined by `inventory-ear/.../META-INF/inventory-ds.xml`, pointing at an in-memory H2 database (show the JDBC URL in a code block); schema generated with `hibernate.hbm2ddl.auto=create-drop`, so **data is reset every time the app is deployed or the server restarts**.
3. **`## Deliberately deprecated / EOL dependencies`** — table `Library | Version | Status | Where it's used`:
   | Library | Version | Status | Where it's used |
   |---|---|---|---|
   | Log4j | 1.2.17 | EOL (2015), superseded by Log4j 2 | `LoggingUtil`, used across all services |
   | Apache Commons Lang | 2.6 | Superseded by `commons-lang3` | `ProductService`, `CustomerService`, `InvoiceNumberGenerator` |
   | Jackson Databind | 2.9.10 | Old line, multiple known CVEs | `InvoiceService` (audit snapshot JSON) |
   | Apache Commons HttpClient | 3.1 | EOL, superseded by HttpComponents HttpClient | `NotificationService` (webhook dispatch) |
   | javax.xml.bind / JAXB RI | 2.3.0 | Standalone legacy JAXB impl (removed from JDK) | All JPA entities (`@XmlRootElement`) |
   | H2 Database | 1.4.197 | Old 1.4.x line, multiple known CVEs | `inventory-ejb` dependency, bundled in the EAR (in-memory database) |
   | JUnit | 4.11 | Old 4.x line | All unit tests |
   | AngularJS | 1.5.11 | EOL since January 2022 | Entire frontend |
   | jQuery | 1.12.4 | Old 1.x line | Frontend (Bootstrap dependency) |
   | Bootstrap | 3.3.7 | Old 3.x line | Frontend styling |
   | Bower | — | Deprecated package manager | `webapp/bower.json` |

   Then a note that the backend uses the **Jakarta EE 8 (`javax.*`) namespace** rather than Jakarta EE 9+ (`jakarta.*`), so a future upgrade requires a package rename across the EJB module — another common modernization-tooling scenario. Then a `---` separator.
4. **`## Running locally`** — opens with a short note: the code base was generated and verified by build and unit tests only (`mvn clean install`). It was **not** deployed to an application server, so installing WildFly, deploying the EAR and checking the app at runtime are the user's job, using the steps below. Then numbered subsections:
   - **`### 1. Prerequisites`** — table `Tool | Version | Notes`: JDK **11** (must be Java 11; build targets `maven.compiler.source/target=11`); Maven 3.6+ (3.9.x verified); WildFly **18–26** (any release still on Jakarta EE 8 / `javax.*`; steps use **WildFly 18.0.1.Final**; do **not** use 27+, which switched to `jakarta.*`). Then a `java -version` / `mvn -version` code block.
   - **`### 2. Install WildFly`** — run from the project root, unpacking into **`./wildfly-local`**:
     ```bash
     mkdir -p ./wildfly-local
     curl -L https://download.jboss.org/wildfly/18.0.1.Final/wildfly-18.0.1.Final.zip -o ./wildfly-local/wildfly-18.0.1.Final.zip
     unzip -q ./wildfly-local/wildfly-18.0.1.Final.zip -d ./wildfly-local
     ```
     A blockquote: the official host is `download.jboss.org`; GitHub Releases does **not** host the distribution `.zip`, so a `github.com/.../releases/download/...` link will 404; a `.tar.gz` is available at the same path. Then `export WILDFLY_HOME=$(pwd)/wildfly-local/wildfly-18.0.1.Final` (with a comment that `$(pwd)` keeps it absolute). Then an **H2 driver** blockquote: no extra setup — WildFly ships `com.h2database.h2`, which `inventory-ds.xml` references; no need to add a module or edit `standalone.xml`.
   - **`### 3. Build`** — `mvn clean install` from the project root; mention it builds all three modules, runs the JUnit 4 + Mockito unit tests in `inventory-ejb` (covering all six services and all six REST resources), and produces `inventory-ear/target/inventory.ear`; then `mvn clean install -DskipTests`.
   - **`### 4. Deploy`** — deployment-scanner model; start `$WILDFLY_HOME/bin/standalone.sh` (leave it running), in another terminal `cp inventory-ear/target/inventory.ear $WILDFLY_HOME/standalone/deployments/`; explain the `inventory.ear.deployed` / `inventory.ear.failed` markers; **to redeploy**, copy the new EAR over the old one.
   - **`### 5. Open the app`** — table `What | URL`: Frontend `http://localhost:8080/inventory-web/`, REST API base `http://localhost:8080/inventory-web/api/`; then a note that JAX-RS is activated via `@ApplicationPath("/api")` in the EJB module and exposed through the web module's context root `/inventory-web`.
   - **`### 6. Smoke-test the API`** — a bash block with `BASE=http://localhost:8080/inventory-web/api`, commented groups: seeded data (`curl -s $BASE/products`, `/customers`, `/reports/low-stock-alerts`), place an order (`curl -s -X POST $BASE/orders -H "Content-Type: application/json" -d '{"customerId":1,"lineItems":[{"productId":1,"quantity":2}]}'`), generate an invoice (`curl -s -X POST $BASE/invoices/generate/1`). Then: all endpoints should return `200`/`201`/`202`; orders, invoices and notifications start empty (only products and customers are seeded). Then a short **"Check the deployment"** list for the user:
     - the list endpoints (`GET $BASE/products`, `/customers`, `/orders`, `/invoices`, `/notifications`) return `200`;
     - product items contain a `lowStock` key and customer items a `fullName` key, but no `orderLineItems` or `orders` key. This shows Jackson, not JSON-B, serialized them (see Troubleshooting);
     - `curl -s -X POST $BASE/reports/low-stock-alerts/dispatch -o /dev/null -w '%{http_code}'` prints `202`;
     - the frontend at `http://localhost:8080/inventory-web/` loads the dashboard with the seeded low-stock alerts (SKU-1003, SKU-1005).
   - **`### Stopping / undeploying`** — stop with `Ctrl-C` (or `$WILDFLY_HOME/bin/jboss-cli.sh --connect command=:shutdown`); undeploy without stopping by `rm $WILDFLY_HOME/standalone/deployments/inventory.ear`.
   - **`### Troubleshooting`** — table `Symptom | Cause / fix` with rows: `inventory.ear.failed` marker appears (check the server console stack trace); REST `500` with `NullPointerException` on a service field (resource not managed as an EJB — all resources must be `@Stateless`); `500` with `LazyInitializationException` / Jackson `JsonMappingException` (relationship serialized after the transaction closed — lazy back-references are `@JsonIgnore`, `Order.customer` / `Invoice.order` are EAGER); list endpoints `500` with `RESTEASY008205: JSON Binding serialization error` (RESTEasy fell back to JSON-B, which ignores `@JsonIgnore` — `web.xml` must set `resteasy.preferJacksonOverJsonB=true`); port 8080 already in use (stop the other process or `standalone.sh -Djboss.socket.binding.port-offset=100`, app then on `:8180`); data empty after restart (expected — in-memory H2 with `create-drop`, `DataSeederBean` reseeds on deploy).
5. **`## Project layout`** — a code block:
   ```
   inventory-management-parent/   (root pom, dependency management)
   ├── inventory-ejb/     EJB module: entities, services, REST resources, tests
   ├── inventory-web/     WAR module: AngularJS 1.x SPA
   └── inventory-ear/     EAR module: packaging + WildFly datasource descriptor
   ```

### .gitignore

Copy `references/templates/root/gitignore.txt` to `.gitignore`.

## 12. Acceptance checklist

Build and tests:
- [ ] `mvn clean install` passes on JDK 11 and produces `inventory-ear/target/inventory.ear`; all tests pass, none skipped.
- [ ] Every behaviour listed in `TEST_PLAN.md` is covered by at least one test, in one test class per service and per resource.

Legacy stack (the modernization targets):
- [ ] All EE imports are `javax.*` (zero `jakarta.*` imports).
- [ ] The POMs, descriptors, `index.html`, `bower.json` and `css/app.css` are the templates, unmodified; the 5 vendored libraries are the real files from `VENDORED_LIBRARIES.md`.
- [ ] Log4j 1.x, Commons Lang 2.x, Jackson, Commons HttpClient 3.1 and JAXB are actually used where §5–§6 say.

Runnable on WildFly (checked statically):
- [ ] All JAX-RS resources are `@Stateless`; every `@EJB` field is typed with a `…Local` interface.
- [ ] `@JsonIgnore` is on `Customer.orders`, `Product.orderLineItems`, `OrderLineItem.order`, `OrderLineItem.product`; `web.xml` is unmodified (it carries `resteasy.preferJacksonOverJsonB=true`).
- [ ] Resources unwrap `EJBException` where they map `IllegalArgumentException` to 400 (pitfall P2).
- [ ] Every named query and inline JPQL refers to existing entity names and fields; every `mappedBy` names an existing field; no table or column is a SQL reserved word; every `@PersistenceContext` uses `unitName = "inventoryPU"`.
- [ ] Every file `index.html` loads exists at that path; each route in `app.config.js` points to an existing partial and a registered controller; each SPA service URL matches an endpoint in §7.

Documentation:
- [ ] Root `README.md` follows the §11 structure, including the "not deployed" note and the "Check the deployment" list; `.gitignore` is the template.

Runtime checks are **out of scope for the generation run**. Do not install WildFly, start a server or deploy the EAR. The user deploys to WildFly 18–26 and runs the README's smoke test and "Check the deployment" list.
