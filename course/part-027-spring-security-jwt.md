# Part 027: Spring Security JWT Authentication

JWT (JSON Web Token) is a compact, self-contained token format used for stateless authentication. Instead of server-side sessions, the token itself contains the user information and is verified with a signature.

---

## 1. JWT Structure

```
JWT = Base64(Header) . Base64(Payload) . Signature

Header (algorithm):
{
  "alg": "HS256",
  "typ": "JWT"
}

Payload (claims):
{
  "sub": "42",           // Subject (user id)
  "username": "jsmith",
  "roles": ["USER"],
  "iat": 1700000000,     // Issued at
  "exp": 1700003600      // Expiry (1 hour later)
}

Signature:
HMACSHA256(base64(header) + "." + base64(payload), secret)
```

---

## 2. Maven Dependencies

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-security</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>

    <!-- JWT library - jjwt -->
    <dependency>
        <groupId>io.jsonwebtoken</groupId>
        <artifactId>jjwt-api</artifactId>
        <version>0.12.3</version>
    </dependency>
    <dependency>
        <groupId>io.jsonwebtoken</groupId>
        <artifactId>jjwt-impl</artifactId>
        <version>0.12.3</version>
        <scope>runtime</scope>
    </dependency>
    <dependency>
        <groupId>io.jsonwebtoken</groupId>
        <artifactId>jjwt-jackson</artifactId>
        <version>0.12.3</version>
        <scope>runtime</scope>
    </dependency>

    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <optional>true</optional>
    </dependency>
</dependencies>
```

---

## 3. JWT Configuration Properties

```yaml
# application.yml
app:
  jwt:
    secret: myVeryLongSecretKeyForJWTSigningThatIsAtLeast256BitsLongForSecurity
    expiration: 86400000          # 24 hours in milliseconds
    refresh-expiration: 604800000  # 7 days in milliseconds
    issuer: blog-api

spring:
  datasource:
    url: jdbc:h2:mem:authdb;DB_CLOSE_DELAY=-1
  jpa:
    hibernate:
      ddl-auto: create-drop
    show-sql: true
```

### JWT Properties Class

```java
package com.example.auth.config;

import lombok.Getter;
import lombok.Setter;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.stereotype.Component;

@Component
@ConfigurationProperties(prefix = "app.jwt")
@Getter @Setter
public class JwtProperties {
    private String secret;
    private long expiration;         // Access token expiry (ms)
    private long refreshExpiration;  // Refresh token expiry (ms)
    private String issuer;
}
```

---

## 4. JWT Token Service

```java
package com.example.auth.security;

import com.example.auth.config.JwtProperties;
import com.example.auth.entity.User;
import io.jsonwebtoken.*;
import io.jsonwebtoken.io.Decoders;
import io.jsonwebtoken.security.Keys;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.security.core.GrantedAuthority;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.stereotype.Service;

import javax.crypto.SecretKey;
import java.util.*;
import java.util.function.Function;
import java.util.stream.Collectors;

@Service
@RequiredArgsConstructor
@Slf4j
public class JwtTokenService {

    private final JwtProperties jwtProperties;

    // Generate signing key from configured secret
    private SecretKey getSigningKey() {
        byte[] keyBytes = Decoders.BASE64.decode(
            Base64.getEncoder().encodeToString(
                jwtProperties.getSecret().getBytes()
            )
        );
        return Keys.hmacShaKeyFor(keyBytes);
    }

    // ================================
    // Access Token
    // ================================

    public String generateAccessToken(UserDetails userDetails) {
        Map<String, Object> claims = new HashMap<>();

        // Add custom claims
        if (userDetails instanceof UserPrincipal principal) {
            claims.put("userId", principal.getId());
            claims.put("email", principal.getEmail());
        }

        // Add roles as claim
        List<String> roles = userDetails.getAuthorities().stream()
            .map(GrantedAuthority::getAuthority)
            .collect(Collectors.toList());
        claims.put("roles", roles);
        claims.put("tokenType", "ACCESS");

        return buildToken(claims, userDetails, jwtProperties.getExpiration());
    }

    // ================================
    // Refresh Token
    // ================================

    public String generateRefreshToken(UserDetails userDetails) {
        Map<String, Object> claims = new HashMap<>();
        claims.put("tokenType", "REFRESH");

        if (userDetails instanceof UserPrincipal principal) {
            claims.put("userId", principal.getId());
        }

        return buildToken(claims, userDetails, jwtProperties.getRefreshExpiration());
    }

    // ================================
    // Token Builder
    // ================================

