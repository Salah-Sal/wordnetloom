# Development Guide

This document provides guidelines for developers contributing to WordNetLoom.

## Table of Contents

- [Getting Started](#getting-started)
- [Development Environment](#development-environment)
- [Project Structure](#project-structure)
- [Building](#building)
- [Testing](#testing)
- [Code Style](#code-style)
- [Architecture Guidelines](#architecture-guidelines)
- [Contributing](#contributing)
- [Debugging](#debugging)

## Getting Started

### Prerequisites

| Software | Version | Purpose |
|----------|---------|---------|
| JDK | 11+ | Development |
| Maven | 3.6+ | Build |
| MySQL | 8.0 | Database |
| Git | 2.x | Version control |
| IDE | IntelliJ/Eclipse | Development |
| JavaFX SDK | 11 | Client development |

### Quick Setup

```bash
# Clone repository
git clone <repository-url> wordnetloom
cd wordnetloom

# Build all modules
mvn clean install

# Start database (Docker)
docker-compose up -d db

# Run server (in separate terminal)
mvn wildfly:run -pl wordnetloom-server

# Run client
java --module-path /path/to/javafx/lib \
     --add-modules javafx.controls,javafx.fxml,javafx.swing,javafx.web \
     -jar wordnetloom-client/target/wordnetloom-client-3.0.jar
```

## Development Environment

### IDE Setup

#### IntelliJ IDEA

1. **Import Project:**
   - File → Open → Select `pom.xml`
   - Choose "Open as Project"

2. **Configure JDK:**
   - File → Project Structure → Project
   - Set SDK to Java 11

3. **Configure JavaFX:**
   - Run → Edit Configurations
   - Add VM options:
     ```
     --module-path /path/to/javafx-sdk-11/lib
     --add-modules javafx.controls,javafx.fxml,javafx.swing,javafx.web
     ```

4. **Enable Annotation Processing:**
   - Settings → Build → Compiler → Annotation Processors
   - Enable annotation processing

#### Eclipse

1. **Import Project:**
   - File → Import → Maven → Existing Maven Projects

2. **Configure Build Path:**
   - Right-click project → Build Path → Configure Build Path
   - Add JavaFX libraries

### Database Setup

```bash
# Start MySQL container
docker-compose up -d db

# Or use local MySQL
mysql -u root -p
CREATE DATABASE wordnet;
CREATE USER 'wordnet'@'localhost' IDENTIFIED BY 'password';
GRANT ALL ON wordnet.* TO 'wordnet'@'localhost';
```

### Running in Development Mode

**Server:**
```bash
# Using WildFly plugin
mvn wildfly:run -pl wordnetloom-server

# Or deploy to local WildFly
mvn package -pl wordnetloom-server
cp wordnetloom-server/target/wordnetloom-server.war $WILDFLY_HOME/standalone/deployments/
```

**Client:**
```bash
# Build and run
mvn package -pl wordnetloom-client
java --module-path /path/to/javafx/lib \
     --add-modules javafx.controls,javafx.fxml \
     -jar wordnetloom-client/target/wordnetloom-client-3.0.jar
```

## Project Structure

```
wordnetloom/
├── pom.xml                        # Parent POM
├── wordnetloom-client/            # Desktop application
│   ├── pom.xml
│   └── src/
│       ├── main/
│       │   ├── java/              # Java sources
│       │   │   └── pl/edu/pwr/wordnetloom/client/
│       │   │       ├── Application.java
│       │   │       ├── config/
│       │   │       ├── events/
│       │   │       ├── model/
│       │   │       ├── service/
│       │   │       └── ui/
│       │   └── resources/
│       │       ├── icons/
│       │       ├── *.fxml
│       │       └── wordnetloom.css
│       └── test/
│           └── java/
│
├── wordnetloom-server/            # REST API server
│   ├── pom.xml
│   └── src/
│       ├── main/
│       │   ├── java/
│       │   │   └── pl/edu/pwr/wordnetloom/server/
│       │   │       └── business/
│       │   │           ├── sense/
│       │   │           ├── synset/
│       │   │           ├── lexicon/
│       │   │           └── ...
│       │   ├── resources/
│       │   │   ├── META-INF/persistence.xml
│       │   │   └── db/migration/
│       │   └── webapp/
│       │       └── WEB-INF/web.xml
│       └── test/
│           └── java/
│
├── docs/                          # Documentation
├── docker-compose.yml             # Docker setup
└── *.dockerfile                   # Docker builds
```

## Building

### Full Build

```bash
mvn clean install
```

### Module-Specific Build

```bash
# Client only
mvn package -pl wordnetloom-client

# Server only
mvn package -pl wordnetloom-server
```

### Skip Tests

```bash
mvn package -DskipTests
```

### Build with Specific Profile

```bash
mvn package -P production
```

## Testing

### Running Tests

```bash
# All tests
mvn test

# Specific module
mvn test -pl wordnetloom-server

# Specific test class
mvn test -pl wordnetloom-server -Dtest=SenseQueryServiceTest

# Specific test method
mvn test -pl wordnetloom-server -Dtest=SenseQueryServiceTest#testFindById
```

### Test Categories

| Category | Location | Framework |
|----------|----------|-----------|
| Unit tests | `src/test/java` | JUnit 5 |
| Integration tests | `src/test/java` | Arquillian |
| UI tests | `src/test/java` | TestFX |

### Writing Tests

**Unit Test Example:**
```java
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class SenseQueryServiceTest {

    @Test
    void testFindById() {
        // Given
        UUID id = UUID.randomUUID();

        // When
        Optional<Sense> result = service.findById(id);

        // Then
        assertTrue(result.isPresent());
        assertEquals(id, result.get().getId());
    }
}
```

**Integration Test Example:**
```java
import org.junit.jupiter.api.Test;
import javax.inject.Inject;

class SenseResourceIT {

    @Inject
    SenseResource resource;

    @Test
    void testSearchSenses() {
        // Given
        String lemma = "dog";

        // When
        JsonObject result = resource.search(Locale.ENGLISH);

        // Then
        assertNotNull(result);
        assertTrue(result.containsKey("rows"));
    }
}
```

**UI Test Example (TestFX):**
```java
import org.testfx.framework.junit5.ApplicationTest;

class SearchViewTest extends ApplicationTest {

    @Override
    public void start(Stage stage) {
        // Initialize view
    }

    @Test
    void testSearchButton() {
        // Given
        clickOn("#lemmaField").write("dog");

        // When
        clickOn("#searchButton");

        // Then
        verifyThat("#resultsTable", hasItems(1));
    }
}
```

## Code Style

### Java Code Style

Follow standard Java conventions with these specifics:

```java
// Package declaration
package pl.edu.pwr.wordnetloom.server.business.sense.control;

// Imports (grouped: java, javax, third-party, project)
import java.util.List;
import java.util.UUID;

import javax.inject.Inject;
import javax.persistence.EntityManager;

import pl.edu.pwr.wordnetloom.server.business.sense.entity.Sense;

// Class documentation
/**
 * Service for querying sense data.
 */
public class SenseQueryService {

    // Constants
    private static final int DEFAULT_LIMIT = 20;

    // Injected dependencies
    @Inject
    private EntityManager em;

    // Public methods
    public Optional<Sense> findById(UUID id) {
        return Optional.ofNullable(em.find(Sense.class, id));
    }

    // Private methods
    private List<Predicate> buildPredicates(SearchFilter filter) {
        // Implementation
    }
}
```

### Naming Conventions

| Type | Convention | Example |
|------|------------|---------|
| Classes | PascalCase | `SenseQueryService` |
| Methods | camelCase | `findById()` |
| Constants | UPPER_SNAKE | `DEFAULT_LIMIT` |
| Variables | camelCase | `senseList` |
| Packages | lowercase | `pl.edu.pwr.wordnetloom` |

### FXML Naming

```xml
<!-- fx:id uses camelCase -->
<TextField fx:id="lemmaField"/>
<Button fx:id="searchButton"/>
<TableView fx:id="resultsTable"/>
```

## Architecture Guidelines

### Server-Side Patterns

#### Boundary-Control-Entity (BCE)

```
boundary/          # REST endpoints
├── SenseResource.java
└── SenseCommandService.java

control/           # Business logic
└── SenseQueryService.java

entity/            # JPA entities
├── Sense.java
└── SenseAttributes.java
```

#### Service Pattern

```java
// Query service (read operations)
public class SenseQueryService {
    public Optional<Sense> findById(UUID id) { }
    public List<Sense> findByFilter(SearchFilter filter) { }
}

// Command service (write operations)
public class SenseCommandService {
    public OperationResult<Sense> save(JsonObject json) { }
    public void delete(UUID id) { }
}
```

### Client-Side Patterns

#### MVVM Pattern

```
View (FXML)        ←→  ViewModel (Java)  ←→  Model (POJO)
SearchView.fxml        SearchViewModel.java   Sense.java
```

#### Event-Driven Communication

```java
// Publishing events
@Inject
Event<LoadGraphEvent> loadGraphEvent;

loadGraphEvent.fire(new LoadGraphEvent(synsetId));

// Observing events
public void onLoadGraph(@Observes LoadGraphEvent event) {
    // Handle event
}
```

### Database Guidelines

- Use UUIDs for distributed-friendly IDs
- Use foreign keys for referential integrity
- Create indexes for frequently queried columns
- Use Flyway for schema migrations

## Contributing

### Git Workflow

1. **Create feature branch:**
   ```bash
   git checkout -b feature/your-feature-name
   ```

2. **Make changes and commit:**
   ```bash
   git add .
   git commit -m "Add feature description"
   ```

3. **Push and create PR:**
   ```bash
   git push origin feature/your-feature-name
   ```

### Commit Message Format

```
<type>: <subject>

<body>

<footer>
```

Types:
- `feat`: New feature
- `fix`: Bug fix
- `docs`: Documentation
- `style`: Formatting
- `refactor`: Code restructuring
- `test`: Adding tests
- `chore`: Maintenance

Example:
```
feat: Add sense search by domain

- Added domain filter to search form
- Updated SenseQueryService to handle domain parameter
- Added UI for domain selection

Closes #123
```

### Pull Request Guidelines

1. Create descriptive PR title
2. Reference related issues
3. Include test coverage
4. Update documentation if needed
5. Ensure CI passes

### Code Review Checklist

- [ ] Code follows style guidelines
- [ ] Tests are included and pass
- [ ] No security vulnerabilities
- [ ] Performance is acceptable
- [ ] Documentation is updated

## Debugging

### Server Debugging

**Remote Debugging:**

1. Start WildFly with debug port:
   ```bash
   $WILDFLY_HOME/bin/standalone.sh --debug 8787
   ```

2. Connect IDE to `localhost:8787`

**Log Debugging:**
```java
import java.util.logging.Logger;

private static final Logger LOG = Logger.getLogger(MyClass.class.getName());

public void myMethod() {
    LOG.info("Entering myMethod");
    LOG.fine("Debug info: " + variable);
}
```

### Client Debugging

**JavaFX SceneBuilder:**
- Use SceneBuilder to edit FXML files visually
- Set `fx:id` attributes for debugging

**UI Debugging:**
```java
// Print component hierarchy
node.lookupAll("*").forEach(System.out::println);

// CSS debugging
node.setStyle("-fx-border-color: red;");
```

### Database Debugging

```sql
-- Enable query logging
SET GLOBAL general_log = 'ON';
SET GLOBAL general_log_file = '/var/log/mysql/query.log';

-- Show slow queries
SHOW FULL PROCESSLIST;

-- Explain query plan
EXPLAIN SELECT * FROM tbl_sense WHERE word_id = ?;
```

### Common Issues

**"Module not found" (JavaFX):**
```bash
# Add module path
java --module-path /path/to/javafx/lib --add-modules javafx.controls,javafx.fxml ...
```

**"Entity not managed":**
```java
// Ensure entity is attached
entity = em.merge(entity);
```

**"LazyInitializationException":**
```java
// Fetch eagerly or use within transaction
@ManyToOne(fetch = FetchType.EAGER)
private Synset synset;
```

## Useful Commands

```bash
# Check Maven version
mvn -version

# Clean Maven cache
mvn dependency:purge-local-repository

# Show dependency tree
mvn dependency:tree

# Run specific profile
mvn package -P development

# Generate Javadoc
mvn javadoc:javadoc

# Check for dependency updates
mvn versions:display-dependency-updates
```

## Resources

- [JavaFX Documentation](https://openjfx.io/javadoc/11/)
- [mvvmfx Framework](https://github.com/sialcasa/mvvmfx)
- [WildFly Documentation](https://docs.wildfly.org/)
- [Hibernate Reference](https://hibernate.org/orm/documentation/)
- [JAX-RS Specification](https://jakarta.ee/specifications/restful-ws/)
