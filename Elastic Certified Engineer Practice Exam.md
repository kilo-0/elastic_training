# Elastic Certified Engineer Practice Exercises
## Using Kibana Sample Data eCommerce Index

---

## Data Management

### Exercise 1: Define an index that satisfies given requirements
**Requirement**: Create a new index called `ecommerce_analytics` that:
- Stores customer purchase behavior with optimized mappings
- Has 3 primary shards and 2 replicas
- Includes custom settings for refresh interval (5s)
- Supports both text search and aggregations on product names

**Task**: Write the PUT request to create this index.




solution:
```Query DSL
PUT /ecommerce_analytics
{
  "settings": {
    "number_of_shards": 3,
    "number_of_replicas": 2,
    "refresh_interval": "5s"
  },
  "mappings": {
    "properties": {
      "product_name": {
        "type": "text",
        "fields": {
          "keyword": {
            "type": "keyword"
          }
        }
      },
      "customer_id": {
        "type": "keyword"
      },
      "purchase_date": {
        "type": "date"
      },
      "total_amount": {
        "type": "float"
      }
    }
  }
}
```


### Exercise 2: Define and use a dynamic template
**Requirement**: Create a dynamic template for the ecommerce index that:
- Automatically maps fields ending with "_price" as `scaled_float` with scaling factor 100
- Maps fields starting with "is_" as boolean
- Maps fields containing "timestamp" as date

**Task**: Write the dynamic template definition.




solution:
```Query DSL
PUT /ecommerce_enhanced
{
  "mappings": {
    "dynamic_templates": [
      {
        "prices": {
          "match": "*_price",
          "mapping": {
            "type": "scaled_float",
            "scaling_factor": 100
          }
        }
      },
      {
        "boolean_fields": {
          "match": "is_*",
          "mapping": {
            "type": "boolean"
          }
        }
      },
      {
        "timestamps": {
          "match": "*timestamp*",
          "mapping": {
            "type": "date"
          }
        }
      }
    ]
  }
}
```


### Exercise 3: Define an ILM policy for time-series index
**Requirement**: Create an ILM policy for ecommerce order data that:
- Rolls over when index reaches 50GB or 30 days
- Moves to warm phase after 7 days with force merge to 1 segment
- Moves to cold phase after 30 days
- Deletes after 90 days

**Task**: Write the ILM policy.




solution:
```Query DSL
PUT /_ilm/policy/ecommerce_orders_policy
{
  "policy": {
    "phases": {
      "hot": {
        "actions": {
          "rollover": {
            "max_size": "50GB",
            "max_age": "30d"
          }
        }
      },
      "warm": {
        "min_age": "7d",
        "actions": {
          "forcemerge": {
            "max_num_segments": 1
          },
          "shrink": {
            "number_of_shards": 1
          }
        }
      },
      "cold": {
        "min_age": "30d",
        "actions": {
          "freeze": {}
        }
      },
      "delete": {
        "min_age": "90d",
        "actions": {
          "delete": {}
        }
      }
    }
  }
}
```


### Exercise 4: Define an index template for data stream
**Requirement**: Create an index template for ecommerce clickstream data that:
- Creates a data stream
- Uses the ILM policy from Exercise 3
- Includes mappings for user_id, product_id, action, and timestamp

**Task**: Write the index template.




solution:
```Query DSL
PUT /_index_template/ecommerce_clickstream_template
{
  "index_patterns": ["ecommerce-clickstream-*"],
  "data_stream": {},
  "template": {
    "settings": {
      "index.lifecycle.name": "ecommerce_orders_policy",
      "number_of_shards": 2,
      "number_of_replicas": 1
    },
    "mappings": {
      "properties": {
        "@timestamp": {
          "type": "date"
        },
        "user_id": {
          "type": "keyword"
        },
        "product_id": {
          "type": "keyword"
        },
        "action": {
          "type": "keyword"
        },
        "session_id": {
          "type": "keyword"
        }
      }
    }
  }
}
```


---

## Searching Data

### Exercise 5: Search for terms and phrases
**Requirement**: Find all orders where:
- Customer's first name is "Mary" or "John"
- Product name contains the phrase "summer dress"

**Task**: Write the search query.




solution:
```Query DSL
GET /kibana_sample_data_ecommerce/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "terms": {
            "customer_first_name.keyword": ["Mary", "John"]
          }
        },
        {
          "match_phrase": {
            "products.product_name": "summer dress"
          }
        }
      ]
    }
  }
}
```


### Exercise 6: Boolean combination search
**Requirement**: Find orders that match ALL of these conditions:
- Order date is after 2024-01-01
- Total price is between $50 and $200
- Customer is female
- Exclude orders from "EUROPE"
- Boost results where manufacturer is "Tigress Enterprises"

**Task**: Write the complex boolean query.




solution:
```Query DSL
GET /kibana_sample_data_ecommerce/_search
{
  "query": {
    "bool": {
      "must": [
        {
          "range": {
            "order_date": {
              "gte": "2024-01-01"
            }
          }
        },
        {
          "range": {
            "taxful_total_price": {
              "gte": 50,
              "lte": 200
            }
          }
        },
        {
          "term": {
            "customer_gender": "FEMALE"
          }
        }
      ],
      "must_not": [
        {
          "term": {
            "geoip.continent_name": "Europe"
          }
        }
      ],
      "should": [
        {
          "match": {
            "manufacturer": {
              "query": "Tigress Enterprises",
              "boost": 2.0
            }
          }
        }
      ]
    }
  }
}
```


