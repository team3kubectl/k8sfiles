# MongoDB Replication + NFS Storage Setup Guide

## Overview
This setup configures:
1. **MongoDB Replication** - 3-node replica set for high availability
2. **NFS CSI Driver** - Network File System storage for persistent data
3. **Automatic Replica Initialization** - Job to initialize the replica set

---

## Prerequisites
- Kubernetes cluster running
- kubectl configured
- NFS server available on your network

---

## Step-by-Step Setup

### 1. Configure NFS Server
Before deploying, update the NFS StorageClass with your NFS server details:

```bash
# Edit nfs-storageclass.yaml and replace:
# - NFS_SERVER_IP: Your NFS server IP (e.g., 192.168.1.50)
# - /mnt/nfs: Your NFS share path (e.g., /exports/mongodb)
```

### 2. Install NFS CSI Driver (Global)
```bash
kubectl apply -f k8s/storage/nfs-csi-driver.yaml
```

Wait for the provisioner deployment to be ready:
```bash
kubectl rollout status deployment/nfs-csi-provisioner -n kube-system
```

### 3. Create NFS StorageClass
```bash
kubectl apply -f k8s/storage/nfs-storageclass.yaml
```

Verify:
```bash
kubectl get storageclass
```

### 4. Deploy/Update MongoDB StatefulSet
```bash
# Stop old StatefulSet if exists
kubectl delete statefulset events-mongodb -n team3-ns --ignore-not-found

# Deploy new StatefulSet with NFS storage
kubectl apply -f k8s/mongodb/mongo-statefulset-updated.yaml
```

Wait for all 3 pods to be ready:
```bash
kubectl get pods -n team3-ns -l app=mongodb -w
```

### 5. Initialize Replica Set
```bash
kubectl apply -f k8s/mongodb/mongo-replica-init-job.yaml
```

Check if initialization succeeded:
```bash
kubectl logs -n team3-ns job/mongo-replica-init
```

---

## Verify Replica Set Status

### Check replica set is initialized:
```bash
kubectl exec -it events-mongodb-0 -n team3-ns -- mongosh --authenticationDatabase admin -u admin -p adminpass --eval "rs.status()"
```

### View replica set configuration:
```bash
kubectl exec -it events-mongodb-0 -n team3-ns -- mongosh --authenticationDatabase admin -u admin -p adminpass --eval "rs.conf()"
```

### Check PersistentVolume claims:
```bash
kubectl get pvc -n team3-ns
```

Expected output should show 3 PVCs for mongo-data.

---

## MongoDB Replication Features ✓

- **3-node Replica Set (rs0)**: Automatic failover & high availability
- **Pod Anti-Affinity**: Replicas spread across different nodes
- **Liveness/Readiness Probes**: Health checks for pod orchestration
- **Persistent NFS Storage**: Data survives pod restarts
- **Authentication Enabled**: Credentials in secret

---

## Common Commands

### Scale replica set (if needed):
```bash
kubectl scale statefulset events-mongodb -n team3-ns --replicas=5
```

### Connect to MongoDB:
```bash
kubectl port-forward -n team3-ns svc/events-mongodb 27017:27017
# From another terminal:
mongosh mongodb://admin:adminpass@localhost:27017/?authSource=admin
```

### View MongoDB logs:
```bash
kubectl logs -n team3-ns events-mongodb-0 -f
```

### Re-initialize replica set (if needed):
```bash
kubectl delete job mongo-replica-init -n team3-ns
kubectl apply -f k8s/mongodb/mongo-replica-init-job.yaml
```

---

## Troubleshooting

### Pods stuck in Pending state:
- Check if PVCs are created: `kubectl get pvc -n team3-ns`
- Check NFS connectivity and StorageClass is working

### Replica set initialization failed:
- Check job logs: `kubectl logs -n team3-ns job/mongo-replica-init`
- Ensure all 3 MongoDB pods are Running
- Re-run the initialization job

### Cannot connect from services:
- Verify namespace is `team3-ns`
- Check connection string uses: `mongodb://admin:password@events-mongodb:27017`
- Ensure services in same namespace

---

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                    team3-ns Namespace                       │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐       │
│  │ events-      │  │ events-      │  │ events-      │       │
│  │ mongodb-0    │  │ mongodb-1    │  │ mongodb-2    │       │
│  │ (Primary)    │  │ (Secondary)  │  │ (Secondary)  │       │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘       │
│         │                  │                  │               │
│  ┌──────▼──────────────────▼──────────────────▼──────┐       │
│  │     ReplicaSet Service: events-mongodb            │       │
│  │     Headless | Port: 27017                        │       │
│  └────────────────────────────────────────────────────┘       │
│         │                  │                  │               │
│  ┌──────▼──────┐  ┌──────▼────────┐  ┌──────▼────────┐     │
│  │ NFS-PVC-0   │  │ NFS-PVC-1     │  │ NFS-PVC-2    │     │
│  │ (/data/db)  │  │ (/data/db)    │  │ (/data/db)   │     │
│  └─────────────┘  └───────────────┘  └──────────────┘     │
│         │                  │                  │               │
└─────────┼──────────────────┼──────────────────┼───────────────┘
          │                  │                  │
          └──────────────────┼──────────────────┘
                  (NFS CSI Driver)
                       │
                ┌──────▼────────┐
                │  NFS Server   │
                │  (Persistent) │
                └───────────────┘
```

---

## Files Created

- `nfs-storageclass.yaml` - NFS storage provisioner
- `nfs-csi-driver.yaml` - NFS CSI driver deployment
- `mongo-statefulset-updated.yaml` - Updated MongoDB with NFS storage + replica set config
- `mongo-replica-init-job.yaml` - Automatic replica set initialization
- `README-SETUP.md` - This guide
