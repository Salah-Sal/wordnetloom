# Configuration Guide

This document describes all configuration options for WordNetLoom.

## Table of Contents

- [Overview](#overview)
- [Client Configuration](#client-configuration)
- [Server Configuration](#server-configuration)
- [Database Configuration](#database-configuration)
- [WildFly Configuration](#wildfly-configuration)
- [Docker Configuration](#docker-configuration)
- [Environment Variables](#environment-variables)

## Overview

WordNetLoom uses several configuration files:

| File | Location | Purpose |
|------|----------|---------|
| `application.properties` | Client | Client settings |
| `persistence.xml` | Server | JPA/Database settings |
| `standalone.xml` | WildFly | Application server |
| `docker-compose.yml` | Root | Docker deployment |

## Client Configuration

### application.properties

Location: `wordnetloom-client/src/main/resources/application.properties`

```properties
# =============================================================================
# SERVER CONNECTION
# =============================================================================

# Server host URL (required)
# The base URL of the WordNetLoom server
server.host=http://localhost:8080

# =============================================================================
# LEXICON ICONS
# =============================================================================

# Map lexicon IDs to icon files
# Icons should be placed in src/main/resources/icons/
# Format: lexicon.{id}={filename}

lexicon.1=polish.png
lexicon.2=english.png
lexicon.3=english.png
```

### Configuration Properties Reference

| Property | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `server.host` | String | Yes | - | Base URL of the server |
| `lexicon.{n}` | String | No | - | Icon filename for lexicon ID n |

### Icon Files

Place icon files in `wordnetloom-client/src/main/resources/icons/`:

- Supported formats: PNG, JPG, GIF
- Recommended size: 16x16 or 24x24 pixels
- Available icons: `polish.png`, `english.png`, `danish.png`

## Server Configuration

### persistence.xml

Location: `wordnetloom-server/src/main/resources/META-INF/persistence.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<persistence xmlns="http://xmlns.jcp.org/xml/ns/persistence"
             xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
             xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/persistence
                                 http://xmlns.jcp.org/xml/ns/persistence/persistence_2_2.xsd"
             version="2.2">

    <persistence-unit name="wordnet" transaction-type="JTA">
        <!-- JNDI datasource name (must match WildFly config) -->
        <jta-data-source>java:jboss/datasources/wordnet</jta-data-source>

        <properties>
            <!-- Hibernate dialect for MySQL 8 -->
            <property name="hibernate.dialect"
                      value="org.hibernate.dialect.MySQL8Dialect"/>

            <!-- SQL logging (set to true for debugging) -->
            <property name="hibernate.show_sql" value="false"/>
            <property name="hibernate.format_sql" value="true"/>

            <!-- Schema validation mode -->
            <!-- Options: validate, update, create, create-drop, none -->
            <property name="hibernate.hbm2ddl.auto" value="validate"/>

            <!-- Statistics for performance monitoring -->
            <property name="hibernate.generate_statistics" value="false"/>

            <!-- Connection pool settings -->
            <property name="hibernate.connection.pool_size" value="10"/>

            <!-- Batch settings for bulk operations -->
            <property name="hibernate.jdbc.batch_size" value="25"/>
            <property name="hibernate.order_inserts" value="true"/>
            <property name="hibernate.order_updates" value="true"/>
        </properties>
    </persistence-unit>
</persistence>
```

### Persistence Properties Reference

| Property | Values | Description |
|----------|--------|-------------|
| `hibernate.dialect` | `MySQL8Dialect`, `MySQL5Dialect` | Database dialect |
| `hibernate.show_sql` | `true`, `false` | Log SQL statements |
| `hibernate.format_sql` | `true`, `false` | Format logged SQL |
| `hibernate.hbm2ddl.auto` | `validate`, `update`, `create`, `none` | Schema handling |
| `hibernate.jdbc.batch_size` | Integer | Batch size for bulk ops |

### web.xml

Location: `wordnetloom-server/src/main/webapp/WEB-INF/web.xml`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns="http://xmlns.jcp.org/xml/ns/javaee"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://xmlns.jcp.org/xml/ns/javaee
                             http://xmlns.jcp.org/xml/ns/javaee/web-app_4_0.xsd"
         version="4.0">

    <display-name>WordNetLoom Server</display-name>

    <!-- Security configuration -->
    <login-config>
        <auth-method>BEARER_TOKEN</auth-method>
    </login-config>

    <!-- Session timeout (minutes) -->
    <session-config>
        <session-timeout>30</session-timeout>
    </session-config>

</web-app>
```

## Database Configuration

### MySQL Server Settings

Recommended MySQL configuration (`my.cnf` or `my.ini`):

```ini
[mysqld]
# Character set
character-set-server = utf8mb4
collation-server = utf8mb4_unicode_ci

# InnoDB settings
innodb_buffer_pool_size = 1G
innodb_log_file_size = 256M
innodb_flush_log_at_trx_commit = 2
innodb_file_per_table = 1

# Connection settings
max_connections = 200
wait_timeout = 28800
interactive_timeout = 28800

# Query cache (deprecated in MySQL 8, use for older versions)
# query_cache_type = 1
# query_cache_size = 64M

# Logging
slow_query_log = 1
slow_query_log_file = /var/log/mysql/slow.log
long_query_time = 2
```

### Connection URL Parameters

```
jdbc:mysql://localhost:3306/wordnet?useSSL=false&serverTimezone=UTC&characterEncoding=UTF-8&useUnicode=true
```

| Parameter | Value | Description |
|-----------|-------|-------------|
| `useSSL` | `false`, `true` | Enable SSL |
| `serverTimezone` | `UTC`, timezone | Server timezone |
| `characterEncoding` | `UTF-8` | Character encoding |
| `useUnicode` | `true` | Enable Unicode |
| `allowPublicKeyRetrieval` | `true` | Allow public key retrieval |

## WildFly Configuration

### Datasource Configuration

Add to `standalone.xml` in the `<datasources>` subsystem:

```xml
<datasources>
    <datasource jndi-name="java:jboss/datasources/wordnet"
                pool-name="WordNetDS"
                enabled="true"
                use-java-context="true">

        <connection-url>jdbc:mysql://localhost:3306/wordnet?useSSL=false&amp;serverTimezone=UTC</connection-url>

        <driver>mysql</driver>

        <pool>
            <min-pool-size>5</min-pool-size>
            <max-pool-size>20</max-pool-size>
            <prefill>true</prefill>
        </pool>

        <security>
            <user-name>wordnet</user-name>
            <password>password</password>
        </security>

        <validation>
            <valid-connection-checker class-name="org.jboss.jca.adapters.jdbc.extensions.mysql.MySQLValidConnectionChecker"/>
            <exception-sorter class-name="org.jboss.jca.adapters.jdbc.extensions.mysql.MySQLExceptionSorter"/>
            <validate-on-match>true</validate-on-match>
        </validation>

        <timeout>
            <blocking-timeout-millis>30000</blocking-timeout-millis>
            <idle-timeout-minutes>5</idle-timeout-minutes>
        </timeout>
    </datasource>

    <drivers>
        <driver name="mysql" module="com.mysql">
            <driver-class>com.mysql.cj.jdbc.Driver</driver-class>
        </driver>
    </drivers>
</datasources>
```

### MySQL Driver Module

Create `$WILDFLY_HOME/modules/system/layers/base/com/mysql/main/module.xml`:

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

### JVM Settings

Edit `standalone.conf` (Linux) or `standalone.conf.bat` (Windows):

```bash
# Memory settings
JAVA_OPTS="-Xms512m -Xmx2048m -XX:MetaspaceSize=128m -XX:MaxMetaspaceSize=512m"

# Garbage collection
JAVA_OPTS="$JAVA_OPTS -XX:+UseG1GC"

# Character encoding
JAVA_OPTS="$JAVA_OPTS -Dfile.encoding=UTF-8"

# Timezone
JAVA_OPTS="$JAVA_OPTS -Duser.timezone=UTC"
```

### CLI Scripts

**configure-mysql.cli** - Configure datasource via CLI:

```
# Connect to WildFly
connect

# Add MySQL driver
/subsystem=datasources/jdbc-driver=mysql:add(driver-name=mysql,driver-module-name=com.mysql,driver-class-name=com.mysql.cj.jdbc.Driver)

# Add datasource
data-source add --name=WordNetDS --jndi-name=java:jboss/datasources/wordnet --driver-name=mysql --connection-url=jdbc:mysql://localhost:3306/wordnet --user-name=wordnet --password=password
```

Run with:
```bash
$WILDFLY_HOME/bin/jboss-cli.sh --file=configure-mysql.cli
```

## Docker Configuration

### docker-compose.yml

```yaml
version: '3.1'

services:
  # MySQL Database
  db:
    image: mysql:8.0.13
    restart: always
    volumes:
      # Init scripts
      - .:/docker-entrypoint-initdb.d
      # Persistent data
      - /storage/docker/mysql-wordnet-db:/var/lib/mysql
    command: --default-authentication-plugin=mysql_native_password
    environment:
      MYSQL_ROOT_PASSWORD: password
      MYSQL_DATABASE: wordnet
    ports:
      - "33366:3306"
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost"]
      interval: 10s
      timeout: 5s
      retries: 5

  # WildFly Application Server
  wildfly:
    build:
      context: .
      dockerfile: wildfly.dockerfile
    restart: always
    depends_on:
      db:
        condition: service_healthy
    environment:
      WAIT_HOSTS: db:3306
      JAVA_OPTS: "-Xms512m -Xmx1024m"
    ports:
      - "8080:8080"   # HTTP
      - "8009:8009"   # AJP
      - "9990:9990"   # Management
    links:
      - db:db

  # Maven Build Container
  maven:
    build:
      context: .
      dockerfile: maven.dockerfile
    container_name: maven
    volumes:
      - .:/wordnetloom
      - maven-repo:/root/.m2
    working_dir: /wordnetloom
    entrypoint: ['mvn']

volumes:
  db_data:
  maven-repo:
```

### Environment Variables

| Variable | Service | Description |
|----------|---------|-------------|
| `MYSQL_ROOT_PASSWORD` | db | MySQL root password |
| `MYSQL_DATABASE` | db | Database name |
| `WAIT_HOSTS` | wildfly | Services to wait for |
| `JAVA_OPTS` | wildfly | JVM options |

### Custom Dockerfile Variables

**wildfly.dockerfile**:
```dockerfile
FROM jboss/wildfly:20.0.1.Final

ARG DB_HOST=db
ARG DB_PORT=3306
ARG DB_NAME=wordnet
ARG DB_USER=wordnet
ARG DB_PASSWORD=password

ENV DB_HOST=${DB_HOST}
ENV DB_PORT=${DB_PORT}
ENV DB_NAME=${DB_NAME}
ENV DB_USER=${DB_USER}
ENV DB_PASSWORD=${DB_PASSWORD}
```

## Environment Variables

### Server Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `DB_HOST` | localhost | Database host |
| `DB_PORT` | 3306 | Database port |
| `DB_NAME` | wordnet | Database name |
| `DB_USER` | wordnet | Database user |
| `DB_PASSWORD` | - | Database password |
| `JWT_SECRET` | - | JWT signing secret |
| `JWT_EXPIRATION` | 86400000 | Token expiration (ms) |

### Setting Environment Variables

**Linux/macOS:**
```bash
export DB_HOST=localhost
export DB_PASSWORD=secret
```

**Windows:**
```cmd
set DB_HOST=localhost
set DB_PASSWORD=secret
```

**Docker Compose:**
```yaml
environment:
  - DB_HOST=db
  - DB_PASSWORD=secret
```

## Security Configuration

### JWT Settings

Configure in `JwtManager.java` or via environment:

```java
// Default values (override with environment variables)
private static final String SECRET = System.getenv("JWT_SECRET") != null
    ? System.getenv("JWT_SECRET")
    : "wordnetloom-default-secret";

private static final long EXPIRATION = System.getenv("JWT_EXPIRATION") != null
    ? Long.parseLong(System.getenv("JWT_EXPIRATION"))
    : 86400000L; // 24 hours
```

### Password Hashing

Passwords are hashed using SHA-256. Configure in `SecurityService`.

## Logging Configuration

### WildFly Logging

In `standalone.xml`:

```xml
<subsystem xmlns="urn:jboss:domain:logging:8.0">
    <console-handler name="CONSOLE">
        <level name="INFO"/>
        <formatter>
            <named-formatter name="COLOR-PATTERN"/>
        </formatter>
    </console-handler>

    <periodic-rotating-file-handler name="FILE">
        <formatter>
            <named-formatter name="PATTERN"/>
        </formatter>
        <file relative-to="jboss.server.log.dir" path="server.log"/>
        <suffix value=".yyyy-MM-dd"/>
        <append value="true"/>
    </periodic-rotating-file-handler>

    <logger category="pl.edu.pwr.wordnetloom">
        <level name="DEBUG"/>
    </logger>

    <root-logger>
        <level name="INFO"/>
        <handlers>
            <handler name="CONSOLE"/>
            <handler name="FILE"/>
        </handlers>
    </root-logger>
</subsystem>
```

### Client Logging (Logback)

Create `logback.xml` in client resources:

```xml
<configuration>
    <appender name="STDOUT" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>
        </encoder>
    </appender>

    <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>logs/wordnetloom-client.log</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>logs/wordnetloom-client.%d{yyyy-MM-dd}.log</fileNamePattern>
            <maxHistory>30</maxHistory>
        </rollingPolicy>
        <encoder>
            <pattern>%d{yyyy-MM-dd HH:mm:ss} [%thread] %-5level %logger{36} - %msg%n</pattern>
        </encoder>
    </appender>

    <logger name="pl.edu.pwr.wordnetloom.client" level="DEBUG"/>
    <logger name="pl.edu.pwr.wordnetloom.client.service" level="INFO"/>

    <root level="INFO">
        <appender-ref ref="STDOUT"/>
        <appender-ref ref="FILE"/>
    </root>
</configuration>
```

## Performance Tuning

### Recommended Production Settings

```properties
# Database connection pool
pool.min-size=10
pool.max-size=50

# Hibernate settings
hibernate.jdbc.batch_size=50
hibernate.order_inserts=true
hibernate.order_updates=true
hibernate.jdbc.fetch_size=50

# Query cache
hibernate.cache.use_second_level_cache=true
hibernate.cache.use_query_cache=true
```

### JVM Tuning

```bash
# Production JVM settings
JAVA_OPTS="-server"
JAVA_OPTS="$JAVA_OPTS -Xms2g -Xmx4g"
JAVA_OPTS="$JAVA_OPTS -XX:+UseG1GC"
JAVA_OPTS="$JAVA_OPTS -XX:MaxGCPauseMillis=200"
JAVA_OPTS="$JAVA_OPTS -XX:+HeapDumpOnOutOfMemoryError"
JAVA_OPTS="$JAVA_OPTS -XX:HeapDumpPath=/var/log/wildfly/"
```
