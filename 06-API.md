# API Documentation

This document covers the REST, GraphQL, and WebSocket APIs provided by the application.

## API Overview

The application provides three types of APIs:

1. **REST API** - Traditional RESTful endpoints
2. **GraphQL API** - Flexible GraphQL queries and mutations
3. **WebSocket API** - Real-time bidirectional communication

All APIs are versioned and documented.

## Base URLs

- **REST API**: `http://localhost:3000/api/v1`
- **GraphQL API**: `http://localhost:3000/api/graphql`
- **WebSocket**: `ws://localhost:3000`

## REST API

### API Versioning

REST API uses URI-based versioning:

```
/api/v1/users
/api/v2/users  # Future version
```

### API Documentation

Swagger/OpenAPI documentation is available at:
- **Development**: `http://localhost:3000/api`
- **Protected**: Basic Auth required (see `.env.local`)

### Common Endpoints

#### Health Check

```http
GET /api/health
```

Response:
```json
{
  "status": "ok",
  "info": {
    "database": { "status": "up" },
    "redis": { "status": "up" }
  }
}
```

### User Endpoints

#### Get Current User

```http
GET /api/v1/user/whoami
Authorization: Bearer <token>
```

Response:
```json
{
  "id": "uuid",
  "email": "user@example.com",
  "username": "username",
  "firstName": "John",
  "lastName": "Doe"
}
```

#### List Users (Offset Pagination)

```http
GET /api/v1/user/all?page=1&limit=10
Authorization: Bearer <token>
```

Response:
```json
{
  "data": [
    {
      "id": "uuid",
      "email": "user@example.com",
      "username": "username"
    }
  ],
  "meta": {
    "page": 1,
    "limit": 10,
    "total": 100,
    "totalPages": 10
  }
}
```

#### List Users (Cursor Pagination)

```http
GET /api/v1/user/all/cursor?limit=10&afterCursor=...
Authorization: Bearer <token>
```

Response:
```json
{
  "data": [...],
  "meta": {
    "count": 10,
    "afterCursor": "...",
    "beforeCursor": null
  }
}
```

#### Get User by ID

```http
GET /api/v1/user/:id
Authorization: Bearer <token>
```

#### Update User Profile

```http
PATCH /api/v1/user/profile
Authorization: Bearer <token>
Content-Type: application/json

{
  "firstName": "John",
  "lastName": "Doe",
  "username": "johndoe"
}
```

#### Delete User

```http
DELETE /api/v1/user/:id
Authorization: Bearer <token>
```

### File Upload Endpoints

#### Upload File

```http
POST /api/v1/file/upload
Authorization: Bearer <token>
Content-Type: multipart/form-data

file: <file>
```

Response:
```json
{
  "url": "https://...",
  "filename": "image.jpg",
  "mimetype": "image/jpeg",
  "size": 12345
}
```

### Authentication Endpoints

Better Auth provides authentication endpoints at `/api/auth/*`:

- `POST /api/auth/sign-up` - Sign up
- `POST /api/auth/sign-in` - Sign in
- `POST /api/auth/sign-out` - Sign out
- `POST /api/auth/send-magic-link` - Send magic link
- `GET /api/auth/sign-in/social/github` - GitHub OAuth
- And more...

See [Authentication Documentation](./04-AUTHENTICATION.md) for details.

### Request/Response Format

#### Request Headers

```http
Authorization: Bearer <token>
Content-Type: application/json
Accept: application/json
```

#### Success Response

```json
{
  "data": { ... },
  "meta": { ... }  // If paginated
}
```

#### Error Response

```json
{
  "statusCode": 404,
  "message": "User not found",
  "error": "Not Found",
  "timestamp": "2024-01-01T00:00:00.000Z",
  "path": "/api/v1/user/123"
}
```

### Pagination

#### Offset Pagination

Query parameters:
- `page` - Page number (default: 1)
- `limit` - Items per page (default: 10)

#### Cursor Pagination

Query parameters:
- `limit` - Items per page
- `afterCursor` - Cursor for next page
- `beforeCursor` - Cursor for previous page

## GraphQL API

### GraphQL Playground

Available at `http://localhost:3000/api/graphql` in development.

### Schema

GraphQL schema is auto-generated from resolvers and available at:
- Schema file: `src/generated/schema.generated.gql`

### Queries

#### Get Current User

```graphql
query {
  whoami {
    id
    email
    username
    firstName
    lastName
  }
}
```

#### Get Users

```graphql
query {
  getUsers {
    id
    email
    username
  }
}
```

