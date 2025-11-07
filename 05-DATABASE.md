# Database Guide

This document covers database management, migrations, seeds, and best practices.

## Database Technology

- **Database**: PostgreSQL 16
- **ORM**: TypeORM 0.3.20
- **Connection**: Connection pooling enabled

## Database Configuration

### Connection Configuration

Located in `src/config/database/database.config.ts`:

```typescript
export function getConfig(): DatabaseConfig {
  return {
    type: 'postgres',
    host: process.env.DATABASE_HOST,
    port: parseInt(process.env.DATABASE_PORT, 10),
    username: process.env.DATABASE_USERNAME,
    password: process.env.DATABASE_PASSWORD,
    database: process.env.DATABASE_NAME,
    // ... more configuration
  };
}
```

### Environment Variables

```bash
DATABASE_HOST=localhost
DATABASE_PORT=5432
DATABASE_USERNAME=postgres
DATABASE_PASSWORD=your_password
DATABASE_NAME=wealtura
DATABASE_LOGGING=true
DATABASE_MAX_CONNECTIONS=100
DATABASE_SSL_MODE=disable
```

### Connection Pooling

- **Pool Size**: Configurable via `DATABASE_MAX_CONNECTIONS` (default: 100)
- **Connection Management**: Handled by TypeORM
- **Idle Timeout**: Managed by PostgreSQL

## Entity Design

### Base Model

All entities extend `BaseModel`:

```typescript
export abstract class BaseModel {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @CreateDateColumn()
  createdAt: Date;

  @UpdateDateColumn()
  updatedAt: Date;

  @DeleteDateColumn()
  deletedAt?: Date; // Soft delete
}
```

**Benefits:**
- Consistent ID generation (UUID)
- Automatic timestamps
- Soft delete support

### Entity Example

```typescript
@Entity('user')
export class UserEntity extends BaseModel {
  @Index({ unique: true, where: '"deletedAt" IS NULL' })
  @Column()
  username: string;

  @Index({ unique: true, where: '"deletedAt" IS NULL' })
  @Column()
  email: string;

  @Column({ type: 'boolean', default: false })
  isEmailVerified: boolean;

  @Column({ type: 'enum', enum: Role, default: Role.User })
  role: Role;
}
```

### Indexing Strategy

**Unique Indexes with Soft Delete:**
```typescript
@Index({ unique: true, where: '"deletedAt" IS NULL' })
```

This ensures uniqueness only for non-deleted records.

**Regular Indexes:**
```typescript
@Index()
@Column()
email: string;
```

## Migrations

### Migration System

TypeORM migrations track database schema changes.

### Creating Migrations

#### Option 1: Generate from Entity Changes

```bash
# Make changes to entity
# Then generate migration
pnpm migration:generate src/database/migrations/AddUserProfileFields
```

This automatically detects entity changes and generates SQL.

#### Option 2: Create Empty Migration

```bash
pnpm migration:create src/database/migrations/AddUserProfileFields
```

Then manually write the migration:

```typescript
import { MigrationInterface, QueryRunner } from 'typeorm';

export class AddUserProfileFields1234567890 implements MigrationInterface {
  public async up(queryRunner: QueryRunner): Promise<void> {
    await queryRunner.query(`
      ALTER TABLE "user"
      ADD COLUMN "firstName" VARCHAR,
      ADD COLUMN "lastName" VARCHAR
    `);
  }

  public async down(queryRunner: QueryRunner): Promise<void> {
    await queryRunner.query(`
      ALTER TABLE "user"
      DROP COLUMN "firstName",
      DROP COLUMN "lastName"
    `);
  }
}
```

### Running Migrations

```bash
# Run all pending migrations
pnpm migration:up

# Revert last migration
pnpm migration:down

# Show migration status
pnpm migration:show
```

### Migration Best Practices

1. **Always write down migrations** - Include rollback logic
2. **Test migrations** - Test both up and down
3. **Review generated SQL** - Verify before committing
4. **One change per migration** - Keep migrations focused
5. **Never edit existing migrations** - Create new ones instead
6. **Backup before production** - Always backup before running

