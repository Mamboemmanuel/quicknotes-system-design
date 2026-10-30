### `docs/data-model.md`

```markdown
# QuickNotes Data Model Specification

This document details the relational schema, SQL DDL statements, indexing, and rationale for QuickNotes.

## 1. Entities & Schema Overview

- **`users`**: Contains user credentials and profile information.
- **`notes`**: Contains core note records linked to a specific user.
- **`tags`**: Categorization labels created by users.
- **`note_tags`**: Junction table resolving the many-to-many relationship between notes and tags.

---

## 2. Relationships

- **Users to Notes (One-to-Many):** One user can own multiple notes. Each note belongs to exactly one user (`notes.user_id -> users.id`).
- **Notes to Tags (Many-to-Many):** A note can have multiple tags, and a tag can be attached to multiple notes. Joined via `note_tags`.

---

## 3. DDL: CREATE TABLE Statements

```sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE notes (
    id SERIAL PRIMARY KEY,
    user_id INT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    title VARCHAR(100) NOT NULL,
    body TEXT,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE tags (
    id SERIAL PRIMARY KEY,
    user_id INT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    name VARCHAR(50) NOT NULL,
    UNIQUE(user_id, name)
);

CREATE TABLE note_tags (
    note_id INT NOT NULL REFERENCES notes(id) ON DELETE CASCADE,
    tag_id INT NOT NULL REFERENCES tags(id) ON DELETE CASCADE,
    PRIMARY KEY (note_id, tag_id)
);