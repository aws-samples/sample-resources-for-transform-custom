# Unit Test Plan

This is the **minimum set of behaviours** to test: 12 test classes, about 69 tests. The method names below are
suggestions in the `methodName_condition_expectedResult` style. You may rename, split or add tests, as long as every
behaviour below is covered and the whole suite passes. Never delete, skip or weaken a test to make the build pass;
fix the production code instead.

All tests live in `inventory-ejb/src/test/java`. They use JUnit 4.11 (`@Test`, `@Before`, `org.junit.Assert`) and
Mockito 3.12.4. Use the plain style below rather than `@RunWith(MockitoJUnitRunner)`, `@Mock` or `@InjectMocks`: in
`@Before public void setUp() throws Exception`, create mocks with `mock(...)`, instantiate the class under test with
`new`, and inject the mocks into its private fields with:

```java
private static void setField(Object target, String fieldName, Object value) throws Exception {
    java.lang.reflect.Field field = target.getClass().getDeclaredField(fieldName);
    field.setAccessible(true);
    field.set(target, value);
}
```

Mock `TypedQuery` only inside small `@SuppressWarnings("unchecked")` helper methods (`TypedQuery<X> q = mock(TypedQuery.class)`).

## Services (package `com.example.inventory.service`) — 35

### ProductServiceTest (7) — injects `em`
| Method | Asserts |
|---|---|
| `createProduct_withValidProduct_persistsAndReturnsIt` | returns the same instance; `verify(em).persist(product)` |
| `createProduct_withMissingSku_throws` | `@Test(expected = IllegalArgumentException.class)` (Commons Lang `Validate`) |
| `updateProduct_whenMissing_returnsNull` | `em.find` → null ⇒ null |
| `findBySku_whenNoResult_returnsNull` | `getSingleResult` throws `NoResultException` ⇒ null |
| `findLowStock_usesLowStockNamedQuery` | `em.createNamedQuery("Product.findLowStock", Product.class)` result is returned |
| `adjustStock_withNegativeDelta_decrementsStock` | 150 − 5 ⇒ 145 |
| `adjustStock_whenInsufficientStock_throws` | 5 − 6 ⇒ `IllegalStateException`, stock stays 5 |

### CustomerServiceTest (6) — injects `em`
`createCustomer_withValidCustomer_persistsAndReturnsIt`, `createCustomer_withMissingEmail_throws` (email `""`),
`updateCustomer_whenExisting_copiesFields`, `updateCustomer_whenMissing_returnsNull`,
`findByEmail_whenNoResult_returnsNull`, `deleteCustomer_whenExisting_removesIt` (`verify(em).remove`).

### OrderServiceTest (6) — injects `em`, `productService`, `customerService`, `notificationService`
Fixture: customer 1 (Alice); product 1 = SKU-1001 @ 19.99, product 2 = SKU-1002 @ 79.99.
| Method | Asserts |
|---|---|
| `placeOrder_withValidRequest_decrementsStockAndComputesTotal` | lines (1×2, 2×1): status NEW, total 119.97, `unitPriceAtOrder` 19.99, `adjustStock(1L,-2)`, `adjustStock(2L,-1)`, persist, EMAIL to alice |
| `placeOrder_withNoLineItems_throws` | `IllegalArgumentException` |
| `placeOrder_withNoCustomer_throws` | `IllegalArgumentException` |
| `placeOrder_whenInsufficientStock_throws` | `doThrow(IllegalStateException).when(productService).adjustStock(1L,-500)` ⇒ propagates; never persist; never send |
| `cancelOrder_restocksLineItemsAndMarksCancelled` | `adjustStock(1L,+3)`, `adjustStock(2L,+1)`, status CANCELLED |
| `updateStatus_toShipped_sendsShipmentNotification` | status SHIPPED, EMAIL sent, no stock change |

### InvoiceServiceTest (5) — injects `em`
Fixture: order 5 with total 100.00; `Invoice.findByOrder` query stubbed in `setUp`.
| Method | Asserts |
|---|---|
| `generateInvoice_computesEightPercentTax` | subtotal 100.00, tax 8.00, total 108.00, PENDING, number matches `INV-\d{8}-\d{6}`, persisted |
| `generateInvoice_whenInvoiceExists_returnsExisting` | same instance returned; `never().persist` |
| `generateInvoice_withUnknownOrder_throws` | `IllegalArgumentException` |
| `markPaid_setsPaymentStatusToPaid` | PAID |
| `markPaid_whenMissing_returnsNull` | null |

