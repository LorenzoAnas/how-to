---

## 🐳 **What is Docker?**

Docker is a platform for developing, shipping, and running applications within lightweight containers. Containers encapsulate applications along with all dependencies, ensuring consistent and isolated environments across development, testing, and production stages.

---

## 🛠 **Installing Docker on Ubuntu Server**

1. **Update packages**:
```bash
sudo apt update
```

2. **Install required dependencies**:
```bash
sudo apt install ca-certificates curl gnupg lsb-release -y
```

3. **Add Docker’s GPG key and official repository**:
```bash
sudo mkdir -m 0755 -p /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
```

4. **Set up Docker repository**:
```bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo \"$VERSION_CODENAME\") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

5. **Install Docker Engine**:
```bash
sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y
```

6. **Verify Docker Installation**:
```bash
sudo docker run hello-world
```

---

## 🌐 **How to Create and Set up Docker Swarm**

Docker Swarm is Docker's native orchestration tool that manages clusters of Docker hosts.

### Initialize Docker Swarm on Master Node:

```bash
sudo docker swarm init --advertise-addr <MASTER_NODE_IP>
```

The output will give you a **join command** for adding worker nodes, similar to:
```bash
docker swarm join --token <TOKEN> <MASTER_NODE_IP>:2377
```

### Add Worker Nodes:

On each node you'd like to add as a worker, run:

```bash
sudo docker swarm join --token <TOKEN> <MASTER_NODE_IP>:2377
```

> **Check your swarm nodes status (on master)**:
```bash
sudo docker node ls
```

---

## 🚀 **Deploying a Service with Two Nodes Sharing an Asynchronous Database Replica**

**Scenario**: Deploying an app that needs a replicated database asynchronously.

### Step-by-step approach:

**1. Create an overlay network for communication:**
```bash
sudo docker network create -d overlay app_network
```

**2. Deploy your asynchronous replicated database (e.g., PostgreSQL with streaming replication):**

Use Docker Swarm service definitions or `docker-compose.yml`.

Here's an example `docker-compose.yml` (simplified):

```yaml
version: '3.8'

services:
  db-primary:
    image: postgres:16
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
      POSTGRES_DB: mydb
    volumes:
      - primary_data:/var/lib/postgresql/data
    networks:
      - app_network
    deploy:
      placement:
        constraints: [node.role == manager]

  db-replica:
    image: postgres:16
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
      POSTGRES_DB: mydb
      PRIMARY_HOST: db-primary
    command: >
      bash -c "
        until pg_basebackup -h $PRIMARY_HOST -D /var/lib/postgresql/data -U user -P -W --wal-method=stream; do
          echo 'Waiting for primary database...'
          sleep 5
        done &&
        postgres -c hot_standby=on
      "
    volumes:
      - replica_data:/var/lib/postgresql/data
    networks:
      - app_network
    deploy:
      replicas: 1
      placement:
        constraints: [node.role == worker]

networks:
  app_network:
    external: true

volumes:
  primary_data:
  replica_data:
```

Deploy with:
```bash
sudo docker stack deploy -c docker-compose.yml myapp
```

**3. Deploy the application itself (e.g., a web frontend/backend) as another service:**
Add a service definition in the compose file to connect to the databases through `app_network`.

---

## 🔎 **Useful Docker Swarm Commands**

- List services:
```bash
sudo docker service ls
```

- Inspect service details:
```bash
sudo docker service ps myapp_db-primary
```

- Check service logs:
```bash
sudo docker service logs myapp_db-primary
```

- Scale a service:
```bash
sudo docker service scale myapp_db-replica=2
```

---

## ✅ **Summary of Benefits**

- **Easy scalability and redundancy:** Quickly scale apps and databases horizontally.
- **High availability:** If one node goes down, services continue operating on other nodes.
- **Clear and reproducible deployment process:** Infrastructure defined as code.

