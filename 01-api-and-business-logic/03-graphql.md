# GraphQL — Client-Driven API Queries

## Why GraphQL

REST returns fixed data structures. GraphQL lets the client specify exactly what it needs — no over-fetching, no under-fetching. One endpoint, flexible queries.

```
REST:    GET /products → returns everything the server decides
GraphQL: POST /graphql → client picks fields, relations, depth
```

## When GraphQL vs REST

| GraphQL | REST |
|---------|------|
| Complex nested data | Simple resources |
| Multiple frontends with different needs | Single consumer |
| Mobile clients (save bandwidth) | Server-to-server |
| Rapidly evolving frontend | Stable API contract |

## Step 1: Setup Spring for GraphQL

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-graphql</artifactId>
</dependency>
```

## Step 2: Define the Schema

```graphql
# src/main/resources/graphql/schema.graphqls
type Query {
    product(id: ID!): Product
    products(category: String, page: Int, size: Int): ProductPage
}

type Mutation {
    createProduct(input: ProductInput!): Product
    updateProduct(id: ID!, input: ProductInput!): Product
}

type Product {
    id: ID!
    name: String!
    price: Float!
    category: Category!
    reviews: [Review!]!
}

type Category {
    id: ID!
    name: String!
}

type Review {
    id: ID!
    rating: Int!
    comment: String!
    author: String!
}

type ProductPage {
    content: [Product!]!
    totalElements: Int!
    totalPages: Int!
}

input ProductInput {
    name: String!
    price: Float!
    categoryId: ID!
}
```

## Step 3: Implement Controllers

```java
@Controller
@RequiredArgsConstructor
public class ProductGraphQLController {
    private final ProductService productService;
    private final CategoryService categoryService;
    private final ReviewService reviewService;

    @QueryMapping
    public Product product(@Argument Long id) {
        return productService.findById(id);
    }

    @QueryMapping
    public ProductPage products(
            @Argument String category,
            @Argument Integer page,
            @Argument Integer size) {
        return productService.findAll(category,
            PageRequest.of(page != null ? page : 0, size != null ? size : 20));
    }

    @MutationMapping
    public Product createProduct(@Argument ProductInput input) {
        return productService.create(input);
    }

    @SchemaMapping
    public Category category(Product product) {
        return categoryService.findById(product.getCategoryId());
    }

    @SchemaMapping
    public List<Review> reviews(Product product) {
        return reviewService.findByProductId(product.getId());
    }
}
```

## Step 4: Fix the N+1 Problem with DataLoader

Without DataLoader, fetching reviews for 10 products fires 11 queries (1 for products + 10 for reviews). DataLoader batches them into 2 queries.

```java
@Controller
@RequiredArgsConstructor
public class ProductGraphQLController {
    private final ReviewService reviewService;

    @SchemaMapping
    public CompletableFuture<List<Review>> reviews(
            Product product,
            DataLoader<Long, List<Review>> reviewDataLoader) {
        return reviewDataLoader.load(product.getId());
    }

    @Bean
    public BatchLoaderRegistry batchLoaderRegistry() {
        var registry = new DefaultBatchLoaderRegistry();
        registry.forTypePair(Long.class, List.class)
            .withName("reviews")
            .registerBatchLoader((productIds, env) ->
                Mono.just(reviewService.findByProductIds(productIds)));
        return registry;
    }
}
```

```java
@Service
@RequiredArgsConstructor
public class ReviewService {
    private final ReviewRepository repository;

    public List<Review> findByProductIds(Collection<Long> productIds) {
        return repository.findByProductIdIn(productIds);
    }
}
```

## Testing with cURL

```bash
# Only fetch name and price — no wasted bandwidth
curl -X POST http://localhost:8080/graphql \
  -H "Content-Type: application/json" \
  -d '{"query": "{ product(id: 1) { name price } }"}'

# Fetch with nested relations
curl -X POST http://localhost:8080/graphql \
  -H "Content-Type: application/json" \
  -d '{"query": "{ product(id: 1) { name reviews { rating comment } } }"}'
```

GraphQL excels when your frontend team needs different slices of the same data. One schema, infinite query shapes. Use DataLoader whenever you resolve a list field — N+1 kills performance silently.
