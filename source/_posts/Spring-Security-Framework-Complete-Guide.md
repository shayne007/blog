---
title: Spring Security Framework - Complete Guide
date: 2025-06-25 20:24:29
tags: [spring, spring security]
categories: [spring]
---

## Core Underlying Principles

Spring Security is built on several fundamental principles that form the backbone of its architecture and functionality. Understanding these principles is crucial for implementing robust security solutions.

### Authentication vs Authorization

**Authentication** answers "Who are you?" while **Authorization** answers "What can you do?" Spring Security treats these as separate concerns, allowing for flexible security configurations.

```java
// Authentication - verifying identity
@Override
protected void configure(AuthenticationManagerBuilder auth) throws Exception {
    auth.inMemoryAuthentication()
        .withUser("user")
        .password(passwordEncoder().encode("password"))
        .roles("USER");
}

// Authorization - defining access rules
@Override
protected void configure(HttpSecurity http) throws Exception {
    http.authorizeRequests()
        .antMatchers("/admin/**").hasRole("ADMIN")
        .antMatchers("/user/**").hasRole("USER")
        .anyRequest().authenticated();
}
```

### Security Filter Chain

Spring Security operates through a chain of filters that intercept HTTP requests. Each filter has a specific responsibility and can either process the request or pass it to the next filter.

{% mermaid flowchart TD %}
    A[HTTP Request] --> B[Security Filter Chain]
    B --> C[SecurityContextPersistenceFilter]
    C --> D[UsernamePasswordAuthenticationFilter]
    D --> E[ExceptionTranslationFilter]
    E --> F[FilterSecurityInterceptor]
    F --> G[Application Controller]
    
    style B fill:#e1f5fe
    style G fill:#e8f5e8
{% endmermaid %}

### SecurityContext and SecurityContextHolder

The SecurityContext stores security information for the current thread of execution. The SecurityContextHolder provides access to this context.

```java
// Getting current authenticated user
Authentication authentication = SecurityContextHolder.getContext().getAuthentication();
String username = authentication.getName();
Collection<? extends GrantedAuthority> authorities = authentication.getAuthorities();

// Setting security context programmatically
UsernamePasswordAuthenticationToken token = 
    new UsernamePasswordAuthenticationToken(user, null, authorities);
SecurityContextHolder.getContext().setAuthentication(token);
```

**Interview Insight**: *"How does Spring Security maintain security context across requests?"*
> Spring Security uses ThreadLocal to store security context, ensuring thread safety. The SecurityContextPersistenceFilter loads the context from HttpSession at the beginning of each request and clears it at the end.

### Principle of Least Privilege

Spring Security encourages granting minimal necessary permissions. This is implemented through role-based and method-level security.

```java
@PreAuthorize("hasRole('ADMIN') or (hasRole('USER') and #username == authentication.name)")
public User getUserDetails(@PathVariable String username) {
    return userService.findByUsername(username);
}
```

## When to Use Spring Security Framework

### Enterprise Applications

Spring Security is ideal for enterprise applications requiring:
- Complex authentication mechanisms (LDAP, OAuth2, SAML)
- Fine-grained authorization
- Audit trails and compliance requirements
- Integration with existing identity providers

### Web Applications with User Management

Perfect for applications featuring:
- User registration and login
- Role-based access control
- Session management
- CSRF protection

```java
@Configuration
@EnableWebSecurity
public class WebSecurityConfig extends WebSecurityConfigurerAdapter {
    
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
            .authorizeRequests()
                .antMatchers("/register", "/login").permitAll()
                .antMatchers("/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            .and()
            .formLogin()
                .loginPage("/login")
                .defaultSuccessUrl("/dashboard")
            .and()
            .logout()
                .logoutSuccessUrl("/login?logout")
            .and()
            .csrf().csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse());
    }
}
```

### REST APIs and Microservices

Essential for securing REST APIs with:
- JWT token-based authentication
- Stateless security
- API rate limiting
- Cross-origin resource sharing (CORS)

