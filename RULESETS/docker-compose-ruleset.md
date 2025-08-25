---
trigger: glob
description: Comprehensive Docker Compose technical ruleset covering file structure, services, networking, volumes, best practices, and common issue mitigation for production-ready containerized applications.
globs: ["**/compose.yaml", "**/compose.yml", "**/docker-compose.yaml", "**/docker-compose.yml", "**/compose.*.yaml", "**/compose.*.yml"]
---

# Docker Compose Rules

## File Structure and Syntax

### Compose File Naming and Location
- Use `compose.yaml` as the preferred filename (YAML extension)
- Place Compose files in project root or use `-f` flag for custom locations
- For multiple environments, use naming convention: `compose.override.yaml`, `compose.prod.yaml`
- Avoid deprecated `docker-compose.yaml` naming unless legacy compatibility required

### Version Field (DEPRECATED)
- **CRITICAL**: Never include `version:` field in new Compose files (removed in Compose v2)
- Start directly with `services:` block
- Legacy `version: "3.8"` syntax is obsolete and causes warnings

```yaml
# ❌ WRONG - deprecated version field
version: "3.8"
services:
  web:
    image: nginx

# ✅ CORRECT - modern syntax
services:
  web:
    image: nginx
```

### YAML Syntax Standards
- Use 2-space indentation consistently
- Quote strings containing special characters, environment variables, or version numbers
- Use explicit `true`/`false` for boolean values
- Prefer long-form YAML syntax over JSON-style arrays

## Services Configuration

### Image and Build Configuration
- Always specify explicit image tags (avoid `latest`)
- Use official Docker images or verified publishers when possible
- For custom builds, specify `context` and optional `dockerfile`

```yaml
services:
  web:
    # ✅ Explicit tag
    image: nginx:1.25-alpine
    
  app:
    build:
      context: ./app
      dockerfile: Dockerfile.prod
      target: production
```

### Container Naming and Hostname
- Avoid `container_name` unless absolutely necessary (breaks scaling)
- Let Compose generate names automatically: `{project}_{service}_{replica}`
- Use `hostname` only when service discovery requires specific hostnames

### Resource Management
- Always define resource limits for production services
- Set memory limits to prevent OOM conditions
- Configure CPU limits for resource-intensive services

```yaml
services:
  web:
    image: nginx:alpine
    deploy:
      resources:
        limits:
          memory: 512M
          cpus: "0.5"
        reservations:
          memory: 256M
          cpus: "0.25"
```

### Health Checks (MANDATORY)
- Define health checks for all services to enable proper orchestration
- Use appropriate test commands for each service type
- Configure realistic intervals and timeouts

```yaml
services:
  web:
    image: nginx:alpine
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:80/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
      start_interval: 5s
```

### Environment Variables and Secrets
- Use `.env` files for non-sensitive configuration
- Never commit secrets in Compose files or `.env` files
- Use Docker secrets for sensitive data in production

```yaml
services:
  db:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: ${DB_NAME:-myapp}
      POSTGRES_USER: ${DB_USER:-postgres}
    env_file:
      - ./config/.env
    secrets:
      - db_password

secrets:
  db_password:
    file: ./secrets/db_password.txt
```

### Port Configuration
- Avoid mapping ports in production Compose files
- Use `expose` for internal communication only
- Map ports only for development or edge services

```yaml
services:
  web:
    image: nginx:alpine
    # ✅ Internal communication only
    expose:
      - "80"
    # ❌ Avoid in production
    # ports:
    #   - "80:80"
    
  frontend:
    image: myapp:latest
    # ✅ OK for edge services
    ports:
      - "3000:3000"
```

### Dependencies and Startup Order
- Use `depends_on` for startup order dependencies
- Implement application-level wait strategies (not just container startup)
- Consider using `healthcheck` with `depends_on` conditions

```yaml
services:
  web:
    image: myapp:latest
    depends_on:
      db:
        condition: service_healthy
        
  db:
    image: postgres:15-alpine
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5
```

### User and Security Configuration
- **CRITICAL**: Never run containers as root in production
- Create and use non-root users
- Apply security options and capabilities restrictions

```yaml
services:
  app:
    image: myapp:latest
    user: "1000:1000"
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE
```

## Networks Configuration

### Default Network Behavior
- Compose creates a default bridge network automatically
- Services can communicate using service names as hostnames
- Explicit network definitions provide better control

### Custom Networks
- Define custom networks for service isolation
- Use bridge driver for single-host deployments
- Use overlay driver for multi-host/swarm deployments

```yaml
services:
  frontend:
    image: nginx:alpine
    networks:
      - public
      - backend
      
  api:
    image: myapi:latest
    networks:
      - backend
      - database
      
  db:
    image: postgres:15-alpine
    networks:
      - database

networks:
  public:
    driver: bridge
  backend:
    driver: bridge
    internal: true
  database:
    driver: bridge
    internal: true
```

### Network Security
- Use `internal: true` for networks that don't need external access
- Implement network segmentation for multi-tier applications
- Configure custom subnets when IP control is required

### Service Discovery and Aliases
- Use network aliases for alternative hostnames
- Configure multiple aliases for service migration scenarios

```yaml
services:
  db:
    image: postgres:15-alpine
    networks:
      backend:
        aliases:
          - database
          - postgres
          - db-primary
```

## Volumes Configuration

### Volume Types and Usage
- Use named volumes for persistent data
- Use bind mounts only for development or configuration
- Use tmpfs for temporary data that shouldn't persist