    private String buildToken(Map<String, Object> extraClaims, UserDetails userDetails, long expiration) {
        Date now = new Date(System.currentTimeMillis());
        Date expiryDate = new Date(System.currentTimeMillis() + expiration);

        return Jwts.builder()
            .claims(extraClaims)
            .subject(userDetails.getUsername())
            .issuer(jwtProperties.getIssuer())
            .issuedAt(now)
            .expiration(expiryDate)
            .id(UUID.randomUUID().toString())  // jti claim - unique token id
            .signWith(getSigningKey())
            .compact();
    }

    // ================================
    // Token Parsing
    // ================================

    public Claims parseToken(String token) {
        return Jwts.parser()
            .verifyWith(getSigningKey())
            .build()
            .parseSignedClaims(token)
            .getPayload();
    }

    public <T> T extractClaim(String token, Function<Claims, T> claimsResolver) {
        final Claims claims = parseToken(token);
        return claimsResolver.apply(claims);
    }

    public String extractUsername(String token) {
        return extractClaim(token, Claims::getSubject);
    }

    public Long extractUserId(String token) {
        return extractClaim(token, claims -> claims.get("userId", Long.class));
    }

    public List<String> extractRoles(String token) {
        return extractClaim(token, claims -> {
            Object roles = claims.get("roles");
            if (roles instanceof List<?> list) {
                return list.stream().map(Object::toString).toList();
            }
            return List.of();
        });
    }

    public String extractTokenType(String token) {
        return extractClaim(token, claims -> claims.get("tokenType", String.class));
    }

    public Date extractExpiration(String token) {
        return extractClaim(token, Claims::getExpiration);
    }

    // ================================
    // Token Validation
    // ================================

    public boolean isTokenValid(String token, UserDetails userDetails) {
        try {
            final String username = extractUsername(token);
            return username.equals(userDetails.getUsername()) && !isTokenExpired(token);
        } catch (Exception e) {
            log.warn("Token validation failed: {}", e.getMessage());
            return false;
        }
    }

    public boolean isTokenExpired(String token) {
        return extractExpiration(token).before(new Date());
    }

    public boolean isRefreshToken(String token) {
        return "REFRESH".equals(extractTokenType(token));
    }

    public boolean isAccessToken(String token) {
        return "ACCESS".equals(extractTokenType(token));
    }

    // Validate token structure without checking expiry
    public boolean validateTokenSignature(String token) {
        try {
            Jwts.parser()
                .verifyWith(getSigningKey())
                .build()
                .parseSignedClaims(token);
            return true;
        } catch (MalformedJwtException e) {
            log.warn("Invalid JWT token: {}", e.getMessage());
        } catch (ExpiredJwtException e) {
            log.warn("JWT token is expired: {}", e.getMessage());
        } catch (UnsupportedJwtException e) {
            log.warn("JWT token is unsupported: {}", e.getMessage());
        } catch (IllegalArgumentException e) {
            log.warn("JWT claims string is empty: {}", e.getMessage());
        } catch (Exception e) {
            log.warn("JWT validation error: {}", e.getMessage());
        }
        return false;
    }

    public long getExpirationMs() {
        return jwtProperties.getExpiration();
    }
}
```

---

## 5. UserPrincipal (with ID and email)

```java
package com.example.auth.security;

import com.example.auth.entity.User;
import lombok.Getter;
import org.springframework.security.core.GrantedAuthority;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.core.userdetails.UserDetails;

import java.util.Collection;
import java.util.stream.Collectors;

public class UserPrincipal implements UserDetails {

    @Getter private final Long id;
    @Getter private final String email;
    private final String username;
    private final String password;
    private final boolean enabled;
    private final boolean accountNonExpired;
    private final boolean accountNonLocked;
    private final boolean credentialsNonExpired;
    private final Collection<? extends GrantedAuthority> authorities;

    private UserPrincipal(User user) {
        this.id = user.getId();
        this.email = user.getEmail();
        this.username = user.getUsername();
        this.password = user.getPassword();
        this.enabled = user.isEnabled();
        this.accountNonExpired = user.isAccountNonExpired();
        this.accountNonLocked = user.isAccountNonLocked();
        this.credentialsNonExpired = user.isCredentialsNonExpired();
        this.authorities = user.getRoles().stream()
            .map(role -> new SimpleGrantedAuthority("ROLE_" + role.name()))
            .collect(Collectors.toSet());
    }

    public static UserPrincipal create(User user) {
        return new UserPrincipal(user);
    }

    @Override public Collection<? extends GrantedAuthority> getAuthorities() { return authorities; }
    @Override public String getPassword() { return password; }
    @Override public String getUsername() { return username; }
    @Override public boolean isAccountNonExpired() { return accountNonExpired; }
    @Override public boolean isAccountNonLocked() { return accountNonLocked; }
    @Override public boolean isCredentialsNonExpired() { return credentialsNonExpired; }
    @Override public boolean isEnabled() { return enabled; }
}
```

---

## 6. JWT Authentication Filter

```java
package com.example.auth.security;

