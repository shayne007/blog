---
title: Elasticsearch Query Performance Optimization Guide
date: 2025-06-17 17:10:10
tags: [elasticsearch]
categories: [elasticsearch]
---

## System-Level Optimizations

### Garbage Collection (GC) Tuning

Elasticsearch relies heavily on the JVM, making GC performance critical for query response times. Poor GC configuration can lead to query timeouts and cluster instability.

**Production Best Practices:**
- Use G1GC for heaps larger than 6GB: `-XX:+UseG1GC`
- Set heap size to 50% of available RAM, but never exceed 32GB
- Configure GC logging for monitoring: `-Xloggc:gc.log -XX:+PrintGCDetails`

```bash
# Optimal JVM settings for production
ES_JAVA_OPTS="-Xms16g -Xmx16g -XX:+UseG1GC -XX:MaxGCPauseMillis=200 -XX:+PrintGC -XX:+PrintGCTimeStamps"
```

**Interview Insight:** *"Why is 32GB the heap size limit?"* - Beyond 32GB, the JVM loses compressed OOPs (Ordinary Object Pointers), effectively doubling pointer sizes and reducing cache efficiency.

### Memory Management and Swappiness

Swapping to disk can destroy Elasticsearch performance, turning millisecond operations into second-long delays.

**Configuration Steps:**
1. Disable swap entirely: `sudo swapoff -a`
2. Configure swappiness: `vm.swappiness=1`
3. Enable memory locking in Elasticsearch:

```yaml
# elasticsearch.yml
bootstrap.memory_lock: true
```

**Production Example:**
```bash
# /etc/sysctl.conf
vm.swappiness=1
vm.max_map_count=262144

# Verify settings
sysctl vm.swappiness
sysctl vm.max_map_count
```

### File Descriptors Optimization

Elasticsearch requires numerous file descriptors for index files, network connections, and internal operations.

```bash
# /etc/security/limits.conf
elasticsearch soft nofile 65536
elasticsearch hard nofile 65536
elasticsearch soft nproc 4096
elasticsearch hard nproc 4096

# Verify current limits
ulimit -n
ulimit -u
```

**Monitoring Script:**
```bash
#!/bin/bash
# Check file descriptor usage
echo "Current FD usage: $(lsof -u elasticsearch | wc -l)"
echo "FD limit: $(ulimit -n)"
```

## Query Optimization Strategies

### Pagination Performance

Deep pagination is one of the most common performance bottlenecks in Elasticsearch applications.

#### Problem with Traditional Pagination

{% mermaid graph %}
    A[Client Request: from=10000, size=10] --> B[Elasticsearch Coordinator]
    B --> C[Shard 1: Fetch 10010 docs]
    B --> D[Shard 2: Fetch 10010 docs]
    B --> E[Shard 3: Fetch 10010 docs]
    C --> F[Coordinator: Sort 30030 docs]
    D --> F
    E --> F
    F --> G[Return 10 docs to client]
{% endmermaid %}

#### Solution 1: Scroll API

Best for processing large datasets sequentially:

```json
# Initial scroll request
POST /my_index/_search?scroll=1m
{
  "size": 1000,
  "query": {
    "match_all": {}
  }
}

# Subsequent scroll requests
POST /_search/scroll
{
  "scroll": "1m",
  "scroll_id": "your_scroll_id_here"
}
```

**Production Use Case:** Log processing pipeline handling millions of documents daily.

#### Solution 2: Search After API

Ideal for real-time pagination with live data:

```json
# First request
GET /my_index/_search
{
  "size": 10,
  "query": {
    "match": {
      "title": "elasticsearch"
    }
  },
  "sort": [
    {"timestamp": {"order": "desc"}},
    {"_id": {"order": "desc"}}
  ]
}

# Next page using search_after
GET /my_index/_search
{
  "size": 10,
  "query": {
    "match": {
      "title": "elasticsearch"
    }
  },
  "sort": [
    {"timestamp": {"order": "desc"}},
    {"_id": {"order": "desc"}}
  ],
  "search_after": ["2023-10-01T10:00:00Z", "doc_id_123"]
}
```

