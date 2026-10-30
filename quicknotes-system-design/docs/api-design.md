# QuickNotes API Design Specification

This document details the RESTful API endpoints for the production-grade QuickNotes service.

## 1. REST Endpoints Overview

| Method | Path | Description | Success Status |
|---|---|---|---|
| `GET` | `/api/v1/notes` | List paginated notes for the authenticated user | `200 OK` |
| `POST` | `/api/v1/notes` | Create a new note | `201 Created` |
| `GET` | `/api/v1/notes/{id}` | Retrieve details of a specific note | `200 OK` |
| `PUT` | `/api/v1/notes/{id}` | Update an existing note | `200 OK` |
| `DELETE` | `/api/v1/notes/{id}` | Soft-delete a note | `200 OK` / `204 No Content` |
| `GET` | `/api/v1/tags` | List all tags belonging to the user | `200 OK` |

---

## 2. Request & Response Examples

### POST `/api/v1/notes` (Create Note)

**Request Headers:**
```http
Authorization: Bearer <jwt_token>
Content-Type: application/json