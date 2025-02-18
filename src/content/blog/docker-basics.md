---
title: "Understanding Docker Basics: A Comprehensive Guide"
author: Obed Macallums
pubDatetime: 2025-02-01T10:00:00Z
slug: docker-basics
description: "A comprehensive guide to Docker fundamentals, including practical examples and best practices for containerizing applications."
tags:
  - docker
  - containerization
  - devops
  - development
featured: true
draft: false
---

Docker simplifies application deployment by containerizing environments. This guide covers basic Docker concepts and commands.

## Table of Contents

## Introduction

Docker has revolutionized how we deploy and run applications. By using containers, it solves the age-old problem of "it works on my machine" by packaging applications and their dependencies into standardized units for development and deployment.

## Key Concepts

### Containers vs Virtual Machines

Unlike traditional VMs that include a full operating system, containers share the host system's OS kernel and isolate the application processes from each other. This makes containers:

- Lighter and faster to start
- More resource-efficient
- Easier to distribute

Here's a visual comparison:

```ascii
Traditional VM:
┌─────────────┐ ┌─────────────┐
│    App 1    │ │    App 2    │
├─────────────┤ ├─────────────┤
│     OS 1    │ │     OS 2    │
├─────────────┤ ├─────────────┤
│  Hypervisor │ │  Hypervisor │
└─────────────┘ └─────────────┘

Containers:
┌────────┐ ┌────────┐ ┌────────┐
│ App 1  │ │ App 2  │ │ App 3  │
├────────┴─┴────────┴─┴────────┤
│         Docker Engine        │
├──────────────────────────────┤
│         Host OS              │
└──────────────────────────────┘
```

### Docker Images

Images are read-only templates containing:

- Application code
- Runtime environment
- System tools
- Libraries
- Settings

## Creating Your First Dockerfile

A Dockerfile is a script of instructions to build a Docker image.

```dockerfile
# Example Dockerfile for a Node.js application
FROM node:16-alpine

WORKDIR /app

# Copy package files
COPY package*.json ./

# Install dependencies
RUN npm install

# Copy application code
COPY . .

# Expose port
EXPOSE 3000

# Start command
CMD ["npm", "start"]
```

## Essential Docker Commands

### Image Management

```bash
# Pull an image
docker pull nginx:latest

# List images
docker images

# Build an image from Dockerfile
docker build -t myapp:1.0 .

# Remove an image
docker rmi nginx:latest
```

### Container Operations

```bash
# Run a container
docker run -d \
  --name my-nginx \
  -p 8080:80 \
  -v $(pwd)/content:/usr/share/nginx/html \
  nginx:latest

# List running containers
docker ps

# List all containers (including stopped)
docker ps -a

# Stop a container
docker stop my-nginx

# Remove a container
docker rm my-nginx
```

## Docker Compose

For multi-container applications, Docker Compose is your friend. Here's an example:

```yaml
version: '3'
services:
  web:
    build: .
    ports:
      - "3000:3000"
    depends_on:
      - db
  db:
    image: mongo:latest
    volumes:
      - mongodb_data:/data/db

volumes:
  mongodb_data:
```

## Practical Example: Running a Web Application

Let's create a simple web application with Node.js and MongoDB:

1. Create the application structure:

    ```bash
    mkdir docker-demo
    cd docker-demo
    ```

2. Create a package.json:

    ```json
    {
    "name": "docker-demo",
    "version": "1.0.0",
    "dependencies": {
        "express": "^4.17.1",
        "mongoose": "^6.0.0"
    }
    }
    ```

3. Create the Dockerfile:

    ```dockerfile
    FROM node:16-alpine
    WORKDIR /app
    COPY package*.json ./
    RUN npm install
    COPY . .
    EXPOSE 3000
    CMD ["node", "app.js"]
    ```

4. Create docker-compose.yml:

    ```yaml
    version: '3'
    services:
    web:
        build: .
        ports:
        - "3000:3000"
        environment:
        - MONGODB_URI=mongodb://db:27017/myapp
    db:
        image: mongo:latest
        volumes:
        - mongodb_data:/data/db

    volumes:
    mongodb_data:
    ```

5. Run the application:

    ```bash
    docker-compose up
    ```

## Best Practices

1. **Use Official Base Images**: Start your Dockerfiles with official images from Docker Hub to ensure security and reliability.
2. **Minimize Layer Count**: Reduce the number of layers in your Docker image by combining commands to decrease image size and improve build speed.
3. **Multi-stage Builds for Production**: Use multi-stage builds to keep your production images lean by separating build-time dependencies from runtime dependencies.
4. **Use .dockerignore**: Exclude unnecessary files and directories from your Docker image using a `.dockerignore` file to reduce image size and build time.
5. **Never Store Secrets in Images**: Avoid storing sensitive information like passwords or API keys directly in your Dockerfile. Use environment variables or Docker secrets instead.

Example of a multi-stage build:

```dockerfile
# Build stage
FROM node:16-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
RUN npm run build

# Production stage
FROM node:16-alpine
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY package*.json ./
RUN npm install --production
EXPOSE 3000
CMD ["npm", "start"]
```

## Troubleshooting

Common commands for debugging:

```bash
# View container logs
docker logs <container-id>

# Execute commands in container
docker exec -it <container-id> /bin/sh

# Inspect container details
docker inspect <container-id>

# View container resource usage
docker stats
```

## Conclusion

Docker has become an essential tool in modern development workflows. By mastering these basics, you're well on your way to containerizing your applications effectively. Remember to:

- Start with simple containers
- Use Docker Compose for complex applications
- Follow best practices
- Keep security in mind

For further learning, explore:

- Docker networking
- Docker volumes
- Container orchestration (Kubernetes)
- CI/CD with Docker

## Resources

- [Official Docker Documentation](https://docs.docker.com/)
- [Docker Hub](https://hub.docker.com/)
- [Docker Compose Documentation](https://docs.docker.com/compose/)