import jakarta.servlet.FilterChain;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.web.authentication.WebAuthenticationDetailsSource;
import org.springframework.stereotype.Component;
import org.springframework.util.StringUtils;
import org.springframework.web.filter.OncePerRequestFilter;

import java.io.IOException;

@Component
@RequiredArgsConstructor
@Slf4j
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    private final JwtTokenService jwtTokenService;
    private final UserDetailsService userDetailsService;

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain)
            throws ServletException, IOException {

        try {
            String jwt = extractJwtFromRequest(request);

            if (StringUtils.hasText(jwt) && jwtTokenService.validateTokenSignature(jwt)) {
                // Only process access tokens in the filter
                if (!jwtTokenService.isAccessToken(jwt)) {
                    log.debug("Refresh token used in Authorization header, rejecting");
                    filterChain.doFilter(request, response);
                    return;
                }

                String username = jwtTokenService.extractUsername(jwt);

                // Only authenticate if not already authenticated
                if (username != null && SecurityContextHolder.getContext().getAuthentication() == null) {
                    UserDetails userDetails = userDetailsService.loadUserByUsername(username);

                    if (jwtTokenService.isTokenValid(jwt, userDetails)) {
                        UsernamePasswordAuthenticationToken authToken =
                            new UsernamePasswordAuthenticationToken(
                                userDetails,
                                null,
                                userDetails.getAuthorities()
                            );

                        authToken.setDetails(
                            new WebAuthenticationDetailsSource().buildDetails(request)
                        );

                        // Set authentication in security context
                        SecurityContextHolder.getContext().setAuthentication(authToken);
                        log.debug("Authenticated user: {}", username);
                    }
                }
            }
        } catch (Exception e) {
            log.error("Cannot set user authentication: {}", e.getMessage());
        }

        filterChain.doFilter(request, response);
    }

    private String extractJwtFromRequest(HttpServletRequest request) {
        String bearerToken = request.getHeader("Authorization");
        if (StringUtils.hasText(bearerToken) && bearerToken.startsWith("Bearer ")) {
            return bearerToken.substring(7);
        }
        return null;
    }

    // Skip filter for public paths
    @Override
    protected boolean shouldNotFilter(HttpServletRequest request) {
        String path = request.getServletPath();
        return path.startsWith("/api/auth/");
    }
}
```

---

## 7. Refresh Token Entity and Repository

```java
@Entity
@Table(name = "refresh_tokens")
@Getter @Setter @NoArgsConstructor @AllArgsConstructor @Builder
public class RefreshToken {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, unique = true, length = 500)
    private String token;

    @OneToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "user_id", nullable = false)
    private User user;

    @Column(nullable = false)
    private LocalDateTime expiresAt;

    @Column(nullable = false)
    @Builder.Default
    private boolean revoked = false;

    @Column(name = "created_at", nullable = false, updatable = false)
    private LocalDateTime createdAt;

    @PrePersist
    protected void onCreate() {
        createdAt = LocalDateTime.now();
    }

    public boolean isExpired() {
        return LocalDateTime.now().isAfter(expiresAt);
    }

    public boolean isValid() {
        return !revoked && !isExpired();
    }
}

@Repository
public interface RefreshTokenRepository extends JpaRepository<RefreshToken, Long> {
    Optional<RefreshToken> findByToken(String token);
    Optional<RefreshToken> findByUserId(Long userId);
    void deleteByUserId(Long userId);
    void deleteByToken(String token);
    boolean existsByUserId(Long userId);
}
```

---

## 8. Refresh Token Service

```java
@Service
@RequiredArgsConstructor
@Transactional
@Slf4j
public class RefreshTokenService {

    private final RefreshTokenRepository refreshTokenRepository;
    private final UserRepository userRepository;
    private final JwtProperties jwtProperties;

    public RefreshToken createRefreshToken(Long userId) {
        // Delete existing refresh token for user
        refreshTokenRepository.findByUserId(userId)
            .ifPresent(existingToken -> refreshTokenRepository.delete(existingToken));

        User user = userRepository.findById(userId)
            .orElseThrow(() -> new UserNotFoundException("User not found: " + userId));

        RefreshToken refreshToken = RefreshToken.builder()
            .user(user)
            .token(UUID.randomUUID().toString())  // Random UUID for refresh token
            .expiresAt(LocalDateTime.now().plusSeconds(
                jwtProperties.getRefreshExpiration() / 1000
            ))
            .revoked(false)
            .build();

        return refreshTokenRepository.save(refreshToken);
    }

