# Components Documentation

This directory contains detailed documentation for each component of the ITL Prometheus Stack.

## Components

- [Prometheus](prometheus.md) - Time-series database and monitoring server
- [Grafana](grafana.md) - Visualization and dashboarding platform  
- [Alertmanager](alertmanager.md) - Alert management and routing
- [Prometheus Operator](prometheus-operator.md) - Kubernetes operator for Prometheus
- [Node Exporter](node-exporter.md) - System metrics collector
- [Kube State Metrics](kube-state-metrics.md) - Kubernetes object metrics

## Architecture Overview

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
│    Grafana      │                            │   ServiceMonitor│
│(Visualization)  │                            │   PodMonitor    │
└─────────────────┘                            │   PrometheusRule│
                                               └─────────────────┘
```

## Component Interactions

1. **Prometheus Operator** manages the lifecycle of Prometheus, Alertmanager, and related resources
2. **Prometheus** scrapes metrics from various exporters and services
3. **Node Exporter** provides system-level metrics from cluster nodes
4. **Kube State Metrics** exposes Kubernetes object state metrics
5. **Alertmanager** receives alerts from Prometheus and routes them to receivers
6. **Grafana** visualizes metrics stored in Prometheus through dashboards

## Quick Navigation

- For installation details: [Installation Guide](../installation.md)
- For configuration options: [Configuration Reference](../configuration.md)
- For storage setup: [Storage Configuration](../storage.md)