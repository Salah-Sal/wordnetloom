# Server Application Guide

This document provides detailed information about the WordNetLoom server application.

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Package Structure](#package-structure)
- [REST API Layer](#rest-api-layer)
- [Business Logic Layer](#business-logic-layer)
- [Data Access Layer](#data-access-layer)
- [Security](#security)
- [Database Migrations](#database-migrations)
- [Extending the Server](#extending-the-server)

## Overview

The WordNetLoom server is a Java EE 8 application that provides RESTful web services for wordnet data management. It runs on WildFly application server and uses MySQL as the database backend.

### Key Technologies

| Technology | Version | Purpose |
|------------|---------|---------|
| Java EE | 8.0 | Enterprise APIs |
| JAX-RS | 2.1 | REST endpoints |
| CDI | 2.0 | Dependency injection |
| JPA | 2.2 | Data persistence |
| Hibernate | 5.3.6 | ORM implementation |
| Flyway | 5.2.4 | Database migrations |
| Nimbus JOSE-JWT | 7.9+ | JWT authentication |
| WildFly | 20.0.1 | Application server |

## Architecture

### Layered Architecture (BCE)

The server follows the Boundary-Control-Entity (BCE) pattern:

```
┌─────────────────────────────────────────────────────────────────┐
│                        BOUNDARY LAYER                            │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │                    REST Resources                            ││
│  │  SenseResource, SynsetResource, LexiconResource, etc.       ││
│  │  - HTTP request handling                                     ││
│  │  - JSON serialization/deserialization                        ││
│  │  - Input validation                                          ││
│  │  - Response building                                         ││
│  └─────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                        CONTROL LAYER                             │
│  ┌────────────────────────┐ ┌────────────────────────┐         │
│  │    Query Services      │ │   Command Services     │         │
│  │  - Read operations     │ │  - Write operations    │         │
│  │  - Search/filter       │ │  - Create/Update/Delete│         │
│  │  - Aggregations        │ │  - Business rules      │         │
│  └────────────────────────┘ └────────────────────────┘         │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                        ENTITY LAYER                              │
│  ┌─────────────────────────────────────────────────────────────┐│
│  │                    JPA Entities                              ││
│  │  Sense, Synset, Lexicon, Word, RelationType, etc.           ││
│  │  - Database mapping                                          ││
│  │  - Relationships                                             ││
│  │  - Constraints                                               ││
│  └─────────────────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                        DATABASE                                  │
│                        (MySQL 8.0)                               │
└─────────────────────────────────────────────────────────────────┘
```

### Request Processing Flow

```
HTTP Request
     │
     ▼
┌─────────────────┐
│  JAX-RS Filter  │ ─── Authentication check
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  REST Resource  │ ─── Request parsing, validation
│   (Boundary)    │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│    Service      │ ─── Business logic
│   (Control)     │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  JPA Entity     │ ─── Database operations
│   (Entity)      │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  EntityBuilder  │ ─── JSON response building
└────────┬────────┘
         │
         ▼
HTTP Response
```

## Package Structure

```
pl.edu.pwr.wordnetloom.server/
└── business/
    │
    ├── EntityBuilder.java           # JSON response builder
    ├── LinkBuilder.java             # HATEOAS link generator
    ├── OperationResult.java         # Result wrapper
    ├── RootResource.java            # API root
    ├── SearchFilter.java            # Search parameters
    ├── SearchFilterExtractor...java # URL parameter extractor
    │
    ├── sense/
    │   ├── boundary/
    │   │   ├── SenseResource.java       # REST endpoints
    │   │   └── SenseCommandService.java # Write operations
    │   ├── control/
    │   │   └── SenseQueryService.java   # Read operations
    │   └── entity/
    │       ├── Sense.java               # JPA entity
    │       ├── SenseAttributes.java     # Extended attributes
    │       ├── SenseExample.java        # Usage examples
    │       └── SenseRelation.java       # Relations
    │
    ├── synset/
    │   ├── boundary/
    │   │   ├── SynsetResource.java
    │   │   └── SynsetCommandService.java
    │   ├── control/
    │   │   └── SynsetQueryService.java
    │   └── entity/
    │       ├── Synset.java
    │       ├── SynsetAttributes.java
    │       ├── SynsetExample.java
    │       └── SynsetRelation.java
    │
    ├── lexicon/
    │   ├── boundary/
    │   │   ├── LexiconResource.java
    │   │   └── LexiconCommandService.java
    │   ├── control/
    │   │   └── LexiconQueryService.java
    │   └── entity/
    │       └── Lexicon.java
    │
    ├── dictionary/
    │   ├── boundary/
    │   │   ├── DictionaryResource.java
    │   │   └── DictionaryCommandService.java
    │   ├── control/
    │   │   └── DictionaryQueryService.java
    │   └── entity/
    │       ├── Dictionary.java
    │       ├── Domain.java
    │       ├── PartOfSpeech.java
    │       ├── Register.java
    │       └── Status.java
    │
    ├── relationtype/
    │   ├── boundary/
    │   │   └── RelationTypeResource.java
    │   └── entity/
    │       ├── RelationType.java
    │       ├── RelationTest.java
    │       └── GlobalWordnetRelationType.java
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
    │   ├── control/
    │   │   └── GraphQueryService.java
    │   └── entity/
    │       ├── NodeExpanded.java
    │       └── NodeHidden.java
    │
    ├── corpusexample/
    │   └── boundary/
    │       └── CorpusExampleResource.java
    │
    └── localisation/
        └── control/
            └── LocalisedStringsQueryService.java
```

## REST API Layer

### Resource Classes

Resource classes handle HTTP requests and define API endpoints:

```java
@Path("/senses")
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
public class SenseResource {

    @Inject
    SenseQueryService queryService;

    @Inject
    SenseCommandService commandService;

    @Inject
    EntityBuilder entityBuilder;

    @Context
    UriInfo uriInfo;

    @GET
    @Path("{id}")
    public JsonObject sense(
            @HeaderParam("Accept-Language") Locale locale,
            @PathParam("id") UUID id) {

        return queryService.findById(id)
            .map(s -> entityBuilder.buildSense(s, locale, uriInfo))
            .orElse(Json.createObjectBuilder().build());
    }

    @POST
    public Response addSense(JsonObject sense) {
        OperationResult<Sense> result = commandService.save(sense);

        if (result.hasErrors()) {
            return Response.status(Response.Status.BAD_REQUEST)
                .entity(result.getErrors())
                .build();
        }

        return Response.created(linkBuilder.forSense(result.getEntity(), uriInfo))
            .build();
    }
}
```

### EntityBuilder

Constructs JSON responses with HATEOAS links:

```java
public class EntityBuilder {

    @Inject
    LocalisedStringsQueryService loc;

    @Inject
    LinkBuilder linkBuilder;

    public JsonObject buildSense(Sense s, URI self, Locale locale) {
        return Json.createObjectBuilder()
            .add("id", s.getId().toString())
            .add("lemma", s.getWord().getWord())
            .add("variant", s.getVariant())
            .add("domain", buildDomain(s.getDomain(), locale))
            .add("part_of_speech", buildPartOfSpeech(s.getPartOfSpeech(), locale))
            .add("_links", Json.createObjectBuilder()
                .add("self", self.toString())
                .add("synset", linkBuilder.forSynset(s.getSynset(), uriInfo).toString())
                .add("relations", linkBuilder.forSenseRelations(s, uriInfo).toString()))
            .build();
    }
}
```

### LinkBuilder

Generates URIs for HATEOAS links:

```java
public class LinkBuilder {

    public URI forSense(Sense sense, UriInfo uriInfo) {
        return uriInfo.getBaseUriBuilder()
            .path(SenseResource.class)
            .path(sense.getId().toString())
            .build();
    }

    public URI forSenseRelations(Sense sense, UriInfo uriInfo) {
        return uriInfo.getBaseUriBuilder()
            .path(SenseResource.class)
            .path(sense.getId().toString())
            .path("relations")
            .build();
    }
}
```

## Business Logic Layer

### Query Services

Handle read operations with JPA queries:

```java
public class SenseQueryService {

    @PersistenceContext
    EntityManager em;

    public Optional<Sense> findById(UUID id) {
        return Optional.ofNullable(em.find(Sense.class, id));
    }

    public List<Sense> findByFilter(SearchFilter filter) {
        CriteriaBuilder cb = em.getCriteriaBuilder();
        CriteriaQuery<Sense> cq = cb.createQuery(Sense.class);
        Root<Sense> root = cq.from(Sense.class);

        List<Predicate> predicates = new ArrayList<>();

        if (filter.getLemma() != null) {
            predicates.add(cb.like(
                root.get("word").get("word"),
                "%" + filter.getLemma() + "%"
            ));
        }

        if (filter.getLexiconId() != null) {
            predicates.add(cb.equal(
                root.get("lexicon").get("id"),
                filter.getLexiconId()
            ));
        }

        cq.where(predicates.toArray(new Predicate[0]));

        return em.createQuery(cq)
            .setFirstResult(filter.getOffset())
            .setMaxResults(filter.getLimit())
            .getResultList();
    }

    public long countWithFilter(SearchFilter filter) {
        // Similar to above but with COUNT query
    }
}
```

### Command Services

Handle write operations:

```java
public class SenseCommandService {

    @PersistenceContext
    EntityManager em;

    @Inject
    SenseQueryService queryService;

    public OperationResult<Sense> save(JsonObject json) {
        OperationResult<Sense> result = new OperationResult<>();

        // Validation
        if (!json.containsKey("lemma")) {
            result.addError("Lemma is required");
            return result;
        }

        // Create entity
        Sense sense = new Sense();
        sense.setId(UUID.randomUUID());

        // Find or create word
        Word word = findOrCreateWord(json.getString("lemma"));
        sense.setWord(word);

        // Set other properties
        sense.setLexicon(em.find(Lexicon.class, json.getJsonNumber("lexicon_id").longValue()));
        sense.setPartOfSpeech(em.find(PartOfSpeech.class, json.getJsonNumber("part_of_speech_id").longValue()));
        sense.setDomain(em.find(Domain.class, json.getJsonNumber("domain_id").longValue()));

        em.persist(sense);
        result.setEntity(sense);

        return result;
    }

    public void deleteSense(UUID id) {
        Sense sense = em.find(Sense.class, id);
        if (sense != null) {
            em.remove(sense);
        }
    }
}
```

### OperationResult

Wrapper for operation results with error handling:

```java
public class OperationResult<T> {

    private T entity;
    private List<String> errors = new ArrayList<>();

    public boolean hasErrors() {
        return !errors.isEmpty();
    }

    public void addError(String error) {
        errors.add(error);
    }

    public T getEntity() { return entity; }
    public void setEntity(T entity) { this.entity = entity; }
    public List<String> getErrors() { return errors; }
}
```

## Data Access Layer

### JPA Entities

Entities map to database tables:

```java
@Entity
@Table(name = "tbl_sense")
public class Sense {

    @Id
    @Column(columnDefinition = "BINARY(16)")
    private UUID id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "word_id")
    private Word word;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "synset_id")
    private Synset synset;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "lexicon_id", nullable = false)
    private Lexicon lexicon;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "part_of_speech_id", nullable = false)
    private PartOfSpeech partOfSpeech;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "domain_id", nullable = false)
    private Domain domain;

    @Column(name = "variant", nullable = false)
    private Integer variant = 1;

    @Column(name = "synset_position")
    private Integer synsetPosition;

    // Getters and setters
}
```

### Persistence Configuration

`persistence.xml` configures JPA:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<persistence xmlns="http://xmlns.jcp.org/xml/ns/persistence"
             version="2.2">

    <persistence-unit name="wordnet" transaction-type="JTA">
        <jta-data-source>java:jboss/datasources/wordnet</jta-data-source>

        <properties>
            <property name="hibernate.dialect"
                      value="org.hibernate.dialect.MySQL8Dialect"/>
            <property name="hibernate.show_sql" value="false"/>
            <property name="hibernate.format_sql" value="true"/>
            <property name="hibernate.hbm2ddl.auto" value="validate"/>
        </properties>
    </persistence-unit>
</persistence>
```

## Security

### JWT Authentication

`JwtManager` handles token creation and validation:

```java
public class JwtManager {

    private static final String SECRET = "wordnetloom-secret-key";
    private static final long EXPIRATION_TIME = 86400000; // 24 hours

    public String createJwt(User user) throws JOSEException {
        JWTClaimsSet claims = new JWTClaimsSet.Builder()
            .subject(user.getEmail())
            .claim("role", user.getRole())
            .claim("firstname", user.getFirstname())
            .claim("lastname", user.getLastname())
            .expirationTime(new Date(System.currentTimeMillis() + EXPIRATION_TIME))
            .build();

        JWSSigner signer = new MACSigner(SECRET);
        SignedJWT signedJWT = new SignedJWT(
            new JWSHeader(JWSAlgorithm.HS256),
            claims
        );
        signedJWT.sign(signer);

        return signedJWT.serialize();
    }

    public boolean validateToken(String token) {
        try {
            SignedJWT signedJWT = SignedJWT.parse(token);
            JWSVerifier verifier = new MACVerifier(SECRET);
            return signedJWT.verify(verifier) &&
                   new Date().before(signedJWT.getJWTClaimsSet().getExpirationTime());
        } catch (Exception e) {
            return false;
        }
    }
}
```

### SecurityResource

Handles authentication endpoints:

```java
@Path("/security")
public class SecurityResource {

    @Inject
    JwtManager jwtManager;

    @Inject
    SecurityService service;

    @POST
    @Path("/authorize")
    @Consumes(MediaType.APPLICATION_FORM_URLENCODED)
    public Response authorize(
            @FormParam("username") String username,
            @FormParam("password") String password) {

        User user = service.authenticate(username, password);

        if (user != null) {
            String token = jwtManager.createJwt(user);
            return Response.ok(new Jwt(token)).build();
        }

        return Response.status(Response.Status.UNAUTHORIZED)
            .entity(entityBuilder.buildErrorObject("Invalid credentials"))
            .build();
    }
}
```

## Database Migrations

### Flyway Configuration

Flyway runs migrations automatically on deployment. Migrations are in:
```
src/main/resources/db/migration/
└── V1_0__Schema.sql
```

### Adding Migrations

Create new migration files:

```sql
-- V1_1__Add_new_table.sql
CREATE TABLE tbl_new_entity (
    id BIGINT(20) NOT NULL AUTO_INCREMENT,
    name VARCHAR(255) NOT NULL,
    PRIMARY KEY (id)
) ENGINE=InnoDB;
```

## Extending the Server

### Adding a New Entity

1. **Create Entity Class:**

```java
@Entity
@Table(name = "tbl_new_entity")
public class NewEntity {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String name;

    // Getters/setters
}
```

2. **Create Query Service:**

```java
public class NewEntityQueryService {
    @PersistenceContext
    EntityManager em;

    public List<NewEntity> findAll() {
        return em.createQuery("SELECT e FROM NewEntity e", NewEntity.class)
            .getResultList();
    }
}
```

3. **Create Command Service:**

```java
public class NewEntityCommandService {
    @PersistenceContext
    EntityManager em;

    public OperationResult<NewEntity> save(JsonObject json) {
        // Implementation
    }
}
```

4. **Create REST Resource:**

```java
@Path("/new-entities")
@Produces(MediaType.APPLICATION_JSON)
@Consumes(MediaType.APPLICATION_JSON)
public class NewEntityResource {

    @Inject
    NewEntityQueryService queryService;

    @GET
    public JsonObject getAll() {
        // Implementation
    }
}
```

5. **Update EntityBuilder and LinkBuilder**

6. **Add Database Migration**

### Adding New Endpoints

Add methods to existing resource classes:

```java
@GET
@Path("{id}/custom-operation")
public Response customOperation(@PathParam("id") UUID id) {
    // Implementation
}
```

### Custom Validation

Add validation in command services:

```java
public OperationResult<Sense> save(JsonObject json) {
    OperationResult<Sense> result = new OperationResult<>();

    // Custom validation
    if (!isValidLemma(json.getString("lemma"))) {
        result.addError("Invalid lemma format");
    }

    if (result.hasErrors()) {
        return result;
    }

    // Continue with save
}
```

## Monitoring and Logging

### Server Logs

WildFly logs are in `$WILDFLY_HOME/standalone/log/server.log`

### Adding Logging

```java
import java.util.logging.Logger;

public class SenseQueryService {

    private static final Logger LOG = Logger.getLogger(SenseQueryService.class.getName());

    public Optional<Sense> findById(UUID id) {
        LOG.info("Finding sense by ID: " + id);
        // Implementation
    }
}
```

### Performance Monitoring

Use JPA statistics in persistence.xml:

```xml
<property name="hibernate.generate_statistics" value="true"/>
```
