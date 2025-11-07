# Monitoring & Observability

This document covers monitoring, logging, and observability tools and practices.

## Monitoring Overview

The application includes comprehensive monitoring:
- **Prometheus** - Metrics collection
- **Grafana** - Metrics visualization
- **Sentry** - Error tracking
- **Structured Logging** - Pino JSON logs

## Prometheus Metrics

### Metrics Endpoint

Prometheus metrics are available at:
- `/metrics` - Prometheus format

### Available Metrics

#### Application Metrics

- HTTP request duration
- HTTP request count
- Error rates
- Active connections

#### System Metrics

- CPU usage
- Memory usage
- Disk usage
- Network I/O

#### Custom Metrics

Add custom business metrics:

```typescript
import { Counter, Histogram } from 'prom-client';

const userCreatedCounter = new Counter({
  name: 'users_created_total',
  help: 'Total number of users created',
});

const requestDuration = new Histogram({
  name: 'http_request_duration_seconds',
  help: 'HTTP request duration in seconds',
});
```

### Prometheus Configuration

Located in `prometheus.config.yml`:

```yaml
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'wealtura-api'
    static_configs:
      - targets: ['wealtura-api:3000']
```

## Grafana Dashboards

### Accessing Grafana

- URL: `http://localhost:3001` (development)
- Username: From `GRAFANA_USERNAME`
- Password: From `GRAFANA_PASSWORD`

### Pre-configured Dashboards

Located in `src/tools/grafana/dashboards/`:

1. **Server Monitoring** - API server metrics
2. **Database Monitoring** - PostgreSQL metrics
3. **Custom Dashboards** - Business metrics

### Dashboard Features

- Real-time metrics
- Historical data
- Alerting rules
- Custom visualizations

## Sentry Error Tracking

### Configuration

Set `SENTRY_DSN` in environment:

```bash
SENTRY_DSN=https://...@sentry.io/...
SENTRY_LOGGING=true
```

### Error Tracking

Sentry automatically tracks:
- Unhandled exceptions
- HTTP errors
- Database errors
- Custom errors

### Custom Error Reporting

```typescript
import * as Sentry from '@sentry/node';

try {
  // Code that might fail
} catch (error) {
  Sentry.captureException(error, {
    tags: { section: 'user-service' },
    extra: { userId: '123' },
  });
  throw error;
}
```

### Release Tracking

Sentry tracks releases for correlation:

```typescript
Sentry.init({
  dsn: process.env.SENTRY_DSN,
  release: process.env.APP_VERSION,
});
```

## Logging

### Structured Logging

Application uses Pino for structured JSON logging:

```typescript
import { Logger } from '@nestjs/common';

private readonly logger = new Logger(UserService.name);

this.logger.log('User created', { userId: '123' });
this.logger.warn('Rate limit approaching', { remaining: 10 });
this.logger.error('Failed to send email', error.stack);
this.logger.debug('Debug information', { data });
```

### Log Levels

- **error** - Errors that need attention
- **warn** - Warnings
- **info** - Informational messages
- **debug** - Debug information
- **verbose** - Verbose logging

### Log Format

Production logs are JSON:

```json
{
  "level": 30,
  "time": 1234567890,
  "msg": "User created",
  "userId": "123",
  "pid": 1234,
  "hostname": "server-1"
}
```

### Log Aggregation

For production, use log aggregation:
- **ELK Stack** (Elasticsearch, Logstash, Kibana)
- **AWS CloudWatch**
- **Google Cloud Logging**
- **Datadog**

## Health Checks

### Health Check Endpoint

Available at `/api/health`:

```json
{
  "status": "ok",
  "info": {
    "database": { "status": "up" },
    "redis": { "status": "up" }
  },
  "error": {},
  "details": {
    "database": { "status": "up" },
    "redis": { "status": "up" }
  }
}
```

### Health Check Usage

Use for:
- Load balancer health checks
- Container orchestration (Kubernetes liveness/readiness)
- Monitoring alerts

## Alerting

### Prometheus Alerts

Configure in Prometheus:

```yaml
groups:
  - name: wealtura
    rules:
      - alert: HighErrorRate
        expr: rate(http_requests_total{status=~"5.."}[5m]) > 0.1
        for: 5m
        annotations:
          summary: "High error rate detected"
```

### Grafana Alerts

Configure in Grafana:
- Alert rules
- Notification channels
- Alert conditions

### Sentry Alerts

Configure in Sentry:
- Error rate thresholds
- Notification rules
- Team assignments

## Performance Monitoring

### Request Tracking

Track request performance:

```typescript
import { Injectable, NestInterceptor, ExecutionContext, CallHandler } from '@nestjs/common';
import { Observable } from 'rxjs';
import { tap } from 'rxjs/operators';

@Injectable()
export class LoggingInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    const start = Date.now();
    return next.handle().pipe(
      tap(() => {
        const duration = Date.now() - start;
        // Log or send to metrics
      }),
    );
  }
}
```

### Database Query Monitoring

Enable query logging:

```bash
DATABASE_LOGGING=true
```

Monitor slow queries and optimize.

### Cache Performance

Monitor cache hit rates:

```typescript
// Track cache hits/misses
cacheHitCounter.inc();
cacheMissCounter.inc();
```

## Monitoring Best Practices

### 1. Key Metrics to Monitor

- **Request Rate** - Requests per second
- **Error Rate** - Error percentage
- **Response Time** - P50, P95, P99
- **Database Connections** - Active connections
- **Cache Hit Rate** - Cache effectiveness
- **Queue Length** - Background job queue
- **Memory Usage** - Application memory
- **CPU Usage** - Application CPU

### 2. Alert Thresholds

Set appropriate thresholds:
- Error rate > 1%
- Response time P95 > 1s
- Memory usage > 80%
- Database connections > 80%

### 3. Dashboard Design

- **Overview Dashboard** - High-level metrics
- **Service Dashboard** - Per-service metrics
- **Infrastructure Dashboard** - System metrics
- **Business Dashboard** - Business metrics

### 4. Log Retention

- **Development**: 7 days
- **Staging**: 30 days
- **Production**: 90+ days

## Troubleshooting

### Metrics Not Appearing

1. **Check Prometheus** - Verify Prometheus is scraping
2. **Check endpoint** - Verify `/metrics` is accessible
3. **Check configuration** - Review Prometheus config

### Logs Not Appearing

1. **Check log level** - Verify `APP_LOG_LEVEL`
2. **Check output** - Verify logs are being written
3. **Check aggregation** - Verify log aggregation is working

### Errors Not in Sentry

1. **Check DSN** - Verify `SENTRY_DSN` is set
2. **Check initialization** - Verify Sentry is initialized
3. **Check network** - Verify Sentry endpoint is accessible

## Next Steps

- Review [Deployment Guide](./10-DEPLOYMENT.md) for production monitoring
- Check [Troubleshooting Guide](./12-TROUBLESHOOTING.md) for monitoring issues
- See [Configuration](./08-CONFIGURATION.md) for monitoring configuration

