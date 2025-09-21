# Storage Configuration

This document covers storage configuration for the ITL Prometheus Stack, including persistent volume setup and storage class requirements.

## Overview

The ITL Prometheus Stack requires persistent storage for three main components:
- **Prometheus**: For metrics data storage
- **Grafana**: For dashboards and configuration
- **Alertmanager**: For alert state and notification history

## Default Storage Configuration

### Storage Class

All components are configured to use the **`ha-gen-lh`** storage class by default:

```yaml
# Prometheus storage
prometheus:
  prometheusSpec:
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: "ha-gen-lh"
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 50Gi

# Grafana storage  
grafana:
  persistence:
    enabled: true
    type: pvc
    storageClassName: "ha-gen-lh"
    accessModes:
      - ReadWriteOnce
    size: 10Gi

# Alertmanager storage
alertmanager:
  alertmanagerSpec:
    storage:
      volumeClaimTemplate:
        spec:
          storageClassName: ha-gen-lh
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 10Gi
```

### Storage Sizes

| Component | Default Size | Purpose |
|-----------|-------------|---------|
| Prometheus | 50Gi | Time-series metrics data |
| Grafana | 10Gi | Dashboards, users, settings |
| Alertmanager | 10Gi | Alert state and history |
| **Total** | **70Gi** | **Combined storage requirement** |

## Storage Class Requirements

### ha-gen-lh Storage Class

The `ha-gen-lh` storage class should provide:
- **High Availability**: Replicated storage across multiple nodes
- **Good Performance**: SSD-backed storage recommended
- **Reliable**: Suitable for production workloads

### Verifying Storage Class

Check if the storage class exists:

```bash
kubectl get storageclass ha-gen-lh
```

View storage class details:

```bash
kubectl describe storageclass ha-gen-lh
```

## Alternative Storage Configurations

### Using Different Storage Classes

To use a different storage class, override the values:

```yaml
# Override storage class for all components
prometheus:
  prometheusSpec:
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: "your-storage-class"

grafana:
  persistence:
    storageClassName: "your-storage-class"

alertmanager:
  alertmanagerSpec:
    storage:
      volumeClaimTemplate:
        spec:
          storageClassName: "your-storage-class"
```

### Adjusting Storage Sizes

For production environments, you may need larger storage:

```yaml
prometheus:
  prometheusSpec:
    storageSpec:
      volumeClaimTemplate:
        spec:
          resources:
            requests:
              storage: 200Gi  # Increased for production

grafana:
  persistence:
    size: 20Gi  # More space for dashboards

alertmanager:
  alertmanagerSpec:
    storage:
      volumeClaimTemplate:
        spec:
          resources:
            requests:
              storage: 20Gi  # More space for alert history
```

### Performance Considerations

For high-performance requirements:

```yaml
prometheus:
  prometheusSpec:
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: "high-performance-ssd"
          resources:
            requests:
              storage: 100Gi
```

## Storage Monitoring

### Disk Usage Monitoring

The stack includes built-in alerts for storage usage:

- **PrometheusTargetDown**: When Prometheus targets are down
- **PrometheusTSDBReloadsFailing**: When TSDB reloads fail
- **PrometheusNotConnectedToAlertmanagers**: Connection issues
- **PrometheusTSDBCompactionsFailing**: Compaction failures

### Storage Metrics

Key metrics to monitor:

```promql
# Prometheus storage usage
prometheus_tsdb_symbol_table_size_bytes
prometheus_tsdb_head_series
prometheus_tsdb_wal_size_bytes

# Available disk space
node_filesystem_avail_bytes{mountpoint="/prometheus"}

# Grafana storage usage  
node_filesystem_avail_bytes{mountpoint="/var/lib/grafana"}
```

## Backup and Recovery

### Prometheus Data

Prometheus data backup strategies:

1. **Snapshot-based**: Use storage-level snapshots
2. **Remote Storage**: Configure remote write to external systems
3. **Federation**: Set up Prometheus federation for data redundancy

```yaml
# Configure remote write for backup
prometheus:
  prometheusSpec:
    remoteWrite:
      - url: "https://your-remote-storage/api/v1/write"
        writeRelabelConfigs:
          - sourceLabels: [__name__]
            regex: "prometheus_.*"
            action: drop
```

### Grafana Backup

Grafana backup includes:
- Dashboard definitions
- Data sources
- Users and organizations
- Plugin configurations

```bash
# Backup Grafana data
kubectl exec -n monitoring deployment/itl-prometheus-grafana -- \
  tar czf - /var/lib/grafana > grafana-backup.tar.gz
```

## Troubleshooting Storage Issues

### Common Issues

1. **PVC Stuck in Pending**
   ```bash
   kubectl describe pvc -n monitoring
   # Check events for storage class issues
   ```

2. **Storage Class Not Found**
   ```bash
   kubectl get storageclass
   # Verify ha-gen-lh exists
   ```

3. **Insufficient Storage**
   ```bash
   kubectl get pvc -n monitoring
   # Check storage usage and limits
   ```

### Volume Expansion

If your storage class supports volume expansion:

```bash
# Edit PVC to increase size
kubectl edit pvc prometheus-data-itl-prometheus-kube-prometheus-prometheus-0 -n monitoring

# Update the storage request
spec:
  resources:
    requests:
      storage: 100Gi  # Increased size
```

### Storage Performance Issues

Monitor storage performance:

```bash
# Check I/O metrics
kubectl top pods -n monitoring --containers
```

## Best Practices

### Production Recommendations

1. **Use SSD Storage**: For better performance
2. **Enable Volume Snapshots**: For backup and recovery
3. **Monitor Disk Usage**: Set up alerts for storage thresholds
4. **Regular Cleanup**: Configure retention policies
   
```yaml
prometheus:
  prometheusSpec:
    retention: 30d  # Adjust based on requirements
    retentionSize: "45GB"  # Leave some buffer space
```

5. **Test Restore Procedures**: Regularly test backup/restore

### Resource Planning

Calculate storage needs based on:
- **Metrics Cardinality**: Number of unique metric series
- **Scrape Interval**: How often metrics are collected
- **Retention Period**: How long to keep data
- **Growth Rate**: Expected increase in monitored services

Formula for Prometheus storage:
```
Storage = (Ingestion Rate * Retention Period * Compression Factor)
```

Where:
- Ingestion Rate: samples/second
- Retention Period: seconds
- Compression Factor: ~0.5 (typical compression ratio)

## Next Steps

- [Configuration Reference](configuration.md) - Advanced configuration options
- [Components Documentation](components/) - Component-specific storage details
- [Troubleshooting](troubleshooting.md) - Storage troubleshooting guide