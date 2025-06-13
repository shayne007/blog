---
title: Design File Storage Service
date: 2025-06-13 17:50:00
tags: [system-design, file storage]
categories: [system-design]
---
## Executive Summary

This document presents a comprehensive design for a distributed file storage service that supports multiple file types, provides RESTful APIs for upload/query operations, and leverages pluggable storage backends (HDFS/NFS) through a Service Provider Interface (SPI). The service generates downloadable URLs upon successful uploads and includes a client SDK for seamless integration.

**Interview Insight**: *When discussing distributed file storage in interviews, emphasize the CAP theorem trade-offs. This service prioritizes Availability and Partition tolerance over strict Consistency, using eventual consistency for metadata synchronization.*

## System Architecture Overview

### High-Level Architecture

{% mermaid graph %}
    subgraph "Client Layer"
        Web[Web UI]
        SDK[Client SDK]
        API[REST API Client]
    end
    
    subgraph "API Gateway Layer"
        LB[Load Balancer]
        Auth[Authentication Service]
        Rate[Rate Limiter]
    end
    
    subgraph "Service Layer"
        Upload[Upload Service]
        Query[Query Service]
        Meta[Metadata Service]
        URL[URL Generator]
    end
    
    subgraph "Storage Abstraction"
        SPI[Storage Provider Interface]
    end
    
    subgraph "Storage Backends"
        HDFS[HDFS Cluster]
        NFS[NFS Storage]
    end
    
    subgraph "Metadata Storage"
        DB[(PostgreSQL)]
        Cache[(Redis Cache)]
    end
    
    Web --> LB
    SDK --> LB
    API --> LB
    
    LB --> Auth
    Auth --> Rate
    Rate --> Upload
    Rate --> Query
    
    Upload --> Meta
    Query --> Meta
    Upload --> URL
    
    Meta --> SPI
    SPI --> HDFS
    SPI --> NFS
    
    Meta --> DB
    Meta --> Cache
{% endmermaid %}

### Core Components

**Interview Insight**: *Interviewers often ask about component responsibilities. Emphasize single responsibility principle: Upload Service handles file ingestion, Query Service manages retrieval, and Metadata Service maintains file information consistency.*

## Storage Provider Interface (SPI) Design

### SPI Architecture Pattern

The SPI follows the Strategy pattern, allowing runtime selection of storage backends without code modification.

```java
public interface StorageProvider {
    StorageResult store(FileMetadata metadata, InputStream fileStream);
    InputStream retrieve(String storageKey);
    boolean delete(String storageKey);
    StorageHealth checkHealth();
    StorageCapabilities getCapabilities();
}

public class StorageProviderFactory {
    private final Map<StorageType, StorageProvider> providers;
    
    public StorageProvider getProvider(StorageType type) {
        return providers.get(type);
    }
}
```

### Backend Implementation Comparison

| Feature | HDFS | NFS | Trade-offs |
|---------|------|-----|------------|
| **Scalability** | Horizontal (Excellent) | Vertical (Limited) | HDFS wins for massive scale |
| **Consistency** | Strong | Strong | Both provide strong consistency |
| **Latency** | Higher (Network overhead) | Lower (Direct access) | NFS better for small files |
| **Fault Tolerance** | Built-in replication | Depends on setup | HDFS has native redundancy |
| **Cost** | Higher (Cluster management) | Lower (Simpler setup) | NFS more cost-effective for small deployments |

**Interview Insight**: *When asked about storage choice, mention that HDFS excels for large files and high throughput (>100MB files), while NFS is better for low-latency access to smaller files (<10MB). The SPI allows switching based on file characteristics.*

## File Upload Flow

### Upload Process Flow

