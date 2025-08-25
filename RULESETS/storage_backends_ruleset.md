---
trigger: model_decision
description: Comprehensive rules for integrating PostgreSQL, Redis, and Qdrant vector database backends with focus on security, performance, and production best practices
---

# Storage Backends: PostgreSQL, Redis, Vector DB (Qdrant) Integration

## PostgreSQL Configuration & Integration

### Connection Management
- **Always use connection pooling** for production applications to avoid connection overhead
- Configure PostgreSQL connection pools with `max_connections` based on server capacity (typically 2x CPU cores as starting point)
- Use PgBouncer or application-level pooling for high-traffic applications
- Set `idle_in_transaction_session_timeout` to prevent hung connections
- Configure `log_connections` and `log_disconnections` for connection monitoring

### SSL/TLS Security Configuration
- **Mandatory SSL/TLS** for production: Set `ssl = on` in `postgresql.conf`
- Use `sslmode=verify-full` for client connections to prevent MITM attacks
- Configure SSL certificates: `ssl_cert_file`, `ssl_key_file`, `ssl_ca_file`
- Set minimum TLS version: `ssl_min_protocol_version = 'TLSv1.2'`
- **Never use** `sslmode=disable` or `sslmode=allow` in production
- Set restrictive SSL cipher suites: `ssl_ciphers = 'HIGH:MEDIUM:+3DES:!aNULL'`

### Authentication & Authorization
- Use strong password authentication with `scram-sha-256` (default in PostgreSQL 14+)
- Configure `pg_hba.conf` with `hostssl` entries for encrypted connections only
- Implement role-based access control (RBAC) with principle of least privilege
- **Never use** `trust` authentication method in production
- Set `password_encryption = scram-sha-256` for secure password storage

### Performance Optimization
- Configure `shared_buffers` to 25% of available RAM (max 8GB)
- Set `work_mem` based on concurrent queries and available RAM
- Enable `track_activity_query_size = 1024` for query monitoring
- Use `VACUUM` and `ANALYZE` automation with `autovacuum = on`
- Configure `checkpoint_completion_target = 0.9` for I/O smoothing

### Integration Patterns
```sql
-- Connection string with security best practices
postgresql://username:password@host:5432/database?sslmode=verify-full&sslcert=client.crt&sslkey=client.key&sslrootcert=ca.crt

-- Connection pooling configuration
max_connections = 200
shared_buffers = 256MB
effective_cache_size = 1GB
maintenance_work_mem = 64MB
```

## Redis Configuration & Integration

### Connection Pooling & Management
- **Always use connection pooling** to reuse existing connections and avoid TCP handshake overhead
- Configure pool size based on application concurrency needs (start with 8-10 connections per service)
- Set connection timeouts: `ConnectTimeout = 1000ms`, `SyncTimeout = 2000ms`
- Use `ConnectionMultiplexer` pattern for .NET or equivalent pooling for other languages
- **Avoid** creating separate connections per operation - leads to connection exhaustion

### Security Configuration
- **Never run Redis without authentication** in production: Set `requirepass` directive
- Use Redis ACL for fine-grained access control (Redis 6.0+)
- Configure `bind` directive to specific interfaces, not `0.0.0.0`
- Enable TLS with `tls-port`, `tls-cert-file`, and `tls-key-file` for encrypted connections
- Set `protected-mode yes` to prevent unauthorized access

### Performance Optimization
- Configure `maxmemory` and appropriate `maxmemory-policy` (e.g., `allkeys-lru`)
- Use Redis persistence strategy: AOF for durability, RDB for performance
- Enable TCP keepalives: `tcp-keepalive = 300`
- Set appropriate `timeout` for idle client connections (300 seconds default)
- Monitor memory usage and implement proper key expiration policies

### Connection Pool Examples
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

```python
# Python with redis-py - Connection pool setup
import redis
pool = redis.ConnectionPool(
    host='redis-host',
    port=6379,
    password=os.getenv('REDIS_PASSWORD'),
    ssl=True,
    ssl_cert_reqs='required',
    ssl_ca_certs='/path/to/ca.crt',
    max_connections=20,
    socket_connect_timeout=5,
    socket_timeout=5
)
client = redis.Redis(connection_pool=pool)
```

