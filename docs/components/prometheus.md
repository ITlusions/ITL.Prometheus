# Prometheus Component

Prometheus is the core time-series database and monitoring server in the ITL Prometheus Stack.

## Overview

Prometheus is responsible for:
- **Metrics Collection**: Scraping metrics from various targets
- **Data Storage**: Storing time-series data locally
- **Query Processing**: Executing PromQL queries
- **Alert Evaluation**: Evaluating alerting rules

## Configuration

### Default Configuration

```yaml
prometheus:
  enabled: true
  prometheusSpec:
    image:
      registry: quay.io
      repository: prometheus/prometheus
      tag: ""  # Uses chart's default version
    
    # Instance configuration
    replicas: 1
    shards: 1
    
    # Data retention
    retention: 10d
    retentionSize: ""
    
    # Storage
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: "ha-gen-lh"
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 50Gi
```

### Key Configuration Options

#### Data Retention

```yaml
prometheus:
  prometheusSpec:
    retention: 30d              # Time-based retention
    retentionSize: "45GB"       # Size-based retention
```

#### Scaling Configuration

```yaml
prometheus:
  prometheusSpec:
    replicas: 2                 # Number of Prometheus replicas
    shards: 1                   # Number of shards (experimental)
```

#### Resource Configuration

```yaml
prometheus:
  prometheusSpec:
    resources:
      requests:
        memory: 2Gi
        cpu: 1000m
      limits:
        memory: 4Gi
        cpu: 2000m
```

## Service Discovery

### ServiceMonitor

ServiceMonitors define how Prometheus discovers and scrapes services:

```yaml
prometheus:
  prometheusSpec:
    serviceMonitorSelectorNilUsesHelmValues: true
    serviceMonitorSelector: {}
    serviceMonitorNamespaceSelector: {}
```

Example ServiceMonitor:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: my-app
  labels:
    release: itl-prometheus
spec:
  selector:
    matchLabels:
      app: my-app
  endpoints:
  - port: metrics
    interval: 30s
    path: /metrics
```

### PodMonitor

PodMonitors discover and scrape pods directly:

```yaml
prometheus:
  prometheusSpec:
    podMonitorSelectorNilUsesHelmValues: true
    podMonitorSelector: {}
    podMonitorNamespaceSelector: {}
```

### Additional Scrape Configs

For custom scraping configurations:

```yaml
prometheus:
  prometheusSpec:
    additionalScrapeConfigs:
      - job_name: 'custom-endpoints'
        static_configs:
          - targets: ['custom-service:8080']
        scrape_interval: 30s
        metrics_path: /custom/metrics
```

## Alerting Rules

### PrometheusRule

Alerting rules are defined using PrometheusRule resources:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: custom-alerts
  labels:
    release: itl-prometheus
spec:
  groups:
  - name: custom.rules
    rules:
    - alert: HighCPUUsage
      expr: rate(cpu_usage_seconds_total[5m]) > 0.8
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "High CPU usage detected"
        description: "CPU usage is above 80% for 5 minutes"
```

### Default Rules Configuration

```yaml
defaultRules:
  create: true
  rules:
    # Enable/disable specific rule groups
    alertmanager: true
    etcd: true
    general: true
    prometheus: true
    node: true
    
  # Custom rule labels and annotations
  additionalRuleLabels: {}
  additionalRuleAnnotations: {}
```

## Remote Storage

### Remote Write

Configure remote write for long-term storage:

```yaml
prometheus:
  prometheusSpec:
    remoteWrite:
      - url: "https://remote-storage.example.com/api/v1/write"
        writeRelabelConfigs:
          - sourceLabels: [__name__]
            regex: "prometheus_.*"
            action: drop
        queueConfig:
          capacity: 10000
          maxShards: 50
```

### Remote Read

Configure remote read for querying external data:

```yaml
prometheus:
  prometheusSpec:
    remoteRead:
      - url: "https://remote-storage.example.com/api/v1/read"
        readRecent: true
```

## Security

### Authentication and Authorization

```yaml
prometheus:
  prometheusSpec:
    # Security context
    securityContext:
      runAsGroup: 2000
      runAsNonRoot: true
      runAsUser: 1000
      fsGroup: 2000
    
    # Web configuration for TLS/Auth
    web:
      tlsConfig:
        cert:
          secret:
            name: prometheus-tls
            key: tls.crt
        keySecret:
          name: prometheus-tls
          key: tls.key
```

