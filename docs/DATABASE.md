# Database Schema Documentation

This document describes the MySQL database schema used by WordNetLoom.

## Table of Contents

- [Overview](#overview)
- [Entity Relationship Diagram](#entity-relationship-diagram)
- [Tables](#tables)
  - [Core Tables](#core-tables)
  - [Dictionary Tables](#dictionary-tables)
  - [Relation Tables](#relation-tables)
  - [User Tables](#user-tables)
  - [Localization Tables](#localization-tables)
- [Indexes](#indexes)
- [Data Types](#data-types)
- [Migration](#migration)

## Overview

WordNetLoom uses MySQL 8.0 with InnoDB engine. The schema is designed to support:

- Multi-language wordnets (lexicons)
- Hierarchical semantic relationships
- Flexible dictionary/classification systems
- Internationalization (i18n) for UI strings
- User management with role-based access

**Character Set:** `utf8mb4`
**Collation:** `utf8mb4_unicode_ci`

## Entity Relationship Diagram

```
┌─────────────────┐
│   tbl_lexicon   │
│─────────────────│
│ id (PK)         │
│ name            │
│ language_name   │
└────────┬────────┘
         │
         │ 1:N
         ▼
┌─────────────────┐         ┌─────────────────┐
│   tbl_synset    │         │    tbl_word     │
│─────────────────│         │─────────────────│
│ id (PK, UUID)   │         │ id (PK, UUID)   │
│ lexicon_id (FK) │         │ word            │
│ status_id (FK)  │         └────────┬────────┘
│ abstract        │                  │
└────────┬────────┘                  │
         │                           │
         │ 1:N                       │ 1:N
         ▼                           │
┌─────────────────┐                  │
│   tbl_sense     │◄─────────────────┘
│─────────────────│
│ id (PK, UUID)   │
│ synset_id (FK)  │
│ word_id (FK)    │
│ lexicon_id (FK) │
│ domain_id (FK)  │
│ pos_id (FK)     │
│ variant         │
└─────────────────┘
         │
         │ 1:1
         ▼
┌─────────────────────┐
│ tbl_sense_attributes│
│─────────────────────│
│ sense_id (PK, FK)   │
│ definition          │
│ comment             │
│ register_id (FK)    │
└─────────────────────┘

Relation Tables:
┌───────────────────────┐     ┌───────────────────────┐
│  tbl_sense_relation   │     │  tbl_synset_relation  │
│───────────────────────│     │───────────────────────│
│ parent_sense_id (FK)  │     │ parent_synset_id (FK) │
│ child_sense_id (FK)   │     │ child_synset_id (FK)  │
│ relation_type_id (FK) │     │ relation_type_id (FK) │
└───────────────────────┘     └───────────────────────┘
```

## Tables

### Core Tables

#### tbl_lexicon

Stores wordnet lexicon metadata.

| Column | Type | Nullable | Description |
|--------|------|----------|-------------|
| id | BIGINT(20) | NO | Primary key (auto-increment) |
| identifier | VARCHAR(255) | NO | Short identifier (e.g., "plwordnet") |
| name | VARCHAR(255) | NO | Full name |
| language_name | VARCHAR(255) | NO | Language (e.g., "Polish") |
| language_shortcut | VARCHAR(5) | YES | ISO code (e.g., "pl") |
| lexicon_version | VARCHAR(255) | NO | Version string |
| license | VARCHAR(255) | YES | License information |
| email | VARCHAR(255) | YES | Contact email |
| reference_url | VARCHAR(255) | YES | External reference URL |
| citation | TEXT | YES | Citation text |
| confidence_score | VARCHAR(255) | YES | Confidence scoring info |
| description | TEXT | YES | Description |
| read_only | BIT(1) | YES | If lexicon is read-only (default: 0) |

**Example:**
```sql
INSERT INTO tbl_lexicon (identifier, name, language_name, language_shortcut, lexicon_version)
VALUES ('plwordnet', 'Polish WordNet', 'Polish', 'pl', '4.0');
```

---

#### tbl_word

Stores unique word forms (lemmas).

| Column | Type | Nullable | Description |
|--------|------|----------|-------------|
| id | BINARY(16) | NO | Primary key (UUID) |
| word | VARCHAR(255) | NO | Word text (unique) |

**Indexes:**
- `word_index` on `word`
- `word_uuid_index` on `id`
- Unique constraint on `word`

---

#### tbl_synset

Stores synsets (synonym sets).

| Column | Type | Nullable | Description |
|--------|------|----------|-------------|
| id | BINARY(16) | NO | Primary key (UUID) |
| lexicon_id | BIGINT(20) | NO | Foreign key to tbl_lexicon |
| status_id | BIGINT(20) | YES | Foreign key to tbl_dictionaries |
| abstract | TINYINT(1) | YES | Whether synset is abstract |

**Foreign Keys:**
- `lexicon_id` → `tbl_lexicon(id)`

---

#### tbl_synset_attributes

Extended attributes for synsets.

| Column | Type | Nullable | Description |
|--------|------|----------|-------------|
| synset_id | BINARY(16) | NO | Primary key, FK to tbl_synset |
| definition | TEXT | YES | Synset definition |
| comment | TEXT | YES | Editorial comment |
| owner_id | BIGINT(20) | YES | FK to tbl_users (creator) |
| error_comment | TEXT | YES | Error notes |
| princeton_id | VARCHAR(255) | YES | Princeton WordNet ID |
| ili_id | VARCHAR(255) | YES | OMW Interlingual Index ID |

---

#### tbl_sense

Stores individual word senses.

| Column | Type | Nullable | Description |
|--------|------|----------|-------------|
| id | BINARY(16) | NO | Primary key (UUID) |
| synset_position | INT(11) | YES | Position within synset |
| variant | INT(11) | NO | Sense variant number (default: 1) |
| domain_id | BIGINT(20) | NO | FK to tbl_domain |
| lexicon_id | BIGINT(20) | NO | FK to tbl_lexicon |
| part_of_speech_id | BIGINT(20) | NO | FK to tbl_part_of_speech |
| status_id | BIGINT(20) | YES | FK to tbl_dictionaries |
| synset_id | BINARY(16) | YES | FK to tbl_synset |
| word_id | BINARY(16) | YES | FK to tbl_word |

**Foreign Keys:**
- `domain_id` → `tbl_domain(id)`
- `lexicon_id` → `tbl_lexicon(id)`
- `part_of_speech_id` → `tbl_part_of_speech(id)`
- `synset_id` → `tbl_synset(id)`
- `word_id` → `tbl_word(id)`

---

#### tbl_sense_attributes

Extended attributes for senses.

| Column | Type | Nullable | Description |
|--------|------|----------|-------------|
| sense_id | BINARY(16) | NO | Primary key, FK to tbl_sense |
| definition | TEXT | YES | Sense definition |
| comment | TEXT | YES | Editorial comment |
| link | VARCHAR(255) | YES | External URL |
| register_id | BIGINT(20) | YES | FK to tbl_dictionaries (register) |
| user_id | BIGINT(20) | YES | FK to tbl_users (editor) |
| error_comment | TEXT | YES | Error notes |

---

#### tbl_sense_examples

Usage examples for senses.

| Column | Type | Nullable | Description |
|--------|------|----------|-------------|
| id | BINARY(16) | NO | Primary key (UUID) |
| example | TEXT | YES | Example sentence |
| type | VARCHAR(30) | NO | Example type (e.g., "corpus") |
| sense_attribute_id | BINARY(16) | YES | FK to tbl_sense |

---

#### tbl_synset_examples

Usage examples for synsets.

| Column | Type | Nullable | Description |
|--------|------|----------|-------------|
| id | BINARY(16) | NO | Primary key (UUID) |
| example | TEXT | YES | Example sentence |
| type | VARCHAR(30) | YES | Example type |
| synset_attribute_id | BINARY(16) | YES | FK to tbl_synset |

---

### Dictionary Tables

#### tbl_dictionaries

Base table for dictionary entries (using single-table inheritance).

| Column | Type | Nullable | Description |
|--------|------|----------|-------------|
| dtype | VARCHAR(31) | NO | Discriminator (Status, Register) |
| id | BIGINT(20) | NO | Primary key (auto-increment) |
| name_id | BIGINT(20) | YES | FK to localized name |
| description_id | BIGINT(20) | YES | FK to localized description |
| tag | VARCHAR(20) | YES | Short tag |
| value | BIGINT(20) | YES | Numeric value |
| color | VARCHAR(7) | YES | Hex color code |
| is_default | BIT(1) | YES | Default selection (default: 0) |

**Foreign Keys:**
- `name_id` → `tbl_application_localised_string(id)`
- `description_id` → `tbl_application_localised_string(id)`

---

#### tbl_domain

Subject domains for classification.

| Column | Type | Nullable | Description |
|--------|------|----------|-------------|
| id | BIGINT(20) | NO | Primary key (auto-increment) |
| name_id | BIGINT(20) | YES | FK to localized name |
| description_id | BIGINT(20) | YES | FK to localized description |

---

#### tbl_part_of_speech

Parts of speech (noun, verb, etc.).

| Column | Type | Nullable | Description |
|--------|------|----------|-------------|
| id | BIGINT(20) | NO | Primary key (auto-increment) |
| name_id | BIGINT(20) | YES | FK to localized name |
| color | VARCHAR(255) | YES | Display color |

---

### Relation Tables

#### tbl_relation_type

Defines types of semantic relations.

| Column | Type | Nullable | Description |
|--------|------|----------|-------------|
| id | BINARY(16) | NO | Primary key (UUID) |
| name_id | BIGINT(20) | YES | FK to localized name |
| description_id | BIGINT(20) | YES | FK to localized description |
| display_text_id | BIGINT(20) | YES | FK to display text |
| short_display_text_id | BIGINT(20) | YES | FK to short text (for viz) |
| relation_argument | VARCHAR(255) | YES | "SENSE" or "SYNSET" |
| global_wordnet_relation_type | VARCHAR(255) | YES | Global WordNet type |
| auto_reverse | BIT(1) | NO | Auto-create reverse relation |
| multilingual | BIT(1) | NO | Cross-lexicon relation |
| color | VARCHAR(255) | YES | Display color |
| node_position | VARCHAR(255) | YES | Graph position (LEFT/TOP/RIGHT/BOTTOM) |
| priority | INT(11) | YES | Display priority |
| parent_relation_type_id | BINARY(16) | YES | FK to parent type |
| reverse_relation_type_id | BINARY(16) | YES | FK to reverse type |

---

#### tbl_sense_relation

Stores sense-to-sense relations.

| Column | Type | Nullable | Description |
|--------|------|----------|-------------|
| parent_sense_id | BINARY(16) | NO | Source sense (PK part) |
| child_sense_id | BINARY(16) | NO | Target sense (PK part) |
| sense_relation_type_id | BINARY(16) | NO | Relation type (PK part) |

**Primary Key:** Composite (parent_sense_id, child_sense_id, sense_relation_type_id)

---

#### tbl_synset_relation

Stores synset-to-synset relations.

| Column | Type | Nullable | Description |
|--------|------|----------|-------------|
| parent_synset_id | BINARY(16) | NO | Source synset (PK part) |
| child_synset_id | BINARY(16) | NO | Target synset (PK part) |
| synset_relation_type_id | BINARY(16) | NO | Relation type (PK part) |

**Primary Key:** Composite (parent_synset_id, child_synset_id, synset_relation_type_id)

---

#### tbl_relation_tests

Test definitions for relations.

| Column | Type | Nullable | Description |
|--------|------|----------|-------------|
| id | BIGINT(20) | NO | Primary key (auto-increment) |
| relation_type_id | BINARY(16) | YES | FK to relation type |
| position | INT(11) | NO | Order position (default: 0) |
| test | TEXT | YES | Test definition |
| element_A_part_of_speech_id | BIGINT(20) | YES | FK to POS for element A |
| element_B_part_of_speech_id | BIGINT(20) | YES | FK to POS for element B |

---

#### tbl_relation_type_allowed_lexicons

Which lexicons can use a relation type.

| Column | Type | Nullable | Description |
|--------|------|----------|-------------|
| lexicon_id | BIGINT(20) | NO | FK to tbl_lexicon (PK part) |
| relation_type_id | BINARY(16) | NO | FK to relation type (PK part) |

---

#### tbl_relation_type_allowed_parts_of_speech

Which parts of speech can use a relation type.

| Column | Type | Nullable | Description |
|--------|------|----------|-------------|
| part_of_speech_id | BIGINT(20) | NO | FK to POS (PK part) |
| relation_type_id | BINARY(16) | NO | FK to relation type (PK part) |

---

### User Tables

#### tbl_users

User accounts.

| Column | Type | Nullable | Description |
|--------|------|----------|-------------|
| id | BIGINT(20) | NO | Primary key (auto-increment) |
| email | VARCHAR(255) | NO | Email (unique, used as username) |
| firstname | VARCHAR(255) | NO | First name |
| lastname | VARCHAR(255) | NO | Last name |
| password | VARCHAR(64) | NO | SHA-256 hashed password |
| role | VARCHAR(255) | YES | USER or ADMIN |

---

#### tbl_users_settings

User preferences.

| Column | Type | Nullable | Description |
|--------|------|----------|-------------|
| user_id | BIGINT(20) | NO | Primary key, FK to tbl_users |
| lexicon_marker | TINYINT(1) | YES | Show lexicon markers |
| chosen_lexicons | VARCHAR(255) | NO | Comma-separated lexicon IDs |
| show_tool_tips | TINYINT(1) | YES | Show tooltips |

---

### Localization Tables

#### tbl_application_localised_string

Stores localized strings for UI and dictionary names.

| Column | Type | Nullable | Description |
|--------|------|----------|-------------|
| id | BIGINT(20) | NO | Primary key (auto-increment, composite) |
| language | VARCHAR(255) | NO | Language code (e.g., "en", "pl") |
| value | TEXT | YES | Localized text value |

**Primary Key:** Composite (id, language)

**Example:**
```sql
-- English name
INSERT INTO tbl_application_localised_string (id, language, value)
VALUES (1, 'en', 'noun');

-- Polish name
INSERT INTO tbl_application_localised_string (id, language, value)
VALUES (1, 'pl', 'rzeczownik');
```

---

### Corpus Tables

#### tbl_corpus_example

Corpus examples from external sources.

| Column | Type | Nullable | Description |
|--------|------|----------|-------------|
| id | BIGINT(20) | NO | Primary key (auto-increment) |
| word | VARCHAR(255) | YES | Word (lemma) |
| text | TEXT | YES | Example sentence |

**Indexes:**
- `word_index` on `word`

---

#### tbl_word_form

Word forms (inflections).

| Column | Type | Nullable | Description |
|--------|------|----------|-------------|
| id | BINARY(16) | NO | Primary key (UUID) |
| word | VARCHAR(255) | YES | Base word |
| form | VARCHAR(255) | YES | Inflected form |
| tag | VARCHAR(255) | YES | Morphological tag |

---

## Indexes

### Performance Indexes

```sql
-- Word lookups
CREATE INDEX word_index ON tbl_word(word);
CREATE INDEX word_uuid_index ON tbl_word(id);

-- Sense lookups
CREATE INDEX sense_uuid_index ON tbl_sense(id);

-- Synset lookups
CREATE INDEX synset_uuid_index ON tbl_synset(id);

-- Relation type lookups
CREATE INDEX relation_type_uuid_index ON tbl_relation_type(id);

-- Corpus example lookups
CREATE INDEX word_index ON tbl_corpus_example(word);
```

### Foreign Key Indexes

Foreign key constraints automatically create indexes on referencing columns.

---

## Data Types

### UUID Storage

UUIDs are stored as `BINARY(16)` for storage efficiency:

```sql
-- Converting UUID to binary
INSERT INTO tbl_synset (id, ...) VALUES (UUID_TO_BIN(UUID()), ...);

-- Converting binary to UUID string
SELECT BIN_TO_UUID(id) AS uuid_string FROM tbl_synset;
```

### Localized Strings Pattern

Localized strings use a join pattern:

```sql
-- Get part of speech name in English
SELECT pos.id, loc.value AS name
FROM tbl_part_of_speech pos
JOIN tbl_application_localised_string loc
  ON pos.name_id = loc.id AND loc.language = 'en';
```

---

## Migration

### Flyway Setup

Database migrations are managed by Flyway. Migration files are located in:

```
wordnetloom-server/src/main/resources/db/migration/
└── V1_0__Schema.sql    # Initial schema
```

### Running Migrations

Migrations run automatically on application startup. To run manually:

```bash
# Using Flyway CLI
flyway -url=jdbc:mysql://localhost:3306/wordnet \
       -user=wordnet \
       -password=password \
       -locations=filesystem:./db/migration \
       migrate
```

### Adding New Migrations

Create a new file following the naming convention:

```
V{version}__{description}.sql
```

Example: `V1_1__Add_language_table.sql`

---

## Backup and Recovery

### Full Backup

```bash
mysqldump -u root -p wordnet > wordnet_backup.sql
```

### Data-Only Backup

```bash
mysqldump -u root -p --no-create-info wordnet > wordnet_data.sql
```

### Restore

```bash
mysql -u root -p wordnet < wordnet_backup.sql
```

---

## Performance Tuning

### Recommended MySQL Settings

```ini
[mysqld]
innodb_buffer_pool_size = 1G
innodb_log_file_size = 256M
innodb_flush_log_at_trx_commit = 2
max_connections = 200
```

### Query Optimization Tips

1. Use pagination for large result sets
2. Add indexes for frequently queried columns
3. Use the `EXPLAIN` command to analyze slow queries
4. Consider read replicas for high-traffic deployments
