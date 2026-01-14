# Architecture Overview

This document describes the architecture of WordNetLoom, including system design, component structure, and key patterns used throughout the codebase.

## System Architecture

WordNetLoom follows a **client-server architecture** with a clear separation between the desktop GUI client and the REST API server.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                              CLIENT TIER                                 │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                    JavaFX Desktop Application                     │   │
│  │  ┌─────────┐  ┌─────────────┐  ┌─────────────┐  ┌────────────┐  │   │
│  │  │  Views  │──│ ViewModels  │──│  Services   │──│ REST Client│  │   │
│  │  │ (FXML)  │  │   (MVVM)    │  │   (CDI)     │  │ (RESTEasy) │  │   │
│  │  └─────────┘  └─────────────┘  └─────────────┘  └────────────┘  │   │
│  └─────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    │ HTTP/REST (JSON)
                                    │ JWT Authentication
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                              SERVER TIER                                 │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                      WildFly Application Server                   │   │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐  │   │
│  │  │  Boundary   │──│   Control   │──│        Entity           │  │   │
│  │  │ (REST API)  │  │ (Services)  │  │   (JPA/Hibernate)       │  │   │
│  │  └─────────────┘  └─────────────┘  └─────────────────────────┘  │   │
│  └─────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────┘
                                    │
                                    │ JDBC
                                    ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                              DATA TIER                                   │
│  ┌─────────────────────────────────────────────────────────────────┐   │
│  │                         MySQL Database                            │   │
│  └─────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────┘
```

## Client Architecture

### MVVM Pattern

The client follows the **Model-View-ViewModel (MVVM)** pattern using the `mvvmfx` framework:

```
┌─────────────────┐     ┌──────────────────┐     ┌─────────────────┐
│      View       │◄───►│    ViewModel     │◄───►│     Model       │
│    (FXML)       │     │    (Java)        │     │    (POJOs)      │
└─────────────────┘     └──────────────────┘     └─────────────────┘
        │                        │                        │
        │ Data Binding           │ Business Logic         │ Data
        │ User Actions           │ State Management       │ Structures
        ▼                        ▼                        ▼
   LoginDialogView.fxml    LoginDialogViewModel.java   User.java
   MainView.fxml           MainViewModel.java          Sense.java
   SearchView.fxml         SearchViewModel.java        Synset.java
```

### Client Package Structure

```
pl.edu.pwr.wordnetloom.client/
├── Application.java              # Entry point (MvvmfxCdiApplication)
├── config/
│   └── ResourceProvider.java     # CDI configuration producer
├── events/                       # CDI event classes
│   ├── LoadGraphEvent.java
│   ├── OpenMainApplicationEvent.java
│   ├── ShowSenseRelationsEvent.java
│   ├── TriggerShutdownEvent.java
│   └── ... (15+ event types)
├── model/                        # Data transfer objects
│   ├── Dictionary.java
│   ├── Lexicon.java
│   ├── Sense.java
│   ├── Synset.java
│   ├── User.java
│   └── ... (30+ model classes)
├── service/                      # Backend communication
│   ├── RemoteService.java        # Main REST client (1200+ lines)
│   ├── Dictionaries.java         # Dictionary cache
│   └── RelationTypeService.java  # Relation type cache
└── ui/                           # UI components (MVVM)
    ├── main/                     # Main application window
    ├── graph/                    # Graph visualization
    ├── search/                   # Search functionality
    ├── senseform/                # Sense editing
    ├── synsetform/               # Synset editing
    ├── relationform/             # Relation editing
    └── ... (30+ UI packages)
```

### Key Client Components

#### RemoteService (`service/RemoteService.java`)

Central service handling all REST API communication:

```java
@Singleton
public class RemoteService {
    private Client client;

    // Authentication
    public Optional<User> login(String email, String password);

    // Sense operations
    public List<Sense> findSenses(SearchFilter filter);
    public Optional<Sense> findSense(UUID id);
    public void saveSense(Sense sense);

