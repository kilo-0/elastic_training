# Elasticsearch Certified Engineer Training

A comprehensive training repository for preparing for the Elastic Certified Engineer exam, featuring hands-on practice exercises and sample datasets for Kibana visualization.

## Overview

This repository contains structured learning materials and practical exercises designed to help you master the skills required for the Elastic Certified Engineer certification. The training covers core Elasticsearch concepts, cluster management, data modeling, search optimization, and Kibana visualization techniques using a containerized Docker environment.

```
elasticsearch-certified-engineer-training/
├── README.md
├── Elastic Certified Engineer Practice Exercises.md
├── datasets/
│   ├── ecommerce_sample.json (included in elastic installation)
│   ├── maritime-sample.csv
│   └── user_behavior.json
├── kibana_dashboards/
│   ├── ecommerce_analytics
│   └── maritime vessel tracking
```
    
## Certification Goals

The Elastic Certified Engineer exam validates your ability to:
- Install, configure, and manage Elasticsearch clusters (including containerized deployments)
- Design and implement effective data mappings and indexing strategies
- Optimize search performance and query execution
- Implement security, monitoring, and backup solutions
- Create meaningful visualizations and dashboards in Kibana
- Work with Elasticsearch in various deployment scenarios including Docker


## Getting Started
# Install Docker here
https://docs.docker.com/

#Install instructions for Elastic on Docker
https://www.elastic.co/docs/deploy-manage/deploy/self-managed/install-elasticsearch-docker-basic


### Prerequisites

- Docker Desktop installed and running
- Docker Compose
- PowerShell (Windows PowerShell or PowerShell Core)
- Basic understanding of JSON and REST APIs
- Familiarity with Docker containers

### Example Installation (using powershell commands inside Docker)

1. **Clone the repository:**
   ```powershell
   git clone https://github.com/yourusername/elasticsearch-certified-engineer-training.git
   cd elasticsearch-certified-engineer-training
   ```

2. **Start the containerized Elasticsearch cluster:**
   ```powershell
   # Start the Docker Compose stack
   docker-compose up -d
   
   # Verify containers are running
   docker-compose ps
   ```

3. **Wait for cluster to be ready and load sample datasets:**
   ```powershell
   # Check cluster health
   Invoke-RestMethod -Uri "http://localhost:9200/_cluster/health" -Method Get
   
   # Load sample datasets
   .\scripts\Load-SampleData.ps1
   ```

## Training Materials

### Practice Exercises

The `Elastic Certified Engineer Practice Exercises.md` file contains:

- **Cluster Management**: Node configuration, shard allocation, and cluster health monitoring
- **Index Management**: Mapping design, analyzers, and index lifecycle policies
- **Search & Aggregations**: Complex queries, aggregation pipelines, and performance optimization
- **Data Processing**: Ingest pipelines, transforms, and data enrichment
- **Security**: Authentication, authorization, and field-level security
- **Monitoring**: Cluster monitoring, alerting, and troubleshooting

### Sample Datasets

#### E-commerce Data (`ecommerce_sample.json`)
- Product catalog with categories, prices, and inventory
- Customer orders and transaction history
- Perfect for practicing aggregations and business analytics

#### Web Server Logs (`web_logs.json`)
- Apache/Nginx access logs with timestamps, IPs, and response codes
- Ideal for log analysis and security monitoring exercises

#### System Metrics (`system_metrics.json`)
- CPU, memory, and disk usage metrics over time
- Great for time-series analysis and performance monitoring

#### User Behavior (`user_behavior.json`)
- User interactions, page views, and session data
- Useful for funnel analysis and user journey mapping

## Usage Examples

### Basic Cluster Health Check
```powershell
Invoke-RestMethod -Uri "http://localhost:9200/_cluster/health?pretty" -Method Get
```

### Load Sample E-commerce Data
```powershell
$headers = @{ "Content-Type" = "application/json" }
$data = Get-Content -Path "datasets\ecommerce_sample.json" -Raw
Invoke-RestMethod -Uri "http://localhost:9200/ecommerce/_bulk?pretty" -Method Post -Headers $headers -Body $data
```

### OR load from  Kibana (/app/home#/tutorial_directory/sampleData)
![image](https://github.com/user-attachments/assets/e857a3ab-0ad7-4af7-8eeb-936566f63caf)



### Create Index with Custom Mapping
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

## Kibana Dashboards

Pre-built dashboards are available in the included Kibana installation that you can inspect to see how the visualizations were made. 

I will also detail building a dashboard with sample maritime data in this repo. 

![image](https://github.com/user-attachments/assets/21c7e688-b06a-4e4b-a5c9-0f3108ebf5a1)


## Exam Preparation Tips

1. **Hands-on Practice**: Complete all exercises in the practice file
2. **Documentation**: Familiarize yourself with the official Elasticsearch documentation
3. **Performance Tuning**: Focus on query optimization and cluster configuration
4. **Security**: Understand role-based access control and security features
5. **Troubleshooting**: Practice diagnosing and resolving common issues

## Study Resources

- [Official Elasticsearch Documentation](https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html)
- [Elastic Certified Engineer Exam Guide](https://www.elastic.co/training/certification)
- [Elasticsearch: The Definitive Guide](https://www.elastic.co/guide/en/elasticsearch/guide/current/index.html)


