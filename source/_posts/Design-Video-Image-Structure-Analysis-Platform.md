---
title: Design Video Image Structure Analysis Platform
date: 2025-06-16 19:28:03
tags: [system-design, structure analysis platform]
categories: [system-design]
---

## System Architecture Overview

The Video Image Structure Analysis Platform is designed as a cloud-native, microservices-based system that processes video streams and images to extract structured metadata about objects like people, vehicles, bikes, and motorbikes. The architecture emphasizes scalability, resilience, and real-time processing capabilities.

{% mermaid graph TB %}
    subgraph "Data Ingestion Layer"
        Camera[Cameras/Video Sources]
        StreamService[StreamingAccessService]
        Camera --> StreamService
    end
    
    subgraph "Processing Layer"
        StructureApp[StructureAppService]
        TaskManager[TaskManagerService]
        StreamService --> StructureApp
        TaskManager --> StructureApp
    end
    
    subgraph "Message Queue"
        Kafka[Apache Kafka]
        StructureApp --> Kafka
    end
    
    subgraph "Storage Layer"
        ElasticSearch[ElasticSearch]
        FastDFS[FastDFS]
        StorageService[StorageAndSearchService]
        Kafka --> StorageService
        StorageService --> ElasticSearch
        StorageService --> FastDFS
    end
    
    subgraph "Application Layer"
        PlatformAPI[AnalysisPlatformService]
        WebUI[AnalysisPlatformUI]
        StorageService --> PlatformAPI
        PlatformAPI --> WebUI
    end
    
    subgraph "Infrastructure"
        Redis[Redis Cache]
        Zookeeper[Zookeeper]
        PlatformAPI --> Redis
        TaskManager --> Zookeeper
    end
{% endmermaid %}

**Interview Insight**: When designing distributed systems, the separation of concerns is crucial. Each service has a single responsibility, and communication patterns are well-defined through message queues and APIs.

## Core Services Architecture

### StreamingAccessService

The StreamingAccessService acts as the gateway for all video inputs, managing camera connections and stream processing.

**Key Responsibilities:**
- Real-time RTMP/RTSP stream ingestion from multiple cameras
- Video transcoding and format normalization
- Frame extraction and batching for analysis
- Load balancing across multiple camera feeds
- Geographic distribution management with camera mapping

**Technical Implementation:**
```java
@Service
@Component
public class StreamingAccessService {
    
    @Autowired
    private RedisTemplate<String, Object> redisTemplate;
    
    public void processVideoStream(String cameraId, VideoStream stream) {
        // Frame extraction logic
        List<Frame> frames = extractFrames(stream);
        
        // Cache camera metadata
        cacheManager.put("camera:" + cameraId, getCameraMetadata(cameraId));
        
        // Send frames for analysis
        structureAppService.analyzeFrames(frames, cameraId);
    }
}
```

**Scaling Strategy**: Use horizontal pod autoscaling based on CPU and memory metrics, with dedicated nodes for video processing to handle I/O intensive operations.

### StructureAppService

This GPU-accelerated service performs the core computer vision analysis using deep learning models.

**ML Pipeline Architecture:**
{% mermaid flowchart LR %}
    Input[Frame Input] --> Preprocess[Preprocessing]
    Preprocess --> Detection[Object Detection]
    Detection --> Classification[Classification]
    Classification --> Attribution[Attribute Extraction]
    Attribution --> Output[JSON Output]
    
    subgraph "Models"
        YOLO[YOLO v8]
        ResNet[ResNet-50]
        Custom[Custom Models]
    end
    
    Detection --> YOLO
    Classification --> ResNet
    Attribution --> Custom
{% endmermaid %}

**Object Analysis Specifications:**

*Person Attributes:*
- Age estimation (age ranges: 0-12, 13-17, 18-30, 31-50, 51-70, 70+)
- Gender classification (male, female, unknown)
- Height estimation using reference objects
- Clothing color detection (top, bottom)
- Body size estimation (small, medium, large)
- Pose estimation for activity recognition