### Migration Naming Convention

```
<Timestamp>-<Description>.ts

Examples:
1746266963361-init.ts
1747406772427-add-firstName-lastName.ts
1747485461165-add-displayUsername.ts
```

## Seeds

### Seed System

TypeORM Extension provides seed functionality for initial data.

### Creating Seeds

```bash
pnpm seed:create src/database/seeds/InitialUsers
```

### Seed Example

```typescript
import { DataSource } from 'typeorm';
import { Seeder } from 'typeorm-extension';
import { UserEntity } from '@/auth/entities/user.entity';

export default class InitialUsersSeeder implements Seeder {
  public async run(dataSource: DataSource): Promise<void> {
    const repository = dataSource.getRepository(UserEntity);

    const users = [
      {
        email: 'admin@example.com',
        username: 'admin',
        role: Role.Admin,
      },
      // ... more users
    ];

    await repository.save(users);
  }
}
```

### Running Seeds

```bash
# Run all seeds
pnpm seed:run
```

### Seed Best Practices

1. **Idempotent** - Seeds should be safe to run multiple times
2. **Development only** - Don't seed production data
3. **Use factories** - Generate test data programmatically
4. **Clean up** - Remove test data after development

## Query Patterns

### Repository Pattern

```typescript
// Find one
const user = await this.userRepository.findOne({
  where: { id },
  relations: ['profile'],
});

// Find many
const users = await this.userRepository.find({
  where: { isActive: true },
  order: { createdAt: 'DESC' },
});

// Create
const newUser = this.userRepository.create({ email, username });
await this.userRepository.save(newUser);

// Update
await this.userRepository.update(id, { firstName: 'John' });

// Delete (soft delete)
await this.userRepository.softDelete(id);

// Hard delete
await this.userRepository.delete(id);
```

### Query Builder

For complex queries:

```typescript
const users = await this.userRepository
  .createQueryBuilder('user')
  .leftJoinAndSelect('user.profile', 'profile')
  .where('user.isActive = :isActive', { isActive: true })
  .andWhere('user.email LIKE :email', { email: '%@example.com' })
  .orderBy('user.createdAt', 'DESC')
  .take(10)
  .skip(0)
  .getMany();
```

### Raw Queries

Use sparingly and with caution:

```typescript
const result = await this.userRepository.query(
  'SELECT * FROM user WHERE email = $1',
  [email]
);
```

## Relationships

### One-to-Many

```typescript
// User entity
@OneToMany(() => Post, post => post.user)
posts: Post[];

// Post entity
@ManyToOne(() => User, user => user.posts)
@JoinColumn({ name: 'userId' })
user: User;
```

### Many-to-Many

```typescript
@ManyToMany(() => Tag)
@JoinTable({
  name: 'post_tags',
  joinColumn: { name: 'postId' },
  inverseJoinColumn: { name: 'tagId' },
})
tags: Tag[];
```

### Eager vs Lazy Loading

**Eager Loading:**
```typescript
@ManyToOne(() => User, { eager: true })
user: User; // Always loaded
```

**Lazy Loading:**
```typescript
@ManyToOne(() => User)
user: Promise<User>; // Loaded when accessed
```

**Explicit Loading:**
```typescript
const user = await this.userRepository.findOne({
  where: { id },
  relations: ['posts'], // Explicitly load
});
```

## Pagination

### Offset Pagination

```typescript
const query = this.userRepository
  .createQueryBuilder('user')
  .orderBy('user.createdAt', 'DESC');

const [users, meta] = await paginate<UserEntity>(query, dto, {
  skipCount: false,
  takeAll: false,
});
```

### Cursor Pagination

```typescript
const queryBuilder = this.userRepository.createQueryBuilder('user');
const paginator = buildPaginator({
  entity: UserEntity,
  alias: 'user',
  paginationKeys: ['createdAt'],
  query: {
    limit: dto.limit,
    order: 'DESC',
    afterCursor: dto.afterCursor,
  },
});

const { data, cursor } = await paginator.paginate(queryBuilder);
```

