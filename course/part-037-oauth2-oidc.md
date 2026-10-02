# Part 037: OAuth2 and OIDC with Spring Security

## Introduction

OAuth2 is the authorization framework that powers "Login with Google", API access tokens, and service-to-service authentication. OpenID Connect (OIDC) builds on OAuth2 to add identity. Spring Security provides first-class support for both as a Resource Server (protecting APIs) and as a Client (logging in via external providers). This part covers everything from Authorization Code flow to multi-tenant Keycloak integration.

---

## 1. OAuth2 Flows

### Authorization Code Flow (with PKCE for SPAs/mobile)

```
User         Browser/App         Authorization Server        Resource Server
 |               |                       |                         |
 |──click login──►|                       |                         |
 |               |──redirect + code_challenge──►|                  |
 |               |                       |                         |
 |◄──login page──|                       |                         |
 |──credentials──►|                       |                         |
 |               |──POST /token (code + verifier)──►|              |
 |               |◄──access_token + id_token───────|               |
 |               |                       |                         |
 |               |──GET /api/data (Bearer token)───────────────────►|
 |               |◄──protected data────────────────────────────────|
```

### Client Credentials Flow (service-to-service)

```
Service A                    Authorization Server           Service B (Resource Server)
    |──POST /token (client_id + client_secret)──►|
    |◄──access_token──────────────────────────────|
    |
    |──GET /api/data (Bearer access_token)─────────────────────────►|
    |◄──protected data──────────────────────────────────────────────|
```

### PKCE (Proof Key for Code Exchange)

```java
// PKCE prevents authorization code interception attacks
// Used for public clients (SPAs, mobile apps) that can't keep client_secret safe

// 1. Generate code_verifier (random string 43-128 chars)
String codeVerifier = generateSecureRandom();

// 2. Generate code_challenge = BASE64URL(SHA256(codeVerifier))
String codeChallenge = base64UrlEncode(sha256(codeVerifier));

// 3. Send code_challenge in authorization request
// 4. Send code_verifier in token request (server verifies)
```

---

## 2. OpenID Connect (OIDC) Concepts

| OAuth2 Term | OIDC Addition |
|------------|---------------|
| Access Token (opaque or JWT) | ID Token (always JWT) |
| Scopes: `read`, `write` | Scope: `openid` triggers OIDC |
| No user info in token | ID Token contains user claims |
| `POST /token` | `GET /userinfo` endpoint |
| — | `/.well-known/openid-configuration` discovery |

### ID Token Claims

```json
{
  "iss": "https://accounts.google.com",
  "sub": "110169484474386594838",
  "aud": "my-client-id",
  "exp": 1699900800,
  "iat": 1699897200,
  "email": "user@example.com",
  "email_verified": true,
  "name": "John Doe",
  "picture": "https://...",
  "given_name": "John",
  "family_name": "Doe",
  "locale": "en"
}
```

---

## 3. Spring Security OAuth2 Resource Server

### Maven Dependencies

```xml
<!-- pom.xml -->
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-security</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-oauth2-client</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
</dependencies>
```

### Resource Server Configuration

```java
// src/main/java/com/example/oauth/config/SecurityConfig.java
package com.example.oauth.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.method.configuration.EnableMethodSecurity;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.oauth2.server.resource.authentication.JwtAuthenticationConverter;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
@EnableWebSecurity
@EnableMethodSecurity(prePostEnabled = true)
public class SecurityConfig {

    private final JwtRoleConverter jwtRoleConverter;

    public SecurityConfig(JwtRoleConverter jwtRoleConverter) {
        this.jwtRoleConverter = jwtRoleConverter;
    }

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        return http
            .csrf(csrf -> csrf.disable())  // Stateless API: CSRF not needed

            .sessionManagement(session ->
                session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))

            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/actuator/health", "/actuator/info").permitAll()
                .requestMatchers("/api/public/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .requestMatchers("/api/users/**").hasAnyRole("USER", "ADMIN")
                .anyRequest().authenticated()
            )

            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(jwt -> jwt
                    .jwtAuthenticationConverter(jwtAuthenticationConverter())
                )
            )

            .build();
    }

    @Bean
    public JwtAuthenticationConverter jwtAuthenticationConverter() {
        JwtAuthenticationConverter converter = new JwtAuthenticationConverter();
        converter.setJwtGrantedAuthoritiesConverter(jwtRoleConverter);
        return converter;
    }
}
```

