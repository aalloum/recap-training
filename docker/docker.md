# 🐳 Docker Cheat Sheet

This document gathers essential Docker commands for managing containers, images, and logs.

---

## 🚀 Running a Container
- **Test the Docker installation:**
  ```sh
  docker container run hello-world
- **Run a container in detached mode (background process):**
  ```sh
  docker container run -d -p 8080:8080 virtualpairprogrammers/fleatman-webapp
- **Expose a container on a specific port:**
  ```sh
   docker container run -p 8161:8080 virtualpairprogrammers/fleatman-webapp  # Accessible at http://localhost:8161
   docker container run -p 80:8080 virtualpairprogrammers/fleatman-webapp   # Accessible at http://localhost (port 80 by default)

# 📦 Managing Images & Containers
- **Download an image from Docker Hub:**
  ```sh
    docker pull ubuntu
    docker image pull virtualpairprogrammers/fleatman-webapp  # Equivalent to: docker pull virtualpairprogrammers/fleatman-webapp

- **List running containers:**
  ```sh
    docker ps  # Equivalent to: docker container ls
  
- **List all containers (including stopped ones):**
  ```sh
     docker ps -a  # Equivalent to: docker container ls -a

- **List all available images:**
  ```sh
     docker image ls

- **Start a container in interactive mode:**
  ```sh
     docker container run -it ubuntu

- **Open a Bash shell inside a container:**
  ```sh
     docker container exec -it <container_id> bash

- **Restart a stopped container:**
  ```sh
     docker container start <container_id>

- **Stop a running container:**
  ```sh
     docker container stop <container_id>

- **Remove a stopped container:**
  ```sh
     docker container rm <container_id>

- **Remove all stopped containers:**
  ```sh
     docker container prune

# 🔍 Monitoring & Logs

- **Check logs of a container:**
  ```sh
     docker container logs <container_id>

- **Follow logs in real-time:**
  ```sh
     docker container logs -f <container_id>

# 🌐 Working with Other Images

- **Download and run a Tomcat container:**
  ```sh
     docker image pull tomcat
     docker container run -d -p 80:8080 tomcat
  
- **📌 Tip: To get the IP address of your Docker virtual machine (Docker Machine):**
   ```sh
     docker-machine ip
