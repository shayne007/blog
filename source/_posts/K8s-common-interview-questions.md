---
title: K8s common interview questions
date: 2025-08-03 15:38:17
tags: [k8s]
categories: [k8s]
---

## What is Kubernetes and why would you use it for Java applications?

### Reference Answer

Kubernetes is an open-source container orchestration platform that automates the deployment, scaling, and management of containerized applications. It acts as a distributed operating system for containerized workloads.

### Key Benefits for Java Applications:

- **Microservices Architecture**: Enables independent scaling and deployment of Java services
- **Service Discovery**: Built-in DNS-based service discovery eliminates hardcoded endpoints
- **Load Balancing**: Automatic distribution of traffic across healthy instances
- **Rolling Deployments**: Zero-downtime deployments with gradual traffic shifting
- **Configuration Management**: Externalized configuration through ConfigMaps and Secrets
- **Resource Management**: Optimal JVM performance through resource quotas and limits
- **Self-Healing**: Automatic restart of failed containers and rescheduling on healthy nodes
- **Horizontal Scaling**: Auto-scaling based on CPU, memory, or custom metrics

### Architecture Overview

{% mermaid graph TB %}
    subgraph "Kubernetes Cluster"
        subgraph "Master Node"
            API[API Server]
            ETCD[etcd]
            SCHED[Scheduler]
            CM[Controller Manager]
        end
        
        subgraph "Worker Node 1"
            KUBELET1[Kubelet]
            PROXY1[Kube-proxy]
            subgraph "Pods"
                POD1[Java App Pod 1]
                POD2[Java App Pod 2]
            end
        end
        
        subgraph "Worker Node 2"
            KUBELET2[Kubelet]
            PROXY2[Kube-proxy]
            POD3[Java App Pod 3]
        end
    end
    
    USERS[Users] --> API
    API --> ETCD
    API --> SCHED
    API --> CM
    SCHED --> KUBELET1
    SCHED --> KUBELET2
{% endmermaid %}

---

## Explain the difference between Pods, Services, and Deployments

### Reference Answer

These are fundamental Kubernetes resources that work together to run and expose applications.

### Pod
- **Definition**: Smallest deployable unit containing one or more containers
- **Characteristics**: Shared network and storage, ephemeral, single IP address
- **Java Context**: Typically one JVM per pod for resource isolation

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: java-app-pod
  labels:
    app: java-app
spec:
  containers:
  - name: java-container
    image: openjdk:11-jre-slim
    ports:
    - containerPort: 8080
    resources:
      requests:
        memory: "512Mi"
        cpu: "500m"
      limits:
        memory: "1Gi"
        cpu: "1000m"
```

### Service
- **Definition**: Abstraction that defines access to a logical set of pods
- **Types**: ClusterIP (internal), NodePort (external via node), LoadBalancer (cloud LB)
- **Purpose**: Provides stable networking and load balancing

```yaml
apiVersion: v1
kind: Service
metadata:
  name: java-app-service
spec:
  selector:
    app: java-app
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8080
  type: ClusterIP
```

### Deployment
- **Definition**: Higher-level resource managing ReplicaSets and pod lifecycles
- **Features**: Rolling updates, rollbacks, replica management, declarative updates

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: java-app-deployment
spec:
  replicas: 3
  selector:
    matchLabels:
      app: java-app
  template:
    metadata:
      labels:
        app: java-app
    spec:
      containers:
      - name: java-app
        image: my-java-app:1.0
        ports:
        - containerPort: 8080
```

### Relationship Diagram

{% mermaid graph LR %}
    DEPLOY[Deployment] --> RS[ReplicaSet]
    RS --> POD1[Pod 1]
    RS --> POD2[Pod 2]
    RS --> POD3[Pod 3]
    
    SVC[Service] --> POD1
    SVC --> POD2
    SVC --> POD3
    
    USERS[External Users] --> SVC
{% endmermaid %}

---

## How do you handle configuration management for Java applications in Kubernetes?

### Reference Answer

Configuration management in Kubernetes separates configuration from application code using ConfigMaps and Secrets, following the twelve-factor app methodology.

### ConfigMaps (Non-sensitive data)

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: java-app-config
data:
  application.properties: |
    server.port=8080
    spring.profiles.active=production
    logging.level.com.example=INFO
    database.pool.size=10
  app.env: "production"
  debug.enabled: "false"
```

### Secrets (Sensitive data)

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: java-app-secrets
type: Opaque
data:
  database-username: dXNlcm5hbWU=  # base64 encoded
  database-password: cGFzc3dvcmQ=  # base64 encoded
  api-key: YWJjZGVmZ2hpams=        # base64 encoded
```

### Using Configuration in Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: java-app
spec:
  template:
    spec:
      containers:
      - name: app
        image: my-java-app:1.0
        # Environment variables from ConfigMap
        env:
        - name: SPRING_PROFILES_ACTIVE
          valueFrom:
            configMapKeyRef:
              name: java-app-config
              key: app.env
        - name: DB_USERNAME
          valueFrom:
            secretKeyRef:
              name: java-app-secrets
              key: database-username
        # Mount ConfigMap as volume
        volumeMounts:
        - name: config-volume
          mountPath: /app/config
          readOnly: true
        - name: secret-volume
          mountPath: /app/secrets
          readOnly: true
      volumes:
      - name: config-volume
        configMap:
          name: java-app-config
      - name: secret-volume
        secret:
          secretName: java-app-secrets
```

### Spring Boot Integration

```java
@Configuration
@ConfigurationProperties(prefix = "app")
public class AppConfig {
    private String environment;
    private boolean debugEnabled;
    
    // getters and setters
}

@RestController
public class ConfigController {
    
    @Value("${database.pool.size:5}")
    private int poolSize;
    
    @Autowired
    private AppConfig appConfig;
}
```

### Configuration Hot-Reloading with Spring Cloud Kubernetes

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-kubernetes-config</artifactId>
</dependency>
```

```yaml
spring:
  cloud:
    kubernetes:
      config:
        enabled: true
        sources:
        - name: java-app-config
          namespace: default
      reload:
        enabled: true
        mode: event
        strategy: refresh
```

---

## Describe resource management and JVM tuning in Kubernetes

### Reference Answer

Resource management in Kubernetes involves setting appropriate CPU and memory requests and limits, while JVM tuning ensures optimal performance within container constraints.

### Resource Requests vs Limits

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: java-app
spec:
  template:
    spec:
      containers:
      - name: java-app
        image: my-java-app:1.0
        resources:
          requests:
            memory: "1Gi"      # Guaranteed memory
            cpu: "500m"        # Guaranteed CPU (0.5 cores)
          limits:
            memory: "2Gi"      # Maximum memory
            cpu: "1000m"       # Maximum CPU (1 core)
```

### JVM Container Awareness

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: java-app
spec:
  template:
    spec:
      containers:
      - name: java-app
        image: openjdk:11-jre-slim
        env:
        - name: JAVA_OPTS
          value: >-
            -XX:+UseContainerSupport
            -XX:MaxRAMPercentage=75.0
            -XX:+UseG1GC
            -XX:+UseStringDeduplication
            -XX:+OptimizeStringConcat
            -Djava.security.egd=file:/dev/./urandom
        resources:
          requests:
            memory: "1Gi"
            cpu: "500m"
          limits:
            memory: "2Gi"
            cpu: "1000m"
```

### Memory Calculation Strategy

{% mermaid graph TD %}
    CONTAINER[Container Memory Limit: 2Gi] --> JVM[JVM Heap: ~75% = 1.5Gi]
    CONTAINER --> NONHEAP[Non-Heap: ~20% = 400Mi]
    CONTAINER --> OS[OS/Buffer: ~5% = 100Mi]
    
    JVM --> HEAP_YOUNG[Young Generation]
    JVM --> HEAP_OLD[Old Generation]
    
    NONHEAP --> METASPACE[Metaspace]
    NONHEAP --> CODECACHE[Code Cache]
    NONHEAP --> STACK[Thread Stacks]
{% endmermaid %}

### Advanced JVM Configuration

```dockerfile
FROM openjdk:11-jre-slim

# JVM tuning for containers
ENV JAVA_OPTS="-server \
    -XX:+UseContainerSupport \
    -XX:MaxRAMPercentage=75.0 \
    -XX:InitialRAMPercentage=50.0 \
    -XX:+UseG1GC \
    -XX:MaxGCPauseMillis=200 \
    -XX:+UseStringDeduplication \
    -XX:+OptimizeStringConcat \
    -XX:+UseCompressedOops \
    -XX:+UseCompressedClassPointers \
    -Djava.security.egd=file:/dev/./urandom \
    -Dfile.encoding=UTF-8 \
    -Duser.timezone=UTC"

COPY app.jar /app/app.jar
EXPOSE 8080
ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -jar /app/app.jar"]
```

### Resource Monitoring

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: java-app
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/path: "/actuator/prometheus"
    prometheus.io/port: "8080"
