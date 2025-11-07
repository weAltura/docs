# Authentication

This document covers the authentication system implemented using Better Auth.

## Overview

The application uses [Better Auth](https://www.better-auth.com/) for comprehensive authentication. Better Auth provides a complete authentication solution with support for multiple authentication methods.

## Supported Authentication Methods

### 1. Email/Password Authentication

Traditional email and password authentication with email verification.

**Features:**
- Email verification required
- Password hashing (handled by Better Auth)
- Password reset functionality
- Secure session management

### 2. Magic Link Authentication

Passwordless authentication via email magic links.

**Features:**
- One-click sign-in via email
- Time-limited links
- Disabled sign-up (only sign-in)

### 3. OAuth Authentication

Social authentication via OAuth providers.

**Currently Supported:**
- GitHub OAuth

**Configuration:**
```typescript
// .env.local
GITHUB_CLIENT_ID=your_client_id
GITHUB_CLIENT_SECRET=your_client_secret
```

### 4. Passkey Authentication

WebAuthn/FIDO2 passkey support for passwordless authentication.

**Features:**
- Biometric authentication
- Hardware security keys
- Platform authenticators

### 5. Two-Factor Authentication (2FA)

Time-based One-Time Password (TOTP) for additional security.

**Features:**
- TOTP-based 2FA
- Backup codes
- QR code generation

## Authentication Flow

### Sign Up Flow

```
1. User submits email/password
   ↓
2. Better Auth validates input
   ↓
3. Password is hashed
   ↓
4. User record created
   ↓
5. Verification email sent (via queue)
   ↓
6. User clicks verification link
   ↓
7. Email verified
   ↓
8. User can sign in
```

### Sign In Flow

```
1. User submits credentials
   ↓
2. Better Auth validates
   ↓
3. Session created
   ↓
4. Access token generated
   ↓
5. Token stored in Redis (secondary storage)
   ↓
6. Session cookie set
   ↓
7. User authenticated
```

### Magic Link Flow

```
1. User requests magic link
   ↓
2. Link generated with token
   ↓
3. Email sent (via queue)
   ↓
4. User clicks link
   ↓
5. Token validated
   ↓
6. Session created
   ↓
7. User authenticated
```

## Better Auth Configuration

### Core Configuration

Located in `src/config/auth/better-auth.config.ts`:

```typescript
export function getConfig({
  configService,
  cacheService,
  authService,
}: {
  configService: ConfigService<GlobalConfig>;
  cacheService: CacheService;
  authService: AuthService;
}): BetterAuthOptions {
  return {
    appName: appConfig.name,
    secret: authConfig.authSecret,
    baseURL: appConfig.url,
    // ... configuration
  };
}
```

### Enabled Plugins

1. **Username Plugin**: Username validation and management
2. **Magic Link Plugin**: Passwordless authentication
3. **Two Factor Plugin**: 2FA support
4. **Passkey Plugin**: WebAuthn support
5. **OpenAPI Plugin**: API documentation (development only)

### Database Schema

Better Auth uses the following tables:

- `user` - User accounts
- `session` - Active sessions
- `account` - OAuth account links
- `verification` - Email verification tokens
- `twoFactor` - 2FA secrets
- `passkey` - Passkey credentials

## Authentication Endpoints

Better Auth provides endpoints at `/api/auth/*`:

### Sign Up
```
POST /api/auth/sign-up
Body: { email, password, name? }
```

### Sign In
```
POST /api/auth/sign-in
Body: { email, password }
```

### Sign Out
```
POST /api/auth/sign-out
Headers: Cookie (session)
```

### Magic Link
```
POST /api/auth/send-magic-link
Body: { email }
```

### OAuth
```
GET /api/auth/sign-in/social/github
```

### 2FA
```
POST /api/auth/two-factor/setup
POST /api/auth/two-factor/enable
POST /api/auth/two-factor/verify
```

### Passkey
```
POST /api/auth/passkey/register
POST /api/auth/passkey/verify
```

**Full API Reference**: Visit `/api/auth/reference` in development mode

## Using Authentication in Code

### Protecting Routes

#### REST API

```typescript
import { UseGuards } from '@nestjs/common';
import { AuthGuard } from '@/auth/auth.guard';

@Controller('users')
@UseGuards(AuthGuard) // Protect entire controller
export class UserController {
  @Get('profile')
  async getProfile() {
    // Protected route
  }
}
```

#### GraphQL

```typescript
import { UseGuards } from '@nestjs/common';
import { AuthGuard } from '@/auth/auth.guard';

@Resolver(() => UserSchema)
@UseGuards(AuthGuard) // Protect entire resolver
export class UserResolver {
  @Query(() => UserSchema)
  async whoami() {
    // Protected query
  }
}
```

### Accessing Current User

#### REST API

```typescript
import { CurrentUserSession } from '@/decorators/auth/current-user-session.decorator';

@Get('whoami')
async getCurrentUser(
  @CurrentUserSession('user') user: CurrentUserSession['user'],
) {
  return user; // { id, email, username, ... }
}
```

#### Full Session Access

```typescript
@Get('profile')
async getProfile(
  @CurrentUserSession() session: CurrentUserSession,
) {
  // session.user - User object
  // session.headers - Request headers
  // session.session - Better Auth session
}
```

### Public Routes

Mark routes as public (no authentication required):

```typescript
import { Public } from '@/decorators/public.decorator';

@Get('health')
@Public() // No authentication required
async health() {
  return { status: 'ok' };
}
```

### Optional Authentication

Allow both authenticated and unauthenticated access:

```typescript
import { AuthOptional } from '@/decorators/auth-optional.decorator';

@Get('posts')
@AuthOptional() // Works with or without auth
async getPosts(
  @CurrentUserSession('user') user?: CurrentUserSession['user'],
) {
  // user is undefined if not authenticated
}
```

## Custom Hooks

Better Auth supports hooks for custom logic:

### Before Hooks

```typescript
import { Hook } from '@/decorators/auth/hooks.decorator';

@Hook('before', 'sign-in')
async beforeSignIn(ctx: any) {
  // Custom logic before sign-in
  // Can modify ctx or throw errors
}
```

### After Hooks

```typescript
@Hook('after', 'sign-in')
async afterSignIn(ctx: any) {
  // Custom logic after sign-in
  // Access user via ctx.user
}
```

## Email Integration

### Custom Email Sending

Better Auth hooks into our email service:

```typescript
emailVerification: {
  sendVerificationEmail: async ({ user, url }) => {
    await authService.verifyEmail({ url, userId: user.id });
  },
},
```

### Email Templates

Email templates are React components in `src/shared/mail/templates/`:

- `EmailVerification.tsx` - Email verification
- `SignInMagicLink.tsx` - Magic link
- `ResetPassword.tsx` - Password reset

## Session Management

### Session Storage

Sessions are stored in two places:

1. **Database** (Primary): Better Auth `session` table
2. **Redis** (Secondary): For scalability and performance

### Session Configuration

```typescript
session: {
  freshAge: 0, // Session freshness
  modelName: 'session',
}
```

### Access Token Storage

Access tokens are stored in Redis:

```typescript
secondaryStorage: {
  get: async (key) => {
    return await cacheService.get({ key: 'AccessToken', args: [key] });
  },
  set: async (key, value, ttl) => {
    await cacheService.set(
      { key: 'AccessToken', args: [key] },
      value,
      { ttl: ttl * 1000 }
    );
  },
  delete: async (key) => {
    await cacheService.delete({ key: 'AccessToken', args: [key] });
  },
}
```

## User Entity

The `UserEntity` extends Better Auth's user model:

```typescript
@Entity('user')
export class UserEntity extends BaseModel {
  @Column()
  username: string;

  @Column()
  email: string;

  @Column({ default: false })
  isEmailVerified: boolean;

  @Column({ type: 'enum', enum: Role, default: Role.User })
  role: Role;

  @Column({ nullable: true })
  firstName?: string;

  @Column({ nullable: true })
  lastName?: string;

  @Column({ nullable: true })
  image?: string;

  @Column({ default: false })
  twoFactorEnabled: boolean;
}
```

## Security Considerations

### Password Security

- Passwords are hashed using secure algorithms (handled by Better Auth)
- Never store plain text passwords
- Password reset requires email verification

### Session Security

- Sessions use secure, HTTP-only cookies
- CSRF protection via Better Auth
- Session expiration configured
- Redis storage for scalability

### Token Security

- Access tokens stored in Redis
- Short expiration times
- Secure token generation

### Rate Limiting

Authentication endpoints are protected by rate limiting (configured in ThrottlerModule).

## Testing Authentication

### Unit Tests

```typescript
describe('AuthGuard', () => {
  it('should allow authenticated requests', async () => {
    // Mock authenticated user
    // Test guard allows access
  });

  it('should reject unauthenticated requests', async () => {
    // Test guard rejects access
  });
});
```

### E2E Tests

```typescript
describe('Authentication (e2e)', () => {
  it('should sign up a new user', () => {
    return request(app.getHttpServer())
      .post('/api/auth/sign-up')
      .send({ email: 'test@example.com', password: 'password123' })
      .expect(201);
  });
});
```

## Troubleshooting

### Common Issues

**1. "Invalid credentials"**
- Check email/password are correct
- Verify email is verified (if required)
- Check user exists in database

**2. "Session expired"**
- Session may have expired
- User needs to sign in again
- Check session configuration

**3. "CSRF token mismatch"**
- Ensure cookies are enabled
- Check CORS configuration
- Verify trusted origins

**4. "Email not sending"**
- Check MailPit is running (development)
- Verify email configuration
- Check queue processor is running

## Best Practices

1. **Always validate input** - Use DTOs with class-validator
2. **Use guards** - Protect routes with AuthGuard
3. **Access user safely** - Use CurrentUserSession decorator
4. **Handle errors** - Provide meaningful error messages
5. **Log security events** - Log authentication attempts
6. **Rate limit** - Protect authentication endpoints
7. **Use HTTPS** - Always in production
8. **Rotate secrets** - Regularly rotate AUTH_SECRET

## Next Steps

- Review [API Documentation](./06-API.md) for API usage
- Check [Database Guide](./05-DATABASE.md) for user schema
- See [Configuration](./08-CONFIGURATION.md) for auth configuration

