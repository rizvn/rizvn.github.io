---
title: Local Development with Docker Compose
categories: [arcitecture]
tags: [docker,compose]
published: false
---

Applications generally have external dependencies, such as databases, message queues, and other services. Setting up and managing these dependencies can be time-consuming and error-prone, especially for developers who are new to the application. Docker Compose is a tool that allows you to define and manage multi-container Docker applications. It provides a simple way to set up and run your application and its dependencies in a consistent environment.

Lets walk through an example of how to use Docker Compose for local development.

Use git to clone an exiting application with multiple external dependencies. 
``` sh
git clone git@github.com:rizvn/go-mcp-server-demo.git mcp-server-demo
```

The compose file is located under docker-compose dir
```sh
cd  mcp-server-demo/docker-compose
```

Below is the docker compose file for the application. It defines three services:
 - keycloak 
 - nginx
 - mcp-server

```yaml
name: mcp-demo
services:
  # Define the Keycloak service
  keycloak:
    image: quay.io/keycloak/keycloak:26.4.0
    command:
      - start-dev
      - --proxy-headers=xforwarded
    environment:
      # Keycloak admin user
      KC_BOOTSTRAP_ADMIN_USERNAME: admin
      KC_BOOTSTRAP_ADMIN_PASSWORD: admin

      # Keycloak configuration
      KC_HOSTNAME_STRICT: false
      KC_HOSTNAME_STRICT_HTTPS: false
      KC_HTTP_ENABLED: true
      KC_HEALTH_ENABLED: true
      KC_METRICS_ENABLED: true

      # JVM options for development
      KC_LOG_LEVEL: INFO

    ports:
      - "9000:8080"
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "curl -f http://localhost:8080/health/ready || exit 1"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 120s
    
    volumes:
      #  use this config for keycloak in development  
      - ./keycloakdb1.mv.db:/opt/keycloak/data/h2/keycloakdb.mv.db

  nginx:
    image: nginx:latest
    ports:
      - "80:80"
    depends_on:
      - keycloak
    volumes:
      # use this config for nginx in development
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    restart: unless-stopped

  mcp-server:
    # build from source code for development
    build:
      context: ../mcp-server
      dockerfile: Dockerfile
    restart: unless-stopped
    environment:
      ISSUER_URL: http://host.docker.internal/realms/demo
      MCP_SERVER_URL: http://localhost:8000
    ports:
      - "8000:8000"
```

Running the following to bring up all the application
```sh
docker compose up -d
```

It will create project named **mcp-demo** and will start all the services defined in the compo

You can use standard commands interact with the containers. Below are some examples of useful commands for managing the containers and viewing logs.:
```sh
# list all projects
docker compose ls -a 

# list all running containers for a project
docker compose -p mcp-demo ps

# view logs for all containers
docker compose -p mcp-demo logs -f

# stop and remove all containers
docker compose -p mcp-demo down

# stop and remove a specific container
docker compose -p mcp-demo stop mcp-server
```

If you make changes to the source code of mcp-server, you can rebuild the image and restart the container to see the changes.
```sh
docker compose -p mcp-demo build mcp-server
docker compose -p mcp-demo restart mcp-server
```