    // Synset operations
    public List<Synset> findSynsets(SearchFilter filter);
    public Optional<Synset> findSynset(UUID id);

    // Graph operations
    public NodeExpanded getSenseGraph(UUID id);
    public NodeExpanded getSynsetGraph(UUID id);
}
```

#### Event System

The client uses CDI events for loose coupling between components:

```java
// Firing events
@Inject Event<LoadGraphEvent> loadGraphEvent;
loadGraphEvent.fire(new LoadGraphEvent(synsetId));

// Observing events
public void onLoadGraph(@Observes LoadGraphEvent event) {
    loadGraph(event.getSynsetId());
}
```

### Graph Visualization

Uses JUNG (Java Universal Network/Graph) library for semantic network visualization:

```
┌───────────────────────────────────────────────────┐
│                  VisualizationViewer              │
│  ┌─────────────────────────────────────────────┐ │
│  │            Graph<Node, Edge>                 │ │
│  │                                              │ │
│  │    [Synset A] ──hypernym──► [Synset B]      │ │
│  │         │                        │          │ │
│  │      meronym                  holonym       │ │
│  │         ▼                        ▼          │ │
│  │    [Synset C]              [Synset D]       │ │
│  │                                              │ │
│  └─────────────────────────────────────────────┘ │
└───────────────────────────────────────────────────┘
```

## Server Architecture

### Layered Architecture (BCE Pattern)

The server follows the **Boundary-Control-Entity** pattern:

```
┌─────────────────────────────────────────────────────────────────┐
│                    BOUNDARY LAYER (REST API)                     │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────────────────┐ │
│  │SenseResource │ │SynsetResource│ │SecurityResource          │ │
│  │ @Path        │ │ @Path        │ │ @Path                    │ │
│  │ @GET @POST   │ │ @GET @POST   │ │ @POST /authorize         │ │
│  └──────────────┘ └──────────────┘ └──────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    CONTROL LAYER (Services)                      │
│  ┌───────────────────┐ ┌───────────────────┐ ┌────────────────┐ │
│  │SenseQueryService  │ │SenseCommandService│ │GraphQueryService│ │
│  │ - findById()      │ │ - save()          │ │ - senseGraph() │ │
│  │ - findByFilter()  │ │ - update()        │ │ - synsetGraph()│ │
│  └───────────────────┘ └───────────────────┘ └────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    ENTITY LAYER (JPA Entities)                   │
│  ┌────────┐ ┌────────┐ ┌────────────┐ ┌─────────────────────┐  │
│  │ Sense  │ │ Synset │ │ Lexicon    │ │ RelationType        │  │
│  │ @Entity│ │ @Entity│ │ @Entity    │ │ @Entity             │  │
│  └────────┘ └────────┘ └────────────┘ └─────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### Server Package Structure

