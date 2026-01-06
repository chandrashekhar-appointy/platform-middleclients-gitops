# Changelog

All notable changes to the Omnix Helm chart will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Planned
- Support for Ingress resources
- NetworkPolicy templates
- Support for initContainers and sidecars
- Job and CronJob templates

## [0.1.0] - 2024-12-02

### Added
- Initial release of Omnix Helm chart
- Deployment resource with flexible configuration
- Service resource (ClusterIP, NodePort, LoadBalancer support)
- ServiceAccount with optional creation
- HorizontalPodAutoscaler (HPA) support
- PodDisruptionBudget (PDB) support
- ConfigMap and Secret management from files
- Volume mounting support
- Environment variable management (direct and from ConfigMap/Secret)
- Liveness and readiness probe configurations
- Resource requests and limits
- Security contexts (pod and container level)
- Custom labels and annotations support
- RollingUpdate and Recreate deployment strategies
- Node selector support
- Image pull secrets support
- Comprehensive README documentation
- Example values.yaml with all features demonstrated

### Features
- Universal chart for any Kubernetes application
- Production-ready configurations
- Follows Kubernetes best practices
- Flexible and customizable for various use cases

---

## Version History

- **v0.1.0** - Initial public release

[Unreleased]: https://github.com/YOUR_USERNAME/omnix-chart/compare/v0.1.0...HEAD
[0.1.0]: https://github.com/YOUR_USERNAME/omnix-chart/releases/tag/v0.1.0
