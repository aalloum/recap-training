# 🐳 Docker Cheat Sheet

This document gathers essential Docker commands for managing containers, images, and logs.

---

## 🚀 Running a Container
- **Test the Docker installation:**
  ```sh
  docker container run hello-world
- **Run a container in detached mode (background process):**
  ```sh
  docker container run -d -p 8080:8080 virtualpairprogrammers/fleetman-webapp
- **Expose a container on a specific port:**
  ```sh
   docker container run -p 8161:8080 virtualpairprogrammers/fleetman-webapp  # Accessible at http://localhost:8161
   docker container run -p 80:8080 virtualpairprogrammers/fleetman-webapp   # Accessible at http://localhost (port 80 by default)

# 📦 Managing Images & Containers
- **Download an image from Docker Hub:**
  ```sh
    docker pull ubuntu
    docker image pull virtualpairprogrammers/fleetman-webapp  # Equivalent to: docker pull virtualpairprogrammers/fleetman-webapp

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

# 🔨 Building Docker Images with `commit`

Docker allows you to create custom images by modifying an existing container and committing the changes.

---

## 🚀 Steps to Build an Image with `commit`

- **List available Docker images:**
   ```sh
   docker image ls

- **Run an interactive Ubuntu container, install Java Development Kit (JDK), and exit:**
   ```sh
   docker container run -it ubuntu
      > apt-get update
      > apt-cache search jdk
      > apt-get install -y openjdk-8-jdk
      > javac  # Check if Java Compiler is available
      > exit   # Exit the container
  
- **Create a new image from the modified container:**
   ```sh
   docker container commit -a "ALLOUM Abderrahim alloum1abderrahim@gmail.com" <container_id> myjdkimage
  
- **Run a new container using the custom image:**
   ```sh
   docker container run -it myjdkimage
  
- **Verify that the new image has been created:**
   ```sh
   docker image ls

# 🔨 Building Docker Images from a Dockerfile

Docker allows you to automate image creation using a **Dockerfile**, which defines the image’s base, dependencies, and startup command.

---

## 🚀 Steps to Build an Image from a Dockerfile

1. **Create a `Dockerfile` with the following content:**
   ```dockerfile
   FROM ubuntu:latest
   MAINTAINER ALLOUM Abderrahim "alloum1abderrahim@gmail.com"
   RUN apt-get update && apt-get install -y openjdk-8-jdk
   CMD ["/bin/bash"]

2. **Build a Docker image using the Dockerfile:**
   ```sh
   docker image build -t jdk-image-from-dockerfile .
   
3. **Run a container from the newly created image and test Java compilation:**
   ```sh
   docker container run -it jdk-image-from-dockerfile
      > javac  # Check if Java Compiler is available

# 📂 Copying Files to Docker Images

When building a Docker image, you can **copy files from your host machine into the image** using the `COPY` instruction in a **Dockerfile**. This is useful for including application files, configuration files, or dependencies.

---

## 🚀 Steps to Copy Files into a Docker Image

1. **Create a `Dockerfile` with the following content:**
   ```dockerfile
   FROM ubuntu:latest
   MAINTAINER ALLOUM Abderrahim "alloum1abderrahim@gmail.com"
   RUN apt-get update && apt-get install -y openjdk-8-jdk
   WORKDIR /usr/local/bin
   COPY test-program.jar .
   CMD ["/bin/bash"]

  WORKDIR /usr/local/bin → Sets the working directory inside the container.

  COPY test-program.jar . → Copies test-program.jar from the host machine to /usr/local/bin in the container.

2. **Place test-program.jar in the same directory as the Dockerfile so it can be copied into the image.**

3. **Build the Docker image:**
   ```sh
   docker image build -t jdk-image-from-dockerfile .
   
4. **Run a container using the newly created image:**
   ```sh
   docker container run -it jdk-image-from-dockerfile

# 🖥️ Docker Image Commands (`CMD`)

The `CMD` instruction in a **Dockerfile** specifies the default command to be executed when a container starts. This is useful for running applications automatically inside the container.

---

## 🚀 Steps to Use `CMD` in a Docker Image

1. **Create a `Dockerfile` with the following content:**
   ```dockerfile
   FROM ubuntu:latest
   MAINTAINER ALLOUM Abderrahim "alloum1abderrahim@gmail.com"
   RUN apt-get update && apt-get install -y openjdk-8-jdk
   WORKDIR /usr/local/bin
   COPY test-program.jar .
   CMD ["java", "-jar", "test-program.jar"]

  CMD ["java", "-jar", "test-program.jar"] → This ensures that when a container starts, it automatically runs the JAR file.