```java
@Configuration
@EnableWebSecurity
public class JwtSecurityConfig {
    
    @Bean
    public JwtAuthenticationEntryPoint jwtAuthenticationEntryPoint() {
        return new JwtAuthenticationEntryPoint();
    }
    
    @Bean
    public JwtRequestFilter jwtRequestFilter() {
        return new JwtRequestFilter();
    }
    
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http.csrf().disable()
            .authorizeRequests()
                .antMatchers("/api/auth/**").permitAll()
                .anyRequest().authenticated()
            .and()
            .exceptionHandling().authenticationEntryPoint(jwtAuthenticationEntryPoint)
            .and()
            .sessionManagement().sessionCreationPolicy(SessionCreationPolicy.STATELESS);
            
        http.addFilterBefore(jwtRequestFilter, UsernamePasswordAuthenticationFilter.class);
    }
}
```

### When NOT to Use Spring Security

- Simple applications with basic authentication needs
- Applications with custom security requirements that conflict with Spring Security's architecture
- Performance-critical applications where the filter chain overhead is unacceptable
- Applications requiring non-standard authentication flows

## User Login, Logout, and Session Management

### Login Process Flow

{% mermaid sequenceDiagram %}
    participant U as User
    participant B as Browser
    participant S as Spring Security
    participant A as AuthenticationManager
    participant P as AuthenticationProvider
    participant D as UserDetailsService
    
    U->>B: Enter credentials
    B->>S: POST /login
    S->>A: Authenticate request
    A->>P: Delegate authentication
    P->>D: Load user details
    D-->>P: Return UserDetails
    P-->>A: Authentication result
    A-->>S: Authenticated user
    S->>B: Redirect to success URL
    B->>U: Display protected resource
{% endmermaid %}

### Custom Login Implementation

```java
@Configuration
@EnableWebSecurity
public class LoginConfig extends WebSecurityConfigurerAdapter {
    
    @Autowired
    private CustomUserDetailsService userDetailsService;
    
    @Autowired
    private CustomAuthenticationSuccessHandler successHandler;
    
    @Autowired
    private CustomAuthenticationFailureHandler failureHandler;
    
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
            .formLogin()
                .loginPage("/custom-login")
                .loginProcessingUrl("/perform-login")
                .usernameParameter("email")
                .passwordParameter("pwd")
                .successHandler(successHandler)
                .failureHandler(failureHandler)
            .and()
            .logout()
                .logoutUrl("/perform-logout")
                .logoutSuccessHandler(customLogoutSuccessHandler())
                .deleteCookies("JSESSIONID")
                .invalidateHttpSession(true);
    }
    
    @Bean
    public CustomLogoutSuccessHandler customLogoutSuccessHandler() {
        return new CustomLogoutSuccessHandler();
    }
}
```

### Custom Authentication Success Handler

```java
@Component
public class CustomAuthenticationSuccessHandler implements AuthenticationSuccessHandler {
    
    private final Logger logger = LoggerFactory.getLogger(CustomAuthenticationSuccessHandler.class);
    
    @Override
    public void onAuthenticationSuccess(HttpServletRequest request, 
                                      HttpServletResponse response,
                                      Authentication authentication) throws IOException {
        
        // Log successful login
        logger.info("User {} logged in successfully", authentication.getName());
        
        // Update last login timestamp
        updateLastLoginTime(authentication.getName());
        
        // Redirect based on role
        String redirectUrl = determineTargetUrl(authentication);
        response.sendRedirect(redirectUrl);
    }
    
    private String determineTargetUrl(Authentication authentication) {
        boolean isAdmin = authentication.getAuthorities().stream()
            .anyMatch(authority -> authority.getAuthority().equals("ROLE_ADMIN"));
        
        return isAdmin ? "/admin/dashboard" : "/user/dashboard";
    }
    
    private void updateLastLoginTime(String username) {
        // Implementation to update user's last login time
    }
}
```

### Session Management

Spring Security provides comprehensive session management capabilities:

```java
@Override
protected void configure(HttpSecurity http) throws Exception {
    http
        .sessionManagement()
            .sessionCreationPolicy(SessionCreationPolicy.IF_REQUIRED)
            .maximumSessions(1)
            .maxSessionsPreventsLogin(false)
            .sessionRegistry(sessionRegistry())
            .and()
        .sessionFixation().migrateSession()
        .invalidSessionUrl("/login?expired");
}

@Bean
public HttpSessionEventPublisher httpSessionEventPublisher() {
    return new HttpSessionEventPublisher();
}

@Bean
public SessionRegistry sessionRegistry() {
    return new SessionRegistryImpl();
}
```