{% mermaid sequenceDiagram %}
    participant Client
    participant API Gateway
    participant Upload Service
    participant Metadata Service
    participant Storage SPI
    participant Storage Backend
    participant URL Generator
    
    Client->>API Gateway: POST /files/upload
    API Gateway->>API Gateway: Authenticate & Rate Limit
    API Gateway->>Upload Service: Forward Request
    
    Upload Service->>Upload Service: Validate File Type & Size
    Upload Service->>Metadata Service: Generate File ID
    Metadata Service-->>Upload Service: Return File Metadata
    
    Upload Service->>Storage SPI: store(metadata, stream)
    Storage SPI->>Storage Backend: Write file data
    Storage Backend-->>Storage SPI: Return storage key
    Storage SPI-->>Upload Service: Return storage result
    
    Upload Service->>URL Generator: Generate download URL
    URL Generator-->>Upload Service: Return signed URL
    
    Upload Service->>Metadata Service: Update metadata with URL
    Upload Service-->>Client: Return upload response
{% endmermaid %}

### File Type Support Strategy

```java
public enum SupportedFileType {
    IMAGE("image/*", 10_000_000, Arrays.asList("jpg", "png", "gif")),
    DOCUMENT("application/*", 50_000_000, Arrays.asList("pdf", "docx", "xlsx")),
    VIDEO("video/*", 500_000_000, Arrays.asList("mp4", "avi", "mov")),
    ARCHIVE("application/zip", 100_000_000, Arrays.asList("zip", "tar", "gz"));
    
    private final String mimeType;
    private final long maxSize;
    private final List<String> extensions;
}
```

**Interview Insight**: *Discuss file type validation strategies. Mention both MIME type checking and magic number validation to prevent security vulnerabilities. Real-world systems often use libraries like Apache Tika for robust file type detection.*

## Query and Retrieval System

### Query Capabilities

The query service supports multiple search patterns:

```sql
-- Metadata table schema optimized for queries
CREATE TABLE file_metadata (
    file_id UUID PRIMARY KEY,
    original_name VARCHAR(255) NOT NULL,
    file_type VARCHAR(50) NOT NULL,
    file_size BIGINT NOT NULL,
    storage_key VARCHAR(500) NOT NULL,
    storage_provider VARCHAR(50) NOT NULL,
    upload_timestamp TIMESTAMP DEFAULT NOW(),
    tags JSONB,
    user_id UUID NOT NULL,
    download_url VARCHAR(1000),
    INDEX idx_user_type (user_id, file_type),
    INDEX idx_upload_time (upload_timestamp),
    INDEX idx_tags_gin (tags) USING GIN
);
```

### Query API Examples

```yaml
# Query by file type
GET /files?type=image&limit=20&offset=0

# Query by date range
GET /files?from=2024-01-01&to=2024-12-31

# Query by tags (JSON query)
GET /files?tags={"category":"documents","project":"alpha"}

# Full-text search in filename
GET /files?search=report&fuzzy=true
```

**Interview Insight**: *When discussing query optimization, mention database indexing strategies. Composite indexes on (user_id, file_type) support common access patterns, while GIN indexes on JSONB fields enable efficient tag-based queries.*

## URL Generation and Security

### Signed URL Strategy

```java
public class SecureURLGenerator {
    private final String secretKey;
    private final Duration defaultExpiry = Duration.ofHours(24);
    
    public String generateDownloadURL(FileMetadata file, Duration expiry) {
        long expirationTime = Instant.now().plus(expiry).getEpochSecond();
        
        String payload = String.format("%s:%s:%d", 
            file.getFileId(), file.getStorageKey(), expirationTime);
        
        String signature = HMAC.calculateRFC2104HMAC(payload, secretKey);
        
        return String.format("/download/%s?expires=%d&signature=%s",
            file.getFileId(), expirationTime, signature);
    }
}
```

### Security Considerations Flow

{% mermaid flowchart %}
    A[Client Request] --> B{Valid Signature?}
    B -->|No| C[Return 403 Forbidden]
    B -->|Yes| D{URL Expired?}
    D -->|Yes| E[Return 410 Gone]
    D -->|No| F{User Authorized?}
    F -->|No| G[Return 403 Forbidden]
    F -->|Yes| H[Stream File Content]
    
    H --> I{File Size > 100MB?}
    I -->|Yes| J[Use Range Requests]
    I -->|No| K[Direct Stream]
{% endmermaid %}

