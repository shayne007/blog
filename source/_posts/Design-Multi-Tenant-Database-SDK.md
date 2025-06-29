---
title: Design Multi-Tenant Database SDK
date: 2025-06-13 18:40:02
tags: [system-design, async invocation]
categories: [system-design]
---

## Overview and Architecture

A Multi-Tenant Database SDK is a critical component in modern SaaS architectures that enables applications to dynamically manage database connections and operations across multiple tenants. This SDK provides a unified interface for database operations while maintaining tenant isolation and optimizing resource utilization through connection pooling and runtime datasource switching.

### Core Architecture Components

{% mermaid graph TB %}
    A[SaaS Application] --> B[Multi-Tenant SDK]
    B --> C[Tenant Context Manager]
    B --> D[Connection Pool Manager]
    B --> E[Database Provider Factory]
    
    C --> F[ThreadLocal Storage]
    D --> G[MySQL Connection Pool]
    D --> H[PostgreSQL Connection Pool]
    
    E --> I[MySQL Provider]
    E --> J[PostgreSQL Provider]
    
    I --> K[(MySQL Database)]
    J --> L[(PostgreSQL Database)]
    
    B --> M[SPI Registry]
    M --> N[Database Provider Interface]
    N --> I
    N --> J
{% endmermaid %}

**Interview Insight**: *"How would you design a multi-tenant database architecture?"*

The key is to balance tenant isolation with resource efficiency. Our SDK uses a **database-per-tenant** approach with dynamic datasource switching, which provides strong isolation while maintaining performance through connection pooling.

## Tenant Context Management

### ThreadLocal Implementation

The tenant context is stored using ThreadLocal to ensure thread-safe tenant identification throughout the request lifecycle.

```java
public class TenantContext {
    private static final ThreadLocal<String> TENANT_ID = new ThreadLocal<>();
    private static final ThreadLocal<String> DATABASE_NAME = new ThreadLocal<>();
    
    public static void setTenant(String tenantId) {
        TENANT_ID.set(tenantId);
        DATABASE_NAME.set("tenant_" + tenantId);
    }
    
    public static String getCurrentTenant() {
        return TENANT_ID.get();
    }
    
    public static String getCurrentDatabase() {
        return DATABASE_NAME.get();
    }
    
    public static void clear() {
        TENANT_ID.remove();
        DATABASE_NAME.remove();
    }
}
```

### Tenant Context Interceptor

```java
@Component
public class TenantContextInterceptor implements HandlerInterceptor {
    
    @Override
    public boolean preHandle(HttpServletRequest request, 
                           HttpServletResponse response, 
                           Object handler) throws Exception {
        
        String tenantId = extractTenantId(request);
        if (tenantId != null) {
            TenantContext.setTenant(tenantId);
        }
        return true;
    }
    
    @Override
    public void afterCompletion(HttpServletRequest request, 
                              HttpServletResponse response, 
                              Object handler, Exception ex) {
        TenantContext.clear();
    }
    
    private String extractTenantId(HttpServletRequest request) {
        // Extract from header, JWT token, or subdomain
        return request.getHeader("X-Tenant-ID");
    }
}
```

**Interview Insight**: *"Why use ThreadLocal for tenant context?"*

ThreadLocal ensures that each request thread maintains its own tenant context without interference from other concurrent requests. This is crucial in multi-threaded web applications where multiple tenants' requests are processed simultaneously.

## Connection Pool Management

### Dynamic DataSource Configuration

```java
@Configuration
public class MultiTenantDataSourceConfig {
    
    @Bean
    public DataSource multiTenantDataSource() {
        MultiTenantDataSource dataSource = new MultiTenantDataSource();
        dataSource.setDefaultTargetDataSource(createDefaultDataSource());
        return dataSource;
    }
    
    @Bean
    public ConnectionPoolManager connectionPoolManager() {
        return new ConnectionPoolManager();
    }
}
```

### Connection Pool Manager Implementation

```java
@Component
public class ConnectionPoolManager {
    private final Map<String, HikariDataSource> dataSources = new ConcurrentHashMap<>();
    private final DatabaseProviderFactory providerFactory;
    
    public ConnectionPoolManager(DatabaseProviderFactory providerFactory) {
        this.providerFactory = providerFactory;
    }
    
    public DataSource getDataSource(String tenantId) {
        return dataSources.computeIfAbsent(tenantId, this::createDataSource);
    }
    
    private HikariDataSource createDataSource(String tenantId) {
        TenantConfig config = getTenantConfig(tenantId);
        DatabaseProvider provider = providerFactory.getProvider(config.getDatabaseType());
        
        HikariConfig hikariConfig = new HikariConfig();
        hikariConfig.setJdbcUrl(provider.buildJdbcUrl(config));
        hikariConfig.setUsername(config.getUsername());
        hikariConfig.setPassword(config.getPassword());
        hikariConfig.setMaximumPoolSize(config.getMaxPoolSize());
        hikariConfig.setMinimumIdle(config.getMinIdle());
        hikariConfig.setConnectionTimeout(config.getConnectionTimeout());
        hikariConfig.setIdleTimeout(config.getIdleTimeout());
        
        return new HikariDataSource(hikariConfig);
    }
    
    public void closeTenantDataSource(String tenantId) {
        HikariDataSource dataSource = dataSources.remove(tenantId);
        if (dataSource != null) {
            dataSource.close();
        }
    }
}
```

**Interview Insight**: *"How do you handle connection pool sizing for multiple tenants?"*

We use adaptive pool sizing based on tenant usage patterns. Each tenant gets a dedicated connection pool with configurable min/max connections. Monitor pool metrics and adjust dynamically based on tenant activity.

## Database Provider Implementation via SPI

### Service Provider Interface

