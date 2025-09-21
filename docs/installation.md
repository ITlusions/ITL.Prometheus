# Installation Guide

This guide covers the installation of the ITL Prometheus Stack using Helm.

## Prerequisites

### System Requirements
- Kubernetes cluster (version 1.19+)
- Helm 3.x installed and configured
- Sufficient cluster resources (see Resource Requirements section)
- Storage class `ha-gen-lh` configured in your cluster

### Resource Requirements

**Minimum Requirements:**
- **CPU**: 2 cores total across all components
- **Memory**: 4Gi RAM total across all components  
- **Storage**: 70Gi total (50Gi Prometheus + 10Gi Grafana + 10Gi Alertmanager)

**Recommended for Production:**
- **CPU**: 4+ cores
- **Memory**: 8Gi+ RAM
- **Storage**: 100Gi+ with high-performance storage class

## Storage Class Configuration

This chart is configured to use the `ha-gen-lh` storage class. Ensure this storage class exists in your cluster:

```bash
kubectl get storageclass ha-gen-lh
```

If the storage class doesn't exist, you can either:
1. Create the `ha-gen-lh` storage class in your cluster
2. Modify the values.yaml to use a different storage class

## Installation Steps

### 1. Add Helm Repository

First, add the repository containing the chart dependencies:

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

### 2. Clone the Repository

```bash
git clone <repository-url>
cd ITL.Prometheus
```

### 3. Install with Default Configuration

```bash
helm install itl-prometheus ./Chart -n monitoring --create-namespace
```

### 4. Install with Custom Values

Create a custom values file or modify the existing `Chart/values.yaml`:

```bash
# Create custom values
cat > custom-values.yaml << EOF
# Custom configuration overrides
grafana:
  adminPassword: "your-secure-password"
  
alertmanager:
  config:
    global:
      smtp_smarthost: 'your-smtp-server:587'
      smtp_from: 'alerts@yourdomain.com'
EOF

# Install with custom values
helm install itl-prometheus ./Chart -n monitoring --create-namespace -f custom-values.yaml
```

## Post-Installation Steps

### 1. Verify Installation

Check that all pods are running:

```bash
kubectl get pods -n monitoring
```

Expected pods:
- `alertmanager-*`
- `grafana-*`
- `prometheus-*`
- `prometheus-node-exporter-*`
- `prometheus-kube-state-metrics-*`
- `prometheus-operator-*`

### 2. Access Grafana Dashboard

Get Grafana admin password (if using default configuration):

```bash
kubectl get secret itl-prometheus-grafana -n monitoring -o jsonpath="{.data.admin-password}" | base64 --decode
```

Default credentials:
- **Username**: `admin`
- **Password**: `prom-operator` (or the value retrieved above)

Port-forward to access Grafana:

```bash
kubectl port-forward -n monitoring svc/itl-prometheus-grafana 3000:80
```

Access Grafana at: http://localhost:3000

### 3. Access Prometheus UI

Port-forward to access Prometheus:

```bash
kubectl port-forward -n monitoring svc/itl-prometheus-kube-prometheus-prometheus 9090:9090
```

Access Prometheus at: http://localhost:9090

### 4. Access Alertmanager UI

Port-forward to access Alertmanager:

```bash
kubectl port-forward -n monitoring svc/itl-prometheus-kube-prometheus-alertmanager 9093:9093
```

Access Alertmanager at: http://localhost:9093

## Configuration Verification

### Check Storage

Verify that persistent volumes are created:

```bash
kubectl get pv | grep monitoring
kubectl get pvc -n monitoring
```

### Check CRDs

Verify that Prometheus CRDs are installed:

```bash
kubectl get crd | grep monitoring.coreos.com
```

## Upgrade Process

To upgrade the chart:

```bash
# Update dependencies
helm dependency update ./Chart

# Upgrade installation
helm upgrade itl-prometheus ./Chart -n monitoring
```

## Uninstallation

To remove the installation:

```bash
# Uninstall the release
helm uninstall itl-prometheus -n monitoring

# Optional: Remove the namespace
kubectl delete namespace monitoring

# Optional: Remove CRDs (be careful with this in multi-tenant environments)
kubectl get crd | grep monitoring.coreos.com | awk '{print $1}' | xargs kubectl delete crd
```

## Troubleshooting

If you encounter issues during installation:

1. **Storage Issues**: Verify the `ha-gen-lh` storage class exists and is functioning
2. **Resource Issues**: Check if your cluster has sufficient resources
3. **CRD Issues**: Ensure you have permissions to create cluster-wide resources
4. **Network Issues**: Verify network policies allow communication between components

For detailed troubleshooting, see the [Troubleshooting Guide](troubleshooting.md).

## Next Steps

- [Configuration Reference](configuration.md) - Customize your installation
- [Storage Configuration](storage.md) - Advanced storage configuration
- [Components Documentation](components/) - Learn about individual components