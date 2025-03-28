## 🐳 What is Docker?

Docker is a platform that lets you develop, ship, and run applications in lightweight, isolated containers. Containers package your application along with all its dependencies, ensuring a consistent environment across development, testing, and production.

---

## 🛠 Installing Docker on Ubuntu

For many users, the easiest way to install Docker is to use Ubuntu’s own package. Note that this may not always be the very latest version, but it’s quick and works well for most needs.

1. **Update your package list:**
   ```bash
   sudo apt update
   ```

2. **Install Docker and Docker-Compose:**
   ```bash
   sudo apt install docker.io -y sudo apt install docker-compose -y
   ```

After installing Docker, you can run Docker commands without needing to prefix them with sudo by adding your user to the “docker” group. Here’s how to do it:

1. **Add Your User to the Docker Group:**  
   Run the following command (replace `$(whoami)` with your username if needed):  
   ```bash
   sudo usermod -aG docker $(whoami)
   ```

2. **Log Out and Log Back In:**  
   This ensures your session is updated with the new group membership. Alternatively, you can reboot your system.

3. **Verify:**  
   Run a test command such as:  
   ```bash
   docker run hello-world
   ```  
   If everything is set up correctly, Docker should run the command without requiring sudo.

4. **Enable and start the Docker service:**
   ```bash
   sudo systemctl enable --now docker
   ```

5. **Verify the installation:**
   ```bash
   sudo docker run hello-world
   ```
   This command pulls a test image and runs it in a container. If successful, you’ll see a welcome message from Docker.

*If you need the latest Docker version or prefer using Docker’s official repositories, you can follow a more detailed installation process. However, for most cases, the steps above are sufficient.*

---

## 🌐 Setting Up Docker Swarm

Docker Swarm is Docker’s built-in orchestration tool for managing a cluster of Docker hosts.

### Initialize Docker Swarm on the Manager Node

Run the following command on your chosen manager node (replace `<MASTER_NODE_IP>` with your node’s IP address):
```bash
sudo docker swarm init --advertise-addr <MASTER_NODE_IP>
```

After initialization, Docker outputs a join token command. It looks similar to:
```bash
docker swarm join --token <TOKEN> <MASTER_NODE_IP>:2377
```

### Add Worker Nodes

On each worker node, join the swarm by running:
```bash
sudo docker swarm join --token <TOKEN> <MASTER_NODE_IP>:2377
```

> To check the status of nodes in your swarm (on the manager node):
> ```bash
> sudo docker node ls
> ```

---

## 🚀 Deploying a Service with an Asynchronous Database Replica

This section shows a simplified example of deploying an app that uses a primary database and a replica.

### Step 1: Create an Overlay Network

Create an overlay network (if not already created) so that your services can communicate:
```bash
sudo docker network create -d overlay app_network
```

### Step 2: Define Your Services

Below is an example `docker-compose.yml` that sets up a primary PostgreSQL database (on a manager node) and a replica (on a worker node). Adjust the settings as needed:

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

Deploy your stack with:
```bash
sudo docker stack deploy -c docker-compose.yml myapp
```

### Step 3: Deploy Your Application

Add your application service (such as a web frontend or backend) to the same `docker-compose.yml` or deploy it separately. Ensure it connects to the `app_network` to communicate with the databases.

---

## 🔎 Useful Docker Swarm Commands

- **List services:**
  ```bash
  sudo docker service ls
  ```

- **Inspect service details:**
  ```bash
  sudo docker service ps myapp_db-primary
  ```

- **Check service logs:**
  ```bash
  sudo docker service logs myapp_db-primary
  ```

- **Scale a service (for example, scaling the replica):**
  ```bash
  sudo docker service scale myapp_db-replica=2
  ```

---

## ✅ Summary of Benefits

- **Simplicity:** Quick installation using Ubuntu’s package makes it easy to get started.
- **Scalability:** Easily scale your applications and databases horizontally.
- **High Availability:** Swarm mode ensures that your services remain operational even if some nodes fail.
- **Infrastructure as Code:** Using Docker Compose and Swarm simplifies the deployment process and makes it reproducible.