```java
public interface DatabaseProvider {
    String getProviderName();
    String buildJdbcUrl(TenantConfig config);
    void createTenantDatabase(TenantConfig config);
    void createTenantTables(String tenantId, List<String> tableSchemas);
    boolean supportsBatch();
    String getDriverClassName();
}
```

### MySQL Provider Implementation

```java
@Component
public class MySQLDatabaseProvider implements DatabaseProvider {
    
    @Override
    public String getProviderName() {
        return "mysql";
    }
    
    @Override
    public String buildJdbcUrl(TenantConfig config) {
        return String.format("jdbc:mysql://%s:%d/%s?useSSL=true&serverTimezone=UTC",
                config.getHost(), config.getPort(), config.getDatabaseName());
    }
    
    @Override
    public void createTenantDatabase(TenantConfig config) {
        try (Connection connection = getAdminConnection(config)) {
            String sql = "CREATE DATABASE IF NOT EXISTS " + config.getDatabaseName() + 
                        " CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci";
            
            try (Statement stmt = connection.createStatement()) {
                stmt.execute(sql);
            }
        } catch (SQLException e) {
            throw new DatabaseException("Failed to create MySQL database for tenant: " + 
                                      config.getTenantId(), e);
        }
    }
    
    @Override
    public void createTenantTables(String tenantId, List<String> tableSchemas) {
        DataSource dataSource = connectionPoolManager.getDataSource(tenantId);
        
        try (Connection connection = dataSource.getConnection()) {
            connection.setAutoCommit(false);
            
            for (String schema : tableSchemas) {
                try (Statement stmt = connection.createStatement()) {
                    stmt.execute(schema);
                }
            }
            
            connection.commit();
        } catch (SQLException e) {
            throw new DatabaseException("Failed to create tables for tenant: " + tenantId, e);
        }
    }
    
    @Override
    public String getDriverClassName() {
        return "com.mysql.cj.jdbc.Driver";
    }
}
```

### PostgreSQL Provider Implementation

```java
@Component
public class PostgreSQLDatabaseProvider implements DatabaseProvider {
    
    @Override
    public String getProviderName() {
        return "postgresql";
    }
    
    @Override
    public String buildJdbcUrl(TenantConfig config) {
        return String.format("jdbc:postgresql://%s:%d/%s",
                config.getHost(), config.getPort(), config.getDatabaseName());
    }
    
    @Override
    public void createTenantDatabase(TenantConfig config) {
        try (Connection connection = getAdminConnection(config)) {
            String sql = "CREATE DATABASE " + config.getDatabaseName() + 
                        " WITH ENCODING 'UTF8' LC_COLLATE='en_US.UTF-8' LC_CTYPE='en_US.UTF-8'";
            
            try (Statement stmt = connection.createStatement()) {
                stmt.execute(sql);
            }
        } catch (SQLException e) {
            throw new DatabaseException("Failed to create PostgreSQL database for tenant: " + 
                                      config.getTenantId(), e);
        }
    }
    
    @Override
    public void createTenantTables(String tenantId, List<String> tableSchemas) {
        DataSource dataSource = connectionPoolManager.getDataSource(tenantId);
        
        try (Connection connection = dataSource.getConnection()) {
            connection.setAutoCommit(false);
            
            for (String schema : tableSchemas) {
                try (Statement stmt = connection.createStatement()) {
                    stmt.execute(schema);
                }
            }
            
            connection.commit();
        } catch (SQLException e) {
            throw new DatabaseException("Failed to create tables for tenant: " + tenantId, e);
        }
    }
    
    @Override
    public String getDriverClassName() {
        return "org.postgresql.Driver";
    }
}
```

### SPI Registry and Factory

```java
@Component
public class DatabaseProviderFactory {
    private final Map<String, DatabaseProvider> providers = new HashMap<>();
    
    @PostConstruct
    public void initializeProviders() {
        ServiceLoader<DatabaseProvider> serviceLoader = ServiceLoader.load(DatabaseProvider.class);
        
        for (DatabaseProvider provider : serviceLoader) {
            providers.put(provider.getProviderName(), provider);
        }
    }
    
    public DatabaseProvider getProvider(String providerName) {
        DatabaseProvider provider = providers.get(providerName.toLowerCase());
        if (provider == null) {
            throw new UnsupportedDatabaseException("Database provider not found: " + providerName);
        }
        return provider;
    }
    
    public Set<String> getSupportedProviders() {
        return providers.keySet();
    }
}
```

**Interview Insight**: *"Why use SPI pattern for database providers?"*

SPI (Service Provider Interface) enables loose coupling and extensibility. New database providers can be added without modifying existing code, following the Open/Closed Principle. It also allows for plugin-based architecture where providers can be loaded dynamically.

## Multi-Tenant Database Operations

### Core SDK Interface

```java
public interface MultiTenantDatabaseSDK {
    void createTenant(String tenantId, TenantConfig config);
    void deleteTenant(String tenantId);
    void executeSql(String sql, Object... params);
    <T> List<T> query(String sql, RowMapper<T> rowMapper, Object... params);
    void executeBatch(List<String> sqlStatements);
    void executeTransaction(TransactionCallback callback);
}
```

### SDK Implementation

