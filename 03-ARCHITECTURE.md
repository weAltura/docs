# Architecture Details

This document provides a detailed overview of the system architecture, design patterns, and technical decisions.

## System Architecture

### Application Layers

The application follows a layered architecture:

```
┌─────────────────────────────────────────┐
│         Presentation Layer              │
│  - REST Controllers                     │
│  - GraphQL Resolvers                     │
│  - WebSocket Gateways                    │
└──────────────┬──────────────────────────┘
               │
┌──────────────▼──────────────────────────┐
│         Application Layer                │
│  - Services (Business Logic)              │
│  - DTOs (Data Transfer Objects)          │
│  - Guards (Authentication/Authorization)  │
└──────────────┬──────────────────────────┘
               │
┌──────────────▼──────────────────────────┐
│         Domain Layer                     │
│  - Entities (Domain Models)              │
│  - Repositories (Data Access)            │
│  - Value Objects                         │
└──────────────┬──────────────────────────┘
               │
┌──────────────▼──────────────────────────┐
│         Infrastructure Layer             │
│  - Database (TypeORM)                    │
│  - Cache (Redis)                         │
│  - External Services (AWS, etc.)         │
└──────────────────────────────────────────┘
```

## Module Architecture

### AppModule Structure

The root `AppModule` is divided into three variants:

1. **AppModule.main()** - Main API server
2. **AppModule.worker()** - Background worker process
3. **AppModule.common()** - Shared modules for both

### Module Dependencies

```
AppModule (main)
├── ConfigModule (global)
├── LoggerModule
├── TypeOrmModule
├── BullModule
├── PrometheusModule
├── CacheManagerModule
├── MailModule
├── I18nModule
├── GraphQLModule
├── ThrottlerModule
├── BullBoardModule
├── ApiModule
│   ├── HealthModule
│   ├── UserModule
│   └── FileModule
├── AuthModule (global)
└── SocketModule

AppModule (worker)
├── [All common modules]
└── WorkerModule
    └── EmailQueueModule
```

## Design Patterns

### 1. Dependency Injection

NestJS uses dependency injection throughout:

```typescript
@Injectable()
export class UserService {
  constructor(
    @InjectRepository(UserEntity)
    private readonly userRepository: Repository<UserEntity>,
    private readonly configService: ConfigService,
  ) {}
}
```

**Benefits:**
- Loose coupling
- Easy testing (mock dependencies)
- Single Responsibility Principle

### 2. Repository Pattern

TypeORM repositories abstract database access:

```typescript
// Service uses repository
const user = await this.userRepository.findOne({ where: { id } });

// Repository handles database operations
// Easy to swap implementations for testing
```

### 3. Service Layer Pattern

Business logic is encapsulated in services:

```typescript
@Injectable()
export class UserService {
  async findOne(id: string): Promise<UserDto> {
    // Business logic here
    const user = await this.userRepository.findOne({ where: { id } });
    if (!user) {
      throw new NotFoundException('User not found');
    }
    return this.mapToDto(user);
  }
}
```

### 4. DTO Pattern

Data Transfer Objects separate API contracts from entities:

```typescript
// Entity (database model)
@Entity('user')
export class UserEntity {
  @Column()
  password: string; // Sensitive data
}

// DTO (API response)
export class UserDto {
  id: string;
  email: string;
  // No password exposed
}
```

### 5. Guard Pattern

Guards handle cross-cutting concerns:

```typescript
@Injectable()
export class AuthGuard implements CanActivate {
  canActivate(context: ExecutionContext): boolean {
    // Authentication logic
    return true;
  }
}
```

### 6. Interceptor Pattern

Interceptors handle request/response transformation:

```typescript
@Injectable()
export class TransformInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    return next.handle().pipe(
      map(data => ({ data, timestamp: new Date() })),
    );
  }
}
```

## Request Lifecycle

### REST API Request Flow

