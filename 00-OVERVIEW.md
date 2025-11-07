# Project Overview

## Introduction

Wealtura Server is a comprehensive NestJS-based backend application designed for scalable startups. It provides a robust foundation with modern best practices, including authentication, database management, API development, background job processing, and monitoring capabilities.

## Technology Stack

### Core Framework
- **NestJS** (v10.4.4) - Progressive Node.js framework for building efficient and scalable server-side applications
- **Fastify** (v4.28.1) - Fast and low overhead web framework (used as HTTP adapter)
- **TypeScript** (v5.6.3) - Typed superset of JavaScript

### Database & ORM
- **PostgreSQL** (v16) - Primary relational database
- **TypeORM** (v0.3.20) - Object-Relational Mapping framework
- **Redis** (v7.0.1) - In-memory data structure store for caching and sessions

### Authentication
- **Better Auth** (v1.2.7) - Complete authentication solution supporting:
  - Email/Password authentication
  - OAuth (GitHub)
  - Magic Link
  - Pass Keys
  - Two-Factor Authentication (2FA)
  - Session Management

### API Technologies
- **REST API** - Traditional RESTful endpoints with Swagger documentation
- **GraphQL** - GraphQL API using Apollo Server
- **WebSocket** - Real-time communication using Socket.io with Redis adapter

### Background Jobs
- **BullMQ** (v5.29.1) - Redis-based queue system for background job processing
- **Bull Board** - Web UI for monitoring and managing queues

### Email
- **React Email** - React-based email template system
- **Nodemailer** - Email sending library
- **MailPit** - Local SMTP server for development email testing

### Caching & Performance
- **Redis** - Caching layer
- **Cache Manager** - Cache abstraction layer

### Monitoring & Observability
- **Prometheus** - Metrics collection
- **Grafana** - Metrics visualization and dashboards
- **Sentry** - Error tracking and monitoring
- **Pino** - Fast JSON logger

### File Storage
- **AWS S3** - Cloud file storage (optional)
- **Local Storage** - Local file storage for development

### Internationalization
- **nestjs-i18n** - Internationalization support with multiple language support

### Rate Limiting
- **@nestjs/throttler** - Rate limiting middleware with Redis storage

### Development Tools
- **SWC** - Fast TypeScript/JavaScript compiler (replaces Webpack)
- **ESLint** - Code linting
- **Prettier** - Code formatting
- **Husky** - Git hooks
- **Commitlint** - Commit message linting
- **Jest** - Testing framework

## Project Structure

```
wealtura-server/
├── src/
│   ├── api/              # API endpoints (REST controllers)
│   │   ├── file/         # File upload endpoints
│   │   ├── health/       # Health check endpoints
│   │   └── user/         # User management endpoints
│   ├── auth/             # Authentication module
│   │   ├── entities/     # Auth-related database entities
│   │   ├── auth.guard.ts # Authentication guard
│   │   └── auth.module.ts
│   ├── common/           # Shared DTOs and types
│   │   ├── dto/          # Data Transfer Objects
│   │   └── types/        # Common TypeScript types
│   ├── config/           # Configuration modules
│   │   ├── app/          # Application configuration
│   │   ├── auth/         # Authentication configuration
│   │   ├── aws/          # AWS configuration
│   │   ├── bull/         # BullMQ configuration
│   │   ├── database/     # Database configuration
│   │   ├── grafana/      # Grafana configuration
│   │   ├── mail/         # Mail configuration
│   │   ├── redis/        # Redis configuration
│   │   ├── sentry/       # Sentry configuration
│   │   └── throttler/    # Rate limiter configuration
│   ├── constants/        # Application constants
│   ├── database/         # Database-related code
│   │   ├── migrations/   # Database migrations
│   │   ├── seeds/        # Database seeds
│   │   └── models/       # Base database models
│   ├── decorators/       # Custom decorators
│   │   ├── auth/         # Auth-related decorators
│   │   └── validators/   # Custom validators
│   ├── generated/        # Generated files (i18n, GraphQL schema)
│   ├── graphql/          # GraphQL configuration
│   ├── i18n/             # Internationalization
│   │   └── translations/ # Translation files
│   ├── interceptors/     # Request/response interceptors
│   ├── middlewares/      # Custom middlewares
│   ├── services/         # External service integrations
│   │   ├── aws/          # AWS services
│   │   └── gcp/          # Google Cloud services
│   ├── shared/           # Shared modules
│   │   ├── cache/        # Caching module
│   │   ├── mail/         # Mail module
│   │   │   └── templates/ # Email templates (React)
│   │   └── socket/       # WebSocket module
│   ├── tools/            # Development tools
│   │   ├── grafana/      # Grafana dashboards
│   │   ├── logger/       # Logger configuration
│   │   └── swagger/      # Swagger configuration
│   ├── utils/            # Utility functions
│   ├── worker/           # Background worker module
│   │   └── queues/       # Queue processors
│   ├── app.module.ts     # Root application module
│   └── main.ts           # Application entry point
├── test/                 # End-to-end tests
├── scripts/              # Utility scripts
├── docker-compose*.yml   # Docker Compose configurations
├── Dockerfile            # Production Dockerfile
├── Dockerfile.dev        # Development Dockerfile
└── package.json          # Dependencies and scripts
```