```java
@Service
public class MultiTenantDatabaseSDKImpl implements MultiTenantDatabaseSDK {
    
    private final ConnectionPoolManager connectionPoolManager;
    private final DatabaseProviderFactory providerFactory;
    private final TenantConfigRepository tenantConfigRepository;
    
    @Override
    public void createTenant(String tenantId, TenantConfig config) {
        try {
            // Create database
            DatabaseProvider provider = providerFactory.getProvider(config.getDatabaseType());
            provider.createTenantDatabase(config);
            
            // Create tables
            List<String> tableSchemas = loadTableSchemas();
            provider.createTenantTables(tenantId, tableSchemas);
            
            // Save tenant configuration
            tenantConfigRepository.save(config);
            
            // Initialize connection pool
            connectionPoolManager.getDataSource(tenantId);
            
        } catch (Exception e) {
            throw new TenantCreationException("Failed to create tenant: " + tenantId, e);
        }
    }
    
    @Override
    public void executeSql(String sql, Object... params) {
        String tenantId = TenantContext.getCurrentTenant();
        DataSource dataSource = connectionPoolManager.getDataSource(tenantId);
        
        try (Connection connection = dataSource.getConnection();
             PreparedStatement stmt = connection.prepareStatement(sql)) {
            
            setParameters(stmt, params);
            stmt.execute();
            
        } catch (SQLException e) {
            throw new DatabaseException("Failed to execute SQL for tenant: " + tenantId, e);
        }
    }
    
    @Override
    public <T> List<T> query(String sql, RowMapper<T> rowMapper, Object... params) {
        String tenantId = TenantContext.getCurrentTenant();
        DataSource dataSource = connectionPoolManager.getDataSource(tenantId);
        
        List<T> results = new ArrayList<>();
        
        try (Connection connection = dataSource.getConnection();
             PreparedStatement stmt = connection.prepareStatement(sql)) {
            
            setParameters(stmt, params);
            
            try (ResultSet rs = stmt.executeQuery()) {
                while (rs.next()) {
                    results.add(rowMapper.mapRow(rs));
                }
            }
            
        } catch (SQLException e) {
            throw new DatabaseException("Failed to query for tenant: " + tenantId, e);
        }
        
        return results;
    }
    
    @Override
    public void executeTransaction(TransactionCallback callback) {
        String tenantId = TenantContext.getCurrentTenant();
        DataSource dataSource = connectionPoolManager.getDataSource(tenantId);
        
        try (Connection connection = dataSource.getConnection()) {
            connection.setAutoCommit(false);
            
            try {
                callback.doInTransaction(connection);
                connection.commit();
            } catch (Exception e) {
                connection.rollback();
                throw e;
            }
            
        } catch (SQLException e) {
            throw new DatabaseException("Transaction failed for tenant: " + tenantId, e);
        }
    }
}
```

## Production Use Cases and Examples

### Use Case 1: SaaS CRM System

```java
@RestController
@RequestMapping("/api/customers")
public class CustomerController {
    
    private final MultiTenantDatabaseSDK databaseSDK;
    
    @GetMapping
    public List<Customer> getCustomers() {
        return databaseSDK.query(
            "SELECT * FROM customers WHERE active = ?",
            (rs) -> new Customer(
                rs.getLong("id"),
                rs.getString("name"),
                rs.getString("email")
            ),
            true
        );
    }
    
    @PostMapping
    public void createCustomer(@RequestBody Customer customer) {
        databaseSDK.executeSql(
            "INSERT INTO customers (name, email, created_at) VALUES (?, ?, ?)",
            customer.getName(),
            customer.getEmail(),
            Timestamp.from(Instant.now())
        );
    }
}
```

### Use Case 2: Tenant Onboarding Process

```java
@Service
public class TenantOnboardingService {
    
    private final MultiTenantDatabaseSDK databaseSDK;
    
    public void onboardNewTenant(TenantRegistration registration) {
        TenantConfig config = TenantConfig.builder()
            .tenantId(registration.getTenantId())
            .databaseType("mysql")
            .host("localhost")
            .port(3306)
            .databaseName("tenant_" + registration.getTenantId())
            .username("tenant_user")
            .password(generateSecurePassword())
            .maxPoolSize(10)
            .minIdle(2)
            .build();
        
        try {
            // Create tenant database and tables
            databaseSDK.createTenant(registration.getTenantId(), config);
            
            // Insert initial data
            insertInitialData(registration);
            
            // Send welcome email
            sendWelcomeEmail(registration);
            
        } catch (Exception e) {
            // Rollback tenant creation
            databaseSDK.deleteTenant(registration.getTenantId());
            throw new TenantOnboardingException("Failed to onboard tenant", e);
        }
    }
}
```

### Use Case 3: Data Migration Between Tenants

```java
@Service
public class TenantDataMigrationService {
    
    private final MultiTenantDatabaseSDK databaseSDK;
    
    public void migrateTenantData(String sourceTenantId, String targetTenantId) {
        // Export data from source tenant
        TenantContext.setTenant(sourceTenantId);
        List<Customer> customers = databaseSDK.query(
            "SELECT * FROM customers",
            this::mapCustomer
        );
        
        // Import data to target tenant
        TenantContext.setTenant(targetTenantId);
        databaseSDK.executeTransaction(connection -> {
            for (Customer customer : customers) {
                PreparedStatement stmt = connection.prepareStatement(
                    "INSERT INTO customers (name, email, created_at) VALUES (?, ?, ?)"
                );
                stmt.setString(1, customer.getName());
                stmt.setString(2, customer.getEmail());
                stmt.setTimestamp(3, customer.getCreatedAt());
                stmt.executeUpdate();
            }
        });
    }
}
```

## Runtime Datasource Switching

### Dynamic DataSource Routing

```java
public class MultiTenantDataSource extends AbstractRoutingDataSource {
    
    @Override
    protected Object determineCurrentLookupKey() {
        return TenantContext.getCurrentTenant();
    }
    
    @Override
    protected DataSource determineTargetDataSource() {
        String tenantId = TenantContext.getCurrentTenant();
        if (tenantId == null) {
            return getDefaultDataSource();
        }
        
        return connectionPoolManager.getDataSource(tenantId);
    }
}
```