spec:
  containers:
  - name: java-app
    image: my-java-app:1.0
    ports:
    - containerPort: 8080
      name: http
```

### Vertical Pod Autoscaler (VPA) Configuration

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: java-app-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: java-app
  updatePolicy:
    updateMode: "Auto"
  resourcePolicy:
    containerPolicies:
    - containerName: java-app
      maxAllowed:
        memory: 4Gi
        cpu: 2000m
      minAllowed:
        memory: 512Mi
        cpu: 250m
```

---

## How do you implement health checks for Java applications?

### Reference Answer

Health checks in Kubernetes use three types of probes to ensure application reliability and proper traffic routing.

### Probe Types

{% mermaid graph LR %}
    subgraph "Pod Lifecycle"
        START[Pod Start] --> STARTUP{Startup Probe}
        STARTUP -->|Pass| READY{Readiness Probe}
        STARTUP -->|Fail| RESTART[Restart Container]
        READY -->|Pass| TRAFFIC[Receive Traffic]
        READY -->|Fail| NO_TRAFFIC[No Traffic]
        TRAFFIC --> LIVE{Liveness Probe}
        LIVE -->|Pass| TRAFFIC
        LIVE -->|Fail| RESTART
    end
{% endmermaid %}

### Spring Boot Actuator Health Endpoints

```java
@RestController
public class HealthController {
    
    @Autowired
    private DataSource dataSource;
    
    @GetMapping("/health/live")
    public ResponseEntity<Map<String, String>> liveness() {
        Map<String, String> status = new HashMap<>();
        status.put("status", "UP");
        status.put("timestamp", Instant.now().toString());
        return ResponseEntity.ok(status);
    }
    
    @GetMapping("/health/ready")
    public ResponseEntity<Map<String, Object>> readiness() {
        Map<String, Object> health = new HashMap<>();
        health.put("status", "UP");
        
        // Check database connectivity
        try {
            dataSource.getConnection().close();
            health.put("database", "UP");
        } catch (Exception e) {
            health.put("database", "DOWN");
            health.put("status", "DOWN");
            return ResponseEntity.status(503).body(health);
        }
        
        // Check external dependencies
        health.put("externalAPI", checkExternalAPI());
        
        return ResponseEntity.ok(health);
    }
    
    private String checkExternalAPI() {
        // Implementation to check external dependencies
        return "UP";
    }
}
```

### Kubernetes Deployment with Health Checks

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: java-app
spec:
  template:
    spec:
      containers:
      - name: java-app
        image: my-java-app:1.0
        ports:
        - containerPort: 8080
        
        # Startup probe for slow-starting applications
        startupProbe:
          httpGet:
            path: /health/live
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 30  # 30 * 5 = 150s max startup time
          
        # Liveness probe
        livenessProbe:
          httpGet:
            path: /health/live
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3   # Restart after 3 consecutive failures
          
        # Readiness probe
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 5
          timeoutSeconds: 3
          successThreshold: 1
          failureThreshold: 3   # Remove from service after 3 failures
```

### Custom Health Indicators

```java
@Component
public class DatabaseHealthIndicator implements HealthIndicator {
    
    @Autowired
    private DataSource dataSource;
    
    @Override
    public Health health() {
        try {
            Connection connection = dataSource.getConnection();
            // Perform a simple query
            PreparedStatement statement = connection.prepareStatement("SELECT 1");
            ResultSet resultSet = statement.executeQuery();
            
            if (resultSet.next()) {
                return Health.up()
                    .withDetail("database", "Available")
                    .withDetail("connectionPool", getConnectionPoolInfo())
                    .build();
            }
        } catch (SQLException e) {
            return Health.down()
                .withDetail("database", "Unavailable")
                .withDetail("error", e.getMessage())
                .build();
        }
        
        return Health.down().build();
    }
    
    private Map<String, Object> getConnectionPoolInfo() {
        // Return connection pool metrics
        Map<String, Object> poolInfo = new HashMap<>();
        poolInfo.put("active", 5);
        poolInfo.put("idle", 3);
        poolInfo.put("max", 10);
        return poolInfo;
    }
}
```

### Application Properties Configuration

```properties
# Health endpoint configuration
management.endpoints.web.exposure.include=health,info,metrics,prometheus
management.endpoint.health.show-details=always
management.endpoint.health.show-components=always
management.health.db.enabled=true
management.health.diskspace.enabled=true

# Custom health check paths
management.server.port=8081
management.endpoints.web.base-path=/actuator
```

### TCP and Command Probes

```yaml
# TCP probe example
livenessProbe:
  tcpSocket:
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10

# Command probe example
livenessProbe:
  exec:
    command:
    - /bin/sh
    - -c
    - "ps aux | grep java"
  initialDelaySeconds: 30
  periodSeconds: 10
```

---

## Explain how to handle persistent data in Java applications on Kubernetes

### Reference Answer

Persistent data in Kubernetes requires understanding storage abstractions and choosing appropriate patterns based on application requirements.

### Storage Architecture

{% mermaid graph TB %}
    subgraph "Storage Layer"
        SC[StorageClass] --> PV[PersistentVolume]
        PVC[PersistentVolumeClaim] --> PV
    end
    
    subgraph "Application Layer"
        POD[Pod] --> PVC
        STATEFULSET[StatefulSet] --> PVC
        DEPLOYMENT[Deployment] --> PVC
    end
    
    subgraph "Physical Storage"
        PV --> DISK[Physical Disk]
        PV --> NFS[NFS Server]
        PV --> CLOUD[Cloud Storage]
    end
{% endmermaid %}

### StorageClass Definition

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp3
  fsType: ext4
  encrypted: "true"
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
```

### PersistentVolumeClaim

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: java-app-storage
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: fast-ssd
  resources:
    requests:
      storage: 10Gi
```

### Java Application with File Storage

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: java-file-processor
spec:
  replicas: 1  # Single replica for ReadWriteOnce
  template:
    spec:
      containers:
      - name: app
        image: my-java-app:1.0
        volumeMounts:
        - name: app-storage
          mountPath: /app/data
        - name: logs-storage
          mountPath: /app/logs
        env:
        - name: DATA_DIR
          value: "/app/data"
        - name: LOG_DIR
          value: "/app/logs"
      volumes:
      - name: app-storage
        persistentVolumeClaim:
          claimName: java-app-storage
      - name: logs-storage
        persistentVolumeClaim:
          claimName: java-app-logs
```

### StatefulSet for Database Applications

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: java-database-app
spec:
  serviceName: "java-db-service"
  replicas: 3
  template:
    spec:
      containers:
      - name: app
        image: my-java-db-app:1.0
        ports:
        - containerPort: 8080
        volumeMounts:
        - name: data-volume
          mountPath: /app/data
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
  volumeClaimTemplates:
  - metadata:
      name: data-volume
    spec:
      accessModes: ["ReadWriteOnce"]
      storageClassName: "fast-ssd"
      resources:
        requests:
          storage: 20Gi
```

### Java File Processing Example

```java
@Service
public class FileProcessingService {
    
    @Value("${app.data.dir:/app/data}")
    private String dataDirectory;
    
    @PostConstruct
    public void init() {
        // Ensure data directory exists
        Path dataPath = Paths.get(dataDirectory);
        if (!Files.exists(dataPath)) {
            try {
                Files.createDirectories(dataPath);
                logger.info("Created data directory: {}", dataPath);
            } catch (IOException e) {
                logger.error("Failed to create data directory", e);
            }
        }
    }
    
    public void processFile(MultipartFile uploadedFile) {
        try {
            String filename = UUID.randomUUID().toString() + "_" + uploadedFile.getOriginalFilename();
            Path filePath = Paths.get(dataDirectory, filename);
            
            // Save uploaded file
            uploadedFile.transferTo(filePath.toFile());
            
            // Process file
            processFileContent(filePath);
            
            // Move to processed directory
            Path processedDir = Paths.get(dataDirectory, "processed");
            Files.createDirectories(processedDir);
            Files.move(filePath, processedDir.resolve(filename));
            
        } catch (IOException e) {
            logger.error("Error processing file", e);
            throw new FileProcessingException("Failed to process file", e);
        }
    }
    
    private void processFileContent(Path filePath) {
        // File processing logic
        try (BufferedReader reader = Files.newBufferedReader(filePath)) {
            reader.lines()
                .filter(line -> !line.trim().isEmpty())
                .forEach(this::processLine);
        } catch (IOException e) {
            logger.error("Error reading file: " + filePath, e);
        }
    }
}
```

### Backup Strategy with CronJob

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: data-backup
spec:
  schedule: "0 2 * * *"  # Daily at 2 AM
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup
            image: alpine:latest
            command:
            - /bin/sh
            - -c
            - |
              apk add --no-cache tar gzip
              DATE=$(date +%Y%m%d_%H%M%S)
              tar -czf /backup/data_backup_$DATE.tar.gz -C /app/data .
              # Keep only last 7 backups
              find /backup -name "data_backup_*.tar.gz" -mtime +7 -delete
            volumeMounts:
            - name: app-data
              mountPath: /app/data
              readOnly: true
            - name: backup-storage
              mountPath: /backup
          volumes:
          - name: app-data
            persistentVolumeClaim:
              claimName: java-app-storage
          - name: backup-storage
            persistentVolumeClaim:
              claimName: backup-storage
          restartPolicy: OnFailure
```