*Vehicle Attributes:*
- License plate recognition using OCR
- Vehicle type classification (sedan, SUV, truck, bus)
- Color detection using color histograms
- Brand recognition using CNN models
- Seatbelt detection for driver safety compliance

**Interview Insight**: GPU resource management is critical. Implement batch processing to maximize GPU utilization and use model optimization techniques like TensorRT for inference acceleration.

### TaskManagerService

Coordinates analysis tasks across the distributed system using Zookeeper for consensus and resource management.

**Resource Management:**
```java
@Service
public class TaskManagerService {
    
    @Autowired
    private CuratorFramework zookeeperClient;
    
    public void scheduleAnalysisTask(AnalysisTask task) {
        // Get available nodes from Zookeeper
        List<String> availableNodes = getAvailableNodes();
        
        // Select optimal node based on resources
        String selectedNode = selectNodeByResources(availableNodes, task);
        
        // Create distributed lock for task assignment
        InterProcessMutex lock = new InterProcessMutex(zookeeperClient, 
            "/tasks/locks/" + task.getId());
        
        try {
            lock.acquire();
            assignTaskToNode(task, selectedNode);
        } finally {
            lock.release();
        }
    }
}
```

**Zookeeper Data Structure:**
- `/nodes/available` - List of active processing nodes
- `/nodes/resources` - Current resource utilization
- `/tasks/pending` - Queue of pending analysis tasks
- `/tasks/processing` - Currently executing tasks
- `/cameras/registry` - Camera registration and metadata

### StorageAndSearchService

Manages structured data storage and provides advanced search capabilities.

**ElasticSearch Index Mapping:**

```json
{
  "person_index": {
    "mappings": {
      "properties": {
        "timestamp": {"type": "date"},
        "camera_id": {"type": "keyword"},
        "location": {"type": "geo_point"},
        "person_id": {"type": "keyword"},
        "age_range": {"type": "keyword"},
        "gender": {"type": "keyword"},
        "height_cm": {"type": "integer"},
        "clothing": {
          "properties": {
            "top_color": {"type": "keyword"},
            "bottom_color": {"type": "keyword"},
            "style": {"type": "text"}
          }
        },
        "body_size": {"type": "keyword"},
        "image_path": {"type": "keyword"},
        "confidence_score": {"type": "float"},
        "feature_vector": {
          "type": "dense_vector",
          "dims": 512
        }
      }
    }
  }
}
```

```json
{
  "vehicle_index": {
    "mappings": {
      "properties": {
        "timestamp": {"type": "date"},
        "camera_id": {"type": "keyword"},
        "location": {"type": "geo_point"},
        "vehicle_id": {"type": "keyword"},
        "license_plate": {"type": "keyword"},
        "vehicle_type": {"type": "keyword"},
        "color": {"type": "keyword"},
        "brand": {"type": "keyword"},
        "model": {"type": "text"},
        "driver_info": {
          "properties": {
            "seatbelt_detected": {"type": "boolean"},
            "phone_usage": {"type": "boolean"}
          }
        },
        "image_path": {"type": "keyword"},
        "confidence_score": {"type": "float"},
        "feature_vector": {
          "type": "dense_vector",
          "dims": 512
        }
      }
    }
  }
}
```

**FastDFS Integration:**
```java
@Service
public class ImageStorageService {
    
    public String storeImage(byte[] imageData, String objectType) {
        // Generate unique filename
        String filename = generateUniqueFilename(objectType);
        
        // Store in FastDFS
        String fileId = fastDFSClient.uploadFile(imageData, "jpg", null);
        
        // Cache mapping in Redis for quick access
        redisTemplate.opsForValue().set("image:" + filename, fileId, 
            Duration.ofHours(24));
        
        return fileId;
    }
}
```

**Interview Insight**: When designing search indices, consider both exact matches and fuzzy searches. Use appropriate analyzers for text fields and optimize for common query patterns.