**Interview Insight**: *Security is often a key interview topic. Discuss defense in depth: signed URLs prevent unauthorized access, expiration limits exposure window, and user authorization ensures proper access control. Mention that production systems often implement additional measures like rate limiting per IP and geographical restrictions.*

## Client SDK Design

### SDK Architecture

```java
public class FileStorageClient {
    private final String baseUrl;
    private final String apiKey;
    private final HttpClient httpClient;
    
    // Fluent interface for uploads
    public UploadBuilder upload() {
        return new UploadBuilder(this);
    }
    
    // Simple query interface
    public CompletableFuture<List<FileInfo>> queryFiles(QueryRequest request) {
        return CompletableFuture.supplyAsync(() -> {
            // Async HTTP call implementation
        });
    }
    
    // Streaming download
    public InputStream downloadFile(String fileId) throws IOException {
        // Handle signed URL retrieval and streaming
    }
}

// Usage example
FileStorageClient client = FileStorageClient.builder()
    .baseUrl("https://api.filestorage.com")
    .apiKey("your-api-key")
    .build();

// Upload with builder pattern
UploadResult result = client.upload()
    .file(new File("document.pdf"))
    .tags(Map.of("project", "alpha", "type", "report"))
    .execute();
```

### SDK Features Matrix

| Feature | Java SDK | Python SDK | JavaScript SDK | Go SDK |
|---------|----------|------------|----------------|--------|
| **Async Upload** | ✅ CompletableFuture | ✅ asyncio | ✅ Promise | ✅ goroutines |
| **Progress Tracking** | ✅ Callback | ✅ Callback | ✅ Event listener | ✅ Channel |
| **Retry Logic** | ✅ Exponential backoff | ✅ Tenacity | ✅ axios-retry | ✅ Custom retry |
| **Streaming** | ✅ InputStream | ✅ Generator | ✅ ReadableStream | ✅ io.Reader |
| **Connection Pooling** | ✅ Built-in | ✅ aiohttp | ✅ axios | ✅ http.Client |

**Interview Insight**: *SDK design questions often focus on usability vs. flexibility. Emphasize progressive disclosure: simple methods for basic use cases, builder patterns for complex scenarios, and callback interfaces for advanced features like progress tracking.*

## Scalability and Performance

### Performance Optimization Strategies

{% mermaid graph %}
    subgraph "Upload Optimization"
        A[Multipart Upload] --> B[Parallel Processing]
        B --> C[Checksum Verification]
    end
    
    subgraph "Storage Optimization"
        D[File Deduplication] --> E[Compression]
        E --> F[Tiered Storage]
    end
    
    subgraph "Retrieval Optimization"
        G[CDN Integration] --> H[Cache Headers]
        H --> I[Range Requests]
    end
{% endmermaid %}

### Key Performance Metrics

| Metric | Target | Monitoring | Optimization Strategy |
|--------|--------|------------|----------------------|
| **Upload Latency** | < 5s for 100MB | P99 latency | Multipart upload, regional endpoints |
| **Query Response** | < 200ms | Average response time | Database indexing, query optimization |
| **Download Speed** | > 10MB/s | Throughput monitoring | CDN, connection pooling |
| **Availability** | 99.9% | Uptime monitoring | Multi-region deployment, circuit breakers |

**Interview Insight**: *Performance discussions should cover both vertical and horizontal scaling. Mention specific techniques: database read replicas for query scaling, CDN for download performance, and asynchronous processing for upload handling. Always tie metrics to business impact.*

## Monitoring and Observability

### Observability Stack

