# Redis Commander Setup Guide

## Overview

Redis Commander is a web-based Redis management interface that provides an intuitive UI for managing Redis databases. It has been integrated into the X's Docker Compose stack.

## Service Configuration

The Redis Commander service is configured in `docker-compose.yml`:

```yaml
redis-commander:
  image: rediscommander/redis-commander:latest
  container_name: foundation-redis-commander-container
  restart: always
  networks:
    - default
  depends_on:
    redis:
      condition: service_healthy
  environment:
    - REDIS_HOST=redis
    - REDIS_PORT=6379
    - REDIS_PASSWORD=${REDIS_PASSWORD}
    - HTTP_USER=admin
    - HTTP_PASSWORD=admin
  expose:
    - 8081
  ports:
    - "8084:8081"
  healthcheck:
    test: ["CMD", "wget", "--no-verbose", "--tries=1", "--spider", "http://localhost:8081"]
    interval: 30s
    timeout: 10s
    retries: 3
    start_period: 40s
  <<: *default-logging
```

## Access Information

### Web Interface
- **URL**: http://localhost:8084
- **HTTP Username**: `admin`
- **HTTP Password**: `admin`

### Redis Connection
The service automatically connects to the Redis container with the following configuration:
- **Host**: `redis` (internal Docker network)
- **Port**: `6379`
- **Password**: Retrieved from `${REDIS_PASSWORD}` environment variable

## Features

Redis Commander provides the following features:

1. **Key Browser**: Browse all Redis keys with filtering and search capabilities
2. **Key Editor**: View and edit key values (strings, hashes, lists, sets, sorted sets)
3. **Key Operations**: Create, delete, rename, and expire keys
4. **Database Selection**: Switch between different Redis databases (0-15)
5. **Command Execution**: Execute Redis commands directly
6. **Connection Management**: Manage multiple Redis connections

## Usage

### Starting the Service

The Redis Commander service starts automatically when you start the Docker Compose stack:

```bash
docker compose up -d
```

Or start it individually:

```bash
docker compose up -d redis-commander
```

### Accessing the Interface

1. Open your browser and navigate to: http://localhost:8084
2. Enter the HTTP authentication credentials:
   - Username: `admin`
   - Password: `admin`
3. The Redis connection is pre-configured, so you can immediately start browsing keys

### Common Operations

#### View Email Queue Status

Redis Commander is particularly useful for monitoring the email queue system:

1. Navigate to the key browser
2. Search for keys matching `email:queue:*`
3. Common keys to check:
   - `email:queue:priority` - Priority email queue (list)
   - `email:queue:pending` - Pending email queue (list)
   - `email:queue:processing` - Currently processing emails (sorted set)
   - `email:queue:failed` - Failed emails (sorted set)
   - `email:queue:stats` - Email queue statistics (hash)

#### Check Queue Lengths

1. Select a queue key (e.g., `email:queue:priority`)
2. View the list length in the key details
3. Inspect individual queue items

#### Monitor Processing Status

1. Check `email:queue:processing` sorted set
2. View timestamps and email metadata
3. Identify stuck or long-running email processing

## Security Notes

1. **HTTP Authentication**: The web interface is protected with HTTP basic authentication (admin/admin)
2. **Network Isolation**: Redis Commander only exposes port 8084 to the host, Redis connection is internal
3. **Password Protection**: Redis password is retrieved from environment variables, not hardcoded
4. **Production Considerations**: In production, consider:
   - Changing default HTTP credentials
   - Using environment-specific passwords
   - Restricting access via firewall rules
   - Using HTTPS with reverse proxy

## Troubleshooting

### Service Not Starting

Check service logs:

```bash
docker compose logs redis-commander
```

### Cannot Connect to Redis

1. Verify Redis service is running:
   ```bash
   docker compose ps redis
   ```

2. Check Redis health:
   ```bash
   docker compose exec redis redis-cli -a ${REDIS_PASSWORD} ping
   ```

3. Verify environment variables:
   ```bash
   docker compose exec redis-commander env | grep REDIS
   ```

### Web Interface Not Accessible

1. Check if port 8084 is already in use:
   ```bash
   lsof -i :8084
   ```

2. Verify service is running:
   ```bash
   docker compose ps redis-commander
   ```

3. Check service health:
   ```bash
   docker compose exec redis-commander wget -O- http://localhost:8081
   ```

## Related Services

- **Redis**: The Redis database service (port 6379, internal)
- **Adminer**: PostgreSQL database management (http://localhost:8082)
- **Mongo Express**: MongoDB management (http://localhost:8081)

## References

- [Redis Commander GitHub](https://github.com/joeferner/redis-commander)
- [Redis Commander Docker Hub](https://hub.docker.com/r/rediscommander/redis-commander)
- [Redis Documentation](https://redis.io/documentation)

