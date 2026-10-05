# SALESTORM – SOLID Principle Mapping

> Deliverable 13 · Owner: LLD & Design Engineer · Reference: `03_LLD/Class_Diagram.png`

| Principle | Where it appears (actual classes) | What it buys us |
|---|---|---|
| **S – Single Responsibility** | `ReservationService` (reserve/confirm/release only) · `ReservationExpiryJob` (expiry only) · `PaymentService` (payment lifecycle) · `PaymentReconciler` (resolving unknown outcomes) · `OrderService` (order lifecycle) · `NotificationListener` (messaging) · `WebhookVerifier` (signature check) · `OutboxEventPublisher` (event persistence) | Each class has one reason to change. Changing SMS provider never touches payment code; changing expiry policy never touches the reserve path. |
| **O – Open/Closed** | `PricingStrategy` → `FlashSalePricing`, `CouponPricing`, `RegularPricing` · `PaymentGateway` → `RazorpayAdapter`, `StripeAdapter` · `DeliveryPartner` → `DelhiveryAdapter`, `ShiprocketAdapter` · `OrderEventListener` observers | **New pricing rule / payment provider / courier = new class**, registered in config/factory. `CheckoutFacade`, `PaymentService`, `FulfilmentListener` are not modified. |
| **L – Liskov Substitution** | Any `PaymentGateway` (incl. `CircuitBreakerGateway` decorator) is usable wherever `PaymentGateway` is expected; every `OrderState` honours `next()` / `canCancel()` contract; `RedisLuaStockGate` vs a test `InMemoryStockGate` | Contract rules: adapters must be idempotent on `merchantRef`, must map provider errors to our `GatewayStatus` enum, must never throw provider-specific exceptions. Substitution doesn't break callers or tests. |
| **I – Interface Segregation** | Small focused interfaces: `StockGate` (2 methods), `ReservationRepository`, `InventoryRepository` (separate), `PaymentUseCase` vs `PaymentGateway` (business API vs provider API), `ProcessedEventStore` (1 method), `DomainEventPublisher` (1 method) | No class depends on methods it doesn't use. `ReservationExpiryJob` only sees `ReservationUseCase`, not the repositories. Webhook controller doesn't see refund APIs. |
| **D – Dependency Inversion** | `ReservationService` depends on `StockGate`, `InventoryRepository`, `DomainEventPublisher` (abstractions), not Redis/JDBC/Kafka classes. `CheckoutFacade` depends on `ReservationUseCase`, `OrderUseCase`, `PaymentUseCase`, `PricingStrategy`. `PaymentService` depends on `PaymentGateway`, obtained from `PaymentGatewayFactory`. Concrete wiring done by Spring DI. | High-level business policy is independent of infrastructure → swap Redis for another gate, Kafka for SQS, Razorpay for Stripe without touching business logic; unit tests use in-memory fakes. |

## Example – adding a new payment provider ("PayU") with minimal change

1. Create `PayUAdapter implements PaymentGateway` (maps PayU API ↔ our `ChargeRequest/ChargeResponse/GatewayStatus`).
2. Register in `PaymentGatewayFactory` (`Provider.PAYU → PayUAdapter`), add config + secrets.
3. **No change** to `PaymentService`, `CheckoutFacade`, `PaymentReconciler`, events, DB schema.

## Example – new pricing strategy ("Buy-2-Get-10%-off")

`class BundlePricing implements PricingStrategy` + register in `PricingEngine` strategy map keyed by `sale.pricingType`. `CheckoutFacade` unchanged.

## Example – new delivery partner

`class BlueDartAdapter implements DeliveryPartner` + routing rule in `FulfilmentListener` (Strategy by pincode/weight). Order and Payment untouched.

## Code sketch (Java) – DIP + OCP in the reservation path

```java
public interface StockGate {
    GateResult tryAcquire(UUID saleId, UUID customerId, int qty, String idemKey);
    void giveBack(UUID saleId, UUID customerId, int qty);
}

@Service
public class ReservationService implements ReservationUseCase {
    private final StockGate gate;                       // abstraction (DIP)
    private final InventoryRepository inventory;        // abstraction (DIP)
    private final ReservationRepository reservations;   // focused interface (ISP)
    private final DomainEventPublisher events;          // outbox-backed

    @Transactional
    public ReservationResult reserve(ReserveCommand cmd) {
        var existing = reservations.findByIdempotencyKey(cmd.idemKey());
        if (existing.isPresent()) return ReservationResult.replay(existing.get());

        var gateResult = gate.tryAcquire(cmd.saleId(), cmd.customerId(), cmd.qty(), cmd.idemKey());
        if (!gateResult.ok()) return ReservationResult.rejected(gateResult.reason());

        try {
            if (!inventory.reserveAtomically(cmd.saleId(), cmd.qty())) {   // conditional UPDATE
                gate.giveBack(cmd.saleId(), cmd.customerId(), cmd.qty());
                return ReservationResult.rejected(Reason.SOLD_OUT);
            }
            var r = reservations.save(Reservation.create(cmd, clock.instant().plus(holdTime)));
            events.publish(new ReservationCreated(r));                      // same transaction
            return ReservationResult.created(r);
        } catch (DuplicateKeyException e) {
            gate.giveBack(cmd.saleId(), cmd.customerId(), cmd.qty());
            return ReservationResult.replay(reservations.findByIdempotencyKey(cmd.idemKey()).orElseThrow());
        }
    }
}
```