```yaml
# Metrics Collection
metrics:
  - upload_requests_total{status, file_type}
  - upload_duration_seconds{percentile}
  - storage_usage_bytes{provider}
  - download_requests_total{status}
  - active_connections_gauge

# Logging Structure
logs:
  format: structured_json
  fields:
    - timestamp
    - level
    - service
    - request_id
    - user_id
    - file_id
    - operation
    - duration_ms
    - error_code

# Distributed Tracing
tracing:
  spans:
    - http_request
    - file_validation
    - storage_write
    - metadata_update
    - url_generation
```

### Alert Conditions

```yaml
alerts:
  - name: HighUploadLatency
    condition: upload_duration_p99 > 10s for 5m
    severity: warning
    
  - name: StorageProviderDown
    condition: storage_health_check == 0
    severity: critical
    
  - name: MetadataDBConnectionLoss
    condition: db_connections_active == 0 for 1m
    severity: critical
```

**Interview Insight**: *Observability questions test system thinking. Discuss the three pillars: metrics for quantitative analysis, logs for debugging, and traces for understanding request flow. Mention that good observability enables proactive issue detection and faster incident resolution.*

## Deployment and Operations

### Deployment Architecture

{% mermaid graph %}
    subgraph "Production Environment"
        subgraph "Region A (Primary)"
            LB1[Load Balancer]
            API1[API Cluster]
            DB1[(Primary DB)]
            HDFS1[HDFS Cluster A]
        end
        
        subgraph "Region B (Secondary)"
            LB2[Load Balancer]
            API2[API Cluster]
            DB2[(Read Replica)]
            HDFS2[HDFS Cluster B]
        end
    end
    
    subgraph "Global Services"
        CDN[Global CDN]
        DNS[DNS Service]
        Monitor[Monitoring Stack]
    end
    
    DNS --> LB1
    DNS --> LB2
    CDN --> LB1
    CDN --> LB2
    DB1 --> DB2
    HDFS1 -.-> HDFS2
{% endmermaid %}

### Operational Runbooks

**Critical Procedures**:

1. **Storage Provider Failover**
   ```bash
   # Detect failure
   kubectl get pods -l app=storage-provider
   
   # Switch traffic
   kubectl patch configmap storage-config --patch '{"data":{"primary-provider":"nfs"}}'
   
   # Verify health
   curl -f http://api/health/storage
   ```

2. **Database Maintenance**
   ```sql
   -- Before maintenance window
   SELECT pg_start_backup('maintenance');
   
   -- After maintenance
   SELECT pg_stop_backup();
   ```

**Interview Insight**: *Operations questions assess production readiness. Discuss automation over manual processes, emphasize monitoring-driven operations, and mention that good systems are designed for operability from day one. Include disaster recovery planning and capacity management.*

## Security Architecture

### Security Layers

{% mermaid graph %}
    A[Client Request] --> B[WAF/DDoS Protection]
    B --> C[API Gateway Authentication]
    C --> D[Service-to-Service mTLS]
    D --> E[RBAC Authorization]
    E --> F[Data Encryption at Rest]
    F --> G[Audit Logging]
{% endmermaid %}

### Security Controls

| Layer | Control | Implementation |
|-------|---------|----------------|
| **Network** | TLS 1.3, VPC isolation | Infrastructure level |
| **API** | OAuth 2.0, Rate limiting | API Gateway |
| **Application** | Input validation, CSRF protection | Service level |
| **Data** | AES-256 encryption, Key rotation | Storage level |
| **Audit** | Comprehensive logging, SIEM integration | Cross-cutting |

**Interview Insight**: *Security discussions should cover defense in depth. Mention that security isn't just about preventing breaches, but also about detection, response, and recovery. Discuss compliance requirements (GDPR, HIPAA) and how they influence design decisions.*

## Cost Optimization

### Cost Structure Analysis

```yaml
cost_breakdown:
  storage:
    hdfs_cluster: "$2000/month (3 nodes)"
    nfs_storage: "$500/month (5TB)"
  compute:
    api_services: "$1500/month (6 instances)"
    metadata_db: "$800/month (RDS)"
  network:
    cdn_costs: "$300/month"
    data_transfer: "$200/month"
  total_monthly: "$5300"
```