### application.yaml for Resource Server

```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          # Option 1: JWKS endpoint (recommended)
          jwk-set-uri: https://accounts.google.com/.well-known/jwks.json

          # Option 2: Issuer URI (auto-discovers JWKS)
          issuer-uri: https://accounts.google.com

          # Option 3: Static public key (for Keycloak or custom AS)
          # public-key-location: classpath:public.pem
```

---

## 4. JWT Validation with JWKS

### Custom JWT Decoder with Validation

```java
// src/main/java/com/example/oauth/config/JwtConfig.java
package com.example.oauth.config;

import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.oauth2.core.DelegatingOAuth2TokenValidator;
import org.springframework.security.oauth2.core.OAuth2Error;
import org.springframework.security.oauth2.core.OAuth2TokenValidator;
import org.springframework.security.oauth2.core.OAuth2TokenValidatorResult;
import org.springframework.security.oauth2.jose.jws.MacAlgorithm;
import org.springframework.security.oauth2.jose.jws.SignatureAlgorithm;
import org.springframework.security.oauth2.jwt.*;

import java.util.Arrays;

@Configuration
public class JwtConfig {

    @Value("${spring.security.oauth2.resourceserver.jwt.issuer-uri}")
    private String issuerUri;

    @Value("${app.security.jwt.audience}")
    private String audience;

    @Bean
    public JwtDecoder jwtDecoder() {
        // Build decoder from JWKS endpoint
        NimbusJwtDecoder jwtDecoder = JwtDecoders.fromIssuerLocation(issuerUri);

        // Add custom validators
        OAuth2TokenValidator<Jwt> audienceValidator = audienceValidator();
        OAuth2TokenValidator<Jwt> issuerValidator = JwtValidators.createDefaultWithIssuer(issuerUri);
        OAuth2TokenValidator<Jwt> customValidator = new DelegatingOAuth2TokenValidator<>(
            issuerValidator,
            audienceValidator,
            new JwtNotBeforeValidator()
        );

        jwtDecoder.setJwtValidator(customValidator);
        return jwtDecoder;
    }

    private OAuth2TokenValidator<Jwt> audienceValidator() {
        return jwt -> {
            if (jwt.getAudience().contains(audience)) {
                return OAuth2TokenValidatorResult.success();
            }
            return OAuth2TokenValidatorResult.failure(
                new OAuth2Error("invalid_token",
                    "The token was not issued for audience: " + audience, null)
            );
        };
    }
}
```

```java
// src/main/java/com/example/oauth/config/JwtNotBeforeValidator.java
package com.example.oauth.config;

import org.springframework.security.oauth2.core.OAuth2Error;
import org.springframework.security.oauth2.core.OAuth2TokenValidator;
import org.springframework.security.oauth2.core.OAuth2TokenValidatorResult;
import org.springframework.security.oauth2.jwt.Jwt;

import java.time.Instant;

public class JwtNotBeforeValidator implements OAuth2TokenValidator<Jwt> {

    @Override
    public OAuth2TokenValidatorResult validate(Jwt jwt) {
        Instant notBefore = jwt.getNotBefore();
        if (notBefore == null) {
            return OAuth2TokenValidatorResult.success();
        }
        if (Instant.now().isAfter(notBefore)) {
            return OAuth2TokenValidatorResult.success();
        }
        return OAuth2TokenValidatorResult.failure(
            new OAuth2Error("invalid_token", "Token is not yet valid (nbf)", null)
        );
    }
}
```

### Role Converter from JWT Claims

