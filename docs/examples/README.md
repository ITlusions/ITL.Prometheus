# Configuration Examples

This directory contains example configurations for common use cases of the ITL Prometheus Stack.

## Available Examples

- [Development Environment](development.yaml) - Minimal configuration for development
- [Production Environment](production.yaml) - Production-ready configuration with HA
- [Custom Storage](custom-storage.yaml) - Different storage class configurations
- [External Access](external-access.yaml) - Ingress and external access configuration
- [Alert Configuration](alerting.yaml) - Custom alerting rules and notifications
- [High Availability](high-availability.yaml) - Multi-replica setup with affinity rules
- [Resource Constrained](resource-constrained.yaml) - Configuration for small clusters

## Usage

To use any of these examples:

```bash
# Download the example
wget https://raw.githubusercontent.com/ITlusions/ITL.Prometheus/main/docs/examples/production.yaml

# Install with the example configuration
helm install itl-prometheus ./Chart -n monitoring --create-namespace -f production.yaml

# Or upgrade existing installation
helm upgrade itl-prometheus ./Chart -n monitoring -f production.yaml
```

## Customization

These examples serve as starting points. Modify them according to your specific requirements:

1. **Storage**: Adjust storage classes and sizes
2. **Resources**: Modify CPU and memory limits based on your cluster
3. **Networking**: Configure ingress, service types, and network policies
4. **Security**: Adjust security contexts and RBAC settings
5. **Monitoring**: Add custom metrics and alerting rules

## Environment-Specific Considerations

### Development
- Reduced resource requirements
- Shorter retention periods
- Simplified configuration
- No high availability

### Staging
- Similar to production but with reduced resources
- Testing of production configuration
- Integration testing setup

### Production
- High availability configuration
- Resource limits and requests properly set
- Security hardening
- Backup and monitoring strategies
- Network policies and ingress configuration

## Validation

Before applying any configuration:

```bash
# Validate the configuration
helm template itl-prometheus ./Chart -f your-config.yaml --debug

# Perform a dry-run
helm install itl-prometheus ./Chart -f your-config.yaml --dry-run
```

## Contributing

To contribute additional examples:

1. Create a new YAML file with descriptive naming
2. Add comprehensive comments explaining the configuration
3. Include a brief description in this README
4. Test the configuration in a real environment
5. Submit a pull request

## Support

For questions about these examples or help with customization:

- Review the [Configuration Reference](../configuration.md)
- Check the [Troubleshooting Guide](../troubleshooting.md)
- Consult the [Installation Guide](../installation.md)