# User Login — OAuth2 + JWT End to End

## The Flow

```
1. Client sends credentials to /api/auth/login
2. Server validates, returns access token (JWT) + refresh token
3. Client includes JWT in Authorization header for every request
4. Server validates JWT on each request via security filter
5. When JWT expires, client uses refresh token to get a new one
```

## Step 1: JWT Utility

```java
@Component
@RequiredArgsConstructor
public class JwtTokenProvider {
    @Value("${jwt.secret}")
    private String secret;

    @Value("${jwt.access-token-validity:3600000}")
    private long accessTokenValidity;

    @Value("${jwt.refresh-token-validity:86400000}")
    private long refreshTokenValidity;

    private SecretKey getSigningKey() {
        return Keys.hmacShaKeyFor(secret.getBytes(StandardCharsets.UTF_8));
    }

    public String generateAccessToken(UserDetails userDetails) {
        var claims = new HashMap<String, Object>();
        claims.put("roles", userDetails.getAuthorities().stream()
            .map(GrantedAuthority::getAuthority).toList());
        return buildToken(claims, userDetails.getUsername(),
            accessTokenValidity);
    }

    public String generateRefreshToken(String username) {
        return buildToken(new HashMap<>(), username, refreshTokenValidity);
    }

    private String buildToken(Map<String, Object> claims,
            String subject, long validity) {
        return Jwts.builder()
            .claims(claims)
            .subject(subject)
            .issuedAt(new Date())
            .expiration(new Date(System.currentTimeMillis() + validity))
            .signWith(getSigningKey())
            .compact();
    }

    public String getUsername(String token) {
        return Jwts.parser()
            .verifyWith(getSigningKey())
            .build()
            .parseSignedClaims(token)
            .getPayload()
            .getSubject();
    }

    public boolean validate(String token) {
        try {
            Jwts.parser().verifyWith(getSigningKey())
                .build().parseSignedClaims(token);
            return true;
        } catch (JwtException | IllegalArgumentException e) {
            return false;
        }
    }
}
```

## Step 2: JWT Authentication Filter

```java
@Component
@RequiredArgsConstructor
public class JwtAuthFilter extends OncePerRequestFilter {
    private final JwtTokenProvider tokenProvider;
    private final UserDetailsService userDetailsService;

    @Override
    protected void doFilterInternal(HttpServletRequest request,
            HttpServletResponse response, FilterChain chain)
            throws ServletException, IOException {
        var header = request.getHeader("Authorization");
        if (header != null && header.startsWith("Bearer ")) {
            var token = header.substring(7);
            if (tokenProvider.validate(token)) {
                var username = tokenProvider.getUsername(token);
                var userDetails = userDetailsService
                    .loadUserByUsername(username);
                var auth = new UsernamePasswordAuthenticationToken(
                    userDetails, null, userDetails.getAuthorities());
                SecurityContextHolder.getContext()
                    .setAuthentication(auth);
            }
        }
        chain.doFilter(request, response);
    }
}
```

## Step 3: Security Configuration

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity
@RequiredArgsConstructor
public class SecurityConfig {
    private final JwtAuthFilter jwtAuthFilter;

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
            .csrf(csrf -> csrf.disable())
            .sessionManagement(s ->
                s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/auth/**").permitAll()
                .anyRequest().authenticated())
            .addFilterBefore(jwtAuthFilter,
                UsernamePasswordAuthenticationFilter.class)
            .build();
    }
}
```

## Step 4: Auth Controller

```java
@RestController
@RequestMapping("/api/auth")
@RequiredArgsConstructor
public class AuthController {
    private final AuthenticationManager authManager;
    private final JwtTokenProvider tokenProvider;
    private final RefreshTokenService refreshTokenService;

    @PostMapping("/login")
    public ResponseEntity<AuthResponse> login(
            @Valid @RequestBody LoginRequest request) {
        var auth = authManager.authenticate(
            new UsernamePasswordAuthenticationToken(
                request.username(), request.password()));
        var userDetails = (UserDetails) auth.getPrincipal();
        var accessToken = tokenProvider.generateAccessToken(userDetails);
        var refreshToken = refreshTokenService.createRefreshToken(
            userDetails.getUsername());
        return ResponseEntity.ok(new AuthResponse(
            accessToken, refreshToken, "Bearer",
            tokenProvider.getAccessTokenValidity()));
    }

    @PostMapping("/refresh")
    public ResponseEntity<AuthResponse> refresh(
            @Valid @RequestBody RefreshRequest request) {
        var refreshToken = refreshTokenService
            .validateAndGet(request.refreshToken());
        var userDetails = userDetailsService
            .loadUserByUsername(refreshToken.getUsername());
        var newAccessToken = tokenProvider
            .generateAccessToken(userDetails);
        var newRefreshToken = refreshTokenService
            .rotate(refreshToken);
        return ResponseEntity.ok(new AuthResponse(
            newAccessToken, newRefreshToken.getToken(),
            "Bearer", tokenProvider.getAccessTokenValidity()));
    }
}
```

## Step 5: Refresh Token Rotation

```java
@Service
@RequiredArgsConstructor
public class RefreshTokenService {
    private final RefreshTokenRepository repository;

    public RefreshToken createRefreshToken(String username) {
        var token = new RefreshToken();
        token.setUsername(username);
        token.setToken(UUID.randomUUID().toString());
        token.setExpiresAt(Instant.now().plus(7, ChronoUnit.DAYS));
        return repository.save(token);
    }

    public RefreshToken validateAndGet(String token) {
        return repository.findByToken(token)
            .filter(t -> t.getExpiresAt().isAfter(Instant.now()))
            .orElseThrow(() -> new InvalidTokenException("Invalid refresh token"));
    }

    public RefreshToken rotate(RefreshToken oldToken) {
        repository.delete(oldToken);
        return createRefreshToken(oldToken.getUsername());
    }
}
```

Refresh token rotation: each refresh token is used once, then replaced. If a token is reused, someone stole it — invalidate all tokens for that user.

## Key Points

- Access tokens are short-lived (1 hour). Refresh tokens are long-lived (7 days).
- Never store tokens in localStorage for web apps — use HttpOnly cookies
- Rotate refresh tokens on every use to detect theft
- JWT is stateless — the server validates the signature, no DB lookup needed
- For token revocation, maintain a blacklist in Redis (check on every request)