### Database Connection with Persistent Storage

```java
@Configuration
public class DatabaseConfig {
    
    @Value("${spring.datasource.url}")
    private String databaseUrl;
    
    @Bean
    @Primary
    public DataSource dataSource() {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl(databaseUrl);
        config.setUsername("${DB_USERNAME}");
        config.setPassword("${DB_PASSWORD}");
        config.setMaximumPoolSize(20);
        config.setMinimumIdle(5);
        config.setConnectionTimeout(30000);
        config.setIdleTimeout(600000);
        config.setMaxLifetime(1800000);
        
        return new HikariDataSource(config);
    }
    
    @Bean
    public PlatformTransactionManager transactionManager() {
        return new DataSourceTransactionManager(dataSource());
    }
}
```

---

## How do you implement service discovery and communication between Java microservices?

### Reference Answer

Kubernetes provides built-in service discovery through DNS, while Java applications can leverage Spring Cloud Kubernetes for enhanced integration.

### Service Discovery Architecture

{% mermaid graph TB %}
    subgraph "Kubernetes Cluster"
        subgraph "Namespace: default"
            SVC1[user-service]
            SVC2[order-service]
            SVC3[payment-service]
            
            POD1[User Service Pods]
            POD2[Order Service Pods]
            POD3[Payment Service Pods]
            
            SVC1 --> POD1
            SVC2 --> POD2
            SVC3 --> POD3
        end
        
        DNS[CoreDNS] --> SVC1
        DNS --> SVC2
        DNS --> SVC3
    end
    
    POD2 -->|user-service.default.svc.cluster.local| POD1
    POD2 -->|payment-service.default.svc.cluster.local| POD3
{% endmermaid %}

### Service Definitions

```yaml
# User Service
apiVersion: v1
kind: Service
metadata:
  name: user-service
  labels:
    app: user-service
spec:
  selector:
    app: user-service
  ports:
  - name: http
    port: 80
    targetPort: 8080
  type: ClusterIP

---
# Order Service  
apiVersion: v1
kind: Service
metadata:
  name: order-service
  labels:
    app: order-service
spec:
  selector:
    app: order-service
  ports:
  - name: http
    port: 80
    targetPort: 8080
  type: ClusterIP

---
# Payment Service
apiVersion: v1
kind: Service
metadata:
  name: payment-service
  labels:
    app: payment-service
spec:
  selector:
    app: payment-service
  ports:
  - name: http
    port: 80
    targetPort: 8080
  type: ClusterIP
```

### Spring Cloud Kubernetes Configuration

```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-kubernetes-client</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.cloud</groupId>
        <artifactId>spring-cloud-starter-loadbalancer</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-webflux</artifactId>
    </dependency>
</dependencies>
```

### Service Discovery Configuration

```java
@Configuration
@EnableDiscoveryClient
public class ServiceDiscoveryConfig {
    
    @Bean
    @LoadBalanced
    public WebClient.Builder webClientBuilder() {
        return WebClient.builder();
    }
    
    @Bean
    public WebClient webClient(WebClient.Builder builder) {
        return builder
            .codecs(configurer -> configurer.defaultCodecs().maxInMemorySize(1024 * 1024))
            .build();
    }
}
```

### Inter-Service Communication

```java
@Service
public class OrderService {
    
    private final WebClient webClient;
    
    public OrderService(WebClient webClient) {
        this.webClient = webClient;
    }
    
    public Mono<UserDto> getUserDetails(String userId) {
        return webClient
            .get()
            .uri("http://user-service/api/users/{userId}", userId)
            .retrieve()
            .onStatus(HttpStatus::isError, response -> {
                return Mono.error(new ServiceException("User service error: " + response.statusCode()));
            })
            .bodyToMono(UserDto.class)
            .timeout(Duration.ofSeconds(5))
            .retry(3);
    }
    
    public Mono<PaymentResponse> processPayment(PaymentRequest request) {
        return webClient
            .post()
            .uri("http://payment-service/api/payments")
            .body(Mono.just(request), PaymentRequest.class)
            .retrieve()
            .bodyToMono(PaymentResponse.class)
            .timeout(Duration.ofSeconds(10));
    }
    
    @Transactional
    public Mono<OrderDto> createOrder(CreateOrderRequest request) {
        return getUserDetails(request.getUserId())
            .flatMap(user -> {
                Order order = new Order();
                order.setUserId(user.getId());
                order.setAmount(request.getAmount());
                order.setStatus(OrderStatus.PENDING);
                
                return Mono.fromCallable(() -> orderRepository.save(order));
            })
            .flatMap(order -> {
                PaymentRequest paymentRequest = new PaymentRequest();
                paymentRequest.setOrderId(order.getId());
                paymentRequest.setAmount(order.getAmount());
                
                return processPayment(paymentRequest)
                    .map(paymentResponse -> {
                        order.setStatus(paymentResponse.isSuccessful() ? 
                            OrderStatus.CONFIRMED : OrderStatus.FAILED);
                        return orderRepository.save(order);
                    });
            })
            .map(this::toOrderDto);
    }
}
```

### Circuit Breaker with Resilience4j

```java
@Component
public class PaymentServiceClient {
    
    private final WebClient webClient;
    private final CircuitBreaker circuitBreaker;
    
    public PaymentServiceClient(WebClient webClient) {
        this.webClient = webClient;
        this.circuitBreaker = CircuitBreaker.ofDefaults("payment-service");
    }
    
    public Mono<PaymentResponse> processPayment(PaymentRequest request) {
        Supplier<Mono<PaymentResponse>> decoratedSupplier = CircuitBreaker
            .decorateSupplier(circuitBreaker, () -> {
                return webClient
                    .post()
                    .uri("http://payment-service/api/payments")
                    .body(Mono.just(request), PaymentRequest.class)
                    .retrieve()
                    .bodyToMono(PaymentResponse.class)
                    .timeout(Duration.ofSeconds(5));
            });
            
        return decoratedSupplier.get()
            .onErrorResume(CallNotPermittedException.class, ex -> {
                // Circuit breaker is open
                return Mono.just(PaymentResponse.failed("Service temporarily unavailable"));
            })
            .onErrorResume(TimeoutException.class, ex -> {
                return Mono.just(PaymentResponse.failed("Payment service timeout"));
            });
    }
}
```

### Application Properties

```yaml
spring:
  cloud:
    kubernetes:
      discovery:
        enabled: true
        all-namespaces: false
        wait-cache-ready: true
      client:
        namespace: default
    loadbalancer:
      ribbon:
        enabled: false

resilience4j:
  circuitbreaker:
    instances:
      payment-service:
        registerHealthIndicator: true
        slidingWindowSize: 10
        minimumNumberOfCalls: 3
        failureRateThreshold: 50
        waitDurationInOpenState: 10s
        permittedNumberOfCallsInHalfOpenState: 2
  retry:
    instances:
      payment-service:
        maxAttempts: 3
        waitDuration: 1s
        exponentialBackoffMultiplier: 2
  timelimiter:
    instances:
      payment-service:
        timeoutDuration: 5s
```

### Service Mesh Integration (Istio)

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: payment-service
spec:
  hosts:
  - payment-service
  http:
  - match:
    - uri:
        prefix: /api/payments
    route:
    - destination:
        host: payment-service
        port:
          number: 80
    timeout: 10s
    retries:
      attempts: 3
      perTryTimeout: 3s
      retryOn: 5xx,reset,connect-failure,refused-stream

---
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: payment-service
spec:
  host: payment-service
  trafficPolicy:
    loadBalancer:
      simple: LEAST_CONN
    connectionPool:
      tcp:
        maxConnections: 10
      http:
        http1MaxPendingRequests: 10
        maxRequestsPerConnection: 2
    circuitBreaker:
      consecutiveErrors: 3
      interval: 30s
      baseEjectionTime: 30s
      maxEjectionPercent: 50
