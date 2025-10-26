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

# Create php-config.ini (fix upload limit)
vi php-config.ini
# Add 6 lines: file_uploads=On, memory_limit=256M, upload_max_filesize=64M,
#              post_max_size=64M, max_execution_time=600, max_input_time=600

# Create kustomization.yaml
vi kustomization.yaml
# Add: secretGenerator, configMapGenerator, resources (see Step 3)

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
- [🧪 Check My Deployment](#check-my-deployment)
- [🏗️ Architecture Components](#architecture-components)
- [🔍 Troubleshooting](#troubleshooting)
- [🔄 How to Apply php-config.ini to Existing Deployment](#how-to-apply-php-configini-to-existing-deployment)
- [🧹 Cleanup](#cleanup)
- [🎯 Next Steps](#next-steps)

---

## 📖 Overview

What this project have:

| Component | Description |
|-----------|-------------|
| **🗄️ MySQL 8.0** | Database with 20Gi storage |
| **📝 WordPress 6.2.1** | WordPress with Apache, 20Gi storage, 64MB upload limit |
| **💾 PersistentVolumeClaims** | Keep your data safe when pod restart |
| **🔗 Kubernetes Services** | LoadBalancer for WordPress, ClusterIP for MySQL |
| **⚙️ Kustomize** | Help us manage configuration easy |
| **🔐 Secrets** | Keep password safe in Kubernetes |
| **📤 PHP Config** | Set upload limit to 64MB (can upload big plugins!) |

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

**Note:** If you use this repository files, they already have upload limit fix included! Skip to Step 3.

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

### Step 3: 📝 Create Configuration Files

In this step, you will create 2 files:
1. **kustomization.yaml** - Tell Kubernetes what to deploy
2. **php-config.ini** - Fix WordPress upload limit

---

**Create kustomization.yaml:**

```bash
vi kustomization.yaml
```

Add the following content:

```yaml
secretGenerator:
- name: mysql-pass
  literals:
  - password=YOUR_PASSWORD

configMapGenerator:
- name: php-config
  files:
  - php-config.ini

resources:
- mysql-deployment.yaml
- wordpress-deployment.yaml
```

**⚠️ Important:** Change `YOUR_PASSWORD` to your own password. Use strong password!

Save and exit (type `:wq` in vi/vim, or you can use `nano`, `code`).

---

**Create php-config.ini:**

This file fix WordPress upload limit (increase from 2MB to 64MB):

```bash
vi php-config.ini
```

**Type this content in the file:**

```ini
file_uploads = On
memory_limit = 256M
upload_max_filesize = 64M
post_max_size = 64M
max_execution_time = 600
max_input_time = 600
```

Save and exit (type `:wq` in vi/vim).

<details>
<summary>💡 How to use vi editor? Click here!</summary>

**Steps to create file with vi:**

1. Run command: `vi php-config.ini`
2. Press `i` to enter insert mode (you see `-- INSERT --` at bottom)
3. Type or paste the 6 lines above
4. Press `Esc` to exit insert mode
5. Type `:wq` and press `Enter` to save and quit

**Don't like vi? Use other editor:**
```bash
# Use nano (easier!)
nano php-config.ini
# After typing content: Press Ctrl+O, Enter, then Ctrl+X

# Or use GUI editor
code php-config.ini     # VS Code
gedit php-config.ini    # Gedit
```

</details>

**💡 What each line means:**

| Line | Meaning |
|------|---------|
| `file_uploads = On` | Allow file uploads (must be On!) |
| `memory_limit = 256M` | PHP can use up to 256MB memory |
| `upload_max_filesize = 64M` | Can upload files up to 64MB (like plugin zip) |
| `post_max_size = 64M` | Can send up to 64MB data in one request |
| `max_execution_time = 600` | Script can run for 600 seconds (10 minutes) |
| `max_input_time = 600` | Can take 600 seconds to upload file |

**📌 Why 64MB?**
- Default is only **2MB** → too small for most plugins! ❌
- **64MB** → can upload most WordPress plugins and themes ✅
- **256MB** memory → enough for normal WordPress
- **600 seconds** → enough time to upload big files

**🎯 Want different size?**

You can change the numbers! Choose based on your site:

| Site Type | upload_max_filesize | post_max_size | memory_limit |
|-----------|---------------------|---------------|--------------|
| 📝 Small blog | 32M | 32M | 128M |
| 🏢 Medium site | 64M | 64M | 256M (recommended) |
| 🏭 Large site | 128M | 128M | 512M |
| 🚀 Enterprise | 256M | 256M | 1024M |

**⚠️ Note:** Bigger numbers = use more memory! Make sure your server can handle it.

**✅ Check your file is correct:**

```bash
# See file content
cat php-config.ini
```

You should see 6 lines like this:
```
file_uploads = On
memory_limit = 256M
upload_max_filesize = 64M
post_max_size = 64M
max_execution_time = 600
max_input_time = 600
```

If file is empty or missing, create it again!

**📁 Check you have all files:**

```bash
ls -l *.yaml *.ini
```

You should see 4 files:
```
kustomization.yaml          ← You created this
mysql-deployment.yaml       ← Downloaded
php-config.ini              ← You created this
wordpress-deployment.yaml   ← Downloaded
```

If missing any file, go back and create it!

**⚠️ Common Mistakes:**

| ❌ Wrong | ✅ Correct |
|---------|----------|
| File name: `php_config.ini` | File name: `php-config.ini` (with dash `-`) |
| Value: `64m` (small m) | Value: `64M` (capital M) |
| Forgot to press `i` in vi | Press `i` first to enter insert mode |
| Created in wrong folder | Put in same folder as kustomization.yaml |
| Forgot to change password | Change `YOUR_PASSWORD` to real password! |

---

## 🚢 Deployment

### Step 4: 🎯 Deploy to Kubernetes

Now we deploy everything! This command will do many things:
- Create the MySQL password secret
- Create PHP config ConfigMap (for upload limit)
- Create services for WordPress and MySQL
- Create persistent volumes
- Deploy WordPress and MySQL pods

```bash
kubectl apply -k ./
```

You will see output like this:

```
kubectl apply -k ./
secret/mysql-pass-5m26tmdb5k created
configmap/php-config-xxxxxxxxxx created
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

## 🧪 Check My Deployment

After deployment, you should check if WordPress and MySQL is working correctly. Here is complete check guide.

### 📋 Quick Check All Components

Run this command to see everything:

```bash
kubectl get all,pvc,secrets -l app=wordpress
```

You should see all resources with status `Running` and `Bound`.

### 🗄️ Check MySQL Database

#### 1. Check MySQL Pod is Running

```bash
kubectl get pods -l tier=mysql
```

Output should show:
```
NAME                              READY   STATUS    RESTARTS   AGE
wordpress-mysql-xxxxxxxxx-xxxxx   1/1     Running   0          5m
```

#### 2. Check MySQL Logs

```bash
kubectl logs -l tier=mysql
```

Look for this message (means MySQL is ready):
```
[Server] /usr/sbin/mysqld: ready for connections
```

#### 3. Test MySQL Database Connection

Connect to MySQL and check database:

```bash
# Get MySQL password from secret
MYSQL_PASSWORD=$(kubectl get secret $(kubectl get secrets | grep mysql-pass | awk '{print $1}') -o jsonpath='{.data.password}' | base64 -d)

# Connect to MySQL pod and check database
kubectl exec -it deployment/wordpress-mysql -- mysql -u wordpress -p${MYSQL_PASSWORD} -e "SHOW DATABASES;"
```

You should see `wordpress` database in list:
```
+--------------------+
| Database           |
+--------------------+
| information_schema |
| performance_schema |
| wordpress          |
+--------------------+
```

#### 4. Check MySQL Service

```bash
kubectl get service wordpress-mysql
```

Should show ClusterIP service:
```
NAME              TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)    AGE
wordpress-mysql   ClusterIP   None         <none>        3306/TCP   5m
```

### 📝 Check WordPress Web

#### 1. Check WordPress Pod is Running

```bash
kubectl get pods -l tier=frontend
```

Output should show:
```
NAME                         READY   STATUS    RESTARTS   AGE
wordpress-xxxxxxxxx-xxxxx    1/1     Running   0          5m
```

#### 2. Check WordPress Logs

```bash
kubectl logs -l tier=frontend
```

Look for Apache started message (no error):
```
AH00558: apache2: Could not reliably determine the server's fully qualified domain name
...
[core:notice] [pid 1] AH00094: Command line: 'apache2 -D FOREGROUND'
```

#### 3. Test WordPress Connection to MySQL

Check if WordPress can connect to database:

```bash
kubectl exec deployment/wordpress -- timeout 2 curl -v telnet://wordpress-mysql:3306
```

If you see `Connected to wordpress-mysql`, it works! ✅
(Exit code 124 is ok, means timeout but connection success)

#### 4. Check WordPress Service

```bash
kubectl get service wordpress
```

Should show LoadBalancer:
```
NAME        TYPE           CLUSTER-IP     EXTERNAL-IP   PORT(S)        AGE
wordpress   LoadBalancer   10.x.x.x       <pending>     80:30525/TCP   5m
```

#### 5. Test WordPress HTTP Response

For Minikube:
```bash
# Get WordPress URL
WORDPRESS_URL=$(minikube service wordpress --url)

# Test HTTP response
curl -I $WORDPRESS_URL
```

For Cloud:
```bash
# Wait for external IP first
kubectl get service wordpress

# Test HTTP response (replace with your EXTERNAL-IP)
curl -I http://<EXTERNAL-IP>
```

You should see `HTTP/1.1 302 Found` or `HTTP/1.1 200 OK` (both is good!)

### 💾 Check Storage (PVC)

```bash
kubectl get pvc
```

Both PVC must be `Bound`:
```
NAME             STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS
mysql-pv-claim   Bound    pvc-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx   20Gi       RWO            standard
wp-pv-claim      Bound    pvc-xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx   20Gi       RWO            standard
```

Check PVC usage:
```bash
# Check MySQL data volume
kubectl exec deployment/wordpress-mysql -- df -h /var/lib/mysql

# Check WordPress data volume
kubectl exec deployment/wordpress -- df -h /var/www/html
```

### 🔐 Check Secrets

```bash
kubectl get secrets | grep mysql-pass
```

Should show secret is created:
```
mysql-pass-xxxxxxxxxx   Opaque   1      10m
```

Verify password is saved (decode it):
```bash
kubectl get secret $(kubectl get secrets | grep mysql-pass | awk '{print $1}') -o jsonpath='{.data.password}' | base64 -d
echo ""
```

### ✅ Complete Deployment Test

Run all checks together with this script:

```bash
echo "=== 🧪 Checking WordPress Deployment ==="
echo ""

echo "1️⃣ Checking all resources..."
kubectl get all,pvc,secrets -l app=wordpress
echo ""

echo "2️⃣ Checking MySQL pod..."
kubectl get pods -l tier=mysql
echo ""

echo "3️⃣ Checking WordPress pod..."
kubectl get pods -l tier=frontend
echo ""

echo "4️⃣ Checking PVC status..."
kubectl get pvc
echo ""

echo "5️⃣ Testing MySQL connection..."
kubectl exec deployment/wordpress -- timeout 2 curl -v telnet://wordpress-mysql:3306 2>&1 | grep -i connected || echo "Connection test done"
echo ""

echo "6️⃣ Getting WordPress URL..."
if command -v minikube &> /dev/null; then
    echo "Minikube detected:"
    minikube service wordpress --url
else
    echo "WordPress Service:"
    kubectl get service wordpress
fi
echo ""

echo "=== ✅ Check Complete! ==="
```

Copy this script and run in terminal. It will check everything for you!

### 🎯 What to Look For

**Everything is Good** ✅ if you see:
- All pods `STATUS: Running`
- All PVC `STATUS: Bound`
- MySQL logs show "ready for connections"
- WordPress can connect to MySQL
- WordPress URL is accessible
- HTTP response is 200 or 302

**Have Problem** ❌ if you see:
- Pods status is `Pending`, `CrashLoopBackOff`, or `Error`
- PVC status is `Pending`
- Can't connect to MySQL from WordPress
- WordPress URL not working
- No HTTP response

If have problem, check [Troubleshooting](#troubleshooting) section below.

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

### 📤 File Upload Size Error (Plugin/Theme Upload)

**Good News:** If you follow setup steps above, upload limit is already set to 64MB! ✅

You can verify by checking:

```bash
# Check PHP settings
kubectl exec deployment/wordpress -- php -i | grep upload_max_filesize
```

Should show: `upload_max_filesize => 64M => 64M`

Or in WordPress admin:
1. Go to **Media** → **Add New**
2. Should show: "Maximum upload file size: 64 MB"

**If you still see 2MB limit or want to change the limit:**

#### Method 1: Update php-config.ini File (Easy)

Edit your `php-config.ini` file and change the values:

```bash
# Edit php-config.ini
vi php-config.ini
```

Change to bigger limits (example 128MB):
```ini
upload_max_filesize = 128M
post_max_size = 128M
memory_limit = 512M
```

Then re-apply:
```bash
kubectl apply -k ./
kubectl rollout restart deployment/wordpress
```

#### Method 2: Using ConfigMap Directly (If you didn't use Step 3)

Create a custom PHP configuration file:

**Step 1:** Create `php-config.ini` file:

```bash
cat > php-config.ini << 'EOF'
file_uploads = On
memory_limit = 256M
upload_max_filesize = 64M
post_max_size = 64M
max_execution_time = 600
EOF
```

**Step 2:** Create ConfigMap from this file:

```bash
kubectl create configmap php-config --from-file=php-config.ini
```

**Step 3:** Edit WordPress deployment to use this ConfigMap:

```bash
kubectl edit deployment wordpress
```

Find the `containers:` section and add volumes and volumeMounts:

```yaml
spec:
  containers:
  - name: wordpress
    image: wordpress:6.2.1-apache
    # ... existing config ...
    volumeMounts:
    - name: wordpress-persistent-storage
      mountPath: /var/www/html
    - name: php-config-volume              # Add this
      mountPath: /usr/local/etc/php/conf.d/uploads.ini    # Add this
      subPath: php-config.ini              # Add this
  volumes:
  - name: wordpress-persistent-storage
    persistentVolumeClaim:
      claimName: wp-pv-claim
  - name: php-config-volume                # Add this
    configMap:                             # Add this
      name: php-config                     # Add this
```

**Step 4:** Save and exit. WordPress pod will restart automatically.

**Step 5:** Verify the changes:

```bash
# Wait for pod to restart
kubectl get pods -l tier=frontend -w

# Check PHP settings
kubectl exec deployment/wordpress -- php -i | grep upload_max_filesize
kubectl exec deployment/wordpress -- php -i | grep post_max_size
```

You should see:
```
upload_max_filesize => 64M => 64M
post_max_size => 64M => 64M
```

#### Method 2: Using .htaccess file (Quick fix)

If method 1 is too complex, try this quick way:

```bash
# Connect to WordPress pod
kubectl exec -it deployment/wordpress -- bash

# Add settings to .htaccess
cat >> /var/www/html/.htaccess << 'EOF'

# Increase upload limits
php_value upload_max_filesize 64M
php_value post_max_size 64M
php_value memory_limit 256M
php_value max_execution_time 600
php_value max_input_time 600
EOF

# Exit pod
exit
```

**Note:** This method may not work if server don't allow .htaccess override.

#### Method 3: Edit php.ini directly (Temporary)

**Warning:** This change will lost when pod restart!

```bash
# Connect to WordPress pod
kubectl exec -it deployment/wordpress -- bash

# Find php.ini location
php --ini

# Edit php.ini (usually at /usr/local/etc/php/php.ini-production)
# Copy and edit
cp /usr/local/etc/php/php.ini-production /usr/local/etc/php/php.ini

# Use sed to change values
sed -i 's/upload_max_filesize = 2M/upload_max_filesize = 64M/g' /usr/local/etc/php/php.ini
sed -i 's/post_max_size = 8M/post_max_size = 64M/g' /usr/local/etc/php/php.ini
sed -i 's/memory_limit = 128M/memory_limit = 256M/g' /usr/local/etc/php/php.ini
sed -i 's/max_execution_time = 30/max_execution_time = 600/g' /usr/local/etc/php/php.ini

# Restart Apache
apache2ctl graceful

# Exit pod
exit
```

#### Method 4: Use Custom WordPress Image (Best for Production)

Create your own WordPress image with custom PHP settings:

**Step 1:** Create `Dockerfile`:

```dockerfile
FROM wordpress:6.2.1-apache

# Copy custom PHP config
COPY php-config.ini /usr/local/etc/php/conf.d/uploads.ini
```

**Step 2:** Create `php-config.ini`:

```ini
file_uploads = On
memory_limit = 256M
upload_max_filesize = 64M
post_max_size = 64M
max_execution_time = 600
max_input_time = 600
```

**Step 3:** Build and push image:

```bash
# Build image
docker build -t your-registry/wordpress-custom:latest .

# Push to registry
docker push your-registry/wordpress-custom:latest
```

**Step 4:** Update deployment to use your image:

```bash
kubectl set image deployment/wordpress wordpress=your-registry/wordpress-custom:latest
```

#### Verify Upload Limit Changed

After applying any method, check in WordPress:

1. Open WordPress admin dashboard
2. Go to **Media** → **Add New**
3. Look at "Maximum upload file size" message
4. Should show: "Maximum upload file size: 64 MB" (instead of 2 MB)

Or check from command line:

```bash
# Get WordPress URL
WORDPRESS_URL=$(minikube service wordpress --url 2>/dev/null || kubectl get service wordpress -o jsonpath='{.status.loadBalancer.ingress[0].ip}')

# Check phpinfo (if you have phpinfo page)
curl -s http://$WORDPRESS_URL/phpinfo.php | grep upload_max_filesize
```

#### Recommended Settings

For different use cases:

**Small blog:**
```ini
upload_max_filesize = 32M
post_max_size = 32M
memory_limit = 256M
```

**Medium site with plugins:**
```ini
upload_max_filesize = 64M
post_max_size = 64M
memory_limit = 256M
```

**Large site with many plugins/themes:**
```ini
upload_max_filesize = 128M
post_max_size = 128M
memory_limit = 512M
```

**Note:** Bigger limits use more memory and resources!

---

## 🔄 How to Apply php-config.ini to Existing Deployment

If you already deployed WordPress before and now want to add the upload limit fix, follow these steps:

### Method 1: Using This Repository Files (Recommended)

If you use the files from this repository, they already have the fix! Just apply:

```bash
# Make sure you have all files in current directory
ls -l *.yaml *.ini

# Apply the changes
kubectl apply -k ./
```

This will:
- Create ConfigMap from php-config.ini
- Update WordPress deployment
- Restart WordPress pod automatically

**Check if it worked:**

```bash
# Wait for pod to restart
kubectl rollout status deployment/wordpress

# Check upload limit
kubectl exec deployment/wordpress -- php -i | grep upload_max_filesize
```

Should show: `upload_max_filesize => 64M => 64M` ✅

### Method 2: Already Deployed Without php-config.ini?

If you already deployed WordPress without the upload limit fix, here's how to add it:

**Step 1: Create php-config.ini file**

```bash
vi php-config.ini
```

Add this content:
```ini
file_uploads = On
memory_limit = 256M
upload_max_filesize = 64M
post_max_size = 64M
max_execution_time = 600
max_input_time = 600
```

Save and exit.

**Step 2: Update kustomization.yaml**

Edit your kustomization.yaml:

```bash
vi kustomization.yaml
```

Add the `configMapGenerator` section:

```yaml
secretGenerator:
- name: mysql-pass
  literals:
  - password=YOUR_PASSWORD

configMapGenerator:           # Add this section
- name: php-config            # Add this line
  files:                      # Add this line
  - php-config.ini            # Add this line

resources:
- mysql-deployment.yaml
- wordpress-deployment.yaml
```

Save and exit.

**Step 3: Download and update wordpress-deployment.yaml**

You need the updated wordpress-deployment.yaml from this repository, or manually add the volume mount:

```bash
# Download updated file
curl -LO https://raw.githubusercontent.com/YOUR_REPO/wordpress-deployment.yaml

# Or manually edit your existing file
vi wordpress-deployment.yaml
```

Add these lines in the deployment section:

```yaml
volumeMounts:
- name: wordpress-persistent-storage
  mountPath: /var/www/html
- name: php-config-volume              # Add this
  mountPath: /usr/local/etc/php/conf.d/uploads.ini    # Add this
  subPath: php-config.ini              # Add this

volumes:
- name: wordpress-persistent-storage
  persistentVolumeClaim:
    claimName: wp-pv-claim
- name: php-config-volume              # Add this
  configMap:                           # Add this
    name: php-config                   # Add this
```

**Step 4: Apply changes**

```bash
# Apply all changes
kubectl apply -k ./

# WordPress pod will restart automatically
kubectl get pods -w
```

**Step 5: Verify**

```bash
# Check PHP settings
kubectl exec deployment/wordpress -- php -i | grep upload_max_filesize
kubectl exec deployment/wordpress -- php -i | grep post_max_size

# Or check in WordPress admin
# Go to Media → Add New
# Should show "Maximum upload file size: 64 MB"
```

### Method 3: Quick Update (If Already Have All Files)

If you already have the updated files from this repository:

```bash
# Just apply
kubectl apply -k ./

# Wait for restart
kubectl rollout status deployment/wordpress

# Done!
```

### 🔍 Troubleshooting Apply Issues

**Problem: ConfigMap not created**

```bash
# Check if ConfigMap exists
kubectl get configmap

# If missing, check kustomization.yaml has configMapGenerator section
cat kustomization.yaml
```

**Problem: Pod not restarting**

```bash
# Force restart
kubectl rollout restart deployment/wordpress

# Check restart status
kubectl rollout status deployment/wordpress
```

**Problem: Still showing 2MB limit**

```bash
# Check if ConfigMap is mounted
kubectl describe pod -l tier=frontend | grep php-config

# Check file exists in pod
kubectl exec deployment/wordpress -- ls -la /usr/local/etc/php/conf.d/

# Check file content
kubectl exec deployment/wordpress -- cat /usr/local/etc/php/conf.d/uploads.ini
```

**Problem: Error "configmap not found"**

The ConfigMap name has hash suffix. Check actual name:

```bash
# See ConfigMap name
kubectl get configmap | grep php-config

# Update wordpress-deployment.yaml to use correct name
# Or let kustomize handle it (recommended)
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
