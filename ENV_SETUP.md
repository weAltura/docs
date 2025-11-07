# Environment Configuration Guide

This project uses a multi-environment file setup following best practices for managing configuration across different environments.

## Environment File Structure

The project uses the following environment files (loaded in priority order):

1. **`.env.{NODE_ENV}`** - Base environment file (e.g., `.env.development`, `.env.test`, `.env.production`)
2. **`.env.local`** - Personal/local overrides (gitignored, highest priority)
3. **`.env`** - Fallback for backward compatibility

### File Overview

| File | Purpose | Committed? | Priority |
|------|---------|------------|----------|
| `.env.example` | Template with all variables documented | ✅ Yes | - |
| `.env.development` | Development defaults (create from `.env.example`) | ❌ No | 1st |
| `.env.test` | Test environment defaults (create from `.env.example`) | ❌ No | 1st (when NODE_ENV=test) |
| `.env.local` | Personal overrides | ❌ No | 2nd (highest) |
| `.env.docker.example` | Docker ports template | ✅ Yes | - |
| `.env.docker` | Docker-specific settings | ❌ No | - |

## Quick Start

### 1. Initial Setup

```bash
# Copy example files
cp .env.example .env.local
cp .env.example .env.development  # Optional: for development defaults
cp .env.docker.example .env.docker

# Edit .env.local with your values (this overrides .env.development)
# Edit .env.development if you want team-wide development defaults
# Edit .env.docker if you need to change Docker ports
```

**Note**: All `.env.*` files (except `.env.example` and `.env.docker.example`) are gitignored. Create them locally as needed.

### 2. Development Workflow (Node Local + Services Docker)

```bash
# Start Docker services (PostgreSQL, Redis, MailPit)
pnpm docker:services:up

# In another terminal, start the Node.js server
pnpm start:dev

# Or use the combined command
pnpm dev
```

### 3. Stop Services

```bash
# Stop services (keeps data)
pnpm docker:services:down

# Stop and remove volumes (fresh start)
pnpm dev:clean
```

## Environment Files Explained

### `.env.example`
- **Purpose**: Complete template with all possible environment variables
- **Contains**: All variables with documentation and example values
- **Usage**: Reference for what variables are available
- **Action**: Copy to `.env.local` and fill in your values

### `.env.development`
- **Purpose**: Development environment defaults
- **Contains**: Sensible defaults for local development
- **Usage**: Automatically loaded when `NODE_ENV=development`
- **Action**: Create from `.env.example`, **DO NOT COMMIT**. Override values in `.env.local` if needed

### `.env.test`
- **Purpose**: Test environment configuration
- **Contains**: Test-specific settings (logging disabled, test database, etc.)
- **Usage**: Automatically loaded when `NODE_ENV=test`
- **Action**: Create from `.env.example`, **DO NOT COMMIT**

### `.env.local`
- **Purpose**: Personal/local overrides
- **Contains**: Your personal settings (passwords, API keys, etc.)
- **Usage**: Highest priority - overrides all other env files
- **Action**: Create from `.env.example`, **DO NOT COMMIT**

### `.env.docker.example` & `.env.docker`
- **Purpose**: Docker container port configuration
- **Contains**: Port mappings for Docker services
- **Usage**: Used by docker-compose files
- **Action**: Copy `.env.docker.example` to `.env.docker` and adjust if needed

## Configuration Priority

When the application starts, environment variables are loaded in this order (later files override earlier ones):

1. `.env.development` (or `.env.test` based on `NODE_ENV`)
2. `.env.local` (your personal overrides)
3. `.env` (fallback)

**Example**: If `DATABASE_PASSWORD` is set in both `.env.development` and `.env.local`, the value from `.env.local` will be used.

## Environment Variables by Category

### Application
- `NODE_ENV` - Environment: `local`, `development`, `staging`, `production`, `test`
- `APP_NAME` - Application name
- `APP_PORT` - Server port (default: 3000)
- `APP_WORKER_PORT` - Worker port (default: 3001)
- `APP_URL` - Base URL for generating absolute URLs
- `APP_CORS_ORIGIN` - CORS allowed origins
- `APP_LOGGING` - Enable/disable logging
- `APP_LOG_LEVEL` - Log level: `error`, `warn`, `info`, `debug`, `verbose`