```yaml
services:
  app:
    image: myapp:latest
    volumes:
      # ✅ Named volume for data persistence
      - app_data:/var/lib/app/data
      # ✅ Config bind mount (read-only)
      - ./config/app.conf:/etc/app/app.conf:ro
      # ✅ Tmpfs for temporary files
      - type: tmpfs
        target: /tmp
        tmpfs:
          size: 100M

volumes:
  app_data:
    driver: local
```

### Volume Security and Permissions
- Set appropriate file permissions for bind mounts
- Use read-only mounts for configuration files
- Avoid mounting Docker socket unless absolutely necessary

### Backup and Persistence Strategy
- Document volume backup procedures
- Use external volumes for critical data
- Consider volume driver options for cloud storage

```yaml
volumes:
  db_data:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: /opt/app/database
```

## Secrets and Configs Management

### Secrets Configuration
- Use Docker secrets for sensitive data
- Never use environment variables for passwords
- Implement proper secret rotation procedures

```yaml
services:
  app:
    image: myapp:latest
    secrets:
      - source: api_key
        target: /run/secrets/api_key
        mode: 0400
      - source: db_password
        target: /run/secrets/db_pass
        uid: "103"
        gid: "103"

secrets:
  api_key:
    external: true
  db_password:
    file: ./secrets/db_password.txt
```

### Configuration Management
- Use configs for non-sensitive configuration files
- Version control config changes
- Implement config validation

```yaml
services:
  nginx:
    image: nginx:alpine
    configs:
      - source: nginx_config
        target: /etc/nginx/nginx.conf
        mode: 0644

configs:
  nginx_config:
    file: ./config/nginx.conf
```

## Best Practices

### Development vs Production
- Maintain separate Compose files for different environments
- Use override files for environment-specific configurations
- Implement proper image tagging strategies

```yaml
# compose.yaml (base)
services:
  app:
    build: .
    environment:
      NODE_ENV: production

---
# compose.override.yaml (development)
services:
  app:
    volumes:
      - .:/app
      - /app/node_modules
    environment:
      NODE_ENV: development
    command: npm run dev
```

### Performance Optimization
- Use multi-stage Dockerfiles to reduce image size
- Implement proper caching strategies
- Use .dockerignore to exclude unnecessary files

### Logging and Monitoring
- Configure centralized logging
- Implement health check endpoints
- Use labels for container metadata

```yaml
services:
  app:
    image: myapp:latest
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
    labels:
      - "traefik.enable=true"
      - "monitoring.enabled=true"
```

### File Organization
- Use .env files for environment-specific variables
- Implement .dockerignore files
- Organize related services in logical groups

## Known Issues and Mitigations

### Networking Issues
- **Issue**: Service-to-service communication failures
- **Mitigation**: Verify services are on same network, use correct hostnames (service names)
- **Debug**: Use `docker compose exec <service> nslookup <target_service>`

### Volume Permissions
- **Issue**: Permission denied errors on volume mounts
- **Mitigation**: Match container user UID/GID with host filesystem permissions
- **Solution**: Use init containers to fix permissions or proper user configuration

### Environment Variable Interpolation
- **Issue**: Variables not expanding correctly
- **Mitigation**: Use proper syntax `${VAR:-default}` and verify .env file loading
- **Debug**: Use `docker compose config` to verify final configuration

### Dependency Management
- **Issue**: Services starting before dependencies are ready
- **Mitigation**: Implement application-level retry logic and health checks
- **Solution**: Use `depends_on` with `condition: service_healthy`

```yaml
services:
  app:
    image: myapp:latest
    depends_on:
      db:
        condition: service_healthy
    restart: on-failure
    environment:
      # Application should implement retry logic
      DB_RETRY_ATTEMPTS: 5
      DB_RETRY_DELAY: 5
```

### Memory and Resource Issues
- **Issue**: Containers killed by OOM killer
- **Mitigation**: Set appropriate memory limits and monitor usage
- **Solution**: Implement proper resource constraints and application profiling

### Build Context Issues
- **Issue**: Large build contexts causing slow builds
- **Mitigation**: Use .dockerignore and minimize build context
- **Solution**: Use multi-stage builds and separate build/runtime contexts

## Security Hardening

### Container Security
- Run containers as non-root users
- Use read-only root filesystems where possible
- Drop unnecessary capabilities and privileges

```yaml
services:
  app:
    image: myapp:latest
    user: "app:app"
    read_only: true
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL
    tmpfs:
      - /tmp:noexec,nosuid,size=100m
```

### Network Security
- Use internal networks for inter-service communication
- Implement proper firewall rules on host
- Avoid exposing unnecessary ports

### Secrets Security
- Never commit secrets to version control
- Use proper secret management solutions
- Implement secret rotation procedures
- Audit secret access and usage

### Image Security
- Use official base images from trusted sources
- Regularly update base images for security patches
- Scan images for vulnerabilities
- Use minimal base images (alpine, distroless)

## Production Deployment Considerations

### Orchestration Platform Selection
- **Docker Compose**: Suitable for single-host deployments, development, testing
- **Docker Swarm**: Better for multi-host, basic orchestration needs
- **Kubernetes**: Recommended for complex production environments
- **Note**: Compose is NOT recommended for large-scale production deployments

### Monitoring and Observability
- Implement comprehensive logging strategy
- Use health checks for service monitoring
- Configure metrics collection and alerting

### Backup and Disaster Recovery
- Implement automated backup procedures for volumes
- Document recovery procedures
- Test backup/restore processes regularly

### Scaling Considerations
- Design services to be stateless when possible
- Use external databases and storage for persistent data
- Implement proper load balancing and service discovery