```java
// src/main/java/com/example/oauth/config/JwtRoleConverter.java
package com.example.oauth.config;

import org.springframework.core.convert.converter.Converter;
import org.springframework.security.core.GrantedAuthority;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.oauth2.jwt.Jwt;
import org.springframework.stereotype.Component;

import java.util.*;
import java.util.stream.Collectors;
import java.util.stream.Stream;

@Component
public class JwtRoleConverter implements Converter<Jwt, Collection<GrantedAuthority>> {

    @Override
    public Collection<GrantedAuthority> convert(Jwt jwt) {
        return Stream.concat(
            extractRealmRoles(jwt).stream(),
            extractResourceRoles(jwt).stream()
        ).collect(Collectors.toSet());
    }

    // Keycloak realm roles: realm_access.roles
    @SuppressWarnings("unchecked")
    private Collection<GrantedAuthority> extractRealmRoles(Jwt jwt) {
        Map<String, Object> realmAccess = jwt.getClaimAsMap("realm_access");
        if (realmAccess == null) return Collections.emptyList();

        List<String> roles = (List<String>) realmAccess.get("roles");
        if (roles == null) return Collections.emptyList();

        return roles.stream()
            .map(role -> new SimpleGrantedAuthority("ROLE_" + role.toUpperCase()))
            .collect(Collectors.toList());
    }

    // Keycloak client roles: resource_access.<clientId>.roles
    @SuppressWarnings("unchecked")
    private Collection<GrantedAuthority> extractResourceRoles(Jwt jwt) {
        Map<String, Object> resourceAccess = jwt.getClaimAsMap("resource_access");
        if (resourceAccess == null) return Collections.emptyList();

        return resourceAccess.values().stream()
            .filter(v -> v instanceof Map)
            .flatMap(v -> {
                List<String> roles = (List<String>) ((Map<?, ?>) v).get("roles");
                return roles == null ? Stream.empty() : roles.stream();
            })
            .map(role -> new SimpleGrantedAuthority("ROLE_" + role.toUpperCase()))
            .collect(Collectors.toList());
    }
}
```

---

## 5. Spring Security OAuth2 Client

### application.yaml for OAuth2 Client

```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          google:
            client-id: ${GOOGLE_CLIENT_ID}
            client-secret: ${GOOGLE_CLIENT_SECRET}
            scope:
              - openid
              - profile
              - email
            redirect-uri: "{baseUrl}/login/oauth2/code/{registrationId}"

          github:
            client-id: ${GITHUB_CLIENT_ID}
            client-secret: ${GITHUB_CLIENT_SECRET}
            scope:
              - user:email
              - read:user

          keycloak:
            client-id: my-spring-app
            client-secret: ${KEYCLOAK_CLIENT_SECRET}
            authorization-grant-type: authorization_code
            scope:
              - openid
              - profile
              - email
              - roles

        provider:
          keycloak:
            issuer-uri: http://localhost:8180/realms/myrealm
            user-name-attribute: preferred_username
```

### OAuth2 Client Security Configuration

```java
// src/main/java/com/example/oauth/config/OAuth2ClientConfig.java
package com.example.oauth.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.web.SecurityFilterChain;
import org.springframework.security.web.authentication.AuthenticationSuccessHandler;

@Configuration
public class OAuth2ClientConfig {

    private final CustomOAuth2UserService customOAuth2UserService;
    private final CustomOidcUserService customOidcUserService;

    public OAuth2ClientConfig(CustomOAuth2UserService customOAuth2UserService,
                              CustomOidcUserService customOidcUserService) {
        this.customOAuth2UserService = customOAuth2UserService;
        this.customOidcUserService = customOidcUserService;
    }

    @Bean
    public SecurityFilterChain clientFilterChain(HttpSecurity http) throws Exception {
        return http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/", "/login", "/error").permitAll()
                .anyRequest().authenticated()
            )

            .oauth2Login(oauth2 -> oauth2
                .loginPage("/login")                          // Custom login page
                .userInfoEndpoint(userInfo -> userInfo
                    .userService(customOAuth2UserService)     // For GitHub (OAuth2)
                    .oidcUserService(customOidcUserService)   // For Google, Keycloak (OIDC)
                )
                .successHandler(authSuccessHandler())
                .failureUrl("/login?error=true")
            )

            .logout(logout -> logout
                .logoutSuccessUrl("/")
                .clearAuthentication(true)
                .invalidateHttpSession(true)
                .deleteCookies("JSESSIONID")
            )

            .build();
    }

    @Bean
    public AuthenticationSuccessHandler authSuccessHandler() {
        return (request, response, authentication) -> {
            // Store user info in session, redirect to dashboard
            response.sendRedirect("/dashboard");
        };
    }
}
```

