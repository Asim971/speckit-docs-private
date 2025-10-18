# JibonFlow Healthcare Platform - Troubleshooting Guide

## Overview

This comprehensive troubleshooting guide covers common issues, error resolutions, diagnostic procedures, and maintenance tasks for the JibonFlow Healthcare Platform.

## Quick Diagnostics

### Health Check Commands

```bash
# Check all services health
curl http://localhost:3000/health

# Check individual service health
curl http://localhost:3001/health  # User Service
curl http://localhost:3002/health  # Auth Service
curl http://localhost:3003/health  # Healthcare Service

# Check database connectivity
docker exec jibonflow-postgres pg_isready -U jibonflow_user -d jibonflow

# Check Redis connectivity
docker exec jibonflow-redis redis-cli ping
```

### System Status Check

```bash
# Check Docker containers status
docker-compose ps

# Check container logs
docker-compose logs api-gateway
docker-compose logs user-service
docker-compose logs auth-service
docker-compose logs healthcare-service

# Check resource usage
docker stats --no-stream
```

## Common Issues and Solutions

### 1. Service Startup Issues

#### Problem: Services fail to start

**Symptoms:**
- Containers exit immediately
- Health checks fail
- Port binding errors

**Diagnostic Steps:**

```bash
# Check container logs
docker-compose logs <service-name>

# Check port availability
netstat -tulpn | grep :3000
netstat -tulpn | grep :3001
netstat -tulpn | grep :3002
netstat -tulpn | grep :3003

# Check Docker daemon status
systemctl status docker
```

**Solutions:**

1. **Port conflicts:**
   ```bash
   # Kill processes using ports
   sudo lsof -ti:3000 | xargs kill -9
   sudo lsof -ti:3001 | xargs kill -9
   
   # Or change ports in docker-compose.yml
   ```

2. **Missing environment variables:**
   ```bash
   # Check .env file exists
   ls -la .env
   
   # Verify environment variables
   docker-compose config
   ```

3. **Database connection issues:**
   ```bash
   # Restart database first
   docker-compose up -d postgres redis
   sleep 30
   docker-compose up -d
   ```

### 2. Database Connection Problems

#### Problem: Cannot connect to PostgreSQL

**Symptoms:**
- Connection timeout errors
- Authentication failures
- Services unable to start

**Diagnostic Steps:**

```bash
# Check PostgreSQL container
docker-compose logs postgres

# Test database connection
docker exec -it jibonflow-postgres psql -U jibonflow_user -d jibonflow

# Check database configuration
docker exec jibonflow-postgres cat /var/lib/postgresql/data/postgresql.conf
```

**Solutions:**

1. **Database not ready:**
   ```bash
   # Wait for database initialization
   docker-compose up -d postgres
   sleep 60
   docker-compose up -d
   ```

2. **Wrong credentials:**
   ```bash
   # Check environment variables
   grep -E "POSTGRES_|DATABASE_" .env
   
   # Reset database with correct credentials
   docker-compose down -v
   docker-compose up -d postgres
   ```

3. **Connection pool exhaustion:**
   ```bash
   # Check active connections
   docker exec jibonflow-postgres psql -U jibonflow_user -d jibonflow -c "SELECT count(*) FROM pg_stat_activity;"
   
   # Restart services to reset connection pools
   docker-compose restart
   ```

### 3. Redis Connection Issues

#### Problem: Redis connection failures

**Symptoms:**
- Session management errors
- Cache miss errors
- Authentication token issues

**Diagnostic Steps:**

```bash
# Check Redis container
docker-compose logs redis

# Test Redis connection
docker exec -it jibonflow-redis redis-cli ping

# Check Redis memory usage
docker exec jibonflow-redis redis-cli INFO memory
```

**Solutions:**

1. **Redis out of memory:**
   ```bash
   # Check memory usage
   docker exec jibonflow-redis redis-cli INFO memory
   
   # Clear Redis cache
   docker exec jibonflow-redis redis-cli FLUSHALL
   
   # Increase memory limit in docker-compose.yml
   ```

2. **Redis configuration issues:**
   ```bash
   # Check Redis configuration
   docker exec jibonflow-redis redis-cli CONFIG GET "*"
   
   # Restart Redis with proper configuration
   docker-compose restart redis
   ```

### 4. Authentication Problems

#### Problem: JWT token issues

**Symptoms:**
- 401 Unauthorized errors
- Token expired messages
- Invalid token errors

**Diagnostic Steps:**

```bash
# Check auth service logs
docker-compose logs auth-service

# Test token generation
curl -X POST http://localhost:3002/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"test@example.com","password":"password"}'

# Verify JWT secret configuration
docker exec jibonflow-auth-service env | grep JWT
```

