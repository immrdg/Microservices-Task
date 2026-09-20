# Microservices Containerization with Docker

## Overview

This project containerizes four Node.js microservices using Docker and orchestrates them with Docker Compose:

| Service | Port | Description |
|---------|------|-------------|
| **User Service** | 3000 | Returns a list of users |
| **Product Service** | 3001 | Returns a list of products |
| **Order Service** | 3002 | Manages orders (GET and POST) |
| **Gateway Service** | 3003 | API gateway that proxies requests to the above services |

## Architecture

```
                    ┌──────────────────┐
  Client ──────────►│  Gateway Service │
                    │    (port 3003)   │
                    └────────┬─────────┘
                             │
              ┌──────────────┼──────────────┐
              ▼              ▼              ▼
     ┌────────────┐  ┌────────────┐  ┌────────────┐
     │   User     │  │  Product   │  │   Order    │
     │  Service   │  │  Service   │  │  Service   │
     │ (port 3000)│  │ (port 3001)│  │ (port 3002)│
     └────────────┘  └────────────┘  └────────────┘
```

All services communicate over a shared Docker bridge network (`microservices-network`). The gateway resolves backend services by their Docker Compose service names (e.g., `http://user-service:3000`).

## Prerequisites

- [Docker](https://docs.docker.com/get-docker/) (v20.10+)
- [Docker Compose](https://docs.docker.com/compose/install/) (v2.0+)

## Project Structure

```
Microservices-Task/
├── Microservices/
│   ├── user-service/
│   │   ├── Dockerfile
│   │   ├── app.js
│   │   └── package.json
│   ├── product-service/
│   │   ├── Dockerfile
│   │   ├── app.js
│   │   └── package.json
│   ├── order-service/
│   │   ├── Dockerfile
│   │   ├── app.js
│   │   └── package.json
│   └── gateway-service/
│       ├── Dockerfile
│       ├── app.js
│       └── package.json
├── docker-compose.yml
└── README.md
```

## Setup Instructions

### 1. Clone the repository

```bash
git clone https://github.com/immrdg/Microservices-Task.git
cd Microservices-Task
```

### 2. Build and start all services

```bash
docker-compose up --build
```

This will:
- Build Docker images for all four services
- Create the `microservices-network` bridge network
- Start all containers with the gateway waiting for the backend services

### 3. Run in detached mode (background)

```bash
docker-compose up --build -d
```

### 4. Stop all services

```bash
docker-compose down
```

## Testing Each Service

### Health Checks

Every service exposes a `/health` endpoint:

```bash
curl http://localhost:3000/health   # User Service
curl http://localhost:3001/health   # Product Service
curl http://localhost:3002/health   # Order Service
curl http://localhost:3003/health   # Gateway Service
```

### User Service (Port 3000)

```bash
# Get all users
curl http://localhost:3000/users
```

Expected response:
```json
[
  { "id": 1, "name": "John Doe" },
  { "id": 2, "name": "Jane Smith" }
]
```

### Product Service (Port 3001)

```bash
# Get all products
curl http://localhost:3001/products
```

Expected response:
```json
[
  { "id": 1, "name": "Laptop", "price": 999 },
  { "id": 2, "name": "Phone", "price": 699 }
]
```

### Order Service (Port 3002)

```bash
# Get all orders
curl http://localhost:3002/orders

# Create a new order
curl -X POST http://localhost:3002/orders \
  -H "Content-Type: application/json" \
  -d '{"userId": 1, "productId": 2}'
```

### Gateway Service (Port 3003)

The gateway aggregates all services under `/api`:

```bash
# Get users via gateway
curl http://localhost:3003/api/users

# Get products via gateway
curl http://localhost:3003/api/products

# Get orders via gateway
curl http://localhost:3003/api/orders

# Create an order via gateway
curl -X POST http://localhost:3003/api/orders \
  -H "Content-Type: application/json" \
  -d '{"userId": 1, "productId": 2}'
```

## Dockerfile Breakdown

Each service uses the same Dockerfile pattern:

```dockerfile
FROM node:18-alpine     # Lightweight Node.js base image
WORKDIR /app            # Set working directory
COPY package.json ./    # Copy dependency manifest first (for layer caching)
RUN npm install         # Install dependencies
COPY . .                # Copy application source code
EXPOSE <port>           # Document the service port
CMD ["node", "app.js"]  # Start the application
```

## Docker Compose Configuration

Key features of the `docker-compose.yml`:

- **Shared network**: All services are on `microservices-network` (bridge driver), enabling inter-service communication by service name
- **Port mapping**: Each service maps its container port to the same host port
- **Dependency management**: The gateway service uses `depends_on` to ensure backend services start first
- **Restart policy**: `unless-stopped` ensures services restart on failure

## Troubleshooting

### Services won't start

```bash
# Check container status
docker-compose ps

# View logs for all services
docker-compose logs

# View logs for a specific service
docker-compose logs gateway-service
```

### Port already in use

If you see `port is already allocated`, stop existing services using those ports:

```bash
# Find what's using the port
lsof -i :3000

# Or stop all containers and rebuild
docker-compose down
docker-compose up --build
```

### Gateway returns 500 errors

The gateway depends on backend services being ready. If the gateway starts before they're fully initialized:

```bash
# Restart just the gateway
docker-compose restart gateway-service
```

### Clean rebuild

If images are cached and changes aren't reflected:

```bash
docker-compose down --rmi all --volumes
docker-compose up --build
```

### Check network connectivity

```bash
# Verify all services are on the same network
docker network inspect microservices-task_microservices-network
```

## Useful Commands

| Command | Description |
|---------|-------------|
| `docker-compose up --build` | Build and start all services |
| `docker-compose up -d` | Start in detached mode |
| `docker-compose down` | Stop and remove containers |
| `docker-compose ps` | List running containers |
| `docker-compose logs -f` | Follow logs from all services |
| `docker-compose restart <service>` | Restart a specific service |