### Custom OAuth2 User Service

```java
// src/main/java/com/example/oauth/service/CustomOAuth2UserService.java
package com.example.oauth.service;

import com.example.oauth.model.AppUser;
import com.example.oauth.repository.UserRepository;
import org.springframework.security.oauth2.client.userinfo.DefaultOAuth2UserService;
import org.springframework.security.oauth2.client.userinfo.OAuth2UserRequest;
import org.springframework.security.oauth2.core.OAuth2AuthenticationException;
import org.springframework.security.oauth2.core.user.OAuth2User;
import org.springframework.stereotype.Service;

@Service
public class CustomOAuth2UserService extends DefaultOAuth2UserService {

    private final UserRepository userRepository;

    public CustomOAuth2UserService(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    @Override
    public OAuth2User loadUser(OAuth2UserRequest userRequest) throws OAuth2AuthenticationException {
        OAuth2User oAuth2User = super.loadUser(userRequest);

        String provider = userRequest.getClientRegistration().getRegistrationId();
        String providerId = oAuth2User.getName();

        // Upsert user in database
        AppUser appUser = userRepository.findByProviderAndProviderId(provider, providerId)
            .orElse(new AppUser());

        appUser.setProvider(provider);
        appUser.setProviderId(providerId);
        appUser.setEmail(oAuth2User.getAttribute("email"));
        appUser.setName(oAuth2User.getAttribute("name"));

        if (provider.equals("github")) {
            appUser.setAvatarUrl(oAuth2User.getAttribute("avatar_url"));
        }

        userRepository.save(appUser);

        return oAuth2User;
    }
}
```

---

## 6. Google/GitHub Social Login

### Login Page

```html
<!-- src/main/resources/templates/login.html (Thymeleaf) -->
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <title>Login</title>
    <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/css/bootstrap.min.css">
</head>
<body>
<div class="container mt-5">
    <div class="row justify-content-center">
        <div class="col-md-4">
            <h2 class="text-center mb-4">Sign In</h2>

            <div th:if="${param.error}" class="alert alert-danger">
                Login failed. Please try again.
            </div>

            <div class="d-grid gap-2">
                <a href="/oauth2/authorization/google"
                   class="btn btn-outline-danger btn-lg">
                    <i class="bi bi-google"></i> Continue with Google
                </a>

                <a href="/oauth2/authorization/github"
                   class="btn btn-outline-dark btn-lg">
                    <i class="bi bi-github"></i> Continue with GitHub
                </a>

                <a href="/oauth2/authorization/keycloak"
                   class="btn btn-outline-primary btn-lg">
                    Continue with Keycloak
                </a>
            </div>
        </div>
    </div>
</div>
</body>
</html>
```

### User Info Controller

```java
// src/main/java/com/example/oauth/controller/UserController.java
package com.example.oauth.controller;

import org.springframework.security.core.annotation.AuthenticationPrincipal;
import org.springframework.security.oauth2.core.oidc.user.OidcUser;
import org.springframework.security.oauth2.core.user.OAuth2User;
import org.springframework.stereotype.Controller;
import org.springframework.ui.Model;
import org.springframework.web.bind.annotation.GetMapping;

@Controller
public class UserController {

    @GetMapping("/dashboard")
    public String dashboard(@AuthenticationPrincipal Object principal, Model model) {
        if (principal instanceof OidcUser oidcUser) {
            model.addAttribute("name", oidcUser.getFullName());
            model.addAttribute("email", oidcUser.getEmail());
            model.addAttribute("picture", oidcUser.getPicture());
            model.addAttribute("provider", "oidc");
        } else if (principal instanceof OAuth2User oauth2User) {
            model.addAttribute("name", oauth2User.getAttribute("name"));
            model.addAttribute("email", oauth2User.getAttribute("email"));
            model.addAttribute("provider", "oauth2");
        }
        return "dashboard";
    }

    @GetMapping("/api/me")
    public Object getCurrentUser(@AuthenticationPrincipal OidcUser user) {
        return Map.of(
            "sub", user.getSubject(),
            "email", user.getEmail(),
            "name", user.getFullName(),
            "roles", user.getAuthorities()
        );
    }
}
```

