# Configuration Reference

This document provides a comprehensive reference for configuring the ITL Prometheus Stack through the values.yaml file.

## Table of Contents

- [Global Configuration](#global-configuration)
- [Prometheus Configuration](#prometheus-configuration)
- [Grafana Configuration](#grafana-configuration)
- [Alertmanager Configuration](#alertmanager-configuration)
- [Storage Configuration](#storage-configuration)
- [Security Configuration](#security-configuration)
- [Network Configuration](#network-configuration)
- [Resource Configuration](#resource-configuration)

## Global Configuration

### Chart Metadata

```yaml
# Chart name and version override
nameOverride: ""
fullnameOverride: ""

# Common labels applied to all resources
commonLabels: {}
  # environment: production
  # team: platform
```

### CRD Management

```yaml
crds:
  enabled: true
  upgradeJob:
    enabled: false  # Set to true for automated CRD upgrades
    forceConflicts: false
```

## Prometheus Configuration

### Basic Prometheus Settings

```yaml
prometheus:
  enabled: true
  
  prometheusSpec:
    # Prometheus image configuration
    image:
      registry: quay.io
      repository: prometheus/prometheus
      tag: ""  # Uses chart default
    
    # Log configuration
    logLevel: info
    logFormat: logfmt
    
    # Replica and sharding
    replicas: 1
    shards: 1
    
    # Data retention
    retention: 10d
    retentionSize: ""
    
    # Web configuration
    routePrefix: /
    externalUrl: ""
```

### Prometheus Storage

```yaml
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
```

### Service Discovery and Monitoring

```yaml
prometheus:
  prometheusSpec:
    # ServiceMonitor selection
    serviceMonitorSelectorNilUsesHelmValues: true
    serviceMonitorSelector: {}
    serviceMonitorNamespaceSelector: {}
    
    # PodMonitor selection
    podMonitorSelectorNilUsesHelmValues: true
    podMonitorSelector: {}
    podMonitorNamespaceSelector: {}
    
    # PrometheusRule selection
    ruleSelectorNilUsesHelmValues: true
    ruleSelector: {}
    ruleNamespaceSelector: {}
```

### Remote Write/Read

```yaml
prometheus:
  prometheusSpec:
    # Remote write configuration
    remoteWrite: []
    # - url: "https://remote-storage/api/v1/write"
    #   writeRelabelConfigs:
    #     - sourceLabels: [__name__]
    #       regex: "prometheus_.*"
    #       action: drop
    
    # Remote read configuration  
    remoteRead: []
    # - url: "https://remote-storage/api/v1/read"
```

## Grafana Configuration

### Basic Grafana Settings

```yaml
grafana:
  enabled: true
  
  # Admin credentials
  adminUser: admin
  adminPassword: prom-operator  # Change in production!
  
  # Default dashboards
  defaultDashboardsEnabled: true
  defaultDashboardsTimezone: utc
  defaultDashboardsEditable: true
  defaultDashboardsInterval: 1m
```

### Grafana Persistence

```yaml
grafana:
  persistence:
    enabled: true
    type: pvc
    storageClassName: "ha-gen-lh"
    accessModes:
      - ReadWriteOnce
    size: 10Gi
    finalizers:
      - kubernetes.io/pvc-protection
```

### Grafana Ingress

```yaml
grafana:
  ingress:
    enabled: false
    ingressClassName: ""
    annotations: {}
      # kubernetes.io/ingress.class: nginx
      # cert-manager.io/cluster-issuer: letsencrypt-prod
    hosts: []
      # - grafana.example.com
    tls: []
      # - secretName: grafana-tls
      #   hosts:
      #     - grafana.example.com
```

### Grafana Datasources

```yaml
grafana:
  sidecar:
    dashboards:
      enabled: true
      label: grafana_dashboard
      labelValue: "1"
      searchNamespace: ALL
    datasources:
      enabled: true
      defaultDatasourceEnabled: true
      label: grafana_datasource
      labelValue: "1"
```

## Alertmanager Configuration

### Basic Alertmanager Settings

```yaml
alertmanager:
  enabled: true
  
  alertmanagerSpec:
    # Alertmanager image
    image:
      registry: quay.io
      repository: prometheus/alertmanager
      tag: v0.28.1
    
    # Replica configuration
    replicas: 1
    
    # Data retention
    retention: 120h
    
    # Log configuration
    logLevel: info
    logFormat: logfmt
```

### Alertmanager Configuration

```yaml
alertmanager:
  config:
    global:
      resolve_timeout: 5m
      # SMTP configuration
      # smtp_smarthost: 'localhost:587'
      # smtp_from: 'alerts@example.com'
    
    # Inhibition rules
    inhibit_rules:
      - source_matchers:
          - 'severity = critical'
        target_matchers:
          - 'severity =~ warning|info'
        equal:
          - 'namespace'
          - 'alertname'
    
    # Routing configuration
    route:
      group_by: ['namespace']
      group_wait: 30s
      group_interval: 5m
      repeat_interval: 12h
      receiver: 'null'
      routes:
        - receiver: 'null'
          matchers:
            - alertname = "Watchdog"
    
    # Receivers
    receivers:
      - name: 'null'
      # - name: 'email'
      #   email_configs:
      #     - to: 'admin@example.com'
      #       subject: '[ALERT] {{ .GroupLabels.alertname }}'
      #       body: |
      #         {{ range .Alerts }}
      #         Alert: {{ .Annotations.summary }}
      #         Description: {{ .Annotations.description }}
      #         {{ end }}
```

### Alertmanager Storage

```yaml
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

## Default Rules Configuration

### Enabling/Disabling Rule Groups

```yaml
defaultRules:
  create: true
  rules:
    # Core Kubernetes monitoring
    alertmanager: true
    etcd: true
    general: true
    k8sContainerCpuUsageSecondsTotal: true
    k8sContainerMemoryCache: true
    k8sContainerMemoryRss: true
    k8sContainerResource: true
    kubernetesApps: true
    kubernetesResources: true
    kubernetesStorage: true
    kubernetesSystem: true
    
    # Prometheus monitoring
    prometheus: true
    prometheusOperator: true
    
    # Node monitoring
    node: true
    nodeExporterAlerting: true
    nodeExporterRecording: true
    
    # Additional components
    kubeStateMetrics: true
    network: true
    windows: false  # Enable if monitoring Windows nodes
```

### Custom Rules

```yaml
# Additional custom rules
additionalPrometheusRulesMap:
  custom-rules:
    groups:
      - name: custom.rules
        rules:
          - alert: HighPodCPU
            expr: rate(container_cpu_usage_seconds_total[5m]) > 0.8
            for: 5m
            labels:
              severity: warning
            annotations:
              summary: "High CPU usage in pod {{ $labels.pod }}"
              description: "Pod {{ $labels.pod }} has high CPU usage"
```

## Security Configuration

### Security Contexts

```yaml
# Prometheus security context
prometheus:
  prometheusSpec:
    securityContext:
      runAsGroup: 2000
      runAsNonRoot: true
      runAsUser: 1000
      fsGroup: 2000
      seccompProfile:
        type: RuntimeDefault

# Grafana security context  
grafana:
  securityContext:
    runAsGroup: 472
    runAsUser: 472
    runAsNonRoot: true
    fsGroup: 472

# Alertmanager security context
alertmanager:
  alertmanagerSpec:
    securityContext:
      runAsGroup: 2000
      runAsNonRoot: true
      runAsUser: 1000
      fsGroup: 2000
      seccompProfile:
        type: RuntimeDefault
```

### RBAC Configuration

```yaml
global:
  rbac:
    create: true
    createAggregateClusterRoles: false

# Service accounts
prometheus:
  serviceAccount:
    create: true
    name: ""
    annotations: {}

grafana:
  serviceAccount:
    create: true
    name: ""
    annotations: {}

alertmanager:
  serviceAccount:
    create: true
    name: ""
    annotations: {}
```

## Network Configuration

### Service Configuration

```yaml
# Prometheus service
prometheus:
  service:
    enabled: true
    type: ClusterIP
    port: 9090
    targetPort: 9090
    annotations: {}

# Grafana service
grafana:
  service:
    enabled: true
    type: ClusterIP
    port: 80
    targetPort: 3000
    annotations: {}

# Alertmanager service
alertmanager:
  service:
    enabled: true
    type: ClusterIP
    port: 9093
    targetPort: 9093
    annotations: {}
```

### Ingress Configuration

```yaml
# Prometheus ingress
prometheus:
  ingress:
    enabled: false
    ingressClassName: ""
    annotations: {}
    hosts: []
    tls: []

# Alertmanager ingress
alertmanager:
  ingress:
    enabled: false
    ingressClassName: ""
    annotations: {}
    hosts: []
    tls: []
```

## Resource Configuration

### Resource Limits and Requests

```yaml
# Prometheus resources
prometheus:
  prometheusSpec:
    resources:
      requests:
        memory: 2Gi
        cpu: 1000m
      limits:
        memory: 4Gi
        cpu: 2000m

# Grafana resources
grafana:
  resources:
    requests:
      memory: 256Mi
      cpu: 100m
    limits:
      memory: 512Mi
      cpu: 200m

# Alertmanager resources
alertmanager:
  alertmanagerSpec:
    resources:
      requests:
        memory: 200Mi
        cpu: 100m
      limits:
        memory: 400Mi
        cpu: 200m
```

### Node Selection and Affinity

```yaml
# Prometheus node selection
prometheus:
  prometheusSpec:
    nodeSelector: {}
      # kubernetes.io/os: linux
    
    affinity: {}
      # nodeAffinity:
      #   requiredDuringSchedulingIgnoredDuringExecution:
      #     nodeSelectorTerms:
      #     - matchExpressions:
      #       - key: node-type
      #         operator: In
      #         values:
      #         - monitoring
    
    tolerations: []
      # - key: "monitoring"
      #   operator: "Equal"
      #   value: "true"
      #   effect: "NoSchedule"
```

## Advanced Configuration

### Additional Scrape Configurations

```yaml
prometheus:
  prometheusSpec:
    additionalScrapeConfigs: []
    # - job_name: 'custom-app'
    #   kubernetes_sd_configs:
    #     - role: endpoints
    #   relabel_configs:
    #     - source_labels: [__meta_kubernetes_service_name]
    #       action: keep
    #       regex: my-custom-service
```

### Custom Alerts

```yaml
# Disable specific default alerts
defaultRules:
  disabled:
    KubeAPIDown: true
    NodeRAIDDegraded: true
```

### Thanos Integration

```yaml
prometheus:
  prometheusSpec:
    thanos:
      objectStorageConfig:
        secret:
          type: S3
          config:
            bucket: "thanos-storage"
            endpoint: "s3.amazonaws.com"
            region: "us-east-1"
```

## Environment-Specific Examples

### Development Environment

```yaml
# Minimal resources for development
prometheus:
  prometheusSpec:
    retention: 7d
    resources:
      requests:
        memory: 512Mi
        cpu: 250m

grafana:
  persistence:
    size: 5Gi
  
alertmanager:
  alertmanagerSpec:
    resources:
      requests:
        memory: 128Mi
        cpu: 50m
```

### Production Environment

```yaml
# Production-ready configuration
prometheus:
  prometheusSpec:
    retention: 30d
    replicas: 2
    resources:
      requests:
        memory: 4Gi
        cpu: 2000m
      limits:
        memory: 8Gi
        cpu: 4000m
    storageSpec:
      volumeClaimTemplate:
        spec:
          resources:
            requests:
              storage: 200Gi

alertmanager:
  alertmanagerSpec:
    replicas: 3
    resources:
      requests:
        memory: 512Mi
        cpu: 200m
```

## Validation

### Configuration Validation

Before applying configuration changes:

```bash
# Validate Helm template
helm template itl-prometheus ./Chart -f values.yaml --debug

# Dry-run installation
helm install itl-prometheus ./Chart --dry-run -f values.yaml
```

### Testing Configuration

```bash
# Test Prometheus configuration
kubectl exec -n monitoring prometheus-itl-prometheus-kube-prometheus-prometheus-0 -- \
  promtool check config /etc/prometheus/config_out/prometheus.env.yaml

# Test Alertmanager configuration
kubectl exec -n monitoring alertmanager-itl-prometheus-kube-prometheus-alertmanager-0 -- \
  amtool check-config /etc/alertmanager/config/alertmanager.yml
```

## Next Steps

- [Storage Configuration](storage.md) - Detailed storage setup
- [Components Documentation](components/) - Component-specific configuration
- [Examples](examples/) - Common configuration patterns
- [Troubleshooting](troubleshooting.md) - Configuration troubleshooting