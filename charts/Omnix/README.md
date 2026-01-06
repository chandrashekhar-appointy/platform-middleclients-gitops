# Omnix Helm Chart

A production-ready, flexible Helm chart for deploying any Kubernetes application with comprehensive configuration options.

[![Chart Version](https://img.shields.io/badge/chart-0.1.0-blue.svg)](https://github.com/YOUR_USERNAME/omnix-chart)
[![Kubernetes](https://img.shields.io/badge/kubernetes-%3E%3D1.19-brightgreen.svg)](https://kubernetes.io/)
[![License](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)

## Overview

Omnix is a universal Helm chart designed to simplify Kubernetes deployments for any application. It provides a comprehensive set of features while maintaining flexibility and following Kubernetes best practices.

### Features

- **Flexible Deployment Configurations**: Support for RollingUpdate and Recreate strategies
- **Auto-scaling**: Built-in HorizontalPodAutoscaler (HPA) support
- **High Availability**: Pod Disruption Budget (PDB) for maintaining application availability
- **Configuration Management**: ConfigMaps and Secrets management from files
- **Volume Mounting**: Easy file mounting from chart's files directory
- **Security**: Pod and container security contexts, service account support
- **Observability**: Liveness and readiness probe configurations
- **Resource Management**: CPU and memory requests/limits
- **Custom Labels & Annotations**: Flexible labeling for all resources

## Prerequisites

- Kubernetes 1.19+
- Helm 3.0+

## Installation

### Add the Helm Repository (if published)

```bash
helm repo add omnix https://YOUR_USERNAME.github.io/omnix-chart
helm repo update
```

### Install from Local Chart

```bash
# Clone the repository
git clone https://github.com/YOUR_USERNAME/omnix-chart.git
cd omnix-chart/helm/omnix

# Install the chart
helm install my-app . -f your-values.yaml
```

### Install from GitHub Release

```bash
# Install specific version
helm install my-app https://github.com/YOUR_USERNAME/omnix-chart/releases/download/v0.1.0/omnix-0.1.0.tgz -f your-values.yaml
```

## Usage

### Basic Installation

```bash
helm install my-application omnix/omnix \
  --set image.name=nginx \
  --set image.tag=1.21 \
  --set replicaCount=3
```

### Installation with Custom Values

Create your own `my-values.yaml` file based on the example `values.yaml`:

```bash
helm install my-application omnix/omnix -f my-values.yaml
```

### Upgrading Your Application

```bash
# Upgrade with new values
helm upgrade my-application omnix/omnix -f my-values.yaml

# Upgrade with new image tag
helm upgrade my-application omnix/omnix \
  --set image.tag=1.22 \
  --reuse-values
```

### Rollback to Previous Version

```bash
# List release history
helm history my-application

# Rollback to specific revision
helm rollback my-application 1
```

### Uninstall

```bash
helm uninstall my-application
```

## Configuration

The `values.yaml` file provided in this chart is an **example configuration** that demonstrates all available options. You should create your own values file tailored to your application's needs.

### Key Configuration Sections

#### Image Configuration

```yaml
image:
  name: your-app
  tag: "1.0.0"
  pullPolicy: IfNotPresent
```

#### Deployment Strategy

```yaml
deployment:
  serviceAccountName: ""  # Optional: custom service account
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
```

#### Environment Variables

```yaml
# Direct environment variables
env:
  LOG_LEVEL: "info"
  ENVIRONMENT: "production"

# From ConfigMaps/Secrets
envFrom:
  - name: app-config
    type: configMap
    files:
      - path: files/configmap/app-config.yaml
```

#### Resource Management

```yaml
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 512Mi
```

#### Auto-scaling (HPA)

```yaml
hpa:
  enable: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70
  targetMemoryUtilizationPercentage: 80
```

#### Health Checks

```yaml
livenessProbe:
  enable: true
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 30
  periodSeconds: 10

readinessProbe:
  enable: true
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
```

#### Volume Mounts

```yaml
volumes:
  - name: app-config
    type: configMap
    mountPath: /etc/app/config.yaml
    subPath: config.yaml
    files:
      - key: config.yaml
        path: files/configmap/config.yaml
```

### Complete Configuration Reference

| Parameter | Description | Default |
|-----------|-------------|---------|
| `nameOverride` | Override chart name | `""` |
| `fullnameOverride` | Override full resource names | `""` |
| `namespaceOverride` | Override namespace | `""` |
| `replicaCount` | Number of replicas | `2` |
| `image.name` | Container image name | `nginx` |
| `image.tag` | Container image tag | `1.21` |
| `image.pullPolicy` | Image pull policy | `IfNotPresent` |
| `imagePullSecrets` | Image pull secrets | `[]` |
| `serviceAccount.enable` | Create service account | `true` |
| `deployment.serviceAccountName` | Custom service account name | `""` |
| `service.type` | Kubernetes service type | `ClusterIP` |
| `hpa.enable` | Enable HPA | `true` |
| `hpa.minReplicas` | Minimum replicas | `2` |
| `hpa.maxReplicas` | Maximum replicas | `10` |
| `pdb.enable` | Enable Pod Disruption Budget | `true` |
| `pdb.minAvailable` | Min available pods | `1` |

For a complete list of all configurable parameters, see the [values.yaml](values.yaml) file.

## File Structure

```
omnix/
├── Chart.yaml              # Chart metadata
├── values.yaml            # Example configuration (customize for your app)
├── README.md              # This file
├── templates/             # Kubernetes manifests
│   ├── _helpers.tpl       # Template helpers
│   ├── deployment.yaml    # Deployment resource
│   ├── service.yaml       # Service resource
│   ├── serviceAccount.yaml
│   ├── hpa.yaml          # HorizontalPodAutoscaler
│   ├── pdb.yaml          # PodDisruptionBudget
│   ├── configmap.yaml
│   ├── configmap-env.yaml
│   ├── configmap-envfrom.yaml
│   └── secret-envfrom.yaml
└── files/                # Application config files
    ├── configmap/
    └── secrets/
```

## Examples

### Deploy a Node.js Application

```yaml
# my-nodejs-app-values.yaml
fullnameOverride: nodejs-app

image:
  name: node
  tag: "18-alpine"

replicaCount: 3

ports:
  http: 3000

env:
  NODE_ENV: "production"
  PORT: "3000"

resources:
  requests:
    cpu: 200m
    memory: 256Mi
  limits:
    cpu: 1000m
    memory: 512Mi

livenessProbe:
  enable: true
  httpGet:
    path: /health
    port: 3000

readinessProbe:
  enable: true
  httpGet:
    path: /ready
    port: 3000
```

```bash
helm install nodejs-app omnix/omnix -f my-nodejs-app-values.yaml
```

### Deploy a Python Flask Application

```yaml
# my-flask-app-values.yaml
fullnameOverride: flask-api

image:
  name: your-registry/flask-app
  tag: "v1.0.0"

replicaCount: 2

ports:
  http: 5000

env:
  FLASK_ENV: "production"

hpa:
  enable: true
  minReplicas: 2
  maxReplicas: 5
  targetCPUUtilizationPercentage: 60
```

```bash
helm install flask-api omnix/omnix -f my-flask-app-values.yaml
```

## Versioning

This chart follows [Semantic Versioning](https://semver.org/):

- **MAJOR** version: Incompatible API changes
- **MINOR** version: Backwards-compatible functionality additions
- **PATCH** version: Backwards-compatible bug fixes

Chart versions are tagged as `vX.Y.Z` (e.g., `v0.1.0`, `v1.2.3`).

## Contributing

Contributions are welcome! Please:

1. Fork the repository
2. Create a feature branch (`git checkout -b feature/amazing-feature`)
3. Commit your changes (`git commit -m 'Add amazing feature'`)
4. Push to the branch (`git push origin feature/amazing-feature`)
5. Open a Pull Request

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## Support

- **Issues**: [GitHub Issues](https://github.com/YOUR_USERNAME/omnix-chart/issues)
- **Discussions**: [GitHub Discussions](https://github.com/YOUR_USERNAME/omnix-chart/discussions)

## Changelog

See [CHANGELOG.md](CHANGELOG.md) for release history and updates.

## Acknowledgments

Built with best practices from the Kubernetes and Helm communities.

---

**Note**: The `values.yaml` file included with this chart is an example configuration showcasing all available features. Create your own values file based on your application's specific requirements.