### Exercise 7: Asynchronous search
**Requirement**: Execute a heavy aggregation query asynchronously that:
- Calculates daily revenue for the last 90 days
- Groups by product category

**Task**: Write the async search request.




solution:
```Query DSL
POST /kibana_sample_data_ecommerce/_async_search?wait_for_completion_timeout=1s&keep_alive=5m
{
  "size": 0,
  "query": {
    "range": {
      "order_date": {
        "gte": "now-90d/d"
      }
    }
  },
  "aggs": {
    "daily_revenue": {
      "date_histogram": {
        "field": "order_date",
        "calendar_interval": "day"
      },
      "aggs": {
        "by_category": {
          "terms": {
            "field": "category.keyword",
            "size": 20
          },
          "aggs": {
            "revenue": {
              "sum": {
                "field": "taxful_total_price"
              }
            }
          }
        }
      }
    }
  }
}

# To get results
GET /_async_search/FmRldE8zREVEUzA2ZVpUeGs2ejJFUFEaMkZ5QTVrSTZSaVN3WlNFVmtlWHJsdzoxMDc=
```


### Exercise 8: Metric and bucket aggregations
**Requirement**: Calculate:
- Average order value by customer gender
- Top 5 manufacturers by total revenue
- Statistical summary of product prices

**Task**: Write the aggregation query.




solution:
```Query DSL
GET /kibana_sample_data_ecommerce/_search
{
  "size": 0,
  "aggs": {
    "avg_order_by_gender": {
      "terms": {
        "field": "customer_gender"
      },
      "aggs": {
        "avg_order_value": {
          "avg": {
            "field": "taxful_total_price"
          }
        }
      }
    },
    "top_manufacturers": {
      "terms": {
        "field": "manufacturer.keyword",
        "size": 5,
        "order": {
          "total_revenue": "desc"
        }
      },
      "aggs": {
        "total_revenue": {
          "sum": {
            "field": "taxful_total_price"
          }
        }
      }
    },
    "price_stats": {
      "stats": {
        "field": "products.price"
      }
    }
  }
}
```


### Exercise 9: Sub-aggregations
**Requirement**: Create a complex aggregation that:
- Groups by day of week
- For each day, finds top 3 product categories
- For each category, calculates average discount percentage
- Includes moving average of daily revenue

**Task**: Write the nested aggregation query.




solution:
```Query DSL
GET /kibana_sample_data_ecommerce/_search
{
  "size": 0,
  "aggs": {
    "by_day_of_week": {
      "terms": {
        "field": "day_of_week",
        "size": 7
      },
      "aggs": {
        "top_categories": {
          "terms": {
            "field": "category.keyword",
            "size": 3
          },
          "aggs": {
            "avg_discount": {
              "avg": {
                "field": "products.discount_percentage"
              }
            }
          }
        },
        "daily_revenue": {
          "sum": {
            "field": "taxful_total_price"
          }
        }
      }
    },
    "revenue_trend": {
      "date_histogram": {
        "field": "order_date",
        "calendar_interval": "day"
      },
      "aggs": {
        "revenue": {
          "sum": {
            "field": "taxful_total_price"
          }
        },
        "moving_avg_revenue": {
          "moving_avg": {
            "buckets_path": "revenue",
            "window": 7
          }
        }
      }
    }
  }
}
```


### Exercise 10: Cross-cluster search
**Requirement**: Search across multiple clusters for orders:
- Local cluster: kibana_sample_data_ecommerce
- Remote cluster: analytics:kibana_sample_data_ecommerce
- Find orders over $1000 from both clusters

**Task**: Write the cross-cluster search.




solution:
```Query DSL
GET /kibana_sample_data_ecommerce,analytics:kibana_sample_data_ecommerce/_search
{
  "query": {
    "range": {
      "taxful_total_price": {
        "gte": 1000
      }
    }
  },
  "_source": ["order_id", "customer_full_name", "taxful_total_price"],
  "sort": [
    {
      "taxful_total_price": {
        "order": "desc"
      }
    }
  ]
}
```


### Exercise 11: Runtime field search
**Requirement**: Create a runtime field that:
- Calculates profit margin (taxful - taxless price)
- Search for orders with profit margin > $10

**Task**: Write the search with runtime field.




solution:
```Query DSL
GET /kibana_sample_data_ecommerce/_search
{
  "runtime_mappings": {
    "profit_margin": {
      "type": "double",
      "script": {
        "source": """
          if (doc['taxful_total_price'].size() > 0 && doc['taxless_total_price'].size() > 0) {
            emit(doc['taxful_total_price'].value - doc['taxless_total_price'].value);
          }
        """
      }
    }
  },
  "query": {
    "range": {
      "profit_margin": {
        "gt": 10
      }
    }
  },
  "fields": ["profit_margin", "order_id", "taxful_total_price", "taxless_total_price"]
}
```


