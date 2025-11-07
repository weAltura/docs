# Configuration Reference

Complete reference for all configuration options in the application.

## Configuration System

The application uses NestJS ConfigModule with environment-based configuration loading.

### Configuration Loading

Configuration files are loaded in priority order:
1. `.env.{NODE_ENV}` (e.g., `.env.development`)
2. `.env.local` (highest priority, gitignored)
3. `.env` (fallback)

See [Environment Setup](./ENV_SETUP.md) for details.

## Application Configuration

### Environment Variables

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `NODE_ENV` | enum | `development` | Environment: `local`, `development`, `staging`, `production`, `test` |
| `APP_NAME` | string | required | Application name |
| `APP_URL` | string | `http://localhost:3000` | Base URL for absolute URLs |
| `APP_PORT` | number | `3000` | HTTP server port |
| `APP_WORKER_PORT` | number | `3001` | Worker process port |
| `IS_HTTPS` | boolean | `false` | Enable HTTPS |
| `IS_WORKER` | boolean | `false` | Run as worker process |
| `APP_DEBUG` | boolean | `false` | Enable debug mode |
| `APP_LOGGING` | boolean | `true` | Enable application logging |
| `APP_LOG_LEVEL` | string | `warn` | Log level: `error`, `warn`, `info`, `debug`, `verbose` |
| `APP_LOG_SERVICE` | enum | `console` | Log service: `console`, `google-logging`, `aws-cloudwatch` |
| `APP_CORS_ORIGIN` | string | `false` | CORS origins (comma-separated or `true` for all) |
| `APP_FALLBACK_LANGUAGE` | string | `en` | Default language for i18n |
| `APP_LOCAL_FILE_UPLOAD` | boolean | `true` | Use local file storage (false = AWS S3) |

## Database Configuration

### Environment Variables

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `DATABASE_HOST` | string | required | Database host |
| `DATABASE_PORT` | number | `5432` | Database port |
| `DATABASE_USERNAME` | string | required | Database username |
| `DATABASE_PASSWORD` | string | required | Database password |
| `DATABASE_NAME` | string | required | Database name |
| `DATABASE_LOGGING` | boolean | `false` | Enable query logging |
| `DATABASE_MAX_CONNECTIONS` | number | `100` | Maximum connection pool size |
| `DATABASE_SSL_MODE` | enum | `disable` | SSL mode: `disable`, `require` |
| `DATABASE_REJECT_UNAUTHORIZED` | boolean | `false` | Reject unauthorized SSL certificates |
| `DATABASE_CA` | string | - | SSL CA certificate |
| `DATABASE_KEY` | string | - | SSL key |
| `DATABASE_CERT` | string | - | SSL certificate |
| `DATABASE_URL` | string | - | Alternative: Full database URL |

## Redis Configuration

### Environment Variables

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `REDIS_HOST` | string | required | Redis host |
| `REDIS_PORT` | number | `6379` | Redis port |
| `REDIS_PASSWORD` | string | - | Redis password |
| `REDIS_TLS` | boolean | `false` | Enable TLS |
| `REDIS_REJECT_UNAUTHORIZED` | boolean | `false` | Reject unauthorized TLS certificates |
| `REDIS_CA` | string | - | TLS CA certificate |
| `REDIS_KEY` | string | - | TLS key |
| `REDIS_CERT` | string | - | TLS certificate |

## Authentication Configuration

### Environment Variables

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `AUTH_SECRET` | string | required | Secret for signing cookies/tokens (min 32 chars) |
| `BASIC_AUTH_USERNAME` | string | `admin` | Basic Auth username (Swagger, Bull Board) |
| `BASIC_AUTH_PASSWORD` | string | required | Basic Auth password |
| `GITHUB_CLIENT_ID` | string | - | GitHub OAuth client ID |
| `GITHUB_CLIENT_SECRET` | string | - | GitHub OAuth client secret |

## Mail Configuration

### Environment Variables

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `MAIL_HOST` | string | required | SMTP host |
| `MAIL_PORT` | number | `587` | SMTP port |
| `MAIL_USER` | string | - | SMTP username |
| `MAIL_PASSWORD` | string | - | SMTP password |
| `MAIL_IGNORE_TLS` | boolean | `false` | Ignore TLS errors (for MailPit) |
| `MAIL_SECURE` | boolean | `false` | Use SSL/TLS |
| `MAIL_REQUIRE_TLS` | boolean | `false` | Require TLS |
| `MAIL_DEFAULT_EMAIL` | string | required | Default sender email |
| `MAIL_DEFAULT_NAME` | string | required | Default sender name |

## Queue Configuration

### Environment Variables

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `QUEUE_REMOVE_ON_COMPLETE` | boolean | `true` | Remove completed jobs |
| `QUEUE_REMOVE_ON_FAIL` | boolean | `false` | Remove failed jobs |
| `QUEUE_FAILED_RETRY_ATTEMPTS` | number | `3` | Number of retry attempts |

## Throttler Configuration

### Environment Variables

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `THROTTLER_ENABLED` | boolean | `true` | Enable rate limiting |
| `THROTTLER_LIMIT` | number | `100` | Maximum requests per window |
| `THROTTLER_TTL` | number | `60` | Time window in seconds |