**Interview Insight**: *"How does Spring Security handle concurrent sessions?"*
> Spring Security can limit concurrent sessions per user through SessionRegistry. When maximum sessions are exceeded, it can either prevent new logins or invalidate existing sessions based on configuration.

### Session Timeout Configuration

```java
// In application.properties
server.servlet.session.timeout=30m

// Programmatic configuration
@Override
protected void configure(HttpSecurity http) throws Exception {
    http
        .sessionManagement()
            .sessionCreationPolicy(SessionCreationPolicy.IF_REQUIRED)
            .and()
        .rememberMe()
            .key("uniqueAndSecret")
            .tokenValiditySeconds(86400) // 24 hours
            .userDetailsService(userDetailsService);
}
```

### Remember Me Functionality

```java
@Configuration
public class RememberMeConfig {
    
    @Bean
    public PersistentTokenRepository persistentTokenRepository() {
        JdbcTokenRepositoryImpl tokenRepository = new JdbcTokenRepositoryImpl();
        tokenRepository.setDataSource(dataSource);
        return tokenRepository;
    }
    
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
            .rememberMe()
                .rememberMeParameter("remember-me")
                .tokenRepository(persistentTokenRepository())
                .tokenValiditySeconds(86400)
                .userDetailsService(userDetailsService);
    }
}
```
## Logout Process

Proper logout implementation is essential for security, ensuring complete cleanup of user sessions and security contexts.

### Comprehensive Logout Configuration

```java
@Configuration
public class LogoutConfig {
    
    @Bean
    public SecurityFilterChain logoutFilterChain(HttpSecurity http) throws Exception {
        return http
            .logout(logout -> logout
                .logoutUrl("/logout")
                .logoutRequestMatcher(new AntPathRequestMatcher("/logout", "POST"))
                .logoutSuccessUrl("/login?logout=true")
                .logoutSuccessHandler(customLogoutSuccessHandler())
                .invalidateHttpSession(true)
                .clearAuthentication(true)
                .deleteCookies("JSESSIONID", "remember-me")
                .addLogoutHandler(customLogoutHandler())
            )
            .build();
    }
    
    @Bean
    public LogoutSuccessHandler customLogoutSuccessHandler() {
        return new CustomLogoutSuccessHandler();
    }
    
    @Bean
    public LogoutHandler customLogoutHandler() {
        return new CustomLogoutHandler();
    }
}
```

### Custom Logout Handlers

```java
@Component
public class CustomLogoutHandler implements LogoutHandler {
    
    @Autowired
    private SessionRegistry sessionRegistry;
    
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    @Override
    public void logout(HttpServletRequest request, HttpServletResponse response, 
            Authentication authentication) {
        
        if (authentication != null) {
            String username = authentication.getName();
            
            // Clear user-specific cache
            redisTemplate.delete("user:cache:" + username);
            redisTemplate.delete("user:permissions:" + username);
            
            // Log logout event
            logger.info("User {} logged out from IP: {}", username, getClientIP(request));
            
            // Invalidate all sessions for this user (optional)
            sessionRegistry.getAllPrincipals().stream()
                .filter(principal -> principal instanceof UserDetails)
                .filter(principal -> ((UserDetails) principal).getUsername().equals(username))
                .forEach(principal -> 
                    sessionRegistry.getAllSessions(principal, false)
                        .forEach(SessionInformation::expireNow)
                );
        }
        
        // Clear security context
        SecurityContextHolder.clearContext();
    }
}

@Component
public class CustomLogoutSuccessHandler implements LogoutSuccessHandler {
    
    @Override
    public void onLogoutSuccess(HttpServletRequest request, HttpServletResponse response, 
            Authentication authentication) throws IOException, ServletException {
        
        // Add logout timestamp to response headers
        response.addHeader("Logout-Time", Instant.now().toString());
        
        // Redirect based on user agent or request parameter
        String redirectUrl = "/login?logout=true";
        String userAgent = request.getHeader("User-Agent");
        
        if (userAgent != null && userAgent.contains("Mobile")) {
            redirectUrl = "/mobile/login?logout=true";
        }
        
        response.sendRedirect(redirectUrl);
    }
}
```

