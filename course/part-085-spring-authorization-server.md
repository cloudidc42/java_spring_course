# Part 085: Spring Authorization Server (OAuth2 Server)

## Overview

Spring Authorization Server is an implementation of the OAuth 2.1 and OpenID Connect 1.0 specifications. It acts as the central identity provider for your microservices ecosystem, issuing JWTs and managing client registrations.

---

## 1. Project Setup

```xml
<!-- pom.xml -->
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-oauth2-authorization-server</artifactId>
    </dependency>
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
    <dependency>
        <groupId>org.postgresql</groupId>
        <artifactId>postgresql</artifactId>
    </dependency>
</dependencies>
```

```yaml
# application.yml
server:
  port: 9000

spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/auth_server
    username: postgres
    password: password
  jpa:
    hibernate:
      ddl-auto: validate
  flyway:
    enabled: true

app:
  auth-server:
    issuer-uri: http://localhost:9000
    access-token-validity: 1h
    refresh-token-validity: 24h
```

---

## 2. Authorization Server Configuration

```java
@Configuration
@EnableWebSecurity
public class AuthorizationServerSecurityConfiguration {

    @Bean
    @Order(1)
    public SecurityFilterChain authorizationServerSecurityFilterChain(HttpSecurity http) throws Exception {
        OAuth2AuthorizationServerConfiguration.applyDefaultSecurity(http);

        http.getConfigurer(OAuth2AuthorizationServerConfigurer.class)
                .oidc(Customizer.withDefaults());  // Enable OpenID Connect 1.0

        http
                // Redirect to login when not authenticated from authorization endpoint
                .exceptionHandling(exceptions -> exceptions
                        .defaultAuthenticationEntryPointFor(
                                new LoginUrlAuthenticationEntryPoint("/login"),
                                new MediaTypeRequestMatcher(MediaType.TEXT_HTML)
                        ))
                // Accept access tokens for User Info and Client Registration
                .oauth2ResourceServer(resourceServer -> resourceServer
                        .jwt(Customizer.withDefaults()));

        return http.build();
    }

    @Bean
    @Order(2)
    public SecurityFilterChain defaultSecurityFilterChain(HttpSecurity http) throws Exception {
        http
                .authorizeHttpRequests(authorize -> authorize
                        .requestMatchers("/login", "/error", "/actuator/health").permitAll()
                        .anyRequest().authenticated())
                .formLogin(form -> form.loginPage("/login"))
                .csrf(csrf -> csrf.ignoringRequestMatchers("/oauth2/token", "/oauth2/introspect"));

        return http.build();
    }

    @Bean
    public UserDetailsService userDetailsService(UserRepository userRepository) {
        return username -> userRepository.findByEmail(username)
                .map(user -> User.withUsername(user.getEmail())
                        .password(user.getPasswordHash())
                        .roles(user.getRoles().toArray(new String[0]))
                        .build())
                .orElseThrow(() -> new UsernameNotFoundException("User not found: " + username));
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder(12);
    }
}
```

---

## 3. Registered Clients

