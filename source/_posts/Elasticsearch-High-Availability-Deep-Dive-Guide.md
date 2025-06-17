---
title: 'Elasticsearch High Availability: Deep Dive Guide'
date: 2025-06-17 17:08:58
tags: [elasticsearch]
categories: [elasticsearch]
---

## Shards and Replicas: The Foundation of Elasticsearch HA

### Understanding Shards

Shards are the fundamental building blocks of Elasticsearch's distributed architecture. Each index is divided into multiple shards, which are essentially independent Lucene indices that can be distributed across different nodes in a cluster.

**Primary Shards:**
- Store the original data
- Handle write operations
- Number is fixed at index creation time
- Cannot be changed without reindexing

**Shard Sizing Best Practices:**
```json
PUT /my_index
{
  "settings": {
    "number_of_shards": 3,
    "number_of_replicas": 2,
    "index.routing.allocation.total_shards_per_node": 2
  }
}
```

### Replica Strategy for High Availability

Replicas are exact copies of primary shards that provide both redundancy and increased read throughput.

**Production Replica Configuration:**
```json
PUT /production_logs
{
  "settings": {
    "number_of_shards": 5,
    "number_of_replicas": 2,
    "index.refresh_interval": "30s",
    "index.translog.durability": "request"
  }
}
```

### Real-World Example: E-commerce Platform

Consider an e-commerce platform handling 1TB of product data:

```json
PUT /products
{
  "settings": {
    "number_of_shards": 10,
    "number_of_replicas": 1,
    "index.routing.allocation.require.box_type": "hot"
  },
  "mappings": {
    "properties": {
      "product_id": {"type": "keyword"},
      "name": {"type": "text"},
      "price": {"type": "double"},
      "category": {"type": "keyword"}
    }
  }
}
```

{% mermaid graph TB %}
    subgraph "Node 1"
        P1[Primary Shard 1]
        R2[Replica Shard 2]
        R3[Replica Shard 3]
    end
    
    subgraph "Node 2"
        P2[Primary Shard 2]
        R1[Replica Shard 1]
        R4[Replica Shard 4]
    end
    
    subgraph "Node 3"
        P3[Primary Shard 3]
        P4[Primary Shard 4]
        R5[Replica Shard 5]
    end
    
    P1 -.->|Replicates to| R1
    P2 -.->|Replicates to| R2
    P3 -.->|Replicates to| R3
    P4 -.->|Replicates to| R4
{% endmermaid %}

**Interview Insight:** *"How would you determine the optimal number of shards for a 500GB index with expected 50% growth annually?"*

**Answer:** Calculate based on shard size (aim for 10-50GB per shard), consider node capacity, and factor in growth. For 500GB growing to 750GB: 15-75 shards initially, typically 20-30 shards with 1-2 replicas.

## TransLog: Ensuring Write Durability

### TransLog Mechanism

The Transaction Log (TransLog) is Elasticsearch's write-ahead log that ensures data durability during unexpected shutdowns or power failures.

**How TransLog Works:**
1. Write operation received
2. Data written to in-memory buffer
3. Operation logged to TransLog
4. Acknowledgment sent to client
5. Periodic flush to Lucene segments

### TransLog Configuration for High Availability

```json
PUT /critical_data
{
  "settings": {
    "index.translog.durability": "request",
    "index.translog.sync_interval": "5s",
    "index.translog.flush_threshold_size": "512mb",
    "index.refresh_interval": "1s"
  }
}
```

**TransLog Durability Options:**
- `request`: Fsync after each request (highest durability, lower performance)
- `async`: Fsync every sync_interval (better performance, slight risk)

### Production Example: Financial Trading System

```json
PUT /trading_transactions
{
  "settings": {
    "number_of_shards": 3,
    "number_of_replicas": 2,
    "index.translog.durability": "request",
    "index.translog.sync_interval": "1s",
    "index.refresh_interval": "1s",
    "index.translog.retention.size": "1gb",
    "index.translog.retention.age": "12h"
  }
}
```

{% mermaid sequenceDiagram %}
    participant Client
    participant ES_Node
    participant TransLog
    participant Lucene
    
    Client->>ES_Node: Index Document
    ES_Node->>TransLog: Write to TransLog
    TransLog-->>ES_Node: Confirm Write
    ES_Node->>Lucene: Add to In-Memory Buffer
    ES_Node-->>Client: Acknowledge Request
    
    Note over ES_Node: Periodic Refresh
    ES_Node->>Lucene: Flush Buffer to Segment
    ES_Node->>TransLog: Clear TransLog Entries
{% endmermaid %}

**Interview Insight:** *"What happens if a node crashes between TransLog write and Lucene flush?"*

**Answer:** On restart, Elasticsearch replays TransLog entries to recover uncommitted operations. The TransLog ensures no acknowledged writes are lost, maintaining data consistency.

## Production HA Challenges and Solutions

### Common Production Issues

#### Split-Brain Syndrome
**Problem:** Network partitions causing multiple master nodes

