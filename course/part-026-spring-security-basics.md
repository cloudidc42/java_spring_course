# Part 026: Spring Security Basics

Spring Security is the de-facto standard for securing Spring-based applications. It provides comprehensive security services for Java enterprise software and handles authentication (who are you?) and authorization (what can you do?).

---

## 1. Core Concepts

### Authentication vs Authorization

```
Authentication: Verifying WHO the user is
  - "Are you really John Smith?"
  - Credentials: username/password, token, certificate
  - Result: SecurityContext with authenticated user

Authorization: Deciding WHAT the user can do
  - "Is John Smith allowed to delete posts?"
  - Based on roles/permissions
  - Applied at URL level or method level
```

### Security Filter Chain Architecture

```
HTTP Request
    │
    ▼
┌─────────────────────────────────────────┐
│         SecurityFilterChain             │
│                                         │
│  UsernamePasswordAuthenticationFilter   │
│  BasicAuthenticationFilter              │
│  BearerTokenAuthenticationFilter        │
│  ExceptionTranslationFilter             │
│  FilterSecurityInterceptor              │
└─────────────────────────────────────────┘
    │
    ▼
Controller / Business Logic
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
    <!-- For testing security -->
    <dependency>
        <groupId>org.springframework.security</groupId>
        <artifactId>spring-security-test</artifactId>
        <scope>test</scope>
    </dependency>
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <optional>true</optional>
    </dependency>
</dependencies>
```

---

## 3. Basic Security Configuration

### Minimal SecurityFilterChain

```java
package com.example.security.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.web.SecurityFilterChain;

@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                // Public endpoints
                .requestMatchers("/api/public/**").permitAll()
                .requestMatchers("/api/posts/**").permitAll()
                // Authenticated endpoints
                .requestMatchers("/api/comments/**").authenticated()
                // Admin only
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                // Any other request requires authentication
                .anyRequest().authenticated()
            )
            .httpBasic(Customizer.withDefaults())  // HTTP Basic auth
            .csrf(csrf -> csrf.disable());          // Disable CSRF for REST APIs

        return http.build();
    }
}
```

### Complete Security Configuration

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity(prePostEnabled = true, securedEnabled = true)
@RequiredArgsConstructor
public class SecurityConfig {

    private final UserDetailsService userDetailsService;
    private final PasswordEncoder passwordEncoder;

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            // Authorization rules
            .authorizeHttpRequests(auth -> auth
                // Public - no authentication needed
                .requestMatchers(HttpMethod.GET, "/api/posts/**").permitAll()
                .requestMatchers(HttpMethod.GET, "/api/categories/**").permitAll()
                .requestMatchers(HttpMethod.GET, "/api/tags/**").permitAll()
                .requestMatchers("/api/auth/**").permitAll()
                .requestMatchers("/actuator/health").permitAll()
                .requestMatchers("/h2-console/**").permitAll()
                .requestMatchers("/swagger-ui/**", "/v3/api-docs/**").permitAll()

                // Role-based access
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .requestMatchers(HttpMethod.POST, "/api/posts/**").hasAnyRole("ADMIN", "AUTHOR")
                .requestMatchers(HttpMethod.PUT, "/api/posts/**").hasAnyRole("ADMIN", "AUTHOR")
                .requestMatchers(HttpMethod.DELETE, "/api/posts/**").hasAnyRole("ADMIN", "AUTHOR")

                // All others need authentication
                .anyRequest().authenticated()
            )

            // Form login
            .formLogin(form -> form
                .loginPage("/login")
                .loginProcessingUrl("/api/auth/login")
                .defaultSuccessUrl("/dashboard", true)
                .failureUrl("/login?error=true")
                .usernameParameter("username")
                .passwordParameter("password")
                .permitAll()
            )

            // Logout
            .logout(logout -> logout
                .logoutUrl("/api/auth/logout")
                .logoutSuccessUrl("/login?logout=true")
                .invalidateHttpSession(true)
                .deleteCookies("JSESSIONID")
                .clearAuthentication(true)
                .permitAll()
            )

