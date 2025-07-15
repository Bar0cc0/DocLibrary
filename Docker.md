# 🐳 Docker Comprehensive Cheat Sheet

A complete guide for Docker commands, container management, and best practices.

## Table of Contents
[1. Dockerfile Basics](#1-dockerfile-basics)
[2. Container Management](#2-container-management)
[3. Image Management](#3-image-management)
[4. Networks](#4-networks)
[5. Port Binding](#5-port-binding)
[6. Data Persistence](#6-data-persistence)
[7. Docker Compose](#7-docker-compose)
[8. Common Scenarios](#8-common-scenarios)
[9. Volume Management Tips](#9-volume-management-tips)

## 1. Dockerfile Basics

```dockerfile
# Dockerfile must have uppercase 'D'
FROM ubuntu:20.04                      # Base image

# Environment variables
ENV APP_HOME=/app \
    NODE_ENV=production

# Set working directory (equivalent to 'cd /app')
WORKDIR /app

# Copy files from host to container
COPY package.json .                    # Copy specific file
COPY . .                               # Copy everything in current dir

# Run commands during build
RUN apt-get update && \
    apt-get install -y nodejs && \
    npm install

# Expose ports
EXPOSE 8080/tcp                        # Document that container listens on port 8080

# Default command when container starts
CMD ["node", "app.js"]                 # Exec form (preferred)
# or
ENTRYPOINT ["node", "app.js"]          # Cannot be overridden at runtime
```

## 2. Container Management

```bash
# List running containers
docker ps                              # Show running containers
docker ps -a                           # Show all containers (including stopped)
docker-compose ps                      # List compose-managed containers

# Container lifecycle
docker run <image>                     # Create and start a new container
docker run --rm <image>                # Remove container when it exits
docker run -d <image>                  # Run in detached mode (background)
docker run -it <image> <command>       # Interactive mode with terminal

# Start/stop existing containers
docker start <container_id|name>       # Start a stopped container
docker stop <container_id|name>        # Stop a running container (SIGTERM)
docker restart <container_id|name>     # Restart a container
docker kill <container_id|name>        # Force stop a container (SIGKILL)

# Access container
docker exec -it <container_id> bash    # Run command in a running container
docker-compose exec <service> bash     # Access service in compose project
docker logs <container_id|name>        # View container logs
docker logs -f <container_id|name>     # Follow log output in real time
docker-compose logs                    # View all compose service logs
docker-compose logs --tail=100 <service> # View last 100 lines of service logs

# Exit container
exit                                   # Exit shell in container
```

## 3. Image Management

```bash
# List images
docker image ls                        # List all local images
docker images --filter "dangling=true" # List dangling images
docker container ls                    # List containers

# Build images
docker build -t <name>:<tag> <path>    # Build from Dockerfile in path
docker build -f /path/to/Dockerfile -t <name>:<tag> <path> # Specify Dockerfile
docker build --no-cache <path>         # Build without using cache
docker build -t <name>:<tag> . --progress=plain # See detailed build output
docker-compose build                   # Build all services in compose file

# Pull/push images
docker pull <image>:<tag>              # Download image from registry
docker push <image>:<tag>              # Upload image to registry

# Clean up
docker rmi <image_id|name>             # Remove an image
docker image prune                     # Remove dangling images
docker image prune -a                  # Remove all unused images
```

## 4. Networks

```bash
# Network commands
docker network ls                      # List all networks
docker network inspect <network>       # Display network details
docker network create <network_name>   # Create a user-defined network
docker network rm <network_name>       # Remove a network

# Connect containers to networks
docker run --network=<network_name> <image>  # Run container in network
docker network connect <network> <container> # Connect existing container to network
docker network disconnect <network> <container>  # Disconnect from network
```

## 5. Port Binding

```bash
# Expose container ports to host
docker run -p <host_port>:<container_port> <image>   # Map specific port
docker run -p 8080:80 nginx            # Example: Map container port 80 to host port 8080
docker run -P <image>                  # Map all exposed ports to random host ports

# In Dockerfile
EXPOSE 80/tcp                          # Document that container listens on port 80
```

## 6. Data Persistence

```bash
# Named volumes
docker run -v <volume_name>:<container_path> <image>  # Create/use named volume
docker run -v postgres_data:/var/lib/postgresql/data postgres  # Example

# Bind mounts (map host directory to container)
docker run -v <host_path>:<container_path> <image>   # Map host dir to container
docker run -v $(pwd):/app node         # Example: Map current directory to /app

# List and manage volumes
docker volume ls                       # List all volumes
docker volume create <name>            # Create a volume
docker volume rm <name>                # Delete a volume
docker volume prune                    # Remove all unused volumes
```

## 7. Docker Compose

Docker Compose automates running multi-container applications.

```yaml
# docker-compose.yml
version: '3'                           # Compose file version

services:
  web_app:                             # Service name
    image: nginx:latest                # Image to use
    build: ./app                       # Or build from Dockerfile in directory
    ports:
      - "8080:80"                      # Port mapping (host:container)
    volumes:
      - ./app:/usr/share/nginx/html    # Volume mapping (host:container)
    environment:                       # Environment variables
      - NODE_ENV=production
    depends_on:                        # Start dependencies first
      - db
    networks:
      - app_network                    # Connect to this network
      
  db:
    image: postgres
    ports:
      - "5432:5432"
    environment:
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=secret123
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - app_network
      
  airflow:                             # Example Airflow service
    image: apache/airflow:2.7.0b1-python3.10
    ports:
      - "8080:8080"
      
networks:
  app_network:                         # Define custom network
    driver: bridge
    
volumes:
  postgres_data:                       # Define named volume
    driver: local
```

```bash
# Docker Compose commands
docker-compose up                      # Create and start containers
docker-compose up -d                   # Start in detached mode
docker-compose down                    # Stop and remove containers
docker-compose down -v                 # Also remove volumes
docker-compose -f custom-file.yaml up  # Use specific compose file
docker-compose build                   # Build all services
```

## 8. Common Scenarios

### Running PostgreSQL with custom config

```bash
docker run -d -p 5432:5432 \
           -e POSTGRES_USER=postgres \
           -e POSTGRES_PASSWORD=secret123 \
           -e POSTGRES_DB=myapp \
           --name postgresdb \
           --network app_network \
           -v postgres_data:/var/lib/postgresql/data \
           postgres:13
```

### Running multiple containers in a network

```bash
# 1. Create a network
docker network create app_network

# 2. Run database container
docker run -d -p 5432:5432 \
           -e POSTGRES_USER=postgres \
           -e POSTGRES_PASSWORD=secret123 \
           --name postgres_db \
           --network app_network \
           postgres

# 3. Run application container in same network
docker run -d -p 8080:8080 \
           -e DB_HOST=postgres_db \
           --name my_app \
           --network app_network \
           my_app_image
```

### Running Selenium container

```bash
docker run -d \
           -p 4444:4444 \
           --shm-size="2g" \
           --name firefox_selenium \
           selenium/standalone-firefox
```

### Accessing Airflow

```bash
# Access Airflow container
docker-compose exec airflow bash

# Get admin password
cat $AIRFLOW_HOME/standalone_admin_password.txt

# Create a new user
airflow users create \
    --username me \
    --firstname Me \
    --lastname User \
    --role Admin \
    --email me@example.com \
    --password me
```

## 9. Volume Management Tips

### When You DON'T Need to Rebuild:

1. **Scripts in mounted directories**: Any changes to Python scripts in mounted volumes
2. **Data files**: Any files you add/modify in mounted data directories

```yaml
volumes:
  - ./scripts:/opt/airflow/ScreenplaysNLP/Scripts
  - ./data:/opt/airflow/ScreenplaysNLP/Data
```

### When You DO Need to Rebuild:

1. **Dockerfile changes**: If you modify the Dockerfile (adding dependencies, etc.)
2. **System packages**: If you need to install additional system packages with apt-get
3. **Python packages**: If you update requirements.txt with new dependencies
4. **Files copied into the image**: Any files that are COPIED into the Docker image instead of mounted as volumes
5. **Entrypoint scripts**: If you change startup or installation scripts