### Request Flow Diagram

{% mermaid sequenceDiagram %}
    participant Client
    participant API Gateway
    participant SaaS Service
    participant SDK
    participant Database
    
    Client->>API Gateway: Request with tenant info
    API Gateway->>SaaS Service: Forward request
    SaaS Service->>SDK: Set tenant context
    SDK->>SDK: Store in ThreadLocal
    SaaS Service->>SDK: Execute database operation
    SDK->>SDK: Determine datasource
    SDK->>Database: Execute query
    Database->>SDK: Return results
    SDK->>SaaS Service: Return results
    SaaS Service->>Client: Return response
{% endmermaid %}

**Interview Insight**: *"How do you handle database connection switching at runtime?"*

We use Spring's AbstractRoutingDataSource combined with ThreadLocal tenant context. The routing happens transparently - when a database operation is requested, the SDK determines the appropriate datasource based on the current tenant context stored in ThreadLocal.

## Performance Optimization Strategies

### Connection Pool Tuning

```java
@ConfigurationProperties(prefix = "multitenant.pool")
public class ConnectionPoolConfig {
    private int maxPoolSize = 10;
    private int minIdle = 2;
    private long connectionTimeout = 30000;
    private long idleTimeout = 600000;
    private long maxLifetime = 1800000;
    private int leakDetectionThreshold = 60000;
    
    // Getters and setters
}
```

### Connection Pool Monitoring

```java
@Component
public class ConnectionPoolMonitor {
    
    private final MeterRegistry meterRegistry;
    private final ConnectionPoolManager poolManager;
    
    @Scheduled(fixedRate = 30000)
    public void monitorConnectionPools() {
        poolManager.getAllDataSources().forEach((tenantId, dataSource) -> {
            HikariPoolMXBean poolMXBean = dataSource.getHikariPoolMXBean();
            
            Gauge.builder("connection.pool.active")
                .tag("tenant", tenantId)
                .register(meterRegistry, poolMXBean, HikariPoolMXBean::getActiveConnections);
                
            Gauge.builder("connection.pool.idle")
                .tag("tenant", tenantId)
                .register(meterRegistry, poolMXBean, HikariPoolMXBean::getIdleConnections);
        });
    }
}
```

### Caching Strategy

```java
@Service
public class TenantConfigCacheService {
    
    private final LoadingCache<String, TenantConfig> configCache;
    
    public TenantConfigCacheService() {
        this.configCache = Caffeine.newBuilder()
            .maximumSize(1000)
            .expireAfterWrite(30, TimeUnit.MINUTES)
            .build(this::loadTenantConfig);
    }
    
    public TenantConfig getTenantConfig(String tenantId) {
        return configCache.get(tenantId);
    }
    
    private TenantConfig loadTenantConfig(String tenantId) {
        return tenantConfigRepository.findByTenantId(tenantId)
            .orElseThrow(() -> new TenantNotFoundException("Tenant not found: " + tenantId));
    }
}
```

## Security and Compliance

### Tenant Isolation Security

```java
@Component
public class TenantSecurityValidator {
    
    public void validateTenantAccess(String requestedTenantId, String authenticatedTenantId) {
        if (!requestedTenantId.equals(authenticatedTenantId)) {
            throw new TenantAccessDeniedException("Cross-tenant access denied");
        }
    }
    
    public void validateSqlInjection(String sql) {
        if (containsSqlInjectionPatterns(sql)) {
            throw new SecurityException("Potential SQL injection detected");
        }
    }
    
    private boolean containsSqlInjectionPatterns(String sql) {
        String[] patterns = {"';", "DROP", "DELETE", "UPDATE", "INSERT", "UNION"};
        String upperSql = sql.toUpperCase();
        
        return Arrays.stream(patterns)
            .anyMatch(upperSql::contains);
    }
}
```

### Encryption and Data Protection

```java
@Component
public class DataEncryptionService {
    
    private final AESUtil aesUtil;
    
    public String encryptSensitiveData(String data, String tenantId) {
        String tenantKey = generateTenantSpecificKey(tenantId);
        return aesUtil.encrypt(data, tenantKey);
    }
    
    public String decryptSensitiveData(String encryptedData, String tenantId) {
        String tenantKey = generateTenantSpecificKey(tenantId);
        return aesUtil.decrypt(encryptedData, tenantKey);
    }
    
    private String generateTenantSpecificKey(String tenantId) {
        // Generate tenant-specific encryption key
        return keyDerivationService.deriveKey(tenantId);
    }
}
```

## Error Handling and Resilience

### Exception Hierarchy

```java
public class DatabaseException extends RuntimeException {
    public DatabaseException(String message, Throwable cause) {
        super(message, cause);
    }
}

public class TenantNotFoundException extends DatabaseException {
    public TenantNotFoundException(String message) {
        super(message, null);
    }
}

public class TenantCreationException extends DatabaseException {
    public TenantCreationException(String message, Throwable cause) {
        super(message, cause);
    }
}
```

### Retry Mechanism

```java
@Component
public class DatabaseRetryService {
    
    private final RetryTemplate retryTemplate;
    
    public DatabaseRetryService() {
        this.retryTemplate = RetryTemplate.builder()
            .maxAttempts(3)
            .exponentialBackoff(1000, 2, 10000)
            .retryOn(SQLException.class, DataAccessException.class)
            .build();
    }
    
    public <T> T executeWithRetry(Supplier<T> operation) {
        return retryTemplate.execute(context -> operation.get());
    }
}
```

### Circuit Breaker Implementation

