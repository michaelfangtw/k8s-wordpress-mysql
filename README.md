# 🚀 Kubernetes WordPress Deployment

This project will help you deploy WordPress and MySQL on Kubernetes. Your data will not lost when pod restart because we use persistent storage.

**📚 Reference:** [Official Kubernetes Tutorial](https://kubernetes.io/docs/tutorials/stateful-application/mysql-wordpress-persistent-volume/)

---

## ⚡ Quick Start

If you already know Kubernetes, just do this:

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

## 📑 Table of Contents

- [📖 Overview](#overview)
- [✅ Prerequisites](#prerequisites)
- [🔧 Setup Instructions](#setup-instructions)
- [🚢 Deployment](#deployment)
- [✔️ Verification](#verification)
- [🌐 Accessing WordPress](#accessing-wordpress)
- [🏗️ Architecture Components](#architecture-components)
- [🔍 Troubleshooting](#troubleshooting)
- [🧹 Cleanup](#cleanup)
- [🎯 Next Steps](#next-steps)

---

## 📖 Overview

What this project have:

| Component | Description |
|-----------|-------------|
| **🗄️ MySQL 8.0** | Database with 20Gi storage |
| **📝 WordPress 6.2.1** | WordPress with Apache, 20Gi storage |
| **💾 PersistentVolumeClaims** | Keep your data safe when pod restart |
| **🔗 Kubernetes Services** | LoadBalancer for WordPress, ClusterIP for MySQL |
| **⚙️ Kustomize** | Help us manage configuration easy |
| **🔐 Secrets** | Keep password safe in Kubernetes |

## ✅ Prerequisites

Before start, you need:

- **Kubernetes Cluster**: You need running cluster (minikube, kind, GKE, EKS, or AKS)
- **kubectl**: Install kubectl and connect to your cluster
- **Basic Knowledge**: You should know about Pods, Services, Deployments, and PersistentVolumes

### Check Your Environment

```bash
# Check kubectl is installed and cluster is accessible
kubectl version 
kubectl cluster-info

# Verify you have a default storage class
kubectl get storageclass
```

## 🔧 Setup Instructions

### Step 1: 📥 Download Configuration Files

First, download the MySQL and WordPress files:

```bash
curl -LO https://k8s.io/examples/application/wordpress/mysql-deployment.yaml
curl -LO https://k8s.io/examples/application/wordpress/wordpress-deployment.yaml
```

### Step 2: 👀 Review the Configuration Files

Each file have three Kubernetes resources:

#### 🗄️ MySQL Deployment Configuration

**File:** `mysql-deployment.yaml`

This file have three things:
1. **🔗 Headless Service** - For internal database connection
2. **💾 PersistentVolumeClaim** - 20Gi storage for database
3. **🚀 Deployment** - MySQL 8.0 container

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

#### 📝 WordPress Deployment Configuration

**File:** `wordpress-deployment.yaml`

This file also have three things:
1. **🌐 LoadBalancer Service** - So we can access WordPress from internet (port 80)
2. **💾 PersistentVolumeClaim** - 20Gi storage for WordPress files
3. **🚀 Deployment** - WordPress 6.2.1-apache container

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

### Step 3: 📝 Create Kustomization File

Now we create `kustomization.yaml` file:

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

**⚠️ Important:** Change `YOUR_PASSWORD` to your own password. Use strong password!

Save and exit (type `:wq` in vi/vim, or you can use `nano`, `code`).

---

## 🚢 Deployment

### Step 4: 🎯 Deploy to Kubernetes

Now we deploy everything! This command will do many things:
- Create the MySQL password secret
- Create services for WordPress and MySQL
- Create persistent volumes
- Deploy WordPress and MySQL pods

```bash
kubectl apply -k ./
```

You will see output like this:

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

> **💡 Note:** The warning about `spec.SessionAffinity` is normal. Don't worry about it.

---

## ✔️ Verification

Now we check if everything is working!

### 🔐 Check Secrets

Check if the MySQL password secret is created:

```bash
kubectl get secrets
```

You should see:

```
kubectl get secrets
NAME                    TYPE     DATA   AGE
mysql-pass-5m26tmdb5k   Opaque   1      17s
```

The secret name have random letters at end. Kustomize add this automatically.

### 💾 Check Persistent Volume Claims

Check if PersistentVolumeClaims is created and bound (STATUS must be `Bound`):

```bash
kubectl get pvc
```

You should see:

```
kubectl get pvc
NAME             STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   VOLUMEATTRIBUTESCLASS   AGE
mysql-pv-claim   Bound    pvc-633cec0c-3e5b-4066-818f-bc1d612e417f   20Gi       RWO            standard       <unset>                 57s
wp-pv-claim      Bound    pvc-2b355104-2c99-42e6-9c3b-4f1be84fef38   20Gi       RWO            standard       <unset>                 57s

```

### 🚀 Check Pods

Check if WordPress and MySQL pods is running:

```bash
kubectl get pods
```

You should see:

```
kubectl get pods
NAME                              READY   STATUS    RESTARTS     AGE
wordpress-6c5888f6dd-xxxxx        1/1     Running   0            115s
wordpress-mysql-6cb8644b8-4bnlr   1/1     Running   0            115s
```

Wait until both pods show `STATUS: Running` and `READY: 1/1`. This is important!

### 🔗 Check Services

Check the WordPress service:

```bash
kubectl get services wordpress
```

You should see:

```
kubectl get services wordpress
NAME        TYPE           CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE
wordpress   LoadBalancer   10.104.219.205   <pending>     80:30525/TCP   2m59s
```

---

## 🌐 Accessing WordPress

### 🔗 Get the WordPress URL

If you use **Minikube**, run this command:

```bash
minikube service wordpress --url
```

You will get URL like this:

```
http://192.168.49.2:30525
```

Open this URL in your browser and you can see WordPress! 🎉

### ☁️ For Cloud Providers

If you use cloud (GKE, EKS, AKS), wait for `EXTERNAL-IP`:

```bash
kubectl get service wordpress --watch
```

When you see external IP, open WordPress at `http://<EXTERNAL-IP>`.

---

## 🏗️ Architecture Components

What we created in Kubernetes:

### 🗄️ MySQL Database
- **Image:** mysql:8.0
- **Storage:** 20Gi persistent volume
- **Service Type:** Headless ClusterIP (only inside cluster)
- **Database Name:** wordpress
- **Password:** Saved in Kubernetes Secret

### 📝 WordPress Frontend
- **Image:** wordpress:6.2.1-apache
- **Storage:** 20Gi persistent volume (for your uploads and themes)
- **Service Type:** LoadBalancer (can access from outside)
- **Port:** 80

### 🔄 Data Flow

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

## 🔍 Troubleshooting

Have problems? Here is how to fix common issues:

### 🚫 Pods Not Starting

Check pod status and see logs:

```bash
kubectl get pods
kubectl describe pod <pod-name>
kubectl logs <pod-name>
```

### 💾 PVC Not Binding

Check PVC status:

```bash
kubectl describe pvc mysql-pv-claim
kubectl describe pvc wp-pv-claim
```

### 🌐 Service Not Accessible

If you use Minikube, make sure tunnel is running:

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

### ⚠️ Connection Refused

If WordPress can't connect to MySQL:

```bash
# Check if MySQL pod is ready
kubectl get pods -l tier=mysql

# See MySQL logs
kubectl logs deployment/wordpress-mysql

# See WordPress logs
kubectl logs deployment/wordpress

# Check MySQL service is there
kubectl get service wordpress-mysql

# Test connection from WordPress pod to MySQL
# If you see "Connected to wordpress-mysql" - it works!
# Exit code 124 (timeout) is ok, means success
kubectl exec deployment/wordpress -- timeout 2 curl -v telnet://wordpress-mysql:3306
```


### 💿 Storage Issues

If PVCs stay in `Pending`:

```bash
# Check storage class
kubectl get storageclass

# See more info about PVC
kubectl describe pvc mysql-pv-claim

# For Minikube, enable storage provisioner
minikube addons enable storage-provisioner
```

---

## 🧹 Cleanup

To delete everything:

```bash
kubectl delete -k ./
```

This command will delete:
- ❌ WordPress and MySQL deployments
- ❌ Services
- ❌ PersistentVolumeClaims (all your data!)
- ❌ Secrets

**⚠️ Warning:** All your WordPress data and database will be deleted forever!

### ✅ Check Cleanup

Make sure everything is deleted:

```bash
kubectl get all,pvc,secrets -l app=wordpress
```

---

## 🎯 Next Steps

What you can do next:

### 1. ✨ Complete WordPress Setup
Open your WordPress URL and finish setup:
- 🌍 Choose language
- 👤 Make admin account
- ⚙️ Set site title and description

### 2. 🔒 Make it Production Ready
```bash
# Make more WordPress pods (for high traffic)
kubectl scale deployment wordpress --replicas=3

# Set resource limits (CPU and memory)
kubectl set resources deployment wordpress --limits=cpu=500m,memory=512Mi
kubectl set resources deployment wordpress-mysql --limits=cpu=1000m,memory=1Gi
```

### 3. 🔐 Add HTTPS
```bash
# Install cert-manager (for SSL certificates)
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.0/cert-manager.yaml

# Setup Ingress with TLS (need ingress controller first)
# More info: https://kubernetes.io/docs/concepts/services-networking/ingress/
```

### 4. 💾 Backup Your Data
```bash
# Backup WordPress files
kubectl exec deployment/wordpress -- tar czf /tmp/wp-backup.tar.gz /var/www/html

# Backup MySQL database
kubectl exec deployment/wordpress-mysql -- mysqldump -u wordpress -p wordpress > backup.sql
```

### 5. 📊 Add Monitoring
- 📈 Use Prometheus to see metrics
- 📉 Make Grafana dashboards
- 📝 Use ELK or Loki for logs

---

## 📚 Additional Resources

- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [WordPress on Docker Hub](https://hub.docker.com/_/wordpress)
- [MySQL on Docker Hub](https://hub.docker.com/_/mysql)
- [Kustomize Documentation](https://kustomize.io/)
- [Kubernetes Storage Classes](https://kubernetes.io/docs/concepts/storage/storage-classes/)

## 🤝 Contributing

You can help make this better! Feel free to open issues or send pull requests.

## 📄 License

This is based on official Kubernetes examples. Same license.

---

**Made with ❤️ for Kubernetes community**
