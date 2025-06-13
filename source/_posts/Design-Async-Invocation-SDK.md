---
title: Design Async Invocation SDK
date: 2025-06-13 18:12:54
tags: [system-design, async invocation]
categories: [system-design]
---

## Executive Summary

The General Async Invoke SDK is a unified framework that abstracts asynchronous method invocation and message-driven communication. It provides a consistent API for developers while supporting multiple message queue backends (Kafka, RocketMQ, Redis) through a Service Provider Interface (SPI) mechanism.

### Key Features
- **Unified API**: Single interface for async operations regardless of underlying MQ
- **Multi-Protocol Support**: Kafka, RocketMQ, Redis with pluggable architecture
- **Callback Management**: Type-safe callback handling with timeout and retry mechanisms
- **Request-Response Pattern**: Correlation ID-based async method invocation
- **Circuit Breaker**: Built-in resilience patterns for production reliability

## Architecture Overview

{% mermaid graph TB %}
    subgraph "Client Application"
        CA[Client App]
        SDK[Async Invoke SDK]
        CB[Callback Manager]
    end
    
    subgraph "SDK Core"
        API[Unified API Layer]
        SPI[SPI Framework]
        CM[Connection Manager]
        SM[Serialization Manager]
    end
    
    subgraph "MQ Implementations"
        KI[Kafka Impl]
        RI[RocketMQ Impl]
        RedisI[Redis Impl]
    end
    
    subgraph "Message Queues"
        K[Kafka Cluster]
        R[RocketMQ Cluster]
        Redis[Redis Cluster]
    end
    
    subgraph "Backend Services"
        BS1[Service A]
        BS2[Service B]
        BS3[Service C]
    end
    
    CA --> SDK
    SDK --> API
    API --> SPI
    SPI --> KI
    SPI --> RI
    SPI --> RedisI
    
    KI --> K
    RI --> R
    RedisI --> Redis
    
    K --> BS1
    R --> BS2
    Redis --> BS3
    
    BS1 --> K
    BS2 --> R
    BS3 --> Redis
    
    K --> KI
    R --> RI
    Redis --> RedisI
    
    KI --> CB
    RI --> CB
    RedisI --> CB
    
    CB --> CA
{% endmermaid %}

**Interview Insight**: *When designing distributed systems, the abstraction layer is crucial. Interviewers often ask about trade-offs between abstraction and performance. Here, we sacrifice some performance for maintainability and flexibility.*

## Core Components

### 1. AsyncInvoker Interface

```java
public interface AsyncInvoker {
    <T> CompletableFuture<T> asyncGet(String endpoint, Class<T> responseType);
    <T> CompletableFuture<T> asyncPost(String endpoint, Object request, Class<T> responseType);
    <T> CompletableFuture<T> asyncPut(String endpoint, Object request, Class<T> responseType);
    <T> CompletableFuture<T> asyncDelete(String endpoint, Class<T> responseType);
    
    // Callback-based methods
    <T> void asyncGet(String endpoint, Class<T> responseType, AsyncCallback<T> callback);
    <T> void asyncPost(String endpoint, Object request, Class<T> responseType, AsyncCallback<T> callback);
}
```

### 2. Message Flow Architecture

{% mermaid sequenceDiagram %}
    participant C as Client
    participant SDK as SDK Core
    participant MQ as Message Queue
    participant S as Backend Service
    
    C->>SDK: asyncPost(endpoint, data, callback)
    SDK->>SDK: Generate correlationId
    SDK->>MQ: Publish request message
    SDK->>C: Return immediately
    
    MQ->>S: Deliver message
    S->>S: Process request
    S->>MQ: Publish response with correlationId
    
    MQ->>SDK: Deliver response
    SDK->>SDK: Match correlationId
    SDK->>C: Invoke callback(result)
{% endmermaid %}

**Interview Insight**: *This sequence diagram demonstrates the Request-Response pattern over async messaging. Interviewers often probe about correlation ID management and potential race conditions.*

## SPI Mechanism Design

### Service Provider Interface Structure