---

## Developing Search Applications

### Exercise 12: Sort results
**Requirement**: Sort search results by:
1. Customer gender (FEMALE first)
2. Order date (newest first)
3. Total price (highest first)
4. Include missing values at the end

**Task**: Write the search with complex sorting.




solution:
```Query DSL
GET /kibana_sample_data_ecommerce/_search
{
  "query": {
    "match_all": {}
  },
  "sort": [
    {
      "customer_gender": {
        "order": "desc",
        "missing": "_last"
      }
    },
    {
      "order_date": {
        "order": "desc"
      }
    },
    {
      "taxful_total_price": {
        "order": "desc",
        "missing": "_last"
      }
    }
  ]
}
```


### Exercise 13: Implement pagination
**Requirement**: Implement three types of pagination:
1. From/Size pagination for first 100 results
2. Search After for deep pagination
3. Scroll API for processing all documents

**Task**: Write all three pagination approaches.




solution:
```Query DSL
# 1. From/Size Pagination (Page 3, 20 results per page)
GET /kibana_sample_data_ecommerce/_search
{
  "from": 40,
  "size": 20,
  "query": {
    "match_all": {}
  },
  "sort": [
    {"order_date": "desc"},
    {"_id": "asc"}
  ]
}

# 2. Search After
# First request
GET /kibana_sample_data_ecommerce/_search
{
  "size": 20,
  "query": {
    "match_all": {}
  },
  "sort": [
    {"order_date": "desc"},
    {"_id": "asc"}
  ]
}

# Subsequent request (using sort values from last hit)
GET /kibana_sample_data_ecommerce/_search
{
  "size": 20,
  "query": {
    "match_all": {}
  },
  "search_after": ["2024-03-15T10:00:00.000Z", "abc123"],
  "sort": [
    {"order_date": "desc"},
    {"_id": "asc"}
  ]
}

# 3. Scroll API
# Initial scroll request
POST /kibana_sample_data_ecommerce/_search?scroll=1m
{
  "size": 100,
  "query": {
    "match_all": {}
  }
}

# Continue scrolling
POST /_search/scroll
{
  "scroll": "1m",
  "scroll_id": "DXF1ZXJ5QW5kRmV0Y2gBAAAAAAAAAD4WYm9laVYtZndUQlNsdDcwakFMNjU1QQ=="
}
```


### Exercise 14: Define and use index aliases
**Requirement**: Create aliases for the ecommerce index that:
- `current_orders`: Only shows orders from last 30 days
- `high_value_orders`: Only shows orders over $500
- `female_customers`: Only shows orders from female customers
- Create a write alias for indexing new data

**Task**: Write the alias definitions.




solution:
```Query DSL
POST /_aliases
{
  "actions": [
    {
      "add": {
        "index": "kibana_sample_data_ecommerce",
        "alias": "current_orders",
        "filter": {
          "range": {
            "order_date": {
              "gte": "now-30d/d"
            }
          }
        }
      }
    },
    {
      "add": {
        "index": "kibana_sample_data_ecommerce",
        "alias": "high_value_orders",
        "filter": {
          "range": {
            "taxful_total_price": {
              "gte": 500
            }
          }
        }
      }
    },
    {
      "add": {
        "index": "kibana_sample_data_ecommerce",
        "alias": "female_customers",
        "filter": {
          "term": {
            "customer_gender": "FEMALE"
          }
        }
      }
    },
    {
      "add": {
        "index": "kibana_sample_data_ecommerce",
        "alias": "ecommerce_write",
        "is_write_index": true
      }
    }
  ]
}

# Test the aliases
GET /current_orders/_count
GET /high_value_orders/_search
{
  "size": 5
}
```


---

## Data Processing

### Exercise 15: Define a mapping
**Requirement**: Create a new mapping for product reviews that includes:
- Nested type for review details
- Completion suggester for product names
- Dense vector for sentiment analysis
- Geo-point for reviewer location

**Task**: Write the mapping definition.




solution:
```Query DSL
PUT /product_reviews
{
  "mappings": {
    "properties": {
      "product_id": {
        "type": "keyword"
      },
      "product_name": {
        "type": "text",
        "fields": {
          "keyword": {
            "type": "keyword"
          },
          "suggest": {
            "type": "completion"
          }
        }
      },
      "reviews": {
        "type": "nested",
        "properties": {
          "reviewer_id": {
            "type": "keyword"
          },
          "reviewer_name": {
            "type": "text"
          },
          "rating": {
            "type": "byte"
          },
          "comment": {
            "type": "text",
            "analyzer": "english"
          },
          "review_date": {
            "type": "date"
          },
          "helpful_votes": {
            "type": "integer"
          }
        }
      },
      "sentiment_vector": {
        "type": "dense_vector",
        "dims": 3,
        "index": true,
        "similarity": "cosine"
      },
      "reviewer_location": {
        "type": "geo_point"
      },
      "average_rating": {
        "type": "half_float"
      }
    }
  }
}
```


### Exercise 16: Multi-fields with different analyzers
**Requirement**: Update the product_name field to support:
- Standard text search
- Exact keyword matching
- Edge n-gram for autocomplete
- Phonetic matching for fuzzy search

