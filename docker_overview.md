# Docker: A Practical Overview

Docker is a platform that lets you **package, ship, and run applications in lightweight, portable containers**.

Instead of installing software directly on a machine, you run it inside a container — a self-contained unit that includes everything the application needs: code, runtime, libraries, and configuration.

---

## 1. The Problem Docker Solves

Before Docker, deploying software often looked like this:

- A developer writes code on their laptop.
- The code works on their machine.
- They deploy it to a server.
- Something is missing: a different OS version, missing dependency, wrong environment variable.
- The application breaks.

This is the classic **"it works on my machine"** problem.

Docker solves it by packaging the application **with its environment**. If the container runs on the developer's laptop, it will run the same way on a test server or in production.

---

## 2. What Is a Container?

A **container** is a running instance of an application image.

Think of it like a shipping container:

- It packages everything together.
- It looks the same no matter where it is.
- It runs the same way on any compatible host.

A container is **not a virtual machine**. It does not include a full operating system. Instead, it shares the host OS kernel and isolates the application process.

---

## 3. Key Docker Concepts

### Image

A **Docker image** is a read-only template used to create containers.

- It contains the application code.
- It includes dependencies and libraries.
- It defines the commands to run the application.

Images are built from a `Dockerfile`.

### Dockerfile

A `Dockerfile` is a text file with instructions for building an image.

Example:

```dockerfile
# Start from a base image
FROM node:20-alpine

# Set the working directory inside the container
WORKDIR /app

# Copy files into the container
COPY package*.json ./
RUN npm install
COPY . .

# Expose the application port
EXPOSE 3000

# Command to run when the container starts
CMD ["node", "server.js"]
```

### Container

A **container** is a runnable instance of an image.

You can start, stop, delete, and inspect containers.

### Registry

A **registry** stores Docker images.

- **Docker Hub** is the default public registry.
- **Private registries** are used inside companies (AWS ECR, GitHub Container Registry, Azure ACR).

### Volume

A **volume** is persistent storage that survives container restarts.

Containers are ephemeral: when you delete a container, its filesystem changes disappear. Volumes let you keep data outside the container.

### Network

Docker **networks** let containers communicate with each other.

By default, containers on the same network can reach each other by name.

---

## 4. How Docker Works

```
┌─────────────────────────────────────┐
│           Your Application          │
│  ┌─────────┐ ┌─────────┐           │
│  │ Container 1 │ Container 2 │       │
│  │ (Node.js)   │ (PostgreSQL)│       │
│  └────┬────┘ └────┬────┘           │
│       │           │                 │
│       └─────┬─────┘                 │
│             │                       │
│      Docker Engine / Daemon         │
│             │                       │
│      Host Operating System          │
└─────────────────────────────────────┘
```

1. You write a `Dockerfile`.
2. Docker builds an **image** from it.
3. You run the image as a **container**.
4. Docker manages the container's resources, networking, and storage.

---

## 5. Common Docker Commands

| Command | What It Does |
|---|---|
| `docker build -t myapp .` | Build an image from a Dockerfile |
| `docker run -p 3000:3000 myapp` | Run a container and map port 3000 |
| `docker ps` | List running containers |
| `docker ps -a` | List all containers |
| `docker stop <container>` | Stop a running container |
| `docker rm <container>` | Remove a container |
| `docker images` | List local images |
| `docker rmi <image>` | Remove an image |
| `docker logs <container>` | View container logs |
| `docker exec -it <container> sh` | Open a shell inside a running container |
| `docker-compose up` | Start multi-container applications |

---

## 6. Docker Compose

**Docker Compose** lets you define and run multi-container applications with a YAML file.

It is useful when your application has multiple parts: a web server, a database, a cache, etc.

Example `docker-compose.yml`:

```yaml
version: '3.8'

services:
  web:
    build: .
    ports:
      - "3000:3000"
    environment:
      - DATABASE_URL=postgres://user:pass@db:5432/mydb
    depends_on:
      - db

  db:
    image: postgres:15
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
      POSTGRES_DB: mydb
    volumes:
      - pgdata:/var/lib/postgresql/data

volumes:
  pgdata:
```

Run it:

```bash
docker-compose up -d
```

This starts both containers, connects them on the same network, and persists database data.

---

## 7. Docker vs Virtual Machines

| Aspect | Docker Container | Virtual Machine |
|---|---|---|
| OS | Shares host OS kernel | Has its own full OS |
| Size | Megabytes | Gigabytes |
| Startup time | Seconds | Minutes |
| Resource usage | Lightweight | Heavy |
| Isolation | Process-level | Hardware-level |
| Portability | Very portable | Less portable |
| Use case | Microservices, apps, dev environments | Full OS isolation, legacy apps |

Containers are faster and lighter because they do not emulate hardware or run a full guest operating system.

---

## 8. Docker vs Installing Directly on the Host

| Aspect | Docker | Host Installation |
|---|---|---|
| Dependency conflicts | Isolated per container | Shared across system |
| Reproducibility | Same everywhere | Depends on host setup |
| Rollback | Easy: use older image | Harder |
| Cleanup | Remove container/image | Uninstall manually |
| Multiple versions | Easy | Often difficult |

---

## 9. Common Use Cases

| Use Case | Why Docker Helps |
|---|---|
| **Local development** | Run the exact same stack locally as in production. |
| **Microservices** | Package each service independently. |
| **CI/CD pipelines** | Build once, test anywhere, deploy the same image. |
| **Scaling** | Spin up multiple identical containers quickly. |
| **Legacy apps** | Isolate old dependencies without polluting the host. |
| **Testing databases** | Run PostgreSQL, Redis, or Kafka with one command. |

---

## 10. When to Use Docker

Use Docker when:

- You want consistency between development and production.
- You are building microservices.
- You need to scale applications horizontally.
- You want easy dependency isolation.
- You are running CI/CD pipelines.

Avoid Docker when:

- The application is a simple script with no dependencies.
- You need full hardware isolation or multiple operating systems.
- The operational overhead is not justified for the use case.

---

## 11. Typical Docker Workflow

```
Write code
    |
    v
Write Dockerfile
    |
    v
Build image  ---->  docker build -t myapp .
    |
    v
Test locally  ---->  docker run -p 3000:3000 myapp
    |
    v
Push to registry  ---->  docker push registry.com/myapp:1.0
    |
    v
Pull and run in production  ---->  docker pull ... && docker run ...
```

This workflow guarantees that the image tested locally is the same one deployed to production.

---

## 12. Summary

- Docker packages applications into **containers** that run consistently anywhere.
- A **Dockerfile** defines how to build an **image**.
- A **container** is a running instance of an image.
- **Docker Compose** manages multi-container applications.
- Docker is lighter and faster than virtual machines because it shares the host OS kernel.
- It is ideal for development, microservices, CI/CD, and scalable deployments.

Docker does not replace good architecture, but it removes a huge class of environment-related bugs by making deployments predictable and repeatable.