```java
@Configuration
public class ClientRegistrationConfiguration {

    @Bean
    public RegisteredClientRepository registeredClientRepository(JdbcTemplate jdbcTemplate) {
        // Store clients in PostgreSQL using the standard Spring schema
        return new JdbcRegisteredClientRepository(jdbcTemplate);
    }

    // Initialize default clients (use DB migrations in production)
    @Bean
    @ConditionalOnProperty("app.auth-server.seed-clients")
    public ApplicationRunner seedClients(RegisteredClientRepository clientRepository) {
        return args -> {
            // Authorization Code + PKCE client (e.g., SPA)
            if (clientRepository.findByClientId("web-app") == null) {
                RegisteredClient webApp = RegisteredClient.withId(UUID.randomUUID().toString())
                        .clientId("web-app")
                        .clientSecret("{noop}secret")  // use encoded in production
                        .clientAuthenticationMethod(ClientAuthenticationMethod.CLIENT_SECRET_BASIC)
                        .authorizationGrantType(AuthorizationGrantType.AUTHORIZATION_CODE)
                        .authorizationGrantType(AuthorizationGrantType.REFRESH_TOKEN)
                        .redirectUri("http://localhost:3000/callback")
                        .redirectUri("http://localhost:3000/silent-refresh")
                        .postLogoutRedirectUri("http://localhost:3000/logout")
                        .scope(OidcScopes.OPENID)
                        .scope(OidcScopes.PROFILE)
                        .scope(OidcScopes.EMAIL)
                        .scope("orders:read")
                        .scope("orders:write")
                        .clientSettings(ClientSettings.builder()
                                .requireAuthorizationConsent(true)
                                .requireProofKey(true)  // PKCE required
                                .build())
                        .tokenSettings(TokenSettings.builder()
                                .accessTokenTimeToLive(Duration.ofHours(1))
                                .refreshTokenTimeToLive(Duration.ofDays(1))
                                .reuseRefreshTokens(false)  // rotation
                                .build())
                        .build();
                clientRepository.save(webApp);
            }

            // Client Credentials client (e.g., backend service)
            if (clientRepository.findByClientId("order-service") == null) {
                RegisteredClient orderService = RegisteredClient.withId(UUID.randomUUID().toString())
                        .clientId("order-service")
                        .clientSecret(passwordEncoder().encode("order-service-secret"))
                        .clientAuthenticationMethod(ClientAuthenticationMethod.CLIENT_SECRET_BASIC)
                        .authorizationGrantType(AuthorizationGrantType.CLIENT_CREDENTIALS)
                        .scope("inventory:read")
                        .scope("payment:write")
                        .tokenSettings(TokenSettings.builder()
                                .accessTokenTimeToLive(Duration.ofMinutes(15))
                                .build())
                        .build();
                clientRepository.save(orderService);
            }

            // Public client (PKCE only, no client secret)
            if (clientRepository.findByClientId("mobile-app") == null) {
                RegisteredClient mobileApp = RegisteredClient.withId(UUID.randomUUID().toString())
                        .clientId("mobile-app")
                        .clientAuthenticationMethod(ClientAuthenticationMethod.NONE)  // public
                        .authorizationGrantType(AuthorizationGrantType.AUTHORIZATION_CODE)
                        .authorizationGrantType(AuthorizationGrantType.REFRESH_TOKEN)
                        .redirectUri("myapp://oauth/callback")
                        .scope(OidcScopes.OPENID)
                        .scope(OidcScopes.PROFILE)
                        .scope("orders:read")
                        .clientSettings(ClientSettings.builder()
                                .requireProofKey(true)
                                .requireAuthorizationConsent(false)
                                .build())
                        .build();
                clientRepository.save(mobileApp);
            }
        };
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

---

## 4. Authorization Code Flow with PKCE

```java
// This flow is handled automatically by the framework.
// Here's how a client (e.g., SPA) would initiate it:

// Step 1: Generate PKCE verifier and challenge in the client
/*
const codeVerifier = base64url(crypto.getRandomValues(new Uint8Array(32)));
const codeChallenge = base64url(await crypto.subtle.digest('SHA-256',
    new TextEncoder().encode(codeVerifier)));

// Step 2: Redirect to authorization endpoint
const authUrl = new URL('http://localhost:9000/oauth2/authorize');
authUrl.searchParams.set('client_id', 'web-app');
authUrl.searchParams.set('response_type', 'code');
authUrl.searchParams.set('scope', 'openid profile email orders:read');
authUrl.searchParams.set('redirect_uri', 'http://localhost:3000/callback');
authUrl.searchParams.set('code_challenge', codeChallenge);
authUrl.searchParams.set('code_challenge_method', 'S256');
authUrl.searchParams.set('state', randomState);
window.location.href = authUrl.toString();

// Step 3: Exchange code for tokens (after user authenticates)
// POST /oauth2/token
// code=<received_code>
// grant_type=authorization_code
// redirect_uri=http://localhost:3000/callback
// code_verifier=<original_verifier>
// client_id=web-app
*/

// On the server side, the AuthorizationCodeGrantAuthenticationConverter
// and AuthorizationCodeGrantAuthenticationProvider handle PKCE validation automatically.