**Task**: Define the multi-field mapping with custom analyzers.




solution:
```Query DSL
PUT /ecommerce_products
{
  "settings": {
    "analysis": {
      "analyzer": {
        "autocomplete": {
          "tokenizer": "autocomplete",
          "filter": ["lowercase"]
        },
        "autocomplete_search": {
          "tokenizer": "standard",
          "filter": ["lowercase"]
        },
        "phonetic_analyzer": {
          "tokenizer": "standard",
          "filter": ["lowercase", "phonetic_filter"]
        }
      },
      "tokenizer": {
        "autocomplete": {
          "type": "edge_ngram",
          "min_gram": 2,
          "max_gram": 10,
          "token_chars": ["letter", "digit"]
        }
      },
      "filter": {
        "phonetic_filter": {
          "type": "phonetic",
          "encoder": "metaphone"
        }
      }
    }
  },
  "mappings": {
    "properties": {
      "product_name": {
        "type": "text",
        "analyzer": "standard",
        "fields": {
          "keyword": {
            "type": "keyword"
          },
          "autocomplete": {
            "type": "text",
            "analyzer": "autocomplete",
            "search_analyzer": "autocomplete_search"
          },
          "phonetic": {
            "type": "text",
            "analyzer": "phonetic_analyzer"
          }
        }
      }
    }
  }
}
```


### Exercise 17: Reindex and Update By Query
**Requirement**: 
1. Reindex ecommerce data to a new index with updated mappings
2. Update all documents to add a "processed" field with current timestamp
3. Update only high-value orders (>$1000) to add a "vip_order" flag

**Task**: Write the reindex and update queries.




solution:
```Query DSL
# 1. Reindex with script to transform data
POST /_reindex
{
  "source": {
    "index": "kibana_sample_data_ecommerce",
    "query": {
      "range": {
        "order_date": {
          "gte": "now-90d/d"
        }
      }
    }
  },
  "dest": {
    "index": "ecommerce_v2"
  },
  "script": {
    "source": """
      ctx._source.total_items = ctx._source.products.length;
      ctx._source.is_high_value = ctx._source.taxful_total_price > 500;
    """
  }
}

# 2. Update all documents to add processed timestamp
POST /ecommerce_v2/_update_by_query
{
  "script": {
    "source": """
      ctx._source.processed_at = new Date().getTime();
      ctx._source.processed = true;
    """
  }
}

# 3. Update high-value orders
POST /ecommerce_v2/_update_by_query
{
  "query": {
    "range": {
      "taxful_total_price": {
        "gt": 1000
      }
    }
  },
  "script": {
    "source": """
      ctx._source.vip_order = true;
      ctx._source.vip_discount_eligible = true;
    """
  }
}
```


### Exercise 18: Define an ingest pipeline
**Requirement**: Create an ingest pipeline that:
- Enriches data with customer age from birth date
- Converts currency to USD if needed
- Extracts domain from email
- Adds timestamp and removes PII fields

**Task**: Write the ingest pipeline.




solution:
```Query DSL
PUT /_ingest/pipeline/ecommerce_processing
{
  "description": "Process ecommerce orders",
  "processors": [
    {
      "script": {
        "description": "Calculate customer age",
        "source": """
          if (ctx.customer_birth_date != null) {
            def birth = ZonedDateTime.parse(ctx.customer_birth_date);
            def now = ZonedDateTime.now();
            ctx.customer_age = ChronoUnit.YEARS.between(birth, now);
          }
        """
      }
    },
    {
      "script": {
        "description": "Convert currency to USD",
        "source": """
          if (ctx.currency != 'USD') {
            // Simplified conversion rates
            def rates = ['EUR': 1.1, 'GBP': 1.3, 'JPY': 0.0091];
            if (rates.containsKey(ctx.currency)) {
              ctx.original_currency = ctx.currency;
              ctx.original_price = ctx.taxful_total_price;
              ctx.taxful_total_price = ctx.taxful_total_price * rates[ctx.currency];
              ctx.currency = 'USD';
            }
          }
        """
      }
    },
    {
      "grok": {
        "field": "email",
        "patterns": ["%{EMAILADDRESS:email_parsed}"],
        "on_failure": [
          {
            "set": {
              "field": "email_domain",
              "value": "unknown"
            }
          }
        ]
      }
    },
    {
      "script": {
        "description": "Extract email domain",
        "source": """
          if (ctx.email != null) {
            def parts = ctx.email.split('@');
            if (parts.length == 2) {
              ctx.email_domain = parts[1];
            }
          }
        """
      }
    },
    {
      "set": {
        "field": "pipeline_timestamp",
        "value": "{{_ingest.timestamp}}"
      }
    },
    {
      "remove": {
        "field": ["customer_phone", "email"],
        "ignore_missing": true
      }
    }
  ],
  "on_failure": [
    {
      "set": {
        "field": "_index",
        "value": "failed-{{ _index }}"
      }
    }
  ]
}

# Test the pipeline
POST /_ingest/pipeline/ecommerce_processing/_simulate
{
  "docs": [
    {
      "_source": {
        "customer_birth_date": "1990-05-15T00:00:00",
        "currency": "EUR",
        "taxful_total_price": 100,
        "email": "customer@example.com",
        "customer_phone": "123-456-7890"
      }
    }
  ]
}
```