**Interview Insight:** *"When would you choose search_after over scroll?"* - Search_after is stateless and handles live data changes better, while scroll is more efficient for complete dataset processing.

### Bulk Operations Optimization

The `_bulk` API significantly reduces network overhead and improves indexing performance.

#### Bulk API Best Practices

```json
POST /_bulk
{"index": {"_index": "my_index", "_id": "1"}}
{"title": "Document 1", "content": "Content here"}
{"index": {"_index": "my_index", "_id": "2"}}
{"title": "Document 2", "content": "More content"}
{"update": {"_index": "my_index", "_id": "3"}}
{"doc": {"status": "updated"}}
{"delete": {"_index": "my_index", "_id": "4"}}
```

**Production Implementation:**
```python
from elasticsearch import Elasticsearch
from elasticsearch.helpers import bulk

def bulk_index_documents(es_client, documents, index_name):
    """
    Efficiently bulk index documents with error handling
    """
    actions = []
    for doc in documents:
        actions.append({
            "_index": index_name,
            "_source": doc
        })
        
        # Process in batches of 1000
        if len(actions) >= 1000:
            try:
                bulk(es_client, actions)
                actions = []
            except Exception as e:
                print(f"Bulk indexing error: {e}")
                
    # Process remaining documents
    if actions:
        bulk(es_client, actions)
```

**Performance Tuning:**
- Optimal batch size: 1000-5000 documents or 5-15MB
- Use multiple threads for parallel bulk requests
- Monitor queue sizes and adjust accordingly

## Index-Level Optimizations

### Refresh Frequency Optimization

The refresh operation makes documents searchable but consumes significant resources.

```yaml
# elasticsearch.yml - Global setting
index.refresh_interval: 30s

# Index-specific setting
PUT /my_index/_settings
{
  "refresh_interval": "30s"
}

# Disable refresh for write-heavy workloads
PUT /my_index/_settings
{
  "refresh_interval": -1
}
```

**Use Case Example:**
```python
# High-volume logging scenario
def setup_logging_index():
    """
    Configure index for write-heavy logging workload
    """
    index_settings = {
        "settings": {
            "refresh_interval": "60s",  # Reduce refresh frequency
            "number_of_replicas": 0,    # Disable replicas during bulk load
            "translog.durability": "async",  # Async translog for speed
            "index.merge.policy.max_merge_at_once": 30
        }
    }
    return index_settings
```

### Field Optimization Strategies

Disable unnecessary features to reduce index size and improve query performance.

#### Source Field Optimization

```json
# Disable _source for analytics-only indices
PUT /analytics_index
{
  "mappings": {
    "_source": {
      "enabled": false
    },
    "properties": {
      "timestamp": {"type": "date"},
      "metric_value": {"type": "double"},
      "category": {"type": "keyword"}
    }
  }
}

# Selective source inclusion
PUT /selective_index
{
  "mappings": {
    "_source": {
      "includes": ["title", "summary"],
      "excludes": ["large_content", "binary_data"]
    }
  }
}
```

#### Doc Values Optimization

```json
# Disable doc_values for fields that don't need aggregations/sorting
PUT /my_index
{
  "mappings": {
    "properties": {
      "searchable_text": {
        "type": "text",
        "doc_values": false
      },
      "aggregatable_field": {
        "type": "keyword",
        "doc_values": true
      }
    }
  }
}
```

**Interview Insight:** *"What are doc_values and when should you disable them?"* - Doc_values enable aggregations, sorting, and scripting but consume disk space. Disable for fields used only in queries, not aggregations.

### Data Lifecycle Management

Separate hot and cold data for optimal resource utilization.

