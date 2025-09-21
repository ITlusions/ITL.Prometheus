# ITL.Prometheus

A comprehensive Kubernetes monitoring solution based on the Prometheus stack, providing monitoring, alerting, and visualization capabilities for Kubernetes clusters.

## Overview

ITL.Prometheus is a Helm chart that deploys a complete monitoring stack including:

- **Prometheus**: Time-series database for metrics collection and storage
- **Grafana**: Visualization and dashboarding platform with pre-built dashboards
- **Alertmanager**: Alert management and routing system
- **Prometheus Operator**: Kubernetes operator for managing Prometheus resources
- **Node Exporter**: System metrics collector for cluster nodes
- **Kube State Metrics**: Kubernetes object state metrics

## Features

- ✅ **Complete Monitoring Stack**: All-in-one solution for Kubernetes monitoring
- ✅ **High Availability**: Configurable for production deployments with HA setup
- ✅ **Custom Storage**: Configured with `ha-gen-lh` storage class for persistent data
- ✅ **Pre-built Dashboards**: Comprehensive Grafana dashboards for common monitoring scenarios
- ✅ **Flexible Alerting**: Customizable alerting rules and notification routing
- ✅ **Security Hardened**: Non-root containers and proper security contexts
- ✅ **Multi-namespace Support**: Monitor services across multiple namespaces

## Quick Start

### Prerequisites

- Kubernetes cluster (version 1.19+)
- Helm 3.x installed
- Storage class `ha-gen-lh` configured in your cluster
- Sufficient cluster resources (4+ CPU, 8Gi+ RAM, 70Gi+ storage)

### Installation

1. Clone the repository:
   ```bash
   git clone <repository-url>
   cd ITL.Prometheus
   ```

2. Install the chart:
   ```bash
   helm install itl-prometheus ./Chart -n monitoring --create-namespace
   ```

3. Access Grafana:
   ```bash
   # Get admin password
   kubectl get secret itl-prometheus-grafana -n monitoring -o jsonpath="{.data.admin-password}" | base64 --decode
   
   # Port forward to Grafana
   kubectl port-forward -n monitoring svc/itl-prometheus-grafana 3000:80
   ```
   
   Visit: http://localhost:3000 (admin/prom-operator)

## Documentation

Comprehensive documentation is available in the `/docs` directory:

- 📖 **[Overview](docs/overview.md)** - Architecture and component overview
- 🚀 **[Installation Guide](docs/installation.md)** - Detailed installation instructions
- ⚙️ **[Configuration Reference](docs/configuration.md)** - Complete configuration options
- 💾 **[Storage Configuration](docs/storage.md)** - Storage setup and requirements
- 🔧 **[Components](docs/components/)** - Individual component documentation
- 📝 **[Examples](docs/examples/)** - Configuration examples for different environments
- 🔍 **[Troubleshooting](docs/troubleshooting.md)** - Common issues and solutions

## Configuration

### Default Configuration

The chart comes with production-ready defaults:

- **Prometheus**: 50Gi storage, 10d retention
- **Grafana**: 10Gi storage, admin/prom-operator credentials
- **Alertmanager**: 10Gi storage, basic alert routing
- **Storage Class**: `ha-gen-lh` for all persistent volumes

### Custom Configuration

Create a custom values file to override defaults:

```yaml
# custom-values.yaml
grafana:
  adminPassword: "your-secure-password"
  ingress:
    enabled: true
    hosts: 
      - grafana.yourdomain.com

prometheus:
  prometheusSpec:
    retention: 30d
    resources:
      requests:
        memory: 4Gi
        cpu: 2000m

alertmanager:
  config:
    global:
      smtp_smarthost: 'smtp.yourdomain.com:587'
    receivers:
      - name: 'email'
        email_configs:
          - to: 'admin@yourdomain.com'
```

Install with custom configuration:
```bash
helm install itl-prometheus ./Chart -n monitoring --create-namespace -f custom-values.yaml
```

## Architecture

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Prometheus    │◄───┤ Prometheus      │◄───┤  Node Exporter  │
│   (Monitoring)  │    │   Operator      │    │ (System Metrics)│
└─────────────────┘    └─────────────────┘    └─────────────────┘
         │                       │                       │
         │                       ▼                       │
         ▼              ┌─────────────────┐              │
┌─────────────────┐     │ Kube State      │              │
│  Alertmanager   │     │   Metrics       │              │
│  (Alerting)     │     │(K8s Metrics)    │              │
└─────────────────┘     └─────────────────┘              │
         │                                                │
         │                                                │
         ▼                                                ▼
┌─────────────────┐                            ┌─────────────────┐
│    Grafana      │                            │ ServiceMonitors │
│(Visualization)  │                            │   PodMonitors   │
└─────────────────┘                            │ PrometheusRules │
                                               └─────────────────┘
```

## Chart Information

- **Chart Name**: `itl-kube-prometheus-stack`
- **Chart Version**: `1.0.0`
- **App Version**: `66.2.2`
- **Base Chart**: `kube-prometheus-stack` version `77.9.1`
- **Dependencies**: Prometheus Community Helm Charts

## Default Access

After installation, you can access the components:

| Component | URL | Default Credentials |
|-----------|-----|-------------------|
| Grafana | http://localhost:3000 | admin / prom-operator |
| Prometheus | http://localhost:9090 | None |
| Alertmanager | http://localhost:9093 | None |

Use `kubectl port-forward` to access the services locally.

## Upgrading

To upgrade the installation:

```bash
# Update dependencies
helm dependency update ./Chart

# Upgrade release
helm upgrade itl-prometheus ./Chart -n monitoring
```

## Uninstalling

To remove the installation:

```bash
# Uninstall the release
helm uninstall itl-prometheus -n monitoring

# Optional: Remove the namespace
kubectl delete namespace monitoring
```

## Support

For support and issues:

1. Check the [Troubleshooting Guide](docs/troubleshooting.md)
2. Review the [Configuration Reference](docs/configuration.md)
3. Consult the component-specific documentation in [docs/components/](docs/components/)

## Contributing

Contributions are welcome! Please:

1. Read the documentation thoroughly
2. Test your changes in a real environment
3. Update documentation as needed
4. Submit a pull request with clear description

## License

This project is licensed under the terms specified in the LICENSE file.