```
1. HTTP Request
   ↓
2. Fastify Adapter
   ↓
3. Middleware Stack
   - Helmet (Security)
   - CORS
   - Compression
   - Body Parser
   ↓
4. Global Guards
   - AuthGuard (Authentication)
   - ThrottlerGuard (Rate Limiting)
   ↓
5. Route Handler
   - Controller Method
   ↓
6. Service Layer
   - Business Logic
   - Data Validation
   ↓
7. Repository Layer
   - Database Operations
   ↓
8. Response Transformation
   - DTO Mapping
   - Serialization
   ↓
9. HTTP Response
```

### GraphQL Request Flow

```
1. GraphQL Request
   ↓
2. Apollo Server
   ↓
3. GraphQL Resolver
   ↓
4. Guards (if applied)
   ↓
5. Service Layer
   ↓
6. Repository Layer
   ↓
7. GraphQL Response
```

### WebSocket Connection Flow

```
1. WebSocket Connection
   ↓
2. Socket.io Server
   ↓
3. Authentication (via handshake)
   ↓
4. Redis Adapter (for scaling)
   ↓
5. Gateway Handler
   ↓
6. Service Layer
   ↓
7. Broadcast/Response
```

## Data Flow

### Creating a User (Example)

```
1. Client → POST /api/v1/user
   ↓
2. UserController.create()
   ↓
3. ValidationPipe validates CreateUserDto
   ↓
4. UserService.create()
   - Business logic
   - Password hashing
   ↓
5. UserRepository.save()
   - TypeORM saves to database
   ↓
6. UserEntity persisted
   ↓
7. UserDto returned (password excluded)
   ↓
8. Response sent to client
```

## Configuration Management

### Environment-Based Configuration

Configuration is loaded based on `NODE_ENV`:

```typescript
// Loads: .env.development → .env.local → .env
envFilePath: getEnvFilePaths()
```

### Configuration Modules

Each feature has its own configuration module:

```typescript
// config/database/database.config.ts
export default registerAs<DatabaseConfig>('database', () => {
  validateConfig(process.env, EnvironmentVariablesValidator);
  return getConfig();
});
```

### Configuration Access

```typescript
// Injected ConfigService
constructor(private configService: ConfigService) {}

// Type-safe access
const dbConfig = this.configService.getOrThrow('database', { infer: true });
```

## Database Architecture

### Entity Design

Entities extend `BaseModel`:

```typescript
@Entity('user')
export class UserEntity extends BaseModel {
  @Column()
  email: string;

  // BaseModel provides:
  // - id (UUID)
  // - createdAt
  // - updatedAt
  // - deletedAt (soft delete)
}
```

### Relationship Patterns

```typescript
// One-to-Many
@OneToMany(() => Post, post => post.user)
posts: Post[];

// Many-to-One
@ManyToOne(() => User, user => user.posts)
user: User;

// Many-to-Many
@ManyToMany(() => Tag)
@JoinTable()
tags: Tag[];
```

### Migration Strategy

1. **Generate migration** from entity changes
2. **Review** generated SQL
3. **Test** migration locally
4. **Commit** migration file
5. **Run** in staging/production

## Caching Strategy

### Cache Layers

1. **Application Cache** (Redis)
   - Session storage
   - Access tokens
   - Frequently accessed data

2. **Query Cache** (TypeORM)
   - Database query results
   - Configurable TTL

### Cache Keys

```typescript
// Structured cache keys
cacheService.get({
  key: 'User',
  args: [userId]
}); // → "User:123"
```

## Authentication Architecture

### Better Auth Integration

Better Auth handles:
- Session management
- Token generation
- OAuth flows
- 2FA

### Custom Integration

```typescript
// Custom hooks
@Hook('before', 'sign-in')
async beforeSignIn(ctx) {
  // Custom logic
}

// Custom email sending
sendVerificationEmail: async ({ user, url }) => {
  await authService.verifyEmail({ url, userId: user.id });
}
```

### Session Storage

- **Primary**: Database (Better Auth tables)
- **Secondary**: Redis (for scalability)

