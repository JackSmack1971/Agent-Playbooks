---
trigger: ["model_decision"]
description: Comprehensive rules for integrating PostgreSQL, Redis, and Qdrant vector database backends with focus on security, performance, and production best practices
globs: ["**/*.py", "**/*.js", "**/*.ts", "**/*.java", "**/*.go", "**/*.cs", "**/*.sql"]
version: "1.0.0"
last_updated: "2025-01-01"
---

# Storage Backends: PostgreSQL, Redis, Vector DB (Qdrant) Integration

## Installation & Setup

- **ALWAYS** use connection pooling for production applications
- **CONFIGURE** SSL/TLS for all database connections in production
- **STORE** credentials securely using environment variables or secret management
- **IMPLEMENT** proper authentication and authorization

```bash
# PostgreSQL
sudo apt-get install postgresql postgresql-contrib

# Redis
sudo apt-get install redis-server

# Qdrant (using Docker)
docker run -p 6333:6333 qdrant/qdrant
```

## Configuration & Initialization

### PostgreSQL Configuration

```sql
-- Connection string with security best practices
postgresql://username:password@host:5432/database?sslmode=verify-full&sslcert=client.crt&sslkey=client.key&sslrootcert=ca.crt

-- Connection pooling configuration
max_connections = 200
shared_buffers = 256MB
effective_cache_size = 1GB
maintenance_work_mem = 64MB
```

### Redis Configuration

```javascript
// Node.js with ioredis - Optimal configuration
const Redis = require('ioredis');
const cluster = new Redis.Cluster([
  { host: 'redis-host', port: 6379 }
], {
  redisOptions: {
    password: process.env.REDIS_PASSWORD,
    tls: {}, // Enable TLS
    connectTimeout: 1000,
    commandTimeout: 2000,
    retryDelayOnFailover: 100
  },
  enableReadyCheck: true,
  maxRetriesPerRequest: 3
});
```

### Qdrant Configuration

```python
# Python client with security best practices
from qdrant_client import QdrantClient

client = QdrantClient(
    url="https://qdrant-cluster.example.com:6333",
    api_key=os.getenv("QDRANT_API_KEY"),
    timeout=30.0,
    https=True
)
```

## Core Concepts / API Usage

### Connection Management

- **PostgreSQL**: Always use connection pooling for production applications to avoid connection overhead
- **Redis**: Use connection pooling to reuse existing connections and avoid TCP handshake overhead
- **Qdrant**: Configure proper timeouts and connection limits for vector operations

### Security Best Practices

- **PostgreSQL**: Use `sslmode=verify-full` for client connections to prevent MITM attacks
- **Redis**: Never run Redis without authentication in production: Set `requirepass` directive
- **Qdrant**: Always use API keys for authentication in production environments

## Security & Permissions

### Network Security

- Use VPC/private networks to isolate database traffic
- Configure firewall rules allowing only necessary ports:
  - PostgreSQL: 5432
  - Redis: 6379 (or custom)
  - Qdrant: 6333 (REST), 6334 (gRPC)
- Implement network segmentation between application tiers

### Authentication Strategy

- Use centralized secret management (AWS Secrets Manager, Azure Key Vault, etc.)
- Rotate credentials regularly using automated processes
- Implement service-to-service authentication with certificates or tokens
- **Never use** default credentials or passwords in production

## Performance & Scalability

### Connection Pool Sizing

- **PostgreSQL**: Start with `max_connections = 2 * CPU_cores`, tune based on workload
- **Redis**: Use 8-20 connections per application instance, enable connection multiplexing
- **Qdrant**: Configure client-side connection pools with reasonable timeouts

### Resource Management

- Monitor connection pool utilization and adjust sizes dynamically
- Implement circuit breakers for failing database connections
- Use read replicas for PostgreSQL to distribute read load
- Configure Redis memory policies to prevent OOM conditions

## Error Handling & Troubleshooting

### Connection Exhaustion

**Problem**: Applications running out of database connections
**Solution**:
- Implement proper connection pooling with appropriate pool sizes
- Use connection pool monitoring and alerting
- Implement connection timeouts to prevent hanging connections
- Use connection multiplexing where supported

### Security Vulnerabilities

