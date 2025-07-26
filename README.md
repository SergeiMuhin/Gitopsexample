# GitOps Demo - Complete CI/CD GitOps Pipeline

A production-ready demonstration of modern DevOps practices featuring a complete CI/CD pipeline with GitOps deployment for a Redis StatefulSet on Kubernetes.

## 🏗️ Architecture Overview

This project demonstrates a full CI/CD flow using:
- **KIND** (Kubernetes in Docker) for local cluster
- **GitHub Actions** for CI/CD pipeline
- **DockerHub** for container registry
- **ArgoCD** for GitOps deployment
- **Helm** for Kubernetes package management
- **NGINX Ingress** for traffic routing
- **Redis StatefulSet** with persistent storage

## 🚀 Quick Start

### Prerequisites
- Docker
- kubectl
- kind
- helm
- git

### 1. Create KIND Cluster
- containerPort: 443 <-- is not a must not being used in this demo

```bash
# Create cluster configuration
cat <<EOF > kind-config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: gitops-demo
nodes:
  - role: control-plane
    extraPortMappings:
      - containerPort: 80
        hostPort: 8080
        protocol: TCP
      - containerPort: 443
        hostPort: 8443
        protocol: TCP
      - containerPort: 30379
        hostPort: 6379
        protocol: TCP
EOF

# Create cluster
kind create cluster --name gitops-demo --config kind-config.yaml
kubectl config use-context kind-gitops-demo
```

### 2. Install NGINX Ingress Controller

```bash
# Install NGINX Ingress for KIND
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.10.0/deploy/static/provider/kind/deploy.yaml

# Add ingress-ready label to node
kubectl label node gitops-demo-control-plane ingress-ready=true

# If the label doesn't work, remove node selector (fallback option)
# kubectl patch deployment ingress-nginx-controller -n ingress-nginx \
#   --type='json' \
#   -p='[{"op": "remove", "path": "/spec/template/spec/nodeSelector"}]'

# Wait for ingress controller to be ready
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=300s
```

### 3. Install ArgoCD

```bash
# Create ArgoCD namespace and install
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

# Configure ArgoCD for insecure access (development only)
kubectl -n argocd patch deployment argocd-server \
  --type='json' \
  -p='[{"op": "add", "path": "/spec/template/spec/containers/0/args/-", "value":"--insecure"}]'

kubectl -n argocd rollout restart deployment argocd-server
```

### 4. Configure ArgoCD Ingress

```bash
# Create ArgoCD ingress for web access
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: argocd-server
  namespace: argocd
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "false"
spec:
  ingressClassName: nginx
  rules:
    - host: argocd.localtest.me
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: argocd-server
                port:
                  number: 80
EOF

!Not a must but can assist if having issue.
# Add to hosts file
echo "127.0.0.1 argocd.localtest.me" | sudo tee -a /etc/hosts
```

### 5. Access ArgoCD

```bash
# Get admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d && echo

# Access ArgoCD UI
# URL: http://argocd.localtest.me:8080
# Username: admin
# Password: (from command above)
```

### 6. Deploy Redis Application

Important: Only proceed after CI pipeline has completed successfully and the container image is available.

```bash
# Apply ArgoCD application (after successful CI build)
kubectl apply -f argocd/redis-app.yaml
```

### 7. Test External Access

```bash
# Test Redis connectivity
redis-cli -h localhost -p 6379 ping
# Expected output: PONG
```

## 🔄 CI/CD Pipeline

### GitHub Actions Workflow

The CI/CD pipeline automatically:

1. **Triggers** on changes to `README.md` or `app/**` or `.github/workflows/**`
2. **Builds** Docker image from `app/Dockerfile`
3. **Pushes** to DockerHub with commit SHA tag
4. **Updates** Helm chart with new image tag
5. **Commits** changes back to repository

### Pipeline Configuration

Located in `.github/workflows/ci.yml`:

