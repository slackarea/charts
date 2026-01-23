# Evolution API Helm Chart

A Helm chart for deploying [Evolution API](https://github.com/EvolutionAPI/evolution-api) - a WhatsApp integration platform on Kubernetes.

## Prerequisites

- Kubernetes 1.23+
- Helm 3.8+
- PV provisioner support in the underlying infrastructure

### Optional: Operators for Production

For production deployments with high availability, install these operators:

```bash
# RabbitMQ Cluster Operator (for quorum queues support)
kubectl apply -f "https://github.com/rabbitmq/cluster-operator/releases/latest/download/cluster-operator.yml"

# CloudNative-PG Operator (for PostgreSQL HA)
kubectl apply --server-side -f https://raw.githubusercontent.com/cloudnative-pg/cloudnative-pg/release-1.25/releases/cnpg-1.25.0.yaml
```

## Installation

### Add the repository

```bash
helm repo add slackarea https://slackarea.github.io/charts
helm repo update
```

### Install the chart

```bash
# Create namespace
kubectl create namespace evolutionapi

# Install with default values
helm install evolution-api slackarea/evolution-api -n evolutionapi

# Install with custom values
helm install evolution-api slackarea/evolution-api -n evolutionapi -f my-values.yaml
```

## Configuration

### Database Options

| Option | Description | Recommended for |
|--------|-------------|-----------------|
| `postgresql.enabled` | Bitnami PostgreSQL subchart | Development |
| `postgresqlStandalone.enabled` | Official PostgreSQL image | Testing |
| `postgresqlOperator.enabled` | CloudNative-PG (requires operator) | **Production** |
| `externalPostgresql.*` | External PostgreSQL | Existing infrastructure |

### Message Queue Options

| Option | Description | Recommended for |
|--------|-------------|-----------------|
| `rabbitmqCluster.enabled` | Bitnami RabbitMQ subchart | Development |
| `rabbitmqStandalone.enabled` | Official RabbitMQ image | Testing |
| `rabbitmqOperator.enabled` | RabbitMQ Cluster Operator (3 nodes, quorum queues) | **Production** |
| `externalRabbitmq.*` | External RabbitMQ | Existing infrastructure |

### Event Streaming Options

| Option | Description | Recommended for |
|--------|-------------|-----------------|
| `kafkaCluster.enabled` | Bitnami Kafka subchart | Development |
| `kafkaStandalone.enabled` | Apache Kafka (KRaft mode) | Testing/Production |
| `externalKafka.*` | External Kafka | Existing infrastructure |

### Ingress Options

| Option | Description |
|--------|-------------|
| `ingress.enabled` | Standard Kubernetes Ingress |
| `ingress.className` | Ingress class (nginx, traefik) |
| `ingressRoute.enabled` | Traefik IngressRoute CRD |

## Example Configurations

### Minimal Development Setup

```yaml
api:
  enabled: true
  server:
    url: "https://evolution-api.example.com"
  authentication:
    apiKey: "your-secure-api-key"

postgresql:
  enabled: true
  auth:
    password: "secure-password"

redis:
  enabled: true
```

### Production Setup with Operators

```yaml
api:
  enabled: true
  replicaCount: 2
  server:
    url: "https://evolution-api.example.com"
  authentication:
    existingSecret: "evolution-api-credentials"
  rabbitmq:
    enabled: true
    globalEnabled: true
    events:
      messagesUpsert: true
      connectionUpdate: true
      # ... other events
  kafka:
    enabled: true
    globalEnabled: true
    events:
      messagesUpsert: true
      # ... other events

# Disable Bitnami subcharts
postgresql:
  enabled: false
rabbitmqCluster:
  enabled: false

# Enable Operators
postgresqlOperator:
  enabled: true
  replicas: 1
  auth:
    username: evolution
    password: "secure-password"
    database: evolution

rabbitmqOperator:
  enabled: true
  replicas: 3  # Required for quorum queues

kafkaStandalone:
  enabled: true

redis:
  enabled: true
```

## Values Reference

See the full [values.yaml](values.yaml) for all configuration options.

### Key Configuration Sections

- `api.*` - Evolution API configuration
- `manager.*` - Evolution Manager (frontend) configuration
- `ingress.*` / `ingressRoute.*` - Ingress configuration
- `postgresql.*` / `postgresqlOperator.*` - PostgreSQL configuration
- `redis.*` - Redis configuration
- `rabbitmqOperator.*` / `rabbitmqStandalone.*` - RabbitMQ configuration
- `kafkaStandalone.*` - Kafka configuration

## Upgrading

```bash
helm upgrade evolution-api . -n evolutionapi -f my-values.yaml
```

## Uninstalling

```bash
helm uninstall evolution-api -n evolutionapi
```

**Note:** PVCs are not automatically deleted. To remove all data:

```bash
kubectl delete pvc -l app.kubernetes.io/instance=evolution-api -n evolutionapi
```

## License

GPL-3.0 - See [LICENSE](LICENSE) file

## Links

- [Evolution API Documentation](https://doc.evolution-api.com)
- [Evolution API GitHub](https://github.com/EvolutionAPI/evolution-api)
