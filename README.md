# Demo App Helm Chart

A production-ready Helm chart for deploying a demo application to Kubernetes with AWS integration, auto-scaling, and comprehensive configuration options.

## Features

- ✅ **AWS Application Load Balancer (ALB)** integration with SSL termination
- ✅ **AWS Secrets Manager** integration with JSON secret extraction
- ✅ **Horizontal Pod Autoscaling (HPA)** with CPU/memory metrics
- ✅ **ConfigMap** for environment variable management
- ✅ **Service Account** with IRSA (IAM Roles for Service Accounts) support
- ✅ **Configurable health probes** for liveness and readiness
- ✅ **Architecture targeting** (ARM64/AMD64) for cost optimization
- ✅ **Production-ready** resource limits and security best practices

## Prerequisites

- Kubernetes 1.19+
- Helm 3.2.0+
- AWS Load Balancer Controller installed in cluster
- AWS Secrets Store CSI Driver (if using AWS Secrets Manager)
- Metrics Server (if using HPA)

## Installation

### Quick Start

```bash
# Install with default values
helm install demo-app .

# Install with custom values
helm install demo-app . -f custom-values.yaml

# Install in specific namespace
helm install demo-app . --namespace demo --create-namespace
```

### Upgrade

```bash
helm upgrade demo-app .
```

### Uninstall

```bash
helm uninstall demo-app
```

## Configuration

### Basic Configuration

| Parameter | Description | Default |
|-----------|-------------|---------|
| `replicaCount` | Number of replicas | `2` |
| `image.repository` | Container image repository | `nginx` |
| `image.tag` | Container image tag | `1.21` |
| `image.pullPolicy` | Image pull policy | `IfNotPresent` |
| `containerPort` | Container port | `8080` |
| `service.type` | Kubernetes service type | `ClusterIP` |
| `service.port` | Service port | `8080` |

### Service Account

```yaml
serviceAccount:
  create: true
  automount: true
  annotations:
    eks.amazonaws.com/role-arn: arn:aws:iam::ACCOUNT:role/ROLE_NAME
  name: ""
```

### Environment Variables (ConfigMap)

```yaml
configMap:
  data:
    APP_ENV: "production"
    LOG_LEVEL: "info"
    PORT: "8080"
    DATABASE_URL: "postgresql://localhost:5432/mydb"
```

### Health Probes

```yaml
probes:
  liveness:
    enabled: true
    httpGet:
      path: /api/status
      port: http
    initialDelaySeconds: 30
    periodSeconds: 10
    timeoutSeconds: 5
    successThreshold: 1
    failureThreshold: 3
  readiness:
    enabled: true
    httpGet:
      path: /api/status
      port: http
    initialDelaySeconds: 5
    periodSeconds: 5
    timeoutSeconds: 3
    successThreshold: 1
    failureThreshold: 3
```

### AWS Application Load Balancer

```yaml
ingress:
  enabled: true
  className: "alb"
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}, {"HTTPS": 443}]'
    alb.ingress.kubernetes.io/ssl-redirect: '443'
    # Optional: Specify ACM certificate ARN
    # alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:region:account:certificate/cert-id
  hosts:
    - host: demo-app.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: demo-app-tls
      hosts:
        - demo-app.example.com
```

### Horizontal Pod Autoscaling

```yaml
autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70
  targetMemoryUtilizationPercentage: 80
```

### AWS Secrets Manager Integration

Configure a single JSON secret in AWS Secrets Manager and extract specific values:

```yaml
secretProviderClass:
  enabled: true
  secretName: "demo-app/secrets"
  jmesPath:
    - path: "database.username"
      alias: "DB_USERNAME"
    - path: "database.password"
      alias: "DB_PASSWORD"
    - path: "api.key"
      alias: "API_KEY"
    - path: "redis.password"
      alias: "REDIS_PASSWORD"
```

**Expected JSON format in AWS Secrets Manager:**

```json
{
  "database": {
    "username": "myuser",
    "password": "mypassword"
  },
  "api": {
    "key": "abc123"
  },
  "redis": {
    "password": "redispass"
  }
}
```

Secrets will be mounted at `/mnt/secrets-store/` with the specified aliases.

### Architecture Targeting

Target specific node pools for cost optimization:

```yaml
# For ARM-based instances (Graviton)
nodeSelector:
  kubernetes.io/arch: arm64

# For Intel/AMD instances
nodeSelector:
  kubernetes.io/arch: amd64
```

### Resource Management

```yaml
resources:
  limits:
    cpu: 500m
    memory: 512Mi
  requests:
    cpu: 250m
    memory: 256Mi
```


## Production Deployment Example

```yaml
# production-values.yaml
replicaCount: 3

image:
  repository: your-registry/demo-app
  tag: "v1.2.3"

containerPort: 8080

serviceAccount:
  create: false
  annotations: []

configMap:
  data:
    APP_ENV: "production"
    LOG_LEVEL: "warn"
    PORT: "8080"

ingress:
  enabled: true
  className: "alb"
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}, {"HTTPS": 443}]'
    alb.ingress.kubernetes.io/ssl-redirect: '443'
    alb.ingress.kubernetes.io/certificate-arn: arn:aws:acm:us-east-1:123456789012:certificate/your-cert-id
  hosts:
    - host: api.yourcompany.com
      paths:
        - path: /
          pathType: Prefix

autoscaling:
  enabled: true
  minReplicas: 3
  maxReplicas: 20
  targetCPUUtilizationPercentage: 70
  targetMemoryUtilizationPercentage: 80

secretProviderClass:
  enabled: true
  secretName: "production/demo-app-secrets"
  jmesPath:
    - path: "database.username"
      alias: "DB_USERNAME"
    - path: "database.password"
      alias: "DB_PASSWORD"
    - path: "api.key"
      alias: "API_KEY"

resources:
  limits:
    cpu: 1000m
    memory: 1Gi
  requests:
    cpu: 500m
    memory: 512Mi

nodeSelector:
  kubernetes.io/arch: arm64  # Use Graviton instances for cost savings
```

Deploy with:

```bash
helm install demo-app . -f production-values.yaml
```

## Troubleshooting

### Common Issues

1. **ALB not creating**: Ensure AWS Load Balancer Controller is installed and has proper IAM permissions
2. **Secrets not mounting**: Verify AWS Secrets Store CSI Driver is installed and IRSA is configured
3. **HPA not scaling**: Check if Metrics Server is running in the cluster
4. **Pods not starting**: Check resource requests don't exceed node capacity

### Debugging Commands

```bash
# Check pod status
kubectl get pods -l app.kubernetes.io/name=demo-app

# View pod logs
kubectl logs -l app.kubernetes.io/name=demo-app

# Check HPA status
kubectl get hpa

# Check ingress status
kubectl get ingress

# Check ALB creation
kubectl describe ingress demo-app

# Check secrets mounting
kubectl exec -it <pod-name> -- ls -la /mnt/secrets-store/
```

## Contributing

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Test with `helm lint .` and `helm template .`
5. Submit a pull request

## License

This Helm chart is licensed under the MIT License.