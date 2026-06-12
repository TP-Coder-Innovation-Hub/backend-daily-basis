# gRPC — High-Performance Service Communication

> **Last verified:** June 2026 — grpc-spring-boot-starter 3.1.0.RELEASE, grpc-protobuf 1.62.0

## Why gRPC

REST is human-readable and flexible. gRPC is fast, strongly typed, and generates client/server code for you. Use gRPC when services talk to each other internally and performance matters.

```
REST:  JSON over HTTP/1.1 → text serialization → flexible but slow
gRPC:  Protobuf over HTTP/2 → binary serialization → fast and typed
```

## When gRPC vs REST

| gRPC | REST |
|------|------|
| Internal service-to-service | Public APIs |
| Low latency required | Human-readable responses |
| Streaming data needed | Simple CRUD |
| Strong contract needed | Browser clients |

## Step 1: Define the Protocol Buffer

```protobuf
// src/main/proto/product.proto
syntax = "proto3";
package com.example.product;

option java_package = "com.example.product.grpc";

service ProductService {
  rpc GetProduct(ProductRequest) returns (ProductResponse);
  rpc ListProducts(ListRequest) returns (stream ProductResponse);
  rpc CreateProducts(stream ProductRequest) returns (BulkResponse);
}

message ProductRequest {
  int64 id = 1;
  string name = 2;
  double price = 3;
}

message ProductResponse {
  int64 id = 1;
  string name = 2;
  double price = 3;
}

message ListRequest {
  int32 page = 1;
  int32 size = 2;
}

message BulkResponse {
  int32 created = 1;
  repeated string errors = 2;
}
```

## Step 2: Add gRPC Spring Boot Starter

```xml
<dependency>
    <groupId>net.devh</groupId>
    <artifactId>grpc-spring-boot-starter</artifactId>
    <version>3.1.0.RELEASE</version>
</dependency>
<dependency>
    <groupId>io.grpc</groupId>
    <artifactId>grpc-protobuf</artifactId>
    <version>1.62.0</version>
</dependency>
```

## Step 3: Implement the Server

```java
@GrpcService
@RequiredArgsConstructor
public class ProductGrpcService extends ProductServiceGrpc.ProductServiceImplBase {
    private final ProductRepository repository;

    @Override
    public void getProduct(ProductRequest request,
            StreamObserver<ProductResponse> responseObserver) {
        var product = repository.findById(request.getId())
            .orElseThrow(() -> new StatusRuntimeException(
                Status.NOT_FOUND.withDescription("Product not found")));
        responseObserver.onNext(ProductResponse.newBuilder()
            .setId(product.getId())
            .setName(product.getName())
            .setPrice(product.getPrice().doubleValue())
            .build());
        responseObserver.onCompleted();
    }

    @Override
    public void listProducts(ListRequest request,
            StreamObserver<ProductResponse> responseObserver) {
        var page = repository.findAll(
            PageRequest.of(request.getPage(), request.getSize()));
        page.forEach(p -> responseObserver.onNext(ProductResponse.newBuilder()
            .setId(p.getId())
            .setName(p.getName())
            .setPrice(p.getPrice().doubleValue())
            .build()));
        responseObserver.onCompleted();
    }

    @Override
    public StreamObserver<ProductRequest> createProducts(
            StreamObserver<BulkResponse> responseObserver) {
        var created = new AtomicInteger(0);
        var errors = new ArrayList<String>();
        return new StreamObserver<>() {
            @Override
            public void onNext(ProductRequest request) {
                try {
                    var product = new Product();
                    product.setName(request.getName());
                    product.setPrice(BigDecimal.valueOf(request.getPrice()));
                    repository.save(product);
                    created.incrementAndGet();
                } catch (Exception e) {
                    errors.add(request.getName() + ": " + e.getMessage());
                }
            }
            @Override
            public void onError(Throwable t) {}
            @Override
            public void onCompleted() {
                responseObserver.onNext(BulkResponse.newBuilder()
                    .setCreated(created.get())
                    .addAllErrors(errors)
                    .build());
                responseObserver.onCompleted();
            }
        };
    }
}
```

## Call Types

| Type | Use Case |
|------|----------|
| Unary | Single request, single response (like REST) |
| Server streaming | Large result sets, real-time updates |
| Client streaming | File upload, bulk inserts |
| Bidirectional | Chat, real-time sync |

## Configuration

```yaml
grpc:
  server:
    port: 9090
  client:
    product-service:
      address: 'static://localhost:9090'
      negotiationType: plaintext
```

gRPC generates the client stub from the `.proto` file. No need to write HTTP client code — just call methods on the stub. The contract is the protobuf definition, not a JSON schema.
