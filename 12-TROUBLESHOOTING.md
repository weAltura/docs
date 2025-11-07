# Troubleshooting Guide

Common issues and their solutions.

## Quick Reference

### Service Status

```bash
# Check Docker services
docker ps

# Check service logs
docker logs wealtura-postgres
docker logs wealtura-redis
docker logs wealtura-api

# Check application logs
pnpm docker:services:logs
```

### Database

```bash
# Check database connection
psql -h localhost -U postgres -d wealtura

# Check migrations
pnpm migration:show

# Check database logs
docker logs wealtura-postgres
```

### Application

```bash
# Check if server is running
curl http://localhost:3000/api/health

# Check application logs
# (View in terminal or log files)
```

## Common Issues

### Application Won't Start

#### Issue: Port Already in Use

**Symptoms:**
```
Error: listen EADDRINUSE: address already in use :::3000
```

**Solutions:**

1. **Find process using port:**
   ```bash
   lsof -i :3000
   # or
   netstat -an | grep 3000
   ```

2. **Kill the process:**
   ```bash
   kill -9 <PID>
   ```

3. **Or change port:**
   ```bash
   # In .env.local
   APP_PORT=3001
   ```

#### Issue: Module Not Found

**Symptoms:**
```
Error: Cannot find module '...'
```

**Solutions:**

1. **Reinstall dependencies:**
   ```bash
   rm -rf node_modules
   pnpm install
   ```

2. **Clear build cache:**
   ```bash
   rm -rf dist
   pnpm build
   ```

3. **Check import paths:**
   - Verify `@/` alias is configured
   - Check `tsconfig.json` paths

#### Issue: Configuration Error

**Symptoms:**
```
Error: Configuration validation failed
```

**Solutions:**

1. **Check environment variables:**
   ```bash
   # Verify .env.local exists
   cat .env.local
   ```

2. **Check required variables:**
   - See [Configuration Reference](./08-CONFIGURATION.md)
   - Verify all required variables are set

3. **Check variable types:**
   - Numbers should be numbers (not strings)
   - Booleans should be `true`/`false` (not `1`/`0`)

### Database Issues

#### Issue: Connection Refused

**Symptoms:**
```
Error: connect ECONNREFUSED 127.0.0.1:5432
```

**Solutions:**

1. **Check Docker service:**
   ```bash
   docker ps | grep postgres
   ```

2. **Start Docker services:**
   ```bash
   pnpm docker:services:up
   ```

3. **Check connection string:**
   ```bash
   # Verify in .env.local
   DATABASE_HOST=localhost
   DATABASE_PORT=5432
   ```

4. **Check Docker port mapping:**
   ```bash
   # Verify in .env.docker
   DOCKER_DATABASE_PORT=5432
   ```

#### Issue: Authentication Failed

**Symptoms:**
```
Error: password authentication failed for user
```

**Solutions:**

1. **Verify credentials:**
   ```bash
   # Check .env.local matches Docker environment
   DATABASE_USERNAME=postgres
   DATABASE_PASSWORD=postgres
   ```

2. **Reset database password:**
   ```bash
   docker exec -it wealtura-postgres psql -U postgres
   ALTER USER postgres WITH PASSWORD 'new_password';
   ```

#### Issue: Migration Fails

**Symptoms:**
```
Error: Migration failed
```

**Solutions:**

1. **Check migration SQL:**
   ```bash
   # Review migration file
   cat src/database/migrations/...ts
   ```

2. **Check database state:**
   ```bash
   pnpm migration:show
   ```

3. **Rollback and retry:**
   ```bash
   pnpm migration:down
   pnpm migration:up
   ```

4. **Check for conflicts:**
   - Verify no conflicting migrations
   - Check database schema matches expected state

### Redis Issues

#### Issue: Connection Refused

**Symptoms:**
```
Error: connect ECONNREFUSED 127.0.0.1:6379
```

**Solutions:**

1. **Check Docker service:**
   ```bash
   docker ps | grep redis
   ```

2. **Start Docker services:**
   ```bash
   pnpm docker:services:up
   ```

3. **Check connection:**
   ```bash
   redis-cli -h localhost -p 6379 -a <password> ping
   ```

#### Issue: Authentication Failed

**Symptoms:**
```
Error: NOAUTH Authentication required
```

**Solutions:**

1. **Verify password:**
   ```bash
   # Check .env.local
   REDIS_PASSWORD=your_password
   ```

2. **Check Docker Redis password:**
   ```bash
   # Should match REDIS_PASSWORD
   ```

### Authentication Issues

#### Issue: Invalid Credentials

**Symptoms:**
```
401 Unauthorized
```