## API Design

### RESTful API Endpoints

**Task Management APIs:**
```http
POST /api/v1/tasks
GET /api/v1/tasks?status={status}&camera_id={id}
GET /api/v1/tasks/{taskId}
PUT /api/v1/tasks/{taskId}
DELETE /api/v1/tasks/{taskId}
```

**Camera Management APIs:**
```http
POST /api/v1/cameras
GET /api/v1/cameras?location={lat,lng}&radius={km}
GET /api/v1/cameras/{cameraId}
PUT /api/v1/cameras/{cameraId}
DELETE /api/v1/cameras/{cameraId}
GET /api/v1/cameras/{cameraId}/stream
```

**Search APIs:**
```http
POST /api/v1/search/persons
POST /api/v1/search/vehicles
POST /api/v1/search/similarity
GET /api/v1/search/export?format={json|csv}
```

**Real-time APIs:**
```http
GET /api/v1/stream/cameras/{cameraId}/ws
GET /api/v1/alerts/ws
```

### API Request/Response Examples

**Person Search Request:**
```json
{
  "filters": {
    "age_range": ["18-30", "31-50"],
    "gender": "male",
    "clothing": {
      "top_color": "blue",
      "bottom_color": "black"
    },
    "time_range": {
      "start": "2024-01-01T00:00:00Z",
      "end": "2024-01-31T23:59:59Z"
    },
    "location": {
      "center": {"lat": 40.7128, "lng": -74.0060},
      "radius": "5km"
    }
  },
  "pagination": {
    "page": 1,
    "size": 20
  },
  "sort": [
    {"field": "timestamp", "order": "desc"},
    {"field": "confidence_score", "order": "desc"}
  ]
}
```

**Similarity Search Request:**
```json
{
  "image": "base64_encoded_image_data",
  "object_type": "person",
  "similarity_threshold": 0.8,
  "max_results": 50,
  "time_range": {
    "start": "2024-01-01T00:00:00Z",
    "end": "2024-01-31T23:59:59Z"
  }
}
```

## Message Queue Architecture

### Kafka Topic Design

**Topic Structure:**
- `video-frames`: Raw frame data for analysis
- `analysis-results`: Structured analysis output
- `alerts`: Real-time alerts and notifications
- `system-metrics`: Performance and health metrics

**Message Schema (Avro):**
```json
{
  "type": "record",
  "name": "AnalysisResult",
  "fields": [
    {"name": "messageId", "type": "string"},
    {"name": "timestamp", "type": "long"},
    {"name": "cameraId", "type": "string"},
    {"name": "frameId", "type": "string"},
    {"name": "objects", "type": {
      "type": "array",
      "items": {
        "type": "record",
        "name": "DetectedObject",
        "fields": [
          {"name": "objectType", "type": "string"},
          {"name": "boundingBox", "type": "string"},
          {"name": "confidence", "type": "float"},
          {"name": "attributes", "type": "string"}
        ]
      }
    }}
  ]
}
```

**Interview Insight**: Schema evolution is important in production systems. Use Avro or Protocol Buffers for backward compatibility when message formats change.

## Microservices Configuration

### Spring Cloud Alibaba Setup

**Application Configuration:**
```yaml
spring:
  application:
    name: video-analysis-platform
  cloud:
    nacos:
      discovery:
        server-addr: ${NACOS_SERVER:localhost:8848}
      config:
        server-addr: ${NACOS_SERVER:localhost:8848}
        file-extension: yaml
    sentinel:
      transport:
        dashboard: ${SENTINEL_DASHBOARD:localhost:8080}
    gateway:
      routes:
        - id: streaming-service
          uri: lb://streaming-access-service
          predicates:
            - Path=/api/v1/streams/**
        - id: analysis-service
          uri: lb://structure-app-service
          predicates:
            - Path=/api/v1/analysis/**
```

