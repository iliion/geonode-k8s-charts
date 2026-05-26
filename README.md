# Helm Chart for Geonode

- [GeoWhat?](#Geonode)
- [Geonode-k8s](#geonode-k8s)

## Geonode

GeoNode is a geospatial content management system, a platform for the management and publication of geospatial data. It brings together mature and stable open-source software projects under a consistent and easy-to-use interface allowing non-specialized users to share data and create interactive maps.

You can find the Sourcecode and more information about geonode under:

- Homepage: https://geonode.org/
- Github: https://github.com/GeoNode/geonode
- Docs: https://docs.geonode.org


## Geonode-k8s

This repository provides a helm chart for **geonode** including additional services as:

- geoserver: source server for sharing geospatial data (https://geoserver.org/)
- rabbitmq: message broker (scalable)
- postgresql database cluster: cnpg  
- memcached (optional): as django cache (scalable)
- nginx: webserver to deliver static content (scalable)
- pycsw: CSW interface (scalable)


## Documentation

To get an overview of the available configuration check out the values [docs](helm-charts/web-gis/README.md).

## Install

## Prerequisites

- A Kubernetes cluster
- [Helm](https://helm.sh/)


## Install chart dependencies

Update helm dependencies via:

```bash
helm repo update
```

## Override desired values

Configure geonode installation. Use the [docs](helm-charts/web-gis/README.md) to understand the parameters.

```bash
vi dev-values.yaml
```

## Install chart

### Dry run first
```bash
helm upgrade --cleanup-on-fail \
  --install \
  --namespace rheticus-electric-towers \
  --values dev-values.yaml \
  --dry-run \
  geonode helm-charts/web-gis
```

### If dry run is clean, apply
```bash
helm upgrade --cleanup-on-fail \
  --install \
  --namespace rheticus-electric-towers \
  --values dev-values.yaml \
  geonode helm-charts/web-gis
```

---

# ⚡ QUICK REFERENCE GUIDE

## 🚀 Common Operations

### Initial Deployment (First Time)
```bash
# Option 1: Automated
chmod +x deploy.sh
./deploy.sh

# Option 2: Manual steps
helm repo update geonode
helm dependency update helm-charts/web-gis
helm upgrade --install -f dev-values.yaml geonode helm-charts/web-gis \
  -n rheticus-electric-towers
```

### Update Configuration
```bash
# Edit configuration
nano dev-values.yaml

# Validate
helm lint -f dev-values.yaml helm-charts/web-gis

# Dry-run
helm upgrade -f dev-values.yaml geonode helm-charts/web-gis \
  -n rheticus-electric-towers --dry-run

# Apply
helm upgrade -f dev-values.yaml geonode helm-charts/web-gis \
  -n rheticus-electric-towers
```

### Update Docker Image
```bash
# Edit dev-values.yaml
# geonode:
#   image:
#     tag: "5.1.0"  ← change version

helm upgrade -f dev-values.yaml geonode helm-helm-charts/web-gis \
  -n rheticus-electric-towers

# Watch rollout
kubectl rollout status statefulset/geonode-geonode \
  -n rheticus-electric-towers
```

### View Current Configuration
```bash
# Show values used in deployment
helm get values geonode -n rheticus-electric-towers

# Show all generated manifests
helm get manifest geonode -n rheticus-electric-towers | less

# Show specific resource
kubectl get configmap geonode-geonode-env -o yaml -n rheticus-electric-towers
```

### Troubleshoot Pods
```bash
# View pod events
kubectl describe pod geonode-geonode-0 -n rheticus-electric-towers

# Tail logs
kubectl logs -f pod/geonode-geonode-0 -c geonode \
  -n rheticus-electric-towers

# Exec into pod
kubectl exec -it pod/geonode-geonode-0 -c geonode \
  -n rheticus-electric-towers -- /bin/bash

# Check init job logs
kubectl logs job/geonode-geonode-init-db-job -f \
  -n rheticus-electric-towers
```

### Database Operations
```bash
# Connect to database
kubectl exec -it cluster-pg-1 -n rheticus-electric-towers \
  -- psql -U postgres -d geonode

# Backup database
kubectl exec cluster-pg-1 -n rheticus-electric-towers \
  -- pg_dump -U postgres geonode > geonode-backup.sql

# Restore database
kubectl exec -i cluster-pg-1 -n rheticus-electric-towers \
  -- psql -U postgres geonode < geonode-backup.sql

# Check replication status
kubectl exec -it cluster-pg-1 -n rheticus-electric-towers \
  -- psql -U postgres -c "SELECT * FROM pg_stat_replication;"
```

### Manage Secrets
```bash
# Update password in database
kubectl exec -it cluster-pg-1 -n rheticus-electric-towers \
  -- psql -U postgres \
  -c "ALTER USER geonode WITH PASSWORD 'new-password';"

# Update password in dev-values.yaml + redeploy
# Edit dev-values.yaml
# database.geonode.password: "new-password"

helm upgrade -f dev-values.yaml geonode helm-charts/web-gis \
  -n rheticus-electric-towers

# Generate new TLS certificate
openssl req -x509 -newkey rsa:4096 -days 365 \
  -keyout tls.key -out tls.crt \
  -subj "/CN=relect.planetek.it" -nodes

kubectl create secret tls geonode-tls-secret \
  --cert=tls.crt --key=tls.key \
  --dry-run=client -o yaml | kubectl apply -f -
```

### Check Health
```bash
# Pod status
kubectl get pods -n rheticus-electric-towers

# Pod details
kubectl get pods -n rheticus-electric-towers -o wide

# Resource usage
kubectl top pods -n rheticus-electric-towers

# Events
kubectl get events -n rheticus-electric-towers --sort-by='.lastTimestamp'

# Services and IPs
kubectl get svc -n rheticus-electric-towers

# Database cluster status
kubectl get postgresql cluster-pg -n rheticus-electric-towers -o wide

# APISIX routes
kubectl get apisixroute -n rheticus-electric-towers
```

### Scaling
```bash
# Scale GeoNode replicas
kubectl scale statefulset geonode-geonode \
  --replicas=3 -n rheticus-electric-towers

# Scale WebGIS replicas
kubectl scale deployment webgis \
  --replicas=3 -n rheticus-electric-towers

# Or edit dev-values.yaml and re-apply:
geonode:
  replicaCount: 3
webgis:
  replicaCount: 3

helm upgrade -f dev-values.yaml geonode helm-helm-charts/web-gis \
  -n rheticus-electric-towers
```

### Backup and Restore
```bash
# Backup Helm release
helm get values geonode -n rheticus-electric-towers > dev-values-backup.yaml
helm get manifest geonode -n rheticus-electric-towers > manifest-backup.yaml

# Backup database
kubectl exec cluster-pg-1 -n rheticus-electric-towers \
  -- pg_dump -U postgres -Fc geonode > db-backup.dump

# Restore from Helm backup
helm upgrade -f dev-values-backup.yaml geonode helm-charts/web-gis \
  -n rheticus-electric-towers --install

# Restore database
kubectl exec -i cluster-pg-1 -n rheticus-electric-towers \
  -- pg_restore -U postgres -d geonode < db-backup.dump
```

### Rollback
```bash
# Show release history
helm history geonode -n rheticus-electric-towers

# Rollback to previous version
helm rollback geonode 1 -n rheticus-electric-towers

# Rollback to specific version
helm rollback geonode 3 -n rheticus-electric-towers
```

### Cleanup
```bash
# Delete specific pod (triggers restart)
kubectl delete pod geonode-geonode-0 \
  -n rheticus-electric-towers

# Delete deployment (full restart)
kubectl rollout restart statefulset/geonode-geonode \
  -n rheticus-electric-towers

# Delete entire release (DESTRUCTIVE - backs up data via PVC)
helm uninstall geonode -n rheticus-electric-towers

# Delete namespace (deletes everything)
kubectl delete namespace rheticus-electric-towers
```

---

## 🔍 Monitoring Commands

```bash
# Watch pod creation in real-time
watch kubectl get pods -n rheticus-electric-towers

# Stream logs from all pods with label
kubectl logs -f -l release=geonode -n rheticus-electric-towers

# Check resource usage in real-time
watch kubectl top pods -n rheticus-electric-towers

# Get CPU/Memory requests vs usage
kubectl get pods -n rheticus-electric-towers -o custom-columns=\
NAME:.metadata.name,\
CPU_REQUEST:.spec.containers[*].resources.requests.cpu,\
CPU_LIMIT:.spec.containers[*].resources.limits.cpu,\
MEM_REQUEST:.spec.containers[*].resources.requests.memory,\
MEM_LIMIT:.spec.containers[*].resources.limits.memory

# List all events sorted by time
kubectl get events -n rheticus-electric-towers \
  --sort-by='.lastTimestamp' | tail -20
```

---

## 🛠️ Useful Aliases

Add to `.bashrc` or `.zshrc`:

```bash
alias k=kubectl
alias ns='kubectl config set-context --current --namespace'
alias kgp='kubectl get pods -n rheticus-electric-towers'
alias kgs='kubectl get svc -n rheticus-electric-towers'
alias klogs='kubectl logs -f -n rheticus-electric-towers'
alias kdesc='kubectl describe -n rheticus-electric-towers'
alias kexec='kubectl exec -it -n rheticus-electric-towers'
alias helm-dry='helm upgrade --dry-run --debug'
alias helm-status='helm status geonode -n rheticus-electric-towers'
alias helm-values='helm get values geonode -n rheticus-electric-towers'
```

Usage:
```bash
kgp                    # = kubectl get pods -n rheticus-electric-towers
klogs job/geonode-geonode-init-db-job
kexec pod/geonode-geonode-0 -c geonode -- /bin/bash
```

---

## 📊 Useful One-Liners

```bash
# Count pods by status
kubectl get pods -n rheticus-electric-towers -o json | \
  jq -r '.items[].status.phase' | sort | uniq -c

# Get pod IPs
kubectl get pods -n rheticus-electric-towers -o wide | \
  awk '{print $1, $6}'

# Check container restart count
kubectl get pods -n rheticus-electric-towers -o json | \
  jq '.items[] | {name: .metadata.name, restarts: .status.containerStatuses[].restartCount}'

# Find failed pods
kubectl get pods -n rheticus-electric-towers --field-selector=status.phase!=Running

# Get resource requests total
kubectl get pods -n rheticus-electric-towers -o json | \
  jq '[.items[].spec.containers[].resources.requests | to_entries | .[]] | group_by(.key) | map({(.[0].key): map(.value | tonumber) | add}) | add'

# Export all resources to YAML
kubectl get all -n rheticus-electric-towers -o yaml > cluster-state.yaml

# Check certificate expiry
kubectl get secret geonode-tls-secret -n rheticus-electric-towers -o json | \
  jq -r '.data."tls.crt" | @base64d' | \
  openssl x509 -noout -enddate
```

---

## 🆘 Emergency Procedures

### Pod in CrashLoopBackOff
```bash
# 1. Check logs for error
kubectl logs pod/geonode-geonode-0 -n rheticus-electric-towers -c geonode

# 2. Check resource limits (might be too low)
kubectl describe pod geonode-geonode-0 -n rheticus-electric-towers | grep -A 5 "Limits\|Requests"

# 3. Check init containers
kubectl logs pod/geonode-geonode-0 -n rheticus-electric-towers --previous

# 4. Check init-db job status
kubectl get job geonode-geonode-init-db-job -n rheticus-electric-towers
kubectl logs job/geonode-geonode-init-db-job -n rheticus-electric-towers

# 5. Manual restart
kubectl rollout restart statefulset/geonode-geonode -n rheticus-electric-towers

# 6. If still failing, scale down and inspect manually
kubectl scale statefulset geonode-geonode --replicas=0 -n rheticus-electric-towers
kubectl scale statefulset geonode-geonode --replicas=1 -n rheticus-electric-towers
```

### Database Cluster Unhealthy
```bash
# Check cluster status
kubectl get postgresql cluster-pg -n rheticus-electric-towers

# Check pod logs
kubectl logs pod/cluster-pg-1 -n rheticus-electric-towers

# Force failover (if primary is stuck)
kubectl set env cluster cluster-pg PGPASSWORD=<password> \
  -n rheticus-electric-towers

# If completely broken, might need to recreate
# (data is on PVC, so survives deletion)
helm uninstall geonode -n rheticus-electric-towers
helm install geonode helm-charts/web-gis -f dev-values.yaml \
  -n rheticus-electric-towers
```

### Out of Disk Space
```bash
# Check PVC usage
kubectl get pvc -n rheticus-electric-towers

# Check pod disk usage
kubectl exec pod/geonode-geonode-0 -n rheticus-electric-towers \
  -c geonode -- df -h

# Expand PVC (if supported by StorageClass)
kubectl patch pvc pvc-geonode-geonode -n rheticus-electric-towers \
  -p '{"spec":{"resources":{"requests":{"storage":"20Gi"}}}}'

# Clean up old files (if safe)
kubectl exec pod/geonode-geonode-0 -n rheticus-electric-towers \
  -c geonode -- rm -rf /mnt/volumes/statics/temp/*
```

### Network Issues
```bash
# Test connectivity from pod
kubectl exec pod/geonode-geonode-0 -n rheticus-electric-towers \
  -c geonode -- curl -v https://relect.planetek.it

# Test DNS resolution
kubectl exec pod/geonode-geonode-0 -n rheticus-electric-towers \
  -c geonode -- nslookup relect.planetek.it

# Check service endpoints
kubectl get endpoints geonode-nginx -n rheticus-electric-towers

# Test direct pod connectivity
kubectl run -it --image=busybox debug --rm -- /bin/sh
> nc -vz geonode-geonode 8000
> wget -O- http://geonode-geonode:8000/
```

---

## 📈 Performance Tuning

```bash
# Monitor real-time resource usage
kubectl top pods -n rheticus-electric-towers --containers

# Get container resource limits
kubectl get pods -n rheticus-electric-towers -o json | \
  jq '.items[] | {pod: .metadata.name, limits: .spec.containers[].resources.limits}'

# Check if pods are hitting CPU limits
kubectl describe pod geonode-geonode-0 -n rheticus-electric-towers | grep -A 3 "cpu"

# Scale up resources
# Edit dev-values.yaml
geonode:
  resources:
    limits:
      cpu: "4000m"     # increase
      memory: "4Gi"    # increase
    requests:
      cpu: "2000m"
      memory: "2Gi"

helm upgrade -f dev-values.yaml geonode helm-helm-charts/web-gis \
  -n rheticus-electric-towers
```