```yaml
name: CI/CD Pipeline
on:
  push:
    branches: [main]
    paths: ['README.md', 'app/**', '.github/workflows/**']

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Login to DockerHub
        uses: docker/login-action@v2
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}
      - name: Build and push
        uses: docker/build-push-action@v4
        with:
          context: ./app
          push: true
          tags: sergeisin/redis-stateful:${{ github.sha }}
      - name: Update Helm values
        run: |
          sed -i "s/tag: \".*\"/tag: \"${{ github.sha }}\"/" helm-chart/redis-stateful/values.yaml
          git config --local user.email "action@github.com"
          git config --local user.name "GitHub Action"
          git add helm-chart/redis-stateful/values.yaml
          git commit -m "Update image tag to ${{ github.sha }}"
          git push
```

### Required Secrets

Configure these secrets in your GitHub repository settings (Settings → Secrets and variables → Actions):
- `DOCKERHUB_USERNAME`: Your DockerHub username
- `DOCKERHUB_TOKEN`: DockerHub access token

Note: Create the DockerHub access token in DockerHub settings, not your GitHub repository.

## 📦 Application Architecture

### Redis StatefulSet

The Redis deployment includes:

- **StatefulSet**: Ordered pod deployment with stable network identity
- **Persistent Storage**: VolumeClaimTemplates for data persistence
- **Headless Service**: For internal StatefulSet communication
- **NodePort Service**: For external access
- **Resource Limits**: CPU and memory constraints

### Helm Chart Structure

```
helm-chart/redis-stateful/
├── Chart.yaml
├── values.yaml
└── templates/
    ├── service-headless.yaml
    ├── service.yaml
    └── statefulset.yaml
```

### Key Configuration

!!Notice!! sergeisin/redis-stateful <--- location of my dockerhub registry

```yaml
# values.yaml
image:
  repository: sergeisin/redis-stateful
  tag: "latest"

service:
  type: ClusterIP
  port: 6379
  headless:
    enabled: true

persistence:
  enabled: true
  size: 1Gi
  accessMode: ReadWriteOnce

exposure:
  type: "nodeport"
  nodePort: 30379
```

## 🎯 Demonstrating Persistent Storage

This section shows how StatefulSet with persistent storage maintains data across pod restarts and failures.

### Step 1: Add Sample Data to Redis

Connect to Redis and populate with demo data:

```bash
# Connect to Redis
redis-cli -h localhost -p 6379

# Add various types of data (run inside redis-cli)
SET demo:name "GitOps Demo Project"
SET demo:version "1.0.0"
SET demo:created "$(date)"
HSET user:1001 name "Gitops Demo" email "gitops@example.com" role "DevOps Engineer"
HSET user:1002 name "Sergei Demo" email "sergei@example.com" role "Site Reliability Engineer"
LPUSH demo:features "CI/CD Pipeline" "GitOps with ArgoCD" "Persistent Storage" "StatefulSet Deployment"
SADD demo:technologies "Kubernetes" "Docker" "Helm" "Redis" "GitHub Actions" "KIND"
ZADD demo:priorities 100 "Data Persistence" 90 "Automated Deployment" 85 "Service Discovery"

# Verify data exists
GET demo:name
HGETALL user:1001
LRANGE demo:features 0 -1
SMEMBERS demo:technologies
ZRANGE demo:priorities 0 -1 WITHSCORES

# Exit Redis
EXIT
```

### Step 2: Simulate Pod Failure

Delete the Redis pod to simulate a failure or restart:

```bash
# Check current pod status
kubectl get pods -l app.kubernetes.io/name=redis-stateful

# Delete the pod (simulating failure)
kubectl delete pod redis-stateful-0

# Watch the pod recreate automatically
kubectl get pods -l app.kubernetes.io/name=redis-stateful -w
# Press Ctrl+C when pod shows STATUS: Running
```

### Step 3: Verify Data Persistence

Reconnect to the new pod and verify all data survived:

```bash
# Reconnect to Redis (wait for pod to be Ready)
redis-cli -h localhost -p 6379

# Verify all data types are intact
GET demo:name
GET demo:version
HGETALL user:1001
HGETALL user:1002
LRANGE demo:features 0 -1
SMEMBERS demo:technologies
ZRANGE demo:priorities 0 -1 WITHSCORES

# All data should be exactly as before pod deletion
EXIT
```

### Step 4: Inspect Persistent Volume Claims

Check the underlying storage infrastructure:

```bash
# View PersistentVolumeClaim status
kubectl get pvc
kubectl describe pvc redis-data-redis-stateful-0

# Check PersistentVolume details
kubectl get pv

# View storage location on KIND node
docker volume ls
# Example: local f1c13d3d74191a30b0b8a6a3f42b317db59ead411588ce69b3e592fe623cf5e3
# Check PV details to confirm
kubectl get pv -o yaml | grep f1c13d3d74191a30b0b8a6a3f42b317db59ead411588ce69b3e592fe623cf5e3
# You will have "Mountpoint" **/f1c13d3d74191a30b0b8a6a3f42b317db59ead411588ce69b3e592fe623cf5e3

Docker Volume (local driver) → KIND node mount → Kubernetes PV → Redis Pod
```

### Expected Results

**✅ Data Persistence**: All Redis data (strings, hashes, lists, sets, sorted sets) survives pod deletion  
**✅ StatefulSet Behavior**: Pod recreates with same name (`redis-stateful-0`) and identity  
**✅ Storage Continuity**: PVC remains bound and data directory persists on node  
**✅ Service Discovery**: Application remains accessible at same endpoints  

### Demo Explanation

This demonstration proves several key concepts:

#### StatefulSet vs Deployment
- **StatefulSet**: Maintains pod identity and persistent storage across restarts
- **Deployment**: Treats pods as cattle, no persistent identity or storage guarantees

#### Persistent Storage Benefits
- **Data Durability**: Application state survives infrastructure failures
- **Stateful Applications**: Enables databases, caches, and other data-persistent services
- **Disaster Recovery**: Data remains available during planned maintenance or unexpected failures

#### Production Implications
- **High Availability**: Application can restart without data loss
- **Backup Strategy**: Data persistence enables point-in-time recovery
- **Scaling Considerations**: Each StatefulSet replica gets independent storage

### Cleanup Demo Data (Optional)

To clean up the demo data while keeping the application running:

```bash
redis-cli -h localhost -p 6379 FLUSHDB
```

This demonstration showcases the power of Kubernetes StatefulSets for managing stateful applications with guaranteed data persistence and ordered deployment patterns.

## 🌐 Demonstrating Kubernetes DNS Resolution

This section showcases how Kubernetes provides service discovery through DNS, demonstrating both regular and headless service resolution patterns.

### Step 1: Create a Debug Pod for DNS Testing

```bash
# Create a debug pod with DNS tools
kubectl run dns-debug --image=busybox:1.35 --rm -it --restart=Never -- sh
```

### Step 2: Test Regular Service DNS Resolution

```bash
# Test DNS resolution for the regular Redis service
nslookup redis-stateful.default.svc.cluster.local

# Short form DNS (same namespace)
nslookup redis-stateful
```

**Expected Output:**
```
Server:         10.96.0.10
Address:        10.96.0.10:53

Name:   redis-stateful.default.svc.cluster.local
Address: 10.96.21.181  # ClusterIP of the service
```

### Step 3: Test Service Connectivity

```bash
# Test connectivity to the service
ping -c 3 redis-stateful.default.svc.cluster.local

# Test actual service port
telnet redis-stateful.default.svc.cluster.local 6379
```