```java
@Component
public class DatabaseCircuitBreaker {
    
    private final CircuitBreaker circuitBreaker;
    
    public DatabaseCircuitBreaker() {
        this.circuitBreaker = CircuitBreaker.ofDefaults("database");
        circuitBreaker.getEventPublisher()
            .onStateTransition(event -> 
                log.info("Circuit breaker state transition: {}", event));
    }
    
    public <T> T executeWithCircuitBreaker(Supplier<T> operation) {
        return circuitBreaker.executeSupplier(operation);
    }
}
```

## Testing Strategies

### Unit Testing

```java
@ExtendWith(MockitoExtension.class)
class MultiTenantDatabaseSDKTest {
    
    @Mock
    private ConnectionPoolManager connectionPoolManager;
    
    @Mock
    private DatabaseProviderFactory providerFactory;
    
    @InjectMocks
    private MultiTenantDatabaseSDKImpl sdk;
    
    @Test
    void shouldExecuteSqlForCurrentTenant() {
        // Given
        String tenantId = "tenant-123";
        TenantContext.setTenant(tenantId);
        
        DataSource mockDataSource = mock(DataSource.class);
        Connection mockConnection = mock(Connection.class);
        PreparedStatement mockStatement = mock(PreparedStatement.class);
        
        when(connectionPoolManager.getDataSource(tenantId)).thenReturn(mockDataSource);
        when(mockDataSource.getConnection()).thenReturn(mockConnection);
        when(mockConnection.prepareStatement(anyString())).thenReturn(mockStatement);
        
        // When
        sdk.executeSql("INSERT INTO users (name) VALUES (?)", "John");
        
        // Then
        verify(mockStatement).setString(1, "John");
        verify(mockStatement).execute();
    }
}
```

### Integration Testing

```java
@SpringBootTest
@TestPropertySource(properties = {
    "spring.datasource.url=jdbc:h2:mem:testdb",
    "spring.jpa.hibernate.ddl-auto=create-drop"
})
class MultiTenantIntegrationTest {
    
    @Autowired
    private MultiTenantDatabaseSDK sdk;
    
    @Test
    void shouldCreateTenantAndExecuteOperations() {
        // Given
        String tenantId = "test-tenant";
        TenantConfig config = createTestTenantConfig(tenantId);
        
        // When
        sdk.createTenant(tenantId, config);
        
        TenantContext.setTenant(tenantId);
        sdk.executeSql("INSERT INTO users (name, email) VALUES (?, ?)", "John", "john@example.com");
        
        List<User> users = sdk.query("SELECT * FROM users", this::mapUser);
        
        // Then
        assertThat(users).hasSize(1);
        assertThat(users.get(0).getName()).isEqualTo("John");
    }
}
```

## Monitoring and Observability

### Metrics Collection

```java
@Component
public class MultiTenantMetrics {
    
    private final Counter tenantCreationCounter;
    private final Timer databaseOperationTimer;
    private final Gauge activeTenantGauge;
    
    public MultiTenantMetrics(MeterRegistry meterRegistry) {
        this.tenantCreationCounter = Counter.builder("tenant.creation.count")
            .register(meterRegistry);
            
        this.databaseOperationTimer = Timer.builder("database.operation.time")
            .register(meterRegistry);
            
        this.activeTenantGauge = Gauge.builder("tenant.active.count")
            .register(meterRegistry, this, MultiTenantMetrics::getActiveTenantCount);
    }
    
    public void recordTenantCreation() {
        tenantCreationCounter.increment();
    }
    
    public void recordDatabaseOperation(Duration duration) {
        databaseOperationTimer.record(duration);
    }
    
    private double getActiveTenantCount() {
        return connectionPoolManager.getActiveTenantCount();
    }
}
```

### Health Checks

```java
@Component
public class MultiTenantHealthIndicator implements HealthIndicator {
    
    private final ConnectionPoolManager connectionPoolManager;
    
    @Override
    public Health health() {
        try {
            int activeTenants = connectionPoolManager.getActiveTenantCount();
            int totalConnections = connectionPoolManager.getTotalActiveConnections();
            
            return Health.up()
                .withDetail("activeTenants", activeTenants)
                .withDetail("totalConnections", totalConnections)
                .build();
                
        } catch (Exception e) {
            return Health.down()
                .withException(e)
                .build();
        }
    }
}
```

## Deployment and Configuration

### Docker Configuration

```dockerfile
FROM openjdk:11-jre-slim

COPY target/multi-tenant-sdk.jar app.jar

ENV JAVA_OPTS="-Xmx2g -Xms1g"
ENV HIKARI_MAX_POOL_SIZE=20
ENV HIKARI_MIN_IDLE=5

EXPOSE 8080

ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -jar /app.jar"]
```

### Kubernetes Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: multi-tenant-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: multi-tenant-app
  template:
    metadata:
      labels:
        app: multi-tenant-app
    spec:
      containers:
      - name: app
        image: multi-tenant-sdk:latest
        ports:
        - containerPort: 8080
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: "production"
        - name: DATABASE_MAX_POOL_SIZE
          value: "20"
        resources:
          requests:
            memory: "1Gi"
            cpu: "500m"
          limits:
            memory: "2Gi"
            cpu: "1000m"
```

## Common Interview Questions and Answers

**Q: "How do you handle tenant data isolation?"**

A: We implement database-per-tenant isolation using dynamic datasource routing. Each tenant has its own database and connection pool, ensuring complete data isolation. The SDK uses ThreadLocal to maintain tenant context throughout the request lifecycle.

**Q: "What happens if a tenant's database becomes unavailable?"**

A: We implement circuit breaker pattern and retry mechanisms. If a tenant's database is unavailable, the circuit breaker opens, preventing cascading failures. We also have health checks that monitor each tenant's database connectivity.

**Q: "How do you handle database migrations across multiple tenants?"**

A: We use a versioned migration system where each tenant's database schema version is tracked. Migrations are applied tenant by tenant, with rollback capabilities. Critical migrations are tested in staging environments first.

**Q: "How do you optimize connection pool usage?"**

A: We use adaptive connection pool sizing based on tenant activity. Inactive tenants have smaller pools, while active tenants get more connections. We also implement connection

## Advanced Features and Extensions

### Tenant Database Sharding

For high-scale scenarios, the SDK supports database sharding across multiple database servers:

```java
@Component
public class TenantShardingManager {
    