### Logout Flow Diagram

{% mermaid sequenceDiagram %}
    participant User
    participant Browser
    participant LogoutFilter
    participant LogoutHandler
    participant SessionRegistry
    participant RedisCache
    participant Database
    
    User->>Browser: Click logout
    Browser->>LogoutFilter: POST /logout
    LogoutFilter->>LogoutHandler: Handle logout
    LogoutHandler->>SessionRegistry: Invalidate sessions
    LogoutHandler->>RedisCache: Clear user cache
    LogoutHandler->>Database: Log logout event
    LogoutHandler-->>LogoutFilter: Cleanup complete
    LogoutFilter->>LogoutFilter: Clear SecurityContext
    LogoutFilter-->>Browser: Redirect to login
    Browser-->>User: Login page with logout message
{% endmermaid %}

## Advanced Authentication Mechanisms

### JWT Token-Based Authentication

```java
@Component
public class JwtTokenUtil {
    
    private static final String SECRET = "mySecretKey";
    private static final int JWT_TOKEN_VALIDITY = 5 * 60 * 60; // 5 hours
    
    public String generateToken(UserDetails userDetails) {
        Map<String, Object> claims = new HashMap<>();
        return createToken(claims, userDetails.getUsername());
    }
    
    private String createToken(Map<String, Object> claims, String subject) {
        return Jwts.builder()
            .setClaims(claims)
            .setSubject(subject)
            .setIssuedAt(new Date(System.currentTimeMillis()))
            .setExpiration(new Date(System.currentTimeMillis() + JWT_TOKEN_VALIDITY * 1000))
            .signWith(SignatureAlgorithm.HS512, SECRET)
            .compact();
    }
    
    public Boolean validateToken(String token, UserDetails userDetails) {
        final String username = getUsernameFromToken(token);
        return (username.equals(userDetails.getUsername()) && !isTokenExpired(token));
    }
}
```

### OAuth2 Integration

```java
@Configuration
@EnableOAuth2Client
public class OAuth2Config {
    
    @Bean
    public OAuth2RestTemplate oauth2RestTemplate(OAuth2ClientContext oauth2ClientContext) {
        return new OAuth2RestTemplate(googleOAuth2ResourceDetails(), oauth2ClientContext);
    }
    
    @Bean
    public OAuth2ProtectedResourceDetails googleOAuth2ResourceDetails() {
        AuthorizationCodeResourceDetails details = new AuthorizationCodeResourceDetails();
        details.setClientId("your-client-id");
        details.setClientSecret("your-client-secret");
        details.setAccessTokenUri("https://oauth2.googleapis.com/token");
        details.setUserAuthorizationUri("https://accounts.google.com/o/oauth2/auth");
        details.setScope(Arrays.asList("email", "profile"));
        return details;
    }
}
```

## Method-Level Security

### Enabling Method Security

```java
@Configuration
@EnableGlobalMethodSecurity(
    prePostEnabled = true,
    securedEnabled = true,
    jsr250Enabled = true
)
public class MethodSecurityConfig extends GlobalMethodSecurityConfiguration {
    
    @Override
    protected MethodSecurityExpressionHandler createExpressionHandler() {
        DefaultMethodSecurityExpressionHandler expressionHandler = 
            new DefaultMethodSecurityExpressionHandler();
        expressionHandler.setPermissionEvaluator(new CustomPermissionEvaluator());
        return expressionHandler;
    }
}
```

### Security Annotations in Action

```java
@Service
public class DocumentService {
    
    @PreAuthorize("hasRole('ADMIN')")
    public void deleteDocument(Long documentId) {
        // Only admins can delete documents
    }
    
    @PreAuthorize("hasRole('USER') and #document.owner == authentication.name")
    public void editDocument(@P("document") Document document) {
        // Users can only edit their own documents
    }
    
    @PostAuthorize("returnObject.owner == authentication.name or hasRole('ADMIN')")
    public Document getDocument(Long documentId) {
        return documentRepository.findById(documentId);
    }
    
    @PreFilter("filterObject.owner == authentication.name")
    public void processDocuments(List<Document> documents) {
        // Process only documents owned by the current user
    }
}
```

