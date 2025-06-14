---
title: Design Message Notification Service
date: 2025-06-13 20:54:18
tags: [system-design, message notification]
categories: [system-design]
---

## System Overview

The Message Notification Service is a scalable, multi-channel notification platform designed to handle 10 million messages per day across email, SMS, and WeChat channels. The system employs event-driven architecture with message queues for decoupling, template-based messaging, and comprehensive delivery tracking.

**Interview Insight**: When discussing notification systems, emphasize the trade-offs between consistency and availability. For notifications, we typically choose availability over strict consistency since delayed delivery is preferable to no delivery.

{% mermaid graph TB %}
    A[Business Services] --> B[MessageNotificationSDK]
    B --> C[API Gateway]
    C --> D[Message Service]
    D --> E[Message Queue]
    E --> F[Channel Processors]
    F --> G[Email Service]
    F --> H[SMS Service]
    F --> I[WeChat Service]
    D --> J[Template Engine]
    D --> K[Scheduler Service]
    F --> L[Delivery Tracker]
    L --> M[Analytics DB]
    D --> N[Message Store]
{% endmermaid %}

## Core Architecture Components

### Message Notification Service API

The central service provides RESTful APIs for immediate and scheduled notifications:

```json
{
  "messageId": "msg_123456",
  "recipients": [
    {
      "userId": "user_001",
      "channels": ["email", "sms"],
      "email": "user@example.com",
      "phone": "+1234567890"
    }
  ],
  "template": {
    "templateId": "welcome_template",
    "variables": {
      "userName": "John Doe",
      "activationLink": "https://app.com/activate/xyz"
    }
  },
  "priority": "high",
  "scheduledAt": "2024-03-15T10:00:00Z",
  "retryPolicy": {
    "maxRetries": 3,
    "backoffMultiplier": 2
  }
}
```

**Interview Insight**: Discuss idempotency here - each message should have a unique ID to prevent duplicate sends. This is crucial for financial notifications or critical alerts.

### Message Queue Architecture

The system uses Apache Kafka for high-throughput message processing with the following topic structure:

- `notification.immediate` - Real-time notifications
- `notification.scheduled` - Scheduled notifications  
- `notification.retry` - Failed message retries
- `notification.dlq` - Dead letter queue for permanent failures

{% mermaid flowchart LR %}
    A[API Gateway] --> B[Message Validator]
    B --> C{Message Type}
    C -->|Immediate| D[notification.immediate]
    C -->|Scheduled| E[notification.scheduled]
    D --> F[Channel Router]
    E --> G[Scheduler Service]
    G --> F
    F --> H[Email Processor]
    F --> I[SMS Processor]
    F --> J[WeChat Processor]
    H --> K[Email Provider]
    I --> L[SMS Provider]
    J --> M[WeChat API]
{% endmermaid %}

**Interview Insight**: Explain partitioning strategy - partition by user ID for ordered processing per user, or by message type for parallel processing. The choice depends on whether message ordering matters for your use case.

### Template Engine Design

Templates support dynamic content injection with internationalization:

```yaml
templates:
  welcome_email:
    subject: "Welcome {{userName}} - {{companyName}}"
    body: |
      <html>
        <body>
          <h1>Welcome {{userName}}!</h1>
          <p>Thank you for joining {{companyName}}.</p>
          <a href="{{activationLink}}">Activate Account</a>
        </body>
      </html>
    channels: ["email"]
    variables:
      - userName: required
      - companyName: required
      - activationLink: required
  
  sms_verification:
    body: "Your {{companyName}} verification code: {{code}}. Valid for {{expiry}} minutes."
    channels: ["sms"]
    variables:
      - companyName: required
      - code: required
      - expiry: required
```

**Interview Insight**: Template versioning is critical for production systems. Discuss A/B testing capabilities where different template versions can be tested simultaneously to optimize engagement rates.

## Scalability and Performance

### High-Volume Message Processing

To handle 10 million messages daily (approximately 116 messages/second average, 1000+ messages/second peak):

**Horizontal Scaling Strategy**:
- Multiple Kafka consumer groups for parallel processing
- Channel-specific processors with independent scaling
- Load balancing across processor instances