#### Get User by ID

```graphql
query {
  getUser(id: "uuid") {
    id
    email
    username
  }
}
```

### Mutations

#### Delete User

```graphql
mutation {
  deleteUser(input: { id: "uuid" }) {
    id
    email
  }
}
```

### Authentication

GraphQL endpoints require authentication via `AuthGuard`:

```typescript
@UseGuards(AuthGuard)
@Resolver(() => UserSchema)
export class UserResolver {
  // Protected resolvers
}
```

### Error Handling

GraphQL returns errors in standard format:

```json
{
  "errors": [
    {
      "message": "User not found",
      "extensions": {
        "code": "NOT_FOUND"
      }
    }
  ],
  "data": null
}
```

## WebSocket API

### Connection

Connect to WebSocket server:

```javascript
import io from 'socket.io-client';

const socket = io('http://localhost:3000', {
  auth: {
    token: 'your-auth-token'
  }
});
```

### Authentication

WebSocket connections are authenticated via handshake:

```typescript
// Server-side authentication
socket.use((socket, next) => {
  const token = socket.handshake.auth.token;
  // Verify token
  next();
});
```

### Events

#### Client → Server

```javascript
// Join room
socket.emit('join', { room: 'user-room' });

// Send message
socket.emit('message', { text: 'Hello' });
```

#### Server → Client

```javascript
// Listen for messages
socket.on('message', (data) => {
  console.log('Received:', data);
});

// Listen for user joined
socket.on('userJoined', (user) => {
  console.log('User joined:', user);
});
```

### Rooms

Organize connections into rooms:

```typescript
// Server-side
socket.join('user-room');
socket.to('user-room').emit('message', data);
```

### Redis Adapter

WebSocket uses Redis adapter for multi-instance support:

```typescript
// Multiple server instances can communicate
// via Redis pub/sub
```

## API Versioning

### REST API Versioning

Version specified in controller:

```typescript
@Controller({
  path: 'user',
  version: '1',  // /api/v1/user
})
```

### GraphQL Versioning

GraphQL doesn't use versioning. Instead:
- Add new fields (backward compatible)
- Deprecate old fields
- Use schema evolution

## Error Handling

### HTTP Status Codes

- `200` - Success
- `201` - Created
- `400` - Bad Request
- `401` - Unauthorized
- `403` - Forbidden
- `404` - Not Found
- `422` - Unprocessable Entity (validation errors)
- `500` - Internal Server Error

### Validation Errors

Validation errors return detailed information:

```json
{
  "statusCode": 422,
  "message": [
    {
      "property": "email",
      "constraints": {
        "isEmail": "email must be an email"
      }
    }
  ]
}
```

## Rate Limiting

All API endpoints are protected by rate limiting:

- **Limit**: Configurable (default: 100 requests)
- **Window**: 60 seconds
- **Storage**: Redis

Rate limit headers:
```
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 99
X-RateLimit-Reset: 1234567890
```

## CORS Configuration

CORS is configured per environment:

```typescript
app.enableCors({
  origin: configService.get('app.corsOrigin'),
  credentials: true,
});
```

## API Testing

### Using Swagger UI

1. Open `http://localhost:3000/api`
2. Authenticate (click "Authorize")
3. Test endpoints directly

### Using cURL

```bash
# Get current user
curl -X GET http://localhost:3000/api/v1/user/whoami \
  -H "Authorization: Bearer <token>"
```

### Using Postman

1. Import OpenAPI spec from Swagger
2. Set up authentication
3. Test endpoints

### Using GraphQL Playground

1. Open `http://localhost:3000/api/graphql`
2. Write queries/mutations
3. Execute and view results

## Best Practices

1. **Use proper HTTP methods** - GET, POST, PATCH, DELETE
2. **Return appropriate status codes** - Follow REST conventions
3. **Validate input** - Use DTOs with class-validator
4. **Handle errors gracefully** - Provide meaningful error messages
5. **Use pagination** - For list endpoints
6. **Version APIs** - For breaking changes
7. **Document APIs** - Keep Swagger docs updated
8. **Rate limit** - Protect against abuse
9. **Authenticate** - Protect sensitive endpoints
10. **Log requests** - For debugging and monitoring

## Next Steps

- Review [Authentication Documentation](./04-AUTHENTICATION.md) for auth endpoints
- Check [Development Workflow](./02-DEVELOPMENT-WORKFLOW.md) for API development
- See [Configuration](./08-CONFIGURATION.md) for API configuration

