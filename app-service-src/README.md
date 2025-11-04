# App Service

Simple tasks API for Helmfile tutorial.

## Features

- Express.js REST API
- PostgreSQL database
- CRUD operations for tasks
- Health check endpoint

## Endpoints

- `GET /health` - Health check
- `GET /api/tasks` - List all tasks
- `POST /api/tasks` - Create task (body: `{"title": "..."}`)
- `DELETE /api/tasks/:id` - Delete task

## Environment Variables

- `PORT` - Server port (default: 3000)
- `DB_HOST` - PostgreSQL host
- `DB_NAME` - Database name (default: appdb)
- `DB_USER` - Database user (default: appuser)
- `DB_PASSWORD` - Database password (required)
- `DB_PORT` - Database port (default: 5432)

## Local Development
```bash
# Install dependencies
npm install

# Set environment variables
export DB_HOST=localhost
export DB_NAME=appdb
export DB_USER=appuser
export DB_PASSWORD=secret

# Start PostgreSQL (docker)
docker run -d \
  --name postgres-dev \
  -e POSTGRES_DB=appdb \
  -e POSTGRES_USER=appuser \
  -e POSTGRES_PASSWORD=secret \
  -p 5432:5432 \
  postgres:15-alpine

# Run app
npm start
```

## Build Docker Image
```bash
# Build
docker build -t app-service:1.0.0 .

# Run
docker run -d \
  --name app-service \
  -e DB_HOST=host.docker.internal \
  -e DB_PASSWORD=secret \
  -p 4000:3000 \
  app-service:1.0.0

# Test
curl http://localhost:3000/health
curl http://localhost:3000/api/tasks
```

## Testing in Kubernetes
```bash
# Port-forward to the service
kubectl port-forward -n dev svc/app-service 3000:80

# Test endpoints
curl http://localhost:3000/health
curl http://localhost:3000/api/tasks
curl -X POST http://localhost:3000/api/tasks -H "Content-Type: application/json" -d '{"title":"New task"}'
```