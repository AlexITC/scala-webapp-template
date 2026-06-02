# Data & infrastructure guidelines

## Rules

**RULE-DB-001 — Anorm raw SQL with explicit row parsers**

All DB access uses Anorm. No ORM. Queries are raw SQL strings; results are mapped via `Macro.parser[CaseClass]()` or hand-written `RowParser`s. Custom column parsers in `daos/package.scala` handle CITEXT, enums, and JSON.

```scala
// Read
SQL"""SELECT id, name, email FROM users WHERE id = $userId"""
  .as(Macro.parser[User]("id", "name", "email").single)

// Write
SQL"""INSERT INTO users (id, name, email) VALUES ($id, $name, $email)""".execute()
```

Use `database.withConnection { implicit conn => ... }` for reads.  
Use `database.withTransaction { implicit conn => ... }` for writes and multi-step operations.

@file: server/src/main/scala/net/wiringbits/repositories/daos/UsersDAO.scala  
@file: server/src/main/scala/net/wiringbits/repositories/daos/package.scala

**RULE-DB-002 — User-scoped queries on every user-owned table**

Every query on a user-owned table must include `WHERE user_id = $userId`. Never trust a client-supplied ID without verifying ownership.

**RULE-DB-003 — DatabaseExecutionContext**

All repository calls run on `DatabaseExecutionContext` (fixed-size pool sized to match HikariCP's 9 connections). Never run DB operations on the default `ExecutionContext`.

@file: server/src/main/scala/net/wiringbits/repositories/UsersRepository.scala

---

## Migration conventions (Play Evolutions)

Migrations live in `server/src/main/resources/evolutions/default/` numbered sequentially (`1.sql`, `2.sql`, …). Auto-apply is on in all environments. Every file has only a `-- !Ups` section — no `-- !Downs` blocks, ever.

**RULE-DB-004 — Constraint naming**

| Type | Pattern | Example |
|---|---|---|
| Primary key | `{table}_{col}_pk` | `CONSTRAINT users_user_id_pk PRIMARY KEY (user_id)` |
| Unique | `{table}_{col}_unique` | `CONSTRAINT users_email_unique UNIQUE (email)` |
| Foreign key | `{table}_{reftable}_fk` | `CONSTRAINT user_logs_users_fk FOREIGN KEY (user_id) REFERENCES users(user_id)` |
| Index | `{table}_{col}_index` | `CREATE INDEX user_logs_user_id_index ON user_logs USING BTREE (user_id)` |

Always name constraints explicitly. Anonymous constraints cannot be referenced in error messages or future migrations.

@file: server/src/main/resources/evolutions/default/1.sql

**RULE-DB-005 — Foreign key indexes**

Every foreign key column must have a corresponding `CREATE INDEX … USING BTREE` statement immediately after the `CREATE TABLE` block. This prevents full table scans when the referenced row is deleted.

```sql
CREATE TABLE user_logs (
    user_log_id UUID NOT NULL,
    user_id     UUID NOT NULL,
    CONSTRAINT user_logs_pk        PRIMARY KEY (user_log_id),
    CONSTRAINT user_logs_users_fk  FOREIGN KEY (user_id) REFERENCES users(user_id)
);
CREATE INDEX user_logs_user_id_index ON user_logs USING BTREE (user_id);
```

@file: server/src/main/resources/evolutions/default/1.sql (lines 26–30)

**RULE-DB-006 — Primary keys are single-column server-generated UUIDs**

No compound primary keys. No `SERIAL` / `BIGSERIAL`. No `DEFAULT gen_random_uuid()`. The application generates the UUID before calling the repository and passes it in.

```sql
-- Column definition — no DEFAULT
user_id UUID NOT NULL,
CONSTRAINT users_user_id_pk PRIMARY KEY (user_id)
```

```scala
// Application generates the ID
val userId = UUID.randomUUID()
usersDAO.create(userId, name, email, ...)
```

**RULE-DB-007 — Timestamps are always `TIMESTAMPTZ`**

All timestamp columns use `TIMESTAMPTZ` (timestamp with time zone), never bare `TIMESTAMP`.

| Use | Definition |
|---|---|
| Creation time | `created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()` |
| Mutation time | `updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()` |
| Scheduled time | `execute_at TIMESTAMPTZ NOT NULL DEFAULT NOW()` |
| Optional domain timestamp | `verified_on TIMESTAMPTZ NULL` |

`DEFAULT NOW()` is allowed on timestamp columns. Avoid `DEFAULT` on business-logic columns (status, name, counts, etc.) — application code should be explicit about initial values.  
Exception: `error_count INT DEFAULT 0` is acceptable because zero is a meaningful invariant, not a hidden default.

@file: server/src/main/resources/evolutions/default/1.sql (lines 12–13, 25, 57)

**RULE-DB-008 — Enums stored as TEXT with a comment**

Do not create SQL `ENUM` types. Store domain enumerations as `TEXT NOT NULL` and document the allowed values with a comment.

```sql
status TEXT NOT NULL, -- pending/success/failed
type   TEXT NOT NULL, -- EmailVerification/PasswordReset
```

Map to Scala sealed traits in the repository layer; the DAO's `package.scala` contains the `ColumnParser` that validates the string.

@file: server/src/main/resources/evolutions/default/1.sql (lines 35, 52)  
@file: server/src/main/scala/net/wiringbits/repositories/daos/package.scala

**RULE-DB-009 — CITEXT only for email**

`CITEXT` (case-insensitive text) is used solely for the `email` column so that uniqueness and lookup are case-insensitive. Do not apply it to other columns.

```sql
email CITEXT NOT NULL,
CONSTRAINT users_email_unique UNIQUE (email)
```

**RULE-DB-010 — No CHECK constraints**

Business-rule validation belongs in the application layer (validators / validated types). Do not add `CHECK` constraints to the schema.

**RULE-DB-011 — Structured payloads use JSONB**

When a row must carry variable structure (e.g. background job payloads), use `JSONB NOT NULL`.

```sql
payload JSONB NOT NULL,
```

@file: server/src/main/resources/evolutions/default/2.sql (line 8)

**RULE-DB-012 — DROP TABLE goes in `-- !Ups`**

When a table is superseded, drop it in the next sequential migration's `-- !Ups` block. Do not try to undo it with a `-- !Downs`.

@file: server/src/main/resources/evolutions/default/2.sql (DROP TABLE user_notifications)

---

## Infrastructure

Deployment is Ansible-based — no Docker Compose in production.

| Playbook | Purpose |
|---|---|
| `infra/server.yml` | Deploy Play backend |
| `infra/web.yml` | Deploy Scala.js frontend |
| `infra/nginx.yml` | Nginx reverse proxy + SSL |

Target OS: Ubuntu 20.04+. Config template: `infra/config/server/dev.env.j2`.

For local development, run PostgreSQL directly. Docker is only needed to run the integration test suite (testcontainers spins up PostgreSQL automatically).

@file: infra/README.md

## Anti-patterns

See [antipatterns.md](antipatterns.md) §Data.
