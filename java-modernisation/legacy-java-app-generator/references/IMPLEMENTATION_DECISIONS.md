# Implementation Decisions and Known Pitfalls

`RECREATE_SPEC.md` is the specification. This file records what was learned when a reference implementation was
built and deployed on WildFly 18.0.1.Final: failures that only show up at runtime, and the gaps in the spec that
the tests depend on. Apply every item below unless it contradicts an explicit sentence in `RECREATE_SPEC.md`.

---

## A. Runtime pitfalls (verified failures and their fixes)

The build and the unit tests catch none of these. They only fail once the EAR is deployed.

### P1: RESTEasy uses JSON-B for `List` responses → `@JsonIgnore` ignored → `LazyInitializationException`
- **Symptom:** every list endpoint returns `500` with `RESTEASY008205: JSON Binding serialization error … Unable to
  serialize property 'orderLineItems' … LazyInitializationException`. Single-object endpoints work because they go
  through Jackson.
- **Cause:** WildFly 18–26 registers both Jackson and JSON-B (Yasson) providers; RESTEasy picks JSON-B for generic
  collection types, and JSON-B ignores `com.fasterxml.jackson.annotation.JsonIgnore`.
- **Fix:** the `web.xml` template sets `resteasy.preferJacksonOverJsonB=true`. Copy it unmodified.
- **Rejected alternatives:** `@JsonbTransient` on entities, a `jboss-deployment-structure.xml` excluding JSON-B, or
  making associations EAGER.

### P2: System exceptions get wrapped when they cross the EJB boundary
- A `RuntimeException` thrown by a `@Stateless` service called from a `@Stateless` resource reaches the resource as
  `javax.ejb.EJBTransactionRolledbackException` (a subclass of `EJBException`) with the original exception as its
  cause. A plain `catch (IllegalArgumentException e)` therefore **never fires in the container**. It only fires in
  unit tests that use mocks.
- **Fix:** wherever a resource maps `IllegalArgumentException` → 400 (`OrderResource.placeOrder`,
  `OrderResource.updateStatus`, `InvoiceResource.generateInvoice`), catch both:
  ```java
  } catch (IllegalArgumentException e) {
      return badRequest(e);
  } catch (EJBException e) {
      // A system exception crossing the EJB boundary arrives wrapped by the container
      if (e.getCause() instanceof IllegalArgumentException) {
          return badRequest(e.getCause());
      }
      throw e;
  }
  ```
  where `badRequest` builds the 400 response described in D10.
- **Accepted consequence:** insufficient stock (`IllegalStateException`) returns **500**, because only
  `IllegalArgumentException` maps to 400.

### P3: Read-only JSON properties must not be sent back to the server
- Jackson serializes the helper getters as `lowStock` (Product), `fullName` (Customer) and `subtotal`
  (OrderLineItem). RESTEasy's Jackson provider rejects unknown properties when it deserializes, and the spec
  forbids `@JsonIgnoreProperties`. If the SPA sends back the object it received, saving a product or customer fails.
- **Fix:** the product and customer detail controllers send a payload containing **only the editable fields**
  (`sku, name, description, unitPrice, quantityOnHand, reorderThreshold` /
  `firstName, lastName, email, phone, shippingAddress`).

### P4: Route ordering
- `GET /products/low-stock` must be its own `@Path("/low-stock")` method. JAX-RS gives the literal path precedence
  over `@Path("/{id}")`. No regex is needed.
- In `app.config.js`, declare `/products/new` **before** `/products/:productId` (and the same for customers), so
  ngRoute matches the literal route first.

---

## B. Backend decisions