### NotificationServiceTest (5) — injects `em`, `webhookUrl`
| Method | Asserts |
|---|---|
| `send_withSystemLogChannel_persistsSuccessfulLog` | successful, channel, recipient, sentDate set, persisted |
| `send_withEmailWhenWebhookUnavailable_recordsFailure` | webhook `http://127.0.0.1:1/...` ⇒ `successful == false`, persisted |
| `send_withSmsWhenWebhookAccepts_recordsSuccess` | loopback `HttpServer` on port 0 returns 200 ⇒ successful |
| `send_withNullChannel_throws` | `IllegalArgumentException` |
| `findAll_returnsNamedQueryResults` | `NotificationLog.findAll` result is returned |

### ReportingServiceTest (6) — injects `em`, `productService`, `notificationService`
| Method | Asserts |
|---|---|
| `getSalesSummary_aggregatesOrderTotals` | orders 100.00 + 50.00 + 25.01 ⇒ count 3, revenue 175.01, average `58.34`; pending 2; low stock 1 |
| `getSalesSummary_withNoOrders_returnsZeroAverage` | count 0, revenue 0, average 0 |
| `getLowStockAlerts_mapsLowStockProducts` | DTO fields `productId, sku, productName, quantityOnHand, reorderThreshold` mapped |
| `getLowStockAlerts_withNoLowStock_returnsEmptyList` | empty |
| `dispatchLowStockAlerts_withAlerts_sendsEmailToOps` | `send(EMAIL, "inventory-ops@example.com", anyString(), contains("SKU-1003"))` |
| `dispatchLowStockAlerts_withNoAlerts_sendsNothing` | `never().send(...)` |

## REST resources (package `com.example.inventory.rest`) — 34
Each test injects the resource's single service field. `jersey-common` 2.29.1 (test scope) lets `Response` be built outside a container.

| Class (count) | Methods |
|---|---|
| ProductResourceTest (6) | `listProducts_returnsOkWithAllProducts` (200), `listLowStock_returnsOkWithLowStockProducts` (200), `getProduct_whenMissing_returnsNotFound` (404), `createProduct_withValidProduct_returnsCreated` (201), `updateProduct_whenMissing_returnsNotFound` (404), `deleteProduct_whenCalled_returnsNoContent` (204 + verify) |
| CustomerResourceTest (6) | `listCustomers_returnsOkWithAllCustomers`, `getCustomer_whenFound_returnsOk`, `getCustomer_whenMissing_returnsNotFound`, `createCustomer_withValidCustomer_returnsCreated`, `updateCustomer_whenFound_returnsOk`, `deleteCustomer_whenCalled_returnsNoContent` |
| OrderResourceTest (6) | `listOrders_withStatusFilter_returnsOrdersWithStatus` (200), `getOrder_whenMissing_returnsNotFound` (404), `placeOrder_withValidRequest_returnsCreated` (201), `placeOrder_withInvalidRequest_returnsBadRequest` (400, entity map `error`), `updateStatus_whenMissing_returnsNotFound` (404), `cancelOrder_whenCalled_returnsNoContent` (204) |
| InvoiceResourceTest (6) | `listInvoices_withoutStatus_returnsAllInvoices`, `listInvoices_withStatus_filtersByPaymentStatus`, `getInvoice_whenMissing_returnsNotFound`, `generateInvoice_withExistingOrder_returnsCreated` (201), `generateInvoice_withUnknownOrder_returnsBadRequest` (400), `markPaid_whenMissing_returnsNotFound` (404) |
| NotificationResourceTest (5) | `listNotifications_withLogs_returnsOk`, `listNotifications_whenEmpty_returnsOkWithEmptyList`, `sendTestNotification_whenCalled_returnsCreated` (201), `sendTestNotification_whenCalled_usesSystemLogChannel`, `sendTestNotification_whenCalled_returnsLogEntity` |
| ReportResourceTest (5) | `getSalesSummary_whenCalled_returnsOk`, `getSalesSummary_whenCalled_returnsSummaryEntity`, `getLowStockAlerts_whenCalled_returnsOkWithAlerts`, `dispatchLowStockAlerts_whenCalled_returnsAccepted` (202), `dispatchLowStockAlerts_whenCalled_delegatesToService` |

Reference total: 7+6+6+5+5+6 + 6+6+6+6+5+5 = 69. A different total is fine if every behaviour above is covered.