---

## 7. Keycloak as Authorization Server

### Keycloak Setup with Docker

```yaml
# docker-compose.yaml
version: '3.8'
services:
  keycloak:
    image: quay.io/keycloak/keycloak:23.0
    command: start-dev
    environment:
      KEYCLOAK_ADMIN: admin
      KEYCLOAK_ADMIN_PASSWORD: admin
      KC_HTTP_PORT: 8180
    ports:
      - "8180:8180"
    volumes:
      - keycloak-data:/opt/keycloak/data

volumes:
  keycloak-data:
```

### Keycloak Realm Configuration

```bash
# Create realm via CLI
/opt/keycloak/bin/kcadm.sh config credentials \
  --server http://localhost:8180 \
  --realm master \
  --user admin \
  --password admin

# Create realm
/opt/keycloak/bin/kcadm.sh create realms \
  -s realm=myrealm \
  -s enabled=true

# Create client
/opt/keycloak/bin/kcadm.sh create clients \
  -r myrealm \
  -s clientId=my-spring-app \
  -s publicClient=false \
  -s secret=my-client-secret \
  -s 'redirectUris=["http://localhost:8080/*"]' \
  -s enabled=true

# Create roles
/opt/keycloak/bin/kcadm.sh create roles -r myrealm -s name=USER
/opt/keycloak/bin/kcadm.sh create roles -r myrealm -s name=ADMIN

# Create test user
/opt/keycloak/bin/kcadm.sh create users \
  -r myrealm \
  -s username=testuser \
  -s email=test@example.com \
  -s enabled=true

/opt/keycloak/bin/kcadm.sh set-password \
  -r myrealm \
  --username testuser \
  --new-password password123
```

### Spring Boot + Keycloak Resource Server

```yaml
# application.yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: http://localhost:8180/realms/myrealm
          jwk-set-uri: http://localhost:8180/realms/myrealm/protocol/openid-connect/certs
```

---

## 8. Role Mapping from JWT Claims

```java
// src/main/java/com/example/oauth/config/KeycloakRoleConverter.java
package com.example.oauth.config;

import org.springframework.core.convert.converter.Converter;
import org.springframework.security.core.GrantedAuthority;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.oauth2.jwt.Jwt;
import org.springframework.stereotype.Component;

import java.util.*;
import java.util.stream.Collectors;
import java.util.stream.Stream;

@Component
public class KeycloakRoleConverter implements Converter<Jwt, Collection<GrantedAuthority>> {

    private static final String REALM_ACCESS_CLAIM = "realm_access";
    private static final String RESOURCE_ACCESS_CLAIM = "resource_access";
    private static final String CLIENT_ID = "my-spring-app";

    @Override
    @SuppressWarnings("unchecked")
    public Collection<GrantedAuthority> convert(Jwt jwt) {
        Set<GrantedAuthority> authorities = new HashSet<>();

        // Extract realm-level roles
        Map<String, Object> realmAccess = jwt.getClaimAsMap(REALM_ACCESS_CLAIM);
        if (realmAccess != null) {
            List<String> realmRoles = (List<String>) realmAccess.get("roles");
            if (realmRoles != null) {
                realmRoles.stream()
                    .map(role -> new SimpleGrantedAuthority("ROLE_" + role.toUpperCase()))
                    .forEach(authorities::add);
            }
        }

        // Extract client-level roles for our specific client
        Map<String, Object> resourceAccess = jwt.getClaimAsMap(RESOURCE_ACCESS_CLAIM);
        if (resourceAccess != null && resourceAccess.containsKey(CLIENT_ID)) {
            Map<String, Object> clientAccess = (Map<String, Object>) resourceAccess.get(CLIENT_ID);
            List<String> clientRoles = (List<String>) clientAccess.get("roles");
            if (clientRoles != null) {
                clientRoles.stream()
                    .map(role -> new SimpleGrantedAuthority("ROLE_" + role.toUpperCase()))
                    .forEach(authorities::add);
            }
        }

        // Extract scopes as authorities (SCOPE_xxx)
        String scope = jwt.getClaimAsString("scope");
        if (scope != null) {
            Arrays.stream(scope.split(" "))
                .map(s -> new SimpleGrantedAuthority("SCOPE_" + s))
                .forEach(authorities::add);
        }

        return authorities;
    }
}
```

