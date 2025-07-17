# Vault Helm Chart

A Helm chart for deploying HashiCorp Vault on Kubernetes.

## Introduction

This chart deploys HashiCorp Vault, a tool for secrets management, encryption as a service, and privileged access management.

## Prerequisites

- Kubernetes 1.19+
- Helm 3.0+
- PV provisioner support in the underlying infrastructure

## Installing the Chart

```bash
helm install my-vault ./vault-app
```

## Configuration

The following table lists the configurable parameters of the vault chart and their default values.

| Parameter | Description | Default |
|-----------|-------------|---------|
| `replicaCount` | Number of vault replicas | `1` |
| `image.repository` | Vault image repository | `hashicorp/vault` |
| `image.tag` | Vault image tag | `1.14.1` |
| `image.pullPolicy` | Image pull policy | `IfNotPresent` |
| `service.type` | Kubernetes service type | `ClusterIP` |
| `service.port` | Service port | `8200` |
| `persistence.enabled` | Enable persistence | `true` |
| `persistence.size` | PVC size | `1Gi` |
| `persistence.storageClassName` | Storage class name | `""` |
| `ingress.enabled` | Enable ingress | `false` |
| `tls.enabled` | Enable TLS | `true` |

### Vault Configuration

The Vault configuration is managed through the `vault.config` section in values.yaml:

```yaml
vault:
  config:
    storage:
      raft:
        node_id: "node-1"
    listener:
      tcp:
        tls_cert_file: /vault/tls/server.crt
        tls_key_file: /vault/tls/server.key
    api_addr: "https://vault:8200"
    cluster_addr: "https://vault:8201"
    ui: true
```

### Deployment Configuration

Deployment-specific settings can be configured under the `deployment` section:

```yaml
deployment:
  replicaCount: 1
  containerPort: 8200
  resources:
    limits:
      memory: "512Mi"
      cpu: "200m"
    requests:
      memory: "256Mi"
      cpu: "100m"
  env:
    - name: VAULT_LOG_LEVEL
      value: "info"
```

## Usage

### Accessing Vault

Once deployed, you can access Vault through:

1. **Port Forward**:
   ```bash
   kubectl port-forward svc/my-vault 8200:8200
   ```

2. **Ingress** (if enabled):
   ```bash
   # Configure ingress in values.yaml
   helm upgrade my-vault ./vault-app --set ingress.enabled=true
   ```

### Initializing Vault

After deployment, you need to initialize Vault:

```bash
# Port forward to access Vault
kubectl port-forward svc/my-vault 8200:8200

# Initialize Vault
vault operator init

# Unseal Vault (repeat 3 times with different keys)
vault operator unseal <key1>
vault operator unseal <key2>
vault operator unseal <key3>
```

## Security Considerations

- TLS is enabled by default
- Vault runs as non-root user (UID 1000)
- Pod security context is configured
- RBAC is enabled for proper permissions

## Troubleshooting

### Check Pod Status
```bash
kubectl get pods -l app.kubernetes.io/name=vault
```

### View Logs
```bash
kubectl logs -l app.kubernetes.io/name=vault
```

### Check Vault Status
```bash
kubectl exec -it <vault-pod> -- vault status
```

## Contributing

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Test your changes
5. Submit a pull request

## License

This project is licensed under the MIT License.

