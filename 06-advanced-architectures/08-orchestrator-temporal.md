# Orchestrator — Temporal for Durable Execution

## Why Temporal

Distributed workflows fail halfway. The payment succeeded but shipping failed. What state are you in? Who retries? Temporal provides durable execution — your workflow code is guaranteed to complete, even if servers crash.

```
Traditional: Write state to DB, poll, retry, handle failures manually
Temporal:    Write workflow code, Temporal handles durability, retries, timeouts
```

## Core Concepts

- **Workflow**: Durable function that orchestrates activities. Never loses state.
- **Activity**: A unit of work (API call, DB write). Can fail and retry.
- **Worker**: Executes workflows and activities. Connects to the Temporal server.

## Step 1: Add Temporal Dependencies

```xml
<dependency>
    <groupId>io.temporal</groupId>
    <artifactId>temporal-sdk</artifactId>
    <version>1.24.0</version>
</dependency>
```

## Step 2: Define Workflow Interface

```java
@WorkflowInterface
public interface OrderWorkflow {
    @WorkflowMethod
    OrderResult processOrder(OrderRequest request);

    @SignalMethod
    void cancelOrder(String reason);

    @QueryMethod
    String getStatus();
}
```

## Step 3: Define Activity Interface

```java
@ActivityInterface
public interface OrderActivities {
    String reserveInventory(OrderRequest request);
    PaymentResult processPayment(PaymentRequest request);
    ShippingResult scheduleShipping(ShippingRequest request);
    void cancelReservation(String reservationId);
    void refundPayment(String paymentId);
}
```

## Step 4: Implement Activities

```java
@Component
@RequiredArgsConstructor
public class OrderActivitiesImpl implements OrderActivities {
    private final InventoryClient inventoryClient;
    private final PaymentClient paymentClient;
    private final ShippingClient shippingClient;

    @Override
    public String reserveInventory(OrderRequest request) {
        return inventoryClient.reserve(
            request.productId(), request.quantity());
    }

    @Override
    @ActivityMethod
    public PaymentResult processPayment(PaymentRequest request) {
        return paymentClient.charge(
            request.customerId(), request.amount());
    }

    @Override
    public ShippingResult scheduleShipping(ShippingRequest request) {
        return shippingClient.schedule(
            request.orderId(), request.address());
    }

    @Override
    public void cancelReservation(String reservationId) {
        inventoryClient.cancel(reservationId);
    }

    @Override
    public void refundPayment(String paymentId) {
        paymentClient.refund(paymentId);
    }
}
```

## Step 5: Implement Workflow (Saga Orchestration)

```java
public class OrderWorkflowImpl implements OrderWorkflow {
    private final OrderActivities activities = Workflow
        .newActivityStub(OrderActivities.class,
            ActivityOptions.newBuilder()
                .setStartToCloseTimeout(Duration.ofSeconds(30))
                .setRetryOptions(RetryOptions.newBuilder()
                    .setMaximumAttempts(3)
                    .setInitialInterval(Duration.ofSeconds(1))
                    .setBackoffCoefficient(2.0)
                    .build())
                .build());

    private String status = "STARTED";
    private boolean cancelled = false;
    private String cancellationReason;

    @Override
    public OrderResult processOrder(OrderRequest request) {
        String reservationId = null;
        String paymentId = null;

        try {
            status = "RESERVING_INVENTORY";
            reservationId = activities.reserveInventory(request);

            if (cancelled) {
                activities.cancelReservation(reservationId);
                status = "CANCELLED";
                return new OrderResult("CANCELLED", cancellationReason);
            }

            status = "PROCESSING_PAYMENT";
            var payment = activities.processPayment(
                new PaymentRequest(request.customerId(),
                    request.amount()));
            paymentId = payment.paymentId();

            status = "SCHEDULING_SHIPPING";
            var shipping = activities.scheduleShipping(
                new ShippingRequest(request.orderId(),
                    request.address()));

            status = "COMPLETED";
            return new OrderResult("COMPLETED", shipping.trackingId());

        } catch (Exception e) {
            if (paymentId != null) activities.refundPayment(paymentId);
            if (reservationId != null)
                activities.cancelReservation(reservationId);
            status = "FAILED";
            return new OrderResult("FAILED", e.getMessage());
        }
    }

    @Override
    public void cancelOrder(String reason) {
        this.cancelled = true;
        this.cancellationReason = reason;
    }

    @Override
    public String getStatus() { return status; }
}
```

## Step 6: Spring Boot Worker

```java
@Configuration
@RequiredArgsConstructor
public class TemporalConfig {
    private final OrderActivitiesImpl orderActivities;

    @Bean
    public WorkflowClient workflowClient() {
        var service = WorkflowServiceStubs.newLocalServiceStubs();
        var client = WorkflowClient.newInstance(service);
        return client;
    }

    @Bean
    public WorkerFactory workerFactory(WorkflowClient client) {
        return WorkerFactory.newInstance(client);
    }

    @PostConstruct
    public void startWorkers() {
        var worker = workerFactory(workflowClient())
            .newWorker("order-task-queue");
        worker.registerWorkflowImplementationTypes(
            OrderWorkflowImpl.class);
        worker.registerActivitiesImplementations(orderActivities);
        workerFactory(workflowClient()).start();
    }
}
```

## Key Points

- Temporal guarantees your workflow completes — server crashes are transparent
- Activities retry automatically with configurable backoff
- Compensating actions in the catch block provide saga rollback
- Signals (`cancelOrder`) let you interact with running workflows
- Queries (`getStatus`) let you inspect workflow state without affecting execution