**Solution:**
```yaml
# elasticsearch.yml
discovery.zen.minimum_master_nodes: 2  # (total_masters / 2) + 1
cluster.initial_master_nodes: ["node-1", "node-2", "node-3"]
```

#### Memory Pressure and GC Issues
**Problem:** Large heaps causing long GC pauses

**Solution:**
```yaml
# jvm.options
-Xms16g
-Xmx16g
-XX:+UseG1GC
-XX:MaxGCPauseMillis=200
```

#### Uneven Shard Distribution
**Problem:** Hot spots on specific nodes

**Solution:**
```json
PUT /_cluster/settings
{
  "transient": {
    "cluster.routing.allocation.balance.shard": 0.45,
    "cluster.routing.allocation.balance.index": 0.55,
    "cluster.routing.allocation.balance.threshold": 1.0
  }
}
```

### Real Production Case Study: Log Analytics Platform

**Challenge:** Processing 100GB/day of application logs with strict SLA requirements

**Architecture:**
{% mermaid graph LR %}
    subgraph "Hot Tier"
        H1[Hot Node 1]
        H2[Hot Node 2]
        H3[Hot Node 3]
    end
    
    subgraph "Warm Tier"
        W1[Warm Node 1]
        W2[Warm Node 2]
    end
    
    subgraph "Cold Tier"
        C1[Cold Node 1]
    end
    
    Apps[Applications] --> LB[Load Balancer]
    LB --> H1
    LB --> H2
    LB --> H3
    
    H1 -.->|Age-based| W1
    H2 -.->|Migration| W2
    W1 -.->|Archive| C1
    W2 -.->|Archive| C1
{% endmermaid %}

**Index Template Configuration:**
```json
PUT /_index_template/logs_template
{
  "index_patterns": ["logs-*"],
  "template": {
    "settings": {
      "number_of_shards": 1,
      "number_of_replicas": 1,
      "index.lifecycle.name": "logs_policy",
      "index.routing.allocation.require.box_type": "hot"
    }
  }
}
```

**Interview Insight:** *"How would you handle a scenario where your Elasticsearch cluster is experiencing high write latency?"*

**Answer:** 
1. Check TransLog settings (reduce durability if acceptable)
2. Optimize refresh intervals
3. Implement bulk indexing
4. Scale horizontally by adding nodes
5. Consider index lifecycle management

## Optimization Strategies for Production HA

### Rate Limiting Implementation

**Circuit Breaker Pattern:**
```java
public class ElasticsearchCircuitBreaker {
    private final CircuitBreaker circuitBreaker;
    private final ElasticsearchClient client;
    
    public CompletableFuture<IndexResponse> indexWithRateLimit(
            IndexRequest request) {
        return circuitBreaker.executeSupplier(() -> {
            return client.index(request);
        });
    }
}
```

**Cluster-level Rate Limiting:**
```json
PUT /_cluster/settings
{
  "transient": {
    "indices.memory.index_buffer_size": "20%",
    "indices.memory.min_index_buffer_size": "96mb",
    "thread_pool.write.queue_size": 1000
  }
}
```

### Message Queue Peak Shaving

**Kafka Integration Example:**
```java
@Component
public class ElasticsearchBulkProcessor {
    
    @KafkaListener(topics = "elasticsearch-queue")
    public void processBulkData(List<String> documents) {
        BulkRequest bulkRequest = new BulkRequest();
        
        documents.forEach(doc -> {
            bulkRequest.add(new IndexRequest("logs")
                .source(doc, XContentType.JSON));
        });
        
        BulkResponse response = client.bulk(bulkRequest, RequestOptions.DEFAULT);
        handleBulkResponse(response);
    }
}
```

**MQ Configuration for Peak Shaving:**
```yaml
spring:
  kafka:
    consumer:
      max-poll-records: 500
      fetch-max-wait: 1000ms
    producer:
      batch-size: 65536
      linger-ms: 100
```

### Single Role Node Architecture

**Dedicated Master Nodes:**
```yaml
# master.yml
node.roles: [master]
discovery.seed_hosts: ["master-1", "master-2", "master-3"]
cluster.initial_master_nodes: ["master-1", "master-2", "master-3"]
```

**Data Nodes Configuration:**
```yaml
# data.yml
node.roles: [data, data_content, data_hot, data_warm]
path.data: ["/data1", "/data2", "/data3"]
```

**Coordinating Nodes:**
```yaml
# coordinator.yml
node.roles: []
http.port: 9200
```

### Dual Cluster Deployment Strategy

**Active-Passive Setup:**
{% mermaid graph TB %}
    subgraph "Primary DC"
        P_LB[Load Balancer]
        P_C1[Cluster 1 Node 1]
        P_C2[Cluster 1 Node 2]
        P_C3[Cluster 1 Node 3]
        
        P_LB --> P_C1
        P_LB --> P_C2
        P_LB --> P_C3
    end
    
    subgraph "Secondary DC"
        S_C1[Cluster 2 Node 1]
        S_C2[Cluster 2 Node 2]
        S_C3[Cluster 2 Node 3]
    end
    
    P_C1 -.->|Cross Cluster Replication| S_C1
    P_C2 -.->|CCR| S_C2
    P_C3 -.->|CCR| S_C3
    
    Apps[Applications] --> P_LB
{% endmermaid %}