    @Transactional(readOnly = true)
    public RefreshToken validateRefreshToken(String token) {
        RefreshToken refreshToken = refreshTokenRepository.findByToken(token)
            .orElseThrow(() -> new InvalidRefreshTokenException("Refresh token not found"));

        if (refreshToken.isRevoked()) {
            throw new InvalidRefreshTokenException("Refresh token has been revoked");
        }

        if (refreshToken.isExpired()) {
            refreshTokenRepository.delete(refreshToken);
            throw new InvalidRefreshTokenException("Refresh token has expired");
        }

        return refreshToken;
    }

    public void revokeRefreshToken(String token) {
        refreshTokenRepository.findByToken(token)
            .ifPresent(rt -> {
                rt.setRevoked(true);
                refreshTokenRepository.save(rt);
            });
    }

    public void revokeAllUserRefreshTokens(Long userId) {
        refreshTokenRepository.deleteByUserId(userId);
    }
}
```

---

## 9. Auth Service with JWT

```java
@Service
@RequiredArgsConstructor
@Transactional
@Slf4j
public class AuthService {

    private final UserRepository userRepository;
    private final PasswordEncoder passwordEncoder;
    private final JwtTokenService jwtTokenService;
    private final RefreshTokenService refreshTokenService;
    private final CustomUserDetailsService userDetailsService;

    public RegisterResponse register(RegisterRequest request) {
        if (userRepository.existsByUsername(request.getUsername())) {
            throw new UserAlreadyExistsException("Username already taken: " + request.getUsername());
        }
        if (userRepository.existsByEmail(request.getEmail())) {
            throw new UserAlreadyExistsException("Email already registered: " + request.getEmail());
        }

        User user = User.builder()
            .username(request.getUsername())
            .email(request.getEmail())
            .password(passwordEncoder.encode(request.getPassword()))
            .roles(Set.of(Role.USER))
            .enabled(true)
            .accountNonExpired(true)
            .accountNonLocked(true)
            .credentialsNonExpired(true)
            .build();

        User savedUser = userRepository.save(user);
        log.info("Registered new user: {}", savedUser.getUsername());

        return RegisterResponse.builder()
            .id(savedUser.getId())
            .username(savedUser.getUsername())
            .email(savedUser.getEmail())
            .message("Registration successful")
            .build();
    }

    public AuthResponse login(LoginRequest request) {
        User user = userRepository.findByUsernameOrEmail(
            request.getUsernameOrEmail(), request.getUsernameOrEmail()
        ).orElseThrow(() -> new BadCredentialsException("Invalid credentials"));

        if (!passwordEncoder.matches(request.getPassword(), user.getPassword())) {
            throw new BadCredentialsException("Invalid credentials");
        }

        if (!user.isEnabled()) {
            throw new DisabledException("Account is disabled");
        }

        UserPrincipal principal = UserPrincipal.create(user);
        String accessToken = jwtTokenService.generateAccessToken(principal);
        RefreshToken refreshToken = refreshTokenService.createRefreshToken(user.getId());

        log.info("User logged in: {}", user.getUsername());

        return AuthResponse.builder()
            .accessToken(accessToken)
            .refreshToken(refreshToken.getToken())
            .tokenType("Bearer")
            .expiresIn(jwtTokenService.getExpirationMs() / 1000)
            .userId(user.getId())
            .username(user.getUsername())
            .email(user.getEmail())
            .roles(user.getRoles())
            .build();
    }

    public AuthResponse refresh(RefreshRequest request) {
        // Validate the refresh token (not expired, not revoked)
        RefreshToken refreshToken = refreshTokenService.validateRefreshToken(request.getRefreshToken());

        User user = refreshToken.getUser();
        UserPrincipal principal = UserPrincipal.create(user);

        // Generate new access token
        String newAccessToken = jwtTokenService.generateAccessToken(principal);

        // Optionally: rotate refresh token (invalidate old, issue new)
        refreshTokenService.revokeRefreshToken(request.getRefreshToken());
        RefreshToken newRefreshToken = refreshTokenService.createRefreshToken(user.getId());

        log.info("Token refreshed for user: {}", user.getUsername());

        return AuthResponse.builder()
            .accessToken(newAccessToken)
            .refreshToken(newRefreshToken.getToken())
            .tokenType("Bearer")
            .expiresIn(jwtTokenService.getExpirationMs() / 1000)
            .userId(user.getId())
            .username(user.getUsername())
            .email(user.getEmail())
            .roles(user.getRoles())
            .build();
    }