    private final List<ShardConfig> shards;
    private final ConsistentHashing<String> hashRing;
    
    public TenantShardingManager(List<ShardConfig> shards) {
        this.shards = shards;
        this.hashRing = new ConsistentHashing<>(
            shards.stream().map(ShardConfig::getShardId).collect(Collectors.toList())
        );
    }
    
    public ShardConfig getShardForTenant(String tenantId) {
        String shardId = hashRing.getNode(tenantId);
        return shards.stream()
            .filter(shard -> shard.getShardId().equals(shardId))
            .findFirst()
            .orElseThrow(() -> new ShardNotFoundException("Shard not found for tenant: " + tenantId));
    }
    
    public void rebalanceShards() {
        // Implement shard rebalancing logic
        for (ShardConfig shard : shards) {
            int currentLoad = calculateShardLoad(shard);
            if (currentLoad > shard.getMaxCapacity() * 0.8) {
                triggerShardSplit(shard);
            }
        }
    }
}
```

### Tenant Migration and Backup

```java
@Service
public class TenantMigrationService {
    
    private final MultiTenantDatabaseSDK databaseSDK;
    private final TenantBackupService backupService;
    
    public void migrateTenant(String tenantId, TenantConfig newConfig) {
        try {
            // Create backup before migration
            String backupId = backupService.createBackup(tenantId);
            
            // Export tenant data
            TenantData exportedData = exportTenantData(tenantId);
            
            // Create new tenant database
            databaseSDK.createTenant(tenantId + "_new", newConfig);
            
            // Import data to new database
            importTenantData(tenantId + "_new", exportedData);
            
            // Validate migration
            if (validateMigration(tenantId, tenantId + "_new")) {
                // Switch to new database
                switchTenantDatabase(tenantId, newConfig);
                
                // Cleanup old database
                databaseSDK.deleteTenant(tenantId + "_old");
            } else {
                // Rollback
                restoreFromBackup(tenantId, backupId);
            }
            
        } catch (Exception e) {
            throw new TenantMigrationException("Migration failed for tenant: " + tenantId, e);
        }
    }
    
    private TenantData exportTenantData(String tenantId) {
        TenantContext.setTenant(tenantId);
        
        TenantData data = new TenantData();
        
        // Export all tables
        List<String> tables = databaseSDK.query(
            "SELECT table_name FROM information_schema.tables WHERE table_schema = ?",
            rs -> rs.getString("table_name"),
            TenantContext.getCurrentDatabase()
        );
        
        for (String table : tables) {
            List<Map<String, Object>> tableData = databaseSDK.query(
                "SELECT * FROM " + table,
                this::mapRowToMap
            );
            data.addTableData(table, tableData);
        }
        
        return data;
    }
}
```

### Read Replica Support

```java
@Component
public class ReadReplicaManager {
    
    private final Map<String, List<DataSource>> readReplicas = new ConcurrentHashMap<>();
    private final LoadBalancer loadBalancer;
    
    public DataSource getReadDataSource(String tenantId) {
        List<DataSource> replicas = readReplicas.get(tenantId);
        if (replicas == null || replicas.isEmpty()) {
            return connectionPoolManager.getDataSource(tenantId); // Fallback to master
        }
        
        return loadBalancer.selectDataSource(replicas);
    }
    
    public void addReadReplica(String tenantId, TenantConfig replicaConfig) {
        DataSource replicaDataSource = createDataSource(replicaConfig);
        readReplicas.computeIfAbsent(tenantId, k -> new ArrayList<>()).add(replicaDataSource);
    }
    
    @Scheduled(fixedRate = 30000)
    public void monitorReplicaHealth() {
        readReplicas.forEach((tenantId, replicas) -> {
            replicas.removeIf(replica -> !isHealthy(replica));
        });
    }
}
```

### Multi-Database Transaction Support

```java
@Component
public class MultiTenantTransactionManager {
    
    private final PlatformTransactionManager transactionManager;
    
    public void executeMultiTenantTransaction(List<String> tenantIds, 
                                            MultiTenantTransactionCallback callback) {
        
        TransactionStatus status = transactionManager.getTransaction(
            new DefaultTransactionDefinition()
        );
        
        try {
            Map<String, Connection> connections = new HashMap<>();
            
            // Get connections for all tenants
            for (String tenantId : tenantIds) {
                DataSource dataSource = connectionPoolManager.getDataSource(tenantId);
                connections.put(tenantId, dataSource.getConnection());
            }
            
            // Execute callback with all connections
            callback.doInTransaction(connections);
            
            // Commit all transactions
            connections.values().forEach(conn -> {
                try {
                    conn.commit();
                } catch (SQLException e) {
                    throw new RuntimeException(e);
                }
            });
            
            transactionManager.commit(status);
            
        } catch (Exception e) {
            transactionManager.rollback(status);
            throw new MultiTenantTransactionException("Multi-tenant transaction failed", e);
        }
    }
}
```

## Performance Benchmarking and Optimization

### Benchmark Results

```java
@Component
public class PerformanceBenchmark {
    
    private final MultiTenantDatabaseSDK sdk;
    private final MeterRegistry meterRegistry;
    
