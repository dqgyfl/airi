# Docker Deployment Guide for Project AIRI

This guide covers the Docker deployment workflow for both the public server and stage-web components of Project AIRI.

## Overview

The project uses GitHub Container Registry (GHCR) for Docker image storage and GitHub Actions for automated builds and deployments.

## Available Docker Images

### Stage-Web (Frontend)
- **Image**: `ghcr.io/moeru-ai/airi/stage-web`
- **Port**: 80 (nginx)
- **Build Context**: Root directory
- **Dockerfile**: `apps/stage-web/Dockerfile`

### Server (Backend)
- **Image**: `ghcr.io/moeru-ai/airi/server`
- **Port**: 3000 (mapped to 6112 in docker-compose)
- **Build Context**: Root directory
- **Dockerfile**: `apps/server/Dockerfile`

## Deployment Workflow

### Automated CI/CD

Docker images are automatically built and pushed on:
- **Release Tags**: Any tag starting with `v*` (e.g., `v1.0.0`)
- **Manual Trigger**: Via GitHub Actions workflow dispatch

### Image Tags

Each service gets the following tags:
- `latest` - Most recent successful build
- `v1.2.3` - Exact version
- `v1.2` - Major.Minor version
- `v1` - Major version (for versions > v0.x)

### Multi-Platform Support

Both images support:
- `linux/amd64` (x86_64)
- `linux/arm64` (ARM64)

## Manual Deployment

### Build Stage-Web Locally

```bash
# Build stage-web
cd /vfs/ai-team/airi
docker build -f apps/stage-web/Dockerfile -t ghcr.io/moeru-ai/airi/stage-web:latest .

# Build server
docker build -f apps/server/Dockerfile -t ghcr.io/moeru-ai/airi/server:latest .
```

### Run Full Stack with Docker Compose

```bash
cd apps/server
docker compose up -d
```

This starts:
- PostgreSQL database (port 5435)
- Redis cache (port 6379)
- API server (port 6112 → 3000)
- Billing consumer (background processor)

### Run Stage-Web Only

```bash
# Using the built image
docker run -p 8080:80 ghcr.io/moeru-ai/airi/stage-web:latest

# Or build and run locally
docker build -f apps/stage-web/Dockerfile -t stage-web .
docker run -p 8080:80 stage-web
```

## GitHub Actions Workflow

### Triggering Builds

1. **Automatic on Tags**: Push a tag like `v1.2.3`
2. **Manual Trigger**: Go to Actions → "Build and Push Docker Images" → Run workflow

### Manual Build Options

When manually triggering, you can specify:
- `all` - Build both services (default)
- `stage-web` - Build only frontend
- `server` - Build only backend

## Development Workflow

### Local Development

```bash
# Stage-Web (development)
pnpm -F @proj-airi/stage-web dev

# Server (development)
pnpm -F @proj-airi/server dev
```

### Production Builds

```bash
# Stage-Web
pnpm -F @proj-airi/stage-web build

# Server
pnpm -F @proj-airi/server build
```

## Environment Variables

### Stage-Web
- `VITE_ENABLE_POSTHOG` - Enable PostHog analytics (set to `true` in production builds)

### Server
- Environment files loaded from `.env` and `.env.local` (optional)
- Database and Redis configuration via docker-compose

## Health Checks

Both services include health checks:
- **API Server**: HTTP GET `/health`
- **Database**: PostgreSQL `pg_isready`
- **Redis**: `redis-cli ping`

## Monitoring

For observability, use the OpenTelemetry setup:

```bash
cd apps/server
docker compose -f docker-compose.otel.yml up -d
```

This provides:
- Jaeger for distributed tracing
- Prometheus for metrics
- Grafana for dashboards