    public void logout(String refreshToken) {
        // Revoke the refresh token
        refreshTokenService.revokeRefreshToken(refreshToken);
        log.info("User logged out, refresh token revoked");
    }

    public void logoutAll(Long userId) {
        // Revoke all refresh tokens for the user
        refreshTokenService.revokeAllUserRefreshTokens(userId);
        log.info("All tokens revoked for user ID: {}", userId);
    }
}
```

---

## 10. Auth Controller

```java
@RestController
@RequestMapping("/api/auth")
@RequiredArgsConstructor
@Slf4j
public class AuthController {

    private final AuthService authService;

    @PostMapping("/register")
    public ResponseEntity<RegisterResponse> register(
            @Valid @RequestBody RegisterRequest request) {
        RegisterResponse response = authService.register(request);
        return ResponseEntity.status(HttpStatus.CREATED).body(response);
    }

    @PostMapping("/login")
    public ResponseEntity<AuthResponse> login(
            @Valid @RequestBody LoginRequest request) {
        AuthResponse response = authService.login(request);
        return ResponseEntity.ok(response);
    }

    @PostMapping("/refresh")
    public ResponseEntity<AuthResponse> refresh(
            @Valid @RequestBody RefreshRequest request) {
        AuthResponse response = authService.refresh(request);
        return ResponseEntity.ok(response);
    }

    @PostMapping("/logout")
    public ResponseEntity<Map<String, String>> logout(
            @Valid @RequestBody LogoutRequest request) {
        authService.logout(request.getRefreshToken());
        return ResponseEntity.ok(Map.of("message", "Logged out successfully"));
    }

    @PostMapping("/logout-all")
    @PreAuthorize("isAuthenticated()")
    public ResponseEntity<Map<String, String>> logoutAll(
            @AuthenticationPrincipal UserPrincipal principal) {
        authService.logoutAll(principal.getId());
        return ResponseEntity.ok(Map.of("message", "All sessions terminated"));
    }

    @GetMapping("/me")
    @PreAuthorize("isAuthenticated()")
    public ResponseEntity<UserProfileResponse> getCurrentUser(
            @AuthenticationPrincipal UserPrincipal principal) {
        return ResponseEntity.ok(UserProfileResponse.builder()
            .id(principal.getId())
            .username(principal.getUsername())
            .email(principal.getEmail())
            .authorities(principal.getAuthorities().stream()
                .map(auth -> auth.getAuthority())
                .toList())
            .build());
    }

    @PostMapping("/change-password")
    @PreAuthorize("isAuthenticated()")
    public ResponseEntity<Map<String, String>> changePassword(
            @Valid @RequestBody ChangePasswordRequest request,
            @AuthenticationPrincipal UserPrincipal principal) {
        authService.changePassword(principal.getId(), request);
        return ResponseEntity.ok(Map.of("message", "Password changed successfully"));
    }
}
```

---

## 11. Security Configuration for JWT

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity(prePostEnabled = true)
@RequiredArgsConstructor
public class JwtSecurityConfig {

    private final CustomUserDetailsService userDetailsService;
    private final JwtAuthenticationFilter jwtAuthenticationFilter;
    private final PasswordEncoder passwordEncoder;

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            // Disable CSRF (stateless JWT does not need it)
            .csrf(csrf -> csrf.disable())

            // CORS configuration
            .cors(cors -> cors.configurationSource(corsConfigurationSource()))

            // Stateless session (no server-side sessions)
            .sessionManagement(session -> session
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            )

            // Request authorization
            .authorizeHttpRequests(auth -> auth
                // Auth endpoints - public
                .requestMatchers("/api/auth/**").permitAll()

                // Swagger/OpenAPI - public
                .requestMatchers("/swagger-ui/**", "/v3/api-docs/**").permitAll()

                // H2 Console (dev only)
                .requestMatchers("/h2-console/**").permitAll()

                // Public read endpoints
                .requestMatchers(HttpMethod.GET, "/api/posts/**").permitAll()
                .requestMatchers(HttpMethod.GET, "/api/categories/**").permitAll()
                .requestMatchers(HttpMethod.GET, "/api/tags/**").permitAll()

                // All others require authentication
                .anyRequest().authenticated()
            )

            // Custom exception handling
            .exceptionHandling(ex -> ex
                // Return 401 for unauthenticated requests (instead of redirect)
                .authenticationEntryPoint(jwtAuthEntryPoint())
                // Return 403 for forbidden requests
                .accessDeniedHandler(jwtAccessDeniedHandler())
            )

            // Add JWT filter before UsernamePasswordAuthenticationFilter
            .addFilterBefore(jwtAuthenticationFilter,
                UsernamePasswordAuthenticationFilter.class)

            // Allow H2 console frames
            .headers(headers -> headers
                .frameOptions(frame -> frame.sameOrigin())
            );

        return http.build();
    }

    @Bean
    public AuthenticationEntryPoint jwtAuthEntryPoint() {
        return (request, response, authException) -> {
            log.error("Unauthorized request: {}", authException.getMessage());
            response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
            response.setContentType(MediaType.APPLICATION_JSON_VALUE);
            response.getWriter().write("""
                {
                  "status": 401,
                  "error": "Unauthorized",
                  "message": "You need to be authenticated to access this resource"
                }
                """);
        };
    }

    @Bean
    public AccessDeniedHandler jwtAccessDeniedHandler() {
        return (request, response, accessDeniedException) -> {
            log.error("Access denied: {}", accessDeniedException.getMessage());
            response.setStatus(HttpServletResponse.SC_FORBIDDEN);
            response.setContentType(MediaType.APPLICATION_JSON_VALUE);
            response.getWriter().write("""
                {
                  "status": 403,
                  "error": "Forbidden",
                  "message": "You don't have permission to access this resource"
                }
                """);
        };
    }

    @Bean
    public AuthenticationManager authenticationManager(
            AuthenticationConfiguration config) throws Exception {
        return config.getAuthenticationManager();
    }

    @Bean
    public DaoAuthenticationProvider authenticationProvider() {
        DaoAuthenticationProvider provider = new DaoAuthenticationProvider();
        provider.setUserDetailsService(userDetailsService);
        provider.setPasswordEncoder(passwordEncoder);
        return provider;
    }

    @Bean
    public CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration config = new CorsConfiguration();
        config.setAllowedOriginPatterns(List.of("*"));
        config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE", "OPTIONS", "PATCH"));
        config.setAllowedHeaders(List.of("*"));
        config.setExposedHeaders(List.of("Authorization"));
        config.setAllowCredentials(true);
        config.setMaxAge(3600L);

        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/**", config);
        return source;
    }
}
```