// Custom consent page (optional)
@Controller
public class ConsentController {

    private final RegisteredClientRepository clientRepository;
    private final OAuth2AuthorizationConsentService consentService;

    @GetMapping("/oauth2/consent")
    public String consent(
            Principal principal,
            Model model,
            @RequestParam(OAuth2ParameterNames.CLIENT_ID) String clientId,
            @RequestParam(OAuth2ParameterNames.SCOPE) String scope,
            @RequestParam(OAuth2ParameterNames.STATE) String state) {

        RegisteredClient client = clientRepository.findByClientId(clientId);
        OAuth2AuthorizationConsent currentConsent =
                consentService.findById(clientId, principal.getName());

        Set<String> approvedScopes = currentConsent != null ?
                currentConsent.getScopes() : new HashSet<>();

        Set<ScopeWithDescription> scopesToApprove = Arrays.stream(scope.split(" "))
                .filter(s -> !approvedScopes.contains(s))
                .map(s -> new ScopeWithDescription(s, getScopeDescription(s)))
                .collect(Collectors.toSet());

        model.addAttribute("clientId", clientId);
        model.addAttribute("clientName", getClientName(client));
        model.addAttribute("state", state);
        model.addAttribute("scopes", scopesToApprove);
        model.addAttribute("previouslyApproved", approvedScopes);

        return "consent";  // consent.html template
    }

    private String getScopeDescription(String scope) {
        return switch (scope) {
            case "openid" -> "Verify your identity";
            case "profile" -> "Access your profile information";
            case "email" -> "Access your email address";
            case "orders:read" -> "View your orders";
            case "orders:write" -> "Create and modify orders";
            default -> scope;
        };
    }

    private String getClientName(RegisteredClient client) {
        return Optional.ofNullable(client.getClientName()).orElse(client.getClientId());
    }

    record ScopeWithDescription(String scope, String description) {}
}
```

---

## 5. Client Credentials Flow

```java
// Used by backend services to call each other
// No user involved — the service authenticates itself

// Client service configuration
@Configuration
public class OrderServiceSecurityConfig {

    @Bean
    public OAuth2AuthorizedClientManager authorizedClientManager(
            ClientRegistrationRepository clientRegistrationRepository,
            OAuth2AuthorizedClientRepository authorizedClientRepository) {

        OAuth2AuthorizedClientProvider provider =
                OAuth2AuthorizedClientProviderBuilder.builder()
                        .clientCredentials()
                        .build();

        DefaultOAuth2AuthorizedClientManager manager = new DefaultOAuth2AuthorizedClientManager(
                clientRegistrationRepository, authorizedClientRepository);
        manager.setAuthorizedClientProvider(provider);
        return manager;
    }

    @Bean
    public WebClient inventoryServiceClient(OAuth2AuthorizedClientManager manager) {
        ServletOAuth2AuthorizedClientExchangeFilterFunction filter =
                new ServletOAuth2AuthorizedClientExchangeFilterFunction(manager);
        filter.setDefaultClientRegistrationId("inventory-service");

        return WebClient.builder()
                .baseUrl("http://inventory-service:8080")
                .apply(filter.oauth2Configuration())
                .build();
    }
}

// application.yml for order-service
/*
spring:
  security:
    oauth2:
      client:
        registration:
          inventory-service:
            client-id: order-service
            client-secret: order-service-secret
            authorization-grant-type: client_credentials
            scope: inventory:read
        provider:
          auth-server:
            issuer-uri: http://localhost:9000
        registration:
          inventory-service:
            provider: auth-server
*/

@Service
public class InventoryClientService {

    private final WebClient inventoryServiceClient;

    public int checkStock(Long productId) {
        return inventoryServiceClient
                .get()
                .uri("/inventory/{productId}/stock", productId)
                .attributes(clientRegistrationId("inventory-service"))
                .retrieve()
                .bodyToMono(StockResponse.class)
                .map(StockResponse::getAvailable)
                .block();
    }
}
```

---

## 6. Token Customization

```java
// Add custom claims to the JWT access token

@Configuration
public class TokenCustomizationConfig {

