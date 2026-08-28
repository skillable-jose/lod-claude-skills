# Relational Database Conventions

> Rules for relational database schema design: tables, columns, indexes, constraints, and relationships. These conventions are DB-agnostic — apply them regardless of engine (SQL Server, PostgreSQL, MySQL, etc.). Engine-specific type names are noted where relevant.

---

## Table Naming

Tables use **singular PascalCase** nouns. No prefixes, suffixes, or underscores.

```
CORRECT: User, OrderItem, EventType, QueueItem
WRONG:   Users, tbl_User, order_items, TBL_ORDER_ITEM
```

Junction tables (many-to-many) are named by concatenating both entity names, in dependency or alphabetical order:

```
CORRECT: UserRole, ProductTag, QueueItemEvent
WRONG:   user_roles_mapping, tbl_UserToRole
```

Reference/lookup tables follow the same rule — no special prefix to denote them as lookups:

```
CORRECT: Stage, EventType, TargetType
WRONG:   lkp_Stage, ref_EventType, Lookup_TargetType
```

---

## Column Naming

All columns use **PascalCase**. No underscores, no type-in-name.

### Primary Keys

Pattern: `[TableName]Id`

```
CORRECT: UserId, OrderItemId, EventTypeId
WRONG:   Id, ID, user_id, pk_user
```

Use the smallest integer type appropriate for the expected row count:
- `TINYINT` (0–255) — small fixed lookup tables (e.g. status codes, event types)
- `SMALLINT` (~32K) — medium lookup tables
- `INT` (~2B) — default for most transactional tables
- `BIGINT` — high-volume tables where INT overflow is a realistic risk

Reference/lookup tables use manually assigned IDs. Transactional tables use auto-increment/identity.

### Foreign Keys

Pattern: `[ReferencedTableName]Id` — the same name as the referenced primary key.

```
CORRECT: QueueId (in QueueItem, references Queue.QueueId)
         StageId (in QueueItem, references Stage.StageId)
WRONG:   queue_fk, fk_QueueId, Queue
```

The FK column type MUST match the referenced PK type exactly.

### Boolean Columns

Pattern: `Is[State]` using the DB's boolean/bit type.

```
CORRECT: IsActive, IsDeadLetter, IsVerified
WRONG:   Active, DeadLetter, active_flag, bActive
```

### Timestamp Columns (CRITICAL)

Pattern: `[Action]At` — always use the **timezone-aware** timestamp type.

| Engine | Type |
|--------|------|
| SQL Server | `DATETIMEOFFSET` |
| PostgreSQL | `TIMESTAMPTZ` |
| MySQL 8+ | `DATETIME` stored as UTC explicitly |

```
CORRECT: CreatedAt, StageStartedAt, VisibleAfter, ProcessedAt
WRONG:   CreatedDate, created_at_utc, timestamp_created, CreateDT
```

**Timestamps have no database DEFAULT.** The application always supplies the value explicitly. Never use `DEFAULT GETUTCDATE()`, `DEFAULT NOW()`, or similar.

### String Columns

Use non-Unicode strings by default. Only use Unicode types when the data explicitly requires it (e.g. multilingual user content).

| Engine | Default | Unicode |
|--------|---------|---------|
| SQL Server | `VARCHAR(n)` | `NVARCHAR(n)` |
| PostgreSQL | `VARCHAR(n)` / `TEXT` | same (UTF-8 natively) |

Always specify a length for bounded strings. Use unbounded (`VARCHAR(MAX)`, `TEXT`) only for genuinely variable-length content like payloads or descriptions.

```
CORRECT: Name VARCHAR(100), Code VARCHAR(20), Payload VARCHAR(MAX)
WRONG:   Name VARCHAR(MAX), Name NVARCHAR(100)  -- unless Unicode is explicitly required
```

---

## Constraint Naming

| Type | Pattern | Example |
|------|---------|---------|
| Primary Key | `PK_[Table]` | `PK_QueueItem` |
| Foreign Key | `FK_[ChildTable]_[ParentTable]` | `FK_QueueItem_Queue` |
| Unique | `UQ_[Table]_[Column(s)]` | `UQ_User_Email` |
| Check | `CK_[Table]_[Rule]` | `CK_Order_TotalPositive` |

---

## Index Naming

| Type | Pattern | Example |
|------|---------|---------|
| Standard | `IX_[Table]_[Column(s)]` | `IX_QueueItem_StageId`, `IX_QueueItem_QueueId_CreatedAt` |
| Unique | `UQ_[Table]_[Column(s)]` | `UQ_User_Email` |
| Purpose-named | `IX_[Table]_[Purpose]` | `IX_QueueItem_Polling` |

List columns in the index name in key-column order. For composite indexes, include only the leading columns unless the name would become unwieldy.

Unique constraints and unique indexes are equivalent in naming — use `UQ_` prefix for both.

---

## Relationships

**Default cascade behavior: restrict.** Foreign keys prevent deletion of a parent row that has children. Do not add `ON DELETE CASCADE` unless explicitly required.

```
CORRECT (default): no cascade — FK prevents orphan deletion
CORRECT (explicit): ON DELETE CASCADE — only when child records are meaningless without parent
WRONG: always cascading deletes as a convenience
```

When a parent must be deletable, handle deletion order explicitly in application code or stored procedures — do not rely on cascade as a workaround.

---

## What NOT to Add by Default (CRITICAL)

These are only added when explicitly requested:

- **Soft delete** (`IsDeleted`, `DeletedAt`) — hard deletes are the default; soft delete is an explicit design decision
- **Optimistic concurrency** (`RowVersion`, `xmin`) — only when concurrent update conflicts are a stated concern
- **Check constraints** — enforce business rules in the application layer; only add DB-level checks when explicitly asked
- **Default values on timestamps** — the application always sets time values; no `DEFAULT NOW()` or equivalent

---

## Code Quality Checklist

Before finalising a schema:

- [ ] All tables are singular PascalCase with no prefixes
- [ ] All PKs follow `[TableName]Id` pattern with appropriate integer size
- [ ] All FK columns match the referenced PK name and type exactly
- [ ] Booleans use `Is[State]` naming
- [ ] Timestamps use timezone-aware type and have no DB default
- [ ] Strings are non-Unicode unless Unicode is explicitly required; lengths are bounded where appropriate
- [ ] All constraints and indexes follow the naming patterns above
- [ ] No cascade deletes unless explicitly required
- [ ] No audit columns, soft delete, or check constraints unless explicitly requested