{% mermaid graph %}
    A[Hot Data<br/>SSD Storage<br/>Frequent Access] --> B[Warm Data<br/>HDD Storage<br/>Occasional Access]
    B --> C[Cold Data<br/>Archive Storage<br/>Rare Access]
    
    A --> D[High Resources<br/>More Replicas]
    B --> E[Medium Resources<br/>Fewer Replicas]
    C --> F[Minimal Resources<br/>Compressed Storage]
{% endmermaid %}

**ILM Policy Example:**
```json
PUT _ilm/policy/logs_policy
{
  "policy": {
    "phases": {
      "hot": {
        "actions": {
          "rollover": {
            "max_size": "1GB",
            "max_age": "1d"
          }
        }
      },
      "warm": {
        "min_age": "7d",
        "actions": {
          "allocate": {
            "number_of_replicas": 0
          }
        }
      },
      "cold": {
        "min_age": "30d",
        "actions": {
          "allocate": {
            "number_of_replicas": 0
          }
        }
      }
    }
  }
}
```

## Memory and Storage Optimization

### Off-Heap Memory Optimization

Elasticsearch uses off-heap memory for various caches and operations.

**Circuit Breaker Configuration:**
```yaml
# elasticsearch.yml
indices.breaker.total.limit: 70%
indices.breaker.fielddata.limit: 40%
indices.breaker.request.limit: 60%
```

**Field Data Cache Management:**
```json
# Monitor field data usage
GET /_nodes/stats/indices/fielddata

# Clear field data cache
POST /_cache/clear?fielddata=true

# Limit field data cache size
PUT /_cluster/settings
{
  "persistent": {
    "indices.fielddata.cache.size": "30%"
  }
}
```

**Production Monitoring Script:**
```bash
#!/bin/bash
# Monitor memory usage
curl -s "localhost:9200/_cat/nodes?v&h=name,heap.percent,ram.percent,fielddata.memory_size,query_cache.memory_size"
```

### Shard Optimization

Proper shard sizing is crucial for performance and cluster stability.

#### Shard Count and Size Guidelines

{% mermaid graph %}
    A[Determine Shard Strategy] --> B{Index Size}
    B -->|< 1GB| C[1 Primary Shard]
    B -->|1-50GB| D[1-5 Primary Shards]
    B -->|> 50GB| E[Calculate: Size/50GB]
    
    C --> F[Small Index Strategy]
    D --> G[Medium Index Strategy]
    E --> H[Large Index Strategy]
    
    F --> I[Minimize Overhead]
    G --> J[Balance Performance]
    H --> K[Distribute Load]
{% endmermaid %}

**Shard Calculation Formula:**
```python
def calculate_optimal_shards(index_size_gb, node_count):
    """
    Calculate optimal shard count based on index size and cluster size
    """
    # Target shard size: 20-50GB
    target_shard_size_gb = 30
    
    # Calculate based on size
    size_based_shards = max(1, index_size_gb // target_shard_size_gb)
    
    # Don't exceed node count (for primary shards)
    optimal_shards = min(size_based_shards, node_count)
    
    return optimal_shards

# Example usage
index_size = 150  # GB
nodes = 5
shards = calculate_optimal_shards(index_size, nodes)
print(f"Recommended shards: {shards}")
```

**Production Shard Settings:**
```json
PUT /optimized_index
{
  "settings": {
    "number_of_shards": 3,
    "number_of_replicas": 1,
    "index.routing.allocation.total_shards_per_node": 2
  }
}
```

### Query Performance Patterns

#### Efficient Query Patterns

```json
# Use filter context for exact matches (cacheable)
GET /my_index/_search
{
  "query": {
    "bool": {
      "must": [
        {"match": {"title": "elasticsearch"}}
      ],
      "filter": [
        {"term": {"status": "published"}},
        {"range": {"date": {"gte": "2023-01-01"}}}
      ]
    }
  }
}

# Avoid wildcard queries on large datasets
# BAD:
# {"wildcard": {"content": "*elasticsearch*"}}

# GOOD: Use match_phrase_prefix for autocomplete
{
  "match_phrase_prefix": {
    "title": "elasticsearch"
  }
}
```

