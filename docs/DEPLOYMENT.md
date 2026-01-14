# Deployment Guide

This document provides instructions for deploying WordNetLoom in various environments.

## Table of Contents

- [Deployment Options](#deployment-options)
- [Docker Deployment](#docker-deployment)
- [Manual Deployment](#manual-deployment)
- [Production Deployment](#production-deployment)
- [Scaling](#scaling)
- [Monitoring](#monitoring)
- [Backup and Recovery](#backup-and-recovery)
- [Troubleshooting](#troubleshooting)

## Deployment Options

| Method | Complexity | Best For |
|--------|------------|----------|
| Docker Compose | Low | Development, small deployments |
| Manual (WildFly) | Medium | Custom configurations |
| Kubernetes | High | Large-scale production |

## Docker Deployment

### Quick Start

```bash
# Clone repository
git clone <repository-url> wordnetloom
cd wordnetloom

# Build and start
./build-server.sh
docker-compose up -d
```

### Build Process

The build process consists of three steps:

1. **Clean previous builds:**
   ```bash
   docker-compose run --rm maven clean
   ```

2. **Package application:**
   ```bash
   docker-compose run --rm maven package
   ```

3. **Build WildFly image:**
   ```bash
   docker-compose build wildfly
   ```

### Docker Compose Services

```yaml
services:
  db:        # MySQL 8.0 database
  wildfly:   # WildFly application server
  maven:     # Build container
```

### Accessing Services

| Service | URL | Port |
|---------|-----|------|
| API | http://localhost:8080/wordnetloom-server/resources/ | 8080 |
| Database | localhost | 33366 |
| WildFly Management | http://localhost:9990 | 9990 |

### Managing Containers

```bash
# Start services
docker-compose up -d

# Stop services
docker-compose down

# View logs
docker-compose logs -f wildfly

# Restart a service
docker-compose restart wildfly

# Rebuild after changes
docker-compose build --no-cache wildfly
docker-compose up -d
```

### Updating the Application

```bash
# Pull latest code
git pull

# Rebuild and redeploy
./build-server.sh
docker-compose down
docker-compose up -d
```

## Manual Deployment

### Prerequisites

- WildFly 20.0.1 or later
- MySQL 8.0
- Java 11

### Step 1: Prepare WildFly

```bash
# Download WildFly
wget https://download.jboss.org/wildfly/20.0.1.Final/wildfly-20.0.1.Final.tar.gz
tar -xzf wildfly-20.0.1.Final.tar.gz
export WILDFLY_HOME=$(pwd)/wildfly-20.0.1.Final
```

### Step 2: Configure MySQL Module

```bash
# Create module directory
mkdir -p $WILDFLY_HOME/modules/system/layers/base/com/mysql/main

# Download MySQL connector
wget https://dev.mysql.com/get/Downloads/Connector-J/mysql-connector-java-8.0.23.tar.gz
tar -xzf mysql-connector-java-8.0.23.tar.gz
cp mysql-connector-java-8.0.23/mysql-connector-java-8.0.23.jar \
   $WILDFLY_HOME/modules/system/layers/base/com/mysql/main/
```

Create `module.xml`:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<module xmlns="urn:jboss:module:1.3" name="com.mysql">
    <resources>
        <resource-root path="mysql-connector-java-8.0.23.jar"/>
    </resources>
    <dependencies>
        <module name="javax.api"/>
        <module name="javax.transaction.api"/>
    </dependencies>
</module>
```

### Step 3: Configure Datasource

Edit `$WILDFLY_HOME/standalone/configuration/standalone.xml`:

```xml
<subsystem xmlns="urn:jboss:domain:datasources:5.0">
    <datasources>
        <datasource jndi-name="java:jboss/datasources/wordnet"
                    pool-name="WordNetDS">
            <connection-url>jdbc:mysql://localhost:3306/wordnet?useSSL=false&amp;serverTimezone=UTC</connection-url>
            <driver>mysql</driver>
            <security>
                <user-name>wordnet</user-name>
                <password>your_password</password>
            </security>
        </datasource>
        <drivers>
            <driver name="mysql" module="com.mysql">
                <driver-class>com.mysql.cj.jdbc.Driver</driver-class>
            </driver>
        </drivers>
    </datasources>
</subsystem>
```

### Step 4: Build Application

```bash
cd wordnetloom
mvn clean package
```

### Step 5: Deploy

```bash
cp wordnetloom-server/target/wordnetloom-server.war \
   $WILDFLY_HOME/standalone/deployments/
```

### Step 6: Start WildFly

```bash
$WILDFLY_HOME/bin/standalone.sh
```

## Production Deployment

### Security Checklist

- [ ] Change default database passwords
- [ ] Configure SSL/TLS certificates
- [ ] Set strong JWT secret
- [ ] Enable firewall rules
- [ ] Configure backup schedule
- [ ] Set up monitoring
- [ ] Review log retention policy

### SSL/TLS Configuration

#### Generate Certificate

```bash
# Self-signed (development)
keytool -genkeypair -alias wordnetloom \
        -keyalg RSA -keysize 2048 \
        -validity 365 \
        -keystore wordnetloom.keystore \
        -storepass changeit

# For production, use certificates from a CA
```

#### Configure WildFly HTTPS

```xml
<security-realm name="ApplicationRealm">
    <server-identities>
        <ssl>
            <keystore path="wordnetloom.keystore"
                      relative-to="jboss.server.config.dir"
                      keystore-password="changeit"
                      alias="wordnetloom"/>
        </ssl>
    </server-identities>
</security-realm>

<subsystem xmlns="urn:jboss:domain:undertow:11.0">
    <server name="default-server">
        <https-listener name="https"
                        socket-binding="https"
                        security-realm="ApplicationRealm"/>
    </server>
</subsystem>
```

### Reverse Proxy (Nginx)

```nginx
server {
    listen 80;
    server_name wordnetloom.example.com;
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name wordnetloom.example.com;

    ssl_certificate /etc/letsencrypt/live/wordnetloom.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/wordnetloom.example.com/privkey.pem;

    location /wordnetloom-server/ {
        proxy_pass http://localhost:8080/wordnetloom-server/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### Database Production Settings

```sql
-- Create production user with limited privileges
CREATE USER 'wordnet_prod'@'%' IDENTIFIED BY 'strong_password_here';
GRANT SELECT, INSERT, UPDATE, DELETE ON wordnet.* TO 'wordnet_prod'@'%';
FLUSH PRIVILEGES;
```

### JVM Production Settings

```bash
# In standalone.conf
JAVA_OPTS="-server"
JAVA_OPTS="$JAVA_OPTS -Xms2g -Xmx4g"
JAVA_OPTS="$JAVA_OPTS -XX:+UseG1GC"
JAVA_OPTS="$JAVA_OPTS -XX:MaxGCPauseMillis=200"
JAVA_OPTS="$JAVA_OPTS -XX:+HeapDumpOnOutOfMemoryError"
JAVA_OPTS="$JAVA_OPTS -XX:HeapDumpPath=/var/log/wildfly/"
JAVA_OPTS="$JAVA_OPTS -Djboss.bind.address=0.0.0.0"
```

### Systemd Service

Create `/etc/systemd/system/wildfly.service`:

```ini
[Unit]
Description=WildFly Application Server
After=network.target mysql.service

[Service]
Type=simple
User=wildfly
Group=wildfly
Environment="JAVA_HOME=/usr/lib/jvm/java-11-openjdk"
ExecStart=/opt/wildfly/bin/standalone.sh
ExecStop=/opt/wildfly/bin/jboss-cli.sh --connect command=:shutdown
Restart=on-failure
RestartSec=10

[Install]
WantedBy=multi-user.target
```

Enable and start:
```bash
sudo systemctl enable wildfly
sudo systemctl start wildfly
```

## Scaling

### Horizontal Scaling

For high availability, deploy multiple WildFly instances:

```
                     ┌─────────────────┐
                     │   Load Balancer │
                     │     (Nginx)     │
                     └────────┬────────┘
                              │
           ┌──────────────────┼──────────────────┐
           │                  │                  │
           ▼                  ▼                  ▼
    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
    │  WildFly 1  │    │  WildFly 2  │    │  WildFly 3  │
    └──────┬──────┘    └──────┬──────┘    └──────┬──────┘
           │                  │                  │
           └──────────────────┼──────────────────┘
                              │
                     ┌────────┴────────┐
                     │  MySQL (Master) │
                     └─────────────────┘
```

### Database Replication

For read scaling, configure MySQL replication:

```
MySQL Master (Write) ─────► MySQL Replica 1 (Read)
                     └────► MySQL Replica 2 (Read)
```

### Caching

Consider adding Redis for caching:

```bash
# Docker
docker run -d -p 6379:6379 redis:alpine
```

## Monitoring

### WildFly Metrics

Access metrics at: `http://localhost:9990/management`

### Health Checks

```bash
# Check API health
curl -f http://localhost:8080/wordnetloom-server/resources/

# Check database connection
mysql -u wordnet -p -e "SELECT 1"
```

### Log Monitoring

```bash
# WildFly logs
tail -f $WILDFLY_HOME/standalone/log/server.log

# Docker logs
docker-compose logs -f wildfly
```

### Prometheus Metrics (Optional)

Add WildFly Prometheus exporter for metrics:

```xml
<extension module="org.wildfly.extension.microprofile.metrics-smallrye"/>
<subsystem xmlns="urn:wildfly:microprofile-metrics-smallrye:2.0">
    <expose-all-subsystems value="true"/>
</subsystem>
```

## Backup and Recovery

### Database Backup

```bash
# Full backup
mysqldump -u root -p --single-transaction --routines --triggers \
          wordnet > wordnet_backup_$(date +%Y%m%d).sql

# Compressed backup
mysqldump -u root -p --single-transaction wordnet | gzip > wordnet_$(date +%Y%m%d).sql.gz
```

### Automated Backup Script

```bash
#!/bin/bash
# backup.sh

BACKUP_DIR=/var/backups/wordnetloom
DATE=$(date +%Y%m%d_%H%M%S)
RETENTION_DAYS=30

# Create backup
mysqldump -u backup_user -p'password' --single-transaction wordnet \
    | gzip > $BACKUP_DIR/wordnet_$DATE.sql.gz

# Remove old backups
find $BACKUP_DIR -name "*.sql.gz" -mtime +$RETENTION_DAYS -delete
```

Add to cron:
```bash
0 2 * * * /opt/scripts/backup.sh
```

### Recovery

```bash
# Restore from backup
gunzip < wordnet_backup.sql.gz | mysql -u root -p wordnet

# Or for uncompressed
mysql -u root -p wordnet < wordnet_backup.sql
```

## Troubleshooting

### Common Issues

#### Deployment Failed

Check deployment status:
```bash
ls -la $WILDFLY_HOME/standalone/deployments/
# Look for .failed files
cat $WILDFLY_HOME/standalone/deployments/wordnetloom-server.war.failed
```

#### Database Connection Issues

```bash
# Test connection
mysql -h localhost -u wordnet -p -e "SELECT 1"

# Check WildFly datasource
$WILDFLY_HOME/bin/jboss-cli.sh --connect
/subsystem=datasources/data-source=WordNetDS:test-connection-in-pool
```

#### Out of Memory

```bash
# Check memory usage
jcmd $(pgrep -f wildfly) VM.native_memory summary

# Increase heap
# Edit standalone.conf
JAVA_OPTS="$JAVA_OPTS -Xmx4g"
```

#### Port Conflicts

```bash
# Check ports
netstat -tlnp | grep -E '8080|3306|9990'

# Kill process using port
fuser -k 8080/tcp
```

### Log Analysis

```bash
# Search for errors
grep -i error $WILDFLY_HOME/standalone/log/server.log

# Recent deployment issues
grep -i deploy $WILDFLY_HOME/standalone/log/server.log | tail -50
```

### Recovery Procedures

#### Rollback Deployment

```bash
# Remove failed deployment
rm $WILDFLY_HOME/standalone/deployments/wordnetloom-server.war*

# Deploy previous version
cp /backups/wordnetloom-server-previous.war \
   $WILDFLY_HOME/standalone/deployments/wordnetloom-server.war
```

#### Database Recovery

```bash
# Stop application
$WILDFLY_HOME/bin/jboss-cli.sh --connect command=:shutdown

# Restore database
mysql -u root -p wordnet < wordnet_backup.sql

# Restart application
$WILDFLY_HOME/bin/standalone.sh
```