    @Bean
    public OAuth2TokenCustomizer<JwtEncodingContext> jwtCustomizer(UserRepository userRepository) {
        return context -> {
            if (OAuth2TokenType.ACCESS_TOKEN.equals(context.getTokenType())) {
                Authentication principal = context.getPrincipal();

                // Add user claims for Authorization Code grant
                if (principal instanceof UsernamePasswordAuthenticationToken ||
                        principal instanceof OAuth2AuthenticationToken) {
                    String username = principal.getName();
                    userRepository.findByEmail(username).ifPresent(user -> {
                        context.getClaims()
                                .claim("user_id", user.getId())
                                .claim("tenant_id", user.getTenantId())
                                .claim("roles", user.getRoles())
                                .claim("permissions", user.getPermissions());
                    });
                }

                // Add client-specific claims for Client Credentials grant
                RegisteredClient client = context.getRegisteredClient();
                context.getClaims()
                        .claim("client_name", client.getClientName())
                        .claim("environment", System.getenv("APP_ENV"));
            }

            // Customize ID token (OIDC)
            if (OidcParameterNames.ID_TOKEN.equals(context.getTokenType().getValue())) {
                context.getClaims()
                        .claim("acr", "urn:mace:incommon:iap:silver");  // authentication class
            }
        };
    }
}

// Access the custom claims in a resource server
@RestController
@RequestMapping("/api/me")
public class UserProfileController {

    @GetMapping
    public ResponseEntity<UserProfileResponse> getProfile(
            @AuthenticationPrincipal Jwt jwt) {

        return ResponseEntity.ok(UserProfileResponse.builder()
                .userId(jwt.getClaimAsString("user_id"))
                .email(jwt.getSubject())
                .tenantId(jwt.getClaimAsString("tenant_id"))
                .roles(jwt.getClaimAsStringList("roles"))
                .build());
    }
}
```

---

## 7. OIDC Support

```java
// Spring Authorization Server enables OIDC automatically
// Endpoints exposed:
// GET /.well-known/openid-configuration  -> discovery document
// GET /oauth2/jwks                        -> JSON Web Key Set
// GET /userinfo                           -> user info (requires openid scope)

// Custom UserInfo endpoint configuration
@Configuration
public class OidcUserInfoConfiguration {

    @Bean
    public OAuth2TokenCustomizer<JwtEncodingContext> idTokenCustomizer(UserRepository userRepository) {
        return context -> {
            if (OidcParameterNames.ID_TOKEN.equals(context.getTokenType().getValue())) {
                String username = context.getPrincipal().getName();
                userRepository.findByEmail(username).ifPresent(user -> {
                    context.getClaims()
                            .claim(StandardClaimNames.NAME, user.getFullName())
                            .claim(StandardClaimNames.GIVEN_NAME, user.getFirstName())
                            .claim(StandardClaimNames.FAMILY_NAME, user.getLastName())
                            .claim(StandardClaimNames.EMAIL, user.getEmail())
                            .claim(StandardClaimNames.EMAIL_VERIFIED, user.isEmailVerified())
                            .claim(StandardClaimNames.PICTURE, user.getAvatarUrl());
                });
            }
        };
    }

    // Custom UserInfo mapper
    @Bean
    public OidcUserInfoService oidcUserInfoService(UserRepository userRepository) {
        return (registeredClient, principal) -> {
            String username = principal.getName();
            AppUser user = userRepository.findByEmail(username).orElseThrow();

            return OidcUserInfo.builder()
                    .subject(username)
                    .name(user.getFullName())
                    .givenName(user.getFirstName())
                    .familyName(user.getLastName())
                    .email(user.getEmail())
                    .emailVerified(user.isEmailVerified())
                    .claim("tenant_id", user.getTenantId())
                    .build();
        };
    }
}
```

---

## 8. Token Introspection Endpoint

```java
// POST /oauth2/introspect
// Content-Type: application/x-www-form-urlencoded
// Authorization: Basic <base64(clientId:clientSecret)>
// Body: token=<access_token>

// Response:
// { "active": true, "sub": "user@example.com", "exp": 1234567890, ... }
// OR
// { "active": false }

