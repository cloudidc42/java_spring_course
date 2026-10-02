# Part 098: Microservices Security Patterns

## Introduction

Securing a distributed system is fundamentally different from securing a monolith. In microservices, every service-to-service call is a potential attack surface. This part covers the security patterns that protect production microservices architectures.

---

## Threat Model in Microservices

```
External Client → API Gateway → Service A → Service B → Database
                                     ↓
                               Service C → External API

Attack vectors:
- Compromised service making unauthorized calls (lateral movement)
- Token theft (JWT replay attacks)
- Man-in-the-middle between services
- Secret exposure (database passwords, API keys)
- Privilege escalation via misconfigured authorization
```

---

## mTLS: Mutual TLS for Service-to-Service Auth

In standard TLS, only the server presents a certificate. In mTLS, both parties present certificates — the server authenticates itself to the client AND the client authenticates itself to the server.

```yaml
# application.yml - Spring Boot mTLS configuration
server:
  port: 8443
  ssl:
    key-store: classpath:keystore/service-a.p12
    key-store-password: ${KEYSTORE_PASSWORD}
    key-store-type: PKCS12
    key-alias: service-a
    # Enable client certificate requirement
    client-auth: need
    trust-store: classpath:keystore/truststore.p12
    trust-store-password: ${TRUSTSTORE_PASSWORD}
    trust-store-type: PKCS12
    protocol: TLS
    enabled-protocols: TLSv1.3,TLSv1.2
    ciphers:
      - TLS_AES_256_GCM_SHA384
      - TLS_CHACHA20_POLY1305_SHA256
      - TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384

# RestTemplate/WebClient configured for mTLS
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://auth.example.com
```

```java
// src/main/java/com/example/security/MtlsConfig.java
package com.example.security;

import lombok.extern.slf4j.Slf4j;
import org.apache.hc.client5.http.impl.classic.HttpClients;
import org.apache.hc.client5.http.impl.io.PoolingHttpClientConnectionManagerBuilder;
import org.apache.hc.client5.http.ssl.SSLConnectionSocketFactoryBuilder;
import org.apache.hc.core5.ssl.SSLContextBuilder;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.io.Resource;
import org.springframework.http.client.HttpComponentsClientHttpRequestFactory;
import org.springframework.web.client.RestClient;

import javax.net.ssl.SSLContext;
import java.security.KeyStore;

@Slf4j
@Configuration
public class MtlsConfig {

    @Value("${client.ssl.key-store}")
    private Resource keyStoreResource;

    @Value("${client.ssl.key-store-password}")
    private String keyStorePassword;

    @Value("${client.ssl.trust-store}")
    private Resource trustStoreResource;

    @Value("${client.ssl.trust-store-password}")
    private String trustStorePassword;

    @Bean
    public RestClient mtlsRestClient() throws Exception {
        KeyStore keyStore = KeyStore.getInstance("PKCS12");
        keyStore.load(keyStoreResource.getInputStream(), keyStorePassword.toCharArray());

        KeyStore trustStore = KeyStore.getInstance("PKCS12");
        trustStore.load(trustStoreResource.getInputStream(), trustStorePassword.toCharArray());

        SSLContext sslContext = SSLContextBuilder.create()
            .loadKeyMaterial(keyStore, keyStorePassword.toCharArray())
            .loadTrustMaterial(trustStore, null)
            .build();

        var sslSocketFactory = SSLConnectionSocketFactoryBuilder.create()
            .setSslContext(sslContext)
            .build();

        var connectionManager = PoolingHttpClientConnectionManagerBuilder.create()
            .setSSLSocketFactory(sslSocketFactory)
            .setMaxConnTotal(100)
            .setMaxConnPerRoute(20)
            .build();

        var httpClient = HttpClients.custom()
            .setConnectionManager(connectionManager)
            .build();

        return RestClient.builder()
            .requestFactory(new HttpComponentsClientHttpRequestFactory(httpClient))
            .build();
    }
}
```

