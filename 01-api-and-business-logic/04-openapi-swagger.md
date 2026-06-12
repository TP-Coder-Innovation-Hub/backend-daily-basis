# OpenAPI / Swagger — API Documentation

> **Last verified:** June 2026 — springdoc 2.3.0

## Why Document Your API

Your API is a contract. Without documentation, consumers guess. OpenAPI (formerly Swagger) generates interactive docs directly from your Spring Boot code.

## Step 1: Add SpringDoc OpenAPI

```xml
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
    <version>2.3.0</version>
</dependency>
```

After adding this dependency, navigate to `http://localhost:8080/swagger-ui.html` for interactive docs and `http://localhost:8080/v3/api-docs` for the JSON spec.

## Step 2: Annotate Your Controllers

```java
@RestController
@RequestMapping("/api/products")
@Tag(name = "Products", description = "Product management APIs")
@RequiredArgsConstructor
public class ProductController {
    private final ProductService service;

    @Operation(summary = "Get product by ID",
        description = "Returns a single product matching the given ID")
    @ApiResponses({
        @ApiResponse(responseCode = "200",
            description = "Product found"),
        @ApiResponse(responseCode = "404",
            description = "Product not found",
            content = @Content(schema = @Schema(
                implementation = ErrorResponse.class)))
    })
    @GetMapping("/{id}")
    public ResponseEntity<ProductResponse> get(
            @Parameter(description = "Product ID", example = "1")
            @PathVariable Long id) {
        return ResponseEntity.ok(service.get(id));
    }

    @Operation(summary = "Create a new product")
    @PostMapping
    public ResponseEntity<ProductResponse> create(
            @Valid @RequestBody ProductRequest request) {
        return ResponseEntity.status(HttpStatus.CREATED)
            .body(service.create(request));
    }

    @Operation(summary = "List products with pagination")
    @GetMapping
    public ResponseEntity<Page<ProductResponse>> list(
            @Parameter(description = "Page number (0-based)")
            @RequestParam(defaultValue = "0") int page,
            @Parameter(description = "Page size")
            @RequestParam(defaultValue = "20") int size) {
        return ResponseEntity.ok(service.list(PageRequest.of(page, size)));
    }
}
```

## Step 3: Annotate DTOs with Schema Metadata

```java
@Schema(description = "Request body for creating or updating a product")
public record ProductRequest(
    @Schema(description = "Product name", example = "Wireless Mouse",
        requiredMode = Schema.RequiredMode.REQUIRED)
    @NotBlank String name,

    @Schema(description = "Price in USD", example = "29.99",
        requiredMode = Schema.RequiredMode.REQUIRED)
    @NotNull @DecimalMin("0.01") BigDecimal price,

    @Schema(description = "Available stock", example = "100")
    @Min(0) Integer stock
) {}

@Schema(description = "Product response")
public record ProductResponse(
    @Schema(description = "Unique identifier", example = "1")
    Long id,
    @Schema(description = "Product name", example = "Wireless Mouse")
    String name,
    @Schema(description = "Price in USD", example = "29.99")
    BigDecimal price,
    @Schema(description = "Available stock", example = "100")
    Integer stock
) {}
```

## Step 4: Global API Info

```java
@Configuration
public class OpenApiConfig {
    @Bean
    public OpenAPI customOpenAPI() {
        return new OpenAPI()
            .info(new Info()
                .title("Product Service API")
                .version("1.0")
                .description("REST API for managing products")
                .contact(new Contact()
                    .name("Backend Team")
                    .email("backend@example.com")))
            .addSecurityItem(new SecurityRequirement().addList("bearerAuth"))
            .components(new Components()
                .addSecuritySchemes("bearerAuth",
                    new SecurityScheme()
                        .type(SecurityScheme.Type.HTTP)
                        .scheme("bearer")
                        .bearerFormat("JWT")));
    }
}
```

## API-First Approach

Instead of generating docs from code, start with the spec file:

```yaml
# api-spec/openapi.yaml — write this FIRST
openapi: 3.1.0
info:
  title: Product Service
  version: 1.0.0
paths:
  /api/products:
    get:
      summary: List products
      parameters:
        - name: page
          in: query
          schema:
            type: integer
            default: 0
      responses:
        '200':
          description: Paginated product list
```

Generate server stubs from the spec:

```bash
# Using openapi-generator Maven plugin
mvn openapi-generator:generate \
  -Dopenapi.generator.inputSpec=api-spec/openapi.yaml \
  -Dopenapi.generator.generatorName=spring
```

API-first means teams agree on the contract before implementation. Frontend can mock the spec while backend builds against it. Code-first is faster to start — use it for internal services.