**Problem**: Exposed databases with weak authentication
**Solution**:
- Always use strong passwords and API keys
- Enable SSL/TLS for all connections
- Implement network isolation and firewall rules
- Regular security audits and penetration testing

### Performance Bottlenecks

**Problem**: Slow query performance across systems
**Solution**:
- Implement proper indexing strategies
- Use connection pooling and multiplexing
- Monitor resource utilization (CPU, memory, I/O)
- Optimize queries and batch operations where possible

## Testing

### Health Checks

```python
# Comprehensive health check implementation
async def check_storage_health():
    health_status = {}

    # PostgreSQL health check
    try:
        async with asyncpg.connect(pg_dsn) as conn:
            await conn.fetchval("SELECT 1")
        health_status['postgresql'] = 'healthy'
    except Exception as e:
        health_status['postgresql'] = f'unhealthy: {e}'

    # Redis health check
    try:
        response = await redis_client.ping()
        health_status['redis'] = 'healthy' if response else 'unhealthy'
    except Exception as e:
        health_status['redis'] = f'unhealthy: {e}'

    # Qdrant health check
    try:
        collections = await qdrant_client.get_collections()
        health_status['qdrant'] = 'healthy'
    except Exception as e:
        health_status['qdrant'] = f'unhealthy: {e}'

    return health_status
```

### Performance Metrics

- Monitor connection pool utilization for all databases
- Track query performance and slow query logs
- Monitor memory and CPU usage for Redis
- Track vector search latency and throughput for Qdrant
- Implement distributed tracing for cross-system operations

## Deployment & Production Patterns

### SSL/TLS Configuration

```yaml
# Docker Compose example with proper TLS
version: '3.8'
services:
  postgres:
    image: postgres:17
    environment:
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_SSL_MODE: require
    volumes:
      - ./certs:/var/lib/postgresql/certs
    command: >
      postgres
      -c ssl=on
      -c ssl_cert_file=/var/lib/postgresql/certs/server.crt
      -c ssl_key_file=/var/lib/postgresql/certs/server.key

  redis:
    image: redis:7
    command: >
      redis-server
      --requirepass ${REDIS_PASSWORD}
      --tls-port 6380
      --tls-cert-file /tls/redis.crt
      --tls-key-file /tls/redis.key
    volumes:
      - ./tls:/tls
```

### Production Deployment Checklist

#### Security
- [ ] SSL/TLS enabled for all database connections
- [ ] Strong authentication configured (no default passwords)
- [ ] Network isolation implemented (VPC, firewall rules)
- [ ] Secrets stored in secure management systems
- [ ] Regular security patches applied

#### Performance
- [ ] Connection pooling configured for all databases
- [ ] Appropriate resource limits set (memory, CPU)
- [ ] Monitoring and alerting implemented
- [ ] Load testing completed under expected traffic
- [ ] Backup and recovery procedures tested

#### Reliability
- [ ] Health checks implemented for all services
- [ ] Circuit breakers configured for external dependencies
- [ ] Error handling and retry logic implemented
- [ ] Disaster recovery procedures documented
- [ ] Regular maintenance windows scheduled

## Known Issues & Mitigations

### Common Integration Issues

- **Connection Exhaustion**: Monitor pool sizes and implement proper connection management
- **Security Vulnerabilities**: Regular audits and credential rotation
- **Performance Bottlenecks**: Query optimization and resource monitoring
- **Data Consistency Issues**: Implement transactional patterns and reconciliation processes

### Troubleshooting & Monitoring

- Implement comprehensive health checks for all storage backends
- Monitor connection pool utilization and resource usage
- Track query performance and implement slow query logging
- Use distributed tracing for cross-system operations

## Version Compatibility Notes

- **PostgreSQL**: 17.x (latest stable)
- **Redis**: 7.x (latest stable)
- **Qdrant**: Latest stable release
- **Breaking changes**: Major version updates may require configuration changes
- **Security updates**: Regular patching for security vulnerabilities

## References

- [PostgreSQL Documentation](https://www.postgresql.org/docs/)
- [Redis Documentation](https://redis.io/documentation)
- [Qdrant Documentation](https://qdrant.tech/documentation/)
- [Database Security Best Practices](https://owasp.org/www-project-cheat-sheets/cheatsheets/Database_Security_Cheat_Sheet.html)