**Interview Insight**: *"What's the difference between @PreAuthorize and @Secured?"*
> @PreAuthorize supports SpEL expressions for complex authorization logic, while @Secured only supports role-based authorization. @PreAuthorize is more flexible and powerful.

## Security Best Practices

### Password Security

```java
@Configuration
public class PasswordConfig {
    
    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder(12);
    }
    
    @Bean
    public PasswordValidator passwordValidator() {
        return new PasswordValidator(Arrays.asList(
            new LengthRule(8, 30),
            new CharacterRule(EnglishCharacterData.UpperCase, 1),
            new CharacterRule(EnglishCharacterData.LowerCase, 1),
            new CharacterRule(EnglishCharacterData.Digit, 1),
            new CharacterRule(EnglishCharacterData.Special, 1),
            new WhitespaceRule()
        ));
    }
}
```

### CSRF Protection

```java
@Override
protected void configure(HttpSecurity http) throws Exception {
    http
        .csrf()
            .csrfTokenRepository(CookieCsrfTokenRepository.withHttpOnlyFalse())
            .ignoringAntMatchers("/api/public/**")
        .and()
        .headers()
            .frameOptions().deny()
            .contentTypeOptions().and()
            .httpStrictTransportSecurity(hstsConfig -> hstsConfig
                .maxAgeInSeconds(31536000)
                .includeSubdomains(true));
}
```

### Input Validation and Sanitization

```java
@RestController
@Validated
public class UserController {
    
    @PostMapping("/users")
    public ResponseEntity<User> createUser(@Valid @RequestBody CreateUserRequest request) {
        // Validation handled by @Valid annotation
        User user = userService.createUser(request);
        return ResponseEntity.ok(user);
    }
}

@Data
public class CreateUserRequest {
    
    @NotBlank(message = "Username is required")
    @Size(min = 3, max = 20, message = "Username must be between 3 and 20 characters")
    @Pattern(regexp = "^[a-zA-Z0-9._-]+$", message = "Username contains invalid characters")
    private String username;
    
    @NotBlank(message = "Email is required")
    @Email(message = "Invalid email format")
    private String email;
    
    @NotBlank(message = "Password is required")
    @Size(min = 8, message = "Password must be at least 8 characters")
    private String password;
}
```

## Common Security Vulnerabilities and Mitigation

### SQL Injection Prevention

```java
@Repository
public class UserRepository {
    
    @Autowired
    private JdbcTemplate jdbcTemplate;
    
    // Vulnerable code (DON'T DO THIS)
    public User findByUsernameUnsafe(String username) {
        String sql = "SELECT * FROM users WHERE username = '" + username + "'";
        return jdbcTemplate.queryForObject(sql, User.class);
    }
    
    // Secure code (DO THIS)
    public User findByUsernameSafe(String username) {
        String sql = "SELECT * FROM users WHERE username = ?";
        return jdbcTemplate.queryForObject(sql, new Object[]{username}, User.class);
    }
}
```

### XSS Prevention

```java
@Configuration
public class SecurityHeadersConfig {
    
    @Bean
    public FilterRegistrationBean<XSSFilter> xssPreventFilter() {
        FilterRegistrationBean<XSSFilter> registrationBean = new FilterRegistrationBean<>();
        registrationBean.setFilter(new XSSFilter());
        registrationBean.addUrlPatterns("/*");
        return registrationBean;
    }
}

public class XSSFilter implements Filter {
    
    @Override
    public void doFilter(ServletRequest request, ServletResponse response, 
                        FilterChain chain) throws IOException, ServletException {
        
        XSSRequestWrapper wrappedRequest = new XSSRequestWrapper((HttpServletRequest) request);
        chain.doFilter(wrappedRequest, response);
    }
}
```

## Testing Spring Security

### Security Testing with MockMvc

```java
@RunWith(SpringRunner.class)
@WebMvcTest(UserController.class)
public class UserControllerSecurityTest {
    
    @Autowired
    private MockMvc mockMvc;
    
    @Test
    @WithMockUser(roles = "ADMIN")
    public void testAdminAccessToUserData() throws Exception {
        mockMvc.perform(get("/admin/users"))
            .andExpect(status().isOk());
    }
    
    @Test
    @WithMockUser(roles = "USER")
    public void testUserAccessToAdminEndpoint() throws Exception {
        mockMvc.perform(get("/admin/users"))
            .andExpect(status().isForbidden());
    }
    
    @Test
    public void testUnauthenticatedAccess() throws Exception {
        mockMvc.perform(get("/user/profile"))
            .andExpect(status().isUnauthorized());
    }
}
```

