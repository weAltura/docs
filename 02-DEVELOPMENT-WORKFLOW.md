# Development Workflow

This document outlines the development practices, coding standards, and workflows used in the Wealtura Server project.

## Development Principles

### 1. Code Quality First
- Write clean, readable, and maintainable code
- Follow TypeScript best practices
- Use proper error handling
- Write meaningful comments for complex logic

### 2. Test-Driven Development (TDD)
- Write tests alongside or before implementation
- Maintain high test coverage
- Test edge cases and error scenarios

### 3. Incremental Development
- Make small, focused commits
- Use feature branches
- Review code before merging

### 4. Documentation
- Document complex logic
- Keep API documentation up-to-date
- Update this documentation when making significant changes

## Git Workflow

### Branch Strategy

We use a simplified Git Flow:

- **main/master** - Production-ready code
- **develop** - Integration branch for features
- **feature/*** - Feature development branches
- **bugfix/*** - Bug fix branches
- **hotfix/*** - Critical production fixes

### Branch Naming Convention

```
feature/user-authentication
feature/add-payment-integration
bugfix/fix-email-sending
hotfix/critical-security-patch
```

### Commit Messages

We follow [Conventional Commits](https://www.conventionalcommits.org/) format:

```
<type>(<scope>): <subject>

<body>

<footer>
```

**Types:**
- `feat`: New feature
- `fix`: Bug fix
- `docs`: Documentation changes
- `style`: Code style changes (formatting, etc.)
- `refactor`: Code refactoring
- `test`: Adding or updating tests
- `chore`: Maintenance tasks
- `perf`: Performance improvements

**Examples:**
```
feat(auth): add two-factor authentication

Implement 2FA using TOTP with support for backup codes.

Closes #123

fix(api): correct pagination offset calculation

The offset was incorrectly calculated when page number was 0.

refactor(database): optimize user query performance

Add indexes to frequently queried columns.
```

### Commit Process

1. **Stage changes**: `git add <files>`
2. **Commit**: `git commit` (Husky will run pre-commit hooks)
3. **Push**: `git push origin <branch-name>`
4. **Create PR**: Open pull request for review

### Pre-commit Hooks

Husky runs the following before each commit:

1. **Linting**: ESLint checks code quality
2. **Formatting**: Prettier formats code
3. **Type checking**: TypeScript type checking
4. **Tests**: Run affected tests (if configured)

To skip hooks (use with caution):
```bash
git commit --no-verify
```

## Code Style Guidelines

### TypeScript

#### Naming Conventions

- **Classes**: PascalCase (`UserService`, `AuthGuard`)
- **Interfaces/Types**: PascalCase (`UserDto`, `AuthConfig`)
- **Variables/Functions**: camelCase (`userId`, `getUserById`)
- **Constants**: UPPER_SNAKE_CASE (`MAX_RETRY_ATTEMPTS`, `DEFAULT_PAGE_SIZE`)
- **Files**: kebab-case (`user.service.ts`, `auth.guard.ts`)

#### Type Definitions

```typescript
// ✅ Good: Explicit types
interface UserDto {
  id: string;
  email: string;
  name: string;
}

function getUserById(id: string): Promise<UserDto> {
  // ...
}

// ❌ Bad: Using `any`
function getUserById(id: any): any {
  // ...
}
```

#### Imports

```typescript
// Order: External → Internal → Relative
import { Injectable } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';

import { UserEntity } from '@/auth/entities/user.entity';
import { UserService } from '@/api/user/user.service';

import { LocalModule } from './local.module';
```

#### Decorators

```typescript
// ✅ Good: Proper decorator usage
@Injectable()
export class UserService {
  constructor(
    @InjectRepository(UserEntity)
    private readonly userRepository: Repository<UserEntity>,
  ) {}
}

// ❌ Bad: Missing decorators
export class UserService {
  constructor(private userRepository: Repository<UserEntity>) {}
}
```

### NestJS Patterns

#### Module Structure

```typescript
@Module({
  imports: [
    // External modules first
    ConfigModule,
    TypeOrmModule.forFeature([UserEntity]),
    // Internal modules
    AuthModule,
  ],
  controllers: [UserController],
  providers: [UserService],
  exports: [UserService], // Export if used by other modules
})
export class UserModule {}
```

#### Service Pattern

```typescript
@Injectable()
export class UserService {
  constructor(
    @InjectRepository(UserEntity)
    private readonly userRepository: Repository<UserEntity>,
    private readonly configService: ConfigService,
  ) {}

  async findOne(id: string): Promise<UserDto> {
    const user = await this.userRepository.findOne({ where: { id } });
    if (!user) {
      throw new NotFoundException('User not found');
    }
    return user;
  }
}
```

#### Controller Pattern

```typescript
@Controller('users')
@ApiTags('users')
export class UserController {
  constructor(private readonly userService: UserService) {}

  @Get(':id')
  @ApiOperation({ summary: 'Get user by ID' })
  @ApiResponse({ status: 200, type: UserDto })
  async findOne(@Param('id') id: string): Promise<UserDto> {
    return this.userService.findOne(id);
  }
}
```

### Error Handling

#### Use Appropriate HTTP Exceptions

```typescript
import {
  NotFoundException,
  BadRequestException,
  UnauthorizedException,
  ForbiddenException,
  ConflictException,
} from '@nestjs/common';

// ✅ Good
if (!user) {
  throw new NotFoundException('User not found');
}

if (user.isBlocked) {
  throw new ForbiddenException('User is blocked');
}

// ❌ Bad
if (!user) {
  throw new Error('User not found');
}
```

#### Custom Error Messages

```typescript
// Use i18n for user-facing messages
throw new NotFoundException(
  this.i18nService.t('user.notFound', { args: { id } }),
);
```

### Database Patterns

#### Repository Usage

```typescript
// ✅ Good: Use repository methods
const user = await this.userRepository.findOne({
  where: { id },
  relations: ['profile'],
});

// ✅ Good: Use query builder for complex queries
const users = await this.userRepository
  .createQueryBuilder('user')
  .leftJoinAndSelect('user.profile', 'profile')
  .where('user.isActive = :isActive', { isActive: true })
  .getMany();

// ❌ Bad: Raw SQL unless necessary
const users = await this.userRepository.query(
  'SELECT * FROM users WHERE is_active = true',
);
```

#### Transactions

```typescript
// ✅ Good: Use transactions for multiple operations
await this.userRepository.manager.transaction(async (manager) => {
  await manager.save(UserEntity, user);
  await manager.save(ProfileEntity, profile);
});
```

## File Organization

### Module Structure

Each feature module should follow this structure:

```
feature-name/
├── dto/              # Data Transfer Objects
│   ├── create-feature.dto.ts
│   └── update-feature.dto.ts
├── schema/           # GraphQL schemas (if applicable)
│   └── feature.schema.ts
├── feature.controller.ts
├── feature.service.ts
├── feature.resolver.ts  # GraphQL resolver (if applicable)
└── feature.module.ts
```

### Shared Code

- **Common DTOs**: `src/common/dto/`
- **Common Types**: `src/common/types/`
- **Utilities**: `src/utils/`
- **Decorators**: `src/decorators/`
- **Constants**: `src/constants/`

## Testing Practices

### Unit Tests

```typescript
describe('UserService', () => {
  let service: UserService;
  let repository: Repository<UserEntity>;

  beforeEach(async () => {
    const module = await Test.createTestingModule({
      providers: [
        UserService,
        {
          provide: getRepositoryToken(UserEntity),
          useValue: {
            findOne: jest.fn(),
            save: jest.fn(),
          },
        },
      ],
    }).compile();

    service = module.get<UserService>(UserService);
    repository = module.get<Repository<UserEntity>>(
      getRepositoryToken(UserEntity),
    );
  });

  it('should find user by id', async () => {
    const user = { id: '1', email: 'test@example.com' };
    jest.spyOn(repository, 'findOne').mockResolvedValue(user as UserEntity);

    const result = await service.findOne('1');

    expect(result).toEqual(user);
    expect(repository.findOne).toHaveBeenCalledWith({ where: { id: '1' } });
  });
});
```

### E2E Tests

```typescript
describe('UserController (e2e)', () => {
  let app: INestApplication;

  beforeAll(async () => {
    const moduleFixture = await Test.createTestingModule({
      imports: [AppModule],
    }).compile();

    app = moduleFixture.createNestApplication();
    await app.init();
  });

  it('/users/:id (GET)', () => {
    return request(app.getHttpServer())
      .get('/api/users/1')
      .expect(200)
      .expect((res) => {
        expect(res.body).toHaveProperty('id');
        expect(res.body).toHaveProperty('email');
      });
  });
});
```

## Code Review Guidelines

### For Authors

1. **Small PRs**: Keep pull requests focused and small
2. **Clear Description**: Explain what and why, not just what
3. **Tests**: Include tests for new features
4. **Documentation**: Update docs if needed
5. **Self-Review**: Review your own code first

### For Reviewers

1. **Be Constructive**: Provide helpful feedback
2. **Check Logic**: Verify business logic is correct
3. **Check Tests**: Ensure adequate test coverage
4. **Check Performance**: Look for performance issues
5. **Check Security**: Identify security concerns

## Debugging

### VS Code Debugging

1. Set breakpoints in your code
2. Press `F5` to start debugging
3. Use debug console to inspect variables
4. Step through code with debug controls

### Logging

```typescript
// Use NestJS Logger
private readonly logger = new Logger(UserService.name);

this.logger.log('User created successfully');
this.logger.warn('Rate limit approaching');
this.logger.error('Failed to send email', error.stack);
this.logger.debug('Debug information', { userId, email });
```

### Database Debugging

```typescript
// Enable query logging in development
DATABASE_LOGGING=true

// Use query builder logging
const query = this.userRepository
  .createQueryBuilder('user')
  .where('user.id = :id', { id: '1' });

console.log(query.getSql()); // Log generated SQL
```

## Performance Considerations

### Database Queries

- Use indexes on frequently queried columns
- Avoid N+1 queries (use relations or joins)
- Use pagination for large datasets
- Cache frequently accessed data

### API Performance

- Implement caching where appropriate
- Use background jobs for heavy operations
- Optimize response payloads
- Use compression (already enabled)

## Security Best Practices

1. **Never commit secrets**: Use environment variables
2. **Validate input**: Use DTOs with class-validator
3. **Sanitize output**: Use serialization
4. **Use parameterized queries**: TypeORM handles this
5. **Rate limiting**: Already configured
6. **Authentication**: Always verify user identity
7. **Authorization**: Check user permissions

## Documentation

### Code Comments

```typescript
/**
 * Finds a user by ID and returns user data.
 *
 * @param id - User UUID
 * @param options - Optional query options
 * @returns User DTO
 * @throws NotFoundException if user not found
 */
async findOne(id: string, options?: FindOneOptions<UserEntity>): Promise<UserDto> {
  // Implementation
}
```

### API Documentation

- Use Swagger decorators for REST APIs
- Document GraphQL schemas
- Include examples in documentation

## Common Tasks

### Adding a New Feature

1. Create feature branch: `git checkout -b feature/new-feature`
2. Create module structure
3. Implement feature
4. Write tests
5. Update documentation
6. Create pull request

### Adding a New API Endpoint

1. Add DTOs in `dto/` folder
2. Add service method
3. Add controller endpoint
4. Add Swagger decorators
5. Write tests
6. Update API documentation

### Adding a Database Migration

1. Generate migration: `pnpm migration:generate src/database/migrations/MigrationName`
2. Review generated migration
3. Test migration: `pnpm migration:up`
4. Commit migration file

## Next Steps

- Review [Architecture Details](./03-ARCHITECTURE.md) for system design
- Check [API Documentation](./06-API.md) for API patterns
- See [Testing Guide](./09-TESTING.md) for testing practices