```

---

## Describe different deployment strategies in Kubernetes

### Reference Answer

Kubernetes supports various deployment strategies to minimize downtime and reduce risk during application updates.

### Deployment Strategies Overview

{% mermaid graph TB %}
    subgraph "Rolling Update"
        RU1[v1 Pod] --> RU2[v1 Pod]
        RU2 --> RU3[v1 Pod]
        RU4[v2 Pod] --> RU5[v2 Pod]
        RU5 --> RU6[v2 Pod]
    end
    
    subgraph "Blue-Green"
        BG1[Blue Environment<br/>v1 Pods] 
        BG2[Green Environment<br/>v2 Pods]
        LB[Load Balancer] --> BG1
        LB -.-> BG2
    end
    
    subgraph "Canary"
        C1[v1 Pods - 90%]
        C2[v2 Pods - 10%]
        CLB[Load Balancer] --> C1
        CLB --> C2
    end
{% endmermaid %}

### Rolling Update (Default Strategy)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: java-app-rolling
spec:
  replicas: 6
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1      # Max pods that can be unavailable
      maxSurge: 2           # Max pods that can be created above desired replica count
  template:
    spec:
      containers:
      - name: java-app
        image: my-java-app:v2
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 5
        livenessProbe:
          httpGet:
            path: /health/live
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 10
```

### Blue-Green Deployment

```yaml
# Blue deployment (current version)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: java-app-blue
  labels:
    version: blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: java-app
      version: blue
  template:
    metadata:
      labels:
        app: java-app
        version: blue
    spec:
      containers:
      - name: java-app
        image: my-java-app:v1

---
# Green deployment (new version)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: java-app-green
  labels:
    version: green
spec:
  replicas: 3
  selector:
    matchLabels:
      app: java-app
      version: green
  template:
    metadata:
      labels:
        app: java-app
        version: green
    spec:
      containers:
      - name: java-app
        image: my-java-app:v2

---
# Service pointing to blue (active) version
apiVersion: v1
kind: Service
metadata:
  name: java-app-service
spec:
  selector:
    app: java-app
    version: blue  # Switch to 'green' when ready
  ports:
  - port: 80
    targetPort: 8080
```

### Canary Deployment with Istio

```yaml
# Primary deployment (90% traffic)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: java-app-primary
spec:
  replicas: 9
  selector:
    matchLabels:
      app: java-app
      version: primary
  template:
    metadata:
      labels:
        app: java-app
        version: primary
    spec:
      containers:
      - name: java-app
        image: my-java-app:v1

---
# Canary deployment (10% traffic)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: java-app-canary
spec:
  replicas: 1
  selector:
    matchLabels:
      app: java-app
      version: canary
  template:
    metadata:
      labels:
        app: java-app
        version: canary
    spec:
      containers:
      - name: java-app
        image: my-java-app:v2

---
# VirtualService for traffic splitting
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: java-app
spec:
  hosts:
  - java-app
  http:
  - match:
    - headers:
        canary:
          exact: "true"
    route:
    - destination:
        host: java-app
        subset: canary
  - route:
    - destination:
        host: java-app
        subset: primary
      weight: 90
    - destination:
        host: java-app
        subset: canary
      weight: 10

---
# DestinationRule defining subsets
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: java-app
spec:
  host: java-app
  subsets:
  - name: primary
    labels:
      version: primary
  - name: canary
    labels:
      version: canary
```

### Recreate Strategy

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: java-app-recreate
spec:
  replicas: 3
  strategy:
    type: Recreate  # Terminates all old pods before creating new ones
  template:
    spec:
      containers:
      - name: java-app
        image: my-java-app:v2
```

### Deployment Automation Script

```bash
#!/bin/bash

DEPLOYMENT_NAME="java-app"
NEW_IMAGE="my-java-app:v2"
NAMESPACE="default"

# Rolling update
kubectl set image deployment/$DEPLOYMENT_NAME java-app=$NEW_IMAGE -n $NAMESPACE

# Wait for rollout to complete
kubectl rollout status deployment/$DEPLOYMENT_NAME -n $NAMESPACE

# Check if rollout was successful
if [ $? -eq 0 ]; then
    echo "Deployment successful"
    
    # Optional: Run smoke tests
    kubectl run smoke-test --rm -i --restart=Never --image=curlimages/curl -- \
        curl -f http://java-app-service/health/ready
    
    if [ $? -eq 0 ]; then
        echo "Smoke tests passed"
    else
        echo "Smoke tests failed, rolling back"
        kubectl rollout undo deployment/$DEPLOYMENT_NAME -n $NAMESPACE
    fi
else
    echo "Deployment failed, rolling back"
    kubectl rollout undo deployment/$DEPLOYMENT_NAME -n $NAMESPACE
fi
```

### Spring Boot Graceful Shutdown

```java
@Component
public class GracefulShutdownHook {
    
    private static final Logger logger = LoggerFactory.getLogger(GracefulShutdownHook.class);
    
    @EventListener
    public void onApplicationEvent(ContextClosedEvent event) {
        logger.info("Received shutdown signal, starting graceful shutdown...");
        
        // Allow ongoing requests to complete
        try {
            Thread.sleep(5000); // Grace period
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
        
        logger.info("Graceful shutdown completed");
    }
}
```

```properties
# application.properties
server.shutdown=graceful
spring.lifecycle.timeout-per-shutdown-phase=30s
```

---

## How do you handle logging and monitoring for Java applications in Kubernetes?

### Reference Answer

Comprehensive logging and monitoring in Kubernetes requires centralized log aggregation, metrics collection, and distributed tracing.

### Logging Architecture

{% mermaid graph TB %}
    subgraph "Kubernetes Cluster"
        subgraph "Application Pods"
            APP1[Java App 1] --> LOGS1[stdout/stderr]
            APP2[Java App 2] --> LOGS2[stdout/stderr]
            APP3[Java App 3] --> LOGS3[stdout/stderr]
        end
        
        subgraph "Log Collection"
            FLUENTD[Fluentd DaemonSet]
            LOGS1 --> FLUENTD
            LOGS2 --> FLUENTD
            LOGS3 --> FLUENTD
        end
        
        subgraph "Monitoring"
            PROMETHEUS[Prometheus]
            APP1 --> PROMETHEUS
            APP2 --> PROMETHEUS
            APP3 --> PROMETHEUS
        end
    end
    
    subgraph "External Systems"
        FLUENTD --> ELASTICSEARCH[Elasticsearch]
        ELASTICSEARCH --> KIBANA[Kibana]
        PROMETHEUS --> GRAFANA[Grafana]
    end
{% endmermaid %}

### Structured Logging Configuration

```xml
<!-- logback-spring.xml -->
<configuration>
    <springProfile name="!local">
        <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
            <encoder class="net.logstash.logback.encoder.LoggingEventCompositeJsonEncoder">
                <providers>
                    <timestamp/>
                    <logLevel/>
                    <loggerName/>
                    <message/>
                    <mdc/>
                    <arguments/>
                    <stackTrace/>
                    <pattern>
                        <pattern>
                            {
                                "service": "java-app",
                                "version": "${APP_VERSION:-unknown}",
                                "pod": "${HOSTNAME:-unknown}",
                                "namespace": "${POD_NAMESPACE:-default}"
                            }
                        </pattern>
                    </pattern>
                </providers>
            </encoder>
        </appender>
    </springProfile>
    
    <springProfile name="local">
        <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
            <encoder>
                <pattern>%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>
            </encoder>
        </appender>
    </springProfile>
    
    <root level="INFO">
        <appender-ref ref="STDOUT"/>
    </root>
    
    <logger name="com.example" level="DEBUG"/>
    <logger name="org.springframework.web" level="DEBUG"/>
</configuration>
```

### Application Logging Code

```java
@RestController
@Slf4j
public class OrderController {
    
    @Autowired
    private OrderService orderService;
    