**Performance Optimizations**:
- Connection pooling for external APIs
- Batch processing for similar notifications
- Asynchronous processing with circuit breakers

{% mermaid sequenceDiagram %}
    participant BS as Business Service
    participant SDK as Notification SDK
    participant API as API Gateway
    participant MQ as Message Queue
    participant CP as Channel Processor
    participant EP as Email Provider
    
    BS->>SDK: sendNotification(request)
    SDK->>API: POST /notifications
    API->>API: Validate & Enrich
    API->>MQ: Publish message
    API-->>SDK: messageId (async)
    SDK-->>BS: messageId
    
    MQ->>CP: Consume message
    CP->>CP: Apply template
    CP->>EP: Send email
    EP-->>CP: Delivery status
    CP->>MQ: Update delivery status
{% endmermaid %}

**Interview Insight**: Discuss the CAP theorem application - in notification systems, we choose availability and partition tolerance over consistency. It's better to potentially send a duplicate notification than to miss sending one entirely.

### Caching Strategy

**Multi-Level Caching**:
- **Template Cache**: Redis cluster for compiled templates
- **User Preference Cache**: User notification preferences and contact info
- **Rate Limiting Cache**: Sliding window counters for rate limiting

## Channel-Specific Implementations

### Email Service
```java
@Service
public class EmailChannelProcessor implements ChannelProcessor {
    
    @Autowired
    private EmailProviderFactory providerFactory;
    
    @Override
    public DeliveryResult process(NotificationMessage message) {
        EmailProvider provider = providerFactory.getProvider(message.getPriority());
        
        EmailContent content = templateEngine.render(
            message.getTemplateId(), 
            message.getVariables()
        );
        
        return provider.send(EmailRequest.builder()
            .to(message.getRecipient().getEmail())
            .subject(content.getSubject())
            .htmlBody(content.getBody())
            .priority(message.getPriority())
            .build());
    }
}
```

**Provider Failover Strategy**:
- Primary: AWS SES (high volume, cost-effective)
- Secondary: SendGrid (reliability backup)
- Tertiary: Mailgun (final fallback)

### SMS Service Implementation

```java
@Service
public class SmsChannelProcessor implements ChannelProcessor {
    
    @Override
    public DeliveryResult process(NotificationMessage message) {
        // Route based on country code for optimal delivery rates
        SmsProvider provider = routingService.selectProvider(
            message.getRecipient().getPhoneNumber()
        );
        
        SmsContent content = templateEngine.render(
            message.getTemplateId(),
            message.getVariables()
        );
        
        return provider.send(SmsRequest.builder()
            .to(message.getRecipient().getPhoneNumber())
            .message(content.getMessage())
            .build());
    }
}
```

**Interview Insight**: SMS routing is geography-dependent. Different providers have better delivery rates in different regions. Discuss how you'd implement intelligent routing based on phone number analysis.

### WeChat Integration

WeChat requires special handling due to its ecosystem:

```java
@Service 
public class WeChatChannelProcessor implements ChannelProcessor {
    
    @Override
    public DeliveryResult process(NotificationMessage message) {
        // WeChat template messages have strict formatting requirements
        WeChatTemplate template = weChatTemplateService.getTemplate(
            message.getTemplateId()
        );
        
        WeChatMessage weChatMessage = WeChatMessage.builder()
            .openId(message.getRecipient().getWeChatOpenId())
            .templateId(template.getWeChatTemplateId())
            .data(transformVariables(message.getVariables()))
            .build();
            
        return weChatApiClient.sendTemplateMessage(weChatMessage);
    }
}
```

## Scheduling and Delivery Management

### Scheduler Service Architecture

{% mermaid flowchart TD %}
    A[Scheduled Messages] --> B[Time-based Partitioner]
    B --> C[Quartz Scheduler Cluster]
    C --> D[Message Trigger]
    D --> E{Delivery Window?}
    E -->|Yes| F[Send to Processing Queue]
    E -->|No| G[Reschedule]
    F --> H[Channel Processors]
    G --> A
{% endmermaid %}

