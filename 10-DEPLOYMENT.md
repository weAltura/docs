# Deployment Guide

This document covers deployment procedures, best practices, and production considerations.

## Deployment Overview

The application supports multiple deployment strategies:
- Docker Compose (recommended for small to medium deployments)
- Docker containers (for orchestrated environments)
- Manual deployment (for custom setups)

## Pre-Deployment Checklist

### Environment Configuration

- [ ] Create `.env.production` with production values
- [ ] Set strong `AUTH_SECRET` (generate with `openssl rand -base64 32`)
- [ ] Configure production database credentials
- [ ] Set up Redis with password
- [ ] Configure production email service (not MailPit)
- [ ] Set `APP_LOCAL_FILE_UPLOAD=false` if using S3
- [ ] Configure AWS credentials (if using S3)
- [ ] Set up Sentry DSN for error tracking
- [ ] Configure CORS origins
- [ ] Enable HTTPS (`IS_HTTPS=true`)
- [ ] Set `NODE_ENV=production`

### Security Checklist

- [ ] All secrets are in environment variables (not in code)
- [ ] Database uses SSL in production
- [ ] Redis uses password authentication
- [ ] Rate limiting is enabled
- [ ] CORS origins are restricted
- [ ] HTTPS is enabled
- [ ] Security headers are configured (Helmet)
- [ ] Basic Auth passwords are strong
- [ ] All dependencies are up to date

### Infrastructure Checklist

- [ ] Database is backed up
- [ ] Redis is configured for persistence
- [ ] File storage is configured (S3 or local)
- [ ] Monitoring is set up (Prometheus/Grafana)
- [ ] Logging is configured
- [ ] Error tracking is configured (Sentry)

## Docker Deployment

### Production Docker Compose

The project includes `docker-compose.prod.yml` for production:

```bash
# Build and start production containers
pnpm docker:prod:up

# Stop production containers
pnpm docker:prod:down
```

### Dockerfile

Production Dockerfile uses multi-stage build:

1. **Development stage** - Install all dependencies
2. **Builder stage** - Build application and run migrations
3. **Production stage** - Minimal production image

### Building Images

```bash
# Build API server image
docker build -t wealtura-api:latest .

# Build worker image (same Dockerfile, different IS_WORKER)
docker build -t wealtura-worker:latest .
```

### Running Containers

```bash
# Start API server
docker run -d \
  --name wealtura-api \
  -p 3000:3000 \
  --env-file .env.production \
  wealtura-api:latest

# Start worker
docker run -d \
  --name wealtura-worker \
  --env-file .env.production \
  -e IS_WORKER=true \
  wealtura-worker:latest
```

## Deployment Script

### Automated Deployment

Located in `bin/deploy.sh`:

```bash
#!/bin/bash
set -e

cd "$(dirname "$(realpath "$0")")" || exit
cd ..

git reset --hard HEAD
git pull origin main
docker build --tag wealtura-server:latest . --no-cache
docker build --tag wealtura-worker:latest . --no-cache
pnpm docker:prod:down
pnpm docker:prod:up
docker volume prune -f
docker image prune -f
```

### Usage

```bash
chmod +x bin/deploy.sh
./bin/deploy.sh
```

## Database Migrations

### Running Migrations in Production

Migrations run automatically during Docker build, but you can also run manually:

```bash
# Inside container
docker exec -it wealtura-api pnpm migration:up

# Or directly
pnpm migration:up
```

### Migration Best Practices

1. **Test migrations** - Always test in staging first
2. **Backup database** - Backup before running migrations
3. **Run during maintenance** - Schedule migrations during low traffic
4. **Monitor** - Watch for errors during migration
5. **Rollback plan** - Have rollback strategy ready

## Environment Variables

### Production Environment File

Create `.env.production`:

```bash
# Application
NODE_ENV=production
APP_NAME=Wealtura
APP_URL=https://api.wealtura.com
APP_PORT=3000
IS_HTTPS=true
APP_LOGGING=true
APP_LOG_LEVEL=warn

# Database
DATABASE_HOST=production-db-host
DATABASE_PORT=5432
DATABASE_USERNAME=prod_user
DATABASE_PASSWORD=strong_password
DATABASE_NAME=wealtura_prod
DATABASE_SSL_MODE=require
DATABASE_REJECT_UNAUTHORIZED=true

# Redis
REDIS_HOST=production-redis-host
REDIS_PORT=6379
REDIS_PASSWORD=strong_redis_password

# Auth
AUTH_SECRET=<generate-strong-secret>
BASIC_AUTH_USERNAME=admin
BASIC_AUTH_PASSWORD=<strong-password>

# Mail (Production SMTP)
MAIL_HOST=smtp.production.com
MAIL_PORT=587
MAIL_USER=smtp_user
MAIL_PASSWORD=smtp_password
MAIL_SECURE=true
MAIL_REQUIRE_TLS=true

# AWS (if using S3)
AWS_REGION=us-east-1
AWS_KEY=<aws-key>
AWS_SECRET=<aws-secret>
AWS_S3_BUCKET=wealtura-prod
APP_LOCAL_FILE_UPLOAD=false

# Sentry
SENTRY_DSN=https://...@sentry.io/...
SENTRY_LOGGING=true
```