    @PostMapping("/orders")
    public ResponseEntity<OrderDto> createOrder(@RequestBody CreateOrderRequest request) {
        String correlationId = UUID.randomUUID().toString();
        
        // Add correlation ID to MDC for request tracing
        MDC.put("correlationId", correlationId);
        MDC.put("operation", "createOrder");
        MDC.put("userId", request.getUserId());
        
        try {
            log.info("Creating order for user: {}", request.getUserId());
            
            OrderDto order = orderService.createOrder(request);
            
            log.info("Order created successfully: orderId={}, amount={}", 
                order.getId(), order.getAmount());
            
            return ResponseEntity.ok(order);
            
        } catch (Exception e) {
            log.error("Failed to create order: {}", e.getMessage(), e);
            throw e;
        } finally {
            MDC.clear();
        }
    }
}
```

### Fluentd Configuration

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluentd-config
data:
  fluent.conf: |
    <source>
      @type tail
      path /var/log/containers/*java-app*.log
      pos_file /var/log/fluentd-containers.log.pos
      tag kubernetes.*
      format json
      read_from_head true
    </source>
    
    <filter kubernetes.**>
      @type kubernetes_metadata
    </filter>
    
    <filter kubernetes.**>
      @type parser
      key_name log
      reserve_data true
      <parse>
        @type json
      </parse>
    </filter>
    
    <match kubernetes.**>
      @type elasticsearch
      host elasticsearch.logging.svc.cluster.local
      port 9200
      index_name kubernetes
      type_name _doc
      <buffer>
        @type file
        path /var/log/fluentd-buffers/kubernetes.system.buffer
        flush_mode interval
        retry_type exponential_backoff
        flush_thread_count 2
        flush_interval 5s
        retry_forever
        retry_max_interval 30
        chunk_limit_size 2M
        queue_limit_length 8
        overflow_action block
      </buffer>
    </match>

---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd
  namespace: kube-system
spec:
  selector:
    matchLabels:
      name: fluentd
  template:
    metadata:
      labels:
        name: fluentd
    spec:
      containers:
      - name: fluentd
        image: fluent/fluentd-kubernetes-daemonset:v1.14-debian-elasticsearch7-1
        volumeMounts:
        - name: varlog
          mountPath: /var/log
        - name: varlibdockercontainers
          mountPath: /var/lib/docker/containers
          readOnly: true
        - name: config-volume
          mountPath: /fluentd/etc
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
      - name: varlibdockercontainers
        hostPath:
          path: /var/lib/docker/containers
      - name: config-volume
        configMap:
          name: fluentd-config
```

### Prometheus Metrics with Micrometer

```java
@Component
public class MetricsConfig {
    
    @Bean
    public MeterRegistryCustomizer<MeterRegistry> metricsCommonTags() {
        return registry -> registry.config().commonTags(
            "application", "java-app",
            "version", System.getProperty("app.version", "unknown")
        );
    }
    
    @Bean
    public TimedAspect timedAspect(MeterRegistry registry) {
        return new TimedAspect(registry);
    }
}

@Service
@Slf4j
public class OrderService {
    
    private final Counter orderCreatedCounter;
    private final Timer orderProcessingTimer;
    private final Gauge activeOrdersGauge;
    
    public OrderService(MeterRegistry meterRegistry) {
        this.orderCreatedCounter = Counter.builder("orders.created")
            .description("Number of orders created")
            .register(meterRegistry);
            
        this.orderProcessingTimer = Timer.builder("orders.processing.time")
            .description("Order processing time")
            .register(meterRegistry);
            
        this.activeOrdersGauge = Gauge.builder("orders.active")
            .description("Number of active orders")
            .register(meterRegistry, this, OrderService::getActiveOrderCount);
    }
    
    @Timed(name = "orders.create", description = "Create order operation")
    public OrderDto createOrder(CreateOrderRequest request) {
        return Timer.Sample.start(orderProcessingTimer)
            .stop(() -> {
                try {
                    // Order creation logic
                    OrderDto order = processOrder(request);
                    orderCreatedCounter.increment(Tags.of("status", "success"));
                    return order;
                } catch (Exception e) {
                    orderCreatedCounter.increment(Tags.of("status", "error"));
                    throw e;
                }
            });
    }
    
    public double getActiveOrderCount() {
        // Return current active order count
        return orderRepository.countByStatus(OrderStatus.ACTIVE);
    }
}
```

### Deployment with Monitoring Annotations

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: java-app
spec:
  template:
    metadata:
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/path: "/actuator/prometheus"
        prometheus.io/port: "8080"
      labels:
        app: java-app
    spec:
      containers:
      - name: java-app
        image: my-java-app:1.0
        ports:
        - containerPort: 8080
          name: http
        env:
        - name: HOSTNAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: APP_VERSION
          value: "1.0"
```

### Distributed Tracing with Jaeger

```xml
<dependency>
    <groupId>io.opentracing.contrib</groupId>
    <artifactId>opentracing-spring-jaeger-cloud-starter</artifactId>
</dependency>
```

```java
@RestController
public class OrderController {
    
    @Autowired
    private Tracer tracer;
    
    @PostMapping("/orders")
    public ResponseEntity<OrderDto> createOrder(@RequestBody CreateOrderRequest request) {
        Span span = tracer.nextSpan()
            .name("create-order")
            .tag("user.id", request.getUserId())
            .tag("order.amount", String.valueOf(request.getAmount()))
            .start();
            
        try (Tracer.SpanInScope ws = tracer.withSpanInScope(span)) {
            OrderDto order = orderService.createOrder(request);
            span.tag("order.id", order.getId());
            return ResponseEntity.ok(order);
        } catch (Exception e) {
            span.tag("error", true);
            span.tag("error.message", e.getMessage());
            throw e;
        } finally {
            span.end();
        }
    }
}
```

### Application Properties for Monitoring

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus,loggers
  endpoint:
    health:
      show-details: always
    metrics:
      enabled: true
  metrics:
    export:
      prometheus:
        enabled: true
    distribution:
      percentiles-histogram:
        http.server.requests: true
      percentiles:
        http.server.requests: 0.5, 0.95, 0.99
      sla:
        http.server.requests: 100ms, 200ms, 500ms

opentracing:
  jaeger:
    service-name: java-app
    sampler:
      type: const
      param: 1
    log-spans: true

logging:
  level:
    io.jaeger: INFO
    io.opentracing: INFO
```

---

## What are the security considerations for Java applications in Kubernetes?

### Reference Answer

Security in Kubernetes requires a multi-layered approach covering container security, network policies, RBAC, and secure configuration management.

### Security Architecture

{% mermaid graph TB %}
    subgraph "Cluster Security"
        RBAC[RBAC] --> API[API Server]
        PSP[Pod Security Standards] --> PODS[Pods]
        NP[Network Policies] --> NETWORK[Pod Network]
    end
    
    subgraph "Pod Security"
        SECCTX[Security Context] --> CONTAINER[Container]
        SECRETS[Secrets] --> CONTAINER
        SA[Service Account] --> CONTAINER
    end
    
    subgraph "Image Security"
        SCAN[Image Scanning] --> REGISTRY[Container Registry]
        SIGN[Image Signing] --> REGISTRY
    end
{% endmermaid %}

### Pod Security Context

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: secure-java-app
spec:
  template:
    spec:
      # Pod-level security context
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        runAsGroup: 3000
        fsGroup: 2000
        seccompProfile:
          type: RuntimeDefault
      
      containers:
      - name: java-app
        image: my-java-app:1.0
        # Container-level security context
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          runAsNonRoot: true
          runAsUser: 1000
          capabilities:
            drop:
            - ALL
            add:
            - NET_BIND_SERVICE  # Only if needed for port < 1024
        
        volumeMounts:
        - name: tmp-volume
          mountPath: /tmp
        - name: cache-volume
          mountPath: /app/cache
        
        resources:
          limits:
            memory: "1Gi"
            cpu: "500m"
          requests:
            memory: "512Mi"
            cpu: "250m"
      
      volumes:
      - name: tmp-volume
        emptyDir: {}
      - name: cache-volume
        emptyDir: {}
```

### Secure Dockerfile

```dockerfile
FROM openjdk:11-jre-slim

# Create non-root user
RUN groupadd -r appgroup && useradd -r -g appgroup -u 1000 appuser

# Create app directory
RUN mkdir -p /app/logs /app/cache && \
    chown -R appuser:appgroup /app

# Copy application
COPY --chown=appuser:appgroup target/app.jar /app/app.jar

# Switch to non-root user
USER appuser

WORKDIR /app

# Expose port
EXPOSE 8080

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:8080/actuator/health || exit 1

ENTRYPOINT ["java", "-jar", "app.jar"]
```

### Network Policies

```yaml
# Deny all ingress traffic by default
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-ingress
spec:
  podSelector: {}
  policyTypes:
  - Ingress

---
# Allow specific ingress traffic to Java app
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: java-app-ingress
spec:
  podSelector:
    matchLabels:
      app: java-app
  policyTypes:
  - Ingress
  - Egress
  ingress:
  # Allow traffic from ingress controller
  - from:
    - namespaceSelector:
        matchLabels:
          name: ingress-nginx
    ports:
    - protocol: TCP
      port: 8080
  # Allow traffic from same namespace
  - from:
    - podSelector:
        matchLabels:
          app: frontend
    ports:
    - protocol: TCP
      port: 8080
  egress:
  # Allow DNS resolution
  - to: []
    ports:
    - protocol: UDP
      port: 53
  # Allow database access
  - to:
    - podSelector:
        matchLabels:
          app: database
    ports:
    - protocol: TCP
      port: 5432
  # Allow external API calls
  - to: []
    ports:
    - protocol: TCP
      port: 443
```

### RBAC Configuration

```yaml
# Service Account
apiVersion: v1
kind: ServiceAccount
metadata:
  name: java-app-sa
  namespace: default

---
# Role with minimal permissions
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: java-app-role
  namespace: default
rules:
- apiGroups: [""]
  resources: ["configmaps", "secrets"]
  verbs: ["get", "list", "watch"]
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]

---
# Role Binding
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: java-app-binding
  namespace: default
subjects:
- kind: ServiceAccount
  name: java-app-sa
  namespace: default
roleRef:
  kind: Role
  name: java-app-role
  apiGroup: rbac.authorization.k8s.io

---
# Deployment using Service Account
apiVersion: apps/v1
kind: Deployment
metadata:
  name: java-app