## Background Job Architecture

### Queue System

```
API Server
    ↓ (enqueue)
BullMQ Queue (Redis)
    ↓ (process)
Worker Process
    ↓ (execute)
Job Processor
```

### Job Types

- **Email Jobs**: Async email sending
- **Future**: Image processing, data exports, etc.

### Worker Process

Separate NestJS application:
- Same codebase
- Different module (`WorkerModule`)
- Processes jobs from queues

## API Architecture

### REST API

- **Versioning**: URI-based (`/api/v1/...`)
- **Documentation**: Swagger/OpenAPI
- **Pagination**: Offset and cursor-based
- **Error Handling**: Standardized error responses

### GraphQL API

- **Schema-first**: GraphQL schema files
- **Resolvers**: Field-level resolvers
- **Authentication**: Same guards as REST
- **Playground**: Available in development

### WebSocket API

- **Socket.io**: Real-time communication
- **Redis Adapter**: Multi-instance support
- **Authentication**: Via handshake
- **Rooms**: Namespace-based organization

## Security Architecture

### Security Layers

1. **Network Layer**
   - HTTPS (production)
   - Helmet security headers

2. **Application Layer**
   - Input validation (class-validator)
   - Output sanitization
   - Rate limiting

3. **Authentication Layer**
   - Better Auth
   - Session management
   - Token validation

4. **Authorization Layer**
   - Role-based access (ready)
   - Resource-level permissions

5. **Data Layer**
   - Parameterized queries (TypeORM)
   - SQL injection protection

## Scalability Considerations

### Horizontal Scaling

**API Server:**
- Stateless design
- Redis for shared state
- Load balancer ready

**WebSocket:**
- Redis adapter for multi-instance
- Sticky sessions not required

**Worker:**
- Multiple worker instances
- Job distribution via Redis

### Vertical Scaling

- Connection pooling
- Efficient query patterns
- Caching strategy
- Background job processing

## Monitoring Architecture

### Metrics Collection

- **Prometheus**: Metrics endpoint
- **Custom Metrics**: Business metrics
- **System Metrics**: CPU, memory, etc.

### Logging

- **Structured Logging**: Pino JSON logs
- **Log Levels**: error, warn, info, debug
- **Context**: Request IDs, user IDs

### Error Tracking

- **Sentry**: Error aggregation
- **Stack Traces**: Full context
- **Release Tracking**: Version correlation

## Performance Optimizations

### Database

- Indexes on frequently queried columns
- Query optimization
- Connection pooling
- Read replicas (future)

### Caching

- Redis caching layer
- Cache invalidation strategy
- TTL configuration

### API

- Response compression
- Pagination
- Field selection (GraphQL)
- Background jobs for heavy operations

## Error Handling

### Error Hierarchy

```
Error
├── HttpException
│   ├── BadRequestException
│   ├── UnauthorizedException
│   ├── ForbiddenException
│   ├── NotFoundException
│   └── ...
└── Custom Exceptions
```

### Error Response Format

```json
{
  "statusCode": 404,
  "message": "User not found",
  "error": "Not Found",
  "timestamp": "2024-01-01T00:00:00.000Z",
  "path": "/api/v1/user/123"
}
```

## Internationalization

### i18n Architecture

- **Translation Files**: JSON per language
- **Resolvers**: Query param, header, accept-language
- **Service**: `I18nService` for translations

### Usage

```typescript
this.i18nService.t('user.notFound', { args: { id } });
```

## File Upload Architecture

### Storage Options

1. **Local Storage**: Development
2. **AWS S3**: Production

### Upload Flow

```
1. Multipart upload
   ↓
2. File validation
   ↓
3. Storage (local/S3)
   ↓
4. Database record
   ↓
5. Response with file URL
```

## Next Steps

- Review [Authentication Documentation](./04-AUTHENTICATION.md)
- Check [Database Guide](./05-DATABASE.md)
- See [API Documentation](./06-API.md)

