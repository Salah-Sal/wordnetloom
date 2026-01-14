# Installation Guide

This guide provides step-by-step instructions for installing and setting up WordNetLoom on your system.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Quick Start with Docker](#quick-start-with-docker)
- [Manual Installation](#manual-installation)
- [Database Setup](#database-setup)
- [Server Installation](#server-installation)
- [Client Installation](#client-installation)
- [Verification](#verification)
- [Troubleshooting](#troubleshooting)

## Prerequisites

### Required Software

| Software | Version | Purpose |
|----------|---------|---------|
| Java JDK | 11+ | Runtime and compilation |
| Maven | 3.x | Build management |
| MySQL | 8.0+ | Database server |
| Git | Any | Source code management |

### Optional Software (for Docker deployment)

| Software | Version | Purpose |
|----------|---------|---------|
| Docker | 19.0+ | Containerization |
| Docker Compose | 1.25+ | Container orchestration |

### System Requirements

- **RAM**: Minimum 4GB, Recommended 8GB
- **Disk Space**: 2GB for application, additional space for database
- **OS**: Windows 10+, macOS 10.14+, or Linux (Ubuntu 18.04+)

## Quick Start with Docker

The fastest way to get WordNetLoom running is using Docker.

### Step 1: Clone the Repository

```bash
git clone <repository-url> wordnetloom
cd wordnetloom
```

### Step 2: Build and Start Services

**On Linux/macOS:**
```bash
chmod +x build-server.sh
./build-server.sh
docker-compose up -d
```

**On Windows (PowerShell):**
```powershell
docker-compose run --rm maven clean
docker-compose run --rm maven package
docker-compose build wildfly
docker-compose up -d
```

### Step 3: Verify Installation

```bash
# Check running containers
docker-compose ps

# Check server logs
docker-compose logs wildfly

# Test API endpoint
curl http://localhost:8080/wordnetloom-server/resources/
```

### Step 4: Access the Application

- **API**: http://localhost:8080/wordnetloom-server/resources/
- **Database**: localhost:33366 (MySQL)

## Manual Installation

For development or when Docker is not available.

### Step 1: Install Java 11

**Windows:**
1. Download JDK 11 from [AdoptOpenJDK](https://adoptopenjdk.net/) or [Oracle](https://www.oracle.com/java/technologies/javase-jdk11-downloads.html)
2. Run the installer
3. Set `JAVA_HOME` environment variable:
   ```cmd
   setx JAVA_HOME "C:\Program Files\Java\jdk-11"
   setx PATH "%PATH%;%JAVA_HOME%\bin"
   ```

**macOS:**
```bash
brew install openjdk@11
export JAVA_HOME=$(/usr/libexec/java_home -v 11)
```

**Linux (Ubuntu/Debian):**
```bash
sudo apt update
sudo apt install openjdk-11-jdk
export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-amd64
```

### Step 2: Install Maven

**Windows:**
1. Download Maven from [Apache Maven](https://maven.apache.org/download.cgi)
2. Extract to `C:\Program Files\Maven`
3. Add to PATH:
   ```cmd
   setx PATH "%PATH%;C:\Program Files\Maven\bin"
   ```

**macOS:**
```bash
brew install maven
```

**Linux:**
```bash
sudo apt install maven
```

### Step 3: Install MySQL 8.0

**Windows:**
1. Download MySQL Installer from [MySQL Downloads](https://dev.mysql.com/downloads/installer/)
2. Run installer and select "MySQL Server 8.0"
3. Configure root password during setup

**macOS:**
```bash
brew install mysql@8.0
brew services start mysql@8.0
```

**Linux:**
```bash
sudo apt install mysql-server-8.0
sudo systemctl start mysql
sudo mysql_secure_installation
```

### Step 4: Install JavaFX SDK (for client)

The client requires JavaFX 11. Download from [OpenJFX](https://openjfx.io/).

1. Download JavaFX SDK 11 for your platform
2. Extract to a known location (e.g., `/opt/javafx-sdk-11.0.2`)
3. Note the path for later configuration

## Database Setup

### Create Database and User

Connect to MySQL as root:

```bash
mysql -u root -p
```

Execute the following SQL:

```sql
-- Create database
CREATE DATABASE wordnet CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

-- Create user
CREATE USER 'wordnet'@'localhost' IDENTIFIED BY 'your_password';

-- Grant privileges
GRANT ALL PRIVILEGES ON wordnet.* TO 'wordnet'@'localhost';
FLUSH PRIVILEGES;

-- Exit
EXIT;
```

### Initialize Schema

The schema is automatically created by Flyway on first server startup. Alternatively, run manually:

```bash
mysql -u wordnet -p wordnet < wordnetloom-server/src/main/resources/db/migration/V1_0__Schema.sql
```

### Create Initial Admin User

```sql
USE wordnet;

INSERT INTO tbl_users (email, firstname, lastname, password, role)
VALUES (
    'admin@example.com',
    'Admin',
    'User',
    SHA2('your_admin_password', 256),
    'ADMIN'
);
```

## Server Installation

### Option A: Deploy to Existing WildFly

If you have WildFly installed:

1. **Build the WAR file:**
   ```bash
   mvn clean package -pl wordnetloom-server
   ```

2. **Configure datasource** (see [CONFIGURATION.md](./CONFIGURATION.md)):
   - Add MySQL driver to WildFly modules
   - Configure datasource in standalone.xml

3. **Deploy:**
   ```bash
   cp wordnetloom-server/target/wordnetloom-server.war $WILDFLY_HOME/standalone/deployments/
   ```

### Option B: Download and Configure WildFly

1. **Download WildFly 20.0.1:**
   ```bash
   wget https://download.jboss.org/wildfly/20.0.1.Final/wildfly-20.0.1.Final.tar.gz
   tar -xzf wildfly-20.0.1.Final.tar.gz
   export WILDFLY_HOME=$(pwd)/wildfly-20.0.1.Final
   ```

2. **Download MySQL Connector:**
   ```bash
   wget https://dev.mysql.com/get/Downloads/Connector-J/mysql-connector-java-8.0.23.tar.gz
   tar -xzf mysql-connector-java-8.0.23.tar.gz
   ```

3. **Add MySQL Module to WildFly:**
   ```bash
   mkdir -p $WILDFLY_HOME/modules/system/layers/base/com/mysql/main
   cp mysql-connector-java-8.0.23/mysql-connector-java-8.0.23.jar \
      $WILDFLY_HOME/modules/system/layers/base/com/mysql/main/
   ```

4. **Create module.xml:**
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

5. **Configure datasource in standalone.xml:**
   Add to `<datasources>` section:
   ```xml
   <datasource jndi-name="java:jboss/datasources/wordnet" pool-name="WordNetDS">
       <connection-url>jdbc:mysql://localhost:3306/wordnet?useSSL=false&amp;serverTimezone=UTC</connection-url>
       <driver>mysql</driver>
       <security>
           <user-name>wordnet</user-name>
           <password>your_password</password>
       </security>
   </datasource>
   ```

6. **Start WildFly:**
   ```bash
   $WILDFLY_HOME/bin/standalone.sh
   ```

7. **Deploy the application:**
   ```bash
   mvn clean package
   cp wordnetloom-server/target/wordnetloom-server.war $WILDFLY_HOME/standalone/deployments/
   ```

## Client Installation

### Build the Client

```bash
mvn clean package -pl wordnetloom-client
```

### Configure Server Connection

Edit `wordnetloom-client/application.properties`:

```properties
# Server URL
server.host=http://localhost:8080

# Lexicon icons (optional)
lexicon.1=polish.png
lexicon.2=english.png
```

### Run the Client

**With JavaFX in module path:**

```bash
java --module-path /path/to/javafx-sdk-11.0.2/lib \
     --add-modules javafx.controls,javafx.fxml,javafx.swing,javafx.web \
     -jar wordnetloom-client/target/wordnetloom-client-3.0.jar
```

**Or use the distribution package:**

```bash
cd wordnetloom-client/target
unzip wordnetloom-client-3.0-dist.zip
cd wordnetloom-client-3.0
java -jar wordnetloom-client-3.0.jar
```

### Building Native Package (Linux)

```bash
./build-linux-client.sh
```

This creates a self-contained application in the `wordnetloom/` directory.

## Verification

### Verify Server

1. **Check server status:**
   ```bash
   curl http://localhost:8080/wordnetloom-server/resources/
   ```

   Expected response:
   ```json
   {
     "_links": {
       "security": "http://localhost:8080/wordnetloom-server/resources/security",
       "lexicons": "http://localhost:8080/wordnetloom-server/resources/lexicons",
       ...
     }
   }
   ```

2. **Test authentication:**
   ```bash
   curl -X POST http://localhost:8080/wordnetloom-server/resources/security/authorize \
        -d "username=admin@example.com&password=your_admin_password" \
        -H "Content-Type: application/x-www-form-urlencoded"
   ```

### Verify Client

1. Launch the client application
2. Enter server URL: `http://localhost:8080`
3. Login with admin credentials
4. Verify main window appears

## Troubleshooting

### Common Issues

#### Java Version Mismatch

**Symptom:** `UnsupportedClassVersionError`

**Solution:** Ensure Java 11 is being used:
```bash
java -version
# Should show: openjdk version "11.x.x"
```

#### Database Connection Failed

**Symptom:** `Communications link failure`

**Solutions:**
1. Verify MySQL is running: `systemctl status mysql`
2. Check connection URL in persistence.xml
3. Verify user credentials
4. Check firewall settings

#### WildFly Deployment Failed

**Symptom:** `.war.failed` file in deployments

**Solutions:**
1. Check WildFly logs: `$WILDFLY_HOME/standalone/log/server.log`
2. Verify datasource configuration
3. Ensure MySQL module is properly configured

#### Client Cannot Connect

**Symptom:** Connection refused error

**Solutions:**
1. Verify server URL in application.properties
2. Check if server is running: `curl http://localhost:8080`
3. Check firewall settings
4. Verify CORS configuration if using different host

#### JavaFX Not Found

**Symptom:** `Module javafx.controls not found`

**Solutions:**
1. Download JavaFX SDK from https://openjfx.io/
2. Add to module path:
   ```bash
   java --module-path /path/to/javafx/lib \
        --add-modules javafx.controls,javafx.fxml \
        -jar wordnetloom-client.jar
   ```

### Getting Help

If you encounter issues not covered here:

1. Check the [FAQ](./FAQ.md) (if available)
2. Search existing issues in the repository
3. Open a new issue with:
   - Operating system and version
   - Java version (`java -version`)
   - Error messages and stack traces
   - Steps to reproduce

## Next Steps

After installation:

1. [Configure the application](./CONFIGURATION.md)
2. [Set up for development](./DEVELOPMENT.md)
3. [Learn the API](./API.md)
4. [Understand the architecture](./ARCHITECTURE.md)