---

## 9. Method Security with @PreAuthorize

### Securing Service Methods

```java
// src/main/java/com/example/oauth/service/ProductService.java
package com.example.oauth.service;

import com.example.oauth.model.Product;
import org.springframework.security.access.prepost.PostFilter;
import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.security.core.annotation.AuthenticationPrincipal;
import org.springframework.security.oauth2.jwt.Jwt;
import org.springframework.stereotype.Service;

import java.util.List;

@Service
public class ProductService {

    // Only ADMIN can create products
    @PreAuthorize("hasRole('ADMIN')")
    public Product createProduct(Product product) {
        return productRepository.save(product);
    }

    // Any authenticated user can read
    @PreAuthorize("isAuthenticated()")
    public Product getProduct(Long id) {
        return productRepository.findById(id).orElseThrow();
    }

    // Filter list: return only user's own items OR admin sees all
    @PreAuthorize("isAuthenticated()")
    @PostFilter("filterObject.ownerId == authentication.name or hasRole('ADMIN')")
    public List<Product> getAllProducts() {
        return productRepository.findAll();
    }

    // Admin or the owner of the product
    @PreAuthorize("hasRole('ADMIN') or @productSecurity.isOwner(#id, authentication)")
    public void deleteProduct(Long id) {
        productRepository.deleteById(id);
    }

    // Check custom JWT claim
    @PreAuthorize("hasAuthority('SCOPE_products:write')")
    public Product updateProduct(Long id, Product product) {
        return productRepository.save(product);
    }

    // SpEL with JWT claims
    @PreAuthorize("#jwt.subject == #userId or hasRole('ADMIN')")
    public Object getUserData(String userId, @AuthenticationPrincipal Jwt jwt) {
        return userService.findById(userId);
    }
}
```

```java
// src/main/java/com/example/oauth/security/ProductSecurity.java
package com.example.oauth.security;

import com.example.oauth.repository.ProductRepository;
import org.springframework.security.core.Authentication;
import org.springframework.stereotype.Component;

@Component("productSecurity")
public class ProductSecurity {

    private final ProductRepository productRepository;

    public ProductSecurity(ProductRepository productRepository) {
        this.productRepository = productRepository;
    }

    public boolean isOwner(Long productId, Authentication authentication) {
        return productRepository.findById(productId)
            .map(p -> p.getOwnerId().equals(authentication.getName()))
            .orElse(false);
    }
}
```

---

## 10. Token Introspection

### Opaque Token Support (when JWT is not used)

```yaml
# application.yaml - for opaque tokens
spring:
  security:
    oauth2:
      resourceserver:
        opaquetoken:
          introspection-uri: http://localhost:8180/realms/myrealm/protocol/openid-connect/token/introspect
          client-id: my-spring-app
          client-secret: my-client-secret
```

```java
// Custom introspector to add extra claims
// src/main/java/com/example/oauth/config/CustomIntrospector.java
package com.example.oauth.config;

import org.springframework.security.oauth2.core.OAuth2AuthenticatedPrincipal;
import org.springframework.security.oauth2.server.resource.introspection.OpaqueTokenIntrospector;
import org.springframework.security.oauth2.server.resource.introspection.SpringOpaqueTokenIntrospector;

public class CustomIntrospector implements OpaqueTokenIntrospector {

    private final OpaqueTokenIntrospector delegate;

    public CustomIntrospector(String introspectionUri, String clientId, String clientSecret) {
        this.delegate = new SpringOpaqueTokenIntrospector(introspectionUri, clientId, clientSecret);
    }

    @Override
    public OAuth2AuthenticatedPrincipal introspect(String token) {
        OAuth2AuthenticatedPrincipal principal = delegate.introspect(token);
        // Augment with database roles, etc.
        return principal;
    }
}
```