### Database
- `DATABASE_HOST` - Database host (use `localhost` for Docker)
- `DATABASE_PORT` - Database port (should match `DOCKER_DATABASE_PORT`)
- `DATABASE_USERNAME` - Database username
- `DATABASE_PASSWORD` - Database password
- `DATABASE_NAME` - Database name
- `DATABASE_LOGGING` - Enable query logging

### Redis
- `REDIS_HOST` - Redis host (use `localhost` for Docker)
- `REDIS_PORT` - Redis port (should match `DOCKER_REDIS_PORT`)
- `REDIS_PASSWORD` - Redis password

### Authentication
- `AUTH_SECRET` - Secret for signing cookies/tokens (generate with: `openssl rand -base64 32`)
- `BASIC_AUTH_USERNAME` - Username for Swagger/Bull Board protection
- `BASIC_AUTH_PASSWORD` - Password for Swagger/Bull Board protection

### Mail
- `MAIL_HOST` - SMTP host (use `localhost` for MailPit)
- `MAIL_PORT` - SMTP port (should match `DOCKER_MAIL_PORT` for MailPit)
- `MAIL_DEFAULT_EMAIL` - Default sender email
- `MAIL_DEFAULT_NAME` - Default sender name

### Docker Ports (`.env.docker`)
- `DOCKER_DATABASE_PORT` - PostgreSQL port (default: 5432)
- `DOCKER_REDIS_PORT` - Redis port (default: 6379)
- `DOCKER_MAIL_PORT` - MailPit SMTP port (default: 1025)
- `DOCKER_MAIL_CLIENT_PORT` - MailPit Web UI port (default: 8025)

See `.env.example` for the complete list of all variables.

## Development Workflows

### Workflow 1: Node Local + Services Docker (Recommended)

**Best for**: Fast development with hot reload

```bash
# 1. Start Docker services
pnpm docker:services:up

# 2. Start Node.js locally (in another terminal)
pnpm start:dev

# Or use combined command
pnpm dev
```

**Benefits**:
- ✅ Fast hot reload
- ✅ Easy debugging
- ✅ Services isolated in Docker
- ✅ Quick restarts (only Node, not services)

### Workflow 2: Everything in Docker

**Best for**: Testing Docker setup or production-like environment

```bash
# Start everything in Docker
pnpm docker:dev:up

# Stop everything
pnpm docker:dev:down
```

**Benefits**:
- ✅ Everything containerized
- ✅ Consistent environment
- ✅ Easy to reset

## Testing

When running tests, the application automatically loads `.env.test`:

```bash
# Run tests (uses .env.test)
pnpm test

# Run e2e tests
pnpm test:e2e
```

## Production

For production, create `.env.production` (do not commit):

```bash
# Copy example
cp .env.example .env.production

# Edit with production values
# Then deploy
pnpm docker:prod:up
```

## Troubleshooting

### Services not starting
- Check that `.env.docker` exists and has correct values
- Verify ports aren't already in use: `lsof -i :5432`

### Connection refused
- Ensure services are running: `docker ps`
- Check that `DATABASE_PORT` matches `DOCKER_DATABASE_PORT`
- Verify `DATABASE_HOST=localhost` (not `127.0.0.1`)

### Wrong environment loaded
- Check `NODE_ENV` value
- Verify `.env.{NODE_ENV}` file exists
- Check file loading order in `src/app.module.ts`

### Port conflicts
- Update ports in `.env.docker`
- Update corresponding ports in `.env.local`
- Restart Docker services

## Best Practices

1. **Never commit any `.env` files**: Only commit `.env.example` and `.env.docker.example` as templates
2. **Use `.env.local` for personal overrides**: This file is gitignored and has highest priority
3. **Create `.env.development` locally**: Copy from `.env.example` and customize for your team
4. **Document new variables**: Add them to `.env.example` with comments
5. **Use strong secrets**: Generate with `openssl rand -base64 32`
6. **Match Docker ports**: Ensure `DATABASE_PORT` matches `DOCKER_DATABASE_PORT`
7. **Share defaults via documentation**: If you have team-wide defaults, document them in `ENV_SETUP.md` or team docs

## Migration from Old Setup

If you're migrating from the old single `.env` file:

1. Copy your current `.env` to `.env.local`
2. Remove any values that are now in `.env.development`
3. Keep only your personal overrides in `.env.local`
4. Delete the old `.env` file (or let it be a fallback)

The new setup is backward compatible - if `.env` exists, it will still be loaded as a fallback.