### Exercise 19: Runtime fields with Painless
**Requirement**: Define runtime fields that:
- Calculate days since order
- Determine customer lifetime value category
- Extract product brand from manufacturer name
- Calculate effective discount percentage

**Task**: Write the runtime field definitions.




solution:
```Query DSL
PUT /kibana_sample_data_ecommerce/_mapping
{
  "runtime": {
    "days_since_order": {
      "type": "long",
      "script": {
        "source": """
          if (doc['order_date'].size() > 0) {
            def orderDate = doc['order_date'].value.toInstant().toEpochMilli();
            def now = new Date().getTime();
            emit((now - orderDate) / 86400000);
          }
        """
      }
    },
    "customer_ltv_category": {
      "type": "keyword",
      "script": {
        "source": """
          // This would normally query historical data
          // Simplified for demonstration
          double totalPrice = doc['taxful_total_price'].value;
          if (totalPrice > 1000) {
            emit('PLATINUM');
          } else if (totalPrice > 500) {
            emit('GOLD');
          } else if (totalPrice > 100) {
            emit('SILVER');
          } else {
            emit('BRONZE');
          }
        """
      }
    },
    "product_brands": {
      "type": "keyword",
      "script": {
        "source": """
          if (doc['manufacturer.keyword'].size() > 0) {
            def manufacturers = doc['manufacturer.keyword'];
            Set brands = new HashSet();
            for (manufacturer in manufacturers) {
              // Extract first word as brand
              def parts = manufacturer.split(' ');
              if (parts.length > 0) {
                brands.add(parts[0]);
              }
            }
            for (brand in brands) {
              emit(brand);
            }
          }
        """
      }
    },
    "effective_discount_pct": {
      "type": "double",
      "script": {
        "source": """
          if (doc['taxful_total_price'].size() > 0 && doc['taxless_total_price'].size() > 0) {
            double taxful = doc['taxful_total_price'].value;
            double taxless = doc['taxless_total_price'].value;
            if (taxless > 0) {
              double discount = ((taxless - taxful) / taxless) * 100;
              emit(Math.round(discount * 100) / 100.0);
            }
          }
        """
      }
    }
  }
}

# Search using runtime fields
GET /kibana_sample_data_ecommerce/_search
{
  "query": {
    "range": {
      "days_since_order": {
        "lte": 30
      }
    }
  },
  "fields": ["days_since_order", "customer_ltv_category", "order_date", "taxful_total_price"],
  "size": 5
}
```


---

## Cluster Management

### Exercise 20: Diagnose shard issues
**Requirement**: Diagnose and fix a cluster with:
- Unassigned shards
- Node disk space issues
- Shard allocation failures

**Task**: Write commands to diagnose and repair.




solution:
```Query DSL
# 1. Check cluster health
GET /_cluster/health?level=indices

# 2. Check allocation explain for unassigned shards
GET /_cluster/allocation/explain
{
  "index": "kibana_sample_data_ecommerce",
  "shard": 0,
  "primary": true
}

# 3. Check node disk usage
GET /_cat/nodes?v&h=name,disk.used_percent,disk.avail,disk.total

# 4. Check shard allocation settings
GET /_cluster/settings?include_defaults=true&filter_path=*.cluster.routing.allocation

# 5. Fix disk watermark issues
PUT /_cluster/settings
{
  "transient": {
    "cluster.routing.allocation.disk.watermark.low": "85%",
    "cluster.routing.allocation.disk.watermark.high": "90%",
    "cluster.routing.allocation.disk.watermark.flood_stage": "95%"
  }
}

# 6. Retry failed shard allocation
POST /_cluster/reroute?retry_failed=true

# 7. Force allocate a specific shard
POST /_cluster/reroute
{
  "commands": [
    {
      "allocate_stale_primary": {
        "index": "kibana_sample_data_ecommerce",
        "shard": 0,
        "node": "node-1",
        "accept_data_loss": true
      }
    }
  ]
}

# 8. Enable shard allocation if disabled
PUT /_cluster/settings
{
  "persistent": {
    "cluster.routing.allocation.enable": "all"
  }
}
```


### Exercise 21: Backup and restore
**Requirement**: 
- Configure a snapshot repository
- Create snapshots of specific indices
- Restore data with index renaming

**Task**: Write the backup and restore commands.




