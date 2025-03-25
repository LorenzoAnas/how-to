## 1. Installing k3s on Ubuntu ARM (Raspberry Pi)

### Why k3s?  
k3s is a lightweight, production-grade Kubernetes distribution built for resource-constrained environments. It comes as a single binary, is easy to install, and natively supports ARM architectures. It also bundles its own container runtime (containerd), so you don’t have to configure Docker separately.

### Step 1: Prepare Your Raspberry Pis
- **Ensure Ubuntu is installed** on each Raspberry Pi.
- **(Optional)** Docker may already be installed, but k3s uses containerd internally. (You can leave Docker installed if needed, as long as there’s no conflict.)

### Step 2: Install k3s on the Master Node
On the Raspberry Pi that you want to use as the master, run:
```bash
curl -sfL https://get.k3s.io | sh -
```
This command downloads and installs k3s, sets up a Kubernetes control plane, and starts the service. Once finished, verify your installation with:
```bash
sudo k3s kubectl get nodes
```
You should see a single node (the master).

### Step 3: Join the Worker Node
On the master node, obtain the join token by running:
```bash
sudo cat /var/lib/rancher/k3s/server/node-token
```
Now, on the second Raspberry Pi (the worker), run the following command (replace `<MASTER_IP>` with the IP address of your master node and `<TOKEN>` with the token you just obtained):
```bash
curl -sfL https://get.k3s.io | K3S_URL=https://<MASTER_IP>:6443 K3S_TOKEN=<TOKEN> sh -
```
After installation, go back to the master and verify that both nodes are present:
```bash
sudo k3s kubectl get nodes
```
You should now see two nodes (one master and one worker).

---

## 2. Converting Your Docker Compose Setup to Kubernetes

Your original Docker Compose file defines three services:
- **db:** PostgreSQL database  
- **django:** Django API service  
- **fetcher:** Order book data fetcher

We’ll convert these into Kubernetes resources. (Note: For a production system, you’d build ARM-compatible Docker images for your Django and fetcher services.)

### 2.1: Deploy PostgreSQL Using a StatefulSet

Create a file named `postgres-statefulset.yaml`:
  
```yaml
apiVersion: v1
kind: Service
metadata:
  name: postgres
  labels:
    app: postgres
spec:
  ports:
    - port: 5432
  clusterIP: None  # Headless service ensures each pod gets a stable DNS name
  selector:
    app: postgres
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: "postgres"
  replicas: 2  # Two replicas to demonstrate basic replication/resilience
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:15
        ports:
        - containerPort: 5432
        env:
          - name: POSTGRES_USER
            value: "user"
          - name: POSTGRES_PASSWORD
            value: "password"
          - name: POSTGRES_DB
            value: "orderbook"
        volumeMounts:
          - name: pgdata
            mountPath: /var/lib/postgresql/data
          # Optionally mount an initialization script
          - name: init-script
            mountPath: /docker-entrypoint-initdb.d
            readOnly: true
  volumeClaimTemplates:
  - metadata:
      name: pgdata
    spec:
      accessModes: ["ReadWriteOnce"]
      resources:
        requests:
          storage: 1Gi
  # To mount your init.sql file, consider creating a ConfigMap.
```

Deploy it with:
```bash
sudo k3s kubectl apply -f postgres-statefulset.yaml
```

### 2.2: Deploy the Django API Service

Create a file named `django-deployment.yaml`:
  
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: crypto-django
spec:
  replicas: 1
  selector:
    matchLabels:
      app: crypto-django
  template:
    metadata:
      labels:
        app: crypto-django
    spec:
      containers:
      - name: django
        image: your-django-image:latest  # Use your ARM-compatible image
        ports:
        - containerPort: 8000
        env:
          - name: DATABASE_URL
            value: "postgres://user:password@postgres:5432/orderbook"
        volumeMounts:
          - name: django-orderbook
            mountPath: /app
      volumes:
      - name: django-orderbook
        hostPath:
          path: /home/ubuntu/crypto_orderbook/django_orderbook  # Adjust to your directory
```

Expose the Django service by creating a Service file named `django-service.yaml`:
  
```yaml
apiVersion: v1
kind: Service
metadata:
  name: crypto-django
spec:
  selector:
    app: crypto-django
  ports:
    - protocol: TCP
      port: 8000
      targetPort: 8000
  type: NodePort
