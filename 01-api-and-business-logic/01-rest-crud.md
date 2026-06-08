# REST CRUD — Build a Complete API

## The Pattern

Every backend starts with CRUD. Spring Boot makes this straightforward with `@RestController`, but doing it *right* means DTOs, validation, proper HTTP status codes, and clean error handling.

## Step 1: Entity and DTO

```java
@Entity
@Table(name = "products")
public class Product {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;
    private BigDecimal price;
    private Integer stock;
}

public record ProductRequest(
    @NotBlank String name,
    @NotNull @DecimalMin("0.01") BigDecimal price,
    @NotNull @Min(0) Integer stock
) {}

public record ProductResponse(Long id, String name, BigDecimal price, Integer stock) {}
```

Always separate your internal entity from the API contract. The DTO pattern prevents over-posting and decouples your API from your persistence model.

## Step 2: Repository and Service

```java
@Repository
public interface ProductRepository extends JpaRepository<Product, Long> {
    boolean existsByName(String name);
}

@Service
@RequiredArgsConstructor
public class ProductService {
    private final ProductRepository repository;

    public ProductResponse create(ProductRequest request) {
        if (repository.existsByName(request.name())) {
            throw new DuplicateResourceException("Product already exists");
        }
        var product = new Product();
        product.setName(request.name());
        product.setPrice(request.price());
        product.setStock(request.stock());
        var saved = repository.save(product);
        return new ProductResponse(saved.getId(), saved.getName(),
            saved.getPrice(), saved.getStock());
    }

    public Page<ProductResponse> list(Pageable pageable) {
        return repository.findAll(pageable)
            .map(p -> new ProductResponse(p.getId(), p.getName(),
                p.getPrice(), p.getStock()));
    }

    public ProductResponse get(Long id) {
        return repository.findById(id)
            .map(p -> new ProductResponse(p.getId(), p.getName(),
                p.getPrice(), p.getStock()))
            .orElseThrow(() -> new ResourceNotFoundException("Product not found"));
    }
}
```

## Step 3: Controller with Proper Error Handling

```java
@RestController
@RequestMapping("/api/products")
@RequiredArgsConstructor
public class ProductController {
    private final ProductService service;

    @PostMapping
    public ResponseEntity<ProductResponse> create(
            @Valid @RequestBody ProductRequest request) {
        return ResponseEntity.status(HttpStatus.CREATED)
            .body(service.create(request));
    }

    @GetMapping
    public ResponseEntity<Page<ProductResponse>> list(Pageable pageable) {
        return ResponseEntity.ok(service.list(pageable));
    }

    @GetMapping("/{id}")
    public ResponseEntity<ProductResponse> get(@PathVariable Long id) {
        return ResponseEntity.ok(service.get(id));
    }

    @PutMapping("/{id}")
    public ResponseEntity<ProductResponse> update(
            @PathVariable Long id,
            @Valid @RequestBody ProductRequest request) {
        return ResponseEntity.ok(service.update(id, request));
    }

    @DeleteMapping("/{id}")
    public ResponseEntity<Void> delete(@PathVariable Long id) {
        service.delete(id);
        return ResponseEntity.noContent().build();
    }
}
```

## Step 4: Global Exception Handler

```java
@RestControllerAdvice
public class ApiExceptionHandler {
    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(ResourceNotFoundException ex) {
        return ResponseEntity.status(HttpStatus.NOT_FOUND)
            .body(new ErrorResponse("NOT_FOUND", ex.getMessage()));
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidation(
            MethodArgumentNotValidException ex) {
        var errors = ex.getBindingResult().getFieldErrors().stream()
            .map(e -> e.getField() + ": " + e.getDefaultMessage())
            .toList();
        return ResponseEntity.badRequest()
            .body(new ErrorResponse("VALIDATION_FAILED", String.join("; ", errors)));
    }

    @ExceptionHandler(DuplicateResourceException.class)
    public ResponseEntity<ErrorResponse> handleDuplicate(DuplicateResourceException ex) {
        return ResponseEntity.status(HttpStatus.CONFLICT)
            .body(new ErrorResponse("DUPLICATE", ex.getMessage()));
    }
}

public record ErrorResponse(String code, String message) {}
```

## Key Takeaways

- Use `record` for DTOs — immutable and concise
- `@Valid` triggers Bean Validation on input
- Return `ResponseEntity` to control status codes explicitly
- Centralize error handling with `@RestControllerAdvice`
- Use `Page<T>` for list endpoints — clients get pagination metadata for free