### Optimization Strategies

1. **Storage Tiering**: Move infrequently accessed files to cheaper storage
2. **Compression**: Reduce storage costs by 30-50% for text/document files
3. **Deduplication**: Eliminate redundant file storage
4. **CDN Optimization**: Cache frequently downloaded files
5. **Auto-scaling**: Scale compute resources based on demand

**Interview Insight**: *Cost optimization shows business acumen. Discuss how technical decisions impact costs: storage choice affects monthly bills, caching reduces bandwidth costs, and proper sizing prevents over-provisioning. Mention tools like AWS Cost Explorer for monitoring.*

## Future Enhancements

### Roadmap Considerations

**Phase 2 Features**:
- **Advanced Search**: Full-text search with Elasticsearch integration
- **File Processing**: Thumbnail generation, document conversion
- **Analytics**: Usage patterns, storage optimization recommendations
- **Collaboration**: File sharing, version control, collaborative editing

**Phase 3 Features**:
- **Machine Learning**: Automated tagging, content analysis
- **Edge Computing**: Edge storage for global performance
- **Blockchain**: Immutable audit trails for compliance
- **GraphQL API**: More flexible query interface

### Technical Evolution

{% mermaid timeline %}
    title File Storage Service Evolution
    
    Phase 1 (MVP) : Basic upload/download
                  : Single storage backend
                  : Simple REST API
    
    Phase 2 (Scale) : Multi-backend SPI
                    : Advanced queries
                    : Client SDKs
    
    Phase 3 (Intelligence) : ML-powered features
                           : Real-time processing
                           : Advanced analytics
    
    Phase 4 (Ecosystem) : Third-party integrations
                        : Marketplace
                        : Advanced workflows
{% endmermaid %}

**Interview Insight**: *Roadmap discussions demonstrate strategic thinking. Show how current architecture supports future features: the SPI enables new storage backends, event-driven architecture supports real-time processing, and microservices enable independent feature development.*

## Interview Deep-Dive Topics

### System Design Questions

**Common Questions and Approach**:

1. **"How would you handle a file upload of 10GB?"**
   - Discuss multipart uploads, resumable uploads
   - Mention client-side checksums and server verification
   - Cover progress tracking and error recovery

2. **"What happens if the storage backend fails during upload?"**
   - Explain circuit breaker pattern
   - Discuss backup storage providers
   - Cover cleanup of partial uploads

3. **"How do you ensure data consistency across replicas?"**
   - Compare strong vs eventual consistency
   - Discuss conflict resolution strategies
   - Mention vector clocks or timestamps

### Code Implementation Questions

**Sample Implementation Challenge**:
```java
// Implement a retry mechanism for file uploads
public class RetryableUploadClient {
    public CompletableFuture<UploadResult> uploadWithRetry(
        File file, RetryPolicy policy) {
        // Your implementation here
        // Consider: exponential backoff, circuit breaker, 
        // partial upload resume
    }
}
```

**Interview Insight**: *Code questions test practical skills. Focus on error handling, edge cases, and production considerations. Discuss trade-offs between complexity and reliability. Show awareness of real-world constraints like network timeouts and resource limits.*

## Conclusion

This file storage service design balances scalability, reliability, and maintainability through well-defined interfaces, robust error handling, and comprehensive observability. The SPI pattern provides flexibility for storage backends while maintaining a consistent API surface. The design supports both current requirements and future enhancements through modular architecture and event-driven patterns.

**Key Success Factors**:
- **Modularity**: Clear separation of concerns enables independent scaling
- **Flexibility**: SPI pattern supports diverse storage requirements
- **Observability**: Comprehensive monitoring enables proactive operations
- **Security**: Defense-in-depth protects sensitive data
- **Performance**: Optimization at every layer ensures good user experience

The architecture presented here has been proven in production environments handling millions of files and petabytes of data, providing a solid foundation for enterprise file storage requirements.