**Service Discovery:**
```java
@SpringBootApplication
@EnableDiscoveryClient
@EnableCircuitBreaker
public class VideoAnalysisPlatformApplication {
    
    @Bean
    @LoadBalanced
    public RestTemplate restTemplate() {
        return new RestTemplate();
    }
    
    @Bean
    public Customizer<Resilience4JCircuitBreakerFactory> globalCustomConfiguration() {
        return factory -> factory.configureDefault(id -> new Resilience4JConfigBuilder(id)
            .timeLimiterConfig(TimeLimiterConfig.custom()
                .timeoutDuration(Duration.ofSeconds(10))
                .build())
            .circuitBreakerConfig(CircuitBreakerConfig.custom()
                .failureRateThreshold(50)
                .waitDurationInOpenState(Duration.ofMillis(1000))
                .slidingWindowSize(2)
                .build())
            .build());
    }
}
```

## Frontend Architecture

### Modern React-based UI

**Key Components:**
- Camera Dashboard with real-time video feeds
- Interactive map showing camera locations
- Advanced search interface with filtering
- Task management console
- Real-time alerts and notifications
- Analytics dashboard with charts and metrics

**Technology Stack:**
- React 18 with TypeScript
- Material-UI for component library
- React Query for state management
- Leaflet.js for mapping
- Socket.io for real-time updates
- Chart.js for data visualization

**Map Integration Example:**
```jsx
const CameraMap = () => {
  const [cameras, setCameras] = useState([]);
  const [selectedCamera, setSelectedCamera] = useState(null);
  
  useEffect(() => {
    const socket = io('/camera-updates');
    socket.on('camera-status', (data) => {
      setCameras(prev => updateCameraStatus(prev, data));
    });
    
    return () => socket.disconnect();
  }, []);
  
  return (
    <MapContainer center={[40.7128, -74.0060]} zoom={13}>
      <TileLayer url="https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png" />
      {cameras.map(camera => (
        <Marker 
          key={camera.id} 
          position={[camera.lat, camera.lng]}
          onClick={() => setSelectedCamera(camera)}
        >
          <Popup>
            <CameraDetails camera={camera} />
          </Popup>
        </Marker>
      ))}
    </MapContainer>
  );
};
```

## Deployment and DevOps

### Docker Configuration

**Multi-stage Dockerfile for Analysis Service:**
```dockerfile
FROM nvidia/cuda:11.8-devel-ubuntu20.04 as builder
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

FROM nvidia/cuda:11.8-runtime-ubuntu20.04
WORKDIR /app
COPY --from=builder /usr/local/lib/python3.8/site-packages /usr/local/lib/python3.8/site-packages
COPY . .
EXPOSE 8080
CMD ["python", "app.py"]
```

**Kubernetes Deployment:**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: structure-app-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: structure-app-service
  template:
    metadata:
      labels:
        app: structure-app-service
    spec:
      containers:
      - name: structure-app
        image: video-analysis/structure-app:latest
        resources:
          requests:
            nvidia.com/gpu: 1
            memory: "4Gi"
            cpu: "2"
          limits:
            nvidia.com/gpu: 1
            memory: "8Gi"
            cpu: "4"
        env:
        - name: KAFKA_BROKERS
          value: "kafka-cluster:9092"
        - name: REDIS_URL
          value: "redis-cluster:6379"
```

## Performance Optimization

### Caching Strategy

**Multi-level Caching:**
- L1: Application-level cache using Caffeine
- L2: Redis distributed cache for shared data
- L3: ElasticSearch query result caching

**Redis Cache Patterns:**
```java
@Service
public class CacheService {
    
    @Cacheable(value = "camera-metadata", key = "#cameraId")
    public CameraMetadata getCameraMetadata(String cameraId) {
        return cameraRepository.findById(cameraId);
    }
    