    @EventListener
    @Async
    public void benchmarkOnStartup(ApplicationReadyEvent event) {
        runConnectionPoolBenchmark();
        runTenantSwitchingBenchmark();
        runConcurrentAccessBenchmark();
    }
    
    private void runConnectionPoolBenchmark() {
        Timer.Sample sample = Timer.start(meterRegistry);
        
        // Test connection acquisition time
        long startTime = System.nanoTime();
        
        for (int i = 0; i < 1000; i++) {
            String tenantId = "tenant-" + (i % 10);
            TenantContext.setTenant(tenantId);
            
            sdk.query("SELECT 1", rs -> rs.getInt(1));
        }
        
        long endTime = System.nanoTime();
        sample.stop(Timer.builder("benchmark.connection.pool").register(meterRegistry));
        
        log.info("Connection pool benchmark: {} ms", (endTime - startTime) / 1_000_000);
    }
    
    private void runTenantSwitchingBenchmark() {
        int iterations = 10000;
        long startTime = System.nanoTime();
        
        for (int i = 0; i < iterations; i++) {
            String tenantId = "tenant-" + (i % 100);
            TenantContext.setTenant(tenantId);
            // Simulate tenant switching overhead
        }
        
        long endTime = System.nanoTime();
        double avgSwitchTime = (endTime - startTime) / 1_000_000.0 / iterations;
        
        log.info("Average tenant switching time: {} ms", avgSwitchTime);
    }
}
```

### Performance Optimization Recommendations

{% mermaid graph LR %}
    A[Performance Optimization] --> B[Connection Pooling]
    A --> C[Caching Strategy]
    A --> D[Query Optimization]
    A --> E[Resource Management]
    
    B --> B1[HikariCP Configuration]
    B --> B2[Pool Size Tuning]
    B --> B3[Connection Validation]
    
    C --> C1[Tenant Config Cache]
    C --> C2[Query Result Cache]
    C --> C3[Schema Cache]
    
    D --> D1[Prepared Statements]
    D --> D2[Batch Operations]
    D --> D3[Index Optimization]
    
    E --> E1[Memory Management]
    E --> E2[Thread Pool Tuning]
    E --> E3[GC Optimization]
{% endmermaid %}

## Security Best Practices

### Security Architecture

{% mermaid graph TB %}
    A[Client Request] --> B[API Gateway]
    B --> C[Authentication Service]
    C --> D[Tenant Authorization]
    D --> E[Multi-Tenant SDK]
    E --> F[Security Validator]
    F --> G[Encrypted Connection]
    G --> H[Tenant Database]
    
    I[Security Layers]
    I --> J[Network Security]
    I --> K[Application Security]
    I --> L[Database Security]
    I --> M[Data Encryption]
{% endmermaid %}

### Advanced Security Features

```java
@Component
public class TenantSecurityEnforcer {
    
    private final TenantPermissionService permissionService;
    private final AuditLogService auditService;
    
    @Around("@annotation(SecureTenantOperation)")
    public Object enforceSecurityz(ProceedingJoinPoint joinPoint) throws Throwable {
        String tenantId = TenantContext.getCurrentTenant();
        String operation = joinPoint.getSignature().getName();
        
        // Validate tenant access
        if (!permissionService.hasPermission(tenantId, operation)) {
            auditService.logUnauthorizedAccess(tenantId, operation);
            throw new TenantAccessDeniedException("Access denied for operation: " + operation);
        }
        
        // Rate limiting
        if (!rateLimiter.tryAcquire(tenantId)) {
            throw new RateLimitExceededException("Rate limit exceeded for tenant: " + tenantId);
        }
        
        try {
            Object result = joinPoint.proceed();
            auditService.logSuccessfulOperation(tenantId, operation);
            return result;
        } catch (Exception e) {
            auditService.logFailedOperation(tenantId, operation, e);
            throw e;
        }
    }
}
```

### Data Masking and Privacy

```java
@Component
public class DataPrivacyManager {
    
    private final Map<String, DataMaskingRule> maskingRules;
    
    public ResultSet maskSensitiveData(ResultSet resultSet, String tenantId) throws SQLException {
        TenantPrivacyConfig config = getPrivacyConfig(tenantId);
        
        while (resultSet.next()) {
            for (DataMaskingRule rule : config.getMaskingRules()) {
                String columnName = rule.getColumnName();
                String originalValue = resultSet.getString(columnName);
                String maskedValue = applyMasking(originalValue, rule.getMaskingType());
                
                // Update result set with masked value
                ((UpdatableResultSet) resultSet).updateString(columnName, maskedValue);
            }
        }
        
        return resultSet;
    }
    
    private String applyMasking(String value, MaskingType type) {
        switch (type) {
            case EMAIL:
                return maskEmail(value);
            case PHONE:
                return maskPhone(value);
            case CREDIT_CARD:
                return maskCreditCard(value);
            default:
                return value;
        }
    }
}
```

## Disaster Recovery and High Availability

### Backup and Recovery Strategy

```java
@Service
public class TenantBackupService {
    
    private final CloudStorageService storageService;
    private final DatabaseProvider databaseProvider;
    
    @Scheduled(cron = "0 0 2 * * *") // Daily at 2 AM
    public void performScheduledBackups() {
        List<String> activeTenants = getActiveTenants();
        
        activeTenants.parallelStream().forEach(tenantId -> {
            try {
                BackupResult result = createBackup(tenantId);
                storageService.uploadBackup(result);
                cleanupOldBackups(tenantId);
            } catch (Exception e) {
                log.error("Backup failed for tenant: {}", tenantId, e);
                alertService.sendBackupFailureAlert(tenantId, e);
            }
        });
    }
    