            // Session management
            .sessionManagement(session -> session
                .sessionCreationPolicy(SessionCreationPolicy.IF_REQUIRED)
                .maximumSessions(1)               // One session per user
                .maxSessionsPreventsLogin(false)  // New login kicks out old session
            )

            // Exception handling
            .exceptionHandling(ex -> ex
                .authenticationEntryPoint(new HttpStatusEntryPoint(HttpStatus.UNAUTHORIZED))
                .accessDeniedHandler((request, response, accessDeniedException) -> {
                    response.setStatus(HttpStatus.FORBIDDEN.value());
                    response.setContentType(MediaType.APPLICATION_JSON_VALUE);
                    response.getWriter().write("{\"error\": \"Access denied\"}");
                })
            )

            // CSRF - disable for REST, enable for form-based
            .csrf(csrf -> csrf
                .ignoringRequestMatchers("/api/**")  // Disable for API
            )

            // CORS
            .cors(cors -> cors.configurationSource(corsConfigurationSource()))

            // H2 console frame support
            .headers(headers -> headers
                .frameOptions(frame -> frame.sameOrigin())
            );

        return http.build();
    }

    @Bean
    public CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration configuration = new CorsConfiguration();
        configuration.setAllowedOrigins(List.of("http://localhost:3000", "https://myapp.com"));
        configuration.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE", "OPTIONS"));
        configuration.setAllowedHeaders(List.of("Authorization", "Content-Type", "X-Requested-With"));
        configuration.setExposedHeaders(List.of("X-Total-Count", "X-Page-Number"));
        configuration.setAllowCredentials(true);
        configuration.setMaxAge(3600L);

        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/**", configuration);
        return source;
    }

    @Bean
    public AuthenticationManager authenticationManager(
            AuthenticationConfiguration authenticationConfiguration) throws Exception {
        return authenticationConfiguration.getAuthenticationManager();
    }

    @Bean
    public DaoAuthenticationProvider authenticationProvider() {
        DaoAuthenticationProvider provider = new DaoAuthenticationProvider();
        provider.setUserDetailsService(userDetailsService);
        provider.setPasswordEncoder(passwordEncoder);
        return provider;
    }
}
```

---

## 4. Password Encoding

```java
@Configuration
public class PasswordConfig {

    // BCrypt - recommended for password hashing
    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder(12);  // Strength: 10-12 is good
    }
}

// Usage example
@Service
@RequiredArgsConstructor
public class UserService {

    private final UserRepository userRepository;
    private final PasswordEncoder passwordEncoder;

    public User register(RegisterRequest request) {
        if (userRepository.existsByUsername(request.getUsername())) {
            throw new UserAlreadyExistsException("Username already taken");
        }
        if (userRepository.existsByEmail(request.getEmail())) {
            throw new UserAlreadyExistsException("Email already registered");
        }

        User user = User.builder()
            .username(request.getUsername())
            .email(request.getEmail())
            .password(passwordEncoder.encode(request.getPassword()))  // Hash password
            .roles(Set.of(Role.USER))
            .enabled(true)
            .build();

        return userRepository.save(user);
    }

    public boolean checkPassword(String rawPassword, String encodedPassword) {
        return passwordEncoder.matches(rawPassword, encodedPassword);
    }
}
```

---

## 5. UserDetailsService and UserDetails

### User Entity

```java
@Entity
@Table(name = "users")
@Getter @Setter @NoArgsConstructor @AllArgsConstructor @Builder
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, unique = true, length = 50)
    private String username;

    @Column(nullable = false, unique = true, length = 100)
    private String email;

    @Column(nullable = false)
    private String password;

    @ElementCollection(fetch = FetchType.EAGER)
    @CollectionTable(name = "user_roles", joinColumns = @JoinColumn(name = "user_id"))
    @Enumerated(EnumType.STRING)
    @Column(name = "role")
    @Builder.Default
    private Set<Role> roles = new HashSet<>();

    @Column(nullable = false)
    @Builder.Default
    private boolean enabled = true;

    @Column(nullable = false)
    @Builder.Default
    private boolean accountNonExpired = true;

    @Column(nullable = false)
    @Builder.Default
    private boolean accountNonLocked = true;

    @Column(nullable = false)
    @Builder.Default
    private boolean credentialsNonExpired = true;

    @Column(name = "created_at", nullable = false, updatable = false)
    private LocalDateTime createdAt;

    @PrePersist
    protected void onCreate() {
        createdAt = LocalDateTime.now();
    }
}

enum Role {
    USER, AUTHOR, MODERATOR, ADMIN
}
```

### UserDetails Implementation

```java
// Adapter: wraps User entity to implement UserDetails
public class UserPrincipal implements UserDetails {

    private final User user;

    public UserPrincipal(User user) {
        this.user = user;
    }

    @Override
    public Collection<? extends GrantedAuthority> getAuthorities() {
        return user.getRoles().stream()
            .map(role -> new SimpleGrantedAuthority("ROLE_" + role.name()))
            .collect(Collectors.toSet());
    }

    @Override
    public String getPassword() {
        return user.getPassword();
    }

    @Override
    public String getUsername() {
        return user.getUsername();
    }

    @Override
    public boolean isAccountNonExpired() {
        return user.isAccountNonExpired();
    }

    @Override
    public boolean isAccountNonLocked() {
        return user.isAccountNonLocked();
    }

    @Override
    public boolean isCredentialsNonExpired() {
        return user.isCredentialsNonExpired();
    }

    @Override
    public boolean isEnabled() {
        return user.isEnabled();
    }

    // Extra methods to access user data
    public Long getId() {
        return user.getId();
    }

    public String getEmail() {
        return user.getEmail();
    }

    public User getUser() {
        return user;
    }
}
```

### UserDetailsService Implementation

```java
@Service
@RequiredArgsConstructor
@Slf4j
public class CustomUserDetailsService implements UserDetailsService {

    private final UserRepository userRepository;

    @Override
    public UserDetails loadUserByUsername(String usernameOrEmail) throws UsernameNotFoundException {
        log.debug("Loading user by username or email: {}", usernameOrEmail);

        User user = userRepository.findByUsernameOrEmail(usernameOrEmail, usernameOrEmail)
            .orElseThrow(() -> new UsernameNotFoundException(
                "User not found with username or email: " + usernameOrEmail));

        return new UserPrincipal(user);
    }
}
```

### User Repository

```java
@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    Optional<User> findByUsername(String username);
    Optional<User> findByEmail(String email);
    Optional<User> findByUsernameOrEmail(String username, String email);
    boolean existsByUsername(String username);
    boolean existsByEmail(String email);
}
```

---

## 6. In-Memory Authentication (for testing/development)

```java
@Configuration
@EnableWebSecurity
public class InMemorySecurityConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/admin/**").hasRole("ADMIN")
                .requestMatchers("/user/**").hasRole("USER")
                .anyRequest().authenticated()
            )
            .formLogin(Customizer.withDefaults())
            .httpBasic(Customizer.withDefaults());

        return http.build();
    }

    @Bean
    public UserDetailsService userDetailsService() {
        UserDetails user = User.withDefaultPasswordEncoder()  // For dev only!
            .username("user")
            .password("password")
            .roles("USER")
            .build();

        UserDetails admin = User.withDefaultPasswordEncoder()
            .username("admin")
            .password("admin")
            .roles("ADMIN", "USER")
            .build();

        return new InMemoryUserDetailsManager(user, admin);
    }
}

// Better approach: use BCrypt for in-memory
@Bean
public UserDetailsService userDetailsService(PasswordEncoder encoder) {
    UserDetails user = User.builder()
        .username("user")
        .password(encoder.encode("password"))
        .roles("USER")
        .build();

    UserDetails admin = User.builder()
        .username("admin")
        .password(encoder.encode("admin123"))
        .roles("ADMIN", "USER")
        .build();

    return new InMemoryUserDetailsManager(user, admin);
}
```

---

## 7. Database-Backed Authentication

```java
// Auth Controller
@RestController
@RequestMapping("/api/auth")
@RequiredArgsConstructor
public class AuthController {

    private final AuthService authService;

    @PostMapping("/register")
    public ResponseEntity<UserResponse> register(@Valid @RequestBody RegisterRequest request) {
        User user = authService.register(request);
        return ResponseEntity.status(HttpStatus.CREATED)
            .body(UserResponse.from(user));
    }

    @PostMapping("/login")
    public ResponseEntity<LoginResponse> login(@Valid @RequestBody LoginRequest request) {
        LoginResponse response = authService.login(request);
        return ResponseEntity.ok(response);
    }

    @PostMapping("/logout")
    public ResponseEntity<Void> logout(HttpServletRequest request) {
        authService.logout(request);
        return ResponseEntity.ok().build();
    }

    @GetMapping("/me")
    @PreAuthorize("isAuthenticated()")
    public ResponseEntity<UserResponse> getCurrentUser(@AuthenticationPrincipal UserPrincipal principal) {
        return ResponseEntity.ok(UserResponse.from(principal.getUser()));
    }
}

// Auth Service
@Service
@RequiredArgsConstructor
@Transactional
public class AuthService {

    private final UserRepository userRepository;
    private final PasswordEncoder passwordEncoder;
    private final AuthenticationManager authenticationManager;

    public User register(RegisterRequest request) {
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
            .build();

        return userRepository.save(user);
    }

    public LoginResponse login(LoginRequest request) {
        // This throws if credentials are invalid
        Authentication authentication = authenticationManager.authenticate(
            new UsernamePasswordAuthenticationToken(
                request.getUsernameOrEmail(),
                request.getPassword()
            )
        );

        // Store in security context
        SecurityContextHolder.getContext().setAuthentication(authentication);

        UserPrincipal principal = (UserPrincipal) authentication.getPrincipal();

        return LoginResponse.builder()
            .userId(principal.getId())
            .username(principal.getUsername())
            .email(principal.getEmail())
            .roles(principal.getUser().getRoles())
            .message("Login successful")
            .build();
    }

    public void logout(HttpServletRequest request) {
        HttpSession session = request.getSession(false);
        if (session != null) {
            session.invalidate();
        }
        SecurityContextHolder.clearContext();
    }
}
```

---

## 8. Role-Based Access Control

### URL-Level Authorization

```java
@Bean
public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
    http
        .authorizeHttpRequests(auth -> auth
            // Public access
            .requestMatchers("/api/public/**").permitAll()

            // Authenticated users only
            .requestMatchers("/api/profile/**").authenticated()

            // Single role
            .requestMatchers("/api/admin/**").hasRole("ADMIN")

            // Multiple roles (OR condition)
            .requestMatchers(HttpMethod.POST, "/api/posts/**").hasAnyRole("ADMIN", "AUTHOR")

            // Specific authority (without ROLE_ prefix)
            .requestMatchers("/api/reports/**").hasAuthority("REPORT_ACCESS")

            // Multiple authorities
            .requestMatchers("/api/finances/**").hasAnyAuthority("FINANCE_READ", "FINANCE_WRITE")

            // SpEL expression
            .requestMatchers("/api/special/**").access(
                new WebExpressionAuthorizationManager(
                    "hasRole('ADMIN') and hasIpAddress('192.168.1.0/24')"
                )
            )

            .anyRequest().authenticated()
        );

    return http.build();
}
```

### Method-Level Security

```java
@Service
@RequiredArgsConstructor
public class PostService {

    private final PostRepository postRepository;

    // Requires ROLE_ADMIN or ROLE_AUTHOR
    @PreAuthorize("hasAnyRole('ADMIN', 'AUTHOR')")
    public Post createPost(CreatePostRequest request) {
        // ...
    }

    // Check if user is owner or admin
    @PreAuthorize("hasRole('ADMIN') or @postSecurityService.isOwner(#postId, authentication.name)")
    public Post updatePost(Long postId, UpdatePostRequest request) {
        // ...
    }

    // Verify result after method executes (filter the return value)
    @PostAuthorize("returnObject.author.username == authentication.name or hasRole('ADMIN')")
    public Post findById(Long id) {
        return postRepository.findById(id)
            .orElseThrow(() -> new PostNotFoundException("Post not found"));
    }

    // Filter collection input parameter
    @PreFilter("filterObject.author.username == authentication.name")
    public List<Post> bulkUpdate(List<Post> posts) {
        return postRepository.saveAll(posts);
    }

    // Filter collection return value
    @PostFilter("filterObject.status == 'PUBLISHED' or hasRole('ADMIN')")
    public List<Post> findAll() {
        return postRepository.findAll();
    }

    // @Secured - simple role check (no SpEL)
    @Secured({"ROLE_ADMIN", "ROLE_MODERATOR"})
    public void approveComment(Long commentId) {
        // ...
    }

    // Admin only - no exceptions
    @PreAuthorize("hasRole('ADMIN')")
    public void deletePost(Long postId) {
        postRepository.deleteById(postId);
    }
}

// Security service for custom checks
@Service("postSecurityService")
@RequiredArgsConstructor
public class PostSecurityService {

    private final PostRepository postRepository;

    public boolean isOwner(Long postId, String username) {
        return postRepository.findById(postId)
            .map(post -> post.getAuthor().getUsername().equals(username))
            .orElse(false);
    }

    public boolean canEditPost(Long postId, Authentication authentication) {
        UserPrincipal principal = (UserPrincipal) authentication.getPrincipal();

        // Admins can edit any post
        if (principal.getUser().getRoles().contains(Role.ADMIN)) {
            return true;
        }

        // Authors can only edit their own posts
        return postRepository.findById(postId)
            .map(post -> post.getAuthor().getId().equals(principal.getId()))
            .orElse(false);
    }
}
```

### Controller with Security Annotations

```java
@RestController
@RequestMapping("/api/posts")
@RequiredArgsConstructor
public class PostController {

    private final PostService postService;

    // Public - no auth needed
    @GetMapping
    public ResponseEntity<Page<PostSummaryDTO>> getAllPosts(
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "10") int size) {
        Pageable pageable = PageRequest.of(page, size);
        return ResponseEntity.ok(postService.findAllPublished(pageable)
            .map(PostSummaryDTO::from));
    }

    // Authenticated user - get their posts
    @GetMapping("/my")
    @PreAuthorize("isAuthenticated()")
    public ResponseEntity<List<PostSummaryDTO>> getMyPosts(
            @AuthenticationPrincipal UserPrincipal principal) {
        return ResponseEntity.ok(postService.findByAuthor(principal.getUsername())
            .stream().map(PostSummaryDTO::from).toList());
    }

    // Author or Admin - create post
    @PostMapping
    @PreAuthorize("hasAnyRole('AUTHOR', 'ADMIN')")
    public ResponseEntity<PostSummaryDTO> createPost(
            @Valid @RequestBody CreatePostRequest request,
            @AuthenticationPrincipal UserPrincipal principal) {
        Post post = postService.createPost(request, principal.getUsername());
        return ResponseEntity.status(HttpStatus.CREATED)
            .body(PostSummaryDTO.from(post));
    }

    // Owner or Admin - update post
    @PutMapping("/{id}")
    @PreAuthorize("hasRole('ADMIN') or @postSecurityService.isOwner(#id, authentication.name)")
    public ResponseEntity<PostSummaryDTO> updatePost(
            @PathVariable Long id,
            @Valid @RequestBody UpdatePostRequest request) {
        Post post = postService.updatePost(id, request);
        return ResponseEntity.ok(PostSummaryDTO.from(post));
    }

    // Admin only - delete
    @DeleteMapping("/{id}")
    @PreAuthorize("hasRole('ADMIN')")
    public ResponseEntity<Void> deletePost(@PathVariable Long id) {
        postService.deletePost(id);
        return ResponseEntity.noContent().build();
    }

    // Admin only - publish
    @PostMapping("/{id}/publish")
    @PreAuthorize("hasRole('ADMIN') or @postSecurityService.isOwner(#id, authentication.name)")
    public ResponseEntity<PostSummaryDTO> publishPost(@PathVariable Long id) {
        Post post = postService.publishPost(id);
        return ResponseEntity.ok(PostSummaryDTO.from(post));
    }
}
```

---

## 9. CSRF Protection

```java
@Configuration
@EnableWebSecurity
public class CsrfConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf
                // Option 1: Disable entirely (REST API with JWT)
                // .disable()

                // Option 2: Disable for specific paths
                .ignoringRequestMatchers("/api/public/**")

                // Option 3: Use custom token repository
                .csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())

                // Option 4: Custom CSRF handler
                .csrfTokenRequestHandler(new CsrfTokenRequestAttributeHandler())
            );

        return http.build();
    }
}

// Returning CSRF token in response header (for SPA clients)
@Component
public class CsrfTokenFilter extends OncePerRequestFilter {

    @Override
    protected void doFilterInternal(HttpServletRequest request, HttpServletResponse response,
                                    FilterChain filterChain) throws ServletException, IOException {
        CsrfToken csrfToken = (CsrfToken) request.getAttribute(CsrfToken.class.getName());
        if (csrfToken != null) {
            response.setHeader("X-CSRF-TOKEN", csrfToken.getToken());
        }
        filterChain.doFilter(request, response);
    }
}
```

---

## 10. CORS Configuration

```java
// Option 1: Global CORS via SecurityFilterChain
@Bean
public CorsConfigurationSource corsConfigurationSource() {
    CorsConfiguration config = new CorsConfiguration();
    config.setAllowedOriginPatterns(List.of("http://localhost:*", "https://*.myapp.com"));
    config.setAllowedMethods(Arrays.asList("GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS"));
    config.setAllowedHeaders(Arrays.asList(
        "Authorization", "Content-Type", "X-Requested-With", "Accept", "Origin"
    ));
    config.setExposedHeaders(Arrays.asList("X-Total-Count", "Authorization"));
    config.setAllowCredentials(true);
    config.setMaxAge(3600L);

    UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
    source.registerCorsConfiguration("/**", config);
    return source;
}

// Option 2: @CrossOrigin on controller
@RestController
@CrossOrigin(origins = "http://localhost:3000", maxAge = 3600)
@RequestMapping("/api/posts")
public class PostController {
    // ...
}

// Option 3: WebMvcConfigurer
@Configuration
public class WebConfig implements WebMvcConfigurer {

    @Override
    public void addCorsMappings(CorsRegistry registry) {
        registry.addMapping("/api/**")
            .allowedOrigins("http://localhost:3000")
            .allowedMethods("GET", "POST", "PUT", "DELETE")
            .allowedHeaders("*")
            .allowCredentials(true)
            .maxAge(3600);
    }
}
```

---

## 11. HTTP Basic Authentication

```java
@Configuration
@EnableWebSecurity
public class BasicAuthConfig {

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/public/**").permitAll()
                .anyRequest().authenticated()
            )
            .httpBasic(basic -> basic
                .realmName("Blog API")
                .authenticationEntryPoint((request, response, ex) -> {
                    response.setStatus(HttpStatus.UNAUTHORIZED.value());
                    response.setHeader("WWW-Authenticate", "Basic realm=\"Blog API\"");
                    response.setContentType(MediaType.APPLICATION_JSON_VALUE);
                    response.getWriter().write("""
                        {"error": "Unauthorized", "message": "Please provide credentials"}
                        """);
                })
            )
            .sessionManagement(session -> session
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS)  // No sessions for Basic auth
            )
            .csrf(csrf -> csrf.disable());

        return http.build();
    }
}
```

---

## 12. Complete Secure REST API Example

### DTOs

```java
@Getter @Setter @NoArgsConstructor @AllArgsConstructor @Builder
public class RegisterRequest {

    @NotBlank(message = "Username is required")
    @Size(min = 3, max = 50, message = "Username must be 3-50 characters")
    @Pattern(regexp = "^[a-zA-Z0-9_]+$", message = "Username can only contain letters, numbers, and underscores")
    private String username;

    @NotBlank(message = "Email is required")
    @Email(message = "Email must be valid")
    private String email;

    @NotBlank(message = "Password is required")
    @Size(min = 8, message = "Password must be at least 8 characters")
    private String password;
}

@Getter @Setter @NoArgsConstructor @AllArgsConstructor @Builder
public class LoginRequest {
    @NotBlank
    private String usernameOrEmail;

    @NotBlank
    private String password;
}

@Getter @Setter @NoArgsConstructor @AllArgsConstructor @Builder
public class LoginResponse {
    private Long userId;
    private String username;
    private String email;
    private Set<Role> roles;
    private String message;
}

@Getter @Setter @NoArgsConstructor @AllArgsConstructor @Builder
public class UserResponse {
    private Long id;
    private String username;
    private String email;
    private Set<Role> roles;
    private LocalDateTime createdAt;

    public static UserResponse from(User user) {
        return UserResponse.builder()
            .id(user.getId())
            .username(user.getUsername())
            .email(user.getEmail())
            .roles(user.getRoles())
            .createdAt(user.getCreatedAt())
            .build();
    }
}
```

### Security Exception Handling

```java
@RestControllerAdvice
@Slf4j
public class SecurityExceptionHandler {

    @ExceptionHandler(AccessDeniedException.class)
    public ResponseEntity<ErrorResponse> handleAccessDenied(AccessDeniedException ex) {
        log.warn("Access denied: {}", ex.getMessage());
        return ResponseEntity.status(HttpStatus.FORBIDDEN)
            .body(ErrorResponse.builder()
                .status(403)
                .error("Forbidden")
                .message("You don't have permission to perform this action")
                .timestamp(LocalDateTime.now())
                .build());
    }

    @ExceptionHandler(AuthenticationException.class)
    public ResponseEntity<ErrorResponse> handleAuthenticationException(AuthenticationException ex) {
        log.warn("Authentication failed: {}", ex.getMessage());
        return ResponseEntity.status(HttpStatus.UNAUTHORIZED)
            .body(ErrorResponse.builder()
                .status(401)
                .error("Unauthorized")
                .message("Invalid username or password")
                .timestamp(LocalDateTime.now())
                .build());
    }

    @ExceptionHandler(UserAlreadyExistsException.class)
    public ResponseEntity<ErrorResponse> handleUserAlreadyExists(UserAlreadyExistsException ex) {
        return ResponseEntity.status(HttpStatus.CONFLICT)
            .body(ErrorResponse.builder()
                .status(409)
                .error("Conflict")
                .message(ex.getMessage())
                .timestamp(LocalDateTime.now())
                .build());
    }
}

@Getter @Setter @Builder
public class ErrorResponse {
    private int status;
    private String error;
    private String message;
    private LocalDateTime timestamp;
    private String path;
}
```

### Admin Controller

```java
@RestController
@RequestMapping("/api/admin")
@PreAuthorize("hasRole('ADMIN')")  // All methods in this controller require ADMIN
@RequiredArgsConstructor
public class AdminController {

    private final UserService userService;

    @GetMapping("/users")
    public ResponseEntity<List<UserResponse>> getAllUsers() {
        return ResponseEntity.ok(userService.findAll().stream()
            .map(UserResponse::from).toList());
    }

    @PutMapping("/users/{id}/roles")
    public ResponseEntity<UserResponse> updateUserRoles(
            @PathVariable Long id,
            @RequestBody Set<Role> roles) {
        User user = userService.updateRoles(id, roles);
        return ResponseEntity.ok(UserResponse.from(user));
    }

    @PutMapping("/users/{id}/enable")
    public ResponseEntity<UserResponse> enableUser(@PathVariable Long id) {
        User user = userService.setEnabled(id, true);
        return ResponseEntity.ok(UserResponse.from(user));
    }

    @PutMapping("/users/{id}/disable")
    public ResponseEntity<UserResponse> disableUser(@PathVariable Long id) {
        User user = userService.setEnabled(id, false);
        return ResponseEntity.ok(UserResponse.from(user));
    }

    @DeleteMapping("/users/{id}")
    public ResponseEntity<Void> deleteUser(@PathVariable Long id) {
        userService.deleteUser(id);
        return ResponseEntity.noContent().build();
    }

    @GetMapping("/stats")
    public ResponseEntity<Map<String, Object>> getStats() {
        Map<String, Object> stats = new HashMap<>();
        stats.put("totalUsers", userService.count());
        stats.put("activeUsers", userService.countEnabled());
        return ResponseEntity.ok(stats);
    }
}
```

### Security Context Helper

```java
// Utility to get current user in services
@Component
public class SecurityUtils {

    public static Optional<UserPrincipal> getCurrentUser() {
        return Optional.ofNullable(SecurityContextHolder.getContext().getAuthentication())
            .filter(Authentication::isAuthenticated)
            .filter(auth -> !"anonymousUser".equals(auth.getPrincipal()))
            .map(auth -> (UserPrincipal) auth.getPrincipal());
    }

    public static String getCurrentUsername() {
        return getCurrentUser()
            .map(UserDetails::getUsername)
            .orElseThrow(() -> new IllegalStateException("No authenticated user found"));
    }

    public static Long getCurrentUserId() {
        return getCurrentUser()
            .map(UserPrincipal::getId)
            .orElseThrow(() -> new IllegalStateException("No authenticated user found"));
    }

    public static boolean hasRole(String role) {
        return getCurrentUser()
            .map(user -> user.getAuthorities().stream()
                .anyMatch(auth -> auth.getAuthority().equals("ROLE_" + role)))
            .orElse(false);
    }

    public static boolean isAdmin() {
        return hasRole("ADMIN");
    }
}
```

---

## 13. Summary Table

| Concept | Class/Annotation | Purpose |
|---|---|---|
| Enable security | `@EnableWebSecurity` | Activate Spring Security |
| Enable method security | `@EnableMethodSecurity` | Allow `@PreAuthorize` etc. |
| Configure rules | `SecurityFilterChain` | HTTP security configuration |
| Password hashing | `BCryptPasswordEncoder` | Secure password storage |
| Load user | `UserDetailsService` | Fetch user from data source |
| User representation | `UserDetails` | User with authorities |
| Pre-method check | `@PreAuthorize` | Check before method runs |
| Post-method check | `@PostAuthorize` | Check return value |
| Filter inputs | `@PreFilter` | Filter collection parameter |
| Filter outputs | `@PostFilter` | Filter collection result |
| Simple role check | `@Secured` | Basic role annotation |
| Current user | `@AuthenticationPrincipal` | Inject authenticated user |
| CSRF protection | `CsrfTokenRepository` | Cross-site request forgery |
| CORS | `CorsConfigurationSource` | Cross-origin requests |

---

## Next Part

**Part 027** covers JWT (JSON Web Token) authentication: generating and validating JWT tokens, implementing a `JwtAuthenticationFilter`, stateless authentication flow, token refresh mechanism, and building a complete auth system with register, login, and logout endpoints.