## Scaling

### Horizontal Scaling

#### API Server

Run multiple API server instances behind a load balancer:

```bash
# Instance 1
docker run -d --name wealtura-api-1 wealtura-api:latest

# Instance 2
docker run -d --name wealtura-api-2 wealtura-api:latest

# Load balancer (nginx, etc.)
upstream api {
  server wealtura-api-1:3000;
  server wealtura-api-2:3000;
}
```

#### Worker

Run multiple worker instances:

```bash
# Worker 1
docker run -d --name wealtura-worker-1 -e IS_WORKER=true wealtura-worker:latest

# Worker 2
docker run -d --name wealtura-worker-2 -e IS_WORKER=true wealtura-worker:latest
```

### Vertical Scaling

Increase resources:
- CPU: More cores for processing
- Memory: More RAM for caching
- Database: Larger connection pool

## Monitoring

### Health Checks

Health check endpoint: `/api/health`

Configure in Docker:
```yaml
healthcheck:
  test: ["CMD", "curl", "-f", "http://localhost:3000/api/health"]
  interval: 30s
  timeout: 10s
  retries: 3
```

### Logging

Production logging:
- Structured JSON logs (Pino)
- Log aggregation (ELK, CloudWatch, etc.)
- Log retention policy

### Metrics

- Prometheus metrics endpoint
- Grafana dashboards
- Alerting rules

## Backup Strategy

### Database Backups

```bash
# Automated daily backup
pg_dump -U postgres -d wealtura_prod > backup_$(date +%Y%m%d).sql

# Restore
psql -U postgres -d wealtura_prod < backup_20240101.sql
```

### File Backups

If using local storage:
- Regular file system backups
- Or migrate to S3 for automatic backups

## Rollback Procedure

### Application Rollback

1. **Stop current containers**
2. **Revert to previous image**
3. **Restart containers**

```bash
docker stop wealtura-api wealtura-worker
docker run -d --name wealtura-api wealtura-api:previous
docker run -d --name wealtura-worker wealtura-worker:previous
```

### Database Rollback

1. **Backup current state**
2. **Run down migration**
3. **Verify data integrity**

```bash
pnpm migration:down
```

## CI/CD Integration

### GitHub Actions

Example workflow:

```yaml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Deploy
        run: |
          ssh user@server './deploy.sh'
```

## Production Best Practices

### Performance

1. **Enable compression** - Already enabled
2. **Use CDN** - For static assets
3. **Database indexes** - Ensure all indexes exist
4. **Caching** - Use Redis caching
5. **Connection pooling** - Configured in TypeORM

### Security

1. **HTTPS only** - Enforce HTTPS
2. **Security headers** - Helmet configured
3. **Rate limiting** - Protect against abuse
4. **Input validation** - All inputs validated
5. **SQL injection** - TypeORM parameterized queries
6. **XSS protection** - Output sanitization

### Reliability

1. **Health checks** - Monitor application health
2. **Graceful shutdown** - Configured
3. **Error handling** - Comprehensive error handling
4. **Retry logic** - For external services
5. **Circuit breakers** - For resilience

## Troubleshooting

### Common Issues

**1. Container won't start**
- Check logs: `docker logs wealtura-api`
- Verify environment variables
- Check port conflicts

**2. Database connection fails**
- Verify database is accessible
- Check credentials
- Verify SSL configuration

**3. High memory usage**
- Check for memory leaks
- Review connection pool size
- Monitor Redis usage

**4. Slow performance**
- Check database queries
- Review indexes
- Monitor Redis cache hit rate

## Next Steps

- Review [Monitoring Guide](./11-MONITORING.md) for production monitoring
- Check [Troubleshooting Guide](./12-TROUBLESHOOTING.md) for common issues
- See [Configuration](./08-CONFIGURATION.md) for production configuration

