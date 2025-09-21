# Troubleshooting Guide

This guide covers common issues and troubleshooting steps for the ITL Prometheus Stack.

## Table of Contents

- [Installation Issues](#installation-issues)
- [Storage Issues](#storage-issues)
- [Performance Issues](#performance-issues)
- [Networking Issues](#networking-issues)
- [Component-Specific Issues](#component-specific-issues)
- [Monitoring and Diagnostics](#monitoring-and-diagnostics)

## Installation Issues

### Chart Installation Failures

#### Issue: Helm install fails with CRD errors

```bash
Error: failed to install CRD: CustomResourceDefinition.apiextensions.k8s.io "prometheuses.monitoring.coreos.com" is invalid
```

**Solution:**
1. Ensure you have cluster-admin permissions
2. Install CRDs manually if needed:
   ```bash
   kubectl apply -f https://raw.githubusercontent.com/prometheus-operator/prometheus-operator/main/example/prometheus-operator-crd/monitoring.coreos.com_prometheuses.yaml
   ```
3. Enable CRD upgrade job:
   ```yaml
   crds:
     upgradeJob:
       enabled: true
   ```

#### Issue: Storage class not found

```bash
Error: persistentvolumeclaim "prometheus-data-..." is invalid: spec.storageClassName: Invalid value: "ha-gen-lh": storageclass.storage.k8s.io "ha-gen-lh" not found
```

**Solution:**
1. Check available storage classes:
   ```bash
   kubectl get storageclass
   ```
2. Create the required storage class or modify values.yaml:
   ```yaml
   prometheus:
     prometheusSpec:
       storageSpec:
         volumeClaimTemplate:
           spec:
             storageClassName: "your-existing-storage-class"
   ```

#### Issue: Insufficient permissions

```bash
Error: configmaps is forbidden: User "system:serviceaccount:default:default" cannot create resource "configmaps" in API group ""
```

**Solution:**
1. Verify RBAC is enabled:
   ```yaml
   global:
     rbac:
       create: true
   ```
2. Check service account permissions:
   ```bash
   kubectl auth can-i create configmaps --as=system:serviceaccount:monitoring:prometheus-operator
   ```

## Storage Issues

### PVC Stuck in Pending State

#### Diagnosis:
```bash
kubectl get pvc -n monitoring
kubectl describe pvc prometheus-data-itl-prometheus-kube-prometheus-prometheus-0 -n monitoring
```

#### Common Causes and Solutions:

1. **No available storage**
   ```bash
   # Check node storage
   kubectl describe nodes | grep -A 5 "Allocated resources"
   ```

2. **Storage class issues**
   ```bash
   # Verify storage class exists and is configured
   kubectl get storageclass ha-gen-lh -o yaml
   ```

3. **Resource quotas**
   ```bash
   # Check resource quotas
   kubectl describe quota -n monitoring
   ```

### Storage Performance Issues

#### Slow disk I/O affecting Prometheus

**Diagnosis:**
```bash
# Check disk usage and I/O
kubectl exec -n monitoring prometheus-itl-prometheus-kube-prometheus-prometheus-0 -- df -h
kubectl top pods -n monitoring --containers
```

**Solutions:**
1. Use SSD storage class
2. Increase IOPS allocation
3. Consider using local storage for better performance

## Performance Issues

### High Memory Usage

#### Prometheus consuming too much memory

**Diagnosis:**
```bash
# Check memory usage
kubectl top pods -n monitoring
kubectl exec -n monitoring prometheus-itl-prometheus-kube-prometheus-prometheus-0 -- \
  promtool query instant 'prometheus_tsdb_head_series'
```

**Solutions:**
1. Reduce retention period:
   ```yaml
   prometheus:
     prometheusSpec:
       retention: 15d  # Reduce from 30d
   ```

2. Implement recording rules to reduce query complexity
3. Increase memory limits:
   ```yaml
   prometheus:
     prometheusSpec:
       resources:
         limits:
           memory: 8Gi
   ```

### Slow Query Performance

#### PromQL queries timing out

**Diagnosis:**
```bash
# Check query performance metrics
kubectl port-forward -n monitoring svc/itl-prometheus-kube-prometheus-prometheus 9090:9090
# Visit http://localhost:9090/graph and check slow queries
```

**Solutions:**
1. Optimize PromQL queries
2. Add query timeout configuration:
   ```yaml
   prometheus:
     prometheusSpec:
       query:
         timeout: 2m
         maxConcurrency: 20
   ```

3. Use recording rules for complex queries

### High Cardinality Issues

#### Too many time series

**Diagnosis:**
```promql
# Check series count by job
topk(10, count by (__name__)({__name__=~".+"}))

# Check label cardinality
topk(10, count by (job)({__name__=~".+"}))
```

**Solutions:**
1. Implement metric relabeling:
   ```yaml
   prometheus:
     prometheusSpec:
       additionalScrapeConfigs:
         - job_name: 'high-cardinality-job'
           metric_relabel_configs:
             - source_labels: [__name__]
               regex: 'unnecessary_metric_.*'
               action: drop
   ```

2. Set sample limits:
   ```yaml
   prometheus:
     prometheusSpec:
       enforcedSampleLimit: 1000000
   ```

## Networking Issues

### Service Discovery Problems

#### ServiceMonitors not being discovered

**Diagnosis:**
```bash
# Check ServiceMonitor labels
kubectl get servicemonitor -n monitoring --show-labels

# Check Prometheus configuration
kubectl exec -n monitoring prometheus-itl-prometheus-kube-prometheus-prometheus-0 -- \
  cat /etc/prometheus/config_out/prometheus.env.yaml
```

**Solutions:**
1. Verify ServiceMonitor labels match selector:
   ```yaml
   apiVersion: monitoring.coreos.com/v1
   kind: ServiceMonitor
   metadata:
     labels:
       release: itl-prometheus  # Must match selector
   ```

2. Check namespace selectors:
   ```yaml
   prometheus:
     prometheusSpec:
       serviceMonitorNamespaceSelector: {}  # Allow all namespaces
   ```

### Ingress Configuration Issues

#### Cannot access Grafana/Prometheus via Ingress

**Diagnosis:**
```bash
# Check ingress status
kubectl get ingress -n monitoring
kubectl describe ingress grafana-ingress -n monitoring

# Check ingress controller logs
kubectl logs -n ingress-nginx deployment/nginx-ingress-controller
```

**Solutions:**
1. Verify ingress class:
   ```yaml
   grafana:
     ingress:
       enabled: true
       ingressClassName: "nginx"
   ```

2. Check DNS resolution and certificates

## Component-Specific Issues

### Grafana Issues

#### Grafana pod crashlooping

**Diagnosis:**
```bash
kubectl logs -n monitoring deployment/itl-prometheus-grafana --previous
```

**Common Solutions:**
1. Storage permission issues:
   ```bash
   kubectl exec -n monitoring deployment/itl-prometheus-grafana -- ls -la /var/lib/grafana
   ```

2. Configuration issues:
   ```yaml
   grafana:
     initChownData:
       enabled: true
   ```

#### Cannot login to Grafana

**Solutions:**
1. Reset admin password:
   ```bash
   kubectl get secret itl-prometheus-grafana -n monitoring -o jsonpath="{.data.admin-password}" | base64 --decode
   ```

2. Check admin user configuration:
   ```yaml
   grafana:
     adminUser: admin
     adminPassword: "your-password"
   ```

### Alertmanager Issues

#### Alerts not being sent

**Diagnosis:**
```bash
# Check Alertmanager configuration
kubectl exec -n monitoring alertmanager-itl-prometheus-kube-prometheus-alertmanager-0 -- \
  amtool config show

# Check Alertmanager logs
kubectl logs -n monitoring alertmanager-itl-prometheus-kube-prometheus-alertmanager-0
```

**Solutions:**
1. Verify receiver configuration:
   ```yaml
   alertmanager:
     config:
       receivers:
         - name: 'email'
           email_configs:
             - to: 'admin@example.com'
               from: 'alerts@example.com'
               smarthost: 'smtp.example.com:587'
   ```

2. Test configuration:
   ```bash
   kubectl exec -n monitoring alertmanager-itl-prometheus-kube-prometheus-alertmanager-0 -- \
     amtool check-config /etc/alertmanager/config/alertmanager.yml
   ```

## Monitoring and Diagnostics

### Health Checks

#### Check component health

```bash
# Prometheus health check
kubectl exec -n monitoring prometheus-itl-prometheus-kube-prometheus-prometheus-0 -- \
  wget -q --spider http://localhost:9090/-/healthy && echo "Healthy" || echo "Unhealthy"

# Grafana health check  
kubectl exec -n monitoring deployment/itl-prometheus-grafana -- \
  curl -f http://localhost:3000/api/health || echo "Unhealthy"

# Alertmanager health check
kubectl exec -n monitoring alertmanager-itl-prometheus-kube-prometheus-alertmanager-0 -- \
  wget -q --spider http://localhost:9093/-/healthy && echo "Healthy" || echo "Unhealthy"
```

### Resource Monitoring

#### Monitor resource usage

```bash
# Check resource usage
kubectl top pods -n monitoring
kubectl describe nodes | grep -A 10 "Non-terminated Pods"

# Check storage usage
kubectl exec -n monitoring prometheus-itl-prometheus-kube-prometheus-prometheus-0 -- df -h
```

### Log Analysis

#### Centralized logging for troubleshooting

```bash
# Prometheus logs
kubectl logs -n monitoring prometheus-itl-prometheus-kube-prometheus-prometheus-0 --tail=100

# Grafana logs
kubectl logs -n monitoring deployment/itl-prometheus-grafana --tail=100

# Alertmanager logs
kubectl logs -n monitoring alertmanager-itl-prometheus-kube-prometheus-alertmanager-0 --tail=100

# Operator logs
kubectl logs -n monitoring deployment/itl-prometheus-kube-prometheus-operator --tail=100
```

## Diagnostic Tools

### Useful Commands

```bash
# Get all resources in monitoring namespace
kubectl get all -n monitoring

# Check events for issues
kubectl get events -n monitoring --sort-by='.lastTimestamp'

# Export current configuration
helm get values itl-prometheus -n monitoring > current-values.yaml

# Check Helm release status
helm status itl-prometheus -n monitoring

# Validate Prometheus configuration
kubectl exec -n monitoring prometheus-itl-prometheus-kube-prometheus-prometheus-0 -- \
  promtool check config /etc/prometheus/config_out/prometheus.env.yaml
```

### Debug Mode

Enable debug logging:

```yaml
prometheus:
  prometheusSpec:
    logLevel: debug

grafana:
  grafana.ini:
    log:
      level: debug

alertmanager:
  alertmanagerSpec:
    logLevel: debug
```

## Getting Help

### Community Resources

- [Prometheus Community](https://prometheus.io/community/)
- [Grafana Community](https://community.grafana.com/)
- [Kubernetes Slack #prometheus-operator](https://kubernetes.slack.com/)

### Collecting Information for Support

When seeking help, collect this information:

```bash
# System information
kubectl version
helm version

# Chart information
helm list -n monitoring
helm get values itl-prometheus -n monitoring

# Resource status
kubectl get pods,pvc,configmaps,secrets -n monitoring
kubectl describe pods -n monitoring

# Logs
kubectl logs -n monitoring deployment/itl-prometheus-kube-prometheus-operator --tail=200
```

## Prevention

### Best Practices

1. **Regular Monitoring**: Set up alerts for the monitoring stack itself
2. **Resource Planning**: Monitor resource usage trends
3. **Configuration Validation**: Test configuration changes in staging
4. **Backup Strategy**: Regular backup of Grafana dashboards and Prometheus data
5. **Documentation**: Keep configuration changes documented

### Automated Monitoring

Set up alerts for monitoring stack health:

```yaml
additionalPrometheusRulesMap:
  monitoring-stack-health:
    groups:
      - name: monitoring.rules
        rules:
          - alert: PrometheusDown
            expr: up{job="prometheus"} == 0
            for: 5m
            
          - alert: GrafanaDown
            expr: up{job="grafana"} == 0
            for: 5m
            
          - alert: AlertmanagerDown
            expr: up{job="alertmanager"} == 0
            for: 5m
```

This troubleshooting guide should help you resolve most common issues with the ITL Prometheus Stack. For complex issues, consider enabling debug logging and analyzing the detailed output.