**Expected Output:**
- **DNS Resolution**: Shows the correct ClusterIP (e.g., 10.96.21.181)
- **Ping**: May show 100% packet loss (services often don't respond to ICMP)
- **Telnet**: Should connect successfully to Redis port

### Step 4: Test Headless Service vs Regular Service

```bash
# Regular service - returns ClusterIP (load balanced)
echo "=== Regular Service DNS ==="
nslookup redis-stateful.default.svc.cluster.local

echo -e "\n=== Headless Service DNS ==="
# Headless service - returns pod IP(s) directly
nslookup redis-stateful-headless.default.svc.cluster.local

echo -e "\n=== Individual Pod DNS (StatefulSet) ==="
# Individual pod - returns specific pod IP
nslookup redis-stateful-0.redis-stateful-headless.default.svc.cluster.local
```

**Expected Output:**
- **Regular Service**: `10.96.21.181` (ClusterIP - virtual IP for load balancing)
- **Headless Service**: `10.244.0.20` (Direct pod IP - no kube-proxy)
- **Individual Pod**: `10.244.0.20` (Specific pod IP with predictable DNS name)

### Cleanup

```bash
# Exit the debug pod
exit
```

### DNS Resolution Patterns Explained

#### Regular Service (ClusterIP)
- **DNS**: Returns virtual ClusterIP
- **Traffic Flow**: Client → ClusterIP → kube-proxy → Pod IP
- **Use Case**: Standard service access with automatic load balancing

#### Headless Service
- **DNS**: Returns actual pod IP(s) directly
- **Traffic Flow**: Client → Pod IP directly (no kube-proxy)
- **Use Case**: StatefulSet identity, database clustering, direct pod access

#### StatefulSet Integration
- **Predictable DNS**: `redis-stateful-0.redis-stateful-headless.default.svc.cluster.local`
- **Stable Identity**: DNS name survives pod restarts
- **Use Case**: Database replication, leader election, cluster coordination

### Common Issues

**DNS Resolution Works but Ping Fails**: Services often don't respond to ICMP ping packets. Use `telnet` to test actual service ports.

**NXDOMAIN Errors**: Normal behavior when using short DNS names - Kubernetes tries multiple search domains before finding the correct one.

### Key Takeaways

- **Service Discovery**: Kubernetes provides automatic DNS for all services
- **Regular Services**: Use kube-proxy for load balancing via virtual ClusterIP
- **Headless Services**: Direct pod access without kube-proxy (internal cluster use only)
- **StatefulSets**: Get predictable, stable DNS names for stateful applications

## 🌐 Service Accessibility

### Design Challenge: Making Redis Accessible from Local Environment

**Requirement**: Make the StatefulSet application accessible for testing and learning
**Solution**: Direct Redis access via NodePort

### Solution Architecture: Direct Redis Access

**Selected Approach**: Expose Redis directly via NodePort
```
Local Tools (redis-cli, RedisInsight) → NodePort → Redis StatefulSet
```

### Design Choice: NodePort over Ingress

**Selected**: NodePort for direct Redis access
**Rationale**:
- **Simplicity**: Direct database access without additional layers
- **KIND Compatibility**: Works perfectly with KIND port forwarding
- **Learning Focus**: Direct access for testing and understanding StatefulSets
- **Tool Integration**: Allows connection with standard Redis tools
- **Educational Value**: Simple, straightforward architecture to learn from

**Alternative Considered**: NGINX Ingress
- **Note**: NGINX Ingress IS used in this project for ArgoCD (HTTP/HTTPS traffic)
- **Redis Consideration**: Ingress handles Layer 7 (HTTP), Redis needs Layer 4 (TCP)
- **Technical Options**: 
  - TCP stream forwarding (complex in KIND)
  - SNI routing with TLS (overkill for learning)
- **Decision**: NodePort for TCP services, Ingress for HTTP services

### Access Methods

#### External Access (Learning & Testing)
```bash
# Direct Redis connection
redis-cli -h localhost -p 6379

# Connection from Redis tools
localhost:6379

# Test connectivity
telnet localhost 6379
```

#### Internal Cluster Access
```bash
# Service-to-service communication
redis-stateful.default.svc.cluster.local:6379

# Pod-to-pod direct access (StatefulSet)
redis-stateful-0.redis-stateful-headless.default.svc.cluster.local:6379
```

### KIND Port Mapping Configuration
```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  extraPortMappings:
  - containerPort: 30379    # Redis NodePort
    hostPort: 6379
    protocol: TCP
```

## 🔧 Technical Decisions

### StatefulSet Application Choice
**Choice**: Redis with persistent storage
- **Pros**: Demonstrates StatefulSet concepts, data persistence, stable network identity
- **Learning Value**: Direct Redis access via NodePort for testing and validation
- **Use Case**: Perfect for showing persistent storage across pod restarts

### Storage Class
**Choice**: Default local-path provisioner (rancher.io/local-path)
- **Pros**: Works out-of-box with KIND, suitable for learning environments
- **Cons**: Data tied to specific node, no replication
- **Production Alternative**: Network-attached storage (EBS, Ceph, GlusterFS)

### Container Registry
**Choice**: DockerHub public registry
- **Pros**: Free, widely supported, simple CI/CD integration
- **Cons**: Public images, potential rate limiting
- **Production Alternative**: Private registries (Amazon ECR, Google GCR, Harbor)

### Networking Strategy
**Choice**: Hybrid approach - NodePort for TCP, Ingress for HTTP
- **Redis (TCP)**: NodePort with KIND port mapping
- **ArgoCD (HTTP)**: NGINX Ingress Controller
- **Rationale**: Use the right tool for each protocol layer
- **Learning Value**: Demonstrates both Layer 4 and Layer 7 networking

### Service Design Pattern
**Overall Architecture**:
```
┌─────────────────┐    ┌─────────────────┐
│   Local Tools   │────│ Redis StatefulSet│
│  (redis-cli)    │    │   (NodePort)    │  
└─────────────────┘    └─────────────────┘

┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Web Browser   │────│  NGINX Ingress  │────│  ArgoCD Service │
│                 │    │  (Layer 7)      │    │  (ClusterIP)    │
└─────────────────┘    └─────────────────┘    └─────────────────┘
```

**Benefits**:
- **Protocol Separation**: TCP (NodePort) vs HTTP (Ingress)
- **Educational Value**: Shows both networking approaches
- **Practical Learning**: Different tools for different use cases
- **Production Insight**: Demonstrates when to use each approach

### Key Learning Points
1. **StatefulSet Value**: Redis demonstrates persistent storage and stable network identity
2. **Service Types**: NodePort for TCP, Ingress for HTTP, headless service for StatefulSet identity
3. **Protocol Layers**: Layer 4 (TCP/NodePort) vs Layer 7 (HTTP/Ingress) networking
4. **Tool Selection**: Different tools for different protocols and use cases
5. **KIND Limitations**: TCP stream forwarding complexity in local environments
6. **Production Patterns**: How local choices translate to cloud environments

### 🔧 Helm Chart Quality & Kubernetes Manifest Hygiene

**Chart Structure and Standards:**
- **Semantic versioning**: Follow SemVer for chart versions with automated bumping in CI/CD
- **Chart linting**: Use `helm lint` and `ct lint` (chart-testing) in CI pipeline for validation
- **Chart documentation**: Generate README.md from values.yaml using helm-docs
- **Dependency management**: Pin chart dependencies with version ranges and lock files

**Kubernetes Manifest Best Practices:**
- **Resource management**: Define CPU/memory requests and limits for all containers
- **Security contexts**: Run containers as non-root with read-only root filesystem
- **Health checks**: Implement readiness and liveness probes for reliable deployments
- **Labels and annotations**: Use consistent labeling strategy following Kubernetes recommended labels

**Template Functions and Reusability:**
- **Template helpers**: Create reusable template functions for common patterns (labels, selectors)
- **Values validation**: Use JSON Schema or template conditionals to validate user inputs
- **Multi-environment support**: Design templates to work across dev/staging/production environments
- **Resource naming**: Use consistent naming conventions with release and chart name prefixes

**Chart Testing and Quality Assurance:**
- **Unit testing**: Test Helm templates with different values combinations
- **Integration testing**: Deploy charts in test environments with chart-testing tool
- **Security scanning**: Scan generated manifests for security misconfigurations
- **Documentation testing**: Validate that chart documentation matches actual behavior

## 🚀 Production Readiness Extensions

## 🔐 Secret Management

### Current: Plain text configuration
### Production: Layered secret security

**Enterprise Registry Authentication:**
- **Certificate-based authentication**: Use client certificates for mutual TLS authentication with private registries instead of username/password
- **OIDC integration**: Leverage cloud provider OIDC for registry authentication without storing credentials
- **Service mesh integration**: Use service mesh identity for registry access within cluster

**Secret Mounting (Enterprise Approach):**
- **File-based secrets**: Mount secrets as files instead of environment variables for better security
- **External secret sync**: Use External Secret Operator to sync from AWS Secrets Manager with automatic updates (5-15 second sync interval)
- **Volume mounts**: Applications read secrets from mounted files that automatically reflect external changes
- **Restricted permissions**: Set appropriate file permissions (0400) on mounted secret files

**External Secret Management:**
- **Vault integration**: Use HashiCorp Vault with dynamic secret generation and automatic rotation
- **Cloud secret managers**: Integrate with AWS Secrets Manager, Azure Key Vault, or GCP Secret Manager via OIDC (using External Secret Operator)
- **Secret injection**: Use tools like Bank-Vaults or secret injection sidecars for runtime secret delivery

## 💾 Dynamic Storage Classes

### Current: Static local-path provisioner
### Production: Cloud-native with replication

**AWS EBS with Encryption:**
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-gp3-encrypted
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  encrypted: "true"
  kmsKeyId: "arn:aws:kms:region:account:key/key-id"
allowVolumeExpansion: true
```
*Requires: EBS CSI controller ServiceAccount with OIDC + IAM role having KMS permissions (kms:Decrypt, kms:GenerateDataKey, kms:CreateGrant)*

**AWS EFS for Shared Storage:**
```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: efs-shared
provisioner: efs.csi.aws.com
parameters:
  provisioningMode: efs-ap
  fileSystemId: fs-12345678
  directoryPerms: "0755"
volumeBindingMode: Immediate
```
*Supports: ReadWriteMany access mode for shared storage across multiple pods*

**Automated Backups:**
```yaml
apiVersion: velero.io/v1
kind: Schedule
metadata:
  name: redis-backup
spec:
  schedule: "0 1 * * *"
  template:
    storageLocation: aws-s3-backup
    ttl: 720h
```

## 🔒 TLS with Ingress

### Current: HTTP-only access
### Production: End-to-end TLS

**TLS Certificate Management:**
- **Corporate certificates**: Use company-issued certificates for internal domains
- **TLS secrets**: Create Kubernetes TLS secrets from certificate files (cert + private key)
- **Certificate rotation**: Implement automated certificate renewal and secret updates

**HTTPS Ingress Configuration:**
- **TLS termination**: NGINX controller handles HTTPS termination using TLS secrets
- **Certificate management**: Reference TLS secrets in Ingress for automatic HTTPS
- **Security headers**: Configure secure TLS protocols and cipher suites in NGINX

**Internal Service Communication:**
- **Service mesh mTLS**: Deploy Istio/Linkerd for automatic mutual TLS between all services with certificate rotation
- **Manual TLS**: Configure Redis and clients with TLS certificates for encrypted communication  
- **Certificate distribution**: Use mounted secrets for service certificates and private keys

## 📊 Observability

### Current: Basic kubectl logs
### Production: Full observability stack

**ELK for Log Aggregation:**
```yaml
# Fluent Bit → Elasticsearch → Kibana
apiVersion: elasticsearch.k8s.elastic.co/v1
kind: Elasticsearch
metadata:
  name: elasticsearch-cluster
spec:
  version: 8.6.0
  nodeSets:
  - name: master
    count: 3
    volumeClaimTemplates:
    - metadata:
        name: elasticsearch-data
      spec:
        storageClassName: ebs-gp3-encrypted
        resources:
          requests:
            storage: 100Gi
```

**Prometheus + Grafana + Mimir:**
```yaml
apiVersion: monitoring.coreos.com/v1
kind: Prometheus
metadata:
  name: prometheus-main
spec:
  retention: 15d
  remoteWrite:
  - url: "http://mimir-nginx.mimir.svc.cluster.local/api/v1/push"
```

**Distributed Tracing:**
```yaml
apiVersion: jaegertracing.io/v1
kind: Jaeger
metadata:
  name: jaeger-production
spec:
  strategy: production
  storage:
    type: elasticsearch
```

## 🛡️ Security Hardening

### Current: Default security context
### Production: Zero-trust security

**Pod Security Standards (Restricted):**
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: default
  labels:
    pod-security.kubernetes.io/enforce: restricted

# StatefulSet with hardened security
spec:
  template:
    spec:
      securityContext:
        runAsNonRoot: true
        runAsUser: 1001
        fsGroup: 1001
      containers:
      - name: redis
        securityContext:
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities:
            drop: [ALL]
```

**Network Policies (Whitelist-Only):**
```yaml
# Default deny-all
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
spec:
  podSelector: {}
  policyTypes: [Ingress, Egress]

# Specific Redis access
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: redis-allow
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/name: redis-stateful
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: approved-client
    ports:
    - port: 6379
```

**Least Privilege RBAC:**
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: redis-minimal
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]
  resourceNames: ["redis-stateful-0"]
