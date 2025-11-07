# Getting Started

This guide will help you set up the Wealtura Server project on your local machine for development.

## Prerequisites

Before you begin, ensure you have the following installed:

### Required Software

1. **Node.js** (v20 or higher)
   - Download from [nodejs.org](https://nodejs.org/)
   - Verify installation: `node --version`

2. **pnpm** (v9.12.0 or higher)
   - Install globally: `npm install -g pnpm@9.12.2`
   - Verify installation: `pnpm --version`
   - The project uses pnpm as the package manager

3. **Docker** and **Docker Compose**
   - Download from [docker.com](https://www.docker.com/)
   - Verify installation: `docker --version` and `docker compose version`
   - Required for running PostgreSQL, Redis, and MailPit services

4. **Git**
   - Download from [git-scm.com](https://git-scm.com/)
   - Verify installation: `git --version`

### Optional Software

- **VS Code** or your preferred IDE
- **PostgreSQL Client** (pgAdmin, DBeaver, etc.) for database inspection
- **Redis CLI** for Redis inspection

## Initial Setup

### 1. Clone the Repository

```bash
git clone <repository-url>
cd wealtura-server
```

### 2. Install Dependencies

```bash
pnpm install
```

This will install all project dependencies including dev dependencies.

### 3. Environment Configuration

#### Step 1: Copy Environment Files

```bash
# Copy the example environment file
cp .env.example .env.local

# Copy Docker environment file
cp .env.docker.example .env.docker

# Optional: Create development defaults
cp .env.example .env.development
```

#### Step 2: Configure Environment Variables

Edit `.env.local` and set the following required variables:

**Application Configuration:**
```bash
NODE_ENV=development
APP_NAME=Wealtura
APP_PORT=3000
APP_WORKER_PORT=3001
APP_URL=http://localhost:3000
```

**Database Configuration:**
```bash
DATABASE_HOST=localhost
DATABASE_PORT=5432
DATABASE_USERNAME=postgres
DATABASE_PASSWORD=your_secure_password
DATABASE_NAME=wealtura
DATABASE_LOGGING=true
```

**Redis Configuration:**
```bash
REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_PASSWORD=your_redis_password
```

**Authentication:**
```bash
# Generate a strong secret: openssl rand -base64 32
AUTH_SECRET=your_generated_auth_secret_here
BASIC_AUTH_USERNAME=admin
BASIC_AUTH_PASSWORD=admin
```

**Mail Configuration (for MailPit):**
```bash
MAIL_HOST=localhost
MAIL_PORT=1025
MAIL_IGNORE_TLS=true
MAIL_SECURE=false
MAIL_DEFAULT_EMAIL=noreply@wealtura.local
MAIL_DEFAULT_NAME=Wealtura Dev
```

**Important**:
- Never commit `.env.local` or any `.env.*` files (except `.env.example` and `.env.docker.example`)
- Generate strong passwords and secrets for production
- See [Environment Setup](./ENV_SETUP.md) for complete configuration reference

#### Step 3: Configure Docker Ports

Edit `.env.docker` and ensure ports match your `.env.local`:

```bash
DOCKER_DATABASE_PORT=5432
DOCKER_REDIS_PORT=6379
DOCKER_MAIL_PORT=1025
DOCKER_MAIL_CLIENT_PORT=8025
```

### 4. Start Docker Services

Start PostgreSQL, Redis, and MailPit using Docker Compose:

```bash
# Start services
pnpm docker:services:up

# Check service status
docker ps

# View service logs
pnpm docker:services:logs
```

Wait for services to be healthy (about 10-15 seconds).

### 5. Run Database Migrations

```bash
# Run all pending migrations
pnpm migration:up

# Check migration status
pnpm migration:show
```

### 6. (Optional) Run Database Seeds

```bash
# Seed initial data
pnpm seed:run
```

### 7. Start Development Server

You have two options:

#### Option A: Combined Command (Recommended)
```bash
# Starts Docker services and development server
pnpm dev
```

#### Option B: Separate Commands
```bash
# Terminal 1: Start Docker services (if not already running)
pnpm docker:services:up

# Terminal 2: Start development server
pnpm start:dev
```

The development server will:
- Start the NestJS API server with hot reload
- Watch and build email templates
- Be available at `http://localhost:3000`

## Verify Installation

### 1. Check API Health

```bash
curl http://localhost:3000/api/health
```

Expected response:
```json
{
  "status": "ok",
  "info": { ... },
  "error": {},
  "details": { ... }
}
```

### 2. Access Swagger Documentation

Open in browser: `http://localhost:3000/api`

You'll be prompted for Basic Auth credentials (from `BASIC_AUTH_USERNAME` and `BASIC_AUTH_PASSWORD`).

### 3. Access MailPit (Email Testing)

Open in browser: `http://localhost:8025`

This is where you can view all emails sent during development.

### 4. Access GraphQL Playground

Open in browser: `http://localhost:3000/api/graphql`

### 5. Access Bull Board (Queue Monitoring)

Open in browser: `http://localhost:3000/api/queues`

Protected with Basic Auth.

## Development Workflow

### Daily Development

1. **Start Services** (if not running):
   ```bash
   pnpm docker:services:up
   ```

2. **Start Development Server**:
   ```bash
   pnpm start:dev
   ```

3. **Make Changes**: Edit files in `src/` - changes will hot reload automatically

4. **Run Tests** (when needed):
   ```bash
   pnpm test
   ```

5. **Stop Services** (when done):
   ```bash
   pnpm docker:services:down
   ```

### Common Tasks

#### Database Migrations

```bash
# Create a new migration
pnpm migration:create src/database/migrations/YourMigrationName

# Generate migration from entity changes
pnpm migration:generate src/database/migrations/YourMigrationName

# Run migrations
pnpm migration:up

# Revert last migration
pnpm migration:down

# Show migration status
pnpm migration:show
```

#### Database Seeds

```bash
# Create a new seed
pnpm seed:create src/database/seeds/YourSeedName

# Run all seeds
pnpm seed:run
```

#### Code Quality

```bash
# Lint code
pnpm lint

# Format code
pnpm format

# Run tests
pnpm test

# Run tests with coverage
pnpm test:cov
```

#### Email Development

```bash
# Start React Email dev server (preview templates)
pnpm email:dev

# Build email templates
pnpm email:build

# Watch email templates (auto-build on changes)
pnpm email:watch
```

## Project Scripts Reference

### Development Scripts

| Script | Description |
|--------|-------------|
| `pnpm dev` | Start Docker services and development server |
| `pnpm start:dev` | Start development server with hot reload |
| `pnpm start:debug` | Start development server with debugger |
| `pnpm build` | Build production bundle |

### Docker Scripts

| Script | Description |
|--------|-------------|
| `pnpm docker:services:up` | Start only services (DB, Redis, Mail) |
| `pnpm docker:services:down` | Stop services |
| `pnpm docker:services:logs` | View service logs |
| `pnpm docker:dev:up` | Start everything in Docker (dev mode) |
| `pnpm docker:dev:down` | Stop Docker dev environment |
| `pnpm docker:prod:up` | Start production Docker environment |
| `pnpm docker:prod:down` | Stop production Docker environment |
| `pnpm dev:clean` | Stop services and remove volumes |

### Database Scripts

| Script | Description |
|--------|-------------|
| `pnpm migration:up` | Run pending migrations |
| `pnpm migration:down` | Revert last migration |
| `pnpm migration:show` | Show migration status |
| `pnpm migration:create` | Create new migration file |
| `pnpm migration:generate` | Generate migration from entities |
| `pnpm seed:run` | Run database seeds |
| `pnpm seed:create` | Create new seed file |

### Testing Scripts

| Script | Description |
|--------|-------------|
| `pnpm test` | Run unit tests |
| `pnpm test:watch` | Run tests in watch mode |
| `pnpm test:cov` | Run tests with coverage |
| `pnpm test:e2e` | Run end-to-end tests |
| `pnpm test:debug` | Run tests with debugger |

### Code Quality Scripts

| Script | Description |
|--------|-------------|
| `pnpm lint` | Lint code with ESLint |
| `pnpm format` | Format code with Prettier |

### Utility Scripts

| Script | Description |
|--------|-------------|
| `pnpm graph:app` | Generate dependency graph |
| `pnpm graph:circular` | Find circular dependencies |
| `pnpm erd:generate` | Generate Entity Relationship Diagram |

## IDE Setup

### VS Code

Recommended extensions:
- **ESLint** - Code linting
- **Prettier** - Code formatting
- **TypeScript and JavaScript Language Features** - TypeScript support
- **Database Client** - Database management
- **Docker** - Docker support

VS Code settings (`.vscode/settings.json`):
```json
{
  "editor.formatOnSave": true,
  "editor.defaultFormatter": "esbenp.prettier-vscode",
  "editor.codeActionsOnSave": {
    "source.fixAll.eslint": true
  },
  "typescript.tsdk": "node_modules/typescript/lib",
  "typescript.enablePromptUseWorkspaceTsdk": true
}
```

### Debugging

The project includes VS Code debug configurations. Press `F5` to start debugging.

Debug configurations are available for:
- **Launch Program** - Debug the main application
- **Attach to Process** - Attach to running process
- **Debug Jest Tests** - Debug unit tests

## Troubleshooting

### Port Already in Use

If you get "port already in use" errors:

1. **Change ports** in `.env.docker` and `.env.local`
2. **Or stop** the process using the port:
   ```bash
   # Find process using port 3000
   lsof -i :3000

   # Kill the process
   kill -9 <PID>
   ```

### Docker Services Not Starting

1. **Check Docker is running**: `docker ps`
2. **Check logs**: `pnpm docker:services:logs`
3. **Check ports**: Ensure ports in `.env.docker` are available
4. **Restart Docker**: Restart Docker Desktop

### Database Connection Issues

1. **Verify Docker service is running**: `docker ps | grep postgres`
2. **Check connection string** in `.env.local`
3. **Verify credentials** match Docker environment variables
4. **Check database logs**: `docker logs wealtura-postgres`

### Module Not Found Errors

1. **Reinstall dependencies**: `rm -rf node_modules && pnpm install`
2. **Clear build cache**: `rm -rf dist`
3. **Rebuild**: `pnpm build`

### TypeScript Errors

1. **Restart TypeScript server** in VS Code (Cmd+Shift+P → "TypeScript: Restart TS Server")
2. **Check tsconfig.json** is correct
3. **Verify all dependencies** are installed

## Next Steps

- Read [Development Workflow](./02-DEVELOPMENT-WORKFLOW.md) for development practices
- Review [Architecture Overview](./00-OVERVIEW.md) to understand the system
- Check [API Documentation](./06-API.md) for API usage
- See [Environment Setup](./ENV_SETUP.md) for detailed environment configuration

## Getting Help

- Check [Troubleshooting Guide](./12-TROUBLESHOOTING.md) for common issues
- Review existing code and documentation
- Ask team members for assistance
- Check NestJS documentation: https://docs.nestjs.com/