```
pl.edu.pwr.wordnetloom.server/
└── business/
    ├── EntityBuilder.java          # JSON response builder
    ├── LinkBuilder.java            # HATEOAS link generator
    ├── OperationResult.java        # Result wrapper with errors
    ├── RootResource.java           # API root endpoint
    ├── SearchFilter.java           # Search parameter holder
    │
    ├── sense/
    │   ├── boundary/
    │   │   └── SenseResource.java      # REST endpoints
    │   ├── control/
    │   │   ├── SenseQueryService.java  # Read operations
    │   │   └── SenseCommandService.java # Write operations
    │   └── entity/
    │       ├── Sense.java              # JPA entity
    │       ├── SenseAttributes.java    # Extended attributes
    │       ├── SenseExample.java       # Usage examples
    │       └── SenseRelation.java      # Sense-to-sense relations
    │
    ├── synset/
    │   ├── boundary/
    │   │   └── SynsetResource.java
    │   ├── control/
    │   │   ├── SynsetQueryService.java
    │   │   └── SynsetCommandService.java
    │   └── entity/
    │       ├── Synset.java
    │       ├── SynsetAttributes.java
    │       └── SynsetRelation.java
    │
    ├── lexicon/
    │   ├── boundary/LexiconResource.java
    │   ├── control/LexiconQueryService.java
    │   └── entity/Lexicon.java
    │
    ├── dictionary/
    │   ├── boundary/DictionaryResource.java
    │   ├── control/DictionaryQueryService.java
    │   └── entity/
    │       ├── Dictionary.java
    │       ├── Domain.java
    │       ├── PartOfSpeech.java
    │       ├── Register.java
    │       └── Status.java
    │
    ├── relationtype/
    │   ├── boundary/RelationTypeResource.java
    │   └── entity/
    │       ├── RelationType.java
    │       └── RelationTest.java
    │
    ├── security/
    │   ├── boundary/
    │   │   ├── SecurityResource.java
    │   │   └── SecurityService.java
    │   ├── control/
    │   │   └── JwtManager.java
    │   └── entity/
    │       └── Jwt.java
    │
    ├── user/
    │   ├── boundary/
    │   │   ├── UserResource.java
    │   │   └── UserService.java
    │   └── entity/
    │       ├── User.java
    │       └── UserSettings.java
    │
    ├── graph/
    │   ├── control/GraphQueryService.java
    │   └── entity/
    │       ├── NodeExpanded.java
    │       └── NodeHidden.java
    │
    ├── corpusexample/
    │   └── boundary/CorpusExampleResource.java
    │
    └── localisation/
        └── control/LocalisedStringsQueryService.java
```

### HATEOAS Implementation

The API follows HATEOAS (Hypermedia as the Engine of Application State) principles:

```java
// LinkBuilder generates navigation links
public class LinkBuilder {
    public URI forSense(Sense sense, UriInfo uriInfo) {
        return uriInfo.getBaseUriBuilder()
            .path(SenseResource.class)
            .path(sense.getId().toString())
            .build();
    }
}

// EntityBuilder creates JSON responses with links
public JsonObject buildSense(Sense s, URI self, Locale locale) {
    return createObjectBuilder()
        .add("id", s.getId().toString())
        .add("lemma", s.getWord().getWord())
        .add("_links", createObjectBuilder()
            .add("self", self.toString())
            .add("synset", linkBuilder.forSynset(s.getSynset()))
            .add("relations", linkBuilder.forSenseRelations(s)))
        .build();
}
```

### Authentication Flow

```
┌────────┐                    ┌────────────────┐                ┌──────────┐
│ Client │                    │ SecurityResource│                │JwtManager│
└───┬────┘                    └───────┬────────┘                └────┬─────┘
    │                                 │                              │
    │ POST /security/authorize        │                              │
    │ {username, password}            │                              │
    │────────────────────────────────►│                              │
    │                                 │                              │
    │                                 │ authenticate()               │
    │                                 │─────────────────────────────►│
    │                                 │                              │
    │                                 │◄─────────────────────────────│
    │                                 │     User (if valid)          │
    │                                 │                              │
    │                                 │ createJwt(user)              │
    │                                 │─────────────────────────────►│
    │                                 │                              │
    │                                 │◄─────────────────────────────│
    │                                 │     JWT Token                │
    │                                 │                              │
    │◄────────────────────────────────│                              │
    │     {token: "eyJ..."}           │                              │
    │                                 │                              │
```

## Data Flow

### Search Flow Example