spec:
  template:
    spec:
      serviceAccountName: java-app-sa
      automountServiceAccountToken: false  # Disable if not needed
      containers:
      - name: java-app
        image: my-java-app:1.0
```

### Secrets Management

```yaml
# Create secret from command line (better than YAML)
# kubectl create secret generic db-credentials \
#   --from-literal=username=dbuser \
#   --from-literal=password=securepassword

apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
data:
  username: ZGJ1c2Vy        # base64 encoded
  password: c2VjdXJlcGFzc3dvcmQ=  # base64 encoded

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: java-app
spec:
  template:
    spec:
      containers:
      - name: java-app
        image: my-java-app:1.0
        env:
        - name: DB_USERNAME
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: username
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: db-credentials
              key: password
        # Or mount as files
        volumeMounts:
        - name: db-credentials
          mountPath: /etc/secrets
          readOnly: true
      volumes:
      - name: db-credentials
        secret:
          secretName: db-credentials
          defaultMode: 0400  # Read-only for owner
```

### Pod Security Standards

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: secure-namespace
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

### Security Configuration in Java

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .sessionManagement()
                .sessionCreationPolicy(SessionCreationPolicy.STATELESS)
            .and()
            .headers()
                .frameOptions().deny()
                .contentTypeOptions()
                .and()
                .httpStrictTransportSecurity(hstsConfig -> hstsConfig
                    .maxAgeInSeconds(31536000)
                    .includeSubdomains(true))
            .and()
            .csrf().disable()
            .authorizeHttpRequests(authz -> authz
                .requestMatchers("/actuator/health", "/actuator/info").permitAll()
                .requestMatchers("/actuator/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            .oauth2ResourceServer(OAuth2ResourceServerConfigurer::jwt);
            
        return http.build();
    }
}

@RestController
public class SecureController {
    
    @GetMapping("/secure-endpoint")
    @PreAuthorize("hasRole('USER')")
    public ResponseEntity<String> secureEndpoint(Authentication authentication) {
        // Log access attempt
        log.info("Secure endpoint accessed by: {}", authentication.getName());
        
        return ResponseEntity.ok("Secure data");
    }
}
```

### Image Scanning with Trivy

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: image-scan
spec:
  template:
    spec:
      restartPolicy: Never
      containers:
      - name: trivy
        image: aquasec/trivy:latest
        command:
        - trivy
        - image
        - --exit-code
        - "1"
        - --severity
        - HIGH,CRITICAL
        - my-java-app:1.0
```

### Admission Controller (OPA Gatekeeper)

```yaml
apiVersion: templates.gatekeeper.sh/v1beta1
kind: ConstraintTemplate
metadata:
  name: k8srequiredsecuritycontext
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredSecurityContext
      validation:
        properties:
          runAsNonRoot:
            type: boolean
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequiredsecuritycontext
        
        violation[{"msg": msg}] {
          container := input.review.object.spec.containers[_]
          not container.securityContext.runAsNonRoot
          msg := "Container must run as non-root user"
        }

---
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredSecurityContext
metadata:
  name: must-run-as-non-root
spec:
  match:
    kinds:
      - apiGroups: ["apps"]
        kinds: ["Deployment"]
  parameters:
    runAsNonRoot: true
```

---

## How do you debug issues in Java applications running on Kubernetes?

### Reference Answer

Debugging Kubernetes applications requires understanding both Kubernetes diagnostics and Java-specific debugging techniques.

### Debugging Workflow

{% mermaid graph TD %}
    ISSUE[Application Issue] --> CHECK1{Pod Status}
    CHECK1 -->|Running| CHECK2{Logs Analysis}
    CHECK1 -->|Pending| EVENTS[Check Events]
    CHECK1 -->|Failed| DESCRIBE[Describe Pod]
    
    CHECK2 -->|App Logs| APPLOGS[Application Logs]
    CHECK2 -->|System Logs| SYSLOGS[System Logs]
    
    EVENTS --> RESOURCES{Resource Issues}
    DESCRIBE --> CONFIG{Config Issues}
    
    APPLOGS --> METRICS[Check Metrics]
    SYSLOGS --> NETWORK[Network Debug]
    
    RESOURCES --> SCALE[Scale Resources]
    CONFIG --> FIX[Fix Configuration]
    
    METRICS --> PROFILE[Java Profiling]
    NETWORK --> CONNECTIVITY[Test Connectivity]
{% endmermaid %}

### Basic Kubernetes Debugging Commands

```bash
# Check pod status
kubectl get pods -l app=java-app

# Describe pod for detailed information
kubectl describe pod <pod-name>

# Get pod logs
kubectl logs <pod-name>
kubectl logs <pod-name> --previous  # Previous container logs
kubectl logs <pod-name> -c <container-name>  # Multi-container pod

# Follow logs in real-time
kubectl logs -f <pod-name>

# Get logs from all pods with label
kubectl logs -l app=java-app --tail=100

# Check events
kubectl get events --sort-by=.metadata.creationTimestamp

# Execute commands in pod
kubectl exec -it <pod-name> -- /bin/bash
kubectl exec -it <pod-name> -- ps aux
kubectl exec -it <pod-name> -- netstat -tlnp
```

### Java Application Debugging

```java
@RestController
public class DebugController {
    
    private static final Logger logger = LoggerFactory.getLogger(DebugController.class);
    
    @Autowired
    private MeterRegistry meterRegistry;
    
    @GetMapping("/debug/health")
    public Map<String, Object> getDetailedHealth() {
        Map<String, Object> health = new HashMap<>();
        
        // JVM Memory info
        MemoryMXBean memoryBean = ManagementFactory.getMemoryMXBean();
        MemoryUsage heapUsage = memoryBean.getHeapMemoryUsage();
        MemoryUsage nonHeapUsage = memoryBean.getNonHeapMemoryUsage();
        
        Map<String, Object> memory = new HashMap<>();
        memory.put("heap", Map.of(
            "used", heapUsage.getUsed(),
            "committed", heapUsage.getCommitted(),
            "max", heapUsage.getMax()
        ));
        memory.put("nonHeap", Map.of(
            "used", nonHeapUsage.getUsed(),
            "committed", nonHeapUsage.getCommitted(),
            "max", nonHeapUsage.getMax()
        ));
        
        // Thread info
        ThreadMXBean threadBean = ManagementFactory.getThreadMXBean();
        Map<String, Object> threads = new HashMap<>();
        threads.put("count", threadBean.getThreadCount());
        threads.put("peak", threadBean.getPeakThreadCount());
        threads.put("daemon", threadBean.getDaemonThreadCount());
        
        // GC info
        List<GarbageCollectorMXBean> gcBeans = ManagementFactory.getGarbageCollectorMXBeans();
        List<Map<String, Object>> gcInfo = gcBeans.stream()
            .map(gc -> Map.of(
                "name", gc.getName(),
                "collections", gc.getCollectionCount(),
                "time", gc.getCollectionTime()
            ))
            .collect(Collectors.toList());
        
        health.put("timestamp", Instant.now());
        health.put("memory", memory);
        health.put("threads", threads);
        health.put("gc", gcInfo);
        health.put("uptime", ManagementFactory.getRuntimeMXBean().getUptime());
        
        return health;
    }
    
    @GetMapping("/debug/threads")
    public Map<String, Object> getThreadDump() {
        ThreadMXBean threadBean = ManagementFactory.getThreadMXBean();
        ThreadInfo[] threadInfos = threadBean.dumpAllThreads(true, true);
        
        Map<String, Object> dump = new HashMap<>();
        dump.put("timestamp", Instant.now());
        dump.put("threadCount", threadInfos.length);
        
        List<Map<String, Object>> threads = Arrays.stream(threadInfos)
            .map(info -> {
                Map<String, Object> thread = new HashMap<>();
                thread.put("name", info.getThreadName());
                thread.put("state", info.getThreadState().toString());
                thread.put("blocked", info.getBlockedCount());
                thread.put("waited", info.getWaitedCount());
                
                if (info.getLockInfo() != null) {
                    thread.put("lock", info.getLockInfo().toString());
                }
                
                return thread;
            })
            .collect(Collectors.toList());
            
        dump.put("threads", threads);
        return dump;
    }
    
    @PostMapping("/debug/gc")
    public String triggerGC() {
        logger.warn("Manually triggering garbage collection - this should not be done in production");
        System.gc();
        return "GC triggered";
    }
}
```

### Remote Debugging Configuration

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: java-app-debug
spec:
  replicas: 1  # Single replica for debugging
  template:
    spec:
      containers:
      - name: java-app
        image: my-java-app:1.0
        ports:
        - containerPort: 8080
          name: http
        - containerPort: 5005
          name: debug
        env:
        - name: JAVA_OPTS
          value: >-
            -agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=*:5005
            -XX:+UseContainerSupport
            -XX:MaxRAMPercentage=75.0
        resources:
          requests:
            memory: "1Gi"
            cpu: "500m"
          limits:
            memory: "2Gi"
            cpu: "1000m"

---
apiVersion: v1
kind: Service
metadata:
  name: java-app-debug-service
spec:
  selector:
    app: java-app
  ports:
  - name: http
    port: 8080
    targetPort: 8080
  - name: debug
    port: 5005
    targetPort: 5005
  type: ClusterIP
```