```java
// src/main/java/com/example/security/MtlsAuthenticationFilter.java
package com.example.security;

import jakarta.servlet.FilterChain;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.authority.SimpleGrantedAuthority;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.stereotype.Component;
import org.springframework.web.filter.OncePerRequestFilter;

import java.io.IOException;
import java.security.cert.X509Certificate;
import java.util.List;

@Slf4j
@Component
@RequiredArgsConstructor
public class MtlsAuthenticationFilter extends OncePerRequestFilter {

    private final ServiceRegistryService serviceRegistry;

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                     HttpServletResponse response,
                                     FilterChain filterChain) throws ServletException, IOException {
        X509Certificate[] certs = (X509Certificate[]) request
            .getAttribute("jakarta.servlet.request.X509Certificate");

        if (certs != null && certs.length > 0) {
            X509Certificate clientCert = certs[0];
            String commonName = extractCommonName(clientCert.getSubjectX500Principal().getName());

            // Validate that this service is registered and trusted
            if (serviceRegistry.isTrustedService(commonName)) {
                List<String> roles = serviceRegistry.getRolesForService(commonName);
                List<SimpleGrantedAuthority> authorities = roles.stream()
                    .map(r -> new SimpleGrantedAuthority("ROLE_" + r))
                    .toList();

                UsernamePasswordAuthenticationToken auth =
                    new UsernamePasswordAuthenticationToken(commonName, null, authorities);

                SecurityContextHolder.getContext().setAuthentication(auth);
                log.debug("mTLS authenticated service: {}", commonName);
            } else {
                log.warn("Untrusted service certificate: {}", commonName);
            }
        }

        filterChain.doFilter(request, response);
    }

    private String extractCommonName(String dn) {
        for (String part : dn.split(",")) {
            String trimmed = part.trim();
            if (trimmed.startsWith("CN=")) {
                return trimmed.substring(3);
            }
        }
        return dn;
    }
}
```

---

## JWT Propagation Across Services

```java
// src/main/java/com/example/security/JwtPropagationInterceptor.java
package com.example.security;

import org.springframework.http.HttpRequest;
import org.springframework.http.client.ClientHttpRequestExecution;
import org.springframework.http.client.ClientHttpRequestInterceptor;
import org.springframework.http.client.ClientHttpResponse;
import org.springframework.security.core.Authentication;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.security.oauth2.server.resource.authentication.JwtAuthenticationToken;
import org.springframework.stereotype.Component;

import java.io.IOException;

/**
 * Automatically forwards the incoming JWT to downstream services.
 * Register as interceptor on your RestClient/RestTemplate.
 */
@Component
public class JwtPropagationInterceptor implements ClientHttpRequestInterceptor {

    @Override
    public ClientHttpResponse intercept(HttpRequest request, byte[] body,
                                         ClientHttpRequestExecution execution)
            throws IOException {
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();

        if (auth instanceof JwtAuthenticationToken jwtAuth) {
            String tokenValue = jwtAuth.getToken().getTokenValue();
            request.getHeaders().setBearerAuth(tokenValue);
        }

        return execution.execute(request, body);
    }
}
```

```java
// src/main/java/com/example/security/ServiceTokenProvider.java
package com.example.security;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.cache.annotation.Cacheable;
import org.springframework.http.MediaType;
import org.springframework.stereotype.Component;
import org.springframework.util.LinkedMultiValueMap;
import org.springframework.util.MultiValueMap;
import org.springframework.web.client.RestClient;

import java.time.Instant;
import java.util.Map;

/**
 * Fetches service-to-service tokens using OAuth2 Client Credentials flow.
 * The service authenticates WITH ITS OWN credentials (not the user's JWT).
 */
@Slf4j
@Component
@RequiredArgsConstructor
public class ServiceTokenProvider {

    private final RestClient restClient;

    @Value("${service.oauth2.client-id}")
    private String clientId;

    @Value("${service.oauth2.client-secret}")
    private String clientSecret;

    @Value("${service.oauth2.token-uri}")
    private String tokenUri;

    @Value("${service.oauth2.scope}")
    private String scope;

    private volatile TokenCache cachedToken = null;

    /**
     * Get a valid service token, refreshing if expired.
     */
    public String getServiceToken() {
        if (cachedToken == null || cachedToken.isExpired()) {
            synchronized (this) {
                if (cachedToken == null || cachedToken.isExpired()) {
                    cachedToken = fetchNewToken();
                }
            }
        }
        return cachedToken.accessToken();
    }

    private TokenCache fetchNewToken() {
        MultiValueMap<String, String> form = new LinkedMultiValueMap<>();
        form.add("grant_type", "client_credentials");
        form.add("client_id", clientId);
        form.add("client_secret", clientSecret);
        form.add("scope", scope);

        Map response = restClient.post()
            .uri(tokenUri)
            .contentType(MediaType.APPLICATION_FORM_URLENCODED)
            .body(form)
            .retrieve()
            .body(Map.class);

        String token = (String) response.get("access_token");
        int expiresIn = (int) response.get("expires_in");

        Instant expiry = Instant.now().plusSeconds(expiresIn - 30); // 30s buffer
        log.debug("Fetched new service token, expires at {}", expiry);
        return new TokenCache(token, expiry);
    }

    record TokenCache(String accessToken, Instant expiry) {
        boolean isExpired() {
            return Instant.now().isAfter(expiry);
        }
    }
}
```