## AWS Configuration

### Environment Variables

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `AWS_REGION` | string | - | AWS region |
| `AWS_KEY` | string | - | AWS access key ID |
| `AWS_SECRET` | string | - | AWS secret access key |
| `AWS_S3_BUCKET` | string | - | S3 bucket name |

**Note**: Required only if `APP_LOCAL_FILE_UPLOAD=false`

## Sentry Configuration

### Environment Variables

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `SENTRY_DSN` | string | - | Sentry DSN (from sentry.io) |
| `SENTRY_LOGGING` | boolean | `false` | Enable Sentry logging |

## Grafana Configuration

### Environment Variables

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `GRAFANA_USERNAME` | string | `admin` | Grafana admin username |
| `GRAFANA_PASSWORD` | string | `admin` | Grafana admin password |

**Note**: Used only when monitoring profile is enabled

## Docker Configuration

### Environment Variables (`.env.docker`)

| Variable | Type | Default | Description |
|----------|------|---------|-------------|
| `DOCKER_DATABASE_PORT` | number | `5432` | PostgreSQL exposed port |
| `DOCKER_REDIS_PORT` | number | `6379` | Redis exposed port |
| `DOCKER_MAIL_PORT` | number | `1025` | MailPit SMTP port |
| `DOCKER_MAIL_CLIENT_PORT` | number | `8025` | MailPit Web UI port |
| `DOCKER_PROMETHEUS_PORT` | number | `9090` | Prometheus port |
| `DOCKER_GRAFANA_PORT` | number | `3001` | Grafana port |
| `DOCKER_PG_EXPORTER` | number | `9187` | PostgreSQL exporter port |

## Configuration Access

### In Services

```ts
import { ConfigService } from '@nestjs/config';

@Injectable()
export class MyService {
  constructor(private configService: ConfigService) {}

  getConfig() {
    // Type-safe access
    const dbConfig = this.configService.getOrThrow('database', { infer: true });

    // Or direct access
    const port = this.configService.get('APP_PORT');
  }
}
```

### Type-Safe Configuration

Configuration is typed via `GlobalConfig`:

```ts
import { GlobalConfig } from '@/config/config.type';

const config = configService.getOrThrow<GlobalConfig['database']>('database', {
  infer: true,
});
```

## Configuration Validation

All configuration is validated on startup:

```ts
// Validators use class-validator
class EnvironmentVariablesValidator {
  @IsString()
  @IsNotEmpty()
  DATABASE_HOST: string;

  @IsInt()
  @Min(0)
  @Max(65535)
  DATABASE_PORT: number;
}
```

Invalid configuration will prevent application startup.

## Environment-Specific Configuration

### Development

```bash
NODE_ENV=development
APP_LOGGING=true
APP_LOG_LEVEL=info
DATABASE_LOGGING=true
```

### Production

```bash
NODE_ENV=production
APP_LOGGING=true
APP_LOG_LEVEL=warn
DATABASE_LOGGING=false
APP_LOCAL_FILE_UPLOAD=false  # Use S3
```

### Test

```bash
NODE_ENV=test
APP_LOGGING=false
THROTTLER_ENABLED=false
```

## Security Considerations

### Secrets

- **Never commit secrets** - Use `.env.local` (gitignored)
- **Rotate secrets regularly** - Especially `AUTH_SECRET`
- **Use strong passwords** - Generate with `openssl rand -base64 32`
- **Limit access** - Restrict who can view production secrets

### Production Checklist

- [ ] Strong `AUTH_SECRET` (min 32 characters)
- [ ] Secure database passwords
- [ ] Redis password set
- [ ] HTTPS enabled (`IS_HTTPS=true`)
- [ ] CORS origins restricted
- [ ] Rate limiting enabled
- [ ] Sentry configured for error tracking
- [ ] AWS credentials configured (if using S3)
- [ ] Database SSL enabled (if required)

## Configuration Files

### Configuration Modules

Located in `src/config/`:

- `app/` - Application configuration
- `auth/` - Authentication configuration
- `aws/` - AWS configuration
- `bull/` - Queue configuration
- `database/` - Database configuration
- `grafana/` - Grafana configuration
- `mail/` - Mail configuration
- `redis/` - Redis configuration
- `sentry/` - Sentry configuration
- `throttler/` - Rate limiter configuration

### Configuration Types

Each configuration module exports:
- Configuration function (`getConfig()`)
- Type definition (`*Config`)
- Validator class (`EnvironmentVariablesValidator`)

## Troubleshooting

### Configuration Not Loading

1. **Check file exists**: Verify `.env.local` exists
2. **Check variable names**: Ensure exact match (case-sensitive)
3. **Check validation**: Review validation errors on startup
4. **Check priority**: Verify `.env.local` overrides base config

### Type Errors

1. **Check types**: Verify `GlobalConfig` types
2. **Check inference**: Use `{ infer: true }` option
3. **Check imports**: Import from correct config module

## Next Steps

- Review [Environment Setup](./ENV_SETUP.md) for environment configuration
- Check [Getting Started](./01-GETTING-STARTED.md) for initial setup
- See [Deployment Guide](./10-DEPLOYMENT.md) for production configuration

