# Kubernetes WordPress Deployment

This project demonstrates how to deploy a production-ready WordPress site with a MySQL database on Kubernetes using persistent volumes. This setup ensures data persistence across pod restarts and provides a scalable, resilient architecture for your WordPress application.

**Reference:** [Official Kubernetes Tutorial](https://kubernetes.io/docs/tutorials/stateful-application/mysql-wordpress-persistent-volume/)

---

## Quick Start

For experienced users, here's the TL;DR:

```bash
# Download manifests
curl -LO https://k8s.io/examples/application/wordpress/mysql-deployment.yaml
curl -LO https://k8s.io/examples/application/wordpress/wordpress-deployment.yaml

# Create kustomization.yaml
vi kustomization.yaml
# Add: secretGenerator, resources (see Step 3)

# Deploy
kubectl apply -k ./

# Get WordPress URL (Minikube)
minikube service wordpress --url
```

---

## Table of Contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Setup Instructions](#setup-instructions)
- [Deployment](#deployment)
- [Verification](#verification)
- [Accessing WordPress](#accessing-wordpress)
- [Architecture Components](#architecture-components)
- [Troubleshooting](#troubleshooting)
- [Cleanup](#cleanup)
- [Next Steps](#next-steps)

---

## Overview

This deployment includes:

| Component | Description |
|-----------|-------------|
| **MySQL 8.0** | Relational database with 20Gi persistent storage |
| **WordPress 6.2.1** | Apache-based web server with 20Gi persistent storage |
| **PersistentVolumeClaims** | Ensures data persistence across pod lifecycle |
| **Kubernetes Services** | LoadBalancer for WordPress, headless service for MySQL |
| **Kustomize** | Declarative configuration management |
| **Secrets** | Secure password management via Kubernetes Secrets |

## Prerequisites

Before you begin, ensure you have:

- **Kubernetes Cluster**: Running cluster (minikube, kind, GKE, EKS, or AKS)
- **kubectl**: CLI tool installed and configured to access your cluster
- **Basic Knowledge**: Understanding of Pods, Services, Deployments, and PersistentVolumes

### Verify Your Environment

```bash
# Check kubectl is installed and cluster is accessible
kubectl version 
kubectl cluster-info

# Verify you have a default storage class
kubectl get storageclass
```

## Setup Instructions

### Step 1: Download Configuration Files

Download the MySQL and WordPress deployment manifests:

```bash
curl -LO https://k8s.io/examples/application/wordpress/mysql-deployment.yaml
curl -LO https://k8s.io/examples/application/wordpress/wordpress-deployment.yaml
```

### Step 2: Review the Configuration Files

The downloaded files contain three Kubernetes resources each:

#### MySQL Deployment Configuration

**File:** `mysql-deployment.yaml`

This file defines three resources:
1. **Headless Service** - Internal DNS for database connectivity
2. **PersistentVolumeClaim** - 20Gi storage for database files
3. **Deployment** - MySQL 8.0 container with environment variables

<details>
<summary>Click to view mysql-deployment.yaml</summary>

```yaml
apiVersion: v1
kind: Service
metadata:
  name: wordpress-mysql
  labels:
    app: wordpress
spec:
  ports:
    - port: 3306
  selector:
    app: wordpress
    tier: mysql
  clusterIP: None
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pv-claim
  labels:
    app: wordpress
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress-mysql
  labels:
    app: wordpress
spec:
  selector:
    matchLabels:
      app: wordpress
      tier: mysql
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: wordpress
        tier: mysql
    spec:
      containers:
      - image: mysql:8.0
        name: mysql
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-pass
              key: password
        - name: MYSQL_DATABASE
          value: wordpress
        - name: MYSQL_USER
          value: wordpress
        - name: MYSQL_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-pass
              key: password
        ports:
        - containerPort: 3306
          name: mysql
        volumeMounts:
        - name: mysql-persistent-storage
          mountPath: /var/lib/mysql
      volumes:
      - name: mysql-persistent-storage
        persistentVolumeClaim:
          claimName: mysql-pv-claim

```

</details>

#### WordPress Deployment Configuration

**File:** `wordpress-deployment.yaml`

This file defines three resources:
1. **LoadBalancer Service** - External access to WordPress (port 80)
2. **PersistentVolumeClaim** - 20Gi storage for WordPress files
3. **Deployment** - WordPress 6.2.1-apache container

<details>
<summary>Click to view wordpress-deployment.yaml</summary>

```yaml
apiVersion: v1
kind: Service
metadata:
  name: wordpress
  labels:
    app: wordpress
spec:
  ports:
    - port: 80
  selector:
    app: wordpress
    tier: frontend
  type: LoadBalancer
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: wp-pv-claim
  labels:
    app: wordpress
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wordpress
  labels:
    app: wordpress
spec:
  selector:
    matchLabels:
      app: wordpress
      tier: frontend
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: wordpress
        tier: frontend
    spec:
      containers:
      - image: wordpress:6.2.1-apache
        name: wordpress
        env:
        - name: WORDPRESS_DB_HOST
          value: wordpress-mysql
        - name: WORDPRESS_DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-pass
              key: password
        - name: WORDPRESS_DB_USER
          value: wordpress
        ports:
        - containerPort: 80
          name: wordpress
        volumeMounts:
        - name: wordpress-persistent-storage
          mountPath: /var/www/html
      volumes:
      - name: wordpress-persistent-storage
        persistentVolumeClaim:
          claimName: wp-pv-claim

```

</details>

---

### Step 3: Create Kustomization File

Create a `kustomization.yaml` file in the same directory:

```bash
vi kustomization.yaml
```

Add the following content:

```yaml
secretGenerator:
- name: mysql-pass
  literals:
  - password=YOUR_PASSWORD
resources:
- mysql-deployment.yaml
- wordpress-deployment.yaml
```

**Important:** Replace `YOUR_PASSWORD` with a strong, secure password of your choice.

Save and exit the editor (`:wq` in vi/vim, or use `nano`, `code`, etc.).

---

## Deployment

### Step 4: Deploy to Kubernetes

Apply all configurations using Kustomize. This single command will:
- Generate the MySQL password secret
- Create services for WordPress and MySQL
- Provision persistent volumes
- Deploy WordPress and MySQL pods

```bash
kubectl apply -k ./
```

Expected output:

```
kubectl apply -k ./
secret/mysql-pass-5m26tmdb5k unchanged
service/wordpress created
Warning: spec.SessionAffinity is ignored for headless services
service/wordpress-mysql created
persistentvolumeclaim/mysql-pv-claim created
persistentvolumeclaim/wp-pv-claim created
deployment.apps/wordpress created
deployment.apps/wordpress-mysql created
```

> **Note:** The warning about `spec.SessionAffinity` for headless services is expected and can be ignored.

---

## Verification

After deployment, verify that all components are running correctly:

### Verify Secrets

Check that the MySQL password secret was created:

```bash
kubectl get secrets
```

Expected output:

```
kubectl get secrets
NAME                    TYPE     DATA   AGE
mysql-pass-5m26tmdb5k   Opaque   1      17s
```

The secret name includes a hash suffix generated by Kustomize for uniqueness.

### Verify Persistent Volume Claims

Check that PersistentVolumeClaims were created and bound (STATUS should be `Bound`):

```bash
kubectl get pvc
```

Expected output:

```
kubectl get pvc
NAME             STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
mysql-pv-claim   Bound    pvc-633cec0c-3e5b-4066-818f-bc1d612e417f   20Gi       RWO            standard       <unset>                 57s
wp-pv-claim      Bound    pvc-2b355104-2c99-42e6-9c3b-4f1be84fef38   20Gi       RWO            standard       <unset>                 57s

```

### Verify Pods

Check that the WordPress and MySQL pods are running:

```bash
kubectl get pods
```

Expected output:

```
kubectl get pods
NAME                              READY   STATUS    RESTARTS     AGE
wordpress-6c5888f6dd-xxxxx        1/1     Running   0            115s
wordpress-mysql-6cb8644b8-4bnlr   1/1     Running   0            115s
```

Wait until both pods show `STATUS: Running` and `READY: 1/1`.

### Verify Services

Check the WordPress service:

```bash
kubectl get services wordpress
```

Expected output:

```
kubectl get services wordpress
NAME        TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
wordpress   LoadBalancer   10.104.219.205   <pending>     80:30525/TCP   2m59s
```

---

## Accessing WordPress

### Get the Service URL

If you're using **Minikube**, get the WordPress URL:

```bash
minikube service wordpress --url
```

Example output:

```
http://192.168.49.2:30525
```

Visit this URL in your browser to access your WordPress installation.

### For Cloud Providers

If using a cloud provider (GKE, EKS, AKS), wait for the `EXTERNAL-IP` to be assigned:

```bash
kubectl get service wordpress --watch
```

Once the external IP appears, access WordPress at `http://<EXTERNAL-IP>`.

---

## Architecture Components

This deployment creates the following Kubernetes resources:

### MySQL Database
- **Image:** mysql:8.0
- **Storage:** 20Gi persistent volume
- **Service Type:** Headless ClusterIP (internal only)
- **Database Name:** wordpress
- **Credentials:** Stored in Kubernetes Secret

### WordPress Frontend
- **Image:** wordpress:6.2.1-apache
- **Storage:** 20Gi persistent volume for uploads and themes
- **Service Type:** LoadBalancer (externally accessible)
- **Port:** 80

### Data Flow

```
Internet → LoadBalancer Service (wordpress)
    ↓
WordPress Pod (wordpress:6.2.1-apache)
    ↓
    ├─→ PVC (wp-pv-claim) → 20Gi Volume (/var/www/html)
    └─→ Headless Service (wordpress-mysql)
            ↓
        MySQL Pod (mysql:8.0)
            ↓
            ├─→ PVC (mysql-pv-claim) → 20Gi Volume (/var/lib/mysql)
            └─→ Secret (mysql-pass) → Database credentials
```

---

## Troubleshooting

Common issues and their solutions:

### Pods Not Starting

Check pod status and logs:

```bash
kubectl get pods
kubectl describe pod <pod-name>
kubectl logs <pod-name>
```

### PVC Not Binding

Check PVC status:

```bash
kubectl describe pvc mysql-pv-claim
kubectl describe pvc wp-pv-claim
```

### Service Not Accessible

For Minikube users, ensure the tunnel is running:

```bash
minikube tunnel
```
```
minikube tunnel
Status:	
	machine: minikube
	pid: 677795
	route: 10.96.0.0/12 -> 192.168.49.2
	minikube: Running
	services: [wordpress]
    errors: 
		minikube: no errors
		router: no errors
		loadbalancer emulator: no errors
```

### Connection Refused

If WordPress cannot connect to MySQL:

```bash
# Check if MySQL pod is ready
kubectl get pods -l tier=mysql

# Check MySQL logs
kubectl logs deployment/wordpress-mysql

# Check WordPress logs for connection errors
kubectl logs deployment/wordpress

# Verify MySQL service exists
kubectl get service wordpress-mysql

# Test database connectivity from WordPress pod
# If you see "Connected to wordpress-mysql" - the connection works!
# Exit code 124 (timeout) is expected and means success
kubectl exec deployment/wordpress -- timeout 2 curl -v telnet://wordpress-mysql:3306
```


### Storage Issues

If PVCs remain in `Pending` state:

```bash
# Check storage class
kubectl get storageclass

# Describe PVC for more details
kubectl describe pvc mysql-pv-claim

# For Minikube, ensure storage provisioner is enabled
minikube addons enable storage-provisioner
```

---

## Cleanup

To remove all resources created by this deployment:

```bash
kubectl delete -k ./
```

This will delete:
- WordPress and MySQL deployments
- Services
- PersistentVolumeClaims (and associated data)
- Secrets

**Warning:** This will permanently delete all WordPress data and database content.

### Verify Cleanup

Ensure all resources are removed:

```bash
kubectl get all,pvc,secrets -l app=wordpress
```

---

## Next Steps

After successfully deploying WordPress, consider these enhancements:

### 1. Complete WordPress Setup
Access your WordPress URL and follow the installation wizard:
- Select language
- Create admin account
- Configure site title and description

### 2. Production Hardening
```bash
# Scale WordPress for high availability
kubectl scale deployment wordpress --replicas=3

# Set resource limits
kubectl set resources deployment wordpress --limits=cpu=500m,memory=512Mi
kubectl set resources deployment wordpress-mysql --limits=cpu=1000m,memory=1Gi
```

### 3. Enable HTTPS
```bash
# Install cert-manager
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.0/cert-manager.yaml

# Configure Ingress with TLS (requires ingress controller)
# See: https://kubernetes.io/docs/concepts/services-networking/ingress/
```

### 4. Backup Strategy
```bash
# Backup WordPress files
kubectl exec deployment/wordpress -- tar czf /tmp/wp-backup.tar.gz /var/www/html

# Backup MySQL database
kubectl exec deployment/wordpress-mysql -- mysqldump -u wordpress -p wordpress > backup.sql
```

### 5. Monitoring and Logging
- Set up Prometheus for metrics
- Configure Grafana dashboards
- Enable centralized logging with ELK or Loki

---

## Additional Resources

- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [WordPress on Docker Hub](https://hub.docker.com/_/wordpress)
- [MySQL on Docker Hub](https://hub.docker.com/_/mysql)
- [Kustomize Documentation](https://kustomize.io/)
- [Kubernetes Storage Classes](https://kubernetes.io/docs/concepts/storage/storage-classes/)

## Contributing

Contributions are welcome! Please feel free to submit issues or pull requests.

## License

This configuration is based on the official Kubernetes examples and follows the same licensing.
