# ELK Stack Setup Guide

Hướng dẫn triển khai và sử dụng ELK stack (Elasticsearch, Filebeat, Kibana) cho dự án Threads.

## 📋 Tổng Quan

ELK stack được triển khai để thu thập, lưu trữ và phân tích logs từ tất cả services:
- **Elasticsearch**: Lưu trữ logs
- **Filebeat**: Thu thập logs từ containers
- **Kibana**: Web UI để query và visualize logs

## 🚀 Deployment

### 1. Deploy Elasticsearch

```bash
# Deploy Elasticsearch
kubectl apply -k manifests/overlays/dev/elasticsearch

# Verify deployment
kubectl get pods -l app=elasticsearch
kubectl get pvc | grep elasticsearch

# Wait for Elasticsearch to be ready (có thể mất 2-3 phút)
kubectl wait --for=condition=ready pod -l app=elasticsearch --timeout=300s
```

### 2. Deploy Filebeat

```bash
# Deploy Filebeat DaemonSet
kubectl apply -k manifests/overlays/dev/filebeat

# Verify deployment (should have 1 pod per node)
kubectl get daemonset filebeat
kubectl get pods -l app=filebeat

# Check Filebeat logs
kubectl logs -l app=filebeat --tail=50
```

### 3. Deploy Kibana

```bash
# Deploy Kibana
kubectl apply -k manifests/overlays/dev/kibana

# Verify deployment
kubectl get pods -l app=kibana

# Wait for Kibana to be ready
kubectl wait --for=condition=ready pod -l app=kibana --timeout=300s
```

### 4. Access Kibana

**Option 1: Port Forward (Local Development)**
```bash
kubectl port-forward svc/kibana 5601:5601

# Open browser: http://localhost:5601
```

**Option 2: Ingress (Production)**
```bash
# Access via domain
# URL: http://kibana.namth.online
```

## 🔧 Initial Configuration

### 1. Create Index Patterns

1. Mở Kibana: http://localhost:5601
2. Vào **Management** → **Stack Management** → **Index Patterns**
3. Click **Create index pattern**
4. Nhập pattern: `filebeat-*`
5. Chọn time field: `@timestamp`
6. Click **Create index pattern**

### 2. Apply ILM Policies

```bash
# Port forward to Elasticsearch
kubectl port-forward svc/elasticsearch 9200:9200

# Apply ILM policies
curl -X PUT "http://localhost:9200/_ilm/policy/filebeat-api-policy" \
  -H 'Content-Type: application/json' \
  -d '{
    "policy": {
      "phases": {
        "hot": {
          "actions": {
            "rollover": {
              "max_age": "1d",
              "max_size": "5gb"
            }
          }
        },
        "delete": {
          "min_age": "30d",
          "actions": {
            "delete": {}
          }
        }
      }
    }
  }'

# Tương tự cho các policies khác (postgres, redis, minio, web)
# Hoặc sử dụng script có sẵn:
kubectl exec -it elasticsearch-0 -- bash
# Trong container:
cd /policies
bash apply-policies.sh
```

### 3. Verify Log Collection

```bash
# Check indices trong Elasticsearch
kubectl exec -it elasticsearch-0 -- curl -X GET "localhost:9200/_cat/indices?v"

# Expected output:
# filebeat-api-2025.11.26
# filebeat-postgres-2025.11.26
# filebeat-redis-2025.11.26
# filebeat-minio-2025.11.26
# filebeat-web-2025.11.26
```

## 🔍 Using Kibana

### Discover Logs

1. Vào **Discover** trong Kibana
2. Chọn index pattern: `filebeat-*`
3. Chọn time range (ví dụ: Last 15 minutes)

### Query Examples

**Xem logs của API service:**
```
service: "api"
```

**Xem tất cả errors:**
```
level: "error"
```

**Xem logs của PostgreSQL:**
```
service: "postgres"
```

**Xem API logs có chứa "login":**
```
service: "api" AND message: *login*
```

**Xem logs từ specific pod:**
```
kubernetes.pod.name: "api-7d8f9c-xyz"
```

### Create Visualizations

1. Vào **Visualize Library**
2. Click **Create visualization**
3. Chọn visualization type (Line, Bar, Pie, etc.)
4. Chọn index pattern: `filebeat-*`
5. Configure metrics và buckets
6. Save visualization

### Create Dashboards

1. Vào **Dashboard**
2. Click **Create dashboard**
3. Add visualizations đã tạo
4. Arrange layout
5. Save dashboard