```java
public interface MessageQueueProvider {
    String getProviderName();
    boolean isAvailable();
    MessagePublisher createPublisher(Properties config);
    MessageSubscriber createSubscriber(Properties config);
    void initialize(Properties config);
    void shutdown();
}
```

### Provider Loading Mechanism

{% mermaid flowchart TD %}
    A[SDK Initialization] --> B[Scan Classpath]
    B --> C[Load META-INF/services]
    C --> D{Provider Available?}
    D -->|Yes| E[Initialize Provider]
    D -->|No| F[Try Next Provider]
    E --> G[Register Provider]
    F --> H{More Providers?}
    H -->|Yes| D
    H -->|No| I[Throw Exception]
    G --> J[SDK Ready]
{% endmermaid %}

**Interview Insight**: *SPI pattern questions often focus on classloader issues and provider precedence. The key is understanding Java's ServiceLoader mechanism and its limitations in complex deployment scenarios.*

## Message Queue Implementations

### Kafka Implementation

```java
@Service("kafka")
public class KafkaMessageQueueProvider implements MessageQueueProvider {
    @Override
    public MessagePublisher createPublisher(Properties config) {
        return new KafkaMessagePublisher(
            new KafkaProducer<>(buildKafkaConfig(config))
        );
    }
    
    @Override
    public MessageSubscriber createSubscriber(Properties config) {
        return new KafkaMessageSubscriber(
            new KafkaConsumer<>(buildKafkaConfig(config))
        );
    }
}
```

### Comparison Matrix

| Feature | Kafka | RocketMQ | Redis |
|---------|-------|----------|-------|
| **Throughput** | Very High | High | Medium |
| **Latency** | Low | Low | Very Low |
| **Durability** | Excellent | Excellent | Configurable |
| **Ordering** | Partition-level | Global/Partition | Limited |
| **Message TTL** | Retention-based | Built-in | Built-in |
| **Memory Usage** | Low | Medium | High |

**Interview Insight**: *When asked about message queue selection, discuss trade-offs. Kafka excels in throughput but has higher operational complexity. Redis offers lowest latency but limited durability guarantees.*

## Async Method Invocation

### Request-Response Correlation

```java
public class CorrelationManager {
    private final ConcurrentHashMap<String, PendingRequest> pendingRequests;
    private final ScheduledExecutorService timeoutExecutor;
    
    public <T> CompletableFuture<T> registerRequest(String correlationId, 
                                                   Class<T> responseType, 
                                                   Duration timeout) {
        CompletableFuture<T> future = new CompletableFuture<>();
        PendingRequest<T> request = new PendingRequest<>(future, responseType, timeout);
        
        pendingRequests.put(correlationId, request);
        scheduleTimeout(correlationId, timeout);
        
        return future;
    }
    
    public void handleResponse(String correlationId, Object response) {
        PendingRequest request = pendingRequests.remove(correlationId);
        if (request != null) {
            request.getFuture().complete(response);
        }
    }
}
```

### Timeout Handling Flow

{% mermaid graph LR %}
    A[Request Sent] --> B[Start Timer]
    B --> C{Response Received?}
    C -->|Yes| D[Complete Future]
    C -->|No| E{Timeout?}
    E -->|No| C
    E -->|Yes| F[Cancel Request]
    F --> G[Complete Exceptionally]
{% endmermaid %}

**Interview Insight**: *Timeout handling is a common interview topic. Discuss the importance of cleaning up resources and preventing memory leaks in long-running applications.*

## Callback Management

### Type-Safe Callback Interface

```java
public interface AsyncCallback<T> {
    void onSuccess(T result);
    void onFailure(Throwable throwable);
    
    default Duration getTimeout() {
        return Duration.ofSeconds(30);
    }
    
    default boolean shouldRetry(Throwable throwable) {
        return throwable instanceof RetryableException;
    }
}
```

### Callback Execution Model

