# Shilp Server Helm Chart

A Helm chart for deploying Shilp Server on Kubernetes.

## Features

- Configurable storage backend (AWS S3 or Persistent Volume)
- Auto-scaling support with HPA
- ConfigMap-based configuration
- Secure secret management for AWS credentials
- Health checks and probes
- Resource limits and requests

## Prerequisites

- Kubernetes 1.19+
- Helm 3.0+
- AWS ECR access (for pulling the image)
- AWS credentials (if using S3 storage)

## Installation

### 1. Create AWS Credentials Secret (if using S3)

```bash
kubectl create secret generic shilp-server-aws-credentials \
  --from-literal=accessKeyId=YOUR_KEY \
  --from-literal=secretAccessKey=YOUR_SECRET
```

If you want to load the secrets from an env file. Store the credentials in file called aws.env

```txt
AWS_ACCESS_KEY_ID=YOUR_KEY
AWS_SECRET_ACCESS_KEY=YOUR_SECRET
```

Make sure the to add this file to `.gitignore `to avoid committing. Then run the following command

```bash
kubectl create secret generic shilp-server-aws-credentials  --from-env-file=aws.env
```

### 2. Create Server Credentials Secret

The chart reads `SETTINGS_ENCRYPTION_KEY` from a Kubernetes secret. Create a secret named `shilp-server-credentials` with the `encryptionKey` entry:

```bash
kubectl create secret generic shilp-server-credentials \
  --from-literal=encryptionKey=YOUR_ENCRYPT_KEY
```

If you want to use a different secret name or key name, configure them with Helm values:

```yaml
server:
  existingSecret: shilp-server-credentials
  encryptKeyKey: encryptionKey
```

### 3. Install the Chart

#### For local testing

```bash
helm install shilp-server ./shilp-server --force -f shilp-server/values-dev.yaml
```

#### With S3 Storage (No Persistent Volume)

```bash
helm install shilp-server ./helm/shilp-server \
  --set env.aws.enableStorage=true \
  --set env.aws.region=ap-south-1 \
  --set env.aws.s3UploadBucket=your-upload-bucket \
  --set env.aws.s3DbBucket=your-db-bucket \
  --set awsCredentials.existingSecret=shilp-server-aws-credentials \
  --set server.existingSecret=shilp-server-credentials
```

#### With Persistent Volume Storage

```bash
helm install shilp-server ./helm/shilp-server \
  --set env.aws.enableStorage=false \
  --set persistence.enabled=true \
  --set persistence.size=20Gi \
  --set persistence.storageClass=standard \
  --set server.existingSecret=shilp-server-credentials \
  --set env.shilpFilePath=/data/shilp \
  --set env.shilpDbPath=/data/db
```

### 4. Upgrade an Existing Installation

```bash
helm upgrade shilp-server ./helm/shilp-server \
  --set env.aws.s3UploadBucket=new-bucket \
  --set server.existingSecret=shilp-server-credentials \
  --reuse-values
```

### 5. Uninstall

```bash
helm uninstall shilp-server
```

## Configuration

### Key Parameters

| Parameter                       | Description                         | Default                                                              |
| ------------------------------- | ----------------------------------- | -------------------------------------------------------------------- |
| `replicaCount`                  | Number of replicas                  | `1`                                                                  |
| `image.repository`              | Image repository                    | `255325274555.dkr.ecr.ap-south-1.amazonaws.com/anvitra/shilp-server` |
| `image.tag`                     | Image tag                           | `arm64_linux`                                                        |
| `env.aws.enableStorage`         | Use AWS S3 for storage              | `false`                                                              |
| `env.aws.region`                | AWS region                          | `ap-south-1`                                                         |
| `env.aws.s3UploadBucket`        | S3 bucket for uploads               | `""`                                                                 |
| `env.aws.s3DbBucket`            | S3 bucket for database              | `""`                                                                 |
| `env.enableMetrics`             | Enable metrics                      | `true`                                                               |
| `env.autoLoadCollections`       | Auto-load collections on startup    | `""`                                                                 |
| `awsCredentials.existingSecret` | Name of secret with AWS credentials | `""`                                                                 |
| `server.existingSecret`         | Name of secret with server credentials | `shilp-server-credentials`                                        |
| `server.encryptKeyKey`          | Secret key used for `SETTINGS_ENCRYPTION_KEY` | `encryptionKey`                                             |
| `persistence.enabled`           | Enable persistent volume            | `true`                                                               |
| `persistence.size`              | Size of persistent volume           | `10Gi`                                                               |
| `persistence.storageClass`      | Storage class name                  | `""`                                                                 |
| `initContainers`                | Init containers to run before main  | `[]` (includes data directory setup)                                 |
| `nodeSelector`                  | Node labels for pod scheduling      | `{}`                                                                 |
| `tolerations`                   | Tolerations for tainted nodes       | `[]`                                                                 |
| `service.type`                  | Service type                        | `ClusterIP`                                                          |
| `service.port`                  | Service port                        | `3000`                                                               |
| `resources.limits.cpu`          | CPU limit                           | `1000m`                                                              |
| `resources.limits.memory`       | Memory limit                        | `1Gi`                                                                |
| `autoscaling.enabled`           | Enable HPA                          | `false`                                                              |

### Storage Configuration

The chart supports two storage modes:

#### 1. AWS S3 Storage (Recommended for Production)

- Set `env.aws.enableStorage=true`
- No persistent volume needed
- Requires AWS credentials secret
- Data stored in S3 buckets

#### 2. Persistent Volume Storage