## 📊 Monitoring

### Check Elasticsearch Health

```bash
kubectl exec -it elasticsearch-0 -- curl -X GET "localhost:9200/_cluster/health?pretty"
```

Expected output:
```json
{
  "cluster_name" : "threads-elasticsearch",
  "status" : "green",  // or "yellow" for single node
  "number_of_nodes" : 1,
  "number_of_data_nodes" : 1
}
```

### Check Storage Usage

```bash
# Check PVC usage
kubectl exec -it elasticsearch-0 -- df -h /usr/share/elasticsearch/data

# Check indices size
kubectl exec -it elasticsearch-0 -- curl -X GET "localhost:9200/_cat/indices?v&h=index,store.size&s=store.size:desc"
```

### Check Filebeat Status

```bash
# Check DaemonSet
kubectl get daemonset filebeat

# Check logs
kubectl logs -l app=filebeat --tail=100

# Expected: "Connection to backoff(elasticsearch..." successful
```

## 🔧 Troubleshooting

### Elasticsearch Pod Not Starting

```bash
# Check pod status
kubectl describe pod elasticsearch-0

# Common issues:
# 1. Insufficient storage
kubectl get pvc

# 2. Memory issues
kubectl top pod elasticsearch-0

# 3. Check logs
kubectl logs elasticsearch-0
```

### Filebeat Not Collecting Logs

```bash
# Check Filebeat logs
kubectl logs -l app=filebeat --tail=100

# Common issues:
# 1. Cannot connect to Elasticsearch
# Check: "Connection to backoff(elasticsearch..." in logs

# 2. RBAC permissions
kubectl get clusterrolebinding filebeat

# 3. Verify autodiscover
kubectl logs -l app=filebeat | grep autodiscover
```

### No Logs in Kibana

```bash
# 1. Check if indices exist
kubectl exec -it elasticsearch-0 -- curl -X GET "localhost:9200/_cat/indices?v"

# 2. Check if logs are being ingested
kubectl exec -it elasticsearch-0 -- curl -X GET "localhost:9200/filebeat-*/_count"

# 3. Verify time range in Kibana (try "Last 24 hours")

# 4. Check if services are generating logs
kubectl logs -l app=api --tail=10
```

### Kibana Cannot Connect to Elasticsearch

```bash
# Check Kibana logs
kubectl logs -l app=kibana

# Verify Elasticsearch service
kubectl get svc elasticsearch

# Test connection from Kibana pod
kubectl exec -it <kibana-pod> -- curl http://elasticsearch:9200
```

## 🗑️ Maintenance

### Manual Index Deletion

```bash
# Delete old indices
kubectl exec -it elasticsearch-0 -- curl -X DELETE "localhost:9200/filebeat-api-2025.11.01"

# Delete indices older than 30 days
kubectl exec -it elasticsearch-0 -- bash
# In container:
curl -X GET "localhost:9200/_cat/indices/filebeat-*?h=index" | \
  while read index; do
    # Parse date and delete if older than 30 days
    # (implement date logic as needed)
  done
```

### Backup Elasticsearch Data

```bash
# Create snapshot repository (using MinIO)
kubectl exec -it elasticsearch-0 -- curl -X PUT "localhost:9200/_snapshot/minio_backup" \
  -H 'Content-Type: application/json' \
  -d '{
    "type": "s3",
    "settings": {
      "bucket": "elasticsearch-backups",
      "endpoint": "minio:9000",
      "protocol": "http"
    }
  }'

# Create snapshot
kubectl exec -it elasticsearch-0 -- curl -X PUT "localhost:9200/_snapshot/minio_backup/snapshot_1"
```

## 🔄 Rollback

Nếu cần xóa ELK stack:

```bash
# Delete all components
kubectl delete -k manifests/overlays/dev/kibana
kubectl delete -k manifests/overlays/dev/filebeat
kubectl delete -k manifests/overlays/dev/elasticsearch

# Delete PVC (WARNING: This deletes all logs!)
kubectl delete pvc -l app=elasticsearch
```

## 📚 Resources

- [Elasticsearch Documentation](https://www.elastic.co/guide/en/elasticsearch/reference/current/index.html)
- [Filebeat Documentation](https://www.elastic.co/guide/en/beats/filebeat/current/index.html)
- [Kibana Documentation](https://www.elastic.co/guide/en/kibana/current/index.html)
