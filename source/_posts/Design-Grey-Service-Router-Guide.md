---
title: Design Grey Service Router Guide
date: 2025-06-12 22:04:26
tags: [system-design, grey service router]
categories: [system-design]
---

# Grey Service Router System Design

## System Overview

A grey service router system enables gradual service version upgrades across multi-tenant environments by providing intelligent routing, database schema migration management, and administrative controls. This system is crucial for production environments where different tenants may need to operate on different service versions simultaneously during rollout phases.

**Interview Insight**: *When discussing grey deployment strategies, emphasize how this approach reduces blast radius during deployments. Unlike blue-green deployments that switch all traffic at once, grey routing allows granular control over which tenants receive new versions, making it ideal for SaaS platforms with diverse customer requirements.*

## Architecture Components

### Core Services Architecture

{% mermaid graph TB %}
    subgraph "Management Layer"
        UI[GreyServiceManageUI]
        Router[GreyServiceRouter]
    end
    
    subgraph "Data Layer"
        Redis[(Redis Cache)]
        TenantDB1[(Tenant DB 1)]
        TenantDB2[(Tenant DB 2)]
        TenantDBN[(Tenant DB N)]
    end
    
    subgraph "Service Discovery"
        Nacos[Nacos Registry]
        ProjectMgmt[Project Management Service]
    end
    
    subgraph "Service Instances"
        SvcV1[Service v1.0]
        SvcV2[Service v1.1]
        SvcV3[Service v2.0]
    end
    
    UI --> Router
    Router --> Redis
    Router --> Nacos
    Router --> ProjectMgmt
    Router --> TenantDB1
    Router --> TenantDB2
    Router --> TenantDBN
    
    SvcV1 --> TenantDB1
    SvcV2 --> TenantDB2
    SvcV3 --> TenantDBN
{% endmermaid %}

### GreyServiceRouter Service

The router service acts as the central orchestrator for version management and request routing. It maintains the mapping between tenants and their designated service versions while handling database schema upgrades.

**Key Responsibilities:**
- **Version Resolution**: Determines which service version to route requests based on tenant configuration
- **Load Balancing**: Distributes requests across available service instances of the designated version
- **Schema Management**: Orchestrates database schema upgrades for tenant-specific databases
- **Health Monitoring**: Tracks service instance availability and removes unhealthy instances from rotation

**Interview Insight**: *The router pattern here demonstrates the Single Responsibility Principle in distributed systems. By centralizing routing logic, we avoid service mesh complexity while maintaining fine-grained control over traffic distribution.*

### Data Structures in Redis

Efficient data structure design in Redis is critical for performance and consistency:

```lua
-- Tenant to Service Version Mapping
-- Key: tenant:{tenant_id}:services
-- Type: Hash
-- Structure: {service_name: version, service_name: version, ...}
HSET tenant:acme-corp:services user-service "v2.1.0"
HSET tenant:acme-corp:services order-service "v1.5.0"

-- Service Version to Instances Mapping
-- Key: service:{service_name}:{version}:instances
-- Type: Set
-- Structure: {instance_id1, instance_id2, ...}
SADD service:user-service:v2.1.0:instances "192.168.1.10:8080"
SADD service:user-service:v2.1.0:instances "192.168.1.11:8080"

-- Tenant Database Schema Versions
-- Key: tenant:{tenant_id}:db_schema
-- Type: Hash
-- Structure: {service_name: schema_version, ...}
HSET tenant:acme-corp:db_schema user-service "20240315001"
HSET tenant:acme-corp:db_schema order-service "20240310002"

-- Service Health Status
-- Key: service:{service_name}:{version}:health
-- Type: Hash with TTL
-- Structure: {instance_id: last_heartbeat, ...}
HSETEX service:user-service:v2.1.0:health 300 "192.168.1.10:8080" "1710451200"
```

### Lua Script for Atomic Operations

Lua scripts ensure atomic operations for version routing and load balancing:

```lua
-- route_and_balance.lua
local tenant_id = ARGV[1]
local service_name = ARGV[2]

-- Get tenant's designated service version
local version = redis.call('HGET', 'tenant:' .. tenant_id .. ':services', service_name)
if not version then
    return {err = 'No version mapping found for tenant ' .. tenant_id .. ' and service ' .. service_name}
end

-- Get healthy instances for this service version
local instances_key = 'service:' .. service_name .. ':' .. version .. ':instances'
local health_key = 'service:' .. service_name .. ':' .. version .. ':health'

local all_instances = redis.call('SMEMBERS', instances_key)
local healthy_instances = {}

for i, instance in ipairs(all_instances) do
    local health_status = redis.call('HGET', health_key, instance)
    if health_status then
        table.insert(healthy_instances, instance)
    end
end

if #healthy_instances == 0 then
    return {err = 'No healthy instances available for service ' .. service_name .. ' version ' .. version}
end

-- Round-robin load balancing
local counter_key = 'lb_counter:' .. service_name .. ':' .. version
local counter = redis.call('INCR', counter_key)
local selected_index = (counter - 1) % #healthy_instances + 1

return {
    version = version,
    instance = healthy_instances[selected_index],
    total_instances = #healthy_instances
}
```

