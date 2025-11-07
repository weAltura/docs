# Testing Guide

This document covers testing strategies, best practices, and testing tools.

## Testing Overview

The application uses Jest as the testing framework with support for:
- Unit tests
- Integration tests
- End-to-end (E2E) tests

## Test Configuration

### Jest Configuration

Located in `jest.config.json`:

```json
{
  "moduleFileExtensions": ["js", "json", "ts"],
  "rootDir": "src",
  "testRegex": ".*\\.spec\\.ts$",
  "transform": {
    "^.+\\.(t|j)s$": "ts-jest"
  },
  "collectCoverageFrom": [
    "**/*.(t|j)s"
  ],
  "coverageDirectory": "../coverage",
  "testEnvironment": "node"
}
```

### E2E Test Configuration

Located in `test/jest-e2e.json`:

```json
{
  "moduleFileExtensions": ["js", "json", "ts"],
  "rootDir": ".",
  "testEnvironment": "node",
  "testRegex": ".e2e-spec.ts$"
}
```

## Running Tests

### All Tests

```bash
pnpm test
```

### Watch Mode

```bash
pnpm test:watch
```

### Coverage

```bash
pnpm test:cov
```

### E2E Tests

```bash
pnpm test:e2e
```

### Debug Tests

```bash
pnpm test:debug
```

## Unit Testing

### Service Tests

Example service test:

```ts
import { Test, TestingModule } from '@nestjs/testing';
import { getRepositoryToken } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { UserService } from './user.service';
import { UserEntity } from '@/auth/entities/user.entity';

describe('UserService', () => {
  let service: UserService;
  let repository: Repository<UserEntity>;

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [
        UserService,
        {
          provide: getRepositoryToken(UserEntity),
          useValue: {
            findOne: jest.fn(),
            save: jest.fn(),
            update: jest.fn(),
            softDelete: jest.fn(),
          },
        },
      ],
    }).compile();

    service = module.get<UserService>(UserService);
    repository = module.get<Repository<UserEntity>>(
      getRepositoryToken(UserEntity),
    );
  });

  it('should be defined', () => {
    expect(service).toBeDefined();
  });

  describe('findOne', () => {
    it('should return a user', async () => {
      const user = { id: '1', email: 'test@example.com' };
      jest.spyOn(repository, 'findOne').mockResolvedValue(user as UserEntity);

      const result = await service.findOneUser('1');

      expect(result).toEqual(user);
      expect(repository.findOne).toHaveBeenCalledWith({
        where: { id: '1' },
      });
    });

    it('should throw NotFoundException if user not found', async () => {
      jest.spyOn(repository, 'findOne').mockResolvedValue(null);

      await expect(service.findOneUser('1')).rejects.toThrow(
        NotFoundException,
      );
    });
  });
});
```

### Controller Tests

```ts
import { Test, TestingModule } from '@nestjs/testing';
import { UserController } from './user.controller';
import { UserService } from './user.service';

describe('UserController', () => {
  let controller: UserController;
  let service: UserService;

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      controllers: [UserController],
      providers: [
        {
          provide: UserService,
          useValue: {
            findOneUser: jest.fn(),
            findAllUsers: jest.fn(),
          },
        },
      ],
    }).compile();

    controller = module.get<UserController>(UserController);
    service = module.get<UserService>(UserService);
  });

  it('should be defined', () => {
    expect(controller).toBeDefined();
  });

  describe('findOne', () => {
    it('should return a user', async () => {
      const user = { id: '1', email: 'test@example.com' };
      jest.spyOn(service, 'findOneUser').mockResolvedValue(user);

      const result = await controller.findUser('1');

      expect(result).toEqual(user);
      expect(service.findOneUser).toHaveBeenCalledWith('1');
    });
  });
});
```

### Guard Tests

```ts
import { ExecutionContext } from '@nestjs/common';
import { AuthGuard } from './auth.guard';

describe('AuthGuard', () => {
  let guard: AuthGuard;

  beforeEach(() => {
    guard = new AuthGuard();
  });

  it('should allow authenticated requests', () => {
    const context = {
      switchToHttp: () => ({
        getRequest: () => ({
          user: { id: '1', email: 'test@example.com' },
        }),
      }),
    } as ExecutionContext;

    expect(guard.canActivate(context)).toBe(true);
  });

  it('should reject unauthenticated requests', () => {
    const context = {
      switchToHttp: () => ({
        getRequest: () => ({}),
      }),
    } as ExecutionContext;

    expect(guard.canActivate(context)).toBe(false);
  });
});
```

## Integration Testing

### Database Integration Tests

```ts
import { Test, TestingModule } from '@nestjs/testing';
import { TypeOrmModule } from '@nestjs/typeorm';
import { UserService } from './user.service';
import { UserEntity } from '@/auth/entities/user.entity';

describe('UserService Integration', () => {
  let service: UserService;
  let module: TestingModule;

  beforeAll(async () => {
    module = await Test.createTestingModule({
      imports: [
        TypeOrmModule.forRoot({
          type: 'postgres',
          // Test database configuration
        }),
        TypeOrmModule.forFeature([UserEntity]),
      ],
      providers: [UserService],
    }).compile();

    service = module.get<UserService>(UserService);
  });

  afterAll(async () => {
    await module.close();
  });

  it('should create and find a user', async () => {
    const user = await service.create({
      email: 'test@example.com',
      username: 'testuser',
    });

    const found = await service.findOneUser(user.id);

    expect(found.email).toBe('test@example.com');
  });
});
```

