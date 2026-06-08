# Batch vs One-by-One — Bulk Database Operations

## The Problem

Inserting 10,000 rows one at a time means 10,000 network round trips to the database. Batch operations send multiple rows in a single round trip.

## Performance Comparison

| Approach | 10,000 rows | Round trips |
|----------|-------------|-------------|
| Individual saves | ~30 seconds | 10,000 |
| JdbcTemplate batch | ~1.5 seconds | ~100 |
| JPA batch flush | ~3 seconds | ~100 |

## Step 1: JdbcTemplate Batch Update

```java
@Repository
@RequiredArgsConstructor
public class ProductBulkRepository {
    private final JdbcTemplate jdbcTemplate;

    public int[] batchInsert(List<ProductCsv> products) {
        return jdbcTemplate.batchUpdate(
            "INSERT INTO products (name, price, category) VALUES (?, ?, ?)",
            products, 100,
            (ps, product) -> {
                ps.setString(1, product.name());
                ps.setBigDecimal(2, product.price());
                ps.setString(3, product.category());
            });
    }

    public int[] batchUpdate(List<ProductPriceUpdate> updates) {
        return jdbcTemplate.batchUpdate(
            "UPDATE products SET price = ? WHERE id = ?",
            updates, 100,
            (ps, update) -> {
                ps.setBigDecimal(1, update.price());
                ps.setLong(2, update.id());
            });
    }
}
```

The second parameter (`100`) is the batch size. JdbcTemplate sends 100 rows per round trip.

## Step 2: JPA Batch Insert

```yaml
# application.yml
spring:
  jpa:
    properties:
      hibernate:
        jdbc:
          batch_size: 50
        order_inserts: true
        order_updates: true
```

```java
@Service
@RequiredArgsConstructor
public class ProductImportService {
    private final EntityManager entityManager;

    @Transactional
    public void batchInsert(List<Product> products) {
        for (int i = 0; i < products.size(); i++) {
            entityManager.persist(products.get(i));
            if (i % 50 == 0) {
                entityManager.flush();
                entityManager.clear();
            }
        }
    }
}
```

`flush()` sends the batch to the database. `clear()` detaches entities from the persistence context, freeing memory. Without `clear()`, all 10,000 entities stay in memory.

## Step 3: The Service Layer

```java
@Service
@RequiredArgsConstructor
public class BulkProductService {
    private final ProductBulkRepository bulkRepo;
    private final ProductRepository productRepo;

    @Transactional
    public BulkImportResult importProducts(List<ProductCsv> csvRows) {
        var valid = new ArrayList<ProductCsv>();
        var skipped = new ArrayList<SkippedRow>();

        for (var row : csvRows) {
            if (row.price().compareTo(BigDecimal.ZERO) <= 0) {
                skipped.add(new SkippedRow(row.name(), "Invalid price"));
                continue;
            }
            valid.add(row);
        }

        var results = bulkRepo.batchInsert(valid);
        var inserted = Arrays.stream(results).sum();
        return new BulkImportResult(inserted, skipped);
    }
}

public record BulkImportResult(int inserted, List<SkippedRow> skipped) {}
public record SkippedRow(String name, String reason) {}
```

## When Batch vs One-by-One

| Batch | One-by-One |
|-------|------------|
| Bulk import (CSV, migration) | Single entity with complex validation |
| Periodic sync jobs | User-triggered single action |
| 100+ rows at once | < 10 rows |
| Data transformations | Business logic per entity |

Batch bypasses most of JPA's entity lifecycle. You get performance but lose:
- Automatic `@PrePersist` / `@PostPersist` callbacks
- Cascade operations
- Automatic dirty checking

Use JdbcTemplate for raw performance when you don't need JPA features. Use JPA batch when you need entity lifecycle events.

## Key Points

- Always batch bulk operations — the performance difference is 10-20x
- Set `hibernate.jdbc.batch_size` to match your chunk size
- Call `flush()` and `clear()` periodically to avoid OutOfMemoryError
- Use JdbcTemplate for pure data operations, JPA batch when you need entity features
- Validate before batching — skip invalid rows rather than failing the whole batch