solution:
```Query DSL
# 1. Register snapshot repository
PUT /_snapshot/ecommerce_backup
{
  "type": "fs",
  "settings": {
    "location": "/mount/backups/ecommerce",
    "compress": true,
    "max_restore_bytes_per_sec": "50mb",
    "max_snapshot_bytes_per_sec": "50mb"
  }
}

# 2. Create snapshot of specific indices
PUT /_snapshot/ecommerce_backup/snapshot_2024_03_15?wait_for_completion=true
{
  "indices": ["kibana_sample_data_ecommerce", "ecommerce_*"],
  "ignore_unavailable": true,
  "include_global_state": false,
  "metadata": {
    "taken_by": "admin",
    "taken_because": "Regular backup before maintenance"
  }
}

# 3. List snapshots
GET /_snapshot/ecommerce_backup/_all

# 4. Get snapshot status
GET /_snapshot/ecommerce_backup/snapshot_2024_03_15/_status

# 5. Restore with rename
POST /_snapshot/ecommerce_backup/snapshot_2024_03_15/_restore
{
  "indices": "kibana_sample_data_ecommerce",
  "ignore_unavailable": true,
  "include_global_state": false,
  "rename_pattern": "kibana_sample_data_(.+)",
  "rename_replacement": "restored_$1_2024",
  "include_aliases": false
}

# 6. Restore specific indices with settings modifications
POST /_snapshot/ecommerce_backup/snapshot_2024_03_15/_restore
{
  "indices": ["ecommerce_*"],
  "index_settings": {
    "index.number_of_replicas": 0,
    "index.refresh_interval": "-1"
  },
  "ignore_index_settings": ["index.routing.allocation.require.*"]
}
```


### Exercise 22: Configure searchable snapshots
**Requirement**: 
- Create a searchable snapshot for cold data
- Mount it with partial caching
- Configure cache size and performance

**Task**: Write the searchable snapshot configuration.




solution:
```Query DSL
# 1. Create a regular snapshot first
PUT /_snapshot/cold_storage/ecommerce_historical?wait_for_completion=true
{
  "indices": "kibana_sample_data_ecommerce",
  "include_global_state": false
}

# 2. Mount as searchable snapshot with partial cache
POST /_snapshot/cold_storage/ecommerce_historical/_mount?wait_for_completion=true
{
  "index": "kibana_sample_data_ecommerce",
  "renamed_index": "ecommerce_historical_searchable",
  "index_settings": {
    "index.number_of_replicas": 0
  },
  "storage": "shared_cache"
}

# 3. Configure cache settings
PUT /_cluster/settings
{
  "persistent": {
    "xpack.searchable.snapshot.cache.size": "10gb",
    "xpack.searchable.snapshot.cache.range_size": "32mb"
  }
}

# 4. Check searchable snapshot stats
GET /_searchable_snapshots/stats

# 5. Clear cache for specific index
POST /ecommerce_historical_searchable/_searchable_snapshots/cache/clear

# 6. Mount with full cache for frequently accessed data
POST /_snapshot/cold_storage/ecommerce_historical/_mount?wait_for_completion=true
{
  "index": "kibana_sample_data_ecommerce",
  "renamed_index": "ecommerce_warm_searchable",
  "index_settings": {
    "index.number_of_replicas": 1
  },
  "storage": "full_copy"
}
```


### Exercise 23: Configure cross-cluster search
**Requirement**: 
- Connect to remote clusters
- Configure security settings
- Search across multiple clusters with different patterns

**Task**: Write the cross-cluster search configuration.




solution:
```Query DSL
# 1. Configure remote cluster connections
PUT /_cluster/settings
{
  "persistent": {
    "cluster": {
      "remote": {
        "eu_cluster": {
          "seeds": ["10.0.1.10:9300", "10.0.1.11:9300"],
          "skip_unavailable": true
        },
        "us_cluster": {
          "seeds": ["10.0.2.10:9300", "10.0.2.11:9300"],
          "skip_unavailable": true
        },
        "asia_cluster": {
          "seeds": ["10.0.3.10:9300"],
          "skip_unavailable": false,
          "transport.ping_schedule": "30s"
        }
      }
    }
  }
}

# 2. Check remote cluster info
GET /_remote/info

# 3. Search across all clusters
GET /kibana_sample_data_ecommerce,eu_cluster:kibana_sample_data_ecommerce,us_cluster:kibana_sample_data_ecommerce/_search
{
  "query": {
    "range": {
      "order_date": {
        "gte": "now-7d/d"
      }
    }
  },
  "aggs": {
    "orders_by_cluster": {
      "terms": {
        "field": "_index",
        "size": 10
      }
    }
  }
}

# 4. Async search across clusters
POST /kibana_sample_data_ecommerce,*:kibana_sample_data_ecommerce/_async_search?ccs_minimize_roundtrips=true
{
  "size": 0,
  "query": {
    "match_all": {}
  },
  "aggs": {
    "sales_by_region": {
      "terms": {
        "field": "geoip.continent_name"
      },
      "aggs": {
        "total_revenue": {
          "sum": {
            "field": "taxful_total_price"
          }
        }
      }
    }
  }
}

# 5. Configure per-search remote cluster settings
GET /*:kibana_sample_data_ecommerce/_search?ccs_minimize_roundtrips=false
{
  "query": {
    "match_all": {}
  },
  "size": 1
}
```


### Exercise 24: Implement cross-cluster replication
**Requirement**: 
- Set up leader and follower indices
- Configure auto-follow patterns
- Handle replication issues

**Task**: Write the CCR configuration.