2. **Place test-program.jar in the same directory as the Dockerfile so it can be copied into the image.**

3. **Build the Docker image:**
   ```sh
   docker image build -t jdk-image-from-dockerfile .

4. **Run a container using the newly created image:**
   ```sh
   docker container run -it jdk-image-from-dockerfile

5. **Manage the container:**

   ```sh
   # List all containers (including stopped ones):
      docker container ls -a
   
   # Restart the stopped container:
      docker container start <container_id>

   # View real-time logs of the running container:
       docker container logs -f <container_id>
   
   # List running containers:
       docker container ls
   
   # Stop a running container:
       docker container stop <container_id>

# ➕ ADD and ENTRYPOINT Commands

The `ADD` and `COPY` instructions are used to add files to a Docker image, while `ENTRYPOINT` and `CMD` define how a container runs when it starts.

---

## 🛠️ **ADD <==> COPY**

- **`ADD`**:
  - Copies files and directories from the host machine to the image.
  - Supports extracting tar archives and downloading files from a URL.
  - Syntax:
    ```dockerfile
    ADD <source> <destination>
    ```

- **`COPY`**:
  - Simply copies files and directories from the host machine to the image.
  - Does not have the extra capabilities of `ADD` (like extracting tar files or downloading files from a URL).
  - Syntax:
    ```dockerfile
    COPY <source> <destination>
    ```

### Key Difference:
- `ADD` can fetch files from a URL or unpack tar archives, while `COPY` is a simpler, more predictable way to copy files.

---

## 🏃‍♂️ **ENTRYPOINT <==> CMD**

- **`ENTRYPOINT`**:
  - Specifies the default application or command to run when the container starts.
  - The container cannot override `ENTRYPOINT` unless using `--entrypoint` during the `docker run` command.
  - Syntax:
    ```dockerfile
    ENTRYPOINT ["executable", "param1", "param2"]
    ```

- **`CMD`**:
  - Provides default arguments to the `ENTRYPOINT` or can be used alone if no `ENTRYPOINT` is specified.
  - The container can override `CMD` by providing arguments when running the container.
  - Syntax:
    ```dockerfile
    CMD ["executable", "param1", "param2"]
    ```

### Key Difference:
- `ENTRYPOINT` sets the **default executable** to run in the container, while `CMD` sets **default arguments** for it.

---

### Example:
```dockerfile
FROM ubuntu:latest
COPY myprogram /usr/local/bin/myprogram
ENTRYPOINT ["/usr/local/bin/myprogram"]
CMD ["--help"]
```

# 🏷️ LABEL Instruction  

The `LABEL` instruction is used to add metadata to a Docker image. You can add information such as the maintainer, creation date, version, and other useful details.

---

## 🚀 Example of Using `LABEL` in a Dockerfile  

- **Create a `Dockerfile` with the following content:**

   ```dockerfile
   FROM ubuntu:latest
   LABEL maintainer="ALLOUM Abderrahim"
   LABEL creationdate="09 November 2024"
   RUN apt-get update && apt-get install -y openjdk-8-jdk
   WORKDIR /usr/local/bin
   COPY test-program.jar .
   CMD ["java", "-jar", "test-program.jar"]
   ```

# 🌐 TOMCAT Dockerfile Example

The following Dockerfile demonstrates how to create a Docker image for running a Tomcat-based web application.

---

## 🚀 Dockerfile Example for Tomcat

1. **Create a `Dockerfile` with the following content:**

   ```dockerfile
   FROM tomcat:8.5.47-jdk8-openjdk
   MAINTAINER Richard Chesterwood "contact@virtualpairprogrammers.com"
   EXPOSE 8080 
   RUN rm -rf /usr/local/tomcat/webapps/*
   COPY ./target/fleetman-0.0.1-SNAPSHOT.war /usr/local/tomcat/webapps/ROOT.war
   ENV JAVA_OPTS="-Dspring.profiles.active=docker-demo"
   CMD ["catalina.sh", "run"]
   ```

2. **Docker Commands to Build and Run the Container**
   ```sh
   # Build the Docker image:
      docker image build -t fleetman-webapp .

   # Get the Docker machine IP (if running on Docker Machine):
       docker-machine ip

   # Run the container with port mapping (8080 to 8080):
       docker container run -p 8080:8080 -it fleetman-webapp

# 🚀 Spring Boot in Docker

This section demonstrates how to build and run a **Spring Boot** application inside a Docker container.

---

## 📝 Dockerfile for Spring Boot

1. **Create a `Dockerfile` with the following content:**

   ```dockerfile
   FROM openjdk:8u131-jdk-alpine
   MAINTAINER Richard Chesterwood "contact@virtualpairprogrammers.com" 
   EXPOSE 8080  
   WORKDIR /usr/local/bin
   COPY ./target/fleetman-0.0.1-SNAPSHOT.jar webapp.jar
   CMD ["java", "-Dspring.profiles.active=docker-demo", "-jar", "webapp.jar"]
   ```
2. **Docker Commands to Build and Run the Container:**
   ```sh
   # Build the Docker image:
      docker image build -t fleetman-webapp .
   
   # Run the container with port mapping (80 on the host to 8080 in the container):
      docker container run -p 80:8080 -it fleetman-webapp

# 🌐 Pushing to Docker Hub

This section explains how to push your Docker image to **Docker Hub**, a cloud-based registry where you can store and share Docker images.

---

## 🚀 Steps to Push an Image to Docker Hub

1. **Create an account on Docker Hub**
  - Sign up for a Docker Hub account at [Docker Hub](https://hub.docker.com/).
  - Choose a Docker ID for your account.

2. **Login to Docker Hub via Command Line**  
   Once your account is created, log in using your **Docker ID** and password:

   ```sh
   docker login
   
  - `username`: Docker_ID (replace with your Docker ID)
  - `password`: xxxx (your Docker Hub password)

3. **Apply a Tag to the Image**  
   After building the Docker image, apply a tag to it in the following format:

   ```sh
   docker image tag <Image_ID> <docker_id>/fleetman-app

- Replace `<Image_ID>` with the image's ID or name.
- Replace `<docker_id>` with your Docker Hub username.

4. **Push the Image to Docker Hub**
   ```sh
   docker image push <docker_id>/fleetman-app

- `Docker ID`: The Docker Hub username you choose when signing up. This is used to uniquely identify your images and repositories on Docker Hub.

- `Tagging`: Tagging the image allows you to give it a name that is unique to your Docker Hub account and repository.

- `Docker Image Push`: Pushing the image uploads it to Docker Hub, making it available to share and use across different systems.

# 🗄️ Database MySQL Container

This section provides instructions on how to run a MySQL container, manage it, and interact with the database.

---

## 🚀 Running a MySQL Container

1. **Start a MySQL container with a root password:**

   ```sh
   docker container run -e MYSQL_ROOT_PASSWORD=password -d mysql:5

- `-e MYSQL_ROOT_PASSWORD=password` → Sets the MySQL root password.
- `-d mysql:5 → Runs the container` in detached mode using MySQL version 5.

    ```sh
    # List running containers:
      docker container run -e MYSQL_ROOT_PASSWORD=password -d mysql:5
  
    # View logs of the running MySQL container:
      docker container logs -f <container_id>
  
    # Access the MySQL container:
      docker container exec -it <container_id> bash

    # Inside the container, access MySQL:
      > mysql -uroot -ppassword
      > show databases;
      docker container stop <container_id>

2. **Running MySQL with a Custom Database:**
    ```sh
    # Start a MySQL container with a pre-created database (fleetman):
      docker container run -e MYSQL_ROOT_PASSWORD=password -e MYSQL_DATABASE=fleetman -d mysql:5
    
    # Access the container shell:
      docker container exec -it <container_id> sh

- `-e MYSQL_DATABASE=fleetman` → Creates the fleetman database on startup.

# 🌐 Network Management in Docker

This section provides instructions on managing Docker networks, creating a custom network, and running containers within it.

---

## 📋 Listing Available Networks

1. **View all existing Docker networks:**

   ```sh
   docker network ls

2. **Creating a Custom Network:**

   ```sh
   # Create a new custom network (my-network):
     docker network create my-network
   
   # Run a MySQL container attached to my-network:
     docker container run --network my-network --name database \
       -e MYSQL_ROOT_PASSWORD=password -e MYSQL_DATABASE=fleetman -d mysql:5

- `--network my-network` → Attaches the container to the `my-network`.
- `--name database` → Assigns the name `database` to the container.
- `-e MYSQL_ROOT_PASSWORD=password` → Sets the MySQL root password.
- `-e MYSQL_DATABASE=fleetman` → Creates the `fleetman` database.
- `-d mysql:5 → Runs the container` in detached mode with MySQL version 5.

   ```sh
   # Run the Fleetman web application container attached to my-network:
     docker container run -d -p 80:8080 --network my-network --name fleetman-webapp fleetman-webapp

- `-d` → Runs the container in detached mode.
- `-p 80:8080` → Maps port `8080` of the container to port `80` on the host machine.
- `--network my-network` → Connects the container to `my-network`.
- `--name fleetman-webapp` → Assigns the name `fleetman-webapp` to the container.

   ```sh
   # Access the fleetman-webapp container’s shell:
     docker container exec -it fleetman-webapp sh

   # Ping an external website
     ping google.com
     ping database

# 🔗 Connecting to a Database Container

This section provides instructions on configuring a Spring Boot application to connect to a MySQL database container and managing the related Docker commands.

---

## 🛠️ 1. Configuring the Application

Create a configuration file (`application-docker-demo.properties`) for connecting to the database:

```properties
spring.cloud.discovery.enabled=false
spring.jpa.properties.hibernate.hbm2ddl.auto=update
security.basic.enabled=false

spring.datasource.driver-class-name=com.mysql.jdbc.Driver
spring.datasource.url=jdbc:mysql://database:3306/fleetman
spring.datasource.username=root
spring.datasource.password=password

logging.level.org.springframework=INFO
```
- `spring.datasource.url=jdbc:mysql://database:3306/fleetman`  => Connects to a MySQL container named database running on port 3306.
- `spring.datasource.username=root`  => Uses root as the database user.
- `spring.datasource.password=password`  => Uses password as the database password.
- `hibernate.hbm2ddl.auto=update`  => Ensures database schema is automatically updated.

## 🚀 2. Managing the Fleetman Web Application

1. **Stopping and Rebuilding the Application:**
   ```sh
   # List running containers:
     docker container ls
   
   # Stop the running fleetman-webapp container:
     docker container stop fleetman-webapp

   # Build a new Docker image for the Fleetman application:
      docker image build -t fleetman-app .

   # List all containers (including stopped ones):
      docker container ls -a

   # Remove the stopped container to free up resources:
      docker container rm fleetman-webapp

   # Run the Fleetman application container connected to the same network as MySQL:
      docker container run -d -p 80:8080 --network my-network --name fleetman-webapp --rm fleetman-webapp

   # Monitor real-time logs of the running application:
      docker container logs -f fleetman-webapp

## 🚀 3. Connecting to the MySQL Database from Another Container

   ```sh
   # Start an interactive Alpine Linux container in the same network:
     docker container run -it --network my-network alpine
   
   # Inside the Alpine container, install the MySQL client:
      apk add --no-cache mysql-client

   # Connect to the MySQL database container:
      mysql -uroot -ppassword -h database

   # Once connected, run SQL queries to explore the database:
      > show databases;
      > use fleetman;
      > show tables;
      > select * from vehicule;
   ```

## 🗄️ Managing Docker Volumes  

This section explains how to create, inspect, and remove Docker volumes for persistent data storage.

---

## 📌 1. Inspect and Clean Up Containers & Volumes  

1. **Inspect a specific container's details:**  
   ```sh
   # Inspect a specific container's details:
   docker container inspect <container_id>

   # Remove all stopped containers:
     docker container prune

   # List existing Docker volumes:
     docker volume ls

   # Remove unused volumes to free space:
      docker volume prune

2. **Inspect a specific container's details:**
   ```sh
   # Run a MySQL container with a named volume (mydata):
     docker container run -v mydata:/var/lib/mysql -d -e MYSQL_ROOT_PASSWORD=password -e MYSQL_DATABASE=fleetman mysql:5
 
   # Inspect the volume details:
     docker volume inspect mydata

   # Access the MySQL container to check the database:
     docker container exec -it <container_id> bash

   # Inside the container, connect to MySQL and verify the database:
      > mysql -ppassword
      > use fleetman;

3. **Cleaning Up & Removing Volumes:**
   ```sh
   # Re-check volume details after using MySQL:
     docker volume inspect mydata

   # Stop the running container:
     docker container stop <container_id>

   # Remove all stopped containers:
      docker container prune

   # List available Docker volumes:
      docker volume ls

   # Remove the mydata volume:
      docker volume rm mydata
   
   # Mount a local folder (C:/Users/Work/mydatabase) to persist MySQL data:
      docker container run -v //c/users/work/mydatabase:/var/lib/mysql -d -e MYSQL_ROOT_PASSWORD=password -e MYSQL_DATABASE=fleetman mysql:5


# 🐳 Docker Integration with Maven

This section explains how to integrate **Docker** with **Maven**, build images, and push them to a repository.

---

## ⚙️ 1. Maven Docker Plugin Configuration

Add the following **Docker Maven Plugin** (`docker-maven-plugin`) to your `pom.xml` file:

```xml
<plugin>
    <groupId>io.fabric8</groupId>
    <artifactId>docker-maven-plugin</artifactId>
    <version>0.21.0</version>

    <configuration>
        <!-- Uncomment the correct Docker host setting if needed -->
        <!-- <dockerHost>http://127.0.0.1:2375</dockerHost> -->  
        <!-- <dockerHost>unix:///var/run/docker.sock</dockerHost> -->

        <verbose>true</verbose>

        <!-- Authentication for Docker Hub (preferably stored in `settings.xml`) -->
        <!-- 
        <authConfig>
            <username>YOUR-USERNAME</username>
            <password>YOUR-PASSWORD</password>
        </authConfig> 
        -->

        <images>
            <image>
                <name>NAME OF IMAGE TO BUILD</name>
                <build>
                    <dockerFileDir>${project.basedir}/src/main/docker/</dockerFileDir>

                    <!-- Copies the JAR file using Maven's Assembly Plugin -->
                    <assembly>
                        <descriptorRef>artifact</descriptorRef>
                    </assembly>

                    <tags>
                        <!-- Change tags to maintain image history, e.g., "production" -->
                        <tag>latest</tag>
                    </tags>
                </build>
            </image>
        </images>
    </configuration>

    <executions>
        <execution>
            <phase>package</phase>
            <goals>
                <goal>build</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```
## ⚙📜 2. Dockerfile for Maven Project

```dockerfile
FROM openjdk:8u131-jdk-alpine
MAINTAINER Richard Chesterwood "contact@virtualpairprogrammers.com"
EXPOSE 8080  
WORKDIR /usr/local/bin
COPY maven/fleetman-0.0.1-SNAPSHOT.jar webapp.jar
CMD ["java", "-Dspring.profiles.active=docker-demo", "-jar", "webapp.jar"]
```
## 🔑 3. Maven settings.xml for Authentication

- Modify your settings.xml file (located in ~/.m2/ on Linux/macOS or %USERPROFILE%\.m2\ on Windows):
```xml
<settings xmlns="http://maven.apache.org/SETTINGS/1.0.0"
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0
                              https://maven.apache.org/xsd/settings-1.0.0.xsd">  

   <servers>
     <server>
       <id>docker.io</id>
       <username>your-username</username>
       <password>your-password</password>
     </server>
   </servers>
</settings>
```

## 🏗️ 4. Build & Push Docker Image
- Run the following Maven command to build and push the Docker image:

```sh
   mvn clean package docker:push
```

# 🚀 Intégration des Pushes avec le Déploiement  

Cette section explique comment **intégrer le déploiement automatique** d'une image Docker avec Maven.

---

## ⚙️ 1. Plugin Maven pour Docker  

Ajoutez le **plugin `docker-maven-plugin`** à votre `pom.xml` :  

```xml
<plugins>
<plugin>
    <groupId>io.fabric8</groupId>
    <artifactId>docker-maven-plugin</artifactId>
    <version>0.21.0</version>

    <configuration>
        <!-- Décommenter si nécessaire pour spécifier l'hôte Docker -->
        <!-- <dockerHost>http://127.0.0.1:2375</dockerHost> -->  
        <!-- <dockerHost>unix:///var/run/docker.sock</dockerHost> -->

        <verbose>true</verbose>

        <!-- Configuration d'authentification (préférer `settings.xml`) -->
        <!-- 
        <authConfig>
            <username>YOUR-USERNAME</username>
            <password>YOUR-PASSWORD</password>
        </authConfig> 
        -->

        <images>
            <image>
                <name>NAME OF IMAGE TO BUILD</name>
                <build>
                    <dockerFileDir>${project.basedir}/src/main/docker/</dockerFileDir>

                    <!-- Copie le JAR avec Maven Assembly Plugin -->
                    <assembly>
                        <descriptorRef>artifact</descriptorRef>
                    </assembly>

                    <tags>
                        <!-- Modifier les tags pour conserver l'historique -->
                        <tag>latest</tag>
                    </tags>
                </build>
            </image>
        </images>
    </configuration>

    <executions>
        <execution>
            <phase>package</phase>
            <goals>
                <goal>build</goal>
            </goals>
        </execution>
        <execution>
            <id>mydeploy</id>
            <phase>deploy</phase>
            <goals>
                <goal>push</goal>
            </goals>
        </execution>
    </executions>
</plugin>

<!-- Plugin optionnel pour désactiver Docker Maven si nécessaire -->
<plugin>
   <artifactId>docker-maven-plugin</artifactId>
   <configuration>
       <skip>true</skip>
   </configuration>
</plugin>
</plugins>
```
## ⚙📜 2. Dockerfile
```dockerfile
FROM openjdk:8u131-jdk-alpine
MAINTAINER Richard Chesterwood "contact@virtualpairprogrammers.com"
EXPOSE 8080  
WORKDIR /usr/local/bin
COPY maven/fleetman-0.0.1-SNAPSHOT.jar webapp.jar
CMD ["java", "-Dspring.profiles.active=docker-demo", "-jar", "webapp.jar"]
```

## 🔑 3. Configuration settings.xml pour Docker Hub
- Modify your settings.xml file (located in ~/.m2/ on Linux/macOS or %USERPROFILE%\.m2\ on Windows):
```xml
<settings xmlns="http://maven.apache.org/SETTINGS/1.0.0"
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0
                              https://maven.apache.org/xsd/settings-1.0.0.xsd">  

   <servers>
     <server>
       <id>docker.io</id>
       <username>your-username</username>
       <password>your-password</password>
     </server>
   </servers>
</settings>
```

## 🏗️ 4. Commande de Build & Deploy
```sh
   mvn clean deploy
```

# 🐳 Docker Compose

Docker Compose permet de **définir et exécuter plusieurs conteneurs** à l'aide d'un fichier YAML.

---

## 🔍 Vérifier la version de Docker Compose
Avant de commencer, assurez-vous que Docker Compose est installé :

```sh
docker-compose -v
```

## 🛠️ Gestion de l'ordre de démarrage (depends_on)
- Avec Docker Compose, l'ordre de démarrage des services peut être contrôlé grâce à depends_on.

## 📜 Fichier docker-compose.yaml

```yaml
version: "3"

services:

   fleetman-webapp:
      image: virtualpairprogrammers/fleetman-production
      networks:
         - fleetman-network
      ports:
         - 80:8080
      depends_on: 
         - database

   database:
      image: mysql:5
      networks:
         - fleetman-network
      environment:
         - MYSQL_ROOT_PASSWORD=password
         - MYSQL_DATABASE=fleetman

networks:
   fleetman-network:

```

- `image` : définit l'image Docker utilisée pour chaque service.
- `ports` : mappe le port interne du conteneur (`8080`) au port externe (`80`).
- `depends_on` : s'assure que `database` démarre avant `fleetman-webapp`.
- `environment` : définit des variables d’environnement pour la base de données.
- `networks` : permet aux services de communiquer sur le même réseau.

## 🔍🚀 Commandes Docker Compose

```sh
    # Démarrer les conteneurs en arrière-plan
       docker-compose -v
       
    #  Afficher les logs de fleetman-webapp
       docker-compose logs -f fleetman-webapp
       
    # Afficher les logs de la base de données
        docker-compose logs -f database
  
    # Arrêter et supprimer tous les conteneurs définis dans docker-compose.yaml
        docker-compose down
```

# 🐳 Docker Swarm

---
## 📌 Commandes de base

```sh
   docker container ls -a
   docker container prune
   docker image ls
```
## 🚀 Initialiser Swarm

```sh
   docker swarm init  # Current node is now a manager
```

## 🌍 Réseaux dans Swarm

```sh
   docker network rm fleetman-network
   docker network create --driver overlay fleetman-network
```
## 📦 Création d'un service (base de données)

```sh
   docker service create -d --network fleetman-network \
      -e MYSQL_ROOT_PASSWORD=password \
      -e MYSQL_DATABASE=fleetman \
      --name database mysql:5
```

## 🖥️ Swarm TP Live (Play with Docker)

```sh
     # Initialisation Swarm avec adresse spécifique
       docker swarm init --advertise-addr 192.168.0.13
     
     # Ajouter un nœud manager
       docker swarm join-token manager
       docker swarm join --token SWMTKN-1-4on9b9chmsg15xfqznz8vvxgy1hiuoyp2r44uthuznaheb0de1-591yc3crr7nh8bkwo18uulzv9 192.168.0.13:2377
       
     # Vérification des nœuds et réseaux
       docker node ls
       docker network create --driver overlay fleetman-network
       docker network ls
```

## 📦 Création des services

```sh
     # Base de données
       docker service create -d --network fleetman-network \
          -e MYSQL_ROOT_PASSWORD=password \
          -e MYSQL_DATABASE=fleetman \
          --name database mysql:5

     # Vérification
       docker service ls
       docker container ls

     # Application web
       docker service create -d --network fleetman-network -p 80:8080 \
           --name fleetman-webapp virtualpairprogrammers/fleetman-production
           
     #  Logs des services
        docker service logs -f database
        docker service logs -f fleetman-webapp
```

# 🐳 Docker Swarm - Stacks

---
## Manager vs Worker

- `Manager` : peut gérer le swarm et exécuter la commande `docker service ls`
- `Worker` : ne peut pas lister les services (`Error: This node is not a swarm manager`)

```sh
     # List Stacks
       docker stack ls
```

## 🚀 Déploiement d'un stack avec Docker Compose
- docker-compose.yaml
```yaml
version: "3"

services:
   fleetman-webapp:
      image: virtualpairprogrammers/fleetman-production
      networks:
         - fleetman-network
      ports:
         - 80:8080
      depends_on: 
         - database

   database:
      image: mysql:5
      networks:
         - fleetman-network
      environment:
         - MYSQL_ROOT_PASSWORD=password
         - MYSQL_DATABASE=fleetman

networks:
   fleetman-network:
```

- Commandes:

```sh
      docker stack deploy -c docker-compose.yaml fleetman-stack
      docker service ls
      docker service ps fleetman-stack_fleetman-webapp  # Historique
```

- Vérification sur Worker:

```sh
      docker container ls
      docker container kill <container_id>
```

- Logs depuis Manager:

```sh
      docker service logs -f fleetman-stack_fleetman-webapp
```

## 📦 Service Répliqué

- docker-compose.yaml
```yaml
version: "3"

services:
   fleetman-webapp:
      image: virtualpairprogrammers/fleetman-production
      networks:
         - fleetman-network
      ports:
         - 80:8080
      depends_on: 
         - database
      deploy: 
         replicas: 2  # Nombre de réplicas

   database:
      image: mysql:5
      networks:
         - fleetman-network
      environment:
         - MYSQL_ROOT_PASSWORD=password
         - MYSQL_DATABASE=fleetman

networks:
   fleetman-network:
```
- Déploiement

```sh
      docker stack deploy -c docker-compose.yaml fleetman-stack
      docker service ls
```

## 🔄 Swarm "Routing Mesh"

- Swarm publie les ports via l'option `-p`

## 📊 Visualizer

- Lancer un visualiseur de cluster pour voir les conteneurs en exécution :

```sh
      docker run -d -p 5000:8080 -v /var/run/docker.sock:/var/run/docker.sock dockersamples/visualizer
```

## 🔄 Rolling Updates
- Permet des mises à jour progressives des conteneurs.

```yaml
version: "3"

services:
   fleetman-webapp:
      image: virtualpairprogrammers/fleetman-production
      networks:
         - fleetman-network
      ports:
         - 80:8080
      depends_on: 
         - database
      deploy: 
         replicas: 2
         update_config:
            parallelism: 1  # Nombre de conteneurs mis à jour en parallèle
            delay: 120s      # Délai entre chaque mise à jour

   database:
      image: mysql:5
      networks:
         - fleetman-network
      environment:
         - MYSQL_ROOT_PASSWORD=password
         - MYSQL_DATABASE=fleetman

networks:
   fleetman-network:

```

## ⚡ Déploiement

```sh
      docker stack deploy -c docker-compose.yaml fleetman-stack
```

- `delay` : Temps d'attente entre la mise à jour d'un groupe de conteneurs.