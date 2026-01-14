# REST API Reference

This document provides a complete reference for the WordNetLoom REST API.

## Table of Contents

- [Overview](#overview)
- [Authentication](#authentication)
- [Common Headers](#common-headers)
- [Response Format](#response-format)
- [Endpoints](#endpoints)
  - [Root](#root)
  - [Security](#security)
  - [Lexicons](#lexicons)
  - [Senses](#senses)
  - [Synsets](#synsets)
  - [Dictionaries](#dictionaries)
  - [Relation Types](#relation-types)
  - [Users](#users)
  - [Corpus Examples](#corpus-examples)
- [Error Handling](#error-handling)
- [Pagination](#pagination)

## Overview

The WordNetLoom API is a RESTful service that follows HATEOAS principles. All responses include navigation links to related resources.

**Base URL:** `http://{host}:{port}/wordnetloom-server/resources`

**Content Type:** All requests and responses use `application/json`

## Authentication

The API uses JWT (JSON Web Token) Bearer authentication.

### Obtaining a Token

```http
POST /security/authorize
Content-Type: application/x-www-form-urlencoded

username=user@example.com&password=yourpassword
```

**Response:**
```json
{
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}
```

### Using the Token

Include the token in the `Authorization` header:

```http
GET /senses/search
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

## Common Headers

| Header | Description | Required |
|--------|-------------|----------|
| `Authorization` | Bearer token for authentication | For protected endpoints |
| `Accept-Language` | Locale for localized responses (e.g., `en`, `pl`) | Optional |
| `Content-Type` | `application/json` for POST/PUT requests | For request body |

## Response Format

### Success Response

All responses follow a consistent structure with HATEOAS links:

```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "property": "value",
  "_links": {
    "self": "http://host/resources/entity/550e8400-e29b-41d4-a716-446655440000",
    "related": "http://host/resources/related"
  }
}
```

### Collection Response

```json
{
  "rows": [...],
  "count": 100,
  "_links": {
    "self": "http://host/resources/entity?page=1",
    "next": "http://host/resources/entity?page=2",
    "prev": "http://host/resources/entity?page=0"
  }
}
```

### Error Response

```json
{
  "code": 400,
  "message": "Description of the error"
}
```

---

## Endpoints

### Root

#### GET /

Returns API navigation links.

**Response:**
```json
{
  "_links": {
    "security": "http://localhost:8080/wordnetloom-server/resources/security",
    "corpus_examples": "http://localhost:8080/wordnetloom-server/resources/corpus-examples",
    "lexicons": "http://localhost:8080/wordnetloom-server/resources/lexicons",
    "dictionaries": "http://localhost:8080/wordnetloom-server/resources/dictionaries",
    "relation_types": "http://localhost:8080/wordnetloom-server/resources/relation-types",
    "senses": "http://localhost:8080/wordnetloom-server/resources/senses",
    "synsets": "http://localhost:8080/wordnetloom-server/resources/synsets"
  }
}
```

---

### Security

#### GET /security

Returns security-related actions.

#### POST /security/authorize

Authenticate and obtain JWT token.

**Request (form-urlencoded):**
| Field | Type | Required | Description |
|-------|------|----------|-------------|
| username | string | Yes | User email |
| password | string | Yes | User password |

**Response (200 OK):**
```json
{
  "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}
```

**Response (401 Unauthorized):**
```json
{
  "code": 400,
  "message": "Username and/or password is incorrect"
}
```

#### GET /security/claims

Get claims from current JWT token.

**Headers:** `Authorization: Bearer <token>`

**Response:**
```json
{
  "sub": "user@example.com",
  "role": "USER",
  "exp": 1640000000000
}
```

#### PUT /security/user

Update current user's profile.

**Headers:** `Authorization: Bearer <token>`

**Request Body:**
```json
{
  "firstname": "John",
  "lastname": "Doe"
}
```

#### PUT /security/change-password

Change current user's password.

**Headers:** `Authorization: Bearer <token>`

**Request Body:**
```json
{
  "password": "newSecurePassword123"
}
```

---

### Lexicons

#### GET /lexicons

List all lexicons.

**Response:**
```json
{
  "rows": [
    {
      "id": 1,
      "identifier": "plwordnet",
      "name": "Polish WordNet",
      "language_name": "Polish",
      "language_shortcut": "pl",
      "lexicon_version": "4.0",
      "_links": {
        "self": "http://host/resources/lexicons/1"
      }
    }
  ],
  "_links": {
    "self": "http://host/resources/lexicons"
  }
}
```

#### GET /lexicons/{id}

Get a specific lexicon.

**Parameters:**
| Name | Type | Description |
|------|------|-------------|
| id | integer | Lexicon ID |

**Response:**
```json
{
  "id": 1,
  "identifier": "plwordnet",
  "name": "Polish WordNet",
  "language_name": "Polish",
  "language_shortcut": "pl",
  "lexicon_version": "4.0",
  "license": "CC BY-SA 4.0",
  "email": "contact@example.com",
  "citation": "Citation text...",
  "_links": {
    "self": "http://host/resources/lexicons/1"
  }
}
```

#### POST /lexicons

Create a new lexicon.

**Request Body:**
```json
{
  "identifier": "enwordnet",
  "name": "English WordNet",
  "language_name": "English",
  "language_shortcut": "en",
  "lexicon_version": "1.0"
}
```

#### PUT /lexicons/{id}

Update a lexicon.

---

### Senses

#### GET /senses

Returns available sense actions.

#### GET /senses/search

Search for senses with filters.

**Query Parameters:**
| Parameter | Type | Description |
|-----------|------|-------------|
| lemma | string | Word to search for |
| lexicon | integer | Lexicon ID |
| part_of_speech | integer | Part of speech ID |
| domain | integer | Domain ID |
| status | integer | Status ID |
| register | integer | Register ID |
| page | integer | Page number (0-indexed) |
| limit | integer | Results per page (default: 20) |

**Example:**
```http
GET /senses/search?lemma=dog&lexicon=1&part_of_speech=2&page=0&limit=10
```

**Response:**
```json
{
  "rows": [
    {
      "id": "550e8400-e29b-41d4-a716-446655440000",
      "lemma": "dog",
      "variant": 1,
      "label": "dog 1",
      "domain": "common",
      "part_of_speech": "noun",
      "_links": {
        "self": "http://host/resources/senses/550e8400-e29b-41d4-a716-446655440000",
        "synset": "http://host/resources/synsets/..."
      }
    }
  ],
  "count": 5,
  "_links": {
    "self": "http://host/resources/senses/search?lemma=dog&page=0",
    "next": "http://host/resources/senses/search?lemma=dog&page=1"
  }
}
```

#### GET /senses/{id}

Get a specific sense.

**Response:**
```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "lemma": "dog",
  "variant": 1,
  "synset_position": 0,
  "definition": "A domesticated carnivorous mammal",
  "comment": "Most common meaning",
  "link": "http://external-resource.com",
  "domain": {
    "id": 1,
    "name": "animal"
  },
  "part_of_speech": {
    "id": 2,
    "name": "noun"
  },
  "status": {
    "id": 1,
    "name": "verified"
  },
  "lexicon": {
    "id": 1,
    "name": "English WordNet"
  },
  "_links": {
    "self": "http://host/resources/senses/550e8400-e29b-41d4-a716-446655440000",
    "synset": "http://host/resources/synsets/...",
    "relations": "http://host/resources/senses/550e8400-e29b-41d4-a716-446655440000/relations",
    "examples": "http://host/resources/senses/550e8400-e29b-41d4-a716-446655440000/examples"
  }
}
```

#### POST /senses

Create a new sense.

**Request Body:**
```json
{
  "lemma": "cat",
  "lexicon_id": 1,
  "part_of_speech_id": 2,
  "domain_id": 1,
  "status_id": 1,
  "definition": "A small domesticated carnivorous mammal"
}
```

**Response (201 Created):**
```
Location: http://host/resources/senses/{new-id}
```

#### PUT /senses/{id}

Update a sense.

#### DELETE /senses/{id}

Delete a sense.

#### PUT /senses/{id}/move-up

Move sense up in synset order.

#### PUT /senses/{id}/move-down

Move sense down in synset order.

#### PUT /senses/{id}/detach-synset

Detach sense from its synset.

#### PUT /senses/{id}/attach-to-synset/{synsetId}

Attach sense to a different synset.

#### GET /senses/{id}/graph

Get sense relationship graph for visualization.

**Response:**
```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "label": "dog 1",
  "pos": "noun",
  "top": [...],
  "bottom": [...],
  "left": [...],
  "right": [...]
}
```

#### GET /senses/{id}/examples

List examples for a sense.

#### POST /senses/{senseId}/examples

Add an example to a sense.

**Request Body:**
```json
{
  "example": "The dog barked loudly.",
  "type": "corpus"
}
```

#### PUT /senses/{senseId}/examples/{exampleId}

Update an example.

#### DELETE /senses/{senseId}/examples/{exampleId}

Delete an example.

#### GET /senses/{id}/relations

Get relations for a sense.

#### GET /senses/relations

Returns sense relation actions.

#### POST /senses/relations

Create a sense relation.

**Request Body:**
```json
{
  "source": "550e8400-e29b-41d4-a716-446655440000",
  "target": "660e8400-e29b-41d4-a716-446655440001",
  "relation_type": "770e8400-e29b-41d4-a716-446655440002"
}
```

#### GET /senses/relations/search

Search sense relations.

**Query Parameters:**
| Parameter | Type | Description |
|-----------|------|-------------|
| source | UUID | Source sense ID |
| target | UUID | Target sense ID |
| relation_type | UUID | Relation type ID |

#### DELETE /senses/relations/{source}/{relationType}/{target}

Delete a sense relation.

---

### Synsets

#### GET /synsets

Returns available synset actions.

#### GET /synsets/search

Search for synsets.

**Query Parameters:**
| Parameter | Type | Description |
|-----------|------|-------------|
| lemma | string | Word in synset |
| lexicon | integer | Lexicon ID |
| part_of_speech | integer | Part of speech ID |
| domain | integer | Domain ID |
| definition | string | Definition text |
| page | integer | Page number |
| limit | integer | Results per page |

#### GET /synsets/{id}

Get a specific synset.

**Response:**
```json
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "lexicon": {
    "id": 1,
    "name": "English WordNet"
  },
  "definition": "A group of synonymous words",
  "abstract": false,
  "status": {
    "id": 1,
    "name": "verified"
  },
  "senses": [
    {
      "id": "...",
      "lemma": "dog",
      "variant": 1
    }
  ],
  "_links": {
    "self": "http://host/resources/synsets/550e8400-e29b-41d4-a716-446655440000",
    "relations": "http://host/resources/synsets/550e8400-e29b-41d4-a716-446655440000/relations",
    "graph": "http://host/resources/synsets/550e8400-e29b-41d4-a716-446655440000/graph"
  }
}
```

#### POST /synsets

Create a new synset.

#### PUT /synsets/{id}

Update a synset.

#### DELETE /synsets/{id}

Delete a synset.

#### PUT /synsets/add-sense-to-new-synset/{senseId}

Create a new synset and add the specified sense to it.

#### GET /synsets/{id}/graph

Get synset relationship graph.

#### GET /synsets/{id}/path-to-hyperonymy

Get path from synset to root hypernym.

#### GET /synsets/{id}/relations

Get relations for a synset.

#### GET /synsets/relations

Returns synset relation actions.

#### POST /synsets/relations

Create a synset relation.

**Request Body:**
```json
{
  "source": "550e8400-e29b-41d4-a716-446655440000",
  "target": "660e8400-e29b-41d4-a716-446655440001",
  "relation_type": "770e8400-e29b-41d4-a716-446655440002"
}
```

#### GET /synsets/relations/search

Search synset relations.

#### DELETE /synsets/relations/{source}/{relationType}/{target}

Delete a synset relation.

#### GET /synsets/{synsetId}/examples

List examples for a synset.

#### POST /synsets/{synsetId}/examples

Add an example to a synset.

#### PUT /synsets/{synsetId}/examples/{exampleId}

Update an example.

#### DELETE /synsets/{synsetId}/examples/{exampleId}

Delete an example.

---

### Dictionaries

#### GET /dictionaries

Get dictionary navigation links.

**Response:**
```json
{
  "_links": {
    "domains": "http://host/resources/dictionaries/domains",
    "parts_of_speech": "http://host/resources/dictionaries/parts-of-speech",
    "statuses": "http://host/resources/dictionaries/statuses",
    "registers": "http://host/resources/dictionaries/registers"
  }
}
```

#### GET /dictionaries/domains

List all domains.

#### GET /dictionaries/domains/{id}

Get a specific domain.

#### GET /dictionaries/parts-of-speech

List all parts of speech.

#### GET /dictionaries/parts-of-speech/{id}

Get a specific part of speech.

#### GET /dictionaries/statuses

List all statuses.

#### GET /dictionaries/statuses/{id}

Get a specific status.

#### POST /dictionaries/statuses

Create a new status.

#### PUT /dictionaries/statuses/{id}

Update a status.

#### GET /dictionaries/registers

List all registers.

#### GET /dictionaries/registers/{id}

Get a specific register.

#### POST /dictionaries/registers

Create a new register.

#### PUT /dictionaries/registers/{id}

Update a register.

---

### Relation Types

#### GET /relation-types

List all relation types.

**Response:**
```json
{
  "rows": [
    {
      "id": "550e8400-e29b-41d4-a716-446655440000",
      "name": "hypernym",
      "display_text": "is a kind of",
      "short_display_text": "hyper",
      "description": "A word with a broader meaning",
      "relation_argument": "SYNSET",
      "auto_reverse": true,
      "multilingual": false,
      "color": "#FF0000",
      "_links": {
        "self": "http://host/resources/relation-types/550e8400-e29b-41d4-a716-446655440000",
        "reverse": "http://host/resources/relation-types/..."
      }
    }
  ]
}
```

---

### Users

#### GET /users

List all users (Admin only).

#### GET /users/{id}

Get a specific user.

#### POST /users

Create a new user (Admin only).

**Request Body:**
```json
{
  "email": "newuser@example.com",
  "firstname": "Jane",
  "lastname": "Doe",
  "password": "securePassword123",
  "role": "USER"
}
```

#### DELETE /users/{id}

Delete a user (Admin only).

---

### Corpus Examples

#### GET /corpus-examples/search

Search corpus examples.

**Query Parameters:**
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| lemma | string | Yes | Word to search for |

**Response:**
```json
{
  "word": "dog",
  "rows": [
    "The dog ran across the field.",
    "My dog is very friendly.",
    "Dogs are known for their loyalty."
  ],
  "_links": {
    "self": "http://host/resources/corpus-examples/search?lemma=dog"
  }
}
```

---

## Error Handling

### HTTP Status Codes

| Code | Meaning |
|------|---------|
| 200 | OK - Request successful |
| 201 | Created - Resource created successfully |
| 204 | No Content - Request successful, no response body |
| 400 | Bad Request - Invalid input |
| 401 | Unauthorized - Authentication required |
| 403 | Forbidden - Insufficient permissions |
| 404 | Not Found - Resource does not exist |
| 500 | Internal Server Error |
| 501 | Not Implemented - Feature not available |

### Error Response Format

```json
{
  "code": 400,
  "message": "Descriptive error message"
}
```

### Validation Errors

For validation failures, errors are returned as an array:

```json
[
  "Lemma is required",
  "Lexicon ID is invalid"
]
```

---

## Pagination

List endpoints support pagination with these parameters:

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| page | integer | 0 | Page number (0-indexed) |
| limit | integer | 20 | Items per page |

### Pagination Links

Paginated responses include navigation links:

```json
{
  "rows": [...],
  "count": 150,
  "_links": {
    "self": "http://host/resources/senses/search?page=1&limit=20",
    "first": "http://host/resources/senses/search?page=0&limit=20",
    "prev": "http://host/resources/senses/search?page=0&limit=20",
    "next": "http://host/resources/senses/search?page=2&limit=20",
    "last": "http://host/resources/senses/search?page=7&limit=20"
  }
}
```

---

## Rate Limiting

Currently, the API does not implement rate limiting. For production deployments, consider implementing rate limiting at the reverse proxy level.

---

## API Versioning

The current API version is v1 (implicit). Future versions may be introduced via URL path (`/v2/`) or header-based versioning.