## E2E Testing

### E2E Test Example

Located in `test/app.e2e-spec.ts`:

```ts
import { Test, TestingModule } from '@nestjs/testing';
import { INestApplication } from '@nestjs/common';
import * as request from 'supertest';
import { AppModule } from './../src/app.module';

describe('AppController (e2e)', () => {
  let app: INestApplication;

  beforeAll(async () => {
    const moduleFixture: TestingModule = await Test.createTestingModule({
      imports: [AppModule],
    }).compile();

    app = moduleFixture.createNestApplication();
    await app.init();
  });

  afterAll(async () => {
    await app.close();
  });

  it('/api/health (GET)', () => {
    return request(app.getHttpServer())
      .get('/api/health')
      .expect(200)
      .expect((res) => {
        expect(res.body).toHaveProperty('status', 'ok');
      });
  });

  describe('/api/v1/user (GET)', () => {
    it('should return 401 without authentication', () => {
      return request(app.getHttpServer())
        .get('/api/v1/user/whoami')
        .expect(401);
    });

    it('should return user with valid token', async () => {
      // Sign in first
      const signInResponse = await request(app.getHttpServer())
        .post('/api/auth/sign-in')
        .send({ email: 'test@example.com', password: 'password' });

      const token = signInResponse.body.token;

      // Get current user
      return request(app.getHttpServer())
        .get('/api/v1/user/whoami')
        .set('Authorization', `Bearer ${token}`)
        .expect(200)
        .expect((res) => {
          expect(res.body).toHaveProperty('id');
          expect(res.body).toHaveProperty('email');
        });
    });
  });
});
```

## Testing Best Practices

### 1. Test Structure

Follow AAA pattern:
- **Arrange** - Set up test data
- **Act** - Execute the code
- **Assert** - Verify results

### 2. Test Naming

Use descriptive names:
```ts
it('should return user when valid ID is provided', () => {});
it('should throw NotFoundException when user does not exist', () => {});
```

### 3. Mocking

Mock external dependencies:
```ts
jest.spyOn(repository, 'findOne').mockResolvedValue(user);
```

### 4. Test Isolation

Each test should be independent:
```ts
beforeEach(() => {
  // Reset mocks
  jest.clearAllMocks();
});
```

### 5. Coverage

Aim for high coverage:
- Critical paths: 100%
- Business logic: >90%
- Utilities: >80%

### 6. Test Data

Use factories for test data:
```ts
const createUser = (overrides = {}) => ({
  id: '1',
  email: 'test@example.com',
  ...overrides,
});
```

## Testing Utilities

### Test Database

Use separate test database:
```bash
DATABASE_NAME=wealtura_test
```

### Test Fixtures

Create reusable test fixtures:
```ts
// test/fixtures/user.fixture.ts
export const userFixture = {
  email: 'test@example.com',
  username: 'testuser',
  password: 'password123',
};
```

### Test Helpers

Create test helpers:
```ts
// test/helpers/auth.helper.ts
export async function createAuthenticatedUser(app: INestApplication) {
  // Sign up and return token
}
```

## Mocking Strategies

### Repository Mocking

```ts
{
  provide: getRepositoryToken(UserEntity),
  useValue: {
    findOne: jest.fn(),
    save: jest.fn(),
    create: jest.fn(),
  },
}
```

### Service Mocking

```ts
{
  provide: UserService,
  useValue: {
    findOne: jest.fn(),
    create: jest.fn(),
  },
}
```

### Config Service Mocking

```ts
{
  provide: ConfigService,
  useValue: {
    get: jest.fn((key: string) => {
      const config = {
        'database.host': 'localhost',
        'database.port': 5432,
      };
      return config[key];
    }),
  },
}
```

## Testing Async Code

### Promises

```ts
it('should handle async operations', async () => {
  const result = await service.asyncOperation();
  expect(result).toBeDefined();
});
```

### Error Handling

```ts
it('should throw error on failure', async () => {
  await expect(service.failingOperation()).rejects.toThrow(Error);
});
```

## Coverage Reports

### View Coverage

After running `pnpm test:cov`, view coverage report:
- HTML report: `coverage/index.html`
- Terminal output: Summary in terminal

### Coverage Thresholds

Configure in `jest.config.json`:
```json
{
  "coverageThreshold": {
    "global": {
      "branches": 80,
      "functions": 80,
      "lines": 80,
      "statements": 80
    }
  }
}
```

## Continuous Integration

### GitHub Actions

Example test workflow:
```yaml
- name: Run tests
  run: pnpm test:cov

- name: Upload coverage
  uses: codecov/codecov-action@v3
```

## Troubleshooting

### Common Issues

**1. Tests Timeout**
- Increase timeout: `jest.setTimeout(10000)`
- Check for hanging promises

**2. Module Not Found**
- Check import paths
- Verify module exports

**3. Database Connection**
- Use test database
- Clean up after tests

**4. Mock Not Working**
- Verify mock setup
- Check mock implementation

## Next Steps

- Review [Development Workflow](./02-DEVELOPMENT-WORKFLOW.md) for testing practices
- Check [API Documentation](./06-API.md) for API testing
- See [Configuration](./08-CONFIGURATION.md) for test configuration