**Solutions:**

1. **Token expiration:**
   ```bash
   # Use refresh token to get new access token
   curl -X POST http://localhost:3002/auth/refresh \
     -H "Content-Type: application/json" \
     -d '{"refreshToken":"your-refresh-token"}'
   ```

2. **JWT secret mismatch:**
   ```bash
   # Ensure all services use same JWT secret
   grep JWT_SECRET .env
   docker-compose restart
   ```

3. **Clock synchronization:**
   ```bash
   # Check system time
   timedatectl status
   
   # Synchronize time if needed
   sudo ntpdate -s time.nist.gov
   ```

### 5. API Gateway Issues

#### Problem: Request routing failures

**Symptoms:**
- 404 Not Found errors
- Service unavailable errors
- Slow response times

**Diagnostic Steps:**

```bash
# Check API Gateway logs
docker-compose logs api-gateway

# Test direct service access
curl http://localhost:3001/health
curl http://localhost:3002/health
curl http://localhost:3003/health

# Check service discovery
docker network inspect jibonflow_jibonflow-network
```

**Solutions:**

1. **Service discovery issues:**
   ```bash
   # Restart API Gateway
   docker-compose restart api-gateway
   
   # Check network connectivity
   docker exec jibonflow-api-gateway ping user-service
   docker exec jibonflow-api-gateway ping auth-service
   ```

2. **Load balancing problems:**
   ```bash
   # Check service health from gateway
   docker exec jibonflow-api-gateway curl http://user-service:3001/health
   
   # Restart unhealthy services
   docker-compose restart user-service
   ```

### 6. Performance Issues

#### Problem: Slow API responses

**Symptoms:**
- High response times
- Timeout errors
- High CPU/memory usage

**Diagnostic Steps:**

```bash
# Check resource usage
docker stats --no-stream

# Monitor response times
curl -w "@curl-format.txt" -o /dev/null -s http://localhost:3000/health

# Check database performance
docker exec jibonflow-postgres psql -U jibonflow_user -d jibonflow -c "SELECT * FROM pg_stat_activity;"
```

**Solutions:**

1. **Database optimization:**
   ```bash
   # Run database performance script
   ./scripts/optimize-performance.sh
   
   # Check slow queries
   docker exec jibonflow-postgres psql -U jibonflow_user -d jibonflow -c "SELECT query, mean_time, calls FROM pg_stat_statements ORDER BY mean_time DESC LIMIT 10;"
   ```

2. **Memory optimization:**
   ```bash
   # Increase container memory limits
   # Edit docker-compose.yml and add:
   # deploy:
   #   resources:
   #     limits:
   #       memory: 512M
   
   docker-compose up -d --force-recreate
   ```

3. **Connection pooling:**
   ```bash
   # Check connection pool settings
   grep -E "POOL_|CONNECTION_" .env
   
   # Optimize connection pool sizes
   # Edit .env and restart services
   ```

## Environment-Specific Issues

### Development Environment

#### Problem: Hot reload not working

**Solutions:**
```bash
# Restart development containers
docker-compose -f docker-compose.dev.yml down
docker-compose -f docker-compose.dev.yml up -d

# Check volume mounts
docker-compose -f docker-compose.dev.yml config
```

#### Problem: Debug port conflicts

**Solutions:**
```bash
# Check debug ports
netstat -tulpn | grep :9229

# Update debug ports in docker-compose.dev.yml
```

### Staging Environment

#### Problem: SSL certificate issues

**Solutions:**
```bash
# Check certificate validity
openssl x509 -in /path/to/cert.pem -text -noout

# Renew certificates
certbot renew --nginx

# Restart services
docker-compose restart
```

### Production Environment

#### Problem: High load issues

**Solutions:**
```bash
# Scale services horizontally
docker-compose up -d --scale user-service=3
docker-compose up -d --scale healthcare-service=2

# Enable load balancing
# Update docker-compose.yml with load balancer configuration
```

## Monitoring and Alerting

### Prometheus Queries

```bash
# High error rate
rate(http_requests_total{status=~"5.."}[5m])

# High response time
histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))

# Database connection pool exhaustion
pg_stat_database_numbackends / pg_settings_max_connections * 100

# Redis memory usage
redis_memory_used_bytes / redis_memory_max_bytes * 100
```

### Log Analysis

```bash
# Search for errors in logs
docker-compose logs | grep -i error

# Filter logs by service
docker-compose logs auth-service | grep -E "(error|warn)"

# Real-time log monitoring
docker-compose logs -f --tail=100

# Export logs for analysis
docker-compose logs > jibonflow-logs-$(date +%Y%m%d).txt
```

## Maintenance Tasks

