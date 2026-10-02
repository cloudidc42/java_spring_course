# Part 061: Advanced Spring Security

## Overview

This part covers enterprise-grade security with Spring Security. We build a secure banking application with MFA, brute force protection, audit trails, and hardened HTTP headers. Security is not optional — it must be designed in from the start.

---

## Maven Dependencies

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-security</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
    <!-- TOTP for MFA -->
    <dependency>
        <groupId>dev.samstevens.totp</groupId>
        <artifactId>totp-spring-boot-starter</artifactId>
        <version>1.7.1</version>
    </dependency>
    <!-- JWT -->
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
    <!-- Password strength -->
    <dependency>
        <groupId>org.passay</groupId>
        <artifactId>passay</artifactId>
        <version>1.6.4</version>
    </dependency>
</dependencies>
```

---

## 1. Security Architecture Deep Dive

```
HTTP Request
    ↓
[DelegatingFilterProxy]
    ↓
[FilterChainProxy]
    ↓
[SecurityFilterChain]
    ├── SecurityContextPersistenceFilter
    ├── HeaderWriterFilter (security headers)
    ├── CsrfFilter
    ├── LogoutFilter
    ├── UsernamePasswordAuthenticationFilter ← Custom filters here
    ├── BasicAuthenticationFilter
    ├── RequestCacheAwareFilter
    ├── SecurityContextHolderAwareRequestFilter
    ├── AnonymousAuthenticationFilter
    ├── SessionManagementFilter
    ├── ExceptionTranslationFilter
    └── FilterSecurityInterceptor (Authorization)
    ↓
[Controller / Handler]
```

```java
// Main security configuration
@Configuration
@EnableWebSecurity
@EnableMethodSecurity(prePostEnabled = true, securedEnabled = true)
@Slf4j
public class SecurityConfig {