**Solutions:**

1. **Check token:**
   - Verify token is valid
   - Check token hasn't expired
   - Verify token format

2. **Check session:**
   - Verify session exists in database
   - Check Redis for access token

3. **Re-authenticate:**
   - Sign out and sign in again
   - Clear cookies and retry

#### Issue: CSRF Token Mismatch

**Symptoms:**
```
403 Forbidden - CSRF token mismatch
```

**Solutions:**

1. **Check cookies:**
   - Ensure cookies are enabled
   - Check cookie domain matches

2. **Check CORS:**
   - Verify CORS origins include your domain
   - Check credentials are enabled

3. **Check trusted origins:**
   - Verify `APP_CORS_ORIGIN` includes your domain

### Email Issues

#### Issue: Email Not Sending

**Symptoms:**
- No emails received
- Email job stuck in queue

**Solutions:**

1. **Check MailPit (development):**
   ```bash
   # Verify MailPit is running
   docker ps | grep mail

   # Check MailPit UI
   open http://localhost:8025
   ```

2. **Check email configuration:**
   ```bash
   # Verify in .env.local
   MAIL_HOST=localhost
   MAIL_PORT=1025
   ```

3. **Check worker process:**
   ```bash
   # Verify worker is running
   IS_WORKER=true pnpm start
   ```

4. **Check queue:**
   - View Bull Board: `http://localhost:3000/api/queues`
   - Check for failed jobs

### Performance Issues

#### Issue: Slow Queries

**Symptoms:**
- API responses are slow
- Database queries take long

**Solutions:**

1. **Enable query logging:**
   ```bash
   DATABASE_LOGGING=true
   ```

2. **Check for missing indexes:**
   ```bash
   # Review entity indexes
   # Add indexes on frequently queried columns
   ```

3. **Optimize queries:**
   - Use query builder for complex queries
   - Avoid N+1 queries
   - Use pagination

#### Issue: High Memory Usage

**Symptoms:**
- Application uses too much memory
- Out of memory errors

**Solutions:**

1. **Check connection pool:**
   ```bash
   # Reduce if too high
   DATABASE_MAX_CONNECTIONS=50
   ```

2. **Check cache:**
   - Review cache TTL
   - Limit cache size

3. **Check for memory leaks:**
   - Review code for leaks
   - Use memory profiler

### Build Issues

#### Issue: TypeScript Errors

**Symptoms:**
```
Type error: ...
```

**Solutions:**

1. **Check TypeScript version:**
   ```bash
   pnpm list typescript
   ```

2. **Restart TypeScript server:**
   - VS Code: Cmd+Shift+P → "TypeScript: Restart TS Server"

3. **Clear TypeScript cache:**
   ```bash
   rm -rf node_modules/.cache
   rm -rf dist
   ```

#### Issue: Build Fails

**Symptoms:**
```
Build failed with errors
```

**Solutions:**

1. **Check for errors:**
   ```bash
   pnpm build
   # Review error messages
   ```

2. **Check dependencies:**
   ```bash
   pnpm install
   ```

3. **Clear build artifacts:**
   ```bash
   rm -rf dist
   pnpm build
   ```

## Debugging Tips

### Enable Debug Logging

```bash
APP_LOG_LEVEL=debug
```

### Database Query Logging

```bash
DATABASE_LOGGING=true
```

### Enable Verbose Logging

```bash
APP_LOG_LEVEL=verbose
```

### Use Debugger

```bash
# Start with debugger
pnpm start:debug

# Attach debugger on port 9229
```

### Check Logs

```bash
# Application logs
# (View in terminal)

# Docker logs
docker logs wealtura-api
docker logs wealtura-postgres
docker logs wealtura-redis
```

## Getting Help

### Check Documentation

1. Review relevant documentation
2. Check configuration reference
3. Review error messages

### Check Logs

1. Application logs
2. Docker logs
3. System logs

### Common Commands

```bash
# Health check
curl http://localhost:3000/api/health

# Check services
docker ps

# View logs
docker logs <container-name>

# Restart services
pnpm docker:services:down
pnpm docker:services:up
```

## Prevention

### Best Practices

1. **Test locally first** - Always test changes locally
2. **Check logs regularly** - Monitor for warnings
3. **Keep dependencies updated** - Regular updates
4. **Backup before changes** - Always backup
5. **Use version control** - Commit frequently
6. **Document changes** - Keep notes on changes

## Next Steps

- Review [Configuration](./08-CONFIGURATION.md) for configuration issues
- Check [Deployment Guide](./10-DEPLOYMENT.md) for deployment issues
- See [Monitoring Guide](./11-MONITORING.md) for monitoring issues