**Interview Insight**: *Using Lua scripts in Redis demonstrates understanding of atomic operations in distributed systems. This pattern prevents race conditions that could occur with multiple Redis commands, especially important for load balancing counters.*

## GreyServiceManageUI Web Application

### Tenant Management Interface

The management UI provides intuitive controls for administrators to manage tenant service versions and trigger database migrations.

**Core Features:**

**Tenant Overview Dashboard**
- Lists all tenants with their current service version matrix
- Shows migration status and health indicators for each tenant's services
- Provides quick actions for common operations

**Service Version Assignment**
- Drag-and-drop interface for assigning service versions to tenants
- Bulk operations for applying version changes to multiple tenants
- Preview mode showing impact analysis before applying changes

**Database Schema Management**
- Visual representation of schema version compatibility matrix
- One-click schema upgrade triggers with progress tracking
- Rollback capabilities with automatic validation

### UI Flow Example

{% mermaid sequenceDiagram %}
    participant Admin as Administrator
    participant UI as ManageUI
    participant Router as GreyServiceRouter
    participant Redis as Redis Cache
    participant DB as Tenant Database

    Admin->>UI: Select tenant "acme-corp"
    UI->>Router: GET /api/tenants/acme-corp/services
    Router->>Redis: HGETALL tenant:acme-corp:services
    Redis-->>Router: {user-service: "v1.0", order-service: "v1.2"}
    Router-->>UI: Service version mapping
    UI-->>Admin: Display service version table

    Admin->>UI: Update user-service to v2.0
    UI->>Router: PUT /api/tenants/acme-corp/services/user-service
    Router->>Redis: HSET tenant:acme-corp:services user-service "v2.0"
    
    Admin->>UI: Trigger schema upgrade
    UI->>Router: POST /api/tenants/acme-corp/schema-upgrade
    Router->>DB: Execute migration scripts
    DB-->>Router: Migration completed
    Router->>Redis: HSET tenant:acme-corp:db_schema user-service "20240320001"
    Router-->>UI: Upgrade successful
{% endmermaid %}

## Database Schema Upgrade Strategy

### Migration Execution Framework

Database schema upgrades require careful orchestration to maintain data integrity and minimize downtime:

**Pre-Migration Validation**
- **Compatibility Check**: Verify new service version compatibility with current schema
- **Dependency Analysis**: Ensure all dependent services support the target schema version
- **Backup Creation**: Create automated backups before schema modifications

**Migration Execution**
- **Progressive Updates**: Apply schema changes in small, reversible increments
- **Online Schema Changes**: Use techniques like MySQL's pt-online-schema-change for large tables
- **Validation Gates**: Verify data integrity after each migration step

**Post-Migration Verification**
- **Data Consistency Checks**: Run automated tests to verify data integrity
- **Performance Validation**: Ensure query performance meets SLA requirements
- **Service Health Verification**: Confirm all services can connect and operate with new schema

### Schema Version Management

```sql
-- Schema version tracking table
CREATE TABLE schema_versions (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    service_name VARCHAR(100) NOT NULL,
    version VARCHAR(50) NOT NULL,
    applied_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    rollback_script TEXT,
    checksum VARCHAR(64) NOT NULL,
    INDEX idx_service_version (service_name, version)
);

-- Migration execution log
CREATE TABLE migration_logs (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    tenant_id VARCHAR(100) NOT NULL,
    service_name VARCHAR(100) NOT NULL,
    from_version VARCHAR(50),
    to_version VARCHAR(50) NOT NULL,
    status ENUM('pending', 'running', 'completed', 'failed', 'rolled_back'),
    started_at TIMESTAMP,
    completed_at TIMESTAMP,
    error_message TEXT,
    INDEX idx_tenant_service (tenant_id, service_name)
);
```

**Interview Insight**: *Schema versioning demonstrates database evolution patterns. The checksum field prevents accidental re-application of migrations, while rollback scripts enable quick recovery. This approach shows understanding of database DevOps practices.*

## Service Discovery Integration

### Nacos Integration Pattern

Integration with Nacos service registry enables dynamic service discovery and health monitoring:

```java
@Component
public class NacosServiceDiscovery {
    
    @Autowired
    private NamingService namingService;
    
    @Scheduled(fixedDelay = 30000) // 30 seconds
    public void syncServiceInstances() {
        try {
            // Fetch all registered services
            ListView<String> services = namingService.getServicesOfServer(1, Integer.MAX_VALUE);
            
            for (String serviceName : services.getData()) {
                List<Instance> instances = namingService.getAllInstances(serviceName);
                updateRedisServiceInstances(serviceName, instances);
            }
        } catch (Exception e) {
            log.error("Failed to sync service instances from Nacos", e);
        }
    }
    
    private void updateRedisServiceInstances(String serviceName, List<Instance> instances) {
        Map<String, Set<String>> versionInstanceMap = instances.stream()
            .filter(Instance::isHealthy)
            .collect(groupingBy(
                i -> i.getMetadata().getOrDefault("version", "unknown"),
                mapping(i -> i.getIp() + ":" + i.getPort(), toSet())
            ));
            
        versionInstanceMap.forEach((version, instanceSet) -> {
            String redisKey = String.format("service:%s:%s:instances", serviceName, version);
            redisTemplate.delete(redisKey);
            redisTemplate.opsForSet().add(redisKey, instanceSet.toArray(new String[0]));
            redisTemplate.expire(redisKey, Duration.ofMinutes(5));
        });
    }
}
```

### Project Management Service Integration

```java
@Service
public class TenantSyncService {
    
    @Value("${project.management.api.url}")
    private String projectManagementUrl;
    
    @Scheduled(cron = "0 */10 * * * *") // Every 10 minutes
    public void syncTenants() {
        try {
            RestTemplate restTemplate = new RestTemplate();
            ResponseEntity<TenantListResponse> response = restTemplate.getForEntity(
                projectManagementUrl + "/api/tenants", 
                TenantListResponse.class
            );
            
            if (response.getStatusCode().is2xxSuccessful()) {
                updateTenantCache(response.getBody().getTenants());
            }
        } catch (Exception e) {
            log.error("Failed to sync tenants from project management service", e);
        }
    }
    
    private void updateTenantCache(List<Tenant> tenants) {
        String tenantsKey = "system:tenants";
        redisTemplate.delete(tenantsKey);
        
        Map<String, String> tenantMap = tenants.stream()
            .collect(toMap(Tenant::getId, Tenant::getName));
            
        redisTemplate.opsForHash().putAll(tenantsKey, tenantMap);
        redisTemplate.expire(tenantsKey, Duration.ofHours(1));
    }
}
```

## Request Routing Implementation

### API Gateway Integration

The routing logic integrates with API gateways to intercept requests and apply tenant-specific routing:

```java
@Component
public class GreyRouteFilter implements GlobalFilter, Ordered {
    
    @Autowired
    private RedisTemplate<String, String> redisTemplate;
    
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String tenantId = extractTenantId(exchange.getRequest());
        String serviceName = extractServiceName(exchange.getRequest());
        
        if (tenantId != null && serviceName != null) {
            return routeToSpecificVersion(exchange, chain, tenantId, serviceName);
        }
        
        return chain.filter(exchange);
    }
    
    private Mono<Void> routeToSpecificVersion(ServerWebExchange exchange, 
                                             GatewayFilterChain chain,
                                             String tenantId, 
                                             String serviceName) {
        
        // Execute Lua script for atomic routing decision
        DefaultRedisScript<Map> script = new DefaultRedisScript<>();
        script.setScriptText(loadLuaScript("route_and_balance.lua"));
        script.setResultType(Map.class);
        
        Map<String, Object> result = redisTemplate.execute(script, 
            Collections.emptyList(), tenantId, serviceName);
            
        if (result.containsKey("err")) {
            return handleRoutingError(exchange, (String) result.get("err"));
        }
        
        String targetInstance = (String) result.get("instance");
        String version = (String) result.get("version");
        
        // Modify request to target specific instance
        ServerHttpRequest modifiedRequest = exchange.getRequest().mutate()
            .header("X-Target-Instance", targetInstance)
            .header("X-Service-Version", version)
            .build();
            
        return chain.filter(exchange.mutate().request(modifiedRequest).build());
    }
    
    @Override
    public int getOrder() {
        return -100; // Execute before other filters
    }
}
```

## Operational Excellence

### Monitoring and Observability

**Key Metrics to Track:**
- **Routing Success Rate**: Percentage of requests successfully routed to appropriate service versions
- **Version Distribution**: Distribution of tenants across different service versions
- **Migration Success Rate**: Success rate of database schema upgrades
- **Service Health**: Real-time health status of service instances across all versions

**Alerting Strategy:**
- **Service Degradation**: Alert when healthy instance count drops below threshold
- **Migration Failures**: Immediate alerts for failed schema upgrades with rollback procedures
- **Routing Anomalies**: Alerts for unusual routing patterns or high error rates