### Redis Integration Anti-Patterns
- **Don't** use `KEYS` command in production - use `SCAN` instead
- **Avoid** large `HGETALL` operations - use `HSCAN` for large hashes
- **Never** use `FLUSHALL` or `FLUSHDB` without proper safeguards
- **Don't** store large objects (>1MB) directly - use compression or chunking

## Qdrant Vector Database Configuration

### Client Initialization & Authentication
- **Always use API keys** for authentication in production environments
- Enable HTTPS/TLS for all client connections: `https: true`
- Configure proper timeouts: `timeout: 30.0` seconds for large operations
- Use connection URLs with explicit schema: `https://` instead of `http://`

### Security Best Practices
- Store API keys in environment variables, never hardcode in source
- Use TLS certificates for production deployments
- Configure proper firewall rules to restrict access to Qdrant ports (6333, 6334)
- Enable mutual TLS (mTLS) for client certificate authentication when required

### Performance Optimization
- Configure search parameters: `hnsw_ef` for recall vs speed tradeoff
- Use appropriate collection configuration: `vector_size`, `distance` metric
- Implement proper indexing thresholds: `indexing_threshold: 20000`
- Monitor memory usage and configure appropriate resource limits

### Client Configuration Examples
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

```javascript
// JavaScript client configuration
import { QdrantClient } from "@qdrant/js-client-rest";

const client = new QdrantClient({
  url: "https://qdrant-cluster.example.com",
  port: 6333,
  apiKey: process.env.QDRANT_API_KEY
});
```

```java
// Java client with gRPC and security
import io.qdrant.client.QdrantClient;
import io.qdrant.client.QdrantGrpcClient;

QdrantClient client = new QdrantClient(
    QdrantGrpcClient.newBuilder(
        "qdrant-cluster.example.com",
        6334,
        true) // Enable TLS
    .withApiKey(System.getenv("QDRANT_API_KEY"))
    .build()
);
```

## Cross-System Security Integration

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
- **Never** use default credentials or passwords in production

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

## Performance Optimization Patterns

### Connection Pool Sizing
- **PostgreSQL**: Start with `max_connections = 2 * CPU_cores`, tune based on workload
- **Redis**: Use 8-20 connections per application instance, enable connection multiplexing
- **Qdrant**: Configure client-side connection pools with reasonable timeouts

### Resource Management
- Monitor connection pool utilization and adjust sizes dynamically
- Implement circuit breakers for failing database connections
- Use read replicas for PostgreSQL to distribute read load
- Configure Redis memory policies to prevent OOM conditions

### Monitoring & Alerting
```sql
-- PostgreSQL connection monitoring
SELECT state, count(*) 
FROM pg_stat_activity 
GROUP BY state;

-- Check for connection pool exhaustion
SELECT count(*) as active_connections,
       setting::int as max_connections,
       round(100.0 * count(*) / setting::int, 2) as percent_used
FROM pg_stat_activity, pg_settings 
WHERE name = 'max_connections';
```

## Common Integration Issues & Solutions

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

### Data Consistency Issues
**Problem**: Inconsistent data across PostgreSQL, Redis, and Qdrant
**Solution**:
- Implement transactional patterns with rollback mechanisms
- Use event-driven architecture for data synchronization
- Implement proper error handling and retry logic
- Monitor data drift and implement reconciliation processes

## Troubleshooting & Monitoring

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

### Error Handling Patterns
- Implement exponential backoff for connection retries
- Use circuit breakers to prevent cascade failures
- Log detailed error information for debugging
- Implement graceful degradation when services are unavailable

## Production Deployment Checklist

### Security
- [ ] SSL/TLS enabled for all database connections
- [ ] Strong authentication configured (no default passwords)
- [ ] Network isolation implemented (VPC, firewall rules)
- [ ] Secrets stored in secure management systems
- [ ] Regular security patches applied

### Performance
- [ ] Connection pooling configured for all databases
- [ ] Appropriate resource limits set (memory, CPU)
- [ ] Monitoring and alerting implemented
- [ ] Load testing completed under expected traffic
- [ ] Backup and recovery procedures tested

### Reliability
- [ ] Health checks implemented for all services
- [ ] Circuit breakers configured for external dependencies
- [ ] Error handling and retry logic implemented
- [ ] Disaster recovery procedures documented
- [ ] Regular maintenance windows scheduled

---

**Version Information**:
- PostgreSQL: 17.x (latest stable)
- Redis: 7.x (latest stable)  
- Qdrant: Latest stable release
- Last Updated: January 2025