| # | Area | Decision |
|---|---|---|
| D01 | `OrderService.placeOrder` | Throw `IllegalArgumentException` when: the request is null or `customerId` is null; `lineItems` is null or empty; the customer is unknown; a line has a null `productId` or a quantity ≤ 0; a product is unknown. Build each line, add its subtotal, then call `productService.adjustStock(productId, -quantity)`. Call `em.persist(order)` **after** the loop, so a stock failure never persists. Then send an `EMAIL` order confirmation to the customer's email. |
| D02 | `OrderService.updateStatus` | Null status → `IllegalArgumentException`. Unknown order → return `null` (the resource returns 404). On `SHIPPED`, send an `EMAIL` shipment notification. |
| D03 | `OrderService.cancelOrder` | Unknown order → `IllegalArgumentException`. Already `CANCELLED` → no-op (prevents restocking twice). Otherwise call `adjustStock(productId, +quantity)` for each line, then set the status to `CANCELLED`. |
| D04 | `ProductService.adjustStock` | Unknown product → `IllegalArgumentException`. If the new quantity would be < 0 → `IllegalStateException`, leaving the stock unchanged. |
| D05 | Create validation | `ProductService.createProduct`: Commons Lang `Validate.notNull(product)`, `notEmpty(sku)`, `notEmpty(name)`, `notNull(unitPrice)`. `CustomerService.createCustomer`: `notNull`, `notEmpty(firstName/lastName/email)`. `NotificationService.send`: `notNull(channel)`, `notEmpty(recipient)`. These throw `IllegalArgumentException`, which the tests expect. |
| D06 | Update / delete | `update*` loads with `em.find`; if missing → `null`; otherwise copy the editable fields onto the managed entity and return it. `delete*` removes the entity if found and is a silent no-op otherwise (the resource always returns 204). |
| D07 | `InvoiceService` | Unknown order → `IllegalArgumentException`. Idempotency check through the `Invoice.findByOrder` named query (`NoResultException` → none). `tax = subtotal × 0.08` with `setScale(2, HALF_UP)`; `subtotal` = order total (null → ZERO). Audit snapshot: log `MAPPER.writeValueAsString(invoice)` from a `private static final ObjectMapper`; catch `JsonProcessingException` and log a warning. |
| D08 | `NotificationService` webhook | Hold the placeholder URL (e.g. `http://localhost:9090/notifications/webhook`) in a **non-final private instance field named `webhookUrl`**, so tests can redirect it by reflection. Commons HttpClient 3.1: `new HttpClient()`, connection and socket timeouts of 2000 ms, a `PostMethod` with form parameters, success = HTTP 2xx, `catch (Exception)` → log a warning and record `false`, `releaseConnection()` in `finally`. Short timeouts matter: the webhook is normally unreachable, and a long timeout would stall every order placement. |
| D09 | `ReportingService` | `getSalesSummary`: load `Order.findAll` and sum the non-null totals in Java. Average = `divide(count, 2, HALF_UP)`, ZERO when there are no orders. `pendingInvoices` = `SELECT COUNT(i) FROM Invoice i WHERE i.paymentStatus = :status` (`Long.class`, PENDING). `lowStockProductCount` and the alerts come from `productService.findLowStock()`. Dispatch: one `EMAIL` to `inventory-ops@example.com` whose body lists each alert's SKU; return without sending when there are no alerts. |
| D10 | Error body | 400 responses carry `Collections.singletonMap("error", message)`. The SPA reads `response.data.error`. |
| D11 | Resources | Lists use `Response.ok(list)`; create uses `Response.status(CREATED).entity(x)`; dispatch uses `Response.accepted()`; delete/cancel use `Response.noContent()`. `NotificationResource.sendTestNotification` sends on the `SYSTEM_LOG` channel and returns the log entity with 201. `OrderResource.listOrders`: `customerId` takes precedence over `status`, otherwise `findAll`. |
| D12 | Entities | Mapping annotations go on fields. Initialize collections to `new ArrayList<>()` (avoids NPEs when building orders). Put `@JsonIgnore` on the field. `Invoice.order` uses `@OneToOne @JoinColumn(name="ORDER_ID", nullable=false, unique=true)`. |

## C. Frontend decisions

| # | Area | Decision |
|---|---|---|
| F01 | Controller names | `ProductListController`, `ProductDetailController`, `CustomerListController`, `CustomerDetailController`, `OrderListController`, `OrderNewController`, `InvoiceListController`, `NotificationListController`, `DashboardController`. The route definitions in `app.config.js` must use these same names. |
| F02 | `$http` calls with no body | `PUT` needs a data argument: `$http.put(baseUrl + '/' + id + '/status', null, { params: { value: status } })`. Body-less POSTs are `$http.post(url)`. Both work with the class-level `@Consumes(APPLICATION_JSON)`, because RESTEasy accepts a request with no `Content-Type`. Do not remove the class-level `@Consumes`. |

## D. Test decisions

| # | Decision |
|---|---|
| T01 | Static imports from `org.junit.Assert` and from `org.mockito.Mockito` / `org.mockito.ArgumentMatchers`. Mocked `TypedQuery`s stub `setParameter(anyString(), any())` to return the same query. |
| T02 | `NotificationServiceTest` sets `webhookUrl` to `http://127.0.0.1:1/notifications/webhook` (connection refused, so failure). The success test starts `com.sun.net.httpserver.HttpServer` bound to `InetAddress.getLoopbackAddress()` on port 0, answers 200, and calls `stop(0)` in `finally`. **Never bind test servers to a wildcard address.** |
| T03 | BigDecimal assertions use `compareTo(...) == 0`, except where the scale is part of the contract (`averageOrderValue` = `new BigDecimal("58.34")` for totals 100.00 + 50.00 + 25.01). |
