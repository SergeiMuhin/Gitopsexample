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
    paths: ['README.md', 'app/**']

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

Configure in GitHub repository settings: (Dockerhub image repository not your github password)
- `DOCKERHUB_USERNAME`: Your DockerHub username
- `DOCKERHUB_TOKEN`: DockerHub access token

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

## 🌐 Service Accessibility

### Design Choice: NodePort

**Selected**: NodePort for external access

**Rationale**:
- **Simplicity**: Direct port mapping without additional complexity
- **Reliability**: No dependency on external load balancers
- **Development**: Perfect for local KIND clusters
- **Demonstration**: Clear traffic flow for interview purposes

**Alternative Considered**: NGINX Ingress TCP forwarding
- More complex setup
- Additional failure points
- Requires understanding of Layer 4 vs Layer 7 routing

### Access Methods

1. **External Access**: `localhost:6379` via NodePort
2. **Internal Access**: `redis-stateful.default.svc.cluster.local:6379`
3. **Pod-to-Pod**: `redis-0.redis-stateful-headless.default.svc.cluster.local:6379`

## 🔧 Technical Decisions

### Storage Class
**Choice**: Default local-path provisioner (rancher.io/local-path)
- **Pros**: Works out-of-box with KIND, suitable for development
- **Cons**: Data tied to specific node, no replication
- **Production Alternative**: Network-attached storage (EBS, Ceph, etc.)

### Container Registry
**Choice**: DockerHub public registry
- **Pros**: Free, widely supported, simple integration
- **Cons**: Public images, rate limiting
- **Production Alternative**: Private registries (ECR, GCR, Harbor)

### Networking
**Choice**: NodePort with KIND port mapping
- **Pros**: Simple, direct access, no external dependencies
- **Cons**: Limited port range, not production-typical
- **Production Alternative**: LoadBalancer with cloud provider integration

### Node Affinity Handling
**Choice**: Label-based node selection with fallback patch
- **Primary**: Add `ingress-ready=true` label to control-plane node
- **Fallback**: Remove nodeSelector if labeling fails
- **Rationale**: KIND clusters need explicit node targeting for ingress controllers

## 🚀 Production Readiness Extensions

### Secret Management
- **Current**: Plain text configuration
- **Production**: 
  - Sealed Secrets for GitOps-compatible secret management
  - External Secrets Operator for cloud secret integration
  - Vault for comprehensive secret management

### Dynamic Storage Classes
- **Current**: Static local-path provisioner
- **Production**:
  - CSI drivers for cloud storage (EBS CSI, Azure Disk CSI)
  - Network storage solutions (Ceph, Longhorn)
  - Backup and disaster recovery strategies

### TLS with Ingress
- **Current**: HTTP-only access
- **Production**:
  - cert-manager for automatic certificate management
  - Let's Encrypt integration for public certificates
  - Private CA for internal services

### Observability
- **Current**: Basic Kubernetes logs
- **Production**:
  - Prometheus + Grafana for metrics and alerting
  - ELK/EFK stack for centralized logging
  - Jaeger for distributed tracing
  - Service mesh (Istio/Linkerd) for advanced observability

### Security Hardening
- **Current**: Default security context
- **Production**:
  - Pod Security Standards (restricted profile)
  - Network Policies for traffic segmentation
  - RBAC with principle of least privilege
  - Image scanning and vulnerability management
  - Runtime security monitoring

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

This is a demonstration project for Gitops Example purposes. The setup showcases:

- Modern CI/CD practices with GitOps
- Kubernetes stateful workload management
- Infrastructure as Code with Helm
- Production-ready architectural patterns
- Comprehensive troubleshooting and operational procedures

---

**Built with ❤️ for GitOps Excellence**