// Custom introspection response
@Configuration
public class IntrospectionConfiguration {

    @Bean
    public OAuth2TokenIntrospectionAuthenticationProvider tokenIntrospectionAuthenticationProvider(
            RegisteredClientRepository clientRepository,
            OAuth2AuthorizationService authorizationService) {

        return new OAuth2TokenIntrospectionAuthenticationProvider(
                clientRepository, authorizationService);
    }
}

// Resource server using introspection (opaque tokens)
// application.yml
/*
spring:
  security:
    oauth2:
      resourceserver:
        opaque-token:
          introspection-uri: http://localhost:9000/oauth2/introspect
          client-id: resource-server
          client-secret: rs-secret
*/
```

---

## 9. Refresh Token Rotation

```java
// Configure refresh token rotation in TokenSettings
RegisteredClient client = RegisteredClient.withId(UUID.randomUUID().toString())
        .clientId("web-app")
        // ...
        .tokenSettings(TokenSettings.builder()
                .accessTokenTimeToLive(Duration.ofMinutes(15))
                .refreshTokenTimeToLive(Duration.ofDays(30))
                .reuseRefreshTokens(false)  // issue new refresh token each time
                .build())
        .build();

// With reuseRefreshTokens(false):
// - Each /oauth2/token?grant_type=refresh_token call returns a NEW refresh token
// - The old refresh token is invalidated immediately
// - Detect refresh token reuse attacks: second use of same token triggers revocation

// Custom refresh token rotation handler
@Component
public class RefreshTokenRotationHandler {

    private final OAuth2AuthorizationService authorizationService;
    private final ApplicationEventPublisher eventPublisher;

    @EventListener
    public void onRefreshTokenRevoked(OAuth2AuthorizationEvent event) {
        // Could detect reuse attempts and lock the account
        if (event instanceof OAuth2AuthorizationRevocationEvent revocationEvent) {
            log.warn("Refresh token revoked for client: {}",
                    revocationEvent.getAuthorization().getRegisteredClientId());
        }
    }
}
```

---

## 10. JWK Set Endpoint

```java
// GET /oauth2/jwks returns the public keys for JWT verification
// Resource servers use this to verify JWTs without calling the auth server

@Configuration
public class JwkSetConfiguration {

    @Value("${app.auth-server.keystore-path:}")
    private String keystorePath;

    @Bean
    public JWKSource<SecurityContext> jwkSource() {
        if (StringUtils.hasText(keystorePath)) {
            return loadFromKeystore();
        }
        // Development: generate in memory
        return generateInMemory();
    }

    private JWKSource<SecurityContext> loadFromKeystore() {
        try {
            KeyStore keyStore = KeyStore.getInstance("PKCS12");
            try (InputStream is = new FileInputStream(keystorePath)) {
                keyStore.load(is, keystorePassword.toCharArray());
            }

            RSAKey rsaKey = RSAKey.load(keyStore, "auth-server-key", keystorePassword.toCharArray());
            return new ImmutableJWKSet<>(new JWKSet(rsaKey));
        } catch (Exception e) {
            throw new IllegalStateException("Failed to load keystore", e);
        }
    }

    private JWKSource<SecurityContext> generateInMemory() {
        KeyPair keyPair = generateRsaKey();
        RSAPublicKey publicKey = (RSAPublicKey) keyPair.getPublic();
        RSAPrivateKey privateKey = (RSAPrivateKey) keyPair.getPrivate();

        RSAKey rsaKey = new RSAKey.Builder(publicKey)
                .privateKey(privateKey)
                .keyID(UUID.randomUUID().toString())
                .keyUse(KeyUse.SIGNATURE)
                .algorithm(JWSAlgorithm.RS256)
                .build();

        return new ImmutableJWKSet<>(new JWKSet(rsaKey));
    }

    private KeyPair generateRsaKey() {
        try {
            KeyPairGenerator generator = KeyPairGenerator.getInstance("RSA");
            generator.initialize(2048);
            return generator.generateKeyPair();
        } catch (NoSuchAlgorithmException e) {
            throw new IllegalStateException("Failed to generate RSA key pair", e);
        }
    }