### Integration Testing with TestContainers

```java
@SpringBootTest
@Testcontainers
public class SecurityIntegrationTest {
    
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:13")
        .withDatabaseName("testdb")
        .withUsername("test")
        .withPassword("test");
    
    @Autowired
    private TestRestTemplate restTemplate;
    
    @Test
    public void testFullAuthenticationFlow() {
        // Test user registration
        ResponseEntity<String> registerResponse = restTemplate.postForEntity(
            "/api/auth/register", 
            new RegisterRequest("test@example.com", "password123"),
            String.class
        );
        
        assertThat(registerResponse.getStatusCode()).isEqualTo(HttpStatus.CREATED);
        
        // Test user login
        ResponseEntity<LoginResponse> loginResponse = restTemplate.postForEntity(
            "/api/auth/login",
            new LoginRequest("test@example.com", "password123"),
            LoginResponse.class
        );
        
        assertThat(loginResponse.getStatusCode()).isEqualTo(HttpStatus.OK);
        assertThat(loginResponse.getBody().getToken()).isNotNull();
    }
}
```

## Performance Optimization

### Security Filter Chain Optimization

```java
@Configuration
public class OptimizedSecurityConfig {
    
    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
            // Disable unnecessary features for API-only applications
            .csrf().disable()
            .sessionManagement().sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            .and()
            // Order matters - put most specific patterns first
            .authorizeRequests()
                .antMatchers("/api/public/**").permitAll()
                .antMatchers(HttpMethod.GET, "/api/products/**").permitAll()
                .antMatchers("/api/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated();
    }
}
```

### Caching Security Context

```java
@Configuration
public class SecurityCacheConfig {
    
    @Bean
    public CacheManager cacheManager() {
        return new ConcurrentMapCacheManager("userCache", "permissionCache");
    }
    
    @Service
    public class CachedUserDetailsService implements UserDetailsService {
        
        @Cacheable(value = "userCache", key = "#username")
        @Override
        public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
            return userRepository.findByUsername(username)
                .map(this::createUserPrincipal)
                .orElseThrow(() -> new UsernameNotFoundException("User not found: " + username));
        }
    }
}
```

## Troubleshooting Common Issues

### Debug Security Configuration

```java
@Configuration
@EnableWebSecurity
@EnableGlobalMethodSecurity(prePostEnabled = true)
public class DebugSecurityConfig extends WebSecurityConfigurerAdapter {
    
    @Override
    public void configure(WebSecurity web) throws Exception {
        web.debug(true); // Enable security debugging
    }
    
    @Bean
    public Logger securityLogger() {
        Logger logger = LoggerFactory.getLogger("org.springframework.security");
        ((ch.qos.logback.classic.Logger) logger).setLevel(Level.DEBUG);
        return logger;
    }
}
```

### Common Configuration Mistakes

```java
// WRONG: Ordering matters in security configuration
http.authorizeRequests()
    .anyRequest().authenticated()  // This catches everything
    .antMatchers("/public/**").permitAll(); // This never gets reached

// CORRECT: Specific patterns first
http.authorizeRequests()
    .antMatchers("/public/**").permitAll()
    .anyRequest().authenticated();
```

**Interview Insight**: *"What happens when Spring Security configuration conflicts occur?"*
> Spring Security evaluates rules in order. The first matching rule wins, so specific patterns must come before general ones. Always place more restrictive rules before less restrictive ones.

## Monitoring and Auditing

### Security Events Logging

```java
@Component
public class SecurityEventListener {
    
    private final Logger logger = LoggerFactory.getLogger(SecurityEventListener.class);
    
    @EventListener
    public void handleAuthenticationSuccess(AuthenticationSuccessEvent event) {
        logger.info("User '{}' logged in successfully from IP: {}", 
            event.getAuthentication().getName(),
            getClientIpAddress());
    }
    
    @EventListener
    public void handleAuthenticationFailure(AbstractAuthenticationFailureEvent event) {
        logger.warn("Authentication failed for user '{}': {}", 
            event.getAuthentication().getName(),
            event.getException().getMessage());
    }
    
    @EventListener
    public void handleAuthorizationFailure(AuthorizationFailureEvent event) {
        logger.warn("Authorization failed for user '{}' accessing resource: {}", 
            event.getAuthentication().getName(),
            event.getRequestUrl());
    }
}
```

