---
name: Prisma schema location
description: Where to find the database schema for this project
type: reference
---

Database schema is at `prisma/schema.prisma`. Reference it to understand data structures.

Current models:
- `User` — id (cuid), email (unique), password (hashed), timestamps, → projects[]
- `Project` — id (cuid), name, userId? (nullable, cascade delete), messages (JSON string, default "[]"), data (JSON string, default "{}"), timestamps