---

## Secret Rotation Strategies

```java
// src/main/java/com/example/security/SecretRotationService.java
package com.example.security;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Service;
import software.amazon.awssdk.services.secretsmanager.SecretsManagerClient;
import software.amazon.awssdk.services.secretsmanager.model.GetSecretValueRequest;
import software.amazon.awssdk.services.secretsmanager.model.GetSecretValueResponse;

import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

/**
 * Demonstrates dynamic secret rotation using AWS Secrets Manager.
 * In production, use Spring Cloud AWS or HashiCorp Vault.
 */
@Slf4j
@Service
@RequiredArgsConstructor
public class SecretRotationService {

    private final SecretsManagerClient secretsManager;
    private final Map<String, CachedSecret> secretCache = new ConcurrentHashMap<>();

    // Refresh secrets every 5 minutes
    @Scheduled(fixedDelay = 300_000)
    public void refreshSecrets() {
        secretCache.forEach((secretName, cached) -> {
            try {
                String newValue = fetchFromSecretsManager(secretName);
                secretCache.put(secretName, new CachedSecret(newValue, System.currentTimeMillis()));
                log.debug("Refreshed secret: {}", secretName);
            } catch (Exception e) {
                log.error("Failed to refresh secret {}: {}", secretName, e.getMessage());
                // Keep using old value - don't fail
            }
        });
    }

    public String getSecret(String secretName) {
        CachedSecret cached = secretCache.get(secretName);

        if (cached == null || cached.isStale()) {
            String value = fetchFromSecretsManager(secretName);
            cached = new CachedSecret(value, System.currentTimeMillis());
            secretCache.put(secretName, cached);
        }

        return cached.value();
    }

    private String fetchFromSecretsManager(String secretName) {
        GetSecretValueRequest request = GetSecretValueRequest.builder()
            .secretId(secretName)
            .build();

        GetSecretValueResponse response = secretsManager.getSecretValue(request);
        return response.secretString();
    }

    record CachedSecret(String value, long fetchedAt) {
        static final long STALE_AFTER_MS = 5 * 60 * 1000; // 5 minutes

        boolean isStale() {
            return System.currentTimeMillis() - fetchedAt > STALE_AFTER_MS;
        }
    }
}
```

---

## Open Policy Agent (OPA) Integration

OPA provides a policy engine for fine-grained authorization decisions.