**Delivery Window Management**:
- Timezone-aware scheduling
- Business hours enforcement
- Frequency capping to prevent spam

### Retry and Failure Handling

**Exponential Backoff Strategy**:
```java
@Component
public class RetryPolicyManager {
    
    public RetryPolicy getRetryPolicy(ChannelType channel, FailureReason reason) {
        return RetryPolicy.builder()
            .maxRetries(getMaxRetries(channel, reason))
            .initialDelay(Duration.ofSeconds(30))
            .backoffMultiplier(2.0)
            .maxDelay(Duration.ofHours(4))
            .jitter(0.1)
            .build();
    }
    
    private int getMaxRetries(ChannelType channel, FailureReason reason) {
        // Email: 3 retries for transient failures, 0 for invalid addresses
        // SMS: 2 retries for network issues, 0 for invalid numbers  
        // WeChat: 3 retries for API limits, 1 for user blocks
    }
}
```

**Interview Insight**: Discuss the importance of classifying failures - temporary vs permanent. Retrying an invalid email address wastes resources, while network timeouts should be retried with backoff.

## MessageNotificationSDK Design

### SDK Architecture

```java
@Component
public class MessageNotificationSDK {
    
    private final NotificationClient notificationClient;
    private final CircuitBreaker circuitBreaker;
    
    public CompletableFuture<MessageResult> sendNotification(NotificationRequest request) {
        return circuitBreaker.executeAsync(() -> 
            notificationClient.sendNotification(request)
        ).exceptionally(throwable -> {
            // Fallback: store in local queue for retry
            localQueueService.enqueue(request);
            return MessageResult.queued(request.getMessageId());
        });
    }
    
    public CompletableFuture<MessageResult> sendScheduledNotification(
        NotificationRequest request, 
        Instant scheduledTime
    ) {
        ScheduledNotificationRequest scheduledRequest = 
            ScheduledNotificationRequest.builder()
                .notificationRequest(request)
                .scheduledAt(scheduledTime)
                .build();
                
        return notificationClient.scheduleNotification(scheduledRequest);
    }
}
```

### SDK Configuration

```yaml
notification:
  client:
    baseUrl: https://notifications.company.com
    timeout: 30s
    retries: 3
  circuit-breaker:
    failure-threshold: 5
    recovery-timeout: 60s
  local-queue:
    enabled: true
    max-size: 1000
    flush-interval: 30s
```

**Interview Insight**: The SDK should be resilient to service unavailability. Discuss local queuing, circuit breakers, and graceful degradation strategies.

## Monitoring and Observability

### Key Metrics Dashboard

**Throughput Metrics**:
- Messages processed per second by channel
- Queue depth and processing latency
- Template rendering performance

**Delivery Metrics**:
- Delivery success rate by channel and provider
- Bounce and failure rates
- Time to delivery distribution

**Business Metrics**:
- User engagement rates
- Opt-out rates by channel
- Cost per notification by channel

{% mermaid graph LR %}
    A[Notification Service] --> B[Metrics Collector]
    B --> C[Prometheus]
    C --> D[Grafana Dashboard]
    B --> E[Application Logs]
    E --> F[ELK Stack]
    B --> G[Distributed Tracing]
    G --> H[Jaeger]
{% endmermaid %}

### Alerting Strategy

**Critical Alerts**:
- Queue depth > 10,000 messages
- Delivery success rate < 95%
- Provider API failure rate > 5%

**Warning Alerts**:
- Processing latency > 30 seconds
- Template rendering errors
- Unusual bounce rate increases

## Security and Compliance

### Data Protection

**Encryption**:
- At-rest: AES-256 encryption for stored messages
- In-transit: TLS 1.3 for all API communications
- PII masking in logs and metrics

**Access Control**:
```java
@PreAuthorize("hasRole('NOTIFICATION_ADMIN') or hasPermission(#request.userId, 'SEND_NOTIFICATION')")
public MessageResult sendNotification(NotificationRequest request) {
    // Implementation
}
```

### Compliance Considerations

