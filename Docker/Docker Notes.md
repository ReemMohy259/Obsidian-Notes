---
share_link: https://share.note.sx/ehi9vexw#h+blzgSTu7LW35HJ+xripA
share_updated: 2026-08-04T03:34:59+03:00
---
**We want to solve this problem** 
✔ Works on my laptop  
❌ Fails on another machine

> Visit this site for more details [**Docker Blog**](https://mksherbini.com:666/intro-to-docker)

#### ==Containerization vs Virtualization== 
- **Containerization** virtualizes the **application layer** by sharing the host operating system.
- **Virtualization** virtualizes the **hardware layer**, allowing each virtual machine to run its own complete operating system. **Strong isolation between environments**.

| Containerization                                  | Virtualization                                        |
| ------------------------------------------------- | ----------------------------------------------------- |
| Shares the host operating system's kernel         | Runs a complete guest operating system                |
| Lightweight and starts quickly                    | Heavier and takes longer to start                     |
| Uses less CPU, memory, and storage                | Uses more resources                                   |
| Best for deploying applications and microservices | Best for running multiple different operating systems |
| Examples: Docker, Kubernetes                      | Examples: VMware, VirtualBox, Hyper-V                 |

```
Containerization
+---------------------------+
| Host Operating System     |
+---------------------------+
| Container A (App + Libs)  |
| Container B (App + Libs)  |
| Container C (App + Libs)  |
+---------------------------+

Virtualization
+---------------------------+
| Hypervisor                |
+---------------------------+
| VM A: OS + App + Libs     |
| VM B: OS + App + Libs     |
| VM C: OS + App + Libs     |
+---------------------------+
| Physical Hardware         |
+---------------------------+
```

---
#### ==Image vs Container==
##### Image
A **container image** is a packaged file that contains:
- The application code
- Required libraries
- Dependencies
- Environment settings
- Startup instructions
Think of it as a **recipe** or **snapshot**. An image itself is **not running**. It is simply stored on your computer or in a registry.
##### Container
A **container** is a **running (or stopped) instance of an image**.
When you start an image:
1. Docker copies the image.
2. It creates a writable layer on top.
3. It starts the application.
That running environment is the **container**.

| Image                                    | Container                           |
| ---------------------------------------- | ----------------------------------- |
| A blueprint or template                  | A running instance of an image      |
| Read-only                                | Read-write                          |
| Static (doesn't change while being used) | Dynamic (can change while running)  |
| Used to create containers                | Used to run applications            |
| Can create many containers               | Exists independently after creation |

---
#### ==Docker Image Commands==
- `docker images` or `docker image ls` : Lists all images on your local Docker host.
- `docker pull <image_name>`: Downloads an image from a registry (e.g., Docker Hub)
- `docker push <image_name>`: Pushes an image to a registry (requires authentication)
- `docker build -t <image_name> .`: Builds a new image from a Docker file in the current directory
- `docker rmi <image_name>`: Removes an image
---
#### ==Docker Container Commands==
- `docker run (-d)(--rm)(--name <container_name>)(-p <host-port>:<container-port>) <image_name>`: Creates and runs a container from an image
- `docker ps` or `docker container ls` (-a): Lists all running containers
- `docker stop <container_name>`: Stops a running container
- `docker start <container_name>`: Starts a stopped container
- `docker exec -it <container_name> (bash or sh)`: Opens a terminal session inside a running container
- `docker rm <container_name>`: Removes a stopped container
---
#### ==Docker Network== 
###### Commands
- `docker network ls`: Lists all networks
- `docker network create my-network`: Creates a new network
- `docker network connect my-network <container_name>`: Connects a container to a network
- `docker network disconnect my-network <container_name>`: Disconnects a container from a network
- `docker network rm my-network`: Removes a network (only if no containers are connected)
##### Network Types
- **Bridge Network** (Default): Great for single-host, isolated environments where you want containers to communicate within the host but restrict external access.
- **Host Network**: Useful when performance is critical and network isolation isn’t necessary.
- **Overlay Network**: Ideal for multi-host environments and distributed applications with Docker Swarm.
- **Macvlan Network**: Good for cases where containers need unique IPs on the physical network, like legacy system integration.
- **None Network**: Useful for isolated, no-network-needed tasks.

> [!NOTE] Note
> For two Docker components (such as containers, services, databases, or APIs) to communicate with each other, they must be connected to the **same Docker network**.
> 
> **Clarification:**
> * For two Docker containers to communicate using **container names (Docker DNS)**, they must be connected to the **same user-defined Docker network**.
>
> * Communicating via a **container's internal IP** also requires network connectivity (typically the same Docker network or a route between networks).
> 
> * If a container exposes a **published port**, other containers or external clients can connect through the **host IP and published port**, even if they are not on the same Docker network.

---
#### ==Docker Volume== 
###### Commands
- `docker volume ls`: Lists all volumes
- `docker volume create my-volume`: Creates a new volume
- `docker run -v my-volume:/data <image_name>`: Runs a container with a mounted volume
- `docker volume inspect my-volume`: Shows detailed information about a volume
- `docker volume rm my-volume`: Removes a volume (only if not in use by any containers)
##### Docker Volumes
Docker volumes are a **persistent storage mechanism** designed to store data generated by and used by Docker containers. They provide a way to decouple the data lifecycle from the container lifecycle, ensuring that data remains intact even when containers are deleted or recreated.

> It's like a **hard disk**

##### Docker Volume Types
* **Named Volumes**
	Named volumes are user-defined volumes that can be easily referenced by name and reused across multiple containers. They are stored in Docker’s internal volume store.
* **Anonymous Volumes**
	Anonymous volumes are created when no name is specified. They are typically used for temporary data that does not need to persist beyond the container’s lifecycle.
* **Bind Mounts**
	Bind mounts map a directory or file from the host filesystem to a container. This allows direct access to the host’s filesystem, making it ideal for scenarios where data needs to be shared between the host and the container.
* **tmpfs Volumes**
	tmpfs volumes mount a temporary filesystem in the container’s memory. They are useful for storing non-persistent data that shouldn’t be written to disk, like sensitive information or temporary files.
---
#### ==General Commands==
- `docker info`: Displays information about the Docker daemon
- `docker version`: Shows the Docker client and server version
- `docker system df`: Displays information regarding the amount of disk space used by the Docker
- `docker system prune`: Removes unused Docker resources (images, networks, volumes)
- `docker inspect <object-id>`: Provides detailed information on Docker objects

---
#### ==Architecture==
**Docker based on client-server architecture**

![[Pasted image 20260517192247.png|572]]

---
#### ==Docker File== 
A **Dockerfile** is a text file that contains a set of instructions for building a Docker image. Instead of manually installing software and configuring an environment, you write the steps in a Dockerfile, and Docker automatically creates the image.

```
Dockerfile --> docker build --> Docker Image --> docker run --> Docker Container
```
##### Instruction
- **FROM**: Sets the base image for the Dockerfile.
- **MAINTAINER**: Sets the author of the Dockerfile.
- **RUN**: Executes a command in the Docker image.
- **COPY**: Copies files from the host machine to the Docker image.
- **ADD**: Like COPY but can handle remote URLs or extract archives.
- **EXPOSE**: Hints to Docker that the container will listen on the specified network ports at runtime.
- **ENV**: Sets environment variables for the Docker image.
- **WORKDIR**: Sets the working directory for the Docker image.
- **CMD**: Specifies the default command to run when the container is started.
- **ENTRYPOINT**: Configures the container to run as an executable.
---
#### ==Dockerfile Layers== 
- A **layer** is a read-only filesystem change created by a Dockerfile instruction.
- Most instructions (`FROM`, `RUN`, `COPY`, `ADD`, etc.) create a new layer.
- Layers are **stacked** to form the final Docker image.
- Docker **caches** layers, so unchanged layers are reused in future builds.
**Example**
```dockerfile
FROM python:3.12
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
CMD ["python", "app.py"]
```

**Layers:**
```
Layer 1: FROM python:3.12
Layer 2: WORKDIR /app
Layer 3: COPY requirements.txt .
Layer 4: RUN pip install -r requirements.txt
Layer 5: COPY . .
```

> **Note:** `CMD` sets metadata and does **not** create a filesystem layer.

##### Layer Cache
If only `app.py` changes:
```
COPY requirements.txt   ✓ Cached
RUN pip install         ✓ Cached
COPY . .                Rebuilt
```
**Best Practice:** Copy dependency files first and install dependencies before copying the rest of the source code to maximize cache reuse.

---
#### ==Multi-Stage Builds== 
- A **multi-stage build** uses **multiple `FROM` instructions** in one Dockerfile.
- The first stage builds the application.
- The final stage copies only the required artifacts, excluding build tools and source files.

**Example**
```dockerfile
# Build stage
FROM node:20 AS builder
WORKDIR /app
COPY . .
RUN npm install && npm run build

# Production stage
FROM nginx:latest
COPY --from=builder /app/dist /usr/share/nginx/html
```

**Benefits**
- Smaller image size
- Faster downloads and startup
- Better security (no build tools in production)
- Cleaner production environment
---
#### ==Docker Compose== 
Docker Compose is a tool that lets you **define and manage multiple Docker containers** using a single YAML file (`docker-compose.yml` or `compose.yml`)**.

Instead of running multiple `docker run` commands, you describe all services in one file and start them together.
##### Why Use Docker Compose?
- Run **multi-container applications** with one command.
- Define services, networks, and volumes in a single file.
- Automatically creates a shared network for services.
- Makes development and deployment easier.
##### Networking
- Docker Compose automatically creates a **default network**.
- All services can communicate using their **service names**.

---
#### CI/CD Pipeline
**CI (Continuous Integration)** and **CD (Continuous Delivery/Deployment)** automate the process of building, testing, and deploying applications.
##### Key Points 
- **CI (Continuous Integration):** Automatically builds and tests every code change.
- **Continuous Delivery:** Automatically prepares releases; production deployment usually requires approval.
- **Continuous Deployment:** Automatically deploys every successful change to production.
- Typical pipeline:  
    **Code → Build → Test → Docker Image → Registry → Deploy → Production**
- Benefits:
    - Faster releases
    - Fewer manual errors
    - Consistent deployments
    - Quick feedback on code changes

```
          Developer
              │
         git commit/push
              │
              ▼
        Source Repository
              │
              ▼
        ┌──────────────┐
        │ CI Pipeline  │
        └──────────────┘
              │
      Build Application
              │
      Run Automated Tests
              │
      Build Docker Image
              │
      Push Image to Registry
              │
              ▼
        ┌──────────────┐
        │ CD Pipeline  │
        └──────────────┘
              │
      Deploy to Staging
              │
      Run Integration Tests
              │
      (Optional Approval)
              │
              ▼
      Deploy to Production
```