## Transactions

### Using Transactions

```typescript
await this.userRepository.manager.transaction(async (manager) => {
  const user = manager.create(UserEntity, { email, username });
  await manager.save(user);

  const profile = manager.create(ProfileEntity, { userId: user.id });
  await manager.save(profile);
});
```

### Transaction Isolation Levels

TypeORM supports standard isolation levels:
- READ UNCOMMITTED
- READ COMMITTED (default)
- REPEATABLE READ
- SERIALIZABLE

## Soft Deletes

### Enabling Soft Deletes

Entities extending `BaseModel` have soft delete support:

```typescript
// Soft delete
await this.userRepository.softDelete(id);

// Restore
await this.userRepository.restore(id);

// Find including deleted
const user = await this.userRepository.findOne({
  where: { id },
  withDeleted: true,
});

// Find only deleted
const deletedUsers = await this.userRepository.find({
  withDeleted: true,
  where: { deletedAt: Not(IsNull()) },
});
```

## Database Logging

### Enable Query Logging

Set in environment:
```bash
DATABASE_LOGGING=true
```

### Custom Logger

Located in `src/database/logger/database-logger.ts`:

```typescript
export class DatabaseLogger implements Logger {
  logQuery(query: string, parameters?: any[]) {
    // Log queries
  }
}
```

## Performance Optimization

### Indexes

Add indexes on frequently queried columns:

```typescript
@Index()
@Column()
email: string;

@Index({ unique: true })
@Column()
username: string;

@Index(['firstName', 'lastName']) // Composite index
```

### Query Optimization

1. **Use select** - Only select needed columns
2. **Use relations** - Avoid N+1 queries
3. **Use pagination** - Don't load all records
4. **Use indexes** - Index frequently queried columns

### Connection Pooling

Configure appropriate pool size:
```bash
DATABASE_MAX_CONNECTIONS=100
```

## Backup and Recovery

### Backup Strategy

1. **Regular Backups** - Automated daily backups
2. **Before Migrations** - Always backup before migrations
3. **Point-in-Time Recovery** - Enable WAL archiving

### Backup Commands

```bash
# Backup database
pg_dump -U postgres -d wealtura > backup.sql

# Restore database
psql -U postgres -d wealtura < backup.sql
```

## Database Schema

### Current Tables

- `user` - User accounts
- `session` - Active sessions (Better Auth)
- `account` - OAuth accounts (Better Auth)
- `verification` - Email verification (Better Auth)
- `twoFactor` - 2FA secrets (Better Auth)
- `passkey` - Passkey credentials (Better Auth)
- `migrations` - Migration tracking
- `seeders` - Seed tracking

### Schema Visualization

Generate ERD:
```bash
pnpm erd:generate
```

## Troubleshooting

### Common Issues

**1. Connection Refused**
- Check Docker service is running
- Verify connection string
- Check firewall settings

**2. Migration Fails**
- Check migration SQL syntax
- Verify database state
- Check for conflicting migrations

**3. Slow Queries**
- Check for missing indexes
- Review query execution plans
- Optimize query patterns

**4. Deadlocks**
- Review transaction isolation
- Check for long-running transactions
- Optimize transaction scope

## Best Practices

1. **Always use migrations** - Never modify schema directly
2. **Test migrations** - Test both up and down
3. **Use transactions** - For multiple related operations
4. **Index wisely** - Don't over-index
5. **Monitor performance** - Use query logging
6. **Backup regularly** - Before major changes
7. **Use soft deletes** - When data retention is needed
8. **Optimize queries** - Use query builder for complex queries

## Next Steps

- Review [Migrations Guide](./05-DATABASE.md#migrations) for migration workflow
- Check [Entity Design](./05-DATABASE.md#entity-design) for entity patterns
- See [Configuration](./08-CONFIGURATION.md) for database configuration