### Disaster Recovery

**Data Backup Strategy:**
- **Redis Snapshots**: Regular snapshots of routing configuration and tenant mappings
- **Database Backups**: Automated backups before each schema migration
- **Configuration Versioning**: Git-based versioning of routing rules and migration scripts

**Recovery Procedures:**
- **Service Rollback**: Automated rollback to previous service versions in case of critical issues
- **Schema Rollback**: Database rollback procedures with automated validation
- **Cache Recovery**: Rapid reconstruction of Redis cache from authoritative sources

**Interview Insight**: *Disaster recovery planning demonstrates production readiness. The ability to quickly rollback both service versions and database schemas shows understanding of risk mitigation in distributed systems.*

## Performance Considerations

### Caching Strategy

- **Multi-level Caching**: Application-level cache backed by Redis for frequently accessed routing decisions
- **Cache Warming**: Proactive cache population during service discovery updates
- **Cache Invalidation**: Event-driven cache invalidation when tenant configurations change

### Load Balancing Optimization

- **Weighted Round Robin**: Consider instance capacity and current load when routing requests
- **Circuit Breaker Pattern**: Prevent cascade failures by temporarily removing unhealthy instances
- **Request Queuing**: Buffer requests during version transitions to prevent data loss

## Security Considerations

### Access Control

- **Role-based Access**: Different permission levels for viewing vs. modifying tenant configurations
- **Audit Logging**: Comprehensive logging of all configuration changes with user attribution
- **API Authentication**: Strong authentication for all management APIs

### Data Protection

- **Encryption in Transit**: TLS encryption for all service-to-service communication
- **Sensitive Data Masking**: Mask sensitive information in logs and monitoring dashboards
- **Schema Migration Security**: Secure execution environment for database migration scripts

## Implementation Showcase

### Complete API Implementation Example

```java
@RestController
@RequestMapping("/api/tenants")
public class TenantManagementController {
    
    @Autowired
    private GreyServiceRouterService routerService;
    
    @GetMapping("/{tenantId}/services")
    public ResponseEntity<Map<String, String>> getTenantServices(@PathVariable String tenantId) {
        Map<String, String> services = routerService.getTenantServiceVersions(tenantId);
        return ResponseEntity.ok(services);
    }
    
    @PutMapping("/{tenantId}/services/{serviceName}")
    public ResponseEntity<ApiResponse> updateServiceVersion(
            @PathVariable String tenantId,
            @PathVariable String serviceName,
            @RequestBody ServiceVersionUpdateRequest request) {
        
        try {
            routerService.updateTenantServiceVersion(tenantId, serviceName, request.getVersion());
            return ResponseEntity.ok(new ApiResponse("success", "Service version updated"));
        } catch (Exception e) {
            return ResponseEntity.badRequest().body(new ApiResponse("error", e.getMessage()));
        }
    }
    
    @PostMapping("/{tenantId}/schema-upgrade")
    public ResponseEntity<ApiResponse> triggerSchemaUpgrade(
            @PathVariable String tenantId,
            @RequestBody SchemaUpgradeRequest request) {
        
        CompletableFuture<String> upgradeResult = routerService.triggerSchemaUpgrade(
            tenantId, request.getServiceName(), request.getToVersion()
        );
        
        return ResponseEntity.accepted().body(
            new ApiResponse("accepted", "Schema upgrade initiated", upgradeResult.toString())
        );
    }
}
```

This grey service router system provides a robust foundation for managing multi-tenant service versions with database schema evolution. The architecture balances operational flexibility with system reliability, enabling gradual rollouts while maintaining data consistency across tenant databases.

**Interview Insight**: *When presenting this system, emphasize the business value: reduced deployment risk, improved customer experience during upgrades, and operational efficiency through automated schema management. The technical complexity serves the business goal of zero-downtime deployments.*

## External References

- **Redis Lua Scripting**: [Redis Lua Scripts Documentation](https://redis.io/docs/manual/programmability/lua-api/)
- **Nacos Service Discovery**: [Nacos Discovery Configuration](https://nacos.io/en-us/docs/discovery-concepts.html)
- **Circuit Breaker Pattern**: [Resilience4j Documentation](https://resilience4j.readme.io/docs/circuitbreaker)
- **Multi-tenant Architecture**: [AWS Multi-Tenant SaaS Architecture](https://aws.amazon.com/blogs/apn/building-a-multi-tenant-saas-application-with-aws-serverless-services/)
- **Database Schema Evolution**: [Evolutionary Database Design](https://martinfowler.com/articles/evodb.html)

This grey service router system provides a robust foundation for managing multi-tenant service deployments with granular version control and database schema evolution capabilities. The architecture emphasizes scalability, reliability, and operational excellence through comprehensive monitoring and testing strategies.