### Port Forwarding for Local Debugging

```bash
# Forward application port
kubectl port-forward deployment/java-app 8080:8080

# Forward debug port
kubectl port-forward deployment/java-app-debug 5005:5005

# Forward multiple ports
kubectl port-forward pod/<pod-name> 8080:8080 5005:5005

# Connect your IDE debugger to localhost:5005
```

### Performance Debugging with JVM Tools

```bash
# Execute JVM diagnostic commands in pod
kubectl exec -it <pod-name> -- jps -l
kubectl exec -it <pod-name> -- jstat -gc <pid> 5s
kubectl exec -it <pod-name> -- jstack <pid>
kubectl exec -it <pod-name> -- jmap -histo <pid>

# Create heap dump
kubectl exec -it <pod-name> -- jcmd <pid> GC.run_finalization
kubectl exec -it <pod-name> -- jcmd <pid> VM.gc
kubectl exec -it <pod-name> -- jcmd <pid> GC.dump /tmp/heapdump.hprof

# Copy heap dump to local machine
kubectl cp <pod-name>:/tmp/heapdump.hprof ./heapdump.hprof
```

### Network Debugging

```yaml
# Network debugging pod
apiVersion: v1
kind: Pod
metadata:
  name: network-debug
spec:
  containers:
  - name: network-tools
    image: nicolaka/netshoot
    command: ["/bin/bash"]
    args: ["-c", "while true; do ping localhost; sleep 30;done"]
  restartPolicy: Always
```

```bash
# Test network connectivity
kubectl exec -it network-debug -- nslookup java-app-service
kubectl exec -it network-debug -- curl -v http://java-app-service:8080/health
kubectl exec -it network-debug -- telnet java-app-service 8080

# Check DNS resolution
kubectl exec -it network-debug -- dig java-app-service.default.svc.cluster.local

# Test external connectivity
kubectl exec -it network-debug -- curl -v https://api.external-service.com

# Network policy testing
kubectl exec -it network-debug -- nc -zv java-app-service 8080
```

### Debugging Init Containers

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: java-app-with-init
spec:
  template:
    spec:
      initContainers:
      - name: wait-for-db
        image: busybox:1.28
        command: ['sh', '-c']
        args:
        - |
          echo "Waiting for database..."
          until nc -z database-service 5432; do
            echo "Database not ready, waiting..."
            sleep 2
          done
          echo "Database is ready!"
      - name: migrate-db
        image: migrate/migrate
        command: ["/migrate"]
        args:
        - "-path=/migrations"
        - "-database=postgresql://user:pass@database-service:5432/db?sslmode=disable"
        - "up"
        volumeMounts:
        - name: migrations
          mountPath: /migrations
      containers:
      - name: java-app
        image: my-java-app:1.0
      volumes:
      - name: migrations
        configMap:
          name: db-migrations
```

```bash
# Check init container logs
kubectl logs <pod-name> -c wait-for-db
kubectl logs <pod-name> -c migrate-db

# Describe pod to see init container status
kubectl describe pod <pod-name>
```

### Application Metrics Debugging

```java
@Component
public class CustomMetrics {
    
    private final Counter httpRequestsTotal;
    private final Timer httpRequestDuration;
    private final Gauge activeConnections;
    
    public CustomMetrics(MeterRegistry meterRegistry) {
        this.httpRequestsTotal = Counter.builder("http_requests_total")
            .description("Total HTTP requests")
            .register(meterRegistry);
            
        this.httpRequestDuration = Timer.builder("http_request_duration")
            .description("HTTP request duration")
            .register(meterRegistry);
            
        this.activeConnections = Gauge.builder("active_connections")
            .description("Active database connections")
            .register(meterRegistry, this, CustomMetrics::getActiveConnections);
    }
    
    public void recordRequest(String method, String endpoint, long duration) {
        httpRequestsTotal.increment(
            Tags.of("method", method, "endpoint", endpoint)
        );
        httpRequestDuration.record(duration, TimeUnit.MILLISECONDS);
    }
    
    public double getActiveConnections() {
        // Return actual active connection count
        return 10.0; // Placeholder
    }
}
```

### Debugging Persistent Volumes

```bash
# Check PV and PVC status
kubectl get pv
kubectl get pvc

# Describe PVC for detailed info
kubectl describe pvc java-app-storage

# Check mounted volumes in pod
kubectl exec -it <pod-name> -- df -h
kubectl exec -it <pod-name> -- ls -la /app/data

# Check file permissions
kubectl exec -it <pod-name> -- ls -la /app/data
kubectl exec -it <pod-name> -- id

# Test file creation
kubectl exec -it <pod-name> -- touch /app/data/test.txt
kubectl exec -it <pod-name> -- echo "test" > /app/data/test.txt
```

### Resource Usage Investigation

```bash
# Check resource usage
kubectl top pods
kubectl top nodes

# Get detailed resource information
kubectl describe node <node-name>

# Check resource quotas
kubectl get resourcequota
kubectl describe resourcequota

# Check limit ranges
kubectl get limitrange
kubectl describe limitrange
```

### Debugging ConfigMaps and Secrets

```bash
# Check ConfigMap content
kubectl get configmap java-app-config -o yaml

# Check Secret content (base64 encoded)
kubectl get secret java-app-secrets -o yaml

# Decode secret values
kubectl get secret java-app-secrets -o jsonpath='{.data.database-password}' | base64 --decode

# Check mounted config in pod
kubectl exec -it <pod-name> -- cat /app/config/application.properties
kubectl exec -it <pod-name> -- env | grep -i database
```

### Automated Debugging Script

```bash
#!/bin/bash

APP_NAME="java-app"
NAMESPACE="default"

echo "=== Kubernetes Debugging Report for $APP_NAME ==="
echo "Timestamp: $(date)"
echo

echo "=== Pod Status ==="
kubectl get pods -l app=$APP_NAME -n $NAMESPACE
echo

echo "=== Recent Events ==="
kubectl get events --sort-by=.metadata.creationTimestamp -n $NAMESPACE | tail -10
echo

echo "=== Pod Description ==="
POD_NAME=$(kubectl get pods -l app=$APP_NAME -n $NAMESPACE -o jsonpath='{.items[0].metadata.name}')
kubectl describe pod $POD_NAME -n $NAMESPACE
echo

echo "=== Application Logs (last 50 lines) ==="
kubectl logs $POD_NAME -n $NAMESPACE --tail=50
echo

echo "=== Resource Usage ==="
kubectl top pod $POD_NAME -n $NAMESPACE
echo

echo "=== Service Status ==="
kubectl get svc -l app=$APP_NAME -n $NAMESPACE
echo

echo "=== ConfigMap Status ==="
kubectl get configmap -l app=$APP_NAME -n $NAMESPACE
echo

echo "=== Secret Status ==="
kubectl get secret -l app=$APP_NAME -n $NAMESPACE
echo

echo "=== Network Connectivity Test ==="
kubectl run debug-pod --rm -i --restart=Never --image=nicolaka/netshoot -- \
  /bin/bash -c "nslookup $APP_NAME-service.$NAMESPACE.svc.cluster.local && \
                curl -s -o /dev/null -w '%{http_code}' http://$APP_NAME-service.$NAMESPACE.svc.cluster.local:8080/health"
```

---

## Explain Ingress and how to expose Java applications externally

### Reference Answer

Ingress provides HTTP and HTTPS routing to services within a Kubernetes cluster, acting as a reverse proxy and load balancer for external traffic.

### Ingress Architecture

{% mermaid graph TB %}
    INTERNET[Internet] --> LB[Load Balancer]
    LB --> INGRESS[Ingress Controller]
    
    subgraph "Kubernetes Cluster"
        INGRESS --> INGRESS_RULES[Ingress Rules]
        INGRESS_RULES --> SVC1[Java App Service]
        INGRESS_RULES --> SVC2[API Service]
        INGRESS_RULES --> SVC3[Frontend Service]
        
        SVC1 --> POD1[Java App Pods]
        SVC2 --> POD2[API Pods]
        SVC3 --> POD3[Frontend Pods]
    end
{% endmermaid %}

### Basic Ingress Configuration

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: java-app-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "60"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "60"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "60"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - myapp.example.com
    secretName: myapp-tls
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: java-app-service
            port:
              number: 80
```

