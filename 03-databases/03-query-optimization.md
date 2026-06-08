# Query Optimization — Indexes, EXPLAIN, and N+1

## The Problem

Slow queries kill performance. The usual culprits: missing indexes, N+1 queries, and unnecessary data loading.

## Index Types

| Index | Best For |
|-------|----------|
| B-tree (default) | Equality and range queries (=, <, >, BETWEEN) |
| Hash | Exact equality only |
| Composite | Queries filtering on multiple columns |
| Partial | Indexing a subset of rows |

```sql
-- Single column index
CREATE INDEX idx_products_name ON products(name);

-- Composite index (column order matters — most selective first)
CREATE INDEX idx_orders_customer_date ON orders(customer_id, created_at);

-- Partial index
CREATE INDEX idx_orders_active ON orders(status)
WHERE status IN ('PENDING', 'PROCESSING');
```

## EXPLAIN ANALYZE

Always measure before optimizing:

```sql
EXPLAIN ANALYZE
SELECT * FROM orders WHERE customer_id = 42;

-- Look for:
-- Seq Scan  → full table scan (bad for large tables)
-- Index Scan → uses an index (good)
-- Bitmap Scan → uses index, then fetches rows (acceptable)
```

```java
@Service
@RequiredArgsConstructor
public class OrderQueryService {
    private final EntityManager entityManager;

    public List<Map<String, Object>> explainQuery(String sql) {
        var query = entityManager.createNativeQuery(
            "EXPLAIN ANALYZE " + sql);
        return query.getResultList().stream()
            .map(row -> Map.of("detail", row.toString()))
            .toList();
    }
}
```

## The N+1 Problem

One query to fetch orders, then N additional queries to fetch each order's items.

```java
// BAD: N+1 queries
@Query("SELECT o FROM Order o WHERE o.status = :status")
List<Order> findByStatus(@Param("status") String status);
// For each order, Hibernate fires: SELECT * FROM order_items WHERE order_id = ?
```

### Fix 1: JOIN FETCH

```java
@Query("SELECT o FROM Order o JOIN FETCH o.items WHERE o.status = :status")
List<Order> findByStatusWithItems(@Param("status") String status);
```

One query with a JOIN. All data fetched in a single round trip.

### Fix 2: @EntityGraph

```java
@Entity
@Table(name = "orders")
@NamedEntityGraph(
    name = "Order.withItems",
    attributeNodes = @NamedAttributeNode("items")
)
public class Order {
    @OneToMany(mappedBy = "order", fetch = FetchType.LAZY)
    private List<OrderItem> items;
}
```

```java
@EntityGraph(value = "Order.withItems")
@Query("SELECT o FROM Order o WHERE o.status = :status")
List<Order> findByStatusWithItems(@Param("status") String status);
```

`@EntityGraph` fetches specified associations eagerly for that query only, without changing the entity's default `LAZY` fetch strategy.

### Fix 3: Batch Fetching

```java
@Entity
@BatchSize(size = 50)
public class Order {
    @OneToMany(mappedBy = "order", fetch = FetchType.LAZY)
    private List<OrderItem> items;
}
```

Hibernate loads lazy associations in batches: `SELECT * FROM order_items WHERE order_id IN (?, ?, ..., ?)`. Instead of 100 queries for 100 orders, you get 2 queries (1 for orders + 1 batch for items).

## Practical Example: Fix Slow Order Listing

```java
// BEFORE: 3 queries per page (orders + customers + items)
@GetMapping("/orders")
public Page<OrderResponse> list(Pageable pageable) {
    return repository.findAll(pageable).map(this::toResponse);
}

// AFTER: 1 query with everything needed
@EntityGraph(value = "Order.withItemsAndCustomer")
@Query("SELECT o FROM Order o WHERE o.status <> 'DELETED'")
Page<Order> findAllWithDetails(Pageable pageable);
```

```java
@NamedEntityGraph(
    name = "Order.withItemsAndCustomer",
    attributeNodes = {
        @NamedAttributeNode("items"),
        @NamedAttributeNode("customer")
    }
)
```

## Key Points

- Add indexes for columns used in WHERE, JOIN, and ORDER BY
- Run `EXPLAIN ANALYZE` before and after changes — measure, don't guess
- N+1 is the most common performance killer in JPA — detect it with SQL logging
- Use `JOIN FETCH` for specific queries, `@EntityGraph` for reusable fetch plans
- `@BatchSize` is a safety net for lazy loading across the board
