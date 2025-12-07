# Keycloak Helm Chart

This Helm chart deploys [Keycloak](https://www.keycloak.org/) on Kubernetes using a StatefulSet for identity and access management.

## Prerequisites

- Kubernetes 1.19+
- Helm 3.0+
- External PostgreSQL database

## Installation

### Add the Helm repository

```bash

helm repo add lracicot-charts https://lracicot.github.io/helm-charts
helm repo update
```

### Install from GitHub Pages

```bash
helm install keycloak lracicot-charts/keycloak
```

### Install from GHCR (OCI)

```bash
helm install keycloak oci://ghcr.io/lracicot/charts/keycloak --version 0.1.0
```

### Install with custom values

```bash
helm install keycloak lracicot-charts/keycloak \
  --set keycloak.admin.password=mysecurepassword \
  --set externalDatabase.host=my-postgres.example.com \
  --set externalDatabase.password=dbpassword
```

## Configuration

The following table lists the configurable parameters of the Keycloak chart and their default values.

### Keycloak Configuration

| Parameter                   | Description                 | Default                     |
| --------------------------- | --------------------------- | --------------------------- |
| `keycloak.replicas`         | Number of Keycloak replicas | `2`                         |
| `keycloak.image.repository` | Keycloak image repository   | `quay.io/keycloak/keycloak` |
| `keycloak.image.tag`        | Keycloak image tag          | `26.4.7`                    |
| `keycloak.image.pullPolicy` | Image pull policy           | `IfNotPresent`              |
| `keycloak.admin.username`   | Admin username              | `admin`                     |
| `keycloak.admin.password`   | Admin password              | `admin`                     |

### Keycloak Environment Configuration

| Parameter                            | Description                         | Default      |
| ------------------------------------ | ----------------------------------- | ------------ |
| `keycloak.config.KC_PROXY_HEADERS`   | Proxy headers mode                  | `xforwarded` |
| `keycloak.config.KC_HTTP_ENABLED`    | Enable HTTP (disable in production) | `true`       |
| `keycloak.config.KC_HOSTNAME_STRICT` | Strict hostname validation          | `false`      |
| `keycloak.config.KC_HEALTH_ENABLED`  | Enable health endpoints             | `true`       |
| `keycloak.config.KC_CACHE`           | Cache implementation                | `ispn`       |

### Resource Configuration

| Parameter                            | Description    | Default  |
| ------------------------------------ | -------------- | -------- |
| `keycloak.resources.limits.cpu`      | CPU limit      | `2000m`  |
| `keycloak.resources.limits.memory`   | Memory limit   | `2000Mi` |
| `keycloak.resources.requests.cpu`    | CPU request    | `500m`   |
| `keycloak.resources.requests.memory` | Memory request | `1700Mi` |

### Health Probes

| Parameter                                  | Description                       | Default |
| ------------------------------------------ | --------------------------------- | ------- |
| `keycloak.startupProbe.periodSeconds`      | Startup probe period              | `1`     |
| `keycloak.startupProbe.failureThreshold`   | Startup probe failure threshold   | `600`   |
| `keycloak.readinessProbe.periodSeconds`    | Readiness probe period            | `10`    |
| `keycloak.readinessProbe.failureThreshold` | Readiness probe failure threshold | `3`     |
| `keycloak.livenessProbe.periodSeconds`     | Liveness probe period             | `10`    |
| `keycloak.livenessProbe.failureThreshold`  | Liveness probe failure threshold  | `3`     |

### Service Configuration

| Parameter             | Description         | Default     |
| --------------------- | ------------------- | ----------- |
| `service.type`        | Service type        | `ClusterIP` |
| `service.port`        | Service port        | `8080`      |
| `service.targetPort`  | Target port         | `8080`      |
| `service.name`        | Service name        | `keycloak`  |
| `service.annotations` | Service annotations | `{}`        |

### Discovery Service

| Parameter                  | Description                       | Default              |
| -------------------------- | --------------------------------- | -------------------- |
| `discoveryService.enabled` | Enable headless discovery service | `true`               |
| `discoveryService.name`    | Discovery service name            | `keycloak-discovery` |
| `discoveryService.type`    | Discovery service type            | `ClusterIP`          |

### External Database Configuration

| Parameter                   | Description           | Default                                |
| --------------------------- | --------------------- | -------------------------------------- |
| `externalDatabase.enabled`  | Use external database | `true`                                 |
| `externalDatabase.type`     | Database type         | `postgres`                             |
| `externalDatabase.host`     | Database host         | `postgresql.default.svc.cluster.local` |
| `externalDatabase.port`     | Database port         | `5432`                                 |
| `externalDatabase.name`     | Database name         | `keycloak`                             |
| `externalDatabase.username` | Database username     | `keycloak`                             |
| `externalDatabase.password` | Database password     | `keycloak`                             |

### Additional Configuration

| Parameter         | Description          | Default |
| ----------------- | -------------------- | ------- |
| `securityContext` | Pod security context | `{}`    |
| `nodeSelector`    | Node selector        | `{}`    |
| `tolerations`     | Tolerations          | `[]`    |
| `affinity`        | Affinity rules       | `{}`    |

## Examples

### Production Setup with External Database

```yaml
# values-production.yaml
keycloak:
  replicas: 3
  admin:
    password: "changeme123!"
  config:
    KC_HTTP_ENABLED: "false"
    KC_HOSTNAME_STRICT: "true"

externalDatabase:
  host: "postgres.production.svc.cluster.local"
  name: "keycloak_prod"
  username: "keycloak_user"
  password: "secure_db_password"

service:
  type: LoadBalancer
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
```

Install with:

```bash
helm install keycloak lracicot-charts/keycloak -f values-production.yaml
```

### Development Setup

```yaml
# values-dev.yaml
keycloak:
  replicas: 1
  resources:
    limits:
      cpu: 1000m
      memory: 1000Mi
    requests:
      cpu: 250m
      memory: 512Mi

externalDatabase:
  host: "postgres-dev"
  name: "keycloak_dev"
```

## Architecture

This chart deploys Keycloak as a StatefulSet with the following components:

- **StatefulSet**: Manages Keycloak pods with stable network identities
- **Service**: ClusterIP service for Keycloak HTTP traffic
- **Discovery Service**: Headless service for pod-to-pod discovery (JGroups clustering)

### Clustering

Keycloak instances use JGroups for clustering and require the headless discovery service to find each other. The chart configures:

- JGroups port: 7800
- JGroups FD port: 57800
- Pod IP binding for IPv6 support

## Database Requirements

This chart requires an **external PostgreSQL database**. The database must be provisioned separately before installing this chart.

### Database Setup Example

```sql
CREATE DATABASE keycloak;
CREATE USER keycloak WITH PASSWORD 'your-password';
GRANT ALL PRIVILEGES ON DATABASE keycloak TO keycloak;
```

## Security Considerations

⚠️ **Important for Production:**

1. **Change default passwords**: Set secure values for `keycloak.admin.password` and `externalDatabase.password`
2. **Enable HTTPS**: Set `keycloak.config.KC_HTTP_ENABLED` to `"false"` and configure TLS
3. **Enable hostname validation**: Set `keycloak.config.KC_HOSTNAME_STRICT` to `"true"`
4. **Use Secrets**: Consider using Kubernetes Secrets instead of plain values
5. **Database security**: Ensure database credentials are properly secured

## Upgrading

To upgrade the chart:

```bash
helm upgrade keycloak lracicot-charts/keycloak
```

## Uninstalling

To uninstall/delete the `keycloak` deployment:

```bash
helm uninstall keycloak
```

## Support

For issues and questions:

- [GitHub Issues](https://github.com/lracicot/helm-charts/issues)
- [Keycloak Documentation](https://www.keycloak.org/documentation)

## License

This chart is provided as-is under the same license as Keycloak.