    @Bean
    public JwtDecoder jwtDecoder(JWKSource<SecurityContext> jwkSource) {
        return OAuth2AuthorizationServerConfiguration.jwtDecoder(jwkSource);
    }

    @Bean
    public AuthorizationServerSettings authorizationServerSettings() {
        return AuthorizationServerSettings.builder()
                .issuer("http://localhost:9000")
                .build();
    }
}
```

---

## 11. Integrating with Spring Security Resource Server

```java
// Resource server that accepts tokens from our auth server

// pom.xml (resource server, e.g., order-service)
// spring-boot-starter-oauth2-resource-server

// application.yml
/*
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: http://localhost:9000  # fetches JWKS automatically
          # OR use explicit JWK set URI:
          # jwk-set-uri: http://localhost:9000/oauth2/jwks
*/

@Configuration
@EnableWebSecurity
@EnableMethodSecurity
public class ResourceServerConfiguration {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
                .authorizeHttpRequests(auth -> auth
                        .requestMatchers("/api/public/**").permitAll()
                        .requestMatchers("/api/admin/**").hasRole("ADMIN")
                        .anyRequest().authenticated())
                .oauth2ResourceServer(oauth2 -> oauth2
                        .jwt(jwt -> jwt
                                .jwtAuthenticationConverter(jwtAuthenticationConverter())));

        return http.build();
    }

    @Bean
    public JwtAuthenticationConverter jwtAuthenticationConverter() {
        JwtGrantedAuthoritiesConverter grantedAuthoritiesConverter = new JwtGrantedAuthoritiesConverter();
        grantedAuthoritiesConverter.setAuthoritiesClaimName("roles");
        grantedAuthoritiesConverter.setAuthorityPrefix("ROLE_");

        JwtAuthenticationConverter converter = new JwtAuthenticationConverter();
        converter.setJwtGrantedAuthoritiesConverter(jwt -> {
            // Combine roles and scopes into authorities
            List<GrantedAuthority> authorities = new ArrayList<>();

            // From "roles" claim
            List<String> roles = jwt.getClaimAsStringList("roles");
            if (roles != null) {
                roles.stream()
                        .map(r -> new SimpleGrantedAuthority("ROLE_" + r))
                        .forEach(authorities::add);
            }

            // From "scope" claim
            String scope = jwt.getClaimAsString("scope");
            if (scope != null) {
                Arrays.stream(scope.split(" "))
                        .map(s -> new SimpleGrantedAuthority("SCOPE_" + s))
                        .forEach(authorities::add);
            }

            return authorities;
        });

        return converter;
    }
}

// Using method security with scopes and roles
@RestController
@RequestMapping("/api/orders")
public class OrderController {

    @GetMapping
    @PreAuthorize("hasAuthority('SCOPE_orders:read')")
    public List<OrderDto> listOrders(@AuthenticationPrincipal Jwt jwt) {
        String userId = jwt.getClaimAsString("user_id");
        return orderService.findByUser(userId);
    }

    @PostMapping
    @PreAuthorize("hasAuthority('SCOPE_orders:write') and hasRole('USER')")
    public ResponseEntity<OrderDto> createOrder(
            @RequestBody OrderRequest request,
            @AuthenticationPrincipal Jwt jwt) {
        String userId = jwt.getClaimAsString("user_id");
        OrderDto order = orderService.create(userId, request);
        return ResponseEntity.created(URI.create("/api/orders/" + order.getId())).body(order);
    }

    @DeleteMapping("/{id}")
    @PreAuthorize("hasRole('ADMIN') or @orderSecurity.isOwner(#id, authentication)")
    public ResponseEntity<Void> cancelOrder(@PathVariable Long id) {
        orderService.cancel(id);
        return ResponseEntity.noContent().build();
    }
}

// Custom security expression
@Component("orderSecurity")
public class OrderSecurityExpressions {

    private final OrderRepository orderRepository;