**Cross-Cluster Replication Setup:**
```json
PUT /_cluster/settings
{
  "persistent": {
    "cluster.remote.secondary": {
      "seeds": ["secondary-cluster:9300"],
      "transport.compress": true
    }
  }
}

PUT /primary_index/_ccr/follow
{
  "remote_cluster": "secondary",
  "leader_index": "primary_index"
}
```

## Advanced HA Monitoring and Alerting

### Key Metrics to Monitor

**Cluster Health Script:**
```bash
#!/bin/bash
CLUSTER_HEALTH=$(curl -s "localhost:9200/_cluster/health")
STATUS=$(echo $CLUSTER_HEALTH | jq -r '.status')

if [ "$STATUS" != "green" ]; then
    echo "ALERT: Cluster status is $STATUS"
    # Send notification
fi
```

**Critical Metrics:**
- Cluster status (green/yellow/red)
- Node availability
- Shard allocation status
- Memory usage and GC frequency
- Search and indexing latency
- TransLog size and flush frequency

### Alerting Configuration Example

```yaml
# alertmanager.yml
groups:
- name: elasticsearch
  rules:
  - alert: ElasticsearchClusterNotHealthy
    expr: elasticsearch_cluster_health_status{color="red"} == 1
    for: 0m
    labels:
      severity: critical
    annotations:
      summary: "Elasticsearch cluster health is RED"
      
  - alert: ElasticsearchNodeDown
    expr: up{job="elasticsearch"} == 0
    for: 1m
    labels:
      severity: warning
```

## Performance Tuning for HA

### Index Lifecycle Management

```json
PUT /_ilm/policy/production_policy
{
  "policy": {
    "phases": {
      "hot": {
        "actions": {
          "rollover": {
            "max_size": "10gb",
            "max_age": "1d"
          }
        }
      },
      "warm": {
        "min_age": "7d",
        "actions": {
          "allocate": {
            "require": {
              "box_type": "warm"
            }
          },
          "forcemerge": {
            "max_num_segments": 1
          }
        }
      },
      "cold": {
        "min_age": "30d",
        "actions": {
          "allocate": {
            "require": {
              "box_type": "cold"
            }
          }
        }
      }
    }
  }
}
```

### Hardware Recommendations

**Production Hardware Specs:**
- **CPU:** 16+ cores for data nodes
- **Memory:** 64GB+ RAM (50% for heap, 50% for filesystem cache)
- **Storage:** NVMe SSDs for hot data, SATA SSDs for warm/cold
- **Network:** 10Gbps+ for inter-node communication

## Interview Questions and Expert Answers

**Q: "How would you recover from a complete cluster failure?"**

**A:** 
1. Restore from snapshot if available
2. If no snapshots, recover using `elasticsearch-node` tool
3. Implement proper backup strategy going forward
4. Consider cross-cluster replication for future disasters

**Q: "Explain the difference between `index.refresh_interval` and TransLog flush."**

**A:** 
- `refresh_interval` controls when in-memory documents become searchable
- TransLog flush persists data to disk for durability
- Refresh affects search visibility, flush affects data safety

**Q: "How do you handle version conflicts in a distributed environment?"**

**A:** 
- Use optimistic concurrency control with version numbers
- Implement retry logic with exponential backoff
- Consider using `_seq_no` and `_primary_term` for more granular control

## Security Considerations for HA

### Authentication and Authorization

```yaml
# elasticsearch.yml
xpack.security.enabled: true
xpack.security.transport.ssl.enabled: true
xpack.security.http.ssl.enabled: true

xpack.security.authc.realms.native.native1:
  order: 0
```

**Role-Based Access Control:**
```json
PUT /_security/role/log_reader
{
  "indices": [
    {
      "names": ["logs-*"],
      "privileges": ["read", "view_index_metadata"]
    }
  ]
}
```

## Best Practices Summary

### Do's
- Always use odd number of master-eligible nodes (3, 5, 7)
- Implement proper monitoring and alerting
- Use index templates for consistent settings
- Regularly test disaster recovery procedures
- Implement proper backup strategies

### Don'ts
- Don't set heap size above 32GB
- Don't disable swap without proper configuration
- Don't ignore yellow cluster status
- Don't use default settings in production
- Don't forget to monitor disk space

## References and Additional Resources

- [Elasticsearch Official Documentation](https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html)
- [Elasticsearch Performance Tuning Guide](https://www.elastic.co/guide/en/elasticsearch/reference/current/tune-for-search-speed.html)
- [Elastic Cloud Architecture Best Practices](https://www.elastic.co/guide/en/cloud/current/ec-getting-started.html)
- [Production Deployment Considerations](https://www.elastic.co/guide/en/elasticsearch/reference/current/system-config.html)

---

*This guide provides a comprehensive foundation for implementing and maintaining highly available Elasticsearch clusters in production environments. Regular updates and testing of these configurations are essential for maintaining optimal performance and reliability.*