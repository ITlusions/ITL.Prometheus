# ITL.Prometheus

This Helm chart provides a complete monitoring and observability stack using the kube-prometheus-stack, which includes Prometheus, Grafana, and Alertmanager. This documentation covers the integration and usage of Loki for log aggregation as part of your observability strategy.

## Loki Integration

### What is Loki?

Loki is a horizontally-scalable, highly-available, multi-tenant log aggregation system inspired by Prometheus. It is designed to be very cost-effective and easy to operate. Unlike other logging systems, Loki is built around the idea of only indexing metadata about your logs (labels), not the contents of the log lines themselves.

### Key Features

- **Cost-effective**: Only indexes metadata, not log content
- **Scalable**: Horizontally scalable architecture  
- **Grafana Integration**: Native integration with Grafana for log visualization
- **PromQL-like**: Uses LogQL, a query language similar to PromQL
- **Multi-tenant**: Built-in multi-tenancy support

### Integration with Prometheus Stack

Loki complements the existing Prometheus monitoring stack:

- **Prometheus**: Collects and stores metrics
- **Grafana**: Visualizes both metrics and logs in unified dashboards  
- **Alertmanager**: Handles alerts from both Prometheus and Loki
- **Loki**: Aggregates and stores logs

This creates a complete observability solution where you can correlate metrics and logs in a single interface.

## Installing Loki

### Option 1: Using Loki Helm Chart

Add the Grafana Helm repository and install Loki:

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

# Install Loki
helm install loki grafana/loki-stack \
  --set grafana.enabled=false \
  --set prometheus.enabled=false \
  --set filebeat.enabled=false \
  --set logstash.enabled=false \
  --namespace monitoring \
  --create-namespace
```

### Option 2: Loki with Promtail

For a complete log collection setup with Promtail:

```bash
helm install loki grafana/loki-stack \
  --set grafana.enabled=false \
  --set prometheus.enabled=false \
  --set promtail.enabled=true \
  --namespace monitoring \
  --create-namespace
```

## Configuration

### Grafana Data Source Configuration

To add Loki as a data source in Grafana, you can configure it through the Grafana UI or via configuration:

```yaml
datasources:
  - name: Loki
    type: loki
    access: proxy
    url: http://loki:3100
    isDefault: false
```

### Log Collection Configuration

#### Promtail Configuration

Promtail is the recommended log collection agent for Loki. Example configuration:

```yaml
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
  - job_name: containers
    static_configs:
      - targets:
          - localhost
        labels:
          job: containerlogs
          __path__: /var/log/containers/*log
```

#### Application Log Configuration

For applications running in Kubernetes, configure logging with proper labels:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-logging-config
data:
  config.yaml: |
    logging:
      level: INFO
      format: json
      labels:
        app: myapp
        environment: production
        version: v1.0.0
```

## Usage Examples

### Basic Log Queries

Use LogQL to query logs in Grafana:

```logql
# Get all logs for a specific app
{app="myapp"}

# Filter logs by level
{app="myapp"} |= "ERROR"

# Count error logs over time
count_over_time({app="myapp"} |= "ERROR"[5m])

# Extract JSON fields
{app="myapp"} | json | status="404"
```

### Advanced Queries

```logql
# Rate of error logs
rate({app="myapp"} |= "ERROR"[5m])

# Top 10 error messages
topk(10, 
  count by (message) (
    {app="myapp"} |= "ERROR" | json
  )
)

# Logs from specific pods
{pod=~"myapp-.*"}
```

### Alerting with Loki

Create alerts based on log patterns:

```yaml
groups:
  - name: loki-alerts
    rules:
      - alert: HighErrorRate
        expr: |
          rate({app="myapp"} |= "ERROR"[5m]) > 0.1
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "High error rate detected in {{ $labels.app }}"
          description: "Error rate is {{ $value }} errors per second"
```

## Best Practices

### Labeling Strategy

1. **Use consistent labels**: Apply the same labels across metrics and logs
2. **Limit label cardinality**: Avoid high-cardinality labels (like user IDs)
3. **Use meaningful labels**: Include environment, service, version, etc.

```yaml
labels:
  environment: production
  service: user-api
  version: v2.1.0
  team: backend
```

### Log Format Recommendations

1. **Use structured logging**: JSON format is preferred
2. **Include correlation IDs**: For tracing requests across services
3. **Consistent timestamp format**: Use ISO 8601 format

```json
{
  "timestamp": "2024-01-15T10:30:00Z",
  "level": "ERROR",
  "message": "Database connection failed",
  "service": "user-api",
  "version": "v2.1.0",
  "trace_id": "abc123",
  "error": {
    "type": "ConnectionError",
    "message": "Timeout after 5s"
  }
}
```

### Performance Optimization

1. **Configure retention policies**: Set appropriate log retention periods
2. **Use log sampling**: For high-volume applications
3. **Optimize queries**: Use specific label selectors
4. **Monitor Loki performance**: Set up monitoring for Loki itself

### Storage Configuration

```yaml
loki:
  config:
    schema_config:
      configs:
        - from: 2020-10-24
          store: boltdb-shipper
          object_store: s3
          schema: v11
          index:
            prefix: index_
            period: 24h
    storage_config:
      boltdb_shipper:
        active_index_directory: /loki/boltdb-shipper-active
        cache_location: /loki/boltdb-shipper-cache
        shared_store: s3
      aws:
        s3: s3://my-loki-bucket
        region: us-east-1
```

## Troubleshooting

### Common Issues

1. **Logs not appearing**: Check Promtail configuration and network connectivity
2. **High memory usage**: Review label cardinality and query patterns
3. **Slow queries**: Optimize LogQL queries with specific label selectors
4. **Missing correlation**: Ensure consistent labeling between metrics and logs

### Debugging Commands

```bash
# Check Loki status
kubectl get pods -n monitoring -l app=loki

# View Loki logs
kubectl logs -n monitoring deployment/loki

# Check Promtail logs
kubectl logs -n monitoring daemonset/promtail

# Test Loki API
curl http://loki:3100/ready
```

## Integration Examples

### Grafana Dashboard

Create dashboards that combine metrics and logs:

```json
{
  "panels": [
    {
      "title": "Application Metrics",
      "targets": [
        {
          "expr": "rate(http_requests_total[5m])",
          "datasource": "Prometheus"
        }
      ]
    },
    {
      "title": "Application Logs",
      "targets": [
        {
          "expr": "{app=\"myapp\"}",
          "datasource": "Loki"
        }
      ]
    }
  ]
}
```

### Correlation Queries

Link metrics and logs using common labels:

```logql
# Show logs for instances with high CPU usage
{instance=~"$(instance)"} 
  and on(instance) 
  (rate(cpu_usage[5m]) > 0.8)
```

## Further Resources

- [Loki Documentation](https://grafana.com/docs/loki/)
- [LogQL Documentation](https://grafana.com/docs/loki/latest/logql/)
- [Promtail Configuration](https://grafana.com/docs/loki/latest/clients/promtail/)
- [Grafana Loki Integration](https://grafana.com/docs/grafana/latest/datasources/loki/)

## Contributing

When contributing to this monitoring stack:

1. Follow labeling conventions
2. Update documentation for any Loki configuration changes
3. Test log queries and alerting rules
4. Consider performance impact of new log sources
