# Vault Helm Chart

A Helm chart for deploying HashiCorp Vault on Kubernetes, converted from Vault CRD to standard Kubernetes resources.

## Introduction

This chart deploys HashiCorp Vault as a StatefulSet with Raft storage backend, providing a highly available secrets management solution. The chart is based on the original Vault CRD configuration but converted to use standard Helm templating.

## Prerequisites

- Kubernetes 1.19+
- Helm 3.0+
- PV provisioner support in the underlying infrastructure
- Storage class for persistent volumes

## Installing the Chart

```bash
# Basic installation
helm install my-vault ./vault-from-crd

# With custom values
helm install my-vault ./vault-from-crd --values custom-values.yaml

# With specific values
helm install my-vault ./vault-from-crd \
  --set vault.size=3 \
  --set vault.image.tag=1.14.1 \
  --set vault.storage.size=10Gi
```

## Configuration

The following table lists the configurable parameters of the vault chart and their default values.

| Parameter | Description | Default |
|-----------|-------------|---------|
| `vault.size` | Number of Vault replicas | `3` |
| `vault.image.repository` | Vault image repository | `hashicorp/vault` |
| `vault.image.tag` | Vault image tag | `1.14.1` |
| `vault.service.type` | Kubernetes service type | `ClusterIP` |
| `vault.service.port` | Service port | `8200` |
| `vault.storage.enabled` | Enable persistent storage | `true` |
| `vault.storage.size` | PVC size | `1Gi` |
| `vault.storage.storageClassName` | Storage class name | `""` |
| `vault.ingress.enabled` | Enable ingress | `false` |
| `tls.enabled` | Enable TLS | `true` |

### Vault Configuration

The Vault configuration is managed through the `vault.config` section:

```yaml
vault:
  config:
    storage:
      raft:
        path: "/vault/file"
    listener:
      tcp:
        address: "0.0.0.0:8200"
        tls_cert_file: /vault/tls/server.crt
        tls_key_file: /vault/tls/server.key
    api_addr: https://vault.default:8200
    cluster_addr: "https://${.Env.POD_NAME}:8201"
    ui: true
```

### External Configuration

The chart supports external Vault configuration including policies, authentication, and secrets:

```yaml
vault:
  externalConfig:
    policies:
      - name: allow_secrets
        rules: path "secret/*" {
          capabilities = ["create", "read", "update", "delete", "list"]
          }
    
    auth:
      - type: kubernetes
        roles:
          - name: devops-400
            bound_service_account_names: ["devops-400", "vault-secrets-webhook"]
            bound_service_account_namespaces: ["devops-400", "vswh"]
            policies: allow_secrets
            ttl: 1h
    
    secrets:
      - path: secret
        type: kv
        description: General secrets.
        options:
          version: 2
```

### Unseal Configuration

Configure Vault unsealing parameters:

```yaml
vault:
  unsealConfig:
    options:
      preFlightChecks: true
      storeRootToken: true
      secretShares: 5
      secretThreshold: 3
    kubernetes:
      secretNamespace: devops-400
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
   helm upgrade my-vault ./vault-from-crd --set vault.ingress.enabled=true
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

### Joining Vault Cluster

For multi-node deployments, join additional nodes to the cluster:

```bash
# On the leader node
vault operator raft list-peers

# On follower nodes
vault operator raft join https://vault-0.vault:8200
```

## Security Considerations

- TLS is enabled by default
- Vault runs as non-root user (UID 1000)
- Pod security context is configured
- RBAC is enabled for proper permissions
- Velero backup hooks are configured for disaster recovery

## Monitoring

Enable Prometheus monitoring:

```yaml
monitoring:
  serviceMonitor:
    enabled: true
    additionalLabels:
      release: prometheus
```

## Backup and Disaster Recovery

The chart includes Velero backup hooks for automated backups:

```yaml
vault:
  veleroEnabled: true
```

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
kubectl exec -it vault-0 -- vault status
```

### Check Raft Peers
```bash
kubectl exec -it vault-0 -- vault operator raft list-peers
```

### Check Storage
```bash
kubectl get pvc -l app.kubernetes.io/name=vault
```

## Advanced Configuration

### Custom Annotations and Labels

```yaml
vault:
  annotations:
    common/annotation: "true"
  
  vaultAnnotations:
    type/instance: "vault"
  
  vaultLabels:
    example.com/log-format: "json"
```

### Resource Limits

```yaml
vault:
  resources:
    vault:
      limits:
        memory: "1Gi"
        cpu: "500m"
      requests:
        memory: "512Mi"
        cpu: "250m"
```

### Node Affinity and Tolerations

```yaml
vault:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: node-role.kubernetes.io/vault
            operator: In
            values: ["true"]
  
  tolerations:
  - effect: NoSchedule
    key: node-role.kubernetes.io/vault
    operator: Equal
    value: "true"
```

## Contributing

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Test your changes
5. Submit a pull request

## License

This project is licensed under the MIT License. 