## Key Features

### 1. Multi-Protocol API Support
- **REST API**: Traditional REST endpoints with versioning
- **GraphQL API**: Flexible GraphQL queries and mutations
- **WebSocket**: Real-time bidirectional communication

### 2. Comprehensive Authentication
- Multiple authentication methods (email/password, OAuth, magic link, passkeys)
- Two-factor authentication support
- Session management
- Role-based access control ready

### 3. Database Management
- TypeORM for database operations
- Migration system for schema changes
- Seed system for initial data
- Support for both offset and cursor-based pagination

### 4. Background Job Processing
- BullMQ for reliable job queuing
- Separate worker process for job execution
- Bull Board UI for job monitoring
- Email queue for asynchronous email sending

### 5. Email System
- React Email for template development
- Handlebars for template rendering
- MailPit for local development testing
- Queue-based email sending

### 6. Caching Strategy
- Redis-based caching
- Cache manager abstraction
- Configurable TTL and cache keys

### 7. Monitoring & Observability
- Prometheus metrics
- Grafana dashboards
- Sentry error tracking
- Structured logging with Pino

### 8. Development Experience
- Hot reload with SWC
- Comprehensive TypeScript support
- Code quality tools (ESLint, Prettier)
- Git hooks for code quality
- Dependency graph visualization
- ERD generation

## Application Architecture

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Client Applications                    │
│              (Web, Mobile, Desktop)                       │
└────────────────────┬──────────────────────────────────────┘
                     │
                     │ HTTP/WebSocket
                     │
┌────────────────────▼──────────────────────────────────────┐
│                  API Server (NestJS)                        │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐               │
│  │   REST   │  │ GraphQL  │  │ WebSocket │               │
│  └──────────┘  └──────────┘  └──────────┘               │
│                                                             │
│  ┌──────────────────────────────────────────────┐          │
│  │         Application Modules                   │          │
│  │  - Auth Module                                │          │
│  │  - User Module                                │          │
│  │  - File Module                                │          │
│  └──────────────────────────────────────────────┘          │
└───────┬───────────────────────────────────────┬────────────┘
        │                                       │
        │                                       │
┌───────▼────────┐                    ┌───────▼────────┐
│   PostgreSQL   │                    │     Redis      │
│   (Database)   │                    │  (Cache/Queue)  │
└────────────────┘                    └────────────────┘
        │                                       │
        │                                       │
┌───────▼───────────────────────────────────────▼────────┐
│              Worker Process (NestJS)                    │
│  ┌──────────────────────────────────────────────┐      │
│  │         Queue Processors                      │      │
│  │  - Email Queue Processor                      │      │
│  └──────────────────────────────────────────────┘      │
└─────────────────────────────────────────────────────────┘
```

### Module Architecture

The application follows NestJS modular architecture:

1. **AppModule** - Root module that orchestrates all other modules
2. **ApiModule** - Groups all API endpoints
3. **AuthModule** - Handles authentication and authorization
4. **WorkerModule** - Background job processing
5. **Shared Modules** - Reusable modules (Cache, Mail, Socket)

### Request Flow

1. **Request arrives** at Fastify adapter
2. **Middleware** processes request (CORS, Helmet, etc.)
3. **Guards** check authentication/authorization
4. **Interceptors** handle cross-cutting concerns
5. **Controller/Resolver** processes business logic
6. **Service** executes business operations
7. **Response** is transformed and returned

## Environment Support

The application supports multiple environments:

- **local** - Local development
- **development** - Development environment
- **staging** - Staging environment
- **production** - Production environment
- **test** - Testing environment

Each environment has its own configuration loaded from environment files.

## Scalability Considerations

### Horizontal Scaling
- Stateless API server (can run multiple instances)
- Redis adapter for WebSocket (enables multi-instance WebSocket)
- Separate worker processes for background jobs

### Vertical Scaling
- Efficient Fastify HTTP server
- Connection pooling for database
- Redis caching to reduce database load

### Database Scaling
- Connection pooling configured
- Indexed database queries
- Migration system for schema evolution

## Security Features

- Helmet.js for security headers
- CORS configuration
- Rate limiting with Redis
- Input validation with class-validator
- SQL injection protection (TypeORM parameterized queries)
- XSS protection
- CSRF protection (via Better Auth)
- Secure session management

## Performance Optimizations

- SWC compiler for fast builds
- Redis caching layer
- Database connection pooling
- Efficient pagination (offset and cursor-based)
- Background job processing for heavy operations
- Static file serving optimization

## Next Steps

- Read [Getting Started Guide](./01-GETTING-STARTED.md) to set up your development environment
- Review [Architecture Details](./03-ARCHITECTURE.md) for deeper understanding
- Check [Development Workflow](./02-DEVELOPMENT-WORKFLOW.md) for development practices