    public boolean isOwner(Long orderId, Authentication authentication) {
        if (!(authentication.getPrincipal() instanceof Jwt jwt)) {
            return false;
        }
        String userId = jwt.getClaimAsString("user_id");
        return orderRepository.findById(orderId)
                .map(order -> order.getCustomerId().equals(userId))
                .orElse(false);
    }
}
```

---

## 12. Real Example: SSO Server for Microservices

```java
// ===== Complete production-ready authorization server =====

// Database schema (Flyway migration)
/*
-- V1__oauth2_schema.sql
-- Spring Authorization Server standard tables

CREATE TABLE oauth2_registered_client (
    id varchar(100) NOT NULL,
    client_id varchar(100) NOT NULL,
    client_id_issued_at timestamp DEFAULT CURRENT_TIMESTAMP NOT NULL,
    client_secret varchar(200) DEFAULT NULL,
    client_secret_expires_at timestamp DEFAULT NULL,
    client_name varchar(200) NOT NULL,
    client_authentication_methods varchar(1000) NOT NULL,
    authorization_grant_types varchar(1000) NOT NULL,
    redirect_uris varchar(1000) DEFAULT NULL,
    post_logout_redirect_uris varchar(1000) DEFAULT NULL,
    scopes varchar(1000) NOT NULL,
    client_settings varchar(2000) NOT NULL,
    token_settings varchar(2000) NOT NULL,
    PRIMARY KEY (id)
);

CREATE TABLE oauth2_authorization_consent (
    registered_client_id varchar(100) NOT NULL,
    principal_name varchar(200) NOT NULL,
    authorities varchar(1000) NOT NULL,
    PRIMARY KEY (registered_client_id, principal_name)
);

CREATE TABLE oauth2_authorization (
    id varchar(100) NOT NULL,
    registered_client_id varchar(100) NOT NULL,
    principal_name varchar(200) NOT NULL,
    authorization_grant_type varchar(100) NOT NULL,
    authorized_scopes varchar(1000) DEFAULT NULL,
    attributes blob DEFAULT NULL,
    state varchar(500) DEFAULT NULL,
    authorization_code_value blob DEFAULT NULL,
    authorization_code_issued_at timestamp DEFAULT NULL,
    authorization_code_expires_at timestamp DEFAULT NULL,
    authorization_code_metadata blob DEFAULT NULL,
    access_token_value blob DEFAULT NULL,
    access_token_issued_at timestamp DEFAULT NULL,
    access_token_expires_at timestamp DEFAULT NULL,
    access_token_metadata blob DEFAULT NULL,
    access_token_type varchar(100) DEFAULT NULL,
    access_token_scopes varchar(1000) DEFAULT NULL,
    oidc_id_token_value blob DEFAULT NULL,
    oidc_id_token_issued_at timestamp DEFAULT NULL,
    oidc_id_token_expires_at timestamp DEFAULT NULL,
    oidc_id_token_metadata blob DEFAULT NULL,
    refresh_token_value blob DEFAULT NULL,
    refresh_token_issued_at timestamp DEFAULT NULL,
    refresh_token_expires_at timestamp DEFAULT NULL,
    refresh_token_metadata blob DEFAULT NULL,
    user_code_value blob DEFAULT NULL,
    user_code_issued_at timestamp DEFAULT NULL,
    user_code_expires_at timestamp DEFAULT NULL,
    user_code_metadata blob DEFAULT NULL,
    device_code_value blob DEFAULT NULL,
    device_code_issued_at timestamp DEFAULT NULL,
    device_code_expires_at timestamp DEFAULT NULL,
    device_code_metadata blob DEFAULT NULL,
    PRIMARY KEY (id)
);
*/

// Main authorization server configuration
@Configuration
public class AuthorizationServerConfig {

    @Bean
    public OAuth2AuthorizationService authorizationService(JdbcTemplate jdbcTemplate,
            RegisteredClientRepository clientRepository) {
        return new JdbcOAuth2AuthorizationService(jdbcTemplate, clientRepository);
    }

    @Bean
    public OAuth2AuthorizationConsentService authorizationConsentService(JdbcTemplate jdbcTemplate,
            RegisteredClientRepository clientRepository) {
        return new JdbcOAuth2AuthorizationConsentService(jdbcTemplate, clientRepository);
    }
}