---

## 12. DTOs

```java
// Register Request
@Getter @Setter @NoArgsConstructor @AllArgsConstructor @Builder
public class RegisterRequest {
    @NotBlank(message = "Username is required")
    @Size(min = 3, max = 50)
    @Pattern(regexp = "^[a-zA-Z0-9_]+$", message = "Username can only contain letters, numbers, underscores")
    private String username;

    @NotBlank(message = "Email is required")
    @Email(message = "Email must be valid")
    private String email;

    @NotBlank(message = "Password is required")
    @Size(min = 8, max = 72, message = "Password must be 8-72 characters")
    private String password;
}

// Login Request
@Getter @Setter @NoArgsConstructor @AllArgsConstructor
public class LoginRequest {
    @NotBlank(message = "Username or email is required")
    private String usernameOrEmail;

    @NotBlank(message = "Password is required")
    private String password;
}

// Auth Response (for login and refresh)
@Getter @Setter @NoArgsConstructor @AllArgsConstructor @Builder
public class AuthResponse {
    private String accessToken;
    private String refreshToken;
    private String tokenType;
    private Long expiresIn;
    private Long userId;
    private String username;
    private String email;
    private Set<Role> roles;
}

// Register Response
@Getter @Setter @NoArgsConstructor @AllArgsConstructor @Builder
public class RegisterResponse {
    private Long id;
    private String username;
    private String email;
    private String message;
}

// Refresh Request
@Getter @Setter @NoArgsConstructor @AllArgsConstructor
public class RefreshRequest {
    @NotBlank(message = "Refresh token is required")
    private String refreshToken;
}

// Logout Request
@Getter @Setter @NoArgsConstructor @AllArgsConstructor
public class LogoutRequest {
    @NotBlank(message = "Refresh token is required")
    private String refreshToken;
}

// Change Password Request
@Getter @Setter @NoArgsConstructor @AllArgsConstructor
public class ChangePasswordRequest {
    @NotBlank
    private String currentPassword;

    @NotBlank
    @Size(min = 8)
    private String newPassword;
}

// User Profile Response
@Getter @Setter @NoArgsConstructor @AllArgsConstructor @Builder
public class UserProfileResponse {
    private Long id;
    private String username;
    private String email;
    private List<String> authorities;
}
```

---

## 13. Token Blacklisting (for logout)