```
┌──────────────────────────────────────────────────────────────────────────┐
│ CLIENT                                                                    │
│                                                                          │
│  [SearchView] ──user input──► [SearchViewModel] ──search()──►            │
│                                                                          │
│  [RemoteService.findSenses(filter)]                                      │
│         │                                                                │
│         │ HTTP GET /senses/search?lemma=dog&pos=1                       │
│         ▼                                                                │
└──────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌──────────────────────────────────────────────────────────────────────────┐
│ SERVER                                                                    │
│                                                                          │
│  [SenseResource.search()] ──► [SenseQueryService.findByFilter()]        │
│                                        │                                 │
│                                        ▼                                 │
│                               [JPA Query Execution]                      │
│                                        │                                 │
│                                        ▼                                 │
│  [EntityBuilder.buildPaginatedSenseSearch()] ◄── List<Sense>            │
│         │                                                                │
│         │ JSON Response with HATEOAS links                              │
│         ▼                                                                │
└──────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌──────────────────────────────────────────────────────────────────────────┐
│ CLIENT                                                                    │
│                                                                          │
│  [RemoteService] ──parse JSON──► [List<Sense>]                          │
│                                        │                                 │
│                                        ▼                                 │
│  [SearchViewModel] ──update──► [ObservableList<SenseRow>]               │
│                                        │                                 │
│                                        ▼                                 │
│  [SearchView TableView] ──display──► User sees results                  │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

## Key Design Patterns

### 1. Repository Pattern (Server)
Query and command services act as repositories for entities.

### 2. DTO Pattern (Client-Server)
Model classes in client serve as DTOs for REST communication.

### 3. Observer Pattern (Client)
CDI events enable loose coupling between UI components.

### 4. Builder Pattern (Server)
`EntityBuilder` and `LinkBuilder` construct JSON responses.

### 5. Service Layer Pattern
Clear separation between REST endpoints and business logic.

### 6. Singleton Pattern (Client)
`RemoteService` and caches are CDI singletons.

## Security Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        Security Flow                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. User Login                                                   │
│     └─► POST /security/authorize (username, password)            │
│         └─► Server validates credentials                         │
│             └─► Returns JWT token                                │
│                                                                  │
│  2. Authenticated Requests                                       │
│     └─► Client sends: Authorization: Bearer <JWT>                │
│         └─► Server validates JWT                                 │
│             └─► Extracts user claims                             │
│                 └─► Processes request                            │
│                                                                  │
│  3. Role-Based Access                                            │
│     └─► USER role: Read/write own data                          │
│     └─► ADMIN role: Full system access                          │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Deployment Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    Docker Compose Setup                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐       │
│  │   db         │    │   wildfly    │    │   maven      │       │
│  │   (MySQL)    │◄───│   (WildFly)  │    │   (Build)    │       │
│  │   Port: 33366│    │   Port: 8080 │    │              │       │
│  └──────────────┘    └──────────────┘    └──────────────┘       │
│                                                                  │
│  Volumes:                                                        │
│  - MySQL data: /storage/docker/mysql-wordnet-db                 │
│  - WAR deployment: copied to WildFly deployments                │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Technology Dependencies

### Client Dependencies
```
mvvmfx (1.8.0)           - MVVM framework
javafx (11)              - UI framework
jung (2.0.1)             - Graph library
resteasy-client (3.6.1)  - REST client
controlsfx (11.0.2)      - Enhanced controls
weld-se-core (3.0.0)     - CDI implementation
logback (1.1.2)          - Logging
```

### Server Dependencies
```
javaee-api (8.0)         - Java EE APIs
hibernate (5.3.6)        - ORM
flyway (5.2.4)           - DB migrations
nimbus-jose-jwt (7.9+)   - JWT handling
resteasy (3.6.1)         - JAX-RS implementation
```

## Performance Considerations

1. **Lazy Loading**: JPA entities use lazy loading for relationships
2. **Pagination**: All list endpoints support pagination
3. **Caching**: Client caches dictionaries and relation types
4. **Connection Pooling**: WildFly manages database connection pool
5. **Async Operations**: Client uses background threads for API calls

## Extensibility Points

1. **New Entity Types**: Add new business packages following BCE pattern
2. **New Relation Types**: Configure via dictionary tables
3. **New Languages**: Add lexicons for additional languages
4. **UI Customization**: Create new FXML views with ViewModels
5. **API Extensions**: Add new REST endpoints in boundary layer