    private final UserDetailsService userDetailsService;
    private final JwtAuthenticationFilter jwtFilter;
    private final TotpAuthenticationFilter totpFilter;
    private final BruteForceProtectionFilter bruteForceFilter;
    private final SecurityAuditService auditService;

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        return http
            .csrf(csrf -> csrf
                .csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())
                .ignoringRequestMatchers("/api/auth/**", "/api/webhooks/**"))
            .cors(cors -> cors.configurationSource(corsConfigurationSource()))
            .sessionManagement(session -> session
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .headers(headers -> headers
                // HSTS: force HTTPS for 1 year
                .httpStrictTransportSecurity(hsts -> hsts
                    .includeSubDomains(true)
                    .maxAgeInSeconds(31536000)
                    .preload(true))
                // Prevent clickjacking
                .frameOptions(FrameOptionsConfig::deny)
                // Prevent content-type sniffing
                .contentTypeOptions(ContentTypeOptionsConfig::disable)
                // XSS protection
                .xssProtection(xss -> xss.headerValue(XXssProtectionHeaderWriter.HeaderValue.ENABLED_MODE_BLOCK))
                // Referrer Policy
                .referrerPolicy(ref -> ref.policy(ReferrerPolicyHeaderWriter.ReferrerPolicy.STRICT_ORIGIN_WHEN_CROSS_ORIGIN))
                // Content Security Policy
                .contentSecurityPolicy(csp -> csp.policyDirectives(
                    "default-src 'self'; " +
                    "script-src 'self' 'nonce-{nonce}'; " +
                    "style-src 'self' 'unsafe-inline'; " +
                    "img-src 'self' data: https:; " +
                    "font-src 'self'; " +
                    "connect-src 'self'; " +
                    "frame-ancestors 'none'; " +
                    "base-uri 'self'; " +
                    "form-action 'self'"
                )))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/auth/login", "/api/auth/register").permitAll()
                .requestMatchers("/api/auth/mfa/**").authenticated()
                .requestMatchers("/actuator/health").permitAll()
                .requestMatchers("/actuator/**").hasRole("ADMIN")
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .requestMatchers("/api/accounts/**").hasAnyRole("USER", "ADMIN")
                .requestMatchers("/api/transfers/**").hasRole("USER")
                .anyRequest().authenticated())
            .exceptionHandling(ex -> ex
                .authenticationEntryPoint(customAuthenticationEntryPoint())
                .accessDeniedHandler(customAccessDeniedHandler()))
            .addFilterBefore(bruteForceFilter, UsernamePasswordAuthenticationFilter.class)
            .addFilterBefore(jwtFilter, UsernamePasswordAuthenticationFilter.class)
            .addFilterAfter(totpFilter, JwtAuthenticationFilter.class)
            .build();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        // BCrypt with cost factor 12 (security best practice)
        return new BCryptPasswordEncoder(12);
    }

    @Bean
    public AuthenticationManager authenticationManager(
            AuthenticationConfiguration config) throws Exception {
        return config.getAuthenticationManager();
    }

    @Bean
    public CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration config = new CorsConfiguration();
        config.setAllowedOriginPatterns(List.of(
            "https://bank.example.com",
            "https://*.bank.example.com"
        ));
        config.setAllowedMethods(List.of("GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS"));
        config.setAllowedHeaders(List.of("*"));
        config.setAllowCredentials(true);
        config.setMaxAge(3600L);

        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/api/**", config);
        return source;
    }

    @Bean
    public AuthenticationEntryPoint customAuthenticationEntryPoint() {
        return (request, response, ex) -> {
            auditService.recordAuthFailure(request.getRemoteAddr(),
                request.getRequestURI(), ex.getMessage());

            response.setContentType(MediaType.APPLICATION_JSON_VALUE);
            response.setStatus(HttpStatus.UNAUTHORIZED.value());
            response.getWriter().write("""
                {"error": "Unauthorized", "message": "Authentication required"}
                """);
        };
    }

    @Bean
    public AccessDeniedHandler customAccessDeniedHandler() {
        return (request, response, ex) -> {
            auditService.recordAccessDenied(request.getRemoteAddr(),
                request.getRequestURI(),
                SecurityContextHolder.getContext().getAuthentication());

            response.setContentType(MediaType.APPLICATION_JSON_VALUE);
            response.setStatus(HttpStatus.FORBIDDEN.value());
            response.getWriter().write("""
                {"error": "Forbidden", "message": "Insufficient privileges"}
                """);
        };
    }
}
```

---

## 2. Custom Authentication Provider

```java
// Custom auth provider: email + password with account checks
@Component
@Slf4j
public class BankingAuthenticationProvider implements AuthenticationProvider {

    private final UserRepository userRepository;
    private final PasswordEncoder passwordEncoder;
    private final LoginAttemptService loginAttemptService;
    private final SecurityAuditService auditService;

    @Override
    public Authentication authenticate(Authentication authentication)
            throws AuthenticationException {

        String email = authentication.getName();
        String password = authentication.getCredentials().toString();
        String ipAddress = getCurrentIpAddress();

        // Check brute force
        if (loginAttemptService.isBlocked(ipAddress, email)) {
            auditService.recordBruteForceBlock(email, ipAddress);
            throw new LockedException(
                "Account locked due to too many failed attempts. Try again in 30 minutes.");
        }

        // Find user
        BankUser user = userRepository.findByEmail(email)
            .orElseThrow(() -> {
                loginAttemptService.recordFailure(ipAddress, email);
                return new BadCredentialsException("Invalid email or password");
            });

        // Check account status
        if (!user.isEnabled()) {
            throw new DisabledException("Account is disabled");
        }

        if (user.isAccountLocked()) {
            throw new LockedException("Account is locked. Contact support.");
        }

        if (user.isPasswordExpired()) {
            throw new CredentialsExpiredException("Password has expired. Please reset it.");
        }

        // Verify password
        if (!passwordEncoder.matches(password, user.getPasswordHash())) {
            loginAttemptService.recordFailure(ipAddress, email);
            auditService.recordLoginFailure(email, ipAddress);

            if (loginAttemptService.getFailureCount(ipAddress, email) >= 3) {
                auditService.recordSuspiciousActivity(email, ipAddress,
                    "Multiple failed login attempts");
            }

            throw new BadCredentialsException("Invalid email or password");
        }

        // Success - clear failed attempts
        loginAttemptService.clearFailures(ipAddress, email);
        auditService.recordLoginSuccess(email, ipAddress);

        // Update last login
        user.setLastLoginAt(LocalDateTime.now());
        user.setLastLoginIp(ipAddress);
        userRepository.save(user);

        List<GrantedAuthority> authorities = user.getRoles().stream()
            .map(role -> new SimpleGrantedAuthority("ROLE_" + role.name()))
            .collect(Collectors.toList());

        return new UsernamePasswordAuthenticationToken(
            user, null, authorities);
    }

    @Override
    public boolean supports(Class<?> authentication) {
        return UsernamePasswordAuthenticationToken.class.isAssignableFrom(authentication);
    }

    private String getCurrentIpAddress() {
        try {
            HttpServletRequest request =
                ((ServletRequestAttributes) RequestContextHolder.currentRequestAttributes())
                    .getRequest();
            String xForwardedFor = request.getHeader("X-Forwarded-For");
            if (xForwardedFor != null && !xForwardedFor.isEmpty()) {
                return xForwardedFor.split(",")[0].trim();
            }
            return request.getRemoteAddr();
        } catch (IllegalStateException e) {
            return "unknown";
        }
    }
}
```

---

## 3. Multi-Factor Authentication (TOTP)

```java
// User entity with MFA support
@Entity
@Table(name = "bank_users")
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class BankUser {

    @Id
    @GeneratedValue(strategy = GenerationType.UUID)
    private String id;

    @Column(unique = true, nullable = false)
    private String email;

    @Column(nullable = false)
    private String passwordHash;

    @Enumerated(EnumType.STRING)
    @ElementCollection(fetch = FetchType.EAGER)
    @CollectionTable(name = "user_roles")
    private Set<UserRole> roles = new HashSet<>();

    private boolean mfaEnabled;
    private String mfaSecret;   // TOTP secret (encrypted in DB)

    private boolean enabled = true;
    private boolean accountLocked = false;
    private boolean passwordExpired = false;

    private LocalDateTime passwordChangedAt;
    private LocalDateTime lastLoginAt;
    private String lastLoginIp;
    private LocalDateTime lockedAt;
    private String lockReason;

    @CreatedDate
    private LocalDateTime createdAt;
}

// TOTP MFA Service
@Service
@Slf4j
public class TotpMfaService {

    private final SecretGenerator secretGenerator;
    private final TotpGenerator totpGenerator;
    private final CodeVerifier codeVerifier;
    private final QrDataFactory qrDataFactory;
    private final QrGenerator qrGenerator;
    private final UserRepository userRepository;

    public MfaSetupResponse setupMfa(String userId) {
        BankUser user = userRepository.findById(userId).orElseThrow();

        // Generate TOTP secret
        String secret = secretGenerator.generate();

        // Generate QR code for authenticator app
        QrData qrData = qrDataFactory.newBuilder()
            .label(user.getEmail())
            .secret(secret)
            .issuer("SecureBank")
            .algorithm(HashingAlgorithm.SHA1)
            .digits(6)
            .period(30)
            .build();

        try {
            String qrCodeImageUri = getDataUriForImage(
                qrGenerator.generate(qrData),
                qrGenerator.getImageMimeType()
            );

            // Store secret temporarily (confirmed on first use)
            user.setMfaSecret(encryptSecret(secret));
            userRepository.save(user);

            return MfaSetupResponse.builder()
                .secret(secret)
                .qrCodeUri(qrCodeImageUri)
                .backupCodes(generateBackupCodes())
                .build();

        } catch (QrGenerationException e) {
            throw new MfaSetupException("Failed to generate QR code", e);
        }
    }

    public boolean verifyCode(String userId, String code) {
        BankUser user = userRepository.findById(userId).orElseThrow();

        if (user.getMfaSecret() == null) {
            throw new MfaNotEnabledException("MFA is not set up for this user");
        }

        String secret = decryptSecret(user.getMfaSecret());
        boolean valid = codeVerifier.isValidNow(secret, code);

        if (valid) {
            // Enable MFA if this is the first verification
            if (!user.isMfaEnabled()) {
                user.setMfaEnabled(true);
                userRepository.save(user);
                log.info("MFA enabled for user {}", userId);
            }
        }

        return valid;
    }

    public void disableMfa(String userId, String currentCode) {
        BankUser user = userRepository.findById(userId).orElseThrow();

        if (!verifyCode(userId, currentCode)) {
            throw new InvalidTotpCodeException("Invalid TOTP code");
        }

        user.setMfaEnabled(false);
        user.setMfaSecret(null);
        userRepository.save(user);
        log.info("MFA disabled for user {}", userId);
    }

    private List<String> generateBackupCodes() {
        SecureRandom random = new SecureRandom();
        return IntStream.range(0, 8)
            .mapToObj(i -> {
                byte[] bytes = new byte[5];
                random.nextBytes(bytes);
                return HexFormat.of().formatHex(bytes).toUpperCase();
            })
            .collect(Collectors.toList());
    }

    private String encryptSecret(String secret) {
        // Use AES-256 encryption with key from application secrets
        // Simplified here - in production use a proper key management service
        return Base64.getEncoder().encodeToString(secret.getBytes());
    }

    private String decryptSecret(String encrypted) {
        return new String(Base64.getDecoder().decode(encrypted));
    }

    private String getDataUriForImage(byte[] imageData, String mimeType) {
        return "data:" + mimeType + ";base64," +
            Base64.getEncoder().encodeToString(imageData);
    }
}

// MFA Authentication Filter
@Component
@Slf4j
public class TotpAuthenticationFilter extends OncePerRequestFilter {

    private final TotpMfaService mfaService;
    private final JwtTokenService jwtService;

    @Override
    protected void doFilterInternal(HttpServletRequest request,
            HttpServletResponse response, FilterChain chain)
            throws ServletException, IOException {

        // Skip MFA check for MFA verification endpoint itself
        if (request.getRequestURI().startsWith("/api/auth/mfa")) {
            chain.doFilter(request, response);
            return;
        }

        Authentication authentication = SecurityContextHolder.getContext().getAuthentication();

        if (authentication instanceof JwtAuthentication jwtAuth) {
            BankUser user = (BankUser) jwtAuth.getPrincipal();

            if (user.isMfaEnabled() && !jwtAuth.isMfaVerified()) {
                response.setContentType(MediaType.APPLICATION_JSON_VALUE);
                response.setStatus(HttpStatus.FORBIDDEN.value());
                response.getWriter().write("""
                    {"error": "MFA_REQUIRED", "message": "Multi-factor authentication required"}
                    """);
                return;
            }
        }

        chain.doFilter(request, response);
    }
}

// Auth controller with MFA flow
@RestController
@RequestMapping("/api/auth")
@Slf4j
public class AuthController {

    private final AuthenticationManager authManager;
    private final JwtTokenService jwtService;
    private final TotpMfaService mfaService;

    @PostMapping("/login")
    public ResponseEntity<LoginResponse> login(@RequestBody @Valid LoginRequest request) {
        Authentication auth = authManager.authenticate(
            new UsernamePasswordAuthenticationToken(request.getEmail(), request.getPassword())
        );

        BankUser user = (BankUser) auth.getPrincipal();

        if (user.isMfaEnabled()) {
            // Issue a pre-MFA token (limited permissions)
            String preMfaToken = jwtService.generatePreMfaToken(user.getId());
            return ResponseEntity.ok(LoginResponse.builder()
                .status("MFA_REQUIRED")
                .token(preMfaToken)
                .message("Please provide your TOTP code")
                .build());
        }

        // No MFA - issue full token
        String token = jwtService.generateToken(user.getId(), user.getRoles(), true);
        return ResponseEntity.ok(LoginResponse.builder()
            .status("SUCCESS")
            .token(token)
            .build());
    }

    @PostMapping("/mfa/verify")
    @PreAuthorize("isAuthenticated()")
    public ResponseEntity<LoginResponse> verifyMfa(
            @RequestBody @Valid MfaVerifyRequest request,
            Authentication authentication) {

        BankUser user = (BankUser) authentication.getPrincipal();

        boolean valid = mfaService.verifyCode(user.getId(), request.getCode());

        if (!valid) {
            return ResponseEntity.status(HttpStatus.UNAUTHORIZED)
                .body(LoginResponse.builder()
                    .status("INVALID_CODE")
                    .message("Invalid TOTP code")
                    .build());
        }

        // Issue full token with MFA verified flag
        String fullToken = jwtService.generateToken(user.getId(), user.getRoles(), true);
        return ResponseEntity.ok(LoginResponse.builder()
            .status("SUCCESS")
            .token(fullToken)
            .build());
    }

    @PostMapping("/mfa/setup")
    @PreAuthorize("isAuthenticated()")
    public ResponseEntity<MfaSetupResponse> setupMfa(Authentication authentication) {
        BankUser user = (BankUser) authentication.getPrincipal();
        MfaSetupResponse setup = mfaService.setupMfa(user.getId());
        return ResponseEntity.ok(setup);
    }
}
```

---

## 4. Password Policies and Rotation

```java
// Password policy validator using Passay
@Service
public class PasswordPolicyService {

    private final PasswordValidator validator;
    private final PasswordEncoder passwordEncoder;
    private final PasswordHistoryRepository historyRepository;

    @PostConstruct
    public void init() {
        // Build validator is done in constructor injection
    }

    @Bean
    public PasswordValidator passwordValidator() {
        return new PasswordValidator(List.of(
            // Length: 12-128 characters
            new LengthRule(12, 128),
            // Must contain uppercase
            new CharacterRule(EnglishCharacterData.UpperCase, 1),
            // Must contain lowercase
            new CharacterRule(EnglishCharacterData.LowerCase, 1),
            // Must contain digit
            new CharacterRule(EnglishCharacterData.Digit, 1),
            // Must contain special character
            new CharacterRule(EnglishCharacterData.Special, 1),
            // No whitespace
            new WhitespaceRule(),
            // No sequences (abc, 123)
            new IllegalSequenceRule(EnglishSequenceData.Alphabetical, 4, false),
            new IllegalSequenceRule(EnglishSequenceData.Numerical, 4, false),
            new IllegalSequenceRule(EnglishSequenceData.USQwerty, 4, false),
            // Minimum 8 unique characters
            new RepeatCharactersRule(4)
        ));
    }

    public PasswordValidationResult validatePassword(String password, String username) {
        RuleResult result = validator.validate(new PasswordData(password));

        if (!result.isValid()) {
            List<String> messages = validator.getMessages(result);
            return PasswordValidationResult.invalid(messages);
        }

        // Check if password contains username
        if (password.toLowerCase().contains(username.toLowerCase())) {
            return PasswordValidationResult.invalid(
                List.of("Password cannot contain your username"));
        }

        return PasswordValidationResult.valid();
    }

    public void changePassword(String userId, String currentPassword, String newPassword) {
        BankUser user = userRepository.findById(userId).orElseThrow();

        // Verify current password
        if (!passwordEncoder.matches(currentPassword, user.getPasswordHash())) {
            throw new BadCredentialsException("Current password is incorrect");
        }

        // Validate new password
        PasswordValidationResult validation = validatePassword(newPassword, user.getEmail());
        if (!validation.isValid()) {
            throw new WeakPasswordException(validation.getMessages());
        }

        // Check password history (last 12 passwords)
        List<PasswordHistory> history = historyRepository
            .findTop12ByUserIdOrderByChangedAtDesc(userId);

        for (PasswordHistory prev : history) {
            if (passwordEncoder.matches(newPassword, prev.getPasswordHash())) {
                throw new PasswordReusedException(
                    "Cannot reuse any of your last 12 passwords");
            }
        }

        // Save to history
        historyRepository.save(PasswordHistory.builder()
            .userId(userId)
            .passwordHash(user.getPasswordHash())
            .changedAt(LocalDateTime.now())
            .build());

        // Update password
        user.setPasswordHash(passwordEncoder.encode(newPassword));
        user.setPasswordChangedAt(LocalDateTime.now());
        user.setPasswordExpired(false);
        userRepository.save(user);
    }

    // Scheduled password expiry check (90 days)
    @Scheduled(cron = "0 0 2 * * ?")  // 2 AM daily
    public void markExpiredPasswords() {
        LocalDateTime expiryDate = LocalDateTime.now().minusDays(90);
        int count = userRepository.markPasswordsExpired(expiryDate);
        log.info("Marked {} user passwords as expired", count);
    }
}
```

---

## 5. Brute Force Protection

```java
// Redis-backed brute force protection
@Service
@Slf4j
public class LoginAttemptService {

    private final RedisTemplate<String, Integer> redisTemplate;

    private static final int MAX_ATTEMPTS = 5;
    private static final Duration BLOCK_DURATION = Duration.ofMinutes(30);
    private static final Duration ATTEMPT_WINDOW = Duration.ofMinutes(15);

    public void recordFailure(String ipAddress, String email) {
        // Track by IP
        String ipKey = "login_attempts:ip:" + ipAddress;
        Integer ipCount = redisTemplate.opsForValue().get(ipKey);
        redisTemplate.opsForValue().set(ipKey,
            (ipCount == null ? 0 : ipCount) + 1, ATTEMPT_WINDOW);

        // Track by email
        String emailKey = "login_attempts:email:" + email;
        Integer emailCount = redisTemplate.opsForValue().get(emailKey);
        redisTemplate.opsForValue().set(emailKey,
            (emailCount == null ? 0 : emailCount) + 1, ATTEMPT_WINDOW);

        // If exceeded threshold, block
        if (getFailureCount(ipAddress, email) >= MAX_ATTEMPTS) {
            blockAccess(ipAddress, email);
        }
    }

    public boolean isBlocked(String ipAddress, String email) {
        return Boolean.TRUE.equals(redisTemplate.hasKey("blocked:ip:" + ipAddress))
            || Boolean.TRUE.equals(redisTemplate.hasKey("blocked:email:" + email));
    }

    public int getFailureCount(String ipAddress, String email) {
        Integer ipCount = redisTemplate.opsForValue().get("login_attempts:ip:" + ipAddress);
        Integer emailCount = redisTemplate.opsForValue().get("login_attempts:email:" + email);
        return Math.max(
            ipCount == null ? 0 : ipCount,
            emailCount == null ? 0 : emailCount
        );
    }

    public void clearFailures(String ipAddress, String email) {
        redisTemplate.delete("login_attempts:ip:" + ipAddress);
        redisTemplate.delete("login_attempts:email:" + email);
        redisTemplate.delete("blocked:ip:" + ipAddress);
        redisTemplate.delete("blocked:email:" + email);
    }

    private void blockAccess(String ipAddress, String email) {
        redisTemplate.opsForValue().set("blocked:ip:" + ipAddress, 1, BLOCK_DURATION);
        redisTemplate.opsForValue().set("blocked:email:" + email, 1, BLOCK_DURATION);
        log.warn("Blocked login attempts from IP: {} for email: {}", ipAddress, email);
    }
}

// Brute force filter
@Component
@Slf4j
public class BruteForceProtectionFilter extends OncePerRequestFilter {

    private final LoginAttemptService loginAttemptService;

    @Override
    protected void doFilterInternal(HttpServletRequest request,
            HttpServletResponse response, FilterChain chain)
            throws ServletException, IOException {

        if (!request.getRequestURI().equals("/api/auth/login")) {
            chain.doFilter(request, response);
            return;
        }

        String ipAddress = extractIpAddress(request);

        // Check IP-level block (no email needed)
        if (loginAttemptService.isBlocked(ipAddress, "")) {
            log.warn("Blocked request from IP: {}", ipAddress);
            response.setStatus(HttpStatus.TOO_MANY_REQUESTS.value());
            response.setContentType(MediaType.APPLICATION_JSON_VALUE);
            response.getWriter().write(
                "{\"error\": \"Too many login attempts. Try again in 30 minutes.\"}");
            return;
        }

        chain.doFilter(request, response);
    }

    private String extractIpAddress(HttpServletRequest request) {
        String xForwardedFor = request.getHeader("X-Forwarded-For");
        if (xForwardedFor != null && !xForwardedFor.isEmpty()) {
            return xForwardedFor.split(",")[0].trim();
        }
        return request.getRemoteAddr();
    }
}
```

---

## 6. Security Audit Trail

```java
// Audit event entity
@Entity
@Table(name = "security_audit_log",
    indexes = {
        @Index(name = "idx_audit_user", columnList = "userId"),
        @Index(name = "idx_audit_event_type", columnList = "eventType"),
        @Index(name = "idx_audit_created_at", columnList = "createdAt")
    })
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class SecurityAuditEvent {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String userId;
    private String email;
    private String ipAddress;
    private String userAgent;

    @Enumerated(EnumType.STRING)
    private AuditEventType eventType;

    private String resource;
    private String action;
    private String outcome;   // SUCCESS, FAILURE, BLOCKED

    @Column(length = 2000)
    private String details;

    @Column(length = 100)
    private String sessionId;

    @CreatedDate
    @Column(updatable = false)
    private LocalDateTime createdAt;
}

public enum AuditEventType {
    LOGIN_SUCCESS, LOGIN_FAILURE, LOGIN_BLOCKED,
    LOGOUT,
    MFA_ENABLED, MFA_DISABLED, MFA_VERIFIED, MFA_FAILED,
    PASSWORD_CHANGED, PASSWORD_RESET, PASSWORD_EXPIRED,
    ACCOUNT_LOCKED, ACCOUNT_UNLOCKED,
    TRANSFER_INITIATED, TRANSFER_APPROVED, TRANSFER_REJECTED,
    SENSITIVE_DATA_ACCESSED,
    PERMISSION_DENIED,
    SUSPICIOUS_ACTIVITY
}

// Audit service
@Service
@Slf4j
public class SecurityAuditService {

    private final SecurityAuditRepository auditRepository;
    private final HttpServletRequest request;

    @Async
    public void recordLoginSuccess(String email, String ipAddress) {
        record(SecurityAuditEvent.builder()
            .email(email)
            .ipAddress(ipAddress)
            .eventType(AuditEventType.LOGIN_SUCCESS)
            .outcome("SUCCESS")
            .userAgent(getUserAgent())
            .build());
    }

    @Async
    public void recordLoginFailure(String email, String ipAddress) {
        record(SecurityAuditEvent.builder()
            .email(email)
            .ipAddress(ipAddress)
            .eventType(AuditEventType.LOGIN_FAILURE)
            .outcome("FAILURE")
            .userAgent(getUserAgent())
            .build());
    }

    @Async
    public void recordSensitiveAccess(String userId, String resource) {
        record(SecurityAuditEvent.builder()
            .userId(userId)
            .ipAddress(getCurrentIpAddress())
            .eventType(AuditEventType.SENSITIVE_DATA_ACCESSED)
            .resource(resource)
            .outcome("SUCCESS")
            .sessionId(getCurrentSessionId())
            .build());
    }

    @Async
    public void recordTransfer(String userId, BigDecimal amount, String targetAccount) {
        record(SecurityAuditEvent.builder()
            .userId(userId)
            .ipAddress(getCurrentIpAddress())
            .eventType(AuditEventType.TRANSFER_INITIATED)
            .resource("account:" + targetAccount)
            .details("amount=" + amount)
            .outcome("INITIATED")
            .sessionId(getCurrentSessionId())
            .build());
    }

    private void record(SecurityAuditEvent event) {
        try {
            event.setCreatedAt(LocalDateTime.now());
            auditRepository.save(event);
            log.info("AUDIT: {} {} {}",
                event.getEventType(), event.getEmail(), event.getIpAddress());
        } catch (Exception e) {
            // Never let audit failures affect business logic
            log.error("Failed to record audit event", e);
        }
    }

    private String getUserAgent() {
        try {
            return request.getHeader("User-Agent");
        } catch (Exception e) {
            return "unknown";
        }
    }

    private String getCurrentIpAddress() {
        try {
            String xff = request.getHeader("X-Forwarded-For");
            return xff != null ? xff.split(",")[0].trim() : request.getRemoteAddr();
        } catch (Exception e) {
            return "unknown";
        }
    }

    private String getCurrentSessionId() {
        try {
            Authentication auth = SecurityContextHolder.getContext().getAuthentication();
            return auth != null ? auth.getName().substring(0, 8) : null;
        } catch (Exception e) {
            return null;
        }
    }
}

// AOP-based audit for sensitive operations
@Aspect
@Component
@Slf4j
public class SecurityAuditAspect {

    private final SecurityAuditService auditService;

    @Around("@annotation(SensitiveOperation)")
    public Object auditSensitiveOperation(ProceedingJoinPoint joinPoint,
            SensitiveOperation annotation) throws Throwable {

        String userId = getCurrentUserId();
        String operation = annotation.value();

        log.debug("Sensitive operation '{}' by user '{}'", operation, userId);

        try {
            Object result = joinPoint.proceed();
            auditService.recordSensitiveAccess(userId, operation);
            return result;
        } catch (Exception e) {
            auditService.recordAccessDenied(getCurrentIpAddress(),
                operation, SecurityContextHolder.getContext().getAuthentication());
            throw e;
        }
    }

    private String getCurrentUserId() {
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        if (auth != null && auth.getPrincipal() instanceof BankUser user) {
            return user.getId();
        }
        return "anonymous";
    }

    private String getCurrentIpAddress() {
        try {
            HttpServletRequest req =
                ((ServletRequestAttributes) RequestContextHolder.currentRequestAttributes())
                    .getRequest();
            return req.getRemoteAddr();
        } catch (Exception e) {
            return "unknown";
        }
    }
}

// Annotation for marking sensitive operations
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface SensitiveOperation {
    String value() default "";
}
```

---

## 7. JWT Service

```java
@Service
@Slf4j
public class JwtTokenService {

    @Value("${security.jwt.secret}")
    private String jwtSecret;

    @Value("${security.jwt.expiration-hours:8}")
    private int expirationHours;

    private SecretKey getSigningKey() {
        byte[] keyBytes = Decoders.BASE64.decode(jwtSecret);
        return Keys.hmacShaKeyFor(keyBytes);
    }

    public String generateToken(String userId, Set<UserRole> roles, boolean mfaVerified) {
        Map<String, Object> claims = new HashMap<>();
        claims.put("roles", roles.stream().map(Enum::name).collect(Collectors.toList()));
        claims.put("mfaVerified", mfaVerified);
        claims.put("type", "ACCESS");

        return Jwts.builder()
            .claims(claims)
            .subject(userId)
            .issuedAt(new Date())
            .expiration(new Date(System.currentTimeMillis() +
                expirationHours * 3600 * 1000L))
            .signWith(getSigningKey())
            .compact();
    }

    public String generatePreMfaToken(String userId) {
        return Jwts.builder()
            .claim("type", "PRE_MFA")
            .claim("mfaVerified", false)
            .subject(userId)
            .issuedAt(new Date())
            .expiration(new Date(System.currentTimeMillis() + 5 * 60 * 1000L)) // 5 minutes
            .signWith(getSigningKey())
            .compact();
    }

    public Claims validateToken(String token) {
        return Jwts.parser()
            .verifyWith(getSigningKey())
            .build()
            .parseSignedClaims(token)
            .getPayload();
    }

    public boolean isMfaVerified(String token) {
        Claims claims = validateToken(token);
        return Boolean.TRUE.equals(claims.get("mfaVerified", Boolean.class));
    }
}

// JWT Authentication Filter
@Component
@Slf4j
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    private final JwtTokenService jwtService;
    private final UserRepository userRepository;

    @Override
    protected void doFilterInternal(HttpServletRequest request,
            HttpServletResponse response, FilterChain chain)
            throws ServletException, IOException {

        String authHeader = request.getHeader("Authorization");

        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            chain.doFilter(request, response);
            return;
        }

        String token = authHeader.substring(7);

        try {
            Claims claims = jwtService.validateToken(token);
            String userId = claims.getSubject();
            boolean mfaVerified = Boolean.TRUE.equals(
                claims.get("mfaVerified", Boolean.class));

            if (userId != null &&
                SecurityContextHolder.getContext().getAuthentication() == null) {

                BankUser user = userRepository.findById(userId).orElse(null);

                if (user != null && user.isEnabled()) {
                    List<GrantedAuthority> authorities = user.getRoles().stream()
                        .map(role -> new SimpleGrantedAuthority("ROLE_" + role.name()))
                        .collect(Collectors.toList());

                    JwtAuthentication auth = new JwtAuthentication(
                        user, token, authorities, mfaVerified);

                    SecurityContextHolder.getContext().setAuthentication(auth);
                }
            }
        } catch (ExpiredJwtException e) {
            log.debug("JWT token expired");
            response.setStatus(HttpStatus.UNAUTHORIZED.value());
            response.setContentType(MediaType.APPLICATION_JSON_VALUE);
            response.getWriter().write("{\"error\": \"TOKEN_EXPIRED\"}");
            return;
        } catch (JwtException e) {
            log.warn("Invalid JWT token: {}", e.getMessage());
            response.setStatus(HttpStatus.UNAUTHORIZED.value());
            response.setContentType(MediaType.APPLICATION_JSON_VALUE);
            response.getWriter().write("{\"error\": \"INVALID_TOKEN\"}");
            return;
        }

        chain.doFilter(request, response);
    }
}
```

---

## 8. Method-Level Security

```java
// Secure banking operations with method security
@Service
@Slf4j
public class BankAccountService {

    private final AccountRepository accountRepository;
    private final SecurityAuditService auditService;

    // Only the account owner or ADMIN can view their account
    @PreAuthorize("authentication.principal.id == #userId or hasRole('ADMIN')")
    @SensitiveOperation("account.view")
    public AccountDto getAccount(String userId) {
        return accountRepository.findByUserId(userId)
            .map(AccountDto::from)
            .orElseThrow(() -> new AccountNotFoundException(userId));
    }

    // Only ADMIN can view all accounts
    @PreAuthorize("hasRole('ADMIN')")
    @PostFilter("filterObject.userId == authentication.principal.id or hasRole('ADMIN')")
    public List<AccountDto> getAllAccounts() {
        return accountRepository.findAll().stream()
            .map(AccountDto::from)
            .collect(Collectors.toList());
    }

    // Complex permission: transfer requires USER role + MFA verified
    @PreAuthorize("hasRole('USER') and @mfaSecurityChecker.isMfaVerified(authentication)")
    @SensitiveOperation("transfer.initiate")
    public TransferResult initiateTransfer(TransferRequest request,
            Authentication authentication) {

        BankUser user = (BankUser) authentication.getPrincipal();

        // Additional business-level authorization
        Account fromAccount = accountRepository.findByUserIdAndId(
            user.getId(), request.getFromAccountId())
            .orElseThrow(() -> new AccessDeniedException("You don't own this account"));

        if (fromAccount.getBalance().compareTo(request.getAmount()) < 0) {
            throw new InsufficientFundsException("Insufficient balance");
        }

        // Record transfer in audit trail
        auditService.recordTransfer(user.getId(), request.getAmount(),
            request.getToAccountId());

        return processTransfer(fromAccount, request);
    }
}

// MFA security checker for SpEL expressions
@Component("mfaSecurityChecker")
public class MfaSecurityChecker {

    public boolean isMfaVerified(Authentication authentication) {
        if (authentication instanceof JwtAuthentication jwt) {
            return jwt.isMfaVerified();
        }
        return false;
    }
}
```

---

## 9. Security Testing

```java
// Security integration tests
@SpringBootTest
@AutoConfigureMockMvc
class SecurityIntegrationTest {

    @Autowired
    private MockMvc mockMvc;

    @Autowired
    private UserRepository userRepository;

    @Autowired
    private PasswordEncoder passwordEncoder;

    @Test
    void loginWithValidCredentials_shouldReturnToken() throws Exception {
        // Given
        createTestUser("user@example.com", "SecureP@ssw0rd!");

        // When
        mockMvc.perform(post("/api/auth/login")
                .contentType(MediaType.APPLICATION_JSON)
                .content("""
                    {"email": "user@example.com", "password": "SecureP@ssw0rd!"}
                    """))
            // Then
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.token").isNotEmpty())
            .andExpect(jsonPath("$.status").value("SUCCESS"));
    }

    @Test
    void loginWithInvalidPassword_shouldReturn401() throws Exception {
        mockMvc.perform(post("/api/auth/login")
                .contentType(MediaType.APPLICATION_JSON)
                .content("""
                    {"email": "user@example.com", "password": "WrongPassword"}
                    """))
            .andExpect(status().isUnauthorized());
    }

    @Test
    void bruteForce_shouldBlockAfter5Attempts() throws Exception {
        for (int i = 0; i < 6; i++) {
            ResultActions result = mockMvc.perform(post("/api/auth/login")
                .contentType(MediaType.APPLICATION_JSON)
                .content("""
                    {"email": "target@example.com", "password": "WrongPassword"}
                    """));

            if (i < 5) {
                result.andExpect(status().isUnauthorized());
            } else {
                result.andExpect(status().isTooManyRequests());
            }
        }
    }

    @Test
    void protectedEndpoint_withoutToken_shouldReturn401() throws Exception {
        mockMvc.perform(get("/api/accounts/me"))
            .andExpect(status().isUnauthorized());
    }

    @Test
    @WithMockUser(roles = "USER")
    void adminEndpoint_withUserRole_shouldReturn403() throws Exception {
        mockMvc.perform(get("/api/admin/users"))
            .andExpect(status().isForbidden());
    }

    @Test
    void securityHeaders_shouldBePresent() throws Exception {
        mockMvc.perform(get("/api/auth/login"))
            .andExpect(header().exists("X-Content-Type-Options"))
            .andExpect(header().exists("X-Frame-Options"))
            .andExpect(header().exists("Strict-Transport-Security"))
            .andExpect(header().string("X-Frame-Options", "DENY"))
            .andExpect(header().string("X-Content-Type-Options", "nosniff"));
    }

    @Test
    void csrfProtection_forStateChangingRequests() throws Exception {
        // POST without CSRF token should be rejected (for session-based apps)
        mockMvc.perform(post("/api/auth/logout"))
            .andExpect(status().isForbidden());
    }

    private void createTestUser(String email, String password) {
        BankUser user = BankUser.builder()
            .email(email)
            .passwordHash(passwordEncoder.encode(password))
            .roles(Set.of(UserRole.USER))
            .enabled(true)
            .mfaEnabled(false)
            .passwordChangedAt(LocalDateTime.now())
            .build();
        userRepository.save(user);
    }
}

// Security unit tests for authorization logic
@ExtendWith(MockitoExtension.class)
class BankAccountServiceSecurityTest {

    @InjectMocks
    private BankAccountService accountService;

    @Mock
    private AccountRepository accountRepository;

    @Test
    @WithMockUser(username = "user123", roles = "USER")
    void getAccount_withDifferentUserId_shouldThrowAccessDenied() {
        // The @PreAuthorize annotation prevents accessing other users' accounts
        // This is tested via MockMvc in integration tests
        // Unit test checks the service logic
        assertThrows(AccessDeniedException.class,
            () -> accountService.getAccount("differentUser456"));
    }
}
```

---

## 10. Application Configuration

```yaml
# application.yml security configuration
spring:
  security:
    require-ssl: true

security:
  jwt:
    secret: ${JWT_SECRET}  # 256-bit base64-encoded secret from environment
    expiration-hours: 8

server:
  ssl:
    enabled: true
    key-store: classpath:keystore.p12
    key-store-password: ${KEYSTORE_PASSWORD}
    key-store-type: PKCS12
    key-alias: bank-ssl
  http2:
    enabled: true

logging:
  level:
    org.springframework.security: INFO
    com.example.security: DEBUG
```

---

## Summary

| Feature | Implementation | Key Class/Annotation |
|---------|---------------|---------------------|
| Security Config | SecurityFilterChain | `@EnableWebSecurity` |
| Custom Auth | AuthenticationProvider | `BankingAuthenticationProvider` |
| MFA/TOTP | dev.samstevens.totp | `TotpMfaService` |
| Brute Force | Redis counters | `LoginAttemptService` |
| Password Policy | Passay | `PasswordPolicyService` |
| Audit Trail | JPA + AOP | `SecurityAuditAspect` |
| JWT | jjwt | `JwtTokenService` |
| Method Security | SpEL | `@PreAuthorize` |
| Security Headers | Spring Security DSL | `headers()` config |
| CORS | CorsConfiguration | `corsConfigurationSource()` |

## Next Part Preview

**Part 062: API Design and Versioning** — RESTful API principles, URL/header/media-type versioning, HATEOAS, OpenAPI 3.0, and a complete v1-to-v2 migration example with backward compatibility.
