# Appwrite Installation Guide

Appwrite is an open-source backend-as-a-service platform that provides developers with all the core APIs required to build modern applications. It includes authentication, databases, storage, functions, and real-time capabilities.

## Table of Contents
1. [Prerequisites](#prerequisites)
2. [Supported Operating Systems](#supported-operating-systems)
3. [Installation](#installation)
4. [Configuration](#configuration)
5. [Service Management](#service-management)
6. [Troubleshooting](#troubleshooting)
7. [Security Considerations](#security-considerations)
8. [Performance Tuning](#performance-tuning)
9. [Backup and Restore](#backup-and-restore)
10. [System Requirements](#system-requirements)
11. [Support](#support)
12. [Contributing](#contributing)
13. [License](#license)
14. [Acknowledgments](#acknowledgments)
15. [Version History](#version-history)
16. [Appendices](#appendices)

## 1. Prerequisites

- Docker and Docker Compose installed
- Minimum 4GB RAM (8GB recommended for production)
- 10GB free disk space minimum
- Domain name with DNS configured (for production)
- SSL certificate (Let's Encrypt recommended)


## 2. Supported Operating Systems

This guide supports installation on:
- RHEL 8/9 and derivatives (CentOS Stream, Rocky Linux, AlmaLinux)
- Debian 11/12
- Ubuntu 20.04/22.04/24.04 LTS
- Arch Linux (rolling release)
- Alpine Linux 3.18+
- openSUSE Leap 15.5+ / Tumbleweed
- SUSE Linux Enterprise Server (SLES) 15+
- macOS 12+ (Monterey and later) 
- FreeBSD 13+
- Windows 10/11/Server 2019+ (where applicable)

## 3. Installation

### Docker Compose Installation (Recommended)

1. **Download Appwrite**:
```bash
# Create installation directory
mkdir -p /opt/appwrite
cd /opt/appwrite

# Download docker-compose.yml
curl -o docker-compose.yml https://appwrite.io/install/compose

# Download .env file
curl -o .env https://appwrite.io/install/env
```

2. **Configure Environment**:
```bash
# Edit environment variables
nano .env

# Key variables to configure:
# _APP_ENV=production
# _APP_DOMAIN=your-domain.com
# _APP_DOMAIN_TARGET=your-domain.com
# _APP_REDIS_PASS=your-redis-password
# _APP_DB_PASS=your-database-password
# _APP_OPENSSL_KEY_V1=your-32-char-key
```

3. **Generate OpenSSL Key**:
```bash
# Generate a secure key
openssl rand -hex 32
```

4. **Start Appwrite**:
```bash
# Start services
docker-compose up -d

# Check status
docker-compose ps

# View logs
docker-compose logs -f
```

### Manual Docker Installation

```bash
# Create network
docker network create appwrite

# Run MariaDB
docker run -d \
  --name appwrite-mariadb \
  --network appwrite \
  -e MYSQL_ROOT_PASSWORD=rootpassword \
  -e MYSQL_DATABASE=appwrite \
  -e MYSQL_USER=appwrite \
  -e MYSQL_PASSWORD=password \
  -v appwrite-mariadb:/var/lib/mysql \
  mariadb:10

# Run Redis
docker run -d \
  --name appwrite-redis \
  --network appwrite \
  -v appwrite-redis:/data \
  redis:alpine

# Run Appwrite
docker run -d \
  --name appwrite \
  --network appwrite \
  -p 80:80 \
  -p 443:443 \
  -e _APP_ENV=production \
  -e _APP_OPENSSL_KEY_V1=your-32-char-key \
  -e _APP_DOMAIN=localhost \
  -e _APP_DB_HOST=appwrite-mariadb \
  -e _APP_DB_PORT=3306 \
  -e _APP_DB_SCHEMA=appwrite \
  -e _APP_DB_USER=appwrite \
  -e _APP_DB_PASS=password \
  -e _APP_REDIS_HOST=appwrite-redis \
  -e _APP_REDIS_PORT=6379 \
  -v appwrite-uploads:/storage/uploads \
  -v appwrite-cache:/storage/cache \
  -v appwrite-config:/storage/config \
  -v appwrite-certificates:/storage/certificates \
  -v appwrite-functions:/storage/functions \
  appwrite/appwrite
```

## 4. Configuration

### SSL/TLS Setup

1. **Let's Encrypt (Automatic)**:
```bash
# Appwrite handles SSL automatically for configured domains
# Ensure ports 80 and 443 are accessible
# Domain must point to server IP
```

2. **Custom SSL Certificate**:
```bash
# Copy certificates
cp /path/to/cert.pem /opt/appwrite/certificates/main.crt
cp /path/to/key.pem /opt/appwrite/certificates/main.key

# Restart Appwrite
cd /opt/appwrite
docker-compose restart
```

### Email Configuration

Edit `.env` file:
```bash
_APP_SMTP_HOST=smtp.gmail.com
_APP_SMTP_PORT=587
_APP_SMTP_SECURE=tls
_APP_SMTP_USERNAME=your-email@gmail.com
_APP_SMTP_PASSWORD=your-app-password
```

### Storage Configuration

1. **Local Storage** (Default):
```bash
# Storage is handled automatically
# Files stored in Docker volumes
```

2. **S3 Compatible Storage**:
```bash
_APP_STORAGE_DEVICE=s3
_APP_STORAGE_S3_ACCESS_KEY=your-access-key
_APP_STORAGE_S3_SECRET=your-secret-key
_APP_STORAGE_S3_REGION=us-east-1
_APP_STORAGE_S3_BUCKET=appwrite-storage
```

### Functions Runtime

Enable additional runtimes in `.env`:
```bash
_APP_FUNCTIONS_RUNTIMES=node-18.0,python-3.10,php-8.1,ruby-3.1
```

## Security Configuration

### Firewall Rules

```bash
# Allow HTTP/HTTPS
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp

# Allow Appwrite console (if different port)
sudo ufw allow 8080/tcp
```

### Security Headers

Appwrite includes security headers by default:
- Content Security Policy
- X-Frame-Options
- X-Content-Type-Options
- Strict-Transport-Security

### API Keys and Secrets

1. **Generate secure keys**:
```bash
# Generate encryption key
openssl rand -hex 32

# Generate session secret
openssl rand -hex 32
```

2. **Rotate keys regularly**:
```bash
# Update in .env
_APP_OPENSSL_KEY_V1=new-key-here

# Restart services
docker-compose restart
```

## Database Management

### Backup

```bash
# Backup database
docker exec appwrite-mariadb mysqldump -u root -p appwrite > backup.sql

# Backup volumes
docker run --rm -v appwrite-mariadb:/data -v $(pwd):/backup ubuntu tar czf /backup/mariadb-backup.tar.gz /data
```

### Restore

```bash
# Restore database
docker exec -i appwrite-mariadb mysql -u root -p appwrite < backup.sql

# Restore volumes
docker run --rm -v appwrite-mariadb:/data -v $(pwd):/backup ubuntu tar xzf /backup/mariadb-backup.tar.gz -C /
```

## Performance Optimization

### Redis Configuration

```bash
# Edit Redis configuration
docker exec -it appwrite-redis redis-cli

# Set max memory
CONFIG SET maxmemory 2gb
CONFIG SET maxmemory-policy allkeys-lru
```

### Database Optimization

```bash
# Optimize tables
docker exec appwrite-mariadb mysqlcheck -u root -p --optimize appwrite

# Configure MariaDB
docker exec -it appwrite-mariadb mysql -u root -p
SET GLOBAL innodb_buffer_pool_size = 1G;
SET GLOBAL innodb_log_file_size = 256M;
```

### Scaling

1. **Horizontal Scaling**:
```bash
# Use Docker Swarm or Kubernetes
# Configure load balancer
# Share storage between instances
```

2. **Vertical Scaling**:
```bash
# Increase container resources
docker-compose down
# Edit docker-compose.yml to add resource limits
docker-compose up -d
```

## Monitoring

### Health Checks

```bash
# Check service health
curl http://localhost/v1/health

# Check specific services
curl http://localhost/v1/health/db
curl http://localhost/v1/health/cache
curl http://localhost/v1/health/time
```

### Logging

```bash
# View all logs
docker-compose logs -f

# View specific service logs
docker-compose logs -f appwrite
docker-compose logs -f appwrite-worker-database

# Export logs
docker-compose logs > appwrite-logs.txt
```

### Metrics

Enable Prometheus metrics:
```bash
_APP_PROMETHEUS_ENABLE=enabled
```

## 6. Troubleshooting

### Common Issues

1. **Port conflicts**:
```bash
# Check port usage
sudo netstat -tlnp | grep -E ':(80|443|8080)'

# Change ports in docker-compose.yml
```

2. **Permission issues**:
```bash
# Fix volume permissions
sudo chown -R 999:999 /var/lib/docker/volumes/appwrite_*
```

3. **Memory issues**:
```bash
# Check memory usage
docker stats

# Increase memory limits in docker-compose.yml
```

### Debug Mode

Enable debug mode in `.env`:
```bash
_APP_ENV=development
_APP_DEBUG=true
```

## Maintenance

### Updates

```bash
# Backup first
./backup.sh

# Pull latest images
docker-compose pull

# Restart services
docker-compose down
docker-compose up -d
```

### Cleanup

```bash
# Remove unused images
docker image prune -a

# Clean build cache
docker builder prune

# Remove old logs
docker-compose logs --tail=0 > /dev/null
```

## SDK Integration

### JavaScript/Node.js

```javascript
const sdk = require('node-appwrite');

const client = new sdk.Client()
    .setEndpoint('https://your-domain.com/v1')
    .setProject('your-project-id')
    .setKey('your-api-key');

const database = new sdk.Databases(client);
const storage = new sdk.Storage(client);
```

### Python

```python
from appwrite.client import Client
from appwrite.services.databases import Databases

client = Client()
client.set_endpoint('https://your-domain.com/v1')
client.set_project('your-project-id')
client.set_key('your-api-key')

databases = Databases(client)
```

### Flutter

```dart
import 'package:appwrite/appwrite.dart';

final client = Client()
    .setEndpoint('https://your-domain.com/v1')
    .setProject('your-project-id');

final account = Account(client);
final databases = Databases(client);
```

## Additional Resources

- [Official Documentation](https://appwrite.io/docs)
- [API Reference](https://appwrite.io/docs/client/account)
- [GitHub Repository](https://github.com/appwrite/appwrite)
- [Community Discord](https://appwrite.io/discord)
- [Docker Hub](https://hub.docker.com/r/appwrite/appwrite)
- [SDK Libraries](https://appwrite.io/docs/sdks)

---

**Note:** This guide is part of the [HowToMgr](https://howtomgr.github.io) collection. Always refer to official documentation for the most up-to-date information.