### Metrics and Monitoring

```java
@Component
public class SecurityMetrics {
    
    private final MeterRegistry meterRegistry;
    private final Counter loginAttempts;
    private final Counter loginFailures;
    
    public SecurityMetrics(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
        this.loginAttempts = Counter.builder("security.login.attempts")
            .description("Total login attempts")
            .register(meterRegistry);
        this.loginFailures = Counter.builder("security.login.failures")
            .description("Failed login attempts")
            .register(meterRegistry);
    }
    
    @EventListener
    public void onLoginAttempt(AuthenticationSuccessEvent event) {
        loginAttempts.increment();
    }
    
    @EventListener
    public void onLoginFailure(AbstractAuthenticationFailureEvent event) {
        loginAttempts.increment();
        loginFailures.increment();
    }
}
```

## Interview Questions and Answers

### Technical Deep Dive Questions

**Q: Explain the difference between authentication and authorization in Spring Security.**
A: Authentication verifies identity ("who are you?") while authorization determines permissions ("what can you do?"). Spring Security separates these concerns - AuthenticationManager handles authentication, while AccessDecisionManager handles authorization decisions.

**Q: How does Spring Security handle stateless authentication?**
A: For stateless authentication, Spring Security doesn't maintain session state. Instead, it uses tokens (like JWT) passed with each request. Configure with `SessionCreationPolicy.STATELESS` and implement token-based filters.

**Q: What is the purpose of SecurityContextHolder?**
A: SecurityContextHolder provides access to the SecurityContext, which stores authentication information for the current thread. It uses ThreadLocal to ensure thread safety and provides three strategies: ThreadLocal (default), InheritableThreadLocal, and Global.

**Q: How do you implement custom authentication in Spring Security?**
A: Implement custom authentication by:
1. Creating a custom AuthenticationProvider
2. Implementing authenticate() method
3. Registering the provider with AuthenticationManager
4. Optionally creating custom Authentication tokens

### Practical Implementation Questions

**Q: How would you secure a REST API with JWT tokens?**
A: Implement JWT security by:
1. Creating JWT utility class for token generation/validation
2. Implementing JwtAuthenticationEntryPoint for unauthorized access
3. Creating JwtRequestFilter to validate tokens
4. Configuring HttpSecurity with stateless session management
5. Adding JWT filter before UsernamePasswordAuthenticationFilter

**Q: What are the security implications of CSRF and how does Spring Security handle it?**
A: CSRF attacks trick users into performing unwanted actions. Spring Security provides CSRF protection by:
1. Generating unique tokens for each session
2. Validating tokens on state-changing requests
3. Storing tokens in HttpSession or cookies
4. Automatically including tokens in forms via Thymeleaf integration

## External Resources and References

- [Spring Security Reference Documentation](https://docs.spring.io/spring-security/reference/)
- [Spring Security OAuth2 Guide](https://spring.io/guides/tutorials/spring-boot-oauth2/)
- [OWASP Security Guidelines](https://owasp.org/www-project-top-ten/)
- [JWT Best Practices](https://tools.ietf.org/html/rfc8725)
- [Spring Boot Security Auto-configuration](https://docs.spring.io/spring-boot/docs/current/reference/html/web.html#web.security)
- [Spring Security Test Documentation](https://docs.spring.io/spring-security/reference/servlet/test/index.html)

## Conclusion

Spring Security provides a comprehensive, flexible framework for securing Java applications. Its architecture based on filters, authentication managers, and security contexts allows for sophisticated security implementations while maintaining clean separation of concerns. Success with Spring Security requires understanding its core principles, proper configuration, and adherence to security best practices.

The framework's strength lies in its ability to handle complex security requirements while providing sensible defaults for common use cases. Whether building traditional web applications or modern microservices, Spring Security offers the tools and flexibility needed to implement robust security solutions.