// Token revocation endpoint controller
@RestController
@RequestMapping("/api/tokens")
public class TokenManagementController {

    private final OAuth2AuthorizationService authorizationService;

    // Allow users to see and revoke their own active sessions
    @GetMapping("/sessions")
    @PreAuthorize("isAuthenticated()")
    public List<ActiveSessionDto> getActiveSessions(@AuthenticationPrincipal Jwt jwt) {
        String principalName = jwt.getSubject();

        // Custom query to find active authorizations
        return authorizationService.findAllByPrincipal(principalName).stream()
                .filter(auth -> auth.getAccessToken() != null
                        && !auth.getAccessToken().isExpired()
                        && !auth.getAccessToken().isInvalidated())
                .map(auth -> ActiveSessionDto.builder()
                        .sessionId(auth.getId())
                        .clientName(getClientName(auth.getRegisteredClientId()))
                        .issuedAt(auth.getAccessToken().getIssuedAt())
                        .expiresAt(auth.getAccessToken().getExpiresAt())
                        .build())
                .collect(Collectors.toList());
    }

    @DeleteMapping("/sessions/{sessionId}")
    @PreAuthorize("isAuthenticated()")
    public ResponseEntity<Void> revokeSession(
            @PathVariable String sessionId,
            @AuthenticationPrincipal Jwt jwt) {

        OAuth2Authorization authorization = authorizationService.findById(sessionId);

        if (authorization == null ||
                !authorization.getPrincipalName().equals(jwt.getSubject())) {
            return ResponseEntity.notFound().build();
        }

        authorizationService.remove(authorization);
        return ResponseEntity.noContent().build();
    }

    private String getClientName(String clientId) {
        // lookup from client repository
        return clientId;
    }
}
```

---

## 13. Security Hardening

```java
@Configuration
public class SecurityHardeningConfig {

    // Rate limiting for token endpoint
    @Bean
    public FilterRegistrationBean<RateLimitFilter> rateLimitFilter() {
        FilterRegistrationBean<RateLimitFilter> registration = new FilterRegistrationBean<>();
        registration.setFilter(new RateLimitFilter(
                Bucket4j.builder()
                        .addLimit(Bandwidth.simple(100, Duration.ofMinutes(1)))
                        .build()));
        registration.addUrlPatterns("/oauth2/token");
        return registration;
    }

    // PKCE enforcement - reject non-PKCE public clients
    @Bean
    public OAuth2AuthorizationCodeRequestAuthenticationValidator pkceValidator() {
        return (authenticationContext) -> {
            OAuth2AuthorizationCodeRequestAuthenticationToken authorizationCodeRequest =
                    authenticationContext.getAuthentication();

            RegisteredClient registeredClient = authenticationContext.get(RegisteredClient.class);

            if (registeredClient.getClientAuthenticationMethods()
                    .contains(ClientAuthenticationMethod.NONE)) {
                // Public client MUST use PKCE
                if (authorizationCodeRequest.getCodeChallenge() == null) {
                    throw new OAuth2AuthorizationCodeRequestAuthenticationException(
                            new OAuth2Error(OAuth2ErrorCodes.INVALID_REQUEST,
                                    "PKCE is required for public clients", null), null);
                }
            }
        };
    }
}
```

---

## Summary

| Feature | Endpoint / Config |
|---|---|
| Authorization Code + PKCE | `GET /oauth2/authorize` |
| Token exchange | `POST /oauth2/token` |
| Token revocation | `POST /oauth2/revoke` |
| Token introspection | `POST /oauth2/introspect` |
| JWK Set (public keys) | `GET /oauth2/jwks` |
| OIDC Discovery | `GET /.well-known/openid-configuration` |
| UserInfo | `GET /userinfo` |
| OIDC logout | `GET /connect/logout` |
| Client registration | `JdbcRegisteredClientRepository` |
| Token storage | `JdbcOAuth2AuthorizationService` |
| Custom claims | `OAuth2TokenCustomizer<JwtEncodingContext>` |

## Next Part Preview

**Part 086** covers Advanced Data Validation — custom constraints, cross-field validation, group-based validation, and a complete order creation validation suite.