---

## 11. Real Example: Multi-tenant App with Keycloak

### Multi-Tenant JWT Decoder

```java
// src/main/java/com/example/oauth/multitenant/MultiTenantJwtDecoder.java
package com.example.oauth.multitenant;

import org.springframework.security.oauth2.jwt.Jwt;
import org.springframework.security.oauth2.jwt.JwtDecoder;
import org.springframework.security.oauth2.jwt.JwtDecoders;
import org.springframework.security.oauth2.jwt.JwtException;
import org.springframework.stereotype.Component;

import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

/**
 * Resolves a JwtDecoder based on the tenant in the JWT issuer claim.
 * Each tenant has its own Keycloak realm.
 */
@Component
public class MultiTenantJwtDecoder implements JwtDecoder {

    private static final String KEYCLOAK_BASE = "http://localhost:8180/realms/";

    // Allowed tenant realms
    private static final Map<String, String> TENANT_REALMS = Map.of(
        "tenant-a", "tenant-a-realm",
        "tenant-b", "tenant-b-realm",
        "tenant-c", "tenant-c-realm"
    );

    // Cache decoders per tenant (lazy init)
    private final Map<String, JwtDecoder> decoderCache = new ConcurrentHashMap<>();

    @Override
    public Jwt decode(String token) throws JwtException {
        String tenantId = extractTenantFromToken(token);

        JwtDecoder decoder = decoderCache.computeIfAbsent(tenantId, this::buildDecoder);
        return decoder.decode(token);
    }

    private String extractTenantFromToken(String token) {
        // Decode JWT header/payload without validation to get issuer
        try {
            String[] parts = token.split("\\.");
            String payload = new String(java.util.Base64.getUrlDecoder().decode(parts[1]));
            com.fasterxml.jackson.databind.ObjectMapper mapper = new com.fasterxml.jackson.databind.ObjectMapper();
            Map<?, ?> claims = mapper.readValue(payload, Map.class);
            String issuer = (String) claims.get("iss");

            // Extract tenant from issuer: http://keycloak/realms/tenant-a-realm
            for (Map.Entry<String, String> entry : TENANT_REALMS.entrySet()) {
                if (issuer.contains(entry.getValue())) {
                    return entry.getKey();
                }
            }
            throw new JwtException("Unknown tenant in issuer: " + issuer);
        } catch (Exception e) {
            throw new JwtException("Cannot extract tenant from token", e);
        }
    }

    private JwtDecoder buildDecoder(String tenantId) {
        String realm = TENANT_REALMS.get(tenantId);
        if (realm == null) {
            throw new JwtException("Unknown tenant: " + tenantId);
        }
        String issuerUri = KEYCLOAK_BASE + realm;
        return JwtDecoders.fromIssuerLocation(issuerUri);
    }
}
```

### Tenant Context

```java
// src/main/java/com/example/oauth/multitenant/TenantContext.java
package com.example.oauth.multitenant;

public class TenantContext {
    private static final ThreadLocal<String> CURRENT_TENANT = new ThreadLocal<>();

    public static void setTenant(String tenantId) {
        CURRENT_TENANT.set(tenantId);
    }

    public static String getTenant() {
        return CURRENT_TENANT.get();
    }

    public static void clear() {
        CURRENT_TENANT.remove();
    }
}
```

```java
// src/main/java/com/example/oauth/multitenant/TenantFilter.java
package com.example.oauth.multitenant;

import jakarta.servlet.*;
import jakarta.servlet.http.HttpServletRequest;
import org.springframework.core.annotation.Order;
import org.springframework.stereotype.Component;

import java.io.IOException;

@Component
@Order(1)
public class TenantFilter implements Filter {

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
            throws IOException, ServletException {
        try {
            HttpServletRequest httpRequest = (HttpServletRequest) request;
            // Extract tenant from subdomain: tenant-a.api.myapp.com
            String host = httpRequest.getServerName();
            String tenantId = host.split("\\.")[0];
            TenantContext.setTenant(tenantId);
            chain.doFilter(request, response);
        } finally {
            TenantContext.clear();
        }
    }
}
```