#### Aggregation Optimization

```json
# Use composite aggregations for large cardinality
GET /my_index/_search
{
  "size": 0,
  "aggs": {
    "my_composite": {
      "composite": {
        "size": 1000,
        "sources": [
          {"category": {"terms": {"field": "category.keyword"}}},
          {"date": {"date_histogram": {"field": "timestamp", "interval": "1d"}}}
        ]
      }
    }
  }
}
```

## Monitoring and Troubleshooting

### Performance Metrics

```bash
# Key performance APIs
curl "localhost:9200/_cat/nodes?v&h=name,heap.percent,cpu,load_1m"
curl "localhost:9200/_cat/indices?v&h=index,docs.count,store.size,pri,rep"
curl "localhost:9200/_nodes/stats/indices/search,indices/indexing"
```

**Slow Query Analysis:**
```json
# Enable slow query logging
PUT /my_index/_settings
{
  "index.search.slowlog.threshold.query.warn": "10s",
  "index.search.slowlog.threshold.query.info": "5s",
  "index.search.slowlog.threshold.fetch.warn": "1s"
}
```

### Common Performance Anti-Patterns

**Interview Questions & Solutions:**

1. **"Why are my deep pagination queries slow?"**
   - Use scroll API for sequential processing
   - Use search_after for real-time pagination
   - Implement caching for frequently accessed pages

2. **"How do you handle high cardinality aggregations?"**
   - Use composite aggregations with pagination
   - Implement pre-aggregated indices for common queries
   - Consider using terms aggregation with execution_hint

3. **"What causes high memory usage in Elasticsearch?"**
   - Large field data caches from aggregations
   - Too many shards causing overhead
   - Inefficient query patterns causing cache thrashing

## Advanced Optimization Techniques

### Index Templates and Aliases

```json
# Optimized index template
PUT /_index_template/logs_template
{
  "index_patterns": ["logs-*"],
  "template": {
    "settings": {
      "number_of_shards": 1,
      "number_of_replicas": 0,
      "refresh_interval": "30s",
      "codec": "best_compression"
    },
    "mappings": {
      "dynamic": "strict",
      "properties": {
        "timestamp": {"type": "date"},
        "message": {"type": "text", "norms": false},
        "level": {"type": "keyword"},
        "service": {"type": "keyword"}
      }
    }
  }
}
```

### Machine Learning and Anomaly Detection

```json
# Use ML for capacity planning
PUT _ml/anomaly_detectors/high_search_rate
{
  "job_id": "high_search_rate",
  "analysis_config": {
    "bucket_span": "15m",
    "detectors": [
      {
        "function": "high_mean",
        "field_name": "search_rate"
      }
    ]
  },
  "data_description": {
    "time_field": "timestamp"
  }
}
```

## Conclusion

Elasticsearch query performance optimization requires a holistic approach combining system-level tuning, query optimization, and proper index design. The key is to:

1. **Monitor continuously** - Use built-in monitoring and custom metrics
2. **Test systematically** - Benchmark changes in isolated environments
3. **Scale progressively** - Start with simple optimizations before complex ones
4. **Plan for growth** - Design with future data volumes in mind

**Critical Interview Insight:** *"Performance optimization is not a one-time task but an ongoing process that requires understanding your data patterns, query characteristics, and growth projections."*

## External Resources

- [Elasticsearch Official Performance Tuning Guide](https://www.elastic.co/guide/en/elasticsearch/reference/current/tune-for-search-speed.html)
- [Elasticsearch Heap Sizing Guide](https://www.elastic.co/guide/en/elasticsearch/reference/current/heap-size.html)
- [Query Performance Best Practices](https://www.elastic.co/guide/en/elasticsearch/reference/current/query-dsl.html)
- [Index Lifecycle Management Documentation](https://www.elastic.co/guide/en/elasticsearch/reference/current/index-lifecycle-management.html)