```java
public class CallbackExecutor {
    private final ExecutorService callbackExecutor;
    private final CircuitBreaker circuitBreaker;
    
    public <T> void executeCallback(AsyncCallback<T> callback, T result, Throwable error) {
        callbackExecutor.submit(() -> {
            try {
                if (error != null) {
                    callback.onFailure(error);
                } else {
                    callback.onSuccess(result);
                }
            } catch (Exception e) {
                // Log callback execution failure
                logger.error("Callback execution failed", e);
            }
        });
    }
}
```

**Interview Insight**: *Callback execution on separate threads prevents blocking the message processing pipeline. Discuss thread pool sizing and bounded queues to prevent resource exhaustion.*

## Error Handling & Resilience

### Circuit Breaker Pattern

{% mermaid stateDiagram-v2 %}
    [*] --> Closed
    Closed --> HalfOpen : Failure threshold reached
    Closed --> Closed : Success
    HalfOpen --> Open : Test call fails
    HalfOpen --> Closed : Test call succeeds
    Open --> HalfOpen : Timeout elapsed
    Open --> Open : Call rejected
{% endmermaid %}

### Retry Mechanism

```java
public class RetryableInvoker {
    private final RetryPolicy retryPolicy;
    
    public <T> CompletableFuture<T> invoke(Supplier<CompletableFuture<T>> operation) {
        return retryAsync(operation, retryPolicy.getMaxAttempts())
            .exceptionally(throwable -> {
                if (throwable instanceof RetryExhaustedException) {
                    // All retries failed
                    throw new InvocationException("Request failed after retries", throwable);
                }
                throw new RuntimeException(throwable);
            });
    }
    
    private <T> CompletableFuture<T> retryAsync(Supplier<CompletableFuture<T>> operation, int attemptsLeft) {
        return operation.get()
            .exceptionally(throwable -> {
                if (attemptsLeft > 1 && isRetryable(throwable)) {
                    return retryAsync(operation, attemptsLeft - 1).join();
                }
                throw new CompletionException(throwable);
            });
    }
}
```

**Interview Insight**: *Exponential backoff with jitter is crucial for preventing thundering herd problems. Interviewers often ask about retry storm scenarios and mitigation strategies.*

## Performance Considerations

### Connection Pooling

```java
public class ConnectionPoolManager {
    private final Map<String, ObjectPool<Connection>> connectionPools;
    
    public Connection borrowConnection(String providerName) {
        return connectionPools.get(providerName).borrowObject();
    }
    
    public void returnConnection(String providerName, Connection connection) {
        connectionPools.get(providerName).returnObject(connection);
    }
}
```

### Serialization Optimization

| Serialization | Pros | Cons | Use Case |
|---------------|------|------|----------|
| **JSON** | Human readable, widely supported | Verbose, slower | Development, debugging |
| **Protobuf** | Compact, fast, schema evolution | Complex setup | High-performance production |
| **Avro** | Schema evolution, compact | Requires schema registry | Data pipelines |
| **MessagePack** | Compact, fast | Limited language support | Performance-critical |

**Interview Insight**: *Serialization choice significantly impacts performance. Discuss schema evolution challenges and backward compatibility requirements.*

## Implementation Examples

### Basic Usage

```java
// Configuration
Properties config = new Properties();
config.setProperty("mq.provider", "kafka");
config.setProperty("kafka.bootstrap.servers", "localhost:9092");

// Initialize SDK
AsyncInvoker invoker = AsyncInvokerBuilder.newBuilder()
    .withConfig(config)
    .build();

// Async method call with CompletableFuture
CompletableFuture<UserProfile> future = invoker.asyncGet("/users/123", UserProfile.class);
future.thenAccept(profile -> {
    System.out.println("User: " + profile.getName());
});

// Async method call with callback
invoker.asyncPost("/users", newUser, UserProfile.class, new AsyncCallback<UserProfile>() {
    @Override
    public void onSuccess(UserProfile result) {
        System.out.println("User created: " + result.getId());
    }
    
    @Override
    public void onFailure(Throwable throwable) {
        System.err.println("Failed to create user: " + throwable.getMessage());
    }
});
```

### Advanced Configuration