solution:
```Query DSL
# 1. Configure remote cluster for CCR
PUT /_cluster/settings
{
  "persistent": {
    "cluster": {
      "remote": {
        "leader_cluster": {
          "seeds": ["10.0.1.10:9300"],
          "skip_unavailable": false
        }
      }
    }
  }
}

# 2. Create follower index
PUT /follower_ecommerce/_ccr/follow?wait_for_active_shards=1
{
  "remote_cluster": "leader_cluster",
  "leader_index": "kibana_sample_data_ecommerce",
  "settings": {
    "index.number_of_replicas": 0
  },
  "max_read_request_operation_count": 5120,
  "max_outstanding_read_requests": 12,
  "max_read_request_size": "32mb",
  "max_write_request_operation_count": 5120,
  "max_write_request_size": "9223372036854775807b",
  "max_outstanding_write_requests": 9,
  "max_write_buffer_count": 2147483647,
  "max_write_buffer_size": "512mb",
  "max_retry_delay": "500ms",
  "read_poll_timeout": "1m"
}

# 3. Create auto-follow pattern
PUT /_ccr/auto_follow/ecommerce_pattern
{
  "remote_cluster": "leader_cluster",
  "leader_index_patterns": ["ecommerce-*", "orders-*"],
  "follow_index_pattern": "{{leader_index}}-replica",
  "settings": {
    "index.number_of_replicas": 0
  },
  "max_read_request_operation_count": 5120,
  "max_outstanding_read_requests": 12
}

# 4. Check replication stats
GET /follower_ecommerce/_ccr/stats

# 5. Pause replication
POST /follower_ecommerce/_ccr/pause_follow

# 6. Resume replication
POST /follower_ecommerce/_ccr/resume_follow
{
  "max_read_request_operation_count": 10240,
  "max_outstanding_read_requests": 16
}

# 7. Unfollow (convert to regular index)
POST /follower_ecommerce/_ccr/unfollow

# 8. Check auto-follow patterns
GET /_ccr/auto_follow

# 9. Get auto-follow stats
GET /_ccr/stats
```


### Exercise 25: Automate snapshots with SLM
**Requirement**: 
- Create SLM policy for daily snapshots
- Configure retention rules
- Set up monitoring and notifications

**Task**: Write the Snapshot Lifecycle Management configuration.




solution:
```Query DSL
# 1. Create SLM policy for daily snapshots
PUT /_slm/policy/daily_ecommerce_backup
{
  "schedule": "0 30 2 * * ?",  // 2:30 AM daily
  "name": "<ecommerce-snapshot-{now/d}>",
  "repository": "ecommerce_backup",
  "config": {
    "indices": ["kibana_sample_data_ecommerce", "ecommerce_*"],
    "include_global_state": false,
    "partial": false,
    "metadata": {
      "policy": "daily_backup",
      "description": "Daily automated backup of ecommerce indices"
    }
  },
  "retention": {
    "expire_after": "30d",
    "min_count": 5,
    "max_count": 50
  }
}

# 2. Create hourly policy for critical data
PUT /_slm/policy/hourly_critical_backup
{
  "schedule": "0 0 * * * ?",  // Every hour
  "name": "<critical-{now/h}>",
  "repository": "ecommerce_backup",
  "config": {
    "indices": ["current_orders", "high_value_orders"],
    "include_global_state": false,
    "partial": true
  },
  "retention": {
    "expire_after": "7d",
    "min_count": 24,
    "max_count": 168
  }
}

# 3. Execute policy manually
POST /_slm/policy/daily_ecommerce_backup/_execute

# 4. Get policy info
GET /_slm/policy/daily_ecommerce_backup

# 5. Get SLM stats
GET /_slm/stats

# 6. Get policy execution history
GET /_slm/policy/daily_ecommerce_backup/_history

# 7. Update policy
PUT /_slm/policy/daily_ecommerce_backup
{
  "schedule": "0 30 3 * * ?",  // Changed to 3:30 AM
  "name": "<ecommerce-snapshot-{now/d}>",
  "repository": "ecommerce_backup",
  "config": {
    "indices": ["kibana_sample_data_ecommerce", "ecommerce_*", "product_reviews"],
    "include_global_state": false,
    "partial": false
  },
  "retention": {
    "expire_after": "45d",
    "min_count": 7,
    "max_count": 60
  }
}

# 8. Stop/Start SLM
POST /_slm/stop
POST /_slm/start

# 9. Delete policy
DELETE /_slm/policy/hourly_critical_backup
```


---

## Advanced Practice Scenarios

### Scenario 1: Complete Data Pipeline
**Requirement**: Build a complete data pipeline that:
1. Ingests raw ecommerce data
2. Enriches it with customer demographics
3. Creates time-based indices with ILM
4. Sets up CCR for disaster recovery