```

Deploy both files:
```bash
sudo k3s kubectl apply -f django-deployment.yaml
sudo k3s kubectl apply -f django-service.yaml
```

### 2.3: Deploy the Fetcher Service

Create a file named `fetcher-deployment.yaml`:
  
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: crypto-fetcher
spec:
  replicas: 1
  selector:
    matchLabels:
      app: crypto-fetcher
  template:
    metadata:
      labels:
        app: crypto-fetcher
    spec:
      containers:
      - name: fetcher
        image: your-fetcher-image:latest  # Use your ARM-compatible image
        env:
          - name: DATABASE_URL
            value: "postgres://user:password@postgres:5432/orderbook"
        volumeMounts:
          - name: src-volume
            mountPath: /app
        tty: true
      volumes:
      - name: src-volume
        hostPath:
          path: /home/ubuntu/crypto_orderbook/src  # Adjust accordingly
```

Deploy it with:
```bash
sudo k3s kubectl apply -f fetcher-deployment.yaml
```

---

## 3. Setting Up Automated Backups with a CronJob

### 3.1: Create a PVC for Backup Storage
Create a file named `backup-pvc.yaml`:
  
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: backup-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

Deploy with:
```bash
sudo k3s kubectl apply -f backup-pvc.yaml
```

### 3.2: Create the Backup CronJob
Create a file named `backup-cronjob.yaml`:
  
```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: postgres-backup
spec:
  schedule: "0 * * * *"  # Every hour on the hour
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup
            image: postgres:15
            command: ["/bin/sh", "-c"]
            args:
              - |
                pg_dump -h postgres-0.postgres -U user orderbook > /backup/backup-$(date +\%Y\%m\%d\%H\%M\%S).sql &&
                echo "Backup complete"
            env:
              - name: PGPASSWORD
                value: "password"
            volumeMounts:
              - name: backup-storage
                mountPath: /backup
          restartPolicy: OnFailure
          volumes:
          - name: backup-storage
            persistentVolumeClaim:
              claimName: backup-pvc
```

Deploy it with:
```bash
sudo k3s kubectl apply -f backup-cronjob.yaml
```

**How It Works:**
- The CronJob creates a pod on schedule (every hour).
- It runs `pg_dump` against the primary PostgreSQL pod (`postgres-0.postgres`) using the headless service DNS.
- The dump is saved to `/backup`, which is mapped to a PVC ensuring storage is persistent and node-independent.

---

## 4. Handling Node Changes and Cluster Dynamics

### Adding a New Node
- **Dynamic Scheduling:**  
  When you add another Raspberry Pi to the cluster, simply install k3s on the new node and join it using the same join command:
  ```bash
  curl -sfL https://get.k3s.io | K3S_URL=https://<MASTER_IP>:6443 K3S_TOKEN=<TOKEN> sh -
  ```
- **StatefulSet Rescheduling:**  
  If a PostgreSQL pod fails on one node, Kubernetes (via k3s) will automatically reschedule it on another available node. The pod will reattach to its existing PVC, ensuring no data loss.
- **Backup CronJob Independence:**  
  The CronJob isn’t tied to any specific node. Its pod might run on any node with access to the backup PVC, so backups continue seamlessly even as nodes are added or removed.

### Scaling and Updates
- **StatefulSet Scaling:**  
  You can scale your PostgreSQL StatefulSet by updating the `replicas` field. New pods will get unique names (e.g., `postgres-2`) and PVCs automatically.
- **Service Discovery:**  
  The headless service (`postgres`) gives each pod a stable DNS entry, ensuring that all components (like your Django and fetcher deployments) can reliably communicate with the database.

---

## 5. Summary of the Full Process Using k3s

1. **Install k3s on Your Raspberry Pis:**  
   - On the master:  
     ```bash
     curl -sfL https://get.k3s.io | sh -
     sudo k3s kubectl get nodes
     ```
   - On the worker:  
     ```bash
     # Obtain the token from the master:
     sudo cat /var/lib/rancher/k3s/server/node-token
     
     # Join the cluster:
     curl -sfL https://get.k3s.io | K3S_URL=https://<MASTER_IP>:6443 K3S_TOKEN=<TOKEN> sh -
     ```

2. **Deploy Your Application Services:**  
   - **PostgreSQL:** Deploy a StatefulSet with a headless Service (`postgres-statefulset.yaml`) to ensure stable pod identities and persistent storage.
   - **Django API & Fetcher:** Convert your Docker Compose services into Kubernetes Deployments (`django-deployment.yaml`, `django-service.yaml`, and `fetcher-deployment.yaml`).

3. **Set Up Automated Backups:**  
   - Create a PVC for backups (`backup-pvc.yaml`).
   - Deploy a CronJob (`backup-cronjob.yaml`) to run periodic backups using `pg_dump`, storing output on the PVC.

4. **Handle Node Changes Dynamically:**  
   - New nodes can be added with the simple k3s join command.
   - StatefulSet pods automatically reattach to their PVCs if rescheduled.
   - The backup process remains robust and node-agnostic.

Using k3s simplifies Kubernetes installation on Ubuntu ARM devices and provides a lean, efficient way to deploy and manage your crypto_orderbook project in a multi-node environment.