- Set `env.aws.enableStorage=false`
- Set `persistence.enabled=true`
- Requires persistent volume
- Data stored locally in cluster

**Note:** When `env.aws.enableStorage=true`, the persistent volume is automatically disabled.

### Environment Variables

All environment variables from `server/sample.env` are supported:

```yaml
env:
  aws:
    enableStorage: "true"
    region: "ap-south-1"
    s3UploadBucket: "my-upload-bucket"
    s3DbBucket: "my-db-bucket"
    enableEmbeddingModels: "false"
    defaultEmbeddingModel: ""
  enableMetrics: "true"
  # collections to be loaded into memory on startup
  autoLoadCollections: "collection1,collection2"
  # Shilp file path
  shilpFilePath: "/data/shilp"
  # Upload file path
  uploadFilePath: "/data/upload"
  # Log level: "debug", "info", "warn", "error"
  logLevel: "info"
```

### Custom Values File

Create a `custom-values.yaml`:

```yaml
replicaCount: 2

image:
  tag: "latest"

imagePullSecrets:
  - name: ecr-credentials

env:
  aws:
    enableStorage: "true"
    region: "ap-south-1"
    s3UploadBucket: "my-uploads"
    s3DbBucket: "my-database"
  enableMetrics: "true"
  autoLoadCollections: "go_stocks,nasdaq_stocks"

awsCredentials:
  existingSecret: "shilp-server-aws-credentials"

server:
  existingSecret: "shilp-server-credentials"
  encryptKeyKey: "encryptionKey"

service:
  type: LoadBalancer
  port: 3000

resources:
  limits:
    cpu: 2000m
    memory: 4Gi
  requests:
    cpu: 2500m
    memory: 5Gi

autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70
```

Install with custom values:

```bash
helm install shilp-server ./helm/shilp-server -f custom-values.yaml
```

## Accessing the Application

### Port Forward (Development)

```bash
kubectl port-forward svc/shilp-server 3000:3000
```

Then access: http://localhost:3000

### Copy files for ingestion

If you want to copy files for ingestion, you can use the following command

```bash
kubectl cp my_stocks.csv \
$(kubectl get pods -l app.kubernetes.io/name=shilp-server -o jsonpath='{.items[0].metadata.name}'):/data/upload/
```

Note you `my_stocks.csv` is the sample file name. You can replace it with the filename you want to upload. Note the file should available in the local where you are running the command. If you are using S3, then you can just upload the file into S3 upload bucket

### Using LoadBalancer

If service type is LoadBalancer:

```bash
kubectl get svc shilp-server
```

Use the EXTERNAL-IP to access the service.

## Monitoring

### Check Pod Status

```bash
kubectl get pods -l app.kubernetes.io/name=shilp-server
```

### View Logs

```bash
kubectl logs -l app.kubernetes.io/name=shilp-server -f
```

### Check Health

```bash
kubectl exec -it deployment/shilp-server -- curl localhost:3000/health
```

## Troubleshooting

### Pod Not Starting

1. Check image pull secrets:

```bash
kubectl get pods
kubectl describe pod <pod-name>
```

2. Verify ECR credentials are valid

3. Check resource availability

### Storage Issues

If using persistent volume:

```bash
kubectl get pvc
kubectl describe pvc shilp-server
```

If using S3:

- Verify AWS credentials secret exists
- Check S3 bucket permissions
- Ensure IAM role has proper permissions

### Configuration Issues

Check ConfigMap:

```bash
kubectl get configmap shilp-server -o yaml
```

Check Secrets:

```bash
kubectl get secret shilp-server-aws-credentials
```

## Examples

### Development Setup

```bash
# Minimal setup with S3 storage
helm install shilp-server ./helm/shilp-server --force -f helm/shilp-server/values-dev.yaml
```

### Production Setup with S3

```bash
# Production with S3, autoscaling
helm install shilp-server ./helm/shilp-server --force -f helm/shilp-server/values-prod.yaml
```

### Node Isolation with Taints and Tolerations

To ensure dedicated nodes for shilp-server that won't run other applications (and vice-versa), use Kubernetes taints and tolerations:

#### Step 1: Taint Dedicated Nodes

First, taint your dedicated nodes to prevent other workloads from scheduling:

```bash
# Taint nodes for shilp-server workloads only
kubectl taint nodes <node-name> app=shilp-server:NoSchedule
kubectl taint nodes <node-name> dedicated=shilp:NoExecute

# Label the nodes for easier selection
kubectl label nodes <node-name> workload=shilp-server
```

**Taint Effects:**

- `NoSchedule`: Prevents new pods without matching tolerations from scheduling
- `NoExecute`: Evicts existing pods without matching tolerations
- `PreferNoSchedule`: Soft preference (tries to avoid scheduling)

Then deploy:

```bash
helm install shilp-server ./helm/shilp-server -f custom-values.yaml
```

#### Benefits of This Approach

1. **Node Isolation**: Dedicated nodes only run shilp-server pods
2. **Resource Guarantee**: Prevents other applications from consuming resources
3. **Security**: Workload isolation for sensitive data
4. **Performance**: Predictable performance without noisy neighbors
5. **Compliance**: Meet regulatory requirements for workload separation

#### Verification

```bash
# Check node taints
kubectl describe node <node-name> | grep Taints

# Verify pods are scheduled on correct nodes
kubectl get pods -o wide -l app.kubernetes.io/name=shilp-server

# Check pod tolerations
kubectl get pod <pod-name> -o jsonpath='{.spec.tolerations}'
```