    public String createBackup(String tenantId) {
        String backupId = generateBackupId(tenantId);
        
        try {
            TenantContext.setTenant(tenantId);
            DataSource dataSource = connectionPoolManager.getDataSource(tenantId);
            
            BackupConfig config = BackupConfig.builder()
                .tenantId(tenantId)
                .backupId(backupId)
                .timestamp(Instant.now())
                .compressionEnabled(true)
                .encryptionEnabled(true)
                .build();
            
            databaseProvider.createBackup(dataSource, config);
            
            return backupId;
            
        } catch (Exception e) {
            throw new BackupException("Backup creation failed for tenant: " + tenantId, e);
        }
    }
    
    public void restoreFromBackup(String tenantId, String backupId) {
        try {
            BackupMetadata metadata = getBackupMetadata(tenantId, backupId);
            InputStream backupStream = storageService.downloadBackup(metadata.getStoragePath());
            
            // Create temporary database for restoration
            String tempTenantId = tenantId + "_restore_" + System.currentTimeMillis();
            databaseProvider.restoreFromBackup(tempTenantId, backupStream);
            
            // Validate restoration
            if (validateRestoration(tenantId, tempTenantId)) {
                // Switch to restored database
                switchTenantDatabase(tenantId, tempTenantId);
            } else {
                throw new RestoreException("Backup validation failed");
            }
            
        } catch (Exception e) {
            throw new RestoreException("Restoration failed for tenant: " + tenantId, e);
        }
    }
}
```

### High Availability Configuration

```java
@Configuration
public class HighAvailabilityConfig {
    
    @Bean
    public LoadBalancer databaseLoadBalancer() {
        return LoadBalancer.builder()
            .algorithm(LoadBalancingAlgorithm.ROUND_ROBIN)
            .healthCheckInterval(Duration.ofSeconds(30))
            .failoverTimeout(Duration.ofSeconds(5))
            .build();
    }
    
    @Bean
    public FailoverManager failoverManager() {
        return new FailoverManager(databaseProviderFactory, alertService);
    }
}

@Component
public class FailoverManager {
    
    private final Map<String, List<TenantConfig>> replicaConfigs = new ConcurrentHashMap<>();
    
    @EventListener
    public void handleDatabaseFailure(DatabaseFailureEvent event) {
        String tenantId = event.getTenantId();
        
        log.warn("Database failure detected for tenant: {}", tenantId);
        
        List<TenantConfig> replicas = replicaConfigs.get(tenantId);
        if (replicas != null && !replicas.isEmpty()) {
            for (TenantConfig replica : replicas) {
                if (isHealthy(replica)) {
                    performFailover(tenantId, replica);
                    break;
                }
            }
        } else {
            alertService.sendCriticalAlert("No healthy replicas available for tenant: " + tenantId);
        }
    }
    
    private void performFailover(String tenantId, TenantConfig replicaConfig) {
        try {
            // Update connection pool to use replica
            connectionPoolManager.updateDataSource(tenantId, replicaConfig);
            
            // Update tenant configuration
            tenantConfigRepository.updateConfig(tenantId, replicaConfig);
            
            log.info("Failover completed for tenant: {}", tenantId);
            alertService.sendFailoverAlert(tenantId, replicaConfig.getHost());
            
        } catch (Exception e) {
            log.error("Failover failed for tenant: {}", tenantId, e);
            alertService.sendFailoverFailureAlert(tenantId, e);
        }
    }
}
```

## External References and Resources

### Documentation and Specifications
- [HikariCP Configuration Guide](https://github.com/brettwooldridge/HikariCP#configuration-knobs-baby)
- [Java SPI Tutorial](https://docs.oracle.com/javase/tutorial/ext/basics/spi.html)
- [Spring Boot Multi-Tenancy](https://spring.io/blog/2022/07/31/how-to-integrate-hibernates-multitenant-feature-with-spring-data-jpa-in-a-spring-boot-application)
- [PostgreSQL Multi-Tenant Patterns](https://www.postgresql.org/docs/current/ddl-schemas.html)
- [MySQL Multi-Tenancy Best Practices](https://dev.mysql.com/doc/refman/8.0/en/multiple-servers.html)

### Performance and Monitoring
- [Micrometer Metrics](https://micrometer.io/docs)
- [Connection Pool Monitoring](https://github.com/brettwooldridge/HikariCP/wiki/MBean-(JMX)-Monitoring-and-Management)
- [Database Performance Tuning](https://use-the-index-luke.com/)

### Security Resources
- [OWASP Database Security](https://owasp.org/www-project-database-security/)
- [Multi-Tenant Security Patterns](https://docs.microsoft.com/en-us/azure/architecture/guide/multitenant/considerations/data-partitioning)
- [SQL Injection Prevention](https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html)

### Cloud and DevOps
- [Kubernetes Database Operators](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/)
- [Docker Multi-Stage Builds](https://docs.docker.com/develop/dev-best-practices/)
- [Terraform Database Provisioning](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/db_instance)

## Conclusion

This Multi-Tenant Database SDK provides a comprehensive solution for managing database operations across multiple tenants in a SaaS environment. The design emphasizes security, performance, and scalability while maintaining simplicity for developers.

Key benefits of this architecture include:

- **Strong tenant isolation** through database-per-tenant approach
- **High performance** via connection pooling and caching strategies
- **Extensibility** through SPI pattern for database providers
- **Production readiness** with monitoring, backup, and failover capabilities
- **Security** with encryption, audit logging, and access controls

The SDK can be extended to support additional database providers, implement more sophisticated sharding strategies, or integrate with cloud-native services. Regular monitoring and performance tuning ensure optimal operation in production environments.

Remember to adapt the configuration and implementation details based on your specific requirements, such as tenant scale, database types, and compliance needs. The provided examples serve as a solid foundation for building a robust multi-tenant database solution.