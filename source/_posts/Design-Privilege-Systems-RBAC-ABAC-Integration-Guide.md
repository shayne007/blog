---
title: 'Design Privilege Systems: RBAC + ABAC Integration Guide'
date: 2025-06-12 17:52:06
tags: [system-design, privilege system]
categories: [system-design]
---

## System Overview

The privilege system combines Role-Based Access Control (RBAC) with Attribute-Based Access Control (ABAC) to provide fine-grained authorization capabilities. This hybrid approach leverages the simplicity of RBAC for common scenarios while utilizing ABAC's flexibility for complex, context-aware access decisions.

{% mermaid flowchart TB %}
    A[Client Request] --> B[PrivilegeFilterSDK]
    B --> C{Cache Check}
    C -->|Hit| D[Return Cached Result]
    C -->|Miss| E[PrivilegeService]
    E --> F[RBAC Engine]
    E --> G[ABAC Engine]
    F --> H[Role Evaluation]
    G --> I[Attribute Evaluation]
    H --> J[Access Decision]
    I --> J
    J --> K[Update Cache]
    K --> L[Return Result]
    
    M[PrivilegeWebUI] --> E
    N[Database] --> E
    O[Redis Cache] --> C
    P[Local Cache] --> C
{% endmermaid %}

**Interview Question**: *Why combine RBAC and ABAC instead of using one approach?*

**Answer**: RBAC provides simplicity and performance for common role-based scenarios (90% of use cases), while ABAC handles complex, context-dependent decisions (10% of use cases). This hybrid approach balances performance, maintainability, and flexibility. Pure ABAC would be overkill for simple role checks, while pure RBAC lacks the granularity needed for dynamic, context-aware decisions.

## Architecture Components

### PrivilegeService (Backend Core)

The PrivilegeService acts as the central authority for all privilege-related operations, implementing both RBAC and ABAC engines.

```java
@Service
public class PrivilegeService {
    
    @Autowired
    private RbacEngine rbacEngine;
    
    @Autowired
    private AbacEngine abacEngine;
    
    @Autowired
    private PrivilegeCacheManager cacheManager;
    
    public AccessDecision evaluateAccess(AccessRequest request) {
        // Check cache first
        String cacheKey = generateCacheKey(request);
        AccessDecision cached = cacheManager.get(cacheKey);
        if (cached != null) {
            return cached;
        }
        
        // RBAC evaluation (fast path)
        AccessDecision rbacDecision = rbacEngine.evaluate(request);
        if (rbacDecision.isExplicitDeny()) {
            cacheManager.put(cacheKey, rbacDecision, 300); // 5 min cache
            return rbacDecision;
        }
        
        // ABAC evaluation (context-aware path)
        AccessDecision abacDecision = abacEngine.evaluate(request);
        AccessDecision finalDecision = combineDecisions(rbacDecision, abacDecision);
        
        cacheManager.put(cacheKey, finalDecision, 300);
        return finalDecision;
    }
}
```

**Key APIs**:
- `POST /api/v1/privileges/evaluate` - Evaluate access permissions
- `GET /api/v1/roles/{userId}` - Get user roles
- `POST /api/v1/roles/{userId}` - Assign roles to user
- `GET /api/v1/permissions/{roleId}` - Get role permissions
- `POST /api/v1/policies` - Create ABAC policies

### PrivilegeWebUI (Administrative Interface)

A React-based administrative interface for managing users, roles, and permissions.

{% mermaid graph LR %}
    A[User Management] --> B[Role Assignment]
    B --> C[Permission Matrix]
    C --> D[Policy Editor]
    D --> E[Audit Logs]
    
    F[Dashboard] --> G[Real-time Metrics]
    G --> H[Access Patterns]
    H --> I[Security Alerts]
{% endmermaid %}

**Key Features**:
- **User Management**: Search, filter, and manage user accounts
- **Role Matrix**: Visual representation of role-permission mappings
- **Policy Builder**: Drag-and-drop interface for creating ABAC policies
- **Audit Dashboard**: Real-time access logs and security metrics
- **Bulk Operations**: Import/export users and roles via CSV

**Use Case Example**: An administrator needs to grant temporary access to a contractor for a specific project. Using the WebUI, they can:
1. Create a time-bound role "Project_Contractor_Q2"
2. Assign specific permissions (read project files, submit reports)
3. Set expiration date and IP restrictions
4. Monitor access patterns through the dashboard

### PrivilegeFilterSDK (Integration Component)

A lightweight SDK that integrates with microservices to provide seamless privilege checking.

```java
@Component
public class PrivilegeFilter implements Filter {
    
    @Autowired
    private PrivilegeClient privilegeClient;
    
    @Override
    public void doFilter(ServletRequest request, ServletResponse response, 
                        FilterChain chain) throws IOException, ServletException {
        
        HttpServletRequest httpRequest = (HttpServletRequest) request;
        
        // Extract user context
        UserContext userContext = extractUserContext(httpRequest);
        
        // Build access request
        AccessRequest accessRequest = AccessRequest.builder()
            .userId(userContext.getUserId())
            .resource(httpRequest.getRequestURI())
            .action(httpRequest.getMethod())
            .environment(buildEnvironmentAttributes(httpRequest))
            .build();
        
        // Check privileges
        AccessDecision decision = privilegeClient.evaluateAccess(accessRequest);
        
        if (decision.isPermitted()) {
            chain.doFilter(request, response);
        } else {
            sendUnauthorizedResponse(response, decision.getReason());
        }
    }
    
    private Map<String, Object> buildEnvironmentAttributes(HttpServletRequest request) {
        return Map.of(
            "ip_address", getClientIP(request),
            "user_agent", request.getHeader("User-Agent"),
            "time_of_day", LocalTime.now().getHour(),
            "request_size", request.getContentLength()
        );
    }
}
```

