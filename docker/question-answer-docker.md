# Docker Interview Questions & Answers

## 1. Docker Basics

### What is Docker, and why use it?
Docker is a containerization platform that allows developers to package applications and their dependencies into lightweight, portable containers. It simplifies deployment, ensures consistency across environments, and improves resource efficiency.

### What is the difference between a virtual machine (VM) and a container?
- **VMs** run full operating systems with their own kernel, making them heavy and slow to start.
- **Containers** share the host OS kernel and are lightweight, faster, and more efficient.

### **How is Docker different from a virtual machine (VM)?**
| Feature        | Docker (Containers) | Virtual Machines (VMs) |
|---------------|--------------------|----------------------|
| Speed        | Fast startup        | Slow startup       |
| Isolation    | Shares OS kernel    | Full OS per VM     |
| Resource Usage | Lightweight        | Heavy (each VM needs its OS) |
| Portability  | Easily portable     | Less portable     |


### What are the main components of Docker?
- **Docker Engine**: The core service that runs containers.
- **Docker CLI**: The command-line tool for managing Docker.
- **Docker Images**: Read-only templates used to create containers.
- **Docker Containers**: Runtime instances of Docker images.
- **Docker Hub**: A cloud-based registry for sharing images.

### Explain Docker's architecture.
Docker follows a client-server architecture:
- **Docker Client**: Sends commands to the Docker Daemon.
- **Docker Daemon**: Manages images, containers, and networks.
- **Docker Registry**: Stores and distributes images.

## 2. Essential Docker Commands

### What is the difference between `docker run`, `docker create`, and `docker start`?
- `docker run`: Creates and starts a container in one command.
- `docker create`: Creates a container but does not start it.
- `docker start`: Starts a previously created container.

### How do you list running and stopped containers?
- Running containers: `docker ps`
- All containers (including stopped ones): `docker ps -a`

### How do you remove a container and an image?
- Remove a container: `docker rm <container_id>`
- Remove an image: `docker rmi <image_id>`

### Which command allows you to enter a running container?
```sh
   docker exec -it <container_id> /bin/bash
```

### What is the difference between `COPY` and `ADD` in a Dockerfile?
- `COPY`: Only copies files from the local machine to the image.
- `ADD`: Can copy files from local, extract compressed files, or download from URLs.

## 3. Dockerfile and Images

### How do you create a custom Docker image?
1. Write a `Dockerfile`.
2. Use `docker build -t my-image .` to build the image.
3. Run it using `docker run my-image`.

### What do the following Dockerfile instructions mean?
- `FROM`: Specifies the base image.
- `RUN`: Executes commands during image build.
- `CMD`: Defines the default command when a container starts.
- `ENTRYPOINT`: Similar to `CMD` but cannot be overridden easily.
- `ENV`: Sets environment variables.
- `VOLUME`: Defines persistent storage locations.

### **What is the difference between CMD and ENTRYPOINT?**
| Feature | CMD | ENTRYPOINT |
|---------|----|------------|
| Overridable | Yes | No |
| Usage | Default command | Primary executable |

- `CMD`: Provides a default command but can be overridden.
- `ENTRYPOINT`: Defines a fixed command that cannot be overridden easily.

### How can you reduce the size of a Docker image?
- Use lightweight base images (e.g., `alpine`).
- Minimize unnecessary layers in the Dockerfile.
- Use multi-stage builds to discard unnecessary files.
- Remove cache files after installation (`rm -rf /var/cache/*`).

### What is multi-stage build, and why use it?
Multi-stage builds allow you to use multiple `FROM` statements in a Dockerfile. This is useful for compiling large applications while keeping the final image small.

```dockerfile
FROM golang AS builder
WORKDIR /app
COPY . .
RUN go build -o myapp

FROM alpine
COPY --from=builder /app/myapp /myapp
CMD ["/myapp"]
```

## 4. Docker Networking & Storage

### What are the different types of Docker networks?
- **Bridge (default)**: Used for container-to-container communication.
- **Host**: Uses the host’s network directly.
- **None**: No network access.
- **Overlay**: Used in Docker Swarm for multi-host networking.

### How do you connect multiple containers?
```sh
   docker network create mynetwork
   docker run --network=mynetwork mycontainer
```

### What is a Docker volume, and how do you create one?
```sh
   docker volume create myvolume
   docker run -v myvolume:/data mycontainer
```

### Difference between bind mount and volume?
- **Bind mount**: Links a specific host directory to a container.
- **Volume**: Managed by Docker and stored separately.

## 5. Docker Compose

### What is Docker Compose, and why use it?
Docker Compose is a tool to define and manage multi-container applications using a `docker-compose.yml` file.

### Structure of a `docker-compose.yml` file?
```yaml
version: '3'
services:
  app:
    image: myapp
    ports:
      - "8080:80"
    volumes:
      - mydata:/app/data
volumes:
  mydata:
```

### How to start and stop services with Docker Compose?
- Start: `docker-compose up -d`
- Stop: `docker-compose down`

## 6. Docker Swarm & Orchestration

### What is Docker Swarm?
Docker Swarm is Docker’s native clustering tool for managing a group of containers across multiple machines.

### How do you scale an application with Docker Swarm?
```sh
   docker service scale myservice=3
```

### Difference between Docker Swarm and Kubernetes?
- **Swarm**: Simpler, built into Docker.
- **Kubernetes**: More powerful, widely used in production.

## 7. Security & Optimization

### How do you secure Docker images?
- Use official images.
- Minimize running processes inside the container.
- Enable user namespaces (`--user`).
- Scan images for vulnerabilities (`docker scan`).

### How do you limit memory and CPU usage?
```sh
   docker run --memory=512m --cpus=1 mycontainer
```

### What is Docker Bench Security?
A script that scans Docker configurations and provides security recommendations.

## 8. Debugging & Troubleshooting

### How do you view container logs?
```sh
   docker logs <container_id>
```

### How do you inspect container details?
```sh
   docker inspect <container_id>
```

### What if a container crashes immediately?
- Check logs: `docker logs <container_id>`
- Run in interactive mode: `docker run -it mycontainer /bin/bash`

### What if you get a `port already in use` error?
- Find the process using the port: `netstat -tulnp | grep <port>`
- Stop the conflicting process.

## 9. Docker in CI/CD

### How to integrate Docker into a CI/CD pipeline?
```sh
   docker build -t myapp .
   docker tag myapp myrepo/myapp:v1
   docker push myrepo/myapp:v1
```

### What is the difference between Docker Hub, GitHub Container Registry, and Amazon ECR?
- **Docker Hub**: Public/private repository for Docker images.
- **GitHub Container Registry**: Integrated with GitHub.
- **Amazon ECR**: Used with AWS services.

### **What is the difference between `docker-compose` and `docker stack deploy`?**
- `docker-compose`: Used for local multi-container setups.
- `docker stack deploy`: Used in Swarm mode for production deployments.

### **How do you migrate a Docker container to another host?**
1. Save the container state:
   ```sh
   docker commit <container_id> myimage
   ```
2. Export the image:
   ```sh
   docker save myimage > myimage.tar
   ```
3. Transfer and load on the new host:
   ```sh
   docker load < myimage.tar