**GDPR Compliance**:
- Right to be forgotten: Automatic message deletion after retention period
- Consent management: Integration with preference center
- Data minimization: Only store necessary message data

**CAN-SPAM Act**:
- Automatic unsubscribe link injection
- Sender identification requirements
- Opt-out processing within 10 business days

**Interview Insight**: Security should be built-in, not bolted-on. Discuss defense in depth - encryption, authentication, authorization, input validation, and audit logging at every layer.

## Performance Benchmarks and Capacity Planning

### Load Testing Results

**Target Performance**:
- 10M messages/day = 115 messages/second average
- Peak capacity: 1,000 messages/second
- 99th percentile latency: < 100ms for API calls
- 95th percentile delivery time: < 30 seconds

**Scaling Calculations**:
```
Email Channel:
- Provider rate limit: 1000 emails/second
- Buffer factor: 2x for burst capacity
- Required instances: 2 (with failover)

SMS Channel:  
- Provider rate limit: 100 SMS/second
- Peak SMS load: ~200/second (20% of total)
- Required instances: 4 (with geographic distribution)
```

### Database Sizing

**Message Storage Requirements**:
- 10M messages/day × 2KB average size = 20GB/day
- 90-day retention = 1.8TB storage requirement
- With replication and indexes: 5TB total

## Cost Optimization Strategies

### Provider Cost Management

**Email Costs**:
- AWS SES: $0.10 per 1,000 emails
- SendGrid: $0.20 per 1,000 emails  
- Strategy: Primary on SES, failover to SendGrid

**SMS Costs**:
- Twilio US: $0.0075 per SMS
- International routing for cost optimization
- Bulk messaging discounts negotiation

**Infrastructure Costs**:
- Kafka cluster: $500/month
- Application servers: $800/month
- Database: $300/month
- Monitoring: $200/month
- **Total**: ~$1,800/month for 10M messages

**Interview Insight**: Always discuss cost optimization in system design. Show understanding of the business impact - a 10% improvement in delivery rates might justify 50% higher costs if it drives revenue.

## Testing Strategy

### Integration Testing

```java
@SpringBootTest
@TestcontainersConfiguration
class NotificationServiceIntegrationTest {
    
    @Test
    void shouldProcessHighVolumeNotifications() {
        // Simulate 1000 concurrent notification requests
        List<CompletableFuture<MessageResult>> futures = IntStream.range(0, 1000)
            .mapToObj(i -> notificationService.sendNotification(createTestRequest(i)))
            .collect(toList());
            
        CompletableFuture.allOf(futures.toArray(new CompletableFuture[0]))
            .join();
            
        // Verify all messages processed within SLA
        assertThat(futures).allSatisfy(future -> 
            assertThat(future.get().getStatus()).isEqualTo(DELIVERED)
        );
    }
}
```

### Chaos Engineering

**Failure Scenarios**:
- Provider API timeouts and rate limiting
- Database connection failures
- Kafka broker failures
- Network partitions between services

## Future Enhancements

### Advanced Features Roadmap

**Machine Learning Integration**:
- Optimal send time prediction per user
- Template A/B testing automation
- Delivery success rate optimization

**Rich Media Support**:
- Image and video attachments
- Interactive email templates
- Push notification rich media

**Advanced Analytics**:
- User engagement scoring
- Campaign performance analytics
- Predictive churn analysis

**Interview Insight**: Always end system design discussions with future considerations. This shows forward thinking and understanding that systems evolve. Discuss how your current architecture would accommodate these enhancements.

## Conclusion

This Message Notification Service design provides a robust, scalable foundation for high-volume, multi-channel notifications. The architecture emphasizes reliability, observability, and maintainability while meeting the 10 million messages per day requirement with room for growth.

Key design principles applied:
- **Decoupling**: Message queues separate concerns and enable independent scaling
- **Reliability**: Multiple failover mechanisms and retry strategies
- **Observability**: Comprehensive monitoring and alerting
- **Security**: Built-in encryption, access control, and compliance features
- **Cost Efficiency**: Provider optimization and resource right-sizing

The system can be deployed incrementally, starting with core notification functionality and adding advanced features as business needs evolve.