```java
// For truly stateless JWT logout, we use a token blacklist (Redis or in-memory)
@Service
@RequiredArgsConstructor
@Slf4j
public class TokenBlacklistService {

    // In-memory blacklist (use Redis in production)
    private final Set<String> blacklistedJtis = ConcurrentHashMap.newKeySet();

    // JTI = JWT ID - unique identifier for each token
    public void blacklistToken(String token, JwtTokenService jwtTokenService) {
        try {
            Claims claims = jwtTokenService.parseToken(token);
            String jti = claims.getId();
            Date expiry = claims.getExpiration();

            if (jti != null) {
                blacklistedJtis.add(jti);
                log.info("Token blacklisted: jti={}", jti);

                // Schedule removal after expiry to avoid memory growth
                scheduleRemoval(jti, expiry);
            }
        } catch (Exception e) {
            log.warn("Could not blacklist token: {}", e.getMessage());
        }
    }

    public boolean isBlacklisted(String token, JwtTokenService jwtTokenService) {
        try {
            Claims claims = jwtTokenService.parseToken(token);
            String jti = claims.getId();
            return jti != null && blacklistedJtis.contains(jti);
        } catch (Exception e) {
            return false;
        }
    }

    private void scheduleRemoval(String jti, Date expiry) {
        long delay = expiry.getTime() - System.currentTimeMillis();
        if (delay > 0) {
            ScheduledExecutorService scheduler = Executors.newSingleThreadScheduledExecutor();
            scheduler.schedule(() -> {
                blacklistedJtis.remove(jti);
                scheduler.shutdown();
            }, delay, TimeUnit.MILLISECONDS);
        }
    }
}

// Update JwtAuthenticationFilter to check blacklist
@Component
@RequiredArgsConstructor
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    private final JwtTokenService jwtTokenService;
    private final UserDetailsService userDetailsService;
    private final TokenBlacklistService tokenBlacklistService;

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain filterChain)
            throws ServletException, IOException {
        try {
            String jwt = extractJwtFromRequest(request);

            if (StringUtils.hasText(jwt) && jwtTokenService.validateTokenSignature(jwt)) {
                // Check if token is blacklisted
                if (tokenBlacklistService.isBlacklisted(jwt, jwtTokenService)) {
                    filterChain.doFilter(request, response);
                    return;
                }

                String username = jwtTokenService.extractUsername(jwt);

                if (username != null && SecurityContextHolder.getContext().getAuthentication() == null) {
                    UserDetails userDetails = userDetailsService.loadUserByUsername(username);

                    if (jwtTokenService.isTokenValid(jwt, userDetails)) {
                        UsernamePasswordAuthenticationToken authToken =
                            new UsernamePasswordAuthenticationToken(
                                userDetails, null, userDetails.getAuthorities()
                            );
                        authToken.setDetails(
                            new WebAuthenticationDetailsSource().buildDetails(request)
                        );
                        SecurityContextHolder.getContext().setAuthentication(authToken);
                    }
                }
            }
        } catch (Exception e) {
            log.error("Cannot set user authentication: {}", e.getMessage());
        }

        filterChain.doFilter(request, response);
    }

    private String extractJwtFromRequest(HttpServletRequest request) {
        String bearerToken = request.getHeader("Authorization");
        if (StringUtils.hasText(bearerToken) && bearerToken.startsWith("Bearer ")) {
            return bearerToken.substring(7);
        }
        return null;
    }
}
```

---

## 14. Role-Based JWT Claims

```java
// Custom JWT with role-based permissions
@Service
@RequiredArgsConstructor
public class JwtTokenService {

    public String generateAccessToken(UserDetails userDetails) {
        Map<String, Object> claims = new HashMap<>();

        if (userDetails instanceof UserPrincipal principal) {
            claims.put("userId", principal.getId());
            claims.put("email", principal.getEmail());

            // Map roles to fine-grained permissions
            Set<String> permissions = new HashSet<>();
            principal.getAuthorities().forEach(auth -> {
                String role = auth.getAuthority();
                permissions.addAll(getRolePermissions(role));
            });
            claims.put("permissions", permissions);
        }

        List<String> roles = userDetails.getAuthorities().stream()
            .map(GrantedAuthority::getAuthority)
            .toList();
        claims.put("roles", roles);
        claims.put("tokenType", "ACCESS");

        return buildToken(claims, userDetails, jwtProperties.getExpiration());
    }

    private Set<String> getRolePermissions(String role) {
        return switch (role) {
            case "ROLE_ADMIN" -> Set.of(
                "posts:read", "posts:write", "posts:delete",
                "users:read", "users:write", "users:delete",
                "comments:moderate"
            );
            case "ROLE_AUTHOR" -> Set.of(
                "posts:read", "posts:write",
                "comments:read", "comments:write"
            );
            case "ROLE_USER" -> Set.of(
                "posts:read",
                "comments:read", "comments:write"
            );
            default -> Set.of();
        };
    }
}

// Using permissions in security checks
@PreAuthorize("authentication.principal.authorities.stream()" +
              ".anyMatch(a -> a.authority == 'ROLE_ADMIN')")
public void adminOnlyAction() {}

// Or with custom permission evaluator
@PreAuthorize("hasPermission('posts', 'write')")
public Post createPost(CreatePostRequest request) {}
```