solution:
```Query DSL
# 1. Create ILM policy
PUT /_ilm/policy/ecommerce_lifecycle
{
  "policy": {
    "phases": {
      "hot": {
        "actions": {
          "rollover": {
            "max_age": "7d",
            "max_size": "50GB"
          }
        }
      },
      "warm": {
        "min_age": "7d",
        "actions": {
          "shrink": {
            "number_of_shards": 1
          },
          "forcemerge": {
            "max_num_segments": 1
          }
        }
      },
      "cold": {
        "min_age": "30d",
        "actions": {
          "searchable_snapshot": {
            "snapshot_repository": "ecommerce_backup"
          }
        }
      }
    }
  }
}

# 2. Create index template
PUT /_index_template/ecommerce_orders_template
{
  "index_patterns": ["ecommerce-orders-*"],
  "template": {
    "settings": {
      "number_of_shards": 3,
      "index.lifecycle.name": "ecommerce_lifecycle",
      "index.lifecycle.rollover_alias": "ecommerce-orders"
    },
    "mappings": {
      "properties": {
        "order_date": {"type": "date"},
        "customer_id": {"type": "keyword"},
        "enriched": {
          "properties": {
            "age_group": {"type": "keyword"},
            "customer_segment": {"type": "keyword"},
            "lifetime_value": {"type": "float"}
          }
        }
      }
    }
  }
}

# 3. Create ingest pipeline with enrichment
PUT /_enrich/policy/customer_enrichment
{
  "match": {
    "indices": "customer_demographics",
    "match_field": "customer_id",
    "enrich_fields": ["age_group", "customer_segment", "lifetime_value"]
  }
}

POST /_enrich/policy/customer_enrichment/_execute

PUT /_ingest/pipeline/ecommerce_enrichment
{
  "processors": [
    {
      "enrich": {
        "policy_name": "customer_enrichment",
        "field": "customer_id",
        "target_field": "enriched"
      }
    }
  ]
}

# 4. Create initial index
PUT /ecommerce-orders-000001
{
  "aliases": {
    "ecommerce-orders": {
      "is_write_index": true
    }
  }
}

# 5. Set up CCR
PUT /ecommerce-orders-replica/_ccr/follow
{
  "remote_cluster": "dr_cluster",
  "leader_index": "ecommerce-orders",
  "settings": {
    "index.number_of_replicas": 1
  }
}
```


### Scenario 2: Performance Optimization Challenge
**Requirement**: Optimize search performance for:
- 1 billion documents
- Complex aggregations
- Real-time search requirements




solution:
```Query DSL
# 1. Create optimized index with custom settings
PUT /ecommerce_optimized
{
  "settings": {
    "number_of_shards": 10,
    "number_of_replicas": 1,
    "index.refresh_interval": "30s",
    "index.translog.durability": "async",
    "index.translog.sync_interval": "30s",
    "index.merge.scheduler.max_thread_count": 4,
    "index.search.idle.after": "30s",
    "index.max_result_window": 10000,
    "index.routing.allocation.total_shards_per_node": 2
  },
  "mappings": {
    "properties": {
      "order_id": {
        "type": "keyword",
        "doc_values": true
      },
      "customer_id": {
        "type": "keyword",
        "eager_global_ordinals": true
      },
      "order_date": {
        "type": "date",
        "format": "date_optional_time||epoch_millis"
      },
      "products": {
        "type": "nested",
        "properties": {
          "product_id": {"type": "keyword"},
          "price": {"type": "scaled_float", "scaling_factor": 100}
        }
      }
    }
  }
}

# 2. Create search template for common queries
PUT /_scripts/ecommerce_search
{
  "script": {
    "lang": "mustache",
    "source": {
      "query": {
        "bool": {
          "filter": [
            {"range": {"order_date": {"gte": "{{start_date}}", "lte": "{{end_date}}"}}},
            {"terms": {"customer_gender": "{{#gender}}{{.}}{{/gender}}"}}
          ]
        }
      },
      "aggs": {
        "date_histogram": {
          "date_histogram": {
            "field": "order_date",
            "calendar_interval": "{{interval}}",
            "min_doc_count": 1
          },
          "aggs": {
            "revenue": {
              "sum": {"field": "taxful_total_price"}
            }
          }
        }
      }
    }
  }
}

# 3. Use search template
GET /ecommerce_optimized/_search/template
{
  "id": "ecommerce_search",
  "params": {
    "start_date": "2024-01-01",
    "end_date": "2024-03-31",
    "gender": ["FEMALE"],
    "interval": "day"
  }
}

# 4. Pre-compute expensive aggregations
PUT /ecommerce_summary
{
  "mappings": {
    "properties": {
      "date": {"type": "date"},
      "daily_revenue": {"type": "float"},
      "daily_orders": {"type": "integer"},
      "top_products": {"type": "keyword"}
    }
  }
}

# 5. Index pre-aggregated data
POST /ecommerce_optimized/_search
{
  "size": 0,
  "aggs": {
    "daily_summary": {
      "date_histogram": {
        "field": "order_date",
        "calendar_interval": "day"
      },
      "aggs": {
        "revenue": {"sum": {"field": "taxful_total_price"}},
        "orders": {"value_count": {"field": "order_id"}}
      }
    }
  }
}
```


---

## Tips for the Exam

1. **Time Management**: Allocate time based on question complexity
2. **Read Carefully**: Pay attention to specific requirements in each question
3. **Test Your Queries**: Always verify your syntax and logic
4. **Know the APIs**: Be familiar with both REST API and Kibana Dev Tools
5. **Understand Version Differences**: Some features may vary by Elasticsearch version

## Additional Resources

- Practice with real Kibana sample data
- Review Elasticsearch documentation for each API
- Create your own scenarios beyond these exercises
- Join Elasticsearch study groups and forums

Good luck with your Elastic Certified Engineer exam!