### Multi-Tenant Security Config

```java
// src/main/java/com/example/oauth/multitenant/MultiTenantSecurityConfig.java
package com.example.oauth.multitenant;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
public class MultiTenantSecurityConfig {

    private final MultiTenantJwtDecoder multiTenantJwtDecoder;

    public MultiTenantSecurityConfig(MultiTenantJwtDecoder multiTenantJwtDecoder) {
        this.multiTenantJwtDecoder = multiTenantJwtDecoder;
    }

    @Bean
    public SecurityFilterChain multiTenantFilterChain(HttpSecurity http) throws Exception {
        return http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/public/**").permitAll()
                .anyRequest().authenticated()
            )
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(jwt -> jwt.decoder(multiTenantJwtDecoder))
            )
            .build();
    }
}
```

### Client Credentials Flow (Service-to-Service)

```java
// src/main/java/com/example/oauth/service/ExternalApiClient.java
package com.example.oauth.service;

import org.springframework.security.oauth2.client.OAuth2AuthorizeRequest;
import org.springframework.security.oauth2.client.OAuth2AuthorizedClientManager;
import org.springframework.security.oauth2.client.OAuth2AuthorizedClient;
import org.springframework.stereotype.Service;
import org.springframework.web.reactive.function.client.WebClient;

@Service
public class ExternalApiClient {

    private final WebClient webClient;
    private final OAuth2AuthorizedClientManager authorizedClientManager;

    public ExternalApiClient(WebClient.Builder webClientBuilder,
                              OAuth2AuthorizedClientManager authorizedClientManager) {
        this.webClient = webClientBuilder
            .baseUrl("https://external-service.com")
            .build();
        this.authorizedClientManager = authorizedClientManager;
    }

    public String callExternalApi() {
        OAuth2AuthorizeRequest request = OAuth2AuthorizeRequest
            .withClientRegistrationId("external-service")
            .principal("service-account")
            .build();

        OAuth2AuthorizedClient client = authorizedClientManager.authorize(request);
        String token = client.getAccessToken().getTokenValue();

        return webClient.get()
            .uri("/api/data")
            .headers(h -> h.setBearerAuth(token))
            .retrieve()
            .bodyToMono(String.class)
            .block();
    }
}
```

```yaml
# application.yaml - client credentials registration
spring:
  security:
    oauth2:
      client:
        registration:
          external-service:
            client-id: my-service
            client-secret: ${SERVICE_CLIENT_SECRET}
            authorization-grant-type: client_credentials
            scope: external-api.read
        provider:
          external-service:
            token-uri: https://auth.external-service.com/oauth/token
```

---

## Summary Table

| Topic | Class/Annotation | Notes |
|-------|-----------------|-------|
| Resource Server | `@EnableWebSecurity` + `oauth2ResourceServer` | JWT or opaque token |
| JWT validation | `JwtDecoder`, `JwtValidators` | Auto-configures from `issuer-uri` |
| JWKS | `jwk-set-uri` | Auto-rotates keys |
| Role mapping | `JwtAuthenticationConverter` | Convert JWT claims to GrantedAuthority |
| Method security | `@PreAuthorize`, `@PostFilter` | Requires `@EnableMethodSecurity` |
| Social login | `oauth2Login()` | Google, GitHub, Keycloak |
| Custom user service | `DefaultOAuth2UserService` | Upsert user on login |
| Client credentials | `OAuth2AuthorizedClientManager` | Service-to-service auth |
| Multi-tenant | Custom `JwtDecoder` | Route by issuer claim |
| PKCE | Automatic for SPAs | Set `requireProofKey=true` in Keycloak |

---

## What's Next

**Part 038: GraphQL with Spring Boot** — Build flexible APIs with GraphQL. We'll cover schema definition, Query/Mutation/Subscription, the N+1 problem with DataLoader, pagination with connections, and security considerations.

---

*End of Part 037: OAuth2 and OIDC with Spring Security*
