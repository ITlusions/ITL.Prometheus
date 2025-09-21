# ITL Prometheus Stack Overview

## Introduction

The ITL Prometheus Stack is a comprehensive monitoring solution based on the popular `kube-prometheus-stack` Helm chart. This chart provides a complete monitoring stack for Kubernetes clusters, including Prometheus, Grafana, and Alertmanager.

## Chart Information

- **Chart Name**: `itl-kube-prometheus-stack`
- **Chart Version**: `1.0.0`
- **App Version**: `66.2.2`
- **Base Chart**: `kube-prometheus-stack` version `77.9.1`

## Architecture

The ITL Prometheus Stack consists of several key components:

### Core Components

1. **Prometheus Operator**: Manages Prometheus instances and related resources
2. **Prometheus**: Time-series database for metrics collection and storage
3. **Grafana**: Visualization and dashboarding platform
4. **Alertmanager**: Alert management and routing

### Additional Components

- **Node Exporter**: Collects system-level metrics from cluster nodes
- **Kube State Metrics**: Exposes cluster-level metrics about Kubernetes objects
- **Custom Resource Definitions (CRDs)**: For managing Prometheus resources

## Key Features

- **Complete Monitoring Stack**: All-in-one solution for Kubernetes monitoring
- **High Availability**: Configurable for HA deployments
- **Custom Storage**: Configured with `ha-gen-lh` storage class for persistent data
- **Flexible Configuration**: Extensive configuration options through values.yaml
- **Dashboard Integration**: Pre-built Grafana dashboards for common monitoring scenarios
- **Alert Management**: Comprehensive alerting rules and notification routing

## Default Configuration Highlights

### Storage Configuration
- **Prometheus Storage**: 50Gi with `ha-gen-lh` storage class
- **Alertmanager Storage**: 10Gi with `ha-gen-lh` storage class  
- **Grafana Storage**: 10Gi with `ha-gen-lh` storage class

### Security
- **Non-root Containers**: All components run as non-root users
- **Security Contexts**: Proper security contexts configured
- **RBAC**: Role-based access control enabled

### Monitoring Scope
- **Default Rules**: Comprehensive set of alerting and recording rules
- **Service Discovery**: Automatic discovery of services and pods
- **Multi-namespace Support**: Can monitor across multiple namespaces

## Dependencies

The chart depends on the upstream `kube-prometheus-stack` chart from the Prometheus Community:
- Repository: `https://prometheus-community.github.io/helm-charts`
- Version: `77.9.1`

## Next Steps

- [Installation Guide](installation.md) - Learn how to install the chart
- [Configuration Reference](configuration.md) - Detailed configuration options
- [Storage Configuration](storage.md) - Storage setup and requirements