### Advanced Ingress with Path-based Routing

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: microservices-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
    nginx.ingress.kubernetes.io/use-regex: "true"
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"
    nginx.ingress.kubernetes.io/rate-limit: "100"
    nginx.ingress.kubernetes.io/rate-limit-window: "1m"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - api.example.com
    secretName: api-tls
  rules:
  - host: api.example.com
    http:
      paths:
      # User service
      - path: /api/users(/|$)(.*)
        pathType: Prefix
        backend:
          service:
            name: user-service
            port:
              number: 80
      # Order service  
      - path: /api/orders(/|$)(.*)
        pathType: Prefix
        backend:
          service:
            name: order-service
            port:
              number: 80
      # Payment service
      - path: /api/payments(/|$)(.*)
        pathType: Prefix
        backend:
          service:
            name: payment-service
            port:
              number: 80
      # Default fallback
      - path: /(.*)
        pathType: Prefix
        backend:
          service:
            name: frontend-service
            port:
              number: 80
```

### Java Application Configuration for Ingress

```java
@RestController
@RequestMapping("/api/orders")
public class OrderController {
    
    // Handle base path properly
    @GetMapping("/health")
    public ResponseEntity<Map<String, String>> health(HttpServletRequest request) {
        Map<String, String> health = new HashMap<>();
        health.put("status", "UP");
        health.put("service", "order-service");
        health.put("path", request.getRequestURI());
        health.put("forwardedPath", request.getHeader("X-Forwarded-Prefix"));
        
        return ResponseEntity.ok(health);
    }
    
    @GetMapping
    public ResponseEntity<List<OrderDto>> getOrders(
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "10") int size,
            HttpServletRequest request) {
        
        // Log forwarded headers for debugging
        String forwardedFor = request.getHeader("X-Forwarded-For");
        String forwardedProto = request.getHeader("X-Forwarded-Proto");
        String forwardedHost = request.getHeader("X-Forwarded-Host");
        
        log.info("Request from: {} via {} to {}", forwardedFor, forwardedProto, forwardedHost);
        
        List<OrderDto> orders = orderService.getOrders(page, size);
        return ResponseEntity.ok(orders);
    }
}
```

### Spring Boot Configuration for Proxy Headers

```yaml
server:
  port: 8080
  servlet:
    context-path: /
  forward-headers-strategy: native
  tomcat:
    remoteip:
      remote-ip-header: X-Forwarded-For
      protocol-header: X-Forwarded-Proto
      port-header: X-Forwarded-Port

management:
  server:
    port: 8081
  endpoints:
    web:
      base-path: /actuator
      exposure:
        include: health,info,metrics,prometheus
```

### SSL/TLS Certificate Management

```yaml
# Using cert-manager for automatic SSL certificates
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: admin@example.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
    - http01:
        ingress:
          class: nginx

---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: java-app-ssl-ingress
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - myapp.example.com
    secretName: myapp-tls-auto  # Will be created by cert-manager
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: java-app-service
            port:
              number: 80
```

### Ingress with Authentication

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: java-app-auth-ingress
  annotations:
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/auth-secret: basic-auth
    nginx.ingress.kubernetes.io/auth-realm: 'Authentication Required'
    # Or OAuth2 authentication
    nginx.ingress.kubernetes.io/auth-url: "https://auth.example.com/oauth2/auth"
    nginx.ingress.kubernetes.io/auth-signin: "https://auth.example.com/oauth2/start"
spec:
  ingressClassName: nginx
  rules:
  - host: secure.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: java-app-service
            port:
              number: 80

---
# Basic auth secret
apiVersion: v1
kind: Secret
metadata:
  name: basic-auth
type: Opaque
data:
  auth: YWRtaW46JGFwcjEkSDY1dnVhNU8kblNEOC9ObDBINFkwL3pmWUZOcUI4MQ==  # admin:admin
```

### Custom Error Pages

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: custom-error-pages
data:
  404.html: |
    <!DOCTYPE html>
    <html>
    <head>
        <title>Page Not Found</title>
        <style>
            body { font-family: Arial, sans-serif; text-align: center; margin-top: 50px; }
            .error-code { font-size: 72px; color: #e74c3c; }
            .error-message { font-size: 24px; color: #7f8c8d; }
        </style>
    </head>
    <body>
        <div class="error-code">404</div>
        <div class="error-message">The page you're looking for doesn't exist.</div>
        <p><a href="/">Go back to homepage</a></p>
    </body>
    </html>
  500.html: |
    <!DOCTYPE html>
    <html>
    <head>
        <title>Internal Server Error</title>
        <style>
            body { font-family: Arial, sans-serif; text-align: center; margin-top: 50px; }
            .error-code { font-size: 72px; color: #e74c3c; }
            .error-message { font-size: 24px; color: #7f8c8d; }
        </style>
    </head>
    <body>
        <div class="error-code">500</div>
        <div class="error-message">Something went wrong on our end.</div>
        <p>Please try again later.</p>
    </body>
    </html>

---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: java-app-custom-errors
  annotations:
    nginx.ingress.kubernetes.io/custom-http-errors: "404,500,503"
    nginx.ingress.kubernetes.io/default-backend: error-pages-service
spec:
  ingressClassName: nginx
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: java-app-service
            port:
              number: 80
```

### Load Balancing and Session Affinity

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: java-app-sticky-sessions
  annotations:
    nginx.ingress.kubernetes.io/affinity: "cookie"
    nginx.ingress.kubernetes.io/affinity-mode: "persistent"
    nginx.ingress.kubernetes.io/session-cookie-name: "JSESSIONID"
    nginx.ingress.kubernetes.io/session-cookie-expires: "86400"
    nginx.ingress.kubernetes.io/session-cookie-max-age: "86400"
    nginx.ingress.kubernetes.io/session-cookie-path: "/"
    nginx.ingress.kubernetes.io/upstream-hash-by: "$remote_addr"
spec:
  ingressClassName: nginx
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: java-app-service
            port:
              number: 80
```

### Ingress Health Checks and Monitoring

```java
@RestController
public class IngressHealthController {
    
    @GetMapping("/health/ingress")
    public ResponseEntity<Map<String, Object>> ingressHealth(HttpServletRequest request) {
        Map<String, Object> health = new HashMap<>();
        health.put("status", "UP");
        health.put("timestamp", Instant.now());
        
        // Include request information for debugging
        Map<String, String> requestInfo = new HashMap<>();
        requestInfo.put("remoteAddr", request.getRemoteAddr());
        requestInfo.put("forwardedFor", request.getHeader("X-Forwarded-For"));
        requestInfo.put("forwardedProto", request.getHeader("X-Forwarded-Proto"));
        requestInfo.put("forwardedHost", request.getHeader("X-Forwarded-Host"));
        requestInfo.put("userAgent", request.getHeader("User-Agent"));
        
        health.put("request", requestInfo);
        
        return ResponseEntity.ok(health);
    }
    
    @GetMapping("/ready")
    public ResponseEntity<String> readiness() {
        // Perform readiness checks
        if (isApplicationReady()) {
            return ResponseEntity.ok("Ready");
        } else {
            return ResponseEntity.status(503).body("Not Ready");
        }
    }
    
    private boolean isApplicationReady() {
        // Check database connectivity, external services, etc.
        return true;
    }
}
```

### Ingress Controller Configuration

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-configuration
  namespace: ingress-nginx
data:
  # Global settings
  proxy-connect-timeout: "60"
  proxy-send-timeout: "60"
  proxy-read-timeout: "60"
  proxy-body-size: "100m"
  
  # Performance tuning
  worker-processes: "auto"
  worker-connections: "1024"
  keepalive-timeout: "65"
  keepalive-requests: "100"
  
  # Security headers
  add-headers: "ingress-nginx/custom-headers"
  
  # Compression
  enable-gzip: "true"
  gzip-types: "text/plain text/css application/json application/javascript text/xml application/xml"
  
  # Rate limiting
  rate-limit: "1000"
  rate-limit-window: "1m"

---
apiVersion: v1
kind: ConfigMap
metadata:
  name: custom-headers
  namespace: ingress-nginx
data:
  X-Content-Type-Options: "nosniff"
  X-Frame-Options: "DENY"
  X-XSS-Protection: "1; mode=block"
  Strict-Transport-Security: "max-age=31536000; includeSubDomains"
  Content-Security-Policy: "default-src 'self'"
```

### Testing Ingress Configuration

```bash
# Test basic connectivity
curl -H "Host: myapp.example.com" http://<ingress-ip>/health

# Test SSL
curl -H "Host: myapp.example.com" https://<ingress-ip>/health

# Test with custom headers
curl -H "Host: myapp.example.com" -H "X-Custom-Header: test" http://<ingress-ip>/api/orders

# Test different paths
curl -H "Host: api.example.com" http://<ingress-ip>/api/users/health
curl -H "Host: api.example.com" http://<ingress-ip>/api/orders/health

# Debug ingress controller logs
kubectl logs -n ingress-nginx deployment/ingress-nginx-controller -f

# Check ingress status
kubectl get ingress
kubectl describe ingress java-app-ingress
```

This comprehensive guide covers the essential Kubernetes concepts and practical implementations that senior Java developers need to understand when working with containerized applications in production environments. 