### Regular Maintenance

#### Daily Tasks
```bash
# Check service health
./scripts/health-check.sh

# Monitor resource usage
docker stats --no-stream >> daily-stats.log

# Backup database
docker exec jibonflow-postgres pg_dump -U jibonflow_user jibonflow > backup-$(date +%Y%m%d).sql
```

#### Weekly Tasks
```bash
# Update Docker images
docker-compose pull
docker-compose up -d

# Clean up unused Docker resources
docker system prune -f

# Rotate logs
docker-compose logs --since 7d > logs/weekly-$(date +%Y%m%d).log
```

#### Monthly Tasks
```bash
# Update system packages
sudo apt update && sudo apt upgrade -y

# Review and update security configurations
./scripts/security-audit.sh

# Performance optimization
./scripts/optimize-performance.sh
```

### Database Maintenance

```bash
# Database vacuum and analyze
docker exec jibonflow-postgres psql -U jibonflow_user -d jibonflow -c "VACUUM ANALYZE;"

# Reindex database
docker exec jibonflow-postgres psql -U jibonflow_user -d jibonflow -c "REINDEX DATABASE jibonflow;"

# Check database size
docker exec jibonflow-postgres psql -U jibonflow_user -d jibonflow -c "SELECT pg_size_pretty(pg_database_size('jibonflow'));"
```

## Recovery Procedures

### Service Recovery

```bash
# Restart individual service
docker-compose restart <service-name>

# Recreate service with latest image
docker-compose up -d --force-recreate <service-name>

# Complete system restart
docker-compose down
docker-compose up -d
```

### Database Recovery

```bash
# Restore from backup
docker exec -i jibonflow-postgres psql -U jibonflow_user -d jibonflow < backup-file.sql

# Point-in-time recovery (if using WAL archiving)
docker exec jibonflow-postgres pg_ctl stop -D /var/lib/postgresql/data
# Restore WAL files and restart
```

### Data Recovery

```bash
# Check data integrity
docker exec jibonflow-postgres psql -U jibonflow_user -d jibonflow -c "SELECT * FROM information_schema.tables;"

# Export specific table data
docker exec jibonflow-postgres pg_dump -U jibonflow_user -d jibonflow -t users > users-backup.sql
```

## Security Troubleshooting

### SSL/TLS Issues

```bash
# Test SSL connection
openssl s_client -connect api.jibonflow.com:443

# Check certificate chain
curl -vI https://api.jibonflow.com

# Verify certificate expiration
echo | openssl s_client -servername api.jibonflow.com -connect api.jibonflow.com:443 2>/dev/null | openssl x509 -noout -dates
```

### Authentication Security

```bash
# Check failed login attempts
docker-compose logs auth-service | grep "failed"

# Monitor suspicious activity
docker-compose logs | grep -E "(401|403|429)"

# Review security headers
curl -I http://localhost:3000/
```

## Contact Support

### Emergency Contacts
- **Production Issues**: production-support@jibonflow.com
- **Security Incidents**: security@jibonflow.com
- **Database Issues**: dba@jibonflow.com

### Support Information to Provide
1. Error messages and stack traces
2. Service logs (last 100 lines)
3. System resource usage
4. Environment configuration (sanitized)
5. Steps to reproduce the issue
6. Timeline of when issue started

### Support Scripts

```bash
# Generate support bundle
./scripts/generate-support-bundle.sh

# Collect system information
./scripts/collect-system-info.sh

# Run diagnostics
./scripts/run-diagnostics.sh
```

## Appendix

### Useful Commands Reference

```bash
# Docker Compose Commands
docker-compose up -d                    # Start services
docker-compose down                     # Stop services
docker-compose logs -f <service>        # Follow logs
docker-compose exec <service> bash      # Shell access
docker-compose ps                       # List containers
docker-compose restart <service>        # Restart service

# Docker Commands
docker stats                           # Resource usage
docker system prune                    # Clean up
docker network ls                      # List networks
docker volume ls                       # List volumes

# Database Commands
pg_isready -h localhost -p 5432        # Check PostgreSQL
redis-cli ping                         # Check Redis
psql -h localhost -U user -d db        # Connect to PostgreSQL
```

### Configuration Files Locations

```
/config/
├── orchestrator.config.json          # Main configuration
├── performance.config.yaml           # Performance settings
├── jibonflow-healthcare.config.json  # Healthcare-specific config
└── schemas/                          # Database schemas

/logs/                                # Application logs
/data/                               # Persistent data
├── postgres/                        # PostgreSQL data
└── redis/                          # Redis data
```

This troubleshooting guide should be updated regularly as new issues are discovered and resolved. Keep this document current with the latest platform changes and operational procedures.