```java
// src/main/java/com/example/security/opa/OpaAuthorizationManager.java
package com.example.security.opa;

import com.fasterxml.jackson.databind.ObjectMapper;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.security.authorization.AuthorizationDecision;
import org.springframework.security.authorization.AuthorizationManager;
import org.springframework.security.core.Authentication;
import org.springframework.security.oauth2.server.resource.authentication.JwtAuthenticationToken;
import org.springframework.security.web.access.intercept.RequestAuthorizationContext;
import org.springframework.stereotype.Component;
import org.springframework.web.client.RestClient;

import java.util.Map;
import java.util.function.Supplier;

/**
 * Delegates authorization decisions to Open Policy Agent.
 * OPA policy file (loan.rego) defines complex authorization rules.
 */
@Slf4j
@Component
@RequiredArgsConstructor
public class OpaAuthorizationManager
        implements AuthorizationManager<RequestAuthorizationContext> {

    private final RestClient opaClient;
    private final ObjectMapper objectMapper;

    @Value("${opa.url:http://localhost:8181}")
    private String opaUrl;

    @Override
    public AuthorizationDecision check(Supplier<Authentication> authSupplier,
                                        RequestAuthorizationContext context) {
        Authentication auth = authSupplier.get();

        if (auth == null || !auth.isAuthenticated()) {
            return new AuthorizationDecision(false);
        }

        try {
            Map<String, Object> input = buildOpaInput(auth, context);
            boolean allowed = queryOpa(input);
            log.debug("OPA decision for {}: {}",
                context.getRequest().getRequestURI(), allowed);
            return new AuthorizationDecision(allowed);
        } catch (Exception e) {
            log.error("OPA query failed, denying access: {}", e.getMessage());
            return new AuthorizationDecision(false);  // Fail closed
        }
    }

    private Map<String, Object> buildOpaInput(Authentication auth,
                                               RequestAuthorizationContext context) {
        var request = context.getRequest();

        Map<String, Object> userInfo = Map.of();
        if (auth instanceof JwtAuthenticationToken jwtAuth) {
            userInfo = Map.of(
                "subject", jwtAuth.getToken().getSubject(),
                "roles", jwtAuth.getToken().getClaimAsStringList("roles"),
                "tenantId", jwtAuth.getToken().getClaimAsString("tenant_id"),
                "email", jwtAuth.getToken().getClaimAsString("email")
            );
        }

        return Map.of(
            "input", Map.of(
                "method", request.getMethod(),
                "path", request.getRequestURI().split("/"),
                "user", userInfo,
                "resource_id", context.getVariables().getOrDefault("id", ""),
                "headers", Map.of(
                    "x-tenant-id", request.getHeader("X-Tenant-ID")
                )
            )
        );
    }

    private boolean queryOpa(Map<String, Object> input) {
        Map response = opaClient.post()
            .uri(opaUrl + "/v1/data/loan/authz/allow")
            .body(input)
            .retrieve()
            .body(Map.class);

        if (response != null && response.containsKey("result")) {
            return Boolean.TRUE.equals(response.get("result"));
        }
        return false;
    }
}
```

```rego
# opa/policies/loan.rego
package loan.authz

import future.keywords.if
import future.keywords.in

default allow = false

# Admin can do anything
allow if {
    "ADMIN" in input.user.roles
}

# Applicants can read their own loans
allow if {
    input.method == "GET"
    input.path[2] == "loans"
    input.path[3] == input.user.subject
}

# Underwriters can approve/reject loans
allow if {
    input.method == "POST"
    input.path[2] == "loans"
    input.path[4] in {"approve", "reject"}
    "UNDERWRITER" in input.user.roles
}

# Services can only access resources in their own tenant
allow if {
    input.headers["x-tenant-id"] == input.user.tenantId
    input.method in {"GET", "POST"}
}
```

---

## Security Context Propagation

```java
// src/main/java/com/example/security/SecurityContextPropagationConfig.java
package com.example.security;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.scheduling.annotation.EnableAsync;
import org.springframework.security.concurrent.DelegatingSecurityContextExecutorService;
import org.springframework.security.task.DelegatingSecurityContextAsyncTaskExecutor;

import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

/**
 * Ensures SecurityContext is propagated to async threads and thread pools.
 * Without this, async methods lose the authentication context.
 */
@Configuration
@EnableAsync
public class SecurityContextPropagationConfig {

    @Bean("securityAwareExecutor")
    public ExecutorService securityAwareExecutor() {
        // Wraps a standard executor with security context propagation
        return new DelegatingSecurityContextExecutorService(
            Executors.newFixedThreadPool(10)
        );
    }
}
```

---

## Network Policies (Kubernetes)

```yaml
# k8s/network-policy.yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: loan-service-network-policy
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: loan-service
  policyTypes:
    - Ingress
    - Egress
  ingress:
    # Only accept traffic from API gateway
    - from:
        - podSelector:
            matchLabels:
              app: api-gateway
      ports:
        - protocol: TCP
          port: 8080
    # Allow from other internal services
    - from:
        - podSelector:
            matchLabels:
              tier: internal-service
      ports:
        - protocol: TCP
          port: 8080
    # Allow prometheus scraping
    - from:
        - namespaceSelector:
            matchLabels:
              name: monitoring
      ports:
        - protocol: TCP
          port: 9090
  egress:
    # Allow access to database
    - to:
        - podSelector:
            matchLabels:
              app: postgres
      ports:
        - protocol: TCP
          port: 5432
    # Allow access to Redis
    - to:
        - podSelector:
            matchLabels:
              app: redis
      ports:
        - protocol: TCP
          port: 6379
    # Allow access to auth service
    - to:
        - podSelector:
            matchLabels:
              app: keycloak
      ports:
        - protocol: TCP
          port: 8443
    # Allow DNS
    - to:
        - namespaceSelector: {}
      ports:
        - protocol: UDP
          port: 53
```