```

**CVE Scanning in CI:**
```yaml
# GitHub Actions security pipeline
- name: Scan with Trivy
  run: |
    docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
      aquasec/trivy:latest image \
      --exit-code 1 \
      --severity HIGH,CRITICAL \
      sergeisin/redis-stateful:${{ github.sha }}

- name: Sign image
  run: cosign sign --yes sergeisin/redis-stateful:${{ github.sha }}
```

**Runtime Security:**
```yaml
# Falco for threat detection
apiVersion: v1
kind: ConfigMap
metadata:
  name: falco-rules
data:
  custom_rules.yaml: |
    - rule: Redis Unauthorized Access
      condition: container.image.tag contains "redis" and spawned_process
      output: Unauthorized Redis access (user=%user.name command=%proc.cmdline)
      priority: WARNING
```

**Encrypted Backup Verification:**
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: backup-verification
spec:
  schedule: "0 4 * * 0"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: verify
            command: ["/bin/sh", "-c", "velero backup get --selector backup-type=production"]
```

## 📈 Implementation Phases

**Phase 1**: Security foundation (Pod Security, Network Policies, RBAC)
**Phase 2**: Observability stack (ELK, Prometheus, Jaeger)  
**Phase 3**: Advanced security (Service mesh, runtime monitoring)
**Phase 4**: Operational excellence (Chaos engineering, cost optimization)