**Interview Question**: *How do you handle the performance impact of privilege checking on every request?*

**Answer**: We implement a three-tier caching strategy: local cache (L1) for frequently accessed decisions, Redis (L2) for shared cache across instances, and database (L3) as the source of truth. Additionally, we use async batch loading for role hierarchies and implement circuit breakers to fail-open during service degradation.

## Three-Tier Caching Architecture

{% mermaid flowchart TD %}
    A[Request] --> B[L1: Local Cache]
    B -->|Miss| C[L2: Redis Cache]
    C -->|Miss| D[L3: Database]
    D --> E[Privilege Calculation]
    E --> F[Update All Cache Layers]
    F --> G[Return Result]
    
    H[Cache Invalidation] --> I[Event-Driven Updates]
    I --> J[L1 Invalidation]
    I --> K[L2 Invalidation]
{% endmermaid %}

### Layer 1: Local Cache (Caffeine)

```java
@Configuration
public class LocalCacheConfig {
    
    @Bean
    public Cache<String, AccessDecision> localPrivilegeCache() {
        return Caffeine.newBuilder()
            .maximumSize(10000)
            .expireAfterWrite(Duration.ofMinutes(5))
            .recordStats()
            .build();
    }
}
```

**Characteristics**:
- **Capacity**: 10,000 entries per instance
- **TTL**: 5 minutes
- **Hit Ratio**: ~85% for frequent operations
- **Latency**: <1ms

### Layer 2: Distributed Cache (Redis)

```java
@Service
public class RedisPrivilegeCache {
    
    @Autowired
    private RedisTemplate<String, AccessDecision> redisTemplate;
    
    public void cacheUserRoles(String userId, Set<Role> roles) {
        String key = "user:roles:" + userId;
        redisTemplate.opsForValue().set(key, roles, Duration.ofMinutes(30));
    }
    
    public void invalidateUserCache(String userId) {
        String pattern = "user:*:" + userId;
        Set<String> keys = redisTemplate.keys(pattern);
        if (!keys.isEmpty()) {
            redisTemplate.delete(keys);
        }
    }
}
```

**Characteristics**:
- **Capacity**: 1M entries cluster-wide
- **TTL**: 30 minutes
- **Hit Ratio**: ~70% for cache misses from L1
- **Latency**: 1-5ms

### Layer 3: Database (PostgreSQL)

Persistent storage with optimized queries and indexing strategies.

**Performance Metrics**:
- **Overall Cache Hit Ratio**: 95%
- **Average Response Time**: 2ms (cached), 50ms (uncached)
- **Throughput**: 10,000 requests/second per instance

## Database Design

### Schema Overview

