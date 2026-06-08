# OWASP Best Practices — Securing Spring Boot APIs

## OWASP Top 10 for APIs

The most common API security vulnerabilities and how to prevent them in Spring Boot.

## 1. Injection (SQL, NoSQL, Command)

```java
// BAD: String concatenation — SQL injection
@Query("SELECT u FROM User u WHERE u.name = '" + name + "'")

// GOOD: Parameterized queries
@Query("SELECT u FROM User u WHERE u.name = :name")
List<User> findByName(@Param("name") String name);
```

JPA parameterized queries and JdbcTemplate `?` placeholders prevent injection. Never concatenate user input into queries.

## 2. Broken Authentication

- Use Spring Security — never write your own auth
- Enforce strong passwords with `PasswordPolicy`
- Implement account lockout after failed attempts
- Use MFA for admin endpoints

```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    return http
        .csrf(csrf -> csrf.disable())
        .sessionManagement(s ->
            s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
        .headers(headers -> headers
            .contentSecurityPolicy(csp ->
                csp.policyDirectives("default-src 'self'"))
            .frameOptions(HeadersConfigurer
                .FrameOptionsConfig::deny))
        .authorizeHttpRequests(auth -> auth
            .anyRequest().authenticated())
        .build();
}
```

## 3. Excessive Data Exposure

```java
// BAD: Returning the entity with all fields including internal data
@GetMapping("/{id}")
public User getUser(@PathVariable Long id) {
    return userRepository.findById(id).get();
}

// GOOD: Return a DTO with only the fields the client needs
@GetMapping("/{id}")
public UserResponse getUser(@PathVariable Long id) {
    return userService.get(id);
}
public record UserResponse(Long id, String name, String email) {}
```

Never expose entities directly. Always use DTOs.

## 4. Rate Limiting

```java
@Configuration
public class RateLimitConfig {
    @Bean
    public FilterRegistrationBean<RateLimitFilter> rateLimitFilter() {
        var registration = new FilterRegistrationBean<RateLimitFilter>();
        registration.setFilter(new RateLimitFilter());
        registration.addUrlPatterns("/api/*");
        return registration;
    }
}

@Component
@RequiredArgsConstructor
public class RateLimitFilter implements Filter {
    private final StringRedisTemplate redis;

    @Override
    public void doFilter(ServletRequest request,
            ServletResponse response, FilterChain chain)
            throws IOException, ServletException {
        var httpRequest = (HttpServletRequest) request;
        var clientId = httpRequest.getRemoteAddr();
        var key = "rate-limit:" + clientId;
        var count = redis.opsForValue().increment(key);
        if (count != null && count == 1) {
            redis.expire(key, Duration.ofMinutes(1));
        }
        if (count != null && count > 100) {
            ((HttpServletResponse) response).sendError(429, "Too many requests");
            return;
        }
        chain.doFilter(request, response);
    }
}
```

## 5. Security Headers

```java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    return http
        .headers(headers -> headers
            .xssProtection(xss -> xss.disable())
            .contentSecurityPolicy(csp ->
                csp.policyDirectives("default-src 'self'"))
            .httpStrictTransportSecurity(hsts ->
                hsts.includeSubDomains(true).maxAgeInSeconds(31536000)))
        .build();
}
```

## 6. Input Validation

```java
public record CreateProductRequest(
    @NotBlank @Size(max = 200) String name,
    @NotNull @DecimalMin("0.01") @DecimalMax("999999.99") BigDecimal price,
    @NotNull @Min(0) @Max(1000000) Integer stock,
    @Pattern(regexp = "^[a-zA-Z0-9-]+$") String sku
) {}

@RestController
public class ProductController {
    @PostMapping
    public ResponseEntity<ProductResponse> create(
            @Valid @RequestBody CreateProductRequest request) {
        return ResponseEntity.status(HttpStatus.CREATED)
            .body(service.create(request));
    }
}
```

`@Valid` triggers validation. Always validate on the server, even if the client validates too.

## 7. CORS Configuration

```java
@Bean
public CorsFilter corsFilter() {
    var config = new CorsConfiguration();
    config.setAllowedOrigins(List.of("https://app.example.com"));
    config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE"));
    config.setAllowedHeaders(List.of("Authorization", "Content-Type"));
    config.setAllowCredentials(true);
    config.setMaxAge(3600L);
    var source = new UrlBasedCorsConfigurationSource();
    source.registerCorsConfiguration("/api/**", config);
    return new CorsFilter(source);
}
```

Never use `*` for allowed origins in production.

## Security Checklist

- [ ] All endpoints require authentication (except public ones)
- [ ] DTOs used for all API responses (no entity exposure)
- [ ] Input validation with `@Valid` on all endpoints
- [ ] Parameterized queries only (no string concatenation)
- [ ] Rate limiting on public endpoints
- [ ] Security headers configured (HSTS, CSP, X-Frame-Options)
- [ ] CORS restricted to known origins
- [ ] Secrets in environment variables or Vault, not code
- [ ] HTTPS everywhere
- [ ] Audit logging for sensitive operations