## 🧪 Testing the Pipeline

### Trigger CI/CD Pipeline

```bash
# Make a change to trigger the pipeline
echo "# Updated $(date)" >> README.md
git add README.md
git commit -m "Trigger CI/CD pipeline"
git push
```

### Monitor Pipeline

1. **GitHub Actions**: Check workflow execution in GitHub Actions tab
2. **ArgoCD**: Monitor sync status in ArgoCD UI
3. **Kubernetes**: Watch pod deployments: `kubectl get pods -w`

### Verify Deployment

```bash
# Check application status
kubectl get applications -n argocd
kubectl get pods -l app.kubernetes.io/name=redis-stateful
kubectl get pvc

# Test connectivity
redis-cli -h localhost -p 6379 info server
```

## 📊 Key Metrics

### Pipeline Performance
- **Build Time**: ~2-3 minutes average
- **Deployment Time**: ~30 seconds for sync
- **Total Lead Time**: ~3-4 minutes from commit to deployment

### Resource Usage
- **Redis Pod**: 250m CPU, 256Mi memory (requests)
- **Storage**: 1Gi persistent volume per pod
- **Network**: NodePort 30379 mapped to localhost:6379

## 🔍 Troubleshooting

### Common Issues

1. **Pod Pending**: Check node resources and affinity rules
   ```bash
   # Check node labels
   kubectl get nodes --show-labels | grep ingress-ready
   
   # Add label if missing
   kubectl label node gitops-demo-control-plane ingress-ready=true
   
   # Or remove nodeSelector as fallback
   kubectl patch deployment ingress-nginx-controller -n ingress-nginx \
     --type='json' \
     -p='[{"op": "remove", "path": "/spec/template/spec/nodeSelector"}]'
   ```
2. **Image Pull Errors**: Verify DockerHub credentials and image tags
3. **Service Not Accessible**: Confirm KIND port mappings and service types
4. **ArgoCD Sync Issues**: Check Git connectivity and Helm chart syntax

### Debug Commands

```bash
# Check cluster status
kubectl cluster-info
kubectl get nodes -o wide

# Examine pod issues
kubectl describe pod <pod-name>
kubectl logs <pod-name>

# Service connectivity
kubectl get svc
kubectl get endpoints

# ArgoCD status
kubectl get applications -n argocd
kubectl describe application redis-stateful -n argocd
```

## 📚 Learning Resources

- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
- [Helm Documentation](https://helm.sh/docs/)
- [KIND Documentation](https://kind.sigs.k8s.io/)
- [NGINX Ingress Controller](https://kubernetes.github.io/ingress-nginx/)

## 🤝 Contributing

This is a demonstration project for Gitops. The setup showcases:

- Modern CI/CD practices with GitOps
- Kubernetes stateful workload management
- Infrastructure as Code with Helm
- Production-ready architectural patterns
- Comprehensive troubleshooting and operational procedures

---

**Built with ❤️ for GitOps Excellence**