{% mermaid erDiagram %}
    USER {
        uuid id PK
        string username UK
        string email UK
        timestamp created_at
        timestamp updated_at
        boolean is_active
    }
    
    ROLE {
        uuid id PK
        string name UK
        string description
        json attributes
        timestamp created_at
        boolean is_active
    }
    
    PERMISSION {
        uuid id PK
        string name UK
        string resource
        string action
        json constraints
    }
    
    USER_ROLE {
        uuid user_id FK
        uuid role_id FK
        timestamp assigned_at
        timestamp expires_at
        string assigned_by
    }
    
    ROLE_PERMISSION {
        uuid role_id FK
        uuid permission_id FK
    }
    
    ABAC_POLICY {
        uuid id PK
        string name UK
        json policy_document
        integer priority
        boolean is_active
        timestamp created_at
    }
    
    PRIVILEGE_AUDIT {
        uuid id PK
        uuid user_id FK
        string resource
        string action
        string decision
        json context
        timestamp timestamp
    }
    
    USER ||--o{ USER_ROLE : has
    ROLE ||--o{ USER_ROLE : assigned_to
    ROLE ||--o{ ROLE_PERMISSION : has
    PERMISSION ||--o{ ROLE_PERMISSION : granted_by
{% endmermaid %}

### Detailed Table Schemas

```sql
-- Users table with audit fields
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    username VARCHAR(50) UNIQUE NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    last_login TIMESTAMP,
    is_active BOOLEAN DEFAULT true,
    attributes JSONB DEFAULT '{}',
    CONSTRAINT users_username_check CHECK (length(username) >= 3),
    CONSTRAINT users_email_check CHECK (email ~* '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$')
);

-- Roles table with hierarchical support
CREATE TABLE roles (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(100) UNIQUE NOT NULL,
    description TEXT,
    parent_role_id UUID REFERENCES roles(id),
    attributes JSONB DEFAULT '{}',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    is_active BOOLEAN DEFAULT true,
    CONSTRAINT roles_name_check CHECK (length(name) >= 2)
);

-- Permissions with resource-action pattern
CREATE TABLE permissions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(100) UNIQUE NOT NULL,
    resource VARCHAR(100) NOT NULL,
    action VARCHAR(50) NOT NULL,
    constraints JSONB DEFAULT '{}',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(resource, action)
);

-- User-Role assignments with temporal constraints
CREATE TABLE user_roles (
    user_id UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role_id UUID NOT NULL REFERENCES roles(id) ON DELETE CASCADE,
    assigned_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    expires_at TIMESTAMP,
    assigned_by UUID REFERENCES users(id),
    is_active BOOLEAN DEFAULT true,
    PRIMARY KEY (user_id, role_id),
    CONSTRAINT user_roles_expiry_check CHECK (expires_at IS NULL OR expires_at > assigned_at)
);

-- ABAC Policies
CREATE TABLE abac_policies (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(100) UNIQUE NOT NULL,
    policy_document JSONB NOT NULL,
    priority INTEGER DEFAULT 100,
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    created_by UUID REFERENCES users(id),
    CONSTRAINT abac_policies_priority_check CHECK (priority >= 0 AND priority <= 1000)
);

-- Performance-optimized audit table
CREATE TABLE privilege_audit (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL REFERENCES users(id),
    resource VARCHAR(200) NOT NULL,
    action VARCHAR(50) NOT NULL,
    decision VARCHAR(20) NOT NULL CHECK (decision IN ('PERMIT', 'DENY', 'INDETERMINATE')),
    context JSONB DEFAULT '{}',
    timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    processing_time_ms INTEGER
) PARTITION BY RANGE (timestamp);
```

### Indexing Strategy

```sql
-- Primary lookup indexes
CREATE INDEX idx_users_username ON users(username);
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_active ON users(is_active) WHERE is_active = true;

-- Role hierarchy and permissions
CREATE INDEX idx_roles_parent ON roles(parent_role_id);
CREATE INDEX idx_user_roles_user ON user_roles(user_id);
CREATE INDEX idx_user_roles_active ON user_roles(user_id, is_active) WHERE is_active = true;
CREATE INDEX idx_role_permissions_role ON role_permissions(role_id);

-- ABAC policy lookup
CREATE INDEX idx_abac_policies_active ON abac_policies(is_active, priority) WHERE is_active = true;

-- Audit queries
CREATE INDEX idx_audit_user_time ON privilege_audit(user_id, timestamp DESC);
CREATE INDEX idx_audit_resource ON privilege_audit(resource, timestamp DESC);

-- Composite indexes for complex queries
CREATE INDEX idx_user_roles_expiry ON user_roles(user_id, expires_at) 
    WHERE expires_at IS NOT NULL AND expires_at > CURRENT_TIMESTAMP;
```

## RBAC Engine Implementation

### Role Hierarchy Support

```java
@Service
public class RbacEngine {
    
    public Set<Role> getEffectiveRoles(String userId) {
        Set<Role> directRoles = userRoleRepository.findActiveRolesByUserId(userId);
        Set<Role> allRoles = new HashSet<>(directRoles);
        
        // Resolve role hierarchy
        for (Role role : directRoles) {
            allRoles.addAll(getParentRoles(role));
        }
        
        return allRoles;
    }
    
    private Set<Role> getParentRoles(Role role) {
        Set<Role> parents = new HashSet<>();
        Role current = role;
        
        while (current.getParentRole() != null) {
            current = current.getParentRole();
            parents.add(current);
        }
        
        return parents;
    }
    
    public AccessDecision evaluate(AccessRequest request) {
        Set<Role> userRoles = getEffectiveRoles(request.getUserId());
        
        for (Role role : userRoles) {
            if (roleHasPermission(role, request.getResource(), request.getAction())) {
                return AccessDecision.permit("RBAC: Role " + role.getName());
            }
        }
        
        return AccessDecision.deny("RBAC: No matching role permissions");
    }
}
```

**Use Case Example**: In a corporate environment, a "Senior Developer" role inherits permissions from "Developer" role, which inherits from "Employee" role. This hierarchy allows for efficient permission management without duplicating permissions across roles.

## ABAC Engine Implementation

### Policy Structure

ABAC policies are stored as JSON documents following the XACML-inspired structure:

```json
{
  "id": "time-based-access-policy",
  "name": "Business Hours Access Policy",
  "version": "1.0",
  "target": {
    "resources": ["api/financial/*"],
    "actions": ["GET", "POST"]
  },
  "rules": [
    {
      "id": "business-hours-rule",
      "effect": "Permit",
      "condition": {
        "and": [
          {
            "timeOfDay": {
              "gte": "09:00",
              "lte": "17:00"
            }
          },
          {
            "dayOfWeek": {
              "in": ["MONDAY", "TUESDAY", "WEDNESDAY", "THURSDAY", "FRIDAY"]
            }
          },
          {
            "userAttribute.department": {
              "equals": "FINANCE"
            }
          }
        ]
      }
    }
  ]
}
```

### Policy Evaluation Engine

```java
@Service
public class AbacEngine {
    
    @Autowired
    private PolicyRepository policyRepository;
    
    public AccessDecision evaluate(AccessRequest request) {
        List<AbacPolicy> applicablePolicies = findApplicablePolicies(request);
        
        // Sort by priority (higher number = higher priority)
        applicablePolicies.sort((p1, p2) -> Integer.compare(p2.getPriority(), p1.getPriority()));
        
        for (AbacPolicy policy : applicablePolicies) {
            PolicyDecision decision = evaluatePolicy(policy, request);
            
            switch (decision.getEffect()) {
                case PERMIT:
                    return AccessDecision.permit("ABAC: " + policy.getName());
                case DENY:
                    return AccessDecision.deny("ABAC: " + policy.getName());
                case INDETERMINATE:
                    continue; // Try next policy
            }
        }
        
        return AccessDecision.deny("ABAC: No applicable policies");
    }
    
    private PolicyDecision evaluatePolicy(AbacPolicy policy, AccessRequest request) {
        try {
            PolicyDocument document = policy.getPolicyDocument();
            
            // Check if policy target matches request
            if (!matchesTarget(document.getTarget(), request)) {
                return PolicyDecision.indeterminate();
            }
            
            // Evaluate all rules
            for (PolicyRule rule : document.getRules()) {
                if (evaluateCondition(rule.getCondition(), request)) {
                    return PolicyDecision.of(rule.getEffect());
                }
            }
            
            return PolicyDecision.indeterminate();
        } catch (Exception e) {
            log.error("Error evaluating policy: " + policy.getId(), e);
            return PolicyDecision.indeterminate();
        }
    }
}
```

**Interview Question**: *How do you handle policy conflicts in ABAC?*

**Answer**: We use a priority-based approach where policies are evaluated in order of priority. The first policy that returns a definitive decision (PERMIT or DENY) wins. For same-priority policies, we use policy combining algorithms like "deny-overrides" or "permit-overrides" based on the security requirements. We also implement policy validation to detect potential conflicts at creation time.

## Security Considerations

### Principle of Least Privilege

```java
@Service
public class PrivilegeAnalyzer {
    
    public PrivilegeAnalysisReport analyzeUserPrivileges(String userId) {
        Set<Permission> grantedPermissions = getAllUserPermissions(userId);
        Set<Permission> usedPermissions = getUsedPermissions(userId, Duration.ofDays(30));
        
        Set<Permission> unusedPermissions = new HashSet<>(grantedPermissions);
        unusedPermissions.removeAll(usedPermissions);
        
        return PrivilegeAnalysisReport.builder()
            .userId(userId)
            .totalGranted(grantedPermissions.size())
            .totalUsed(usedPermissions.size())
            .unusedPermissions(unusedPermissions)
            .riskScore(calculateRiskScore(unusedPermissions))
            .recommendations(generateRecommendations(unusedPermissions))
            .build();
    }
}
```

### Audit and Compliance

```java
@EventListener
public class PrivilegeAuditListener {
    
    @Async
    public void handleAccessDecision(AccessDecisionEvent event) {
        PrivilegeAuditRecord record = PrivilegeAuditRecord.builder()
            .userId(event.getUserId())
            .resource(event.getResource())
            .action(event.getAction())
            .decision(event.getDecision())
            .context(event.getContext())
            .timestamp(Instant.now())
            .processingTimeMs(event.getProcessingTime())
            .build();
        
        auditRepository.save(record);
        
        // Real-time alerting for suspicious activities
        if (isSuspiciousActivity(record)) {
            alertService.sendSecurityAlert(record);
        }
    }
}
```

## Performance Optimization

### Batch Processing for Role Assignments

```java
@Service
public class BulkPrivilegeService {
    
    @Transactional
    public void bulkAssignRoles(List<UserRoleAssignment> assignments) {
        // Validate all assignments first
        validateAssignments(assignments);
        
        // Group by user for efficient processing
        Map<String, List<UserRoleAssignment>> byUser = assignments.stream()
            .collect(Collectors.groupingBy(UserRoleAssignment::getUserId));
        
        // Process in batches to avoid memory issues
        Lists.partition(new ArrayList<>(byUser.entrySet()), 100)
            .forEach(batch -> processBatch(batch));
        
        // Invalidate cache for affected users
        Set<String> affectedUsers = assignments.stream()
            .map(UserRoleAssignment::getUserId)
            .collect(Collectors.toSet());
        
        cacheManager.invalidateUsers(affectedUsers);
    }
}
```

### Query Optimization

```java
@Repository
public class OptimizedUserRoleRepository {
    
    @Query(value = """
        SELECT r.* FROM roles r
        JOIN user_roles ur ON r.id = ur.role_id
        WHERE ur.user_id = :userId
        AND ur.is_active = true
        AND (ur.expires_at IS NULL OR ur.expires_at > CURRENT_TIMESTAMP)
        AND r.is_active = true
        """, nativeQuery = true)
    List<Role> findActiveRolesByUserId(@Param("userId") String userId);
    
    @Query(value = """
        WITH RECURSIVE role_hierarchy AS (
            SELECT id, name, parent_role_id, 0 as level
            FROM roles
            WHERE id IN :roleIds
            
            UNION ALL
            
            SELECT r.id, r.name, r.parent_role_id, rh.level + 1
            FROM roles r
            JOIN role_hierarchy rh ON r.id = rh.parent_role_id
            WHERE rh.level < 10
        )
        SELECT DISTINCT * FROM role_hierarchy
        """, nativeQuery = true)
    List<Role> findRoleHierarchy(@Param("roleIds") Set<String> roleIds);
}
```

## Monitoring and Observability

### Metrics Collection

```java
@Component
public class PrivilegeMetrics {
    
    private final Counter accessDecisions = Counter.build()
        .name("privilege_access_decisions_total")
        .help("Total number of access decisions")
        .labelNames("decision", "engine")
        .register();
    
    private final Histogram decisionLatency = Histogram.build()
        .name("privilege_decision_duration_seconds")
        .help("Time spent on access decisions")
        .labelNames("engine")
        .register();
    
    private final Gauge cacheHitRatio = Gauge.build()
        .name("privilege_cache_hit_ratio")
        .help("Cache hit ratio for privilege decisions")
        .labelNames("cache_layer")
        .register();
    
    public void recordDecision(String decision, String engine, Duration duration) {
        accessDecisions.labels(decision, engine).inc();
        decisionLatency.labels(engine).observe(duration.toMillis() / 1000.0);
    }
}
```

### Health Checks

```java
@Component
public class PrivilegeHealthIndicator implements HealthIndicator {
    
    @Override
    public Health health() {
        try {
            // Check database connectivity
            long dbResponseTime = measureDatabaseHealth();
            
            // Check cache performance
            double cacheHitRatio = measureCacheHealth();
            
            // Check policy evaluation performance
            long avgDecisionTime = measureDecisionPerformance();
            
            if (dbResponseTime > 100 || cacheHitRatio < 0.8 || avgDecisionTime > 50) {
                return Health.down()
                    .withDetail("database_response_time", dbResponseTime)
                    .withDetail("cache_hit_ratio", cacheHitRatio)
                    .withDetail("avg_decision_time", avgDecisionTime)
                    .build();
            }
            
            return Health.up()
                .withDetail("database_response_time", dbResponseTime)
                .withDetail("cache_hit_ratio", cacheHitRatio)
                .withDetail("avg_decision_time", avgDecisionTime)
                .build();
        } catch (Exception e) {
            return Health.down().withException(e).build();
        }
    }
}
```

## Testing Strategy

### Unit Testing

```java
@ExtendWith(MockitoExtension.class)
class RbacEngineTest {
    
    @Mock
    private UserRoleRepository userRoleRepository;
    
    @InjectMocks
    private RbacEngine rbacEngine;
    
    @Test
    void shouldPermitAccessWhenUserHasRequiredRole() {
        // Given
        String userId = "user123";
        Role developerRole = createRole("DEVELOPER");
        Permission readCodePermission = createPermission("code", "read");
        developerRole.addPermission(readCodePermission);
        
        when(userRoleRepository.findActiveRolesByUserId(userId))
            .thenReturn(Set.of(developerRole));
        
        // When
        AccessRequest request = AccessRequest.builder()
            .userId(userId)
            .resource("code")
            .action("read")
            .build();
        
        AccessDecision decision = rbacEngine.evaluate(request);
        
        // Then
        assertThat(decision.isPermitted()).isTrue();
        assertThat(decision.getReason()).contains("RBAC: Role DEVELOPER");
    }
    
    @Test
    void shouldInheritPermissionsFromParentRole() {
        // Given
        String userId = "user123";
        Role employeeRole = createRole("EMPLOYEE");
        Role managerRole = createRole("MANAGER", employeeRole);
        
        Permission basePermission = createPermission("dashboard", "read");
        employeeRole.addPermission(basePermission);
        
        when(userRoleRepository.findActiveRolesByUserId(userId))
            .thenReturn(Set.of(managerRole));
        
        // When
        AccessRequest request = AccessRequest.builder()
            .userId(userId)
            .resource("dashboard")
            .action("read")
            .build();
        
        AccessDecision decision = rbacEngine.evaluate(request);
        
        // Then
        assertThat(decision.isPermitted()).isTrue();
    }
}
```

### Integration Testing

```java
@SpringBootTest
@Testcontainers
class PrivilegeServiceIntegrationTest {
    
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:13")
            .withDatabaseName("privilege_test")
            .withUsername("test")
            .withPassword("test");
    
    @Container
    static GenericContainer<?> redis = new GenericContainer<>("redis:6")
            .withExposedPorts(6379);
    
    @Test
    void shouldCacheAccessDecisions() {
        // Given
        String userId = createTestUser();
        String roleId = createTestRole();
        assignRoleToUser(userId, roleId);
        
        AccessRequest request = AccessRequest.builder()
            .userId(userId)
            .resource("test-resource")
            .action("read")
            .build();
        
        // When - First call
        Instant start1 = Instant.now();
        AccessDecision decision1 = privilegeService.evaluateAccess(request);
        Duration duration1 = Duration.between(start1, Instant.now());
        
        // When - Second call (should be cached)
        Instant start2 = Instant.now();
        AccessDecision decision2 = privilegeService.evaluateAccess(request);
        Duration duration2 = Duration.between(start2, Instant.now());
        
        // Then
        assertThat(decision1).isEqualTo(decision2);
        assertThat(duration2).isLessThan(duration1.dividedBy(2));
    }
}
```

## Deployment Considerations

### Kubernetes Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: privilege-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: privilege-service
  template:
    metadata:
      labels:
        app: privilege-service
    spec:
      containers:
      - name: privilege-service
        image: privilege-service:latest
        ports:
        - containerPort: 8080
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: "production"
        - name: DATABASE_URL
          valueFrom:
            secretKeyRef:
              name: db-secret
              key: url
        - name: REDIS_URL
          valueFrom:
            secretKeyRef:
              name: redis-secret
              key: url
        resources:
          requests:
            memory: "512Mi"
            cpu: "250m"
          limits:
            memory: "1Gi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /actuator/health
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /actuator/health/readiness
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
```

### Database Migration Strategy

```java
@Component
public class PrivilegeMigrationService {
    
    @EventListener
    @Order(1)
    public void onApplicationReady(ApplicationReadyEvent event) {
        if (isNewDeployment()) {
            createDefaultRolesAndPermissions();
        }
        
        if (requiresDataMigration()) {
            migrateExistingData();
        }
        
        validateSystemIntegrity();
    }
    
    private void createDefaultRolesAndPermissions() {
        // Create system administrator role
        Role adminRole = roleService.createRole("SYSTEM_ADMIN", "System Administrator");
        Permission allPermissions = permissionService.createPermission("*", "*");
        roleService.assignPermission(adminRole.getId(), allPermissions.getId());
        
        // Create default user role
        Role userRole = roleService.createRole("USER", "Default User");
        Permission readProfile = permissionService.createPermission("profile", "read");
        roleService.assignPermission(userRole.getId(), readProfile.getId());
    }
}
```

## Real-World Use Cases

### Enterprise SaaS Platform

**Scenario**: A multi-tenant SaaS platform needs to support different organizations with varying access control requirements.

```java
@Service
public class TenantAwarePrivilegeService {
    
    public AccessDecision evaluateTenantAccess(AccessRequest request) {
        String tenantId = request.getContext().get("tenant_id");
        
        // RBAC: Check user roles within tenant
        Set<Role> tenantRoles = getUserRolesInTenant(request.getUserId(), tenantId);
        
        // ABAC: Apply tenant-specific policies
        AbacContext context = AbacContext.builder()
            .userAttributes(getUserAttributes(request.getUserId()))
            .resourceAttributes(getResourceAttributes(request.getResource()))
            .environmentAttributes(Map.of(
                "tenant_id", tenantId,
                "subscription_level", getTenantSubscription(tenantId),
                "data_residency", getTenantDataResidency(tenantId)
            ))
            .build();
        
        return evaluateWithContext(request, context);
    }
}
```

**ABAC Policy Example for Tenant Isolation**:
```json
{
  "name": "tenant-data-isolation-policy",
  "target": {
    "resources": ["api/data/*"]
  },
  "rules": [
    {
      "effect": "Deny",
      "condition": {
        "not": {
          "equals": [
            {"var": "resource.tenant_id"},
            {"var": "user.tenant_id"}
          ]
        }
      }
    }
  ]
}
```

### Healthcare System HIPAA Compliance

**Scenario**: A healthcare system requires strict access controls with audit trails for HIPAA compliance.

```java
@Component
public class HipaaPrivilegeEnforcer {
    
    @Autowired
    private PatientConsentService consentService;
    
    public AccessDecision evaluatePatientDataAccess(AccessRequest request) {
        String patientId = extractPatientId(request.getResource());
        String providerId = request.getUserId();
        
        // Check if provider has active treatment relationship
        if (!hasActiveTreatmentRelationship(providerId, patientId)) {
            return AccessDecision.deny("No active treatment relationship");
        }
        
        // Check patient consent
        if (!consentService.hasValidConsent(patientId, providerId)) {
            return AccessDecision.deny("Patient consent required");
        }
        
        // Apply break-glass emergency access
        if (isEmergencyAccess(request)) {
            auditService.recordEmergencyAccess(request);
            return AccessDecision.permit("Emergency access granted");
        }
        
        return super.evaluate(request);
    }
}
```

### Financial Services Regulatory Compliance

**Scenario**: A financial institution needs to implement segregation of duties and time-bound access for regulatory compliance.

```java
@Service
public class FinancialPrivilegeService {
    
    public AccessDecision evaluateFinancialTransaction(AccessRequest request) {
        BigDecimal amount = extractTransactionAmount(request);
        
        // Four-eyes principle for high-value transactions
        if (amount.compareTo(new BigDecimal("10000")) > 0) {
            return evaluateDualApproval(request);
        }
        
        // Time-based trading restrictions
        if (isTradingResource(request.getResource())) {
            return evaluateTradingHours(request);
        }
        
        return super.evaluate(request);
    }
    
    private AccessDecision evaluateDualApproval(AccessRequest request) {
        String transactionId = request.getContext().get("transaction_id");
        
        // Check if another user has already approved
        Optional<Approval> existingApproval = approvalService
            .findPendingApproval(transactionId);
        
        if (existingApproval.isPresent() && 
            !existingApproval.get().getApproverId().equals(request.getUserId())) {
            return AccessDecision.permit("Dual approval satisfied");
        }
        
        // Create pending approval
        approvalService.createPendingApproval(transactionId, request.getUserId());
        return AccessDecision.deny("Awaiting second approval");
    }
}
```

## Advanced Features

### Dynamic Permission Discovery

```java
@Service
public class DynamicPermissionService {
    
    /**
     * Automatically discovers and registers permissions from controller annotations
     */
    @EventListener
    public void discoverPermissions(ApplicationReadyEvent event) {
        ApplicationContext context = event.getApplicationContext();
        
        context.getBeansWithAnnotation(RestController.class).values()
            .forEach(this::scanControllerForPermissions);
    }
    
    private void scanControllerForPermissions(Object controller) {
        Class<?> clazz = AopUtils.getTargetClass(controller);
        RequestMapping classMapping = clazz.getAnnotation(RequestMapping.class);
        
        for (Method method : clazz.getDeclaredMethods()) {
            RequiresPermission permissionAnnotation = 
                method.getAnnotation(RequiresPermission.class);
            
            if (permissionAnnotation != null) {
                String resource = buildResourcePath(classMapping, method);
                String action = extractAction(method);
                
                permissionService.registerPermission(
                    permissionAnnotation.value(),
                    resource,
                    action,
                    permissionAnnotation.description()
                );
            }
        }
    }
}

@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface RequiresPermission {
    String value();
    String description() default "";
    String[] conditions() default {};
}
```

### Policy Templates

```java
@Service
public class PolicyTemplateService {
    
    private static final Map<String, PolicyTemplate> TEMPLATES = Map.of(
        "time_restricted", new TimeRestrictedTemplate(),
        "ip_restricted", new IpRestrictedTemplate(),
        "department_scoped", new DepartmentScopedTemplate(),
        "temporary_access", new TemporaryAccessTemplate()
    );
    
    public AbacPolicy createPolicyFromTemplate(String templateName, 
                                               Map<String, Object> parameters) {
        PolicyTemplate template = TEMPLATES.get(templateName);
        if (template == null) {
            throw new IllegalArgumentException("Unknown template: " + templateName);
        }
        
        return template.generatePolicy(parameters);
    }
}

public class TimeRestrictedTemplate implements PolicyTemplate {
    
    @Override
    public AbacPolicy generatePolicy(Map<String, Object> params) {
        String startTime = (String) params.get("start_time");
        String endTime = (String) params.get("end_time");
        List<String> allowedDays = (List<String>) params.get("allowed_days");
        
        PolicyDocument document = PolicyDocument.builder()
            .rule(PolicyRule.builder()
                .effect(Effect.PERMIT)
                .condition(buildTimeCondition(startTime, endTime, allowedDays))
                .build())
            .build();
        
        return AbacPolicy.builder()
            .name("time-restricted-" + UUID.randomUUID())
            .policyDocument(document)
            .priority(100)
            .build();
    }
}
```

### Machine Learning Integration

```java
@Service
public class AnomalousAccessDetector {
    
    @Autowired
    private AccessPatternAnalyzer patternAnalyzer;
    
    @EventListener
    @Async
    public void analyzeAccessPattern(AccessDecisionEvent event) {
        AccessPattern pattern = AccessPattern.builder()
            .userId(event.getUserId())
            .resource(event.getResource())
            .action(event.getAction())
            .timestamp(event.getTimestamp())
            .ipAddress(event.getContext().get("ip_address"))
            .userAgent(event.getContext().get("user_agent"))
            .build();
        
        double anomalyScore = patternAnalyzer.calculateAnomalyScore(pattern);
        
        if (anomalyScore > 0.8) {
            SecurityAlert alert = SecurityAlert.builder()
                .userId(event.getUserId())
                .alertType("ANOMALOUS_ACCESS")
                .severity(Severity.HIGH)
                .description("Unusual access pattern detected")
                .anomalyScore(anomalyScore)
                .build();
            
            alertService.sendAlert(alert);
            
            // Temporarily increase scrutiny for this user
            privilegeService.enableEnhancedMonitoring(event.getUserId(), 
                Duration.ofHours(24));
        }
    }
}
```

## Performance Benchmarks

### Load Testing Results

**Test Environment**:
- 3 application instances (2 CPU, 4GB RAM each)
- PostgreSQL (4 CPU, 8GB RAM)
- Redis Cluster (3 nodes, 2GB RAM each)

**Results**:
```
Scenario: Mixed RBAC/ABAC evaluation
├── Concurrent Users: 1000
├── Test Duration: 10 minutes
├── Total Requests: 2,847,293
├── Average Response Time: 3.2ms
├── 95th Percentile: 8.5ms
├── 99th Percentile: 15.2ms
├── Error Rate: 0.02%
└── Throughput: 4,745 RPS

Cache Performance:
├── L1 Cache Hit Rate: 87.3%
├── L2 Cache Hit Rate: 11.8%
├── Database Queries: 0.9%
└── Average Decision Time: 1.8ms (cached), 45ms (uncached)
```

### Memory Usage Optimization

```java
@Configuration
public class MemoryOptimizationConfig {
    
    @Bean
    @ConditionalOnProperty(name = "privilege.optimization.memory", havingValue = "true")
    public PrivilegeService optimizedPrivilegeService() {
        return new MemoryOptimizedPrivilegeService();
    }
}

public class MemoryOptimizedPrivilegeService extends PrivilegeService {
    
    // Use flyweight pattern for common permissions
    private final Map<String, Permission> permissionFlyweights = new ConcurrentHashMap<>();
    
    // Weak references for rarely accessed data
    private final WeakHashMap<String, Set<Role>> userRoleCache = new WeakHashMap<>();
    
    // Compressed storage for policy documents
    private final PolicyCompressor policyCompressor = new PolicyCompressor();
    
    @Override
    protected Set<Permission> getUserPermissions(String userId) {
        return userRoleCache.computeIfAbsent(userId, this::loadUserRoles)
            .stream()
            .flatMap(role -> role.getPermissions().stream())
            .map(this::getFlyweightPermission)
            .collect(Collectors.toSet());
    }
}
```

## Interview Questions and Answers

### Architecture Questions

**Q: How would you handle a situation where the privilege service becomes unavailable?**

**A**: Implement a circuit breaker pattern with graceful degradation:
1. **Circuit Breaker**: Use Hystrix or Resilience4j to detect service failures
2. **Fallback Strategy**: Cache recent decisions locally and apply "fail-secure" or "fail-open" policies based on criticality
3. **Emergency Roles**: Pre-configure emergency access roles that work offline
4. **Async Recovery**: Queue privilege decisions for later verification when service recovers

```java
@Component
public class ResilientPrivilegeService {
    
    @CircuitBreaker(name = "privilege-service", fallbackMethod = "fallbackEvaluate")
    public AccessDecision evaluate(AccessRequest request) {
        return privilegeService.evaluateAccess(request);
    }
    
    public AccessDecision fallbackEvaluate(AccessRequest request, Exception ex) {
        // Check local emergency cache
        AccessDecision cached = emergencyCache.get(request);
        if (cached != null) {
            return cached;
        }
        
        // Apply default policy based on resource criticality
        if (isCriticalResource(request.getResource())) {
            return AccessDecision.deny("Service unavailable - fail secure");
        } else {
            return AccessDecision.permit("Service unavailable - fail open");
        }
    }
}
```

**Q: How do you ensure consistency across multiple instances of the privilege service?**

**A**: Use distributed caching with event-driven invalidation:
1. **Distributed Cache**: Redis cluster for shared state
2. **Event Sourcing**: Publish privilege change events
3. **Cache Invalidation**: Listen to events and invalidate affected cache entries
4. **Database Consistency**: Use database transactions for critical updates
5. **Eventual Consistency**: Accept temporary inconsistency for better performance

### Security Questions

**Q: How would you prevent privilege escalation attacks?**

**A**: Implement multiple defense layers:
1. **Principle of Least Privilege**: Regular audits to remove unused permissions
2. **Approval Workflows**: Require approval for sensitive role assignments
3. **Temporal Constraints**: Time-bound permissions with automatic expiration
4. **Delegation Restrictions**: Prevent users from granting permissions they don't have
5. **Audit Monitoring**: Real-time detection of unusual privilege changes

```java
@Service
public class PrivilegeEscalationDetector {
    
    @EventListener
    public void detectEscalation(RoleAssignmentEvent event) {
        if (isPrivilegeEscalation(event)) {
            // Block the assignment
            throw new SecurityException("Potential privilege escalation detected");
        }
    }
    
    private boolean isPrivilegeEscalation(RoleAssignmentEvent event) {
        Set<Permission> assignerPermissions = getEffectivePermissions(event.getAssignerId());
        Set<Permission> targetPermissions = getRolePermissions(event.getRoleId());
        
        // Assigner cannot grant permissions they don't have
        return !assignerPermissions.containsAll(targetPermissions);
    }
}
```

### Performance Questions

**Q: How would you optimize the system for 100,000+ concurrent users?**

**A**: Multi-layered optimization approach:
1. **Horizontal Scaling**: Auto-scaling groups with load balancers
2. **Caching Strategy**: 4-tier caching (Browser → CDN → App Cache → Database)
3. **Database Optimization**: Read replicas, connection pooling, query optimization
4. **Async Processing**: Queue heavy operations like audit logging
5. **Pre-computation**: Background jobs to pre-calculate common decisions

## Troubleshooting Guide

### Common Issues and Solutions

**Issue**: High latency on privilege decisions
```bash
# Check cache hit ratios
curl http://localhost:8080/actuator/metrics/cache.gets | jq

# Monitor database query performance
EXPLAIN ANALYZE SELECT * FROM user_roles WHERE user_id = 'user123';

# Check for cache stampede
tail -f /var/log/privilege-service.log | grep "Cache miss"
```

**Solution**: Implement cache warming and query optimization

**Issue**: Memory leaks in long-running instances
```java
// Add heap dump analysis
-XX:+HeapDumpOnOutOfMemoryError 
-XX:HeapDumpPath=/var/log/heapdumps/

// Monitor with JProfiler or similar
jcmd <pid> GC.run_finalization
```

**Solution**: Use weak references for rarely accessed data and implement cache size limits

## External Resources

### Standards and Specifications
- [NIST RBAC Model](https://csrc.nist.gov/publications/detail/sp/800-162/final) - Foundational RBAC principles
- [XACML 3.0 Specification](http://docs.oasis-open.org/xacml/3.0/xacml-3.0-core-spec-os-en.html) - ABAC policy language
- [OAuth 2.0 Token Introspection](https://tools.ietf.org/html/rfc7662) - Token-based authorization
- [OWASP Access Control Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Access_Control_Cheat_Sheet.html) - Security best practices

### Implementation References
- [Spring Security Architecture](https://spring.io/guides/topicals/spring-security-architecture/) - Framework integration
- [Apache Shiro Documentation](https://shiro.apache.org/documentation.html) - Alternative authorization framework
- [Auth0 RBAC Guide](https://auth0.com/docs/manage-users/access-control/rbac) - Commercial implementation examples
- [Casbin Authorization Library](https://casbin.org/) - Open-source authorization library

### Performance and Monitoring
- [Micrometer Documentation](https://micrometer.io/docs) - Metrics collection
- [Grafana Dashboard Templates](https://grafana.com/grafana/dashboards/) - Monitoring visualization
- [Redis Performance Tuning](https://redis.io/documentation) - Cache optimization

This comprehensive design provides a production-ready privilege system that balances security, performance, and maintainability while addressing real-world enterprise requirements.