---

## 15. Exception Classes

```java
public class UserAlreadyExistsException extends RuntimeException {
    public UserAlreadyExistsException(String message) { super(message); }
}

public class UserNotFoundException extends RuntimeException {
    public UserNotFoundException(String message) { super(message); }
}

public class InvalidRefreshTokenException extends RuntimeException {
    public InvalidRefreshTokenException(String message) { super(message); }
}

// Global exception handler for auth errors
@RestControllerAdvice
@Slf4j
public class AuthExceptionHandler {

    @ExceptionHandler(UserAlreadyExistsException.class)
    public ResponseEntity<ErrorResponse> handleUserExists(UserAlreadyExistsException ex) {
        return ResponseEntity.status(HttpStatus.CONFLICT)
            .body(new ErrorResponse(409, "Conflict", ex.getMessage()));
    }

    @ExceptionHandler(BadCredentialsException.class)
    public ResponseEntity<ErrorResponse> handleBadCredentials(BadCredentialsException ex) {
        return ResponseEntity.status(HttpStatus.UNAUTHORIZED)
            .body(new ErrorResponse(401, "Unauthorized", "Invalid username or password"));
    }

    @ExceptionHandler(InvalidRefreshTokenException.class)
    public ResponseEntity<ErrorResponse> handleInvalidRefreshToken(InvalidRefreshTokenException ex) {
        return ResponseEntity.status(HttpStatus.UNAUTHORIZED)
            .body(new ErrorResponse(401, "Unauthorized", ex.getMessage()));
    }

    @ExceptionHandler(DisabledException.class)
    public ResponseEntity<ErrorResponse> handleDisabled(DisabledException ex) {
        return ResponseEntity.status(HttpStatus.FORBIDDEN)
            .body(new ErrorResponse(403, "Forbidden", "Account is disabled"));
    }
}
```

---

## 16. Complete Auth Flow Sequence

```
Client                    Server
  │                          │
  │  POST /api/auth/register │
  │─────────────────────────►│
  │                          │ Validate, Hash Password, Save User
  │  201 Created             │
  │◄─────────────────────────│
  │                          │
  │  POST /api/auth/login    │
  │─────────────────────────►│
  │  {username, password}    │ Verify, Generate Access + Refresh Token
  │                          │
  │  200 {accessToken,       │
  │       refreshToken}      │
  │◄─────────────────────────│
  │                          │
  │  GET /api/posts          │
  │  Authorization: Bearer   │
  │  <accessToken>           │
  │─────────────────────────►│
  │                          │ JwtFilter validates token
  │  200 {posts data}        │
  │◄─────────────────────────│
  │                          │
  │  [accessToken expires]   │
  │                          │
  │  POST /api/auth/refresh  │
  │  {refreshToken}          │
  │─────────────────────────►│
  │                          │ Validate refresh, Issue new access token
  │  200 {new accessToken,   │
  │       new refreshToken}  │
  │◄─────────────────────────│
  │                          │
  │  POST /api/auth/logout   │
  │  {refreshToken}          │
  │─────────────────────────►│
  │                          │ Revoke refresh token
  │  200 {message: logout}   │
  │◄─────────────────────────│
```

---

## 17. Summary Table

| Concept | Class/Method | Purpose |
|---|---|---|
| Token generation | `JwtTokenService.generateAccessToken()` | Create signed JWT |
| Token validation | `JwtTokenService.isTokenValid()` | Verify signature and expiry |
| Claims extraction | `JwtTokenService.extractClaim()` | Read data from token |
| Filter | `JwtAuthenticationFilter` | Intercept requests, set authentication |
| Refresh tokens | `RefreshTokenService` | Issue and rotate refresh tokens |
| Token storage | `RefreshToken` entity | Persist refresh tokens in DB |
| Blacklisting | `TokenBlacklistService` | Invalidate issued tokens |
| Config | `JwtProperties` | Centralize JWT configuration |
| Stateless config | `SessionCreationPolicy.STATELESS` | No server sessions |
| Login | `AuthService.login()` | Authenticate and issue tokens |
| Register | `AuthService.register()` | Create user account |
| Logout | `AuthService.logout()` | Revoke refresh token |

---

## Next Part

**Part 028** covers advanced Spring Data features: custom repository implementations, Specifications API, QueryDSL, projections, solving the N+1 problem, soft delete, optimistic locking with `@Version`, second-level caching, and database migrations with Flyway/Liquibase.