### Network Policies

```yaml
prometheus:
  networkPolicy:
    enabled: true
    policyTypes:
      - Ingress
      - Egress
    ingress:
      - from:
          - namespaceSelector:
              matchLabels:
                name: monitoring
    egress:
      - to: []
        ports:
          - protocol: TCP
            port: 443
```

## Performance Tuning

### Query Performance

```yaml
prometheus:
  prometheusSpec:
    # Query configuration
    query:
      timeout: 2m
      maxConcurrency: 20
      maxSamples: 50000000
    
    # TSDB configuration
    tsdb:
      outOfOrderTimeWindow: 0s
    
    # WAL compression
    walCompression: true
```

### Cardinality Management

```yaml
prometheus:
  prometheusSpec:
    # Sample limits
    enforcedSampleLimit: 0
    enforcedTargetLimit: 0
    enforcedLabelLimit: 0
    
    # Per-scrape limits
    sampleLimit: false
```

## Monitoring Prometheus

### Key Metrics

Monitor Prometheus itself with these metrics:

```promql
# Prometheus rule evaluation duration
prometheus_rule_evaluation_duration_seconds

# Number of samples ingested
prometheus_tsdb_symbol_table_size_bytes

# WAL size
prometheus_tsdb_wal_size_bytes

# Storage usage
prometheus_tsdb_head_series

# Query performance  
prometheus_engine_query_duration_seconds
```

### Health Checks

Prometheus provides several health endpoints:

- `/-/healthy`: General health check
- `/-/ready`: Readiness check
- `/api/v1/status/runtimeinfo`: Runtime information
- `/api/v1/status/buildinfo`: Build information

### Common Alerts

Key alerts to monitor Prometheus health:

```yaml
- alert: PrometheusConfigurationReloadFailure
  expr: prometheus_config_last_reload_successful != 1
  for: 10m
  
- alert: PrometheusNotificationQueueRunningFull
  expr: prometheus_notifications_queue_length / prometheus_notifications_queue_capacity > 0.9
  for: 5m

- alert: PrometheusTSDBReloadsFailing
  expr: increase(prometheus_tsdb_reloads_failures_total[3h]) > 0
  for: 4h
```

## Troubleshooting

### Common Issues

1. **High Memory Usage**
   ```bash
   # Check series count
   kubectl exec -n monitoring prometheus-itl-prometheus-kube-prometheus-prometheus-0 -- \
     promtool query instant 'prometheus_tsdb_head_series'
   ```

2. **Slow Queries**
   ```bash
   # Check query performance
   kubectl logs -n monitoring prometheus-itl-prometheus-kube-prometheus-prometheus-0 | \
     grep "query_duration_seconds"
   ```

3. **Storage Issues**
   ```bash
   # Check storage usage
   kubectl exec -n monitoring prometheus-itl-prometheus-kube-prometheus-prometheus-0 -- \
     df -h /prometheus
   ```

### Debugging

Enable debug logging:

```yaml
prometheus:
  prometheusSpec:
    logLevel: debug
```

Access Prometheus web UI:

```bash
kubectl port-forward -n monitoring svc/itl-prometheus-kube-prometheus-prometheus 9090:9090
```

Then visit: http://localhost:9090

## Best Practices

### Resource Planning

1. **Memory**: ~1-2GB per million active series
2. **CPU**: Depends on query complexity and frequency
3. **Storage**: ~1-2 bytes per sample on disk
4. **Retention**: Balance between storage cost and data needs

### Configuration

1. **Use ServiceMonitors**: Instead of static configs when possible
2. **Label Management**: Keep cardinality under control
3. **Regular Backups**: Implement backup strategy for critical data
4. **Monitor the Monitor**: Set up alerts for Prometheus itself

### High Availability

For production deployments:

```yaml
prometheus:
  prometheusSpec:
    replicas: 2
    podAntiAffinity: "hard"
    affinity:
      podAntiAffinity:
        requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchLabels:
              app.kubernetes.io/name: prometheus
          topologyKey: kubernetes.io/hostname
```

## References

- [Prometheus Documentation](https://prometheus.io/docs/)
- [Prometheus Operator API](https://github.com/prometheus-operator/prometheus-operator/blob/main/Documentation/api-reference/api.md)
- [PromQL Guide](https://prometheus.io/docs/prometheus/latest/querying/basics/)
- [Storage Configuration](../storage.md)
- [Configuration Reference](../configuration.md)