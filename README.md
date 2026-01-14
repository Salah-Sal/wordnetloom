# WordNetLoom

**WordNetLoom** is a comprehensive desktop and server application for building, editing, and managing wordnets (lexical databases). Developed at Wroclaw University of Science and Technology (PWr), it provides linguists and researchers with powerful tools for creating and maintaining semantic networks of words.

## Overview

WordNetLoom consists of two main components:

- **Desktop Client**: A JavaFX-based GUI application for end-users to interact with wordnet data
- **REST API Server**: A Java EE backend service that manages data persistence and business logic

## Features

- **Multi-language Support**: Create and manage wordnets for multiple languages (Polish, English, etc.)
- **Lexicon Management**: Organize words into lexicons with version control
- **Synset Operations**: Create, edit, and relate synsets (sets of synonymous words)
- **Sense Management**: Handle individual word senses with detailed attributes
- **Relation Types**: Define and manage semantic relationships (hyperonymy, synonymy, etc.)
- **Graph Visualization**: Visual representation of semantic relationships using JUNG library
- **Search Functionality**: Advanced search across senses and synsets
- **User Management**: Role-based access control (USER, ADMIN roles)
- **Corpus Examples**: Link example sentences to word senses
- **Internationalization**: Multi-locale support for UI and data

## Quick Start

### Prerequisites

- Java 11 or higher
- Maven 3.x
- MySQL 8.0
- Docker & Docker Compose (for containerized deployment)

### Running with Docker

```bash
# Build and start all services
./build-server.sh
docker-compose up -d
```

### Manual Build

```bash
# Build all modules
mvn clean package

# Deploy server WAR to WildFly
# Run client JAR
java -jar wordnetloom-client/target/wordnetloom-client-3.0.jar
```

## Project Structure

```
wordnetloom/
├── wordnetloom-client/     # JavaFX desktop application
│   ├── src/main/java/      # Client source code
│   └── src/main/resources/ # FXML, CSS, properties
├── wordnetloom-server/     # REST API server
│   ├── src/main/java/      # Server source code
│   └── src/main/resources/ # Persistence config, migrations
├── docs/                   # Documentation
├── docker-compose.yml      # Docker orchestration
└── pom.xml                 # Root Maven configuration
```

## Documentation

Detailed documentation is available in the [docs](./docs) folder:

- [Architecture Overview](./docs/ARCHITECTURE.md) - System design and patterns
- [Installation Guide](./docs/INSTALLATION.md) - Setup instructions
- [API Reference](./docs/API.md) - REST endpoint documentation
- [Database Schema](./docs/DATABASE.md) - Data model documentation
- [Client Guide](./docs/CLIENT.md) - Desktop application guide
- [Server Guide](./docs/SERVER.md) - Backend service documentation
- [Configuration](./docs/CONFIGURATION.md) - Configuration options
- [Deployment](./docs/DEPLOYMENT.md) - Production deployment guide
- [Development](./docs/DEVELOPMENT.md) - Contributing guide

## Technology Stack

| Component | Technology |
|-----------|------------|
| **Client UI** | JavaFX 11, FXML |
| **Client Architecture** | MVVM (mvvmfx 1.8.0) |
| **Graph Visualization** | JUNG 2.0.1 |
| **Server Framework** | Java EE 8 |
| **REST API** | JAX-RS (RESTEasy 3.6.1) |
| **Database** | MySQL 8.0 |
| **ORM** | Hibernate 5.3.6 |
| **Migrations** | Flyway 5.2.4 |
| **Authentication** | JWT (Nimbus JOSE) |
| **Application Server** | WildFly 20.0.1 |
| **Containerization** | Docker, Docker Compose |

## Domain Concepts

### Key Entities

- **Lexicon**: A wordnet for a specific language with metadata (version, license, etc.)
- **Synset**: A set of synonymous word senses representing a single concept
- **Sense**: A specific meaning of a word, linked to a synset
- **Word**: A lexical entry (lemma)
- **Relation Type**: Defines semantic relationships (hypernymy, meronymy, etc.)
- **Domain**: Subject area classification (e.g., medicine, law)
- **Part of Speech**: Grammatical category (noun, verb, adjective, etc.)

### Relationships

```
Lexicon (1) ─────── (*) Synset
                        │
Synset  (1) ─────── (*) Sense ─── (1) Word
                        │
Sense   (*) ─────── (*) Sense     (via SenseRelation)
Synset  (*) ─────── (*) Synset    (via SynsetRelation)
```

## API Overview

The server exposes a RESTful API with HATEOAS links:

| Endpoint | Description |
|----------|-------------|
| `GET /` | API root with navigation links |
| `GET /lexicons` | List all lexicons |
| `GET /senses/search` | Search senses with filters |
| `GET /synsets/search` | Search synsets with filters |
| `POST /security/authorize` | Authenticate and get JWT |
| `GET /dictionaries` | Get dictionary metadata |

See [API Documentation](./docs/API.md) for complete endpoint reference.

## License

Copyright (C) Wroclaw University of Science and Technology (PWr), 2013-2020.

This program is free software: you can redistribute it and/or modify it under the terms of the GNU Lesser General Public License as published by the Free Software Foundation, either version 3 of the License, or (at your option) any later version.

See [LICENSE.md](./LICENSE.md) for full license text.

## Contributing

We welcome contributions! Please see our [Development Guide](./docs/DEVELOPMENT.md) for:

- Setting up a development environment
- Code style guidelines
- Submitting pull requests
- Reporting issues

## Support

For questions, issues, or feature requests, please open an issue on the project repository.

---

*WordNetLoom - Building semantic networks for language understanding*