---

## API Gateway Security Layer

```java
// src/main/java/com/example/gateway/GatewaySecurityConfig.java
package com.example.gateway;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.cloud.gateway.filter.GlobalFilter;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.Ordered;
import org.springframework.core.annotation.Order;
import org.springframework.http.HttpStatus;
import org.springframework.http.server.reactive.ServerHttpRequest;
import org.springframework.web.server.ServerWebExchange;
import reactor.core.publisher.Mono;

import java.util.Set;

@Slf4j
@Configuration
@RequiredArgsConstructor
public class GatewaySecurityConfig {

    private final RateLimiterService rateLimiter;
    private final TokenBlacklistService blacklist;

    @Bean
    @Order(Ordered.HIGHEST_PRECEDENCE)
    public GlobalFilter rateLimitingFilter() {
        return (exchange, chain) -> {
            String clientIp = getClientIp(exchange);
            String path = exchange.getRequest().getPath().value();

            if (!rateLimiter.isAllowed(clientIp, path)) {
                log.warn("Rate limit exceeded for {} on {}", clientIp, path);
                exchange.getResponse().setStatusCode(HttpStatus.TOO_MANY_REQUESTS);
                exchange.getResponse().getHeaders()
                    .add("Retry-After", String.valueOf(rateLimiter.getRetryAfterSeconds(clientIp)));
                return exchange.getResponse().setComplete();
            }

            return chain.filter(exchange);
        };
    }

    @Bean
    @Order(Ordered.HIGHEST_PRECEDENCE + 1)
    public GlobalFilter tokenValidationFilter() {
        return (exchange, chain) -> {
            String token = extractToken(exchange);

            if (token != null && blacklist.isBlacklisted(token)) {
                log.warn("Blacklisted token used from {}",
                    getClientIp(exchange));
                exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
                return exchange.getResponse().setComplete();
            }

            return chain.filter(exchange);
        };
    }

    @Bean
    @Order(Ordered.HIGHEST_PRECEDENCE + 2)
    public GlobalFilter requestEnrichmentFilter() {
        return (exchange, chain) -> {
            // Strip hop-by-hop headers before forwarding
            ServerHttpRequest modifiedRequest = exchange.getRequest().mutate()
                .headers(headers -> {
                    headers.remove("X-Real-IP");
                    headers.remove("X-Forwarded-For");
                })
                // Add trusted headers that downstream services can rely on
                .header("X-Gateway-Request-Id",
                    java.util.UUID.randomUUID().toString())
                .header("X-Real-IP", getClientIp(exchange))
                .build();

            return chain.filter(exchange.mutate().request(modifiedRequest).build());
        };
    }

    private String getClientIp(ServerWebExchange exchange) {
        String forwarded = exchange.getRequest().getHeaders()
            .getFirst("X-Forwarded-For");
        if (forwarded != null) {
            return forwarded.split(",")[0].trim();
        }
        var remoteAddress = exchange.getRequest().getRemoteAddress();
        return remoteAddress != null ? remoteAddress.getAddress().getHostAddress() : "unknown";
    }

    private String extractToken(ServerWebExchange exchange) {
        String header = exchange.getRequest().getHeaders()
            .getFirst("Authorization");
        if (header != null && header.startsWith("Bearer ")) {
            return header.substring(7);
        }
        return null;
    }
}
```

---

## Security Scanning in CI/CD

```yaml
# .github/workflows/security-scan.yml
name: Security Scanning

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  dependency-check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: OWASP Dependency Check
        uses: dependency-check/Dependency-Check_Action@main
        with:
          project: 'loan-service'
          path: '.'
          format: 'HTML'
          args: >
            --failOnCVSS 7
            --enableRetired

      - name: Upload results
        uses: actions/upload-artifact@v3
        if: always()
        with:
          name: dependency-check-report
          path: reports/

  sast:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: SpotBugs / FindSecBugs
        run: mvn spotbugs:check -P security

      - name: Semgrep SAST
        uses: semgrep/semgrep-action@v1
        with:
          config: >-
            p/java
            p/spring
            p/owasp-top-ten

  container-scan:
    runs-on: ubuntu-latest
    needs: [dependency-check]
    steps:
      - uses: actions/checkout@v4

      - name: Build Docker image
        run: docker build -t loan-service:${{ github.sha }} .

      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: loan-service:${{ github.sha }}
          format: 'sarif'
          output: 'trivy-results.sarif'
          exit-code: '1'
          severity: 'CRITICAL,HIGH'

      - name: Upload Trivy scan results
        uses: github/codeql-action/upload-sarif@v2
        if: always()
        with:
          sarif_file: 'trivy-results.sarif'

  secret-detection:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0  # Full history for detecting old secrets

      - name: Detect secrets with Trufflehog
        uses: trufflesecurity/trufflehog@main
        with:
          path: ./
          base: ${{ github.event.repository.default_branch }}
          extra_args: --debug --only-verified
```