    @CacheEvict(value = "search-results", allEntries = true)
    public void evictSearchCache() {
        // Triggered when new analysis results are added
    }
}
```

### Database Optimization

**ElasticSearch Performance Tuning:**
- Use index templates for consistent mapping
- Implement index lifecycle management (ILM)
- Configure appropriate shard and replica counts
- Use bulk operations for high-throughput indexing

**Query Optimization:**
```json
{
  "query": {
    "bool": {
      "filter": [
        {"term": {"camera_id": "camera_001"}},
        {"range": {"timestamp": {"gte": "2024-01-01", "lte": "2024-01-31"}}}
      ],
      "must": [
        {"match": {"clothing.top_color": "blue"}}
      ]
    }
  },
  "sort": [{"timestamp": {"order": "desc"}}],
  "size": 20
}
```

## Monitoring and Observability

### Comprehensive Monitoring Stack

**Key Metrics:**
- Frame processing throughput (frames/second)
- Analysis accuracy and confidence scores
- System resource utilization (CPU, GPU, memory)
- API response times and error rates
- Kafka message lag and throughput

**Prometheus Metrics:**
```java
@Component
public class MetricsCollector {
    
    private final Counter framesProcessed = Counter.build()
        .name("frames_processed_total")
        .help("Total number of frames processed")
        .labelNames("camera_id", "status")
        .register();
    
    private final Histogram analysisLatency = Histogram.build()
        .name("analysis_latency_seconds")
        .help("Time taken for frame analysis")
        .register();
    
    public void recordFrameProcessed(String cameraId, String status) {
        framesProcessed.labels(cameraId, status).inc();
    }
}
```

**Interview Insight**: Observability is crucial in ML systems. Track both technical metrics (latency, throughput) and business metrics (accuracy, detection rates) to understand system health and model performance.

## Security Considerations

### Multi-layered Security Approach

**Authentication and Authorization:**
- JWT-based authentication with refresh tokens
- Role-based access control (RBAC)
- API rate limiting and throttling
- Secure video stream transmission using HTTPS/WSS

**Data Protection:**
- Encryption at rest for stored images and metadata
- PII anonymization for privacy compliance
- Audit logging for all system operations
- Network segmentation between services

**Security Configuration:**
```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(authz -> authz
                .requestMatchers("/api/v1/public/**").permitAll()
                .requestMatchers("/api/v1/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(jwt -> jwt.jwtDecoder(jwtDecoder()))
            );
        return http.build();
    }
}
```

## Scalability and Resilience

### Horizontal Scaling Strategies

**Auto-scaling Configuration:**
- HPA based on CPU, memory, and custom metrics
- Vertical Pod Autoscaler for optimal resource allocation
- Cluster autoscaling for node management
- Geographic distribution for disaster recovery

**Resilience Patterns:**
- Circuit breaker for external service calls
- Retry mechanisms with exponential backoff
- Bulkhead isolation for critical components
- Graceful degradation during system overload

**Load Balancing:**
```yaml
apiVersion: v1
kind: Service
metadata:
  name: analysis-platform-service
spec:
  selector:
    app: analysis-platform
  ports:
  - port: 80
    targetPort: 8080
  type: LoadBalancer
  sessionAffinity: ClientIP
```

## Production Deployment Checklist

**Pre-deployment Validation:**
- Model accuracy testing with validation datasets
- Performance benchmarking under expected load
- Security penetration testing
- Disaster recovery procedure validation
- Database backup and restore testing

**Monitoring Setup:**
- Alert rules for critical system metrics
- Log aggregation and analysis
- Performance dashboards
- Health check endpoints
- SLA monitoring and reporting

**Interview Insight**: Production readiness involves more than just functional requirements. Consider operational aspects like monitoring, alerting, backup strategies, and incident response procedures from the design phase.

This Video Image Structure Analysis Platform provides a robust, scalable solution for real-time video analysis with advanced object detection and attribute extraction capabilities. The microservices architecture ensures maintainability and scalability, while the comprehensive monitoring and security measures make it production-ready for large-scale deployments.