```java
AsyncInvoker invoker = AsyncInvokerBuilder.newBuilder()
    .withProvider("kafka")
    .withConnectionPool(10, 50) // min, max connections
    .withTimeout(Duration.ofSeconds(30))
    .withRetryPolicy(RetryPolicy.exponentialBackoff(3, Duration.ofSeconds(1)))
    .withCircuitBreaker(CircuitBreakerConfig.builder()
        .failureThreshold(5)
        .timeoutDuration(Duration.ofSeconds(10))
        .build())
    .withSerialization(SerializationType.PROTOBUF)
    .build();
```

### Spring Boot Integration

```java
@Configuration
@EnableAsyncInvoke
public class AsyncInvokeConfig {
    
    @Bean
    @ConfigurationProperties("async.invoke")
    public AsyncInvokeProperties asyncInvokeProperties() {
        return new AsyncInvokeProperties();
    }
    
    @Bean
    public AsyncInvoker asyncInvoker(AsyncInvokeProperties properties) {
        return AsyncInvokerBuilder.newBuilder()
            .withProperties(properties)
            .build();
    }
}

@Service
public class UserService {
    
    @Autowired
    private AsyncInvoker asyncInvoker;
    
    public CompletableFuture<User> getUserAsync(Long userId) {
        return asyncInvoker.asyncGet("/users/" + userId, User.class);
    }
}
```

## Interview Insights

### Common Questions & Answers

**Q: How do you handle message ordering in distributed systems?**
A: *Message ordering depends on the use case. Kafka provides partition-level ordering, which is sufficient for most scenarios. For global ordering, you can use a single partition (sacrificing throughput) or implement application-level sequencing with timestamp-based ordering.*

**Q: What happens if the callback thread pool is exhausted?**
A: *We use bounded queues with rejection policies. The RejectedExecutionHandler can either:*
- *Block the caller (risk of deadlock)*
- *Execute in the calling thread (impacts message processing)*
- *Drop the task (data loss)*
- *Use a separate overflow queue with different processing guarantees*

**Q: How do you handle duplicate messages?**
A: *Implement idempotency at the application level using correlation IDs or unique request identifiers. The SDK provides hooks for duplicate detection, but business logic must ensure operations are idempotent.*

**Q: What's the difference between at-least-once and exactly-once delivery?**
A: *At-least-once guarantees message delivery but allows duplicates. Exactly-once is theoretically impossible in distributed systems but can be achieved through idempotent operations and deduplication. Our SDK supports both patterns through configuration.*

**Q: How do you monitor the health of async operations?**
A: *We expose metrics through JMX/Micrometer:*
- *Request/response rates*
- *Latency percentiles*
- *Error rates by type*
- *Circuit breaker states*
- *Queue depths and consumer lag*

### Performance Tuning Tips

1. **Batch Operations**: Group multiple requests to reduce network overhead
2. **Connection Reuse**: Implement proper connection pooling
3. **Serialization**: Choose appropriate serialization format for your use case
4. **Thread Pool Sizing**: Size thread pools based on I/O vs CPU-bound operations
5. **Memory Management**: Monitor for memory leaks in long-running applications

### Production Considerations

- **Graceful Shutdown**: Implement proper cleanup of resources and in-flight requests
- **Health Checks**: Provide endpoints for monitoring system health
- **Configuration Management**: Support dynamic configuration updates without restart
- **Security**: Implement authentication and authorization for message queues
- **Observability**: Comprehensive logging, metrics, and distributed tracing

## Conclusion

The General Async Invoke SDK provides a robust foundation for building scalable, resilient distributed systems. By abstracting message queue specifics behind a unified interface, it enables teams to focus on business logic while maintaining flexibility in infrastructure choices.

The SPI mechanism ensures extensibility, while built-in resilience patterns like circuit breakers and retry logic provide production-ready reliability. The combination of CompletableFuture and callback-based APIs caters to different programming styles and use cases.

Key success factors for implementation:
- Thorough testing of error scenarios
- Comprehensive monitoring and alerting
- Clear documentation for developers
- Performance benchmarking across different MQ providers
- Regular review of timeout and retry configurations

This design balances flexibility, performance, and ease of use while providing the foundation for enterprise-grade asynchronous communication patterns.