---

## Secure Inter-Service Communication

```java
// src/main/java/com/example/security/SecureServiceClient.java
package com.example.security;

import io.github.resilience4j.circuitbreaker.annotation.CircuitBreaker;
import io.github.resilience4j.retry.annotation.Retry;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;
import org.springframework.web.client.RestClient;

import java.util.Optional;

/**
 * Demonstrates secure service-to-service calls with:
 * - Service token (Client Credentials OAuth2)
 * - Circuit breaker
 * - Retry
 * - Audit logging
 */
@Slf4j
@Service
@RequiredArgsConstructor
public class SecureServiceClient {

    private final RestClient restClient;
    private final ServiceTokenProvider tokenProvider;
    private final AuditService auditService;

    @CircuitBreaker(name = "credit-service", fallbackMethod = "getCreditScoreFallback")
    @Retry(name = "credit-service")
    public CreditScoreResponse getCreditScore(String applicantId) {
        log.debug("Calling credit service for applicant: {}", applicantId);

        String token = tokenProvider.getServiceToken();

        try {
            CreditScoreResponse response = restClient.get()
                .uri("https://credit-service.internal/v1/scores/{id}", applicantId)
                .header("Authorization", "Bearer " + token)
                .header("X-Service-Name", "loan-service")
                .header("X-Request-ID", java.util.UUID.randomUUID().toString())
                .retrieve()
                .body(CreditScoreResponse.class);

            auditService.logServiceCall("credit-service", "getCreditScore",
                applicantId, "SUCCESS");
            return response;

        } catch (Exception e) {
            auditService.logServiceCall("credit-service", "getCreditScore",
                applicantId, "FAILURE: " + e.getMessage());
            throw e;
        }
    }

    private CreditScoreResponse getCreditScoreFallback(String applicantId, Exception e) {
        log.warn("Credit service circuit breaker open for applicant {}. Using default score.", applicantId);
        return CreditScoreResponse.defaultResponse(applicantId);
    }

    record CreditScoreResponse(String applicantId, int score, double dti, boolean isDefault) {
        static CreditScoreResponse defaultResponse(String applicantId) {
            return new CreditScoreResponse(applicantId, 0, 0.0, true);
        }
    }
}
```

---

## Summary

| Pattern | Technology | Use Case |
|---|---|---|
| Service authentication | mTLS | Services authenticate to each other |
| User token propagation | JWT Bearer header | Forward user identity across services |
| Service-to-service token | OAuth2 Client Credentials | Service acts as itself (not user) |
| Authorization policies | OPA (Rego) | Complex, externalized authorization rules |
| Secret management | AWS Secrets Manager / Vault | Dynamic secret rotation |
| Network isolation | Kubernetes NetworkPolicy | Restrict which pods can communicate |
| Gateway security | Spring Cloud Gateway filters | Rate limiting, token blacklist, header sanitization |
| Dependency scanning | OWASP, Trivy | Find CVEs in dependencies and images |
| SAST | Semgrep, SpotBugs | Detect code-level security issues |
| Secret detection | Trufflehog | Find accidentally committed secrets |

### Key Takeaways
- mTLS authenticates both endpoints; TLS only authenticates the server
- Propagate user JWT to downstream services to preserve the user's identity and audit trail
- Use Client Credentials for service-to-service calls where no user is involved
- OPA externalizes authorization — policies can be updated without redeploying services
- Fail closed on authorization errors — deny by default, not allow by default
- NetworkPolicies should restrict ingress AND egress, not just ingress
- Scan in CI/CD before every merge — shift security left

---

## Next Part Preview

**Part 099: Performance Testing and Optimization** covers profiling with async-profiler and JFR, heap dump analysis, GC tuning, load testing with Gatling and k6, and a complete case study optimizing a 5-second endpoint to under 50ms.
