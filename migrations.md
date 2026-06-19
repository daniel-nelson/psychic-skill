# Dream Migrations - Detailed Reference

## Overview

Dream migrations use Kysely under the hood. **Migrations are always created via generators** (`g:resource`, `g:model`, `g:sti-child`, or `g:migration`) - never create migration files by hand. This reference covers the Kysely API and DreamMigrationHelpers available for editing generated migrations.

**Inviolable: never modify a migration file that has been merged into `main`.** Once merged, any change to that migration's behavior must be expressed as a new migration. Production data already reflects the original migration's effect; rewriting history would skip the new behavior entirely on machines that have already recorded the original as applied. (This restates Critical Rule #5 from `SKILL.md` because it's the foundation for everything below.)

Migrations that are still on a feature branch — i.e., not yet merged into `main` — may be freely edited in place. There is no production data to worry about, so create columns with the correct constraints (e.g., `NOT NULL`) directly; don't write backfill `UPDATE` statements for data that doesn't exist yet.

When you edit an unmerged migration in place after it has already been applied locally, run `pnpm psy db:reset` to bring the DB back in sync — `pnpm psy db:migrate` won't re-run an already-recorded migration, so your edits won't take effect. `db:reset` drops, recreates, runs all migrations from scratch, and re-syncs types in one command. (Per `console.md`, all `psy` commands default to `NODE_ENV=test`; prefix with `NODE_ENV=development` to reset the dev DB instead — but be aware of the dev-DB caveats in `console.md`.)

This applies to **any** in-place edit that changes the resulting schema — adding a column to a `createTable`, retrofitting `@SoftDelete()`, tightening a constraint, adjusting `asEnum([...])` values for an STI discriminator, anything. The single rule covers the single situation: if you edited a migration that has already been applied locally, `db:reset`. Do NOT chain a follow-up migration to "fix" an unmerged one — edit the original.

This reset is especially important after hand-editing Kysely DDL fragments such as check constraints, partial indexes, or raw `sql` expressions. TypeScript can pass while Postgres rejects the generated DDL, so a fresh `pnpm psy db:reset` is the verification step for an edited unmerged migration.

Use `git diff --name-only origin/main -- api/src/db/migrations/` to confirm a migration is still on your branch before editing.

### NOT NULL columns and defaults

When a NOT NULL column has a single obvious domain default (`'normal'`, `'pending'`, `false`, `0`), encode it at the migration layer with `col.defaultTo(value).notNull()`. This:

- Lets callers omit the field without runtime failure.
- Removes the temptation to add silent in-controller defaults that drift from "default if missing" into "default if unrecognized" — the latter is the bug shape that swallows malformed enum values from the client.

If the default is genuinely caller-dependent, leave the column NOT NULL with no default — the resulting 400/500 on a missing value is preferable to silent coercion. The rule is: encode at the DB layer or fail loudly; never default silently in caller code.

#### Adding a NOT NULL column to a table that already has rows

A NOT NULL column with no default fails the migration the moment it runs against a populated table — Postgres has nothing to put in the existing rows. When the default is caller-dependent (so you don't want a permanent DB default per the rule above), add the column *with* a default and then drop that default in a separate statement in the same migration:

```typescript
await db.schema
  .alterTable('conversation_threads')
  .addColumn('channel', sql`conversation_thread_channels_enum`, col => col.notNull().defaultTo('web'))
  .execute()

// existing rows are now backfilled with 'web'; drop the default so new rows
// must set the value intentionally in application code
await db.schema
  .alterTable('conversation_threads')
  .alterColumn('channel', col => col.dropDefault())
  .execute()
```

The temporary default backfills existing rows; dropping it afterward keeps the "no silent default in caller code" guarantee for everything written from here on. (If a permanent default genuinely fits the domain, keep it and skip the drop — that's the `col.defaultTo(value).notNull()` case above.) Column-shorthand generators omit the default by design, since they can't know whether the table is populated — this is expected generate-then-edit territory, not a generator bug.

`pnpm psy db:migrate` runs migrations then sync. If post-sync fails (e.g., a model references an old table name), the migration itself is not reverted — `db.ts` has already been regenerated from the current database state. Fix the problem and run `pnpm psy sync` to complete the process.

### Generating column-only migrations

For schema changes that just add columns (or a foreign key) to an existing table, `pnpm psy g:migration` accepts the same column shorthand as `g:resource` and `g:model` — including `BelongsTo:type` and `:optional` modifiers. See the canonical examples in [SKILL.md — Adding Properties to Existing Models](SKILL.md#adding-properties-to-existing-models). Hand-edit the generated migration only when the change isn't expressible as column shorthand (check constraints, enum alterations, custom backfill).

## DreamMigrationHelpers

`DreamMigrationHelpers` (imported from `@rvoh/dream/db`) provides convenience methods for common migration operations. **Prefer a DreamMigrationHelpers method over compound Kysely calls whenever one is available.** Consult the TSDocs on `DreamMigrationHelpers` for the full list of available methods and their signatures - methods include helpers for extensions, enum manipulation, deferrable constraints, GIN indexes, table renaming, and more.

```typescript
import { DreamMigrationHelpers } from '@rvoh/dream/db'
```

## Create Table

```typescript
export async function up(db: Kysely<any>): Promise<void> {
  await db.schema
    .createTable('places')
    .addColumn('id', 'uuid', col => col.primaryKey().defaultTo(sql`uuidv7()`))
    .addColumn('name', sql`citext`, col => col.notNull())
    .addColumn('style', sql`place_styles_enum`, col => col.notNull())
    .addColumn('sleeps', 'integer', col => col.notNull())
    .addColumn('host_id', 'uuid', col => col.references('hosts.id').onDelete('restrict').notNull())
    .addColumn('deleted_at', 'timestamp')
    .addColumn('created_at', 'timestamp', col => col.notNull())
    .addColumn('updated_at', 'timestamp', col => col.notNull())
    .execute()

  // Always create indexes for foreign keys
  await db.schema.createIndex('places_host_id').on('places').column('host_id').execute()
}

export async function down(db: Kysely<any>): Promise<void> {
  await db.schema.dropIndex('places_host_id').execute()
  await db.schema.dropTable('places').execute()
}
```

## Rename Table

Use `DreamMigrationHelpers.renameTable` instead of Kysely's `alterTable(...).renameTo(...)` — it automatically renames the associated sequence (for tables with serial primary keys) in addition to the table itself:

```typescript
export async function up(db: Kysely<any>): Promise<void> {
  await DreamMigrationHelpers.renameTable(db, 'host_places', 'host_listings')
}

export async function down(db: Kysely<any>): Promise<void> {
  await DreamMigrationHelpers.renameTable(db, 'host_listings', 'host_places')
}
```

**Full workflow for renaming a table:**

1. `pnpm psy g:migration rename-host-places-to-host-listings`
2. Edit the migration to use `DreamMigrationHelpers.renameTable(db, 'host_places', 'host_listings')`
3. Update all models referencing the old table: change `return 'host_places' as const` to `return 'host_listings' as const` in each model's `table` getter
4. `pnpm psy db:migrate`

Step 3 must happen before step 4. `db:migrate` runs migrations then sync. The sync step regenerates `db.ts` from database introspection (which now sees the new table name), then runs post-sync which boots the app and loads models. If any model's `table` getter still returns the old name, post-sync fails with `InvalidTableName`.

## Primary Key Patterns

```typescript
// UUID with uuidv7 (recommended)
.addColumn('id', 'uuid', col => col.primaryKey().defaultTo(sql`uuidv7()`))

// bigserial (auto-incrementing)
.addColumn('id', 'bigserial', col => col.primaryKey())
```

## Column Types

### Text Types
```typescript
.addColumn('name', 'varchar')                    // varchar(255) default
.addColumn('name', 'varchar(128)')               // varchar with length
.addColumn('name', sql`citext`, col => col.notNull())  // Case-insensitive (requires extension)
.addColumn('bio', 'text')                        // Unlimited text
```

### Numeric Types
```typescript
.addColumn('count', 'integer')
.addColumn('id', 'bigint')
.addColumn('price', sql`decimal(10, 2)`)         // With precision
.addColumn('rating', 'real')                     // Float
.addColumn('score', 'double precision')          // Double
```

### Date/Time Types
```typescript
.addColumn('created_at', 'timestamp', col => col.notNull())
.addColumn('birthday', 'date')
.addColumn('start_time', 'time')
.addColumn('scheduled_at', 'timestamp with time zone')
```

### Boolean
```typescript
.addColumn('active', 'boolean', col => col.notNull().defaultTo(false))
```

### JSON

Reaching for `json` or `jsonb` should be a stop-and-reconsider moment, not a default. These column types carry no schema, so Dream's column-type inference, validation surface, and Psychic's auto-derived OpenAPI request/response shapes all short-circuit at that field — the column becomes an untyped blob from the framework's perspective, and every consumer has to re-derive the shape by hand.

Before adding one, work through whether the data is really shaped like an associated model. Repeated keys with their own lifecycle (created/updated/deleted independently, queried by value, validated per-row) almost always want their own table with a HasMany / BelongsTo. A bounded set of attributes that always travel with the parent row usually wants real columns. Reach for `jsonb` only when the data is genuinely schemaless or polymorphic in a way no relational shape captures (third-party webhook payloads stored verbatim, opaque per-tenant configuration, audit snapshots), and write down in the migration *why* the relational alternative was rejected.

```typescript
.addColumn('metadata', 'jsonb', col => col.notNull().defaultTo(sql`'{}'::jsonb`))
.addColumn('config', 'jsonb')  // Optional — null is the initial state
.addColumn('settings', 'json')  // Prefer jsonb
```

### UUID
```typescript
.addColumn('uuid', 'uuid', col => col.notNull().defaultTo(sql`gen_random_uuid()`).unique())
```

### Encrypted Columns
```typescript
.addColumn('encrypted_phone', 'text', col => col.notNull())
// Model uses @deco.Encrypted() and property 'phone'
```

## Enum Types

### Create Enum
```typescript
await db.schema
  .createType('place_styles_enum')
  .asEnum(['cottage', 'cabin', 'treehouse', 'tent', 'cave'])
  .execute()

// Use in table
.addColumn('style', sql`place_styles_enum`, col => col.notNull())
```

### Enum Array
```typescript
await db.schema
  .createType('bed_types_enum')
  .asEnum(['twin', 'bunk', 'queen', 'king', 'cot', 'sofabed'])
  .execute()

// Array column with empty default
.addColumn('bed_types', sql`bed_types_enum[]`, col => col.notNull().defaultTo('{}'))
```

Array columns (enum arrays, `text[]`, `integer[]`, etc.) work seamlessly with Dream — set and read them as regular arrays. **In-place mutations (e.g. `.push()`) are not detected by dirty tracking** because the check compares array identity, not contents. Always reassign the entire array to trigger an update.

```typescript
await Kitchen.create({ appliances: ['microwave', 'stove'] })

const kitchen = await Kitchen.findOrFail(id)

// Create a new array to trigger dirty tracking:
await kitchen.update({ appliances: [...kitchen.appliances, 'dishwasher'] })
// Or the assign and save equivalent:
kitchen.appliances = [...kitchen.appliances, 'dishwasher']
await kitchen.save()
```

### Modify Enum

```typescript
// Add value (idempotent)
await DreamMigrationHelpers.addEnumValue(db, {
  enumName: 'place_styles_enum',
  value: 'mansion',
})

// Drop value (requires data migration)
await DreamMigrationHelpers.dropEnumValue(db, {
  enumName: 'place_styles_enum',
  value: 'dump',
  replacements: [
    { table: 'places', column: 'style', replaceWith: 'cave' },
  ],
})
```

### Renaming an Enum Value (Two-Migration Pattern)

PostgreSQL cannot add and use a new enum value in the same transaction, and Dream runs all pending migrations in a single transaction by default. To rename an enum value (add the new name, migrate data from old to new, drop the old name), you must use **two separate migration files**.

**Migration 1** — add the new enum value:

```bash
pnpm psy g:migration add-treehouse-to-place-styles
```

```typescript
export async function up(db: Kysely<any>): Promise<void> {
  await DreamMigrationHelpers.addEnumValue(db, {
    enumName: 'place_styles_enum',
    value: 'treehouse',
  })
}

export async function down(): Promise<void> {}
```

**Migration 2** — replace the old value with the new one and drop it:

```bash
pnpm psy g:migration replace-lean-to-with-treehouse
```

```typescript
export async function up(db: Kysely<any>): Promise<void> {
  await DreamMigrationHelpers.dropEnumValue(db, {
    enumName: 'place_styles_enum',
    value: 'lean_to',
    replacements: [
      {
        table: 'places',
        column: 'style',
        behavior: 'replace',
        replaceWith: 'treehouse',
      },
    ],
  })
}

export async function down(db: Kysely<any>): Promise<void> {
  await DreamMigrationHelpers.addEnumValue(db, {
    enumName: 'place_styles_enum',
    value: 'lean_to',
  })
}
```

Dream automatically starts a new transaction when it encounters `dropEnumValue` in a migration, ensuring the new enum value from Migration 1 is committed and visible before Migration 2 tries to use it as a replacement. If an enum is used on multiple tables/columns, add a separate entry in the `replacements` array for each.

### Forcing a New Transaction in Migrations

Dream runs all pending migrations in a single transaction by default. If a migration file contains the string `DreamMigrationHelpers.dropEnumValue` or `DreamMigrationHelpers.newTransaction()`, Dream runs that file in its own transaction, separate from the migrations before and after it.

This detection is a naive string search on the file contents — it looks for those exact strings. Do not rename `DreamMigrationHelpers` on import (e.g., `import { DreamMigrationHelpers as DMH }`) or the transaction boundary will not be detected.

```typescript
export async function up(db: Kysely<any>): Promise<void> {
  DreamMigrationHelpers.newTransaction()
  await db.schema.alterTable('places')...
}
```

## Foreign Keys

```typescript
// Basic FK
.addColumn('user_id', 'uuid', col =>
  col.references('users.id').onDelete('restrict').notNull()
)

// Optional FK
.addColumn('approver_id', 'uuid', col =>
  col.references('users.id').onDelete('set null')
)

// Unique FK (one-to-one)
.addColumn('user_id', 'uuid', col =>
  col.references('users.id').onDelete('restrict').notNull().unique()
)
```

Delete behaviors: `'restrict'`, `'cascade'`, `'no action'`, `'set null'`

**Self-referential FKs:** The generator names the column after the model (e.g., `Room:belongs_to:optional` on Room creates `room_id`). For a clearer name like `parent_id`, use the `@alias` shorthand (`Room@parent:belongs_to:optional` produces `parent_id` and `parent: Room`) — see the BelongsTo alias section below. Hand-edit the migration only when the alias shorthand can't express what you need.

**Always create indexes for foreign keys:**
```typescript
await db.schema.createIndex('posts_user_id').on('posts').column('user_id').execute()
```

## Indexes

```typescript
// Single column
await db.schema
  .createIndex('places_host_id')
  .on('places')
  .column('host_id')
  .execute()

// Multi-column
await db.schema
  .createIndex('localized_texts_localizable_for_locale')
  .on('localized_texts')
  .columns(['localizable_type', 'localizable_id', 'locale'])
  .unique()
  .execute()

// Type column for STI
await db.schema
  .createIndex('rooms_type')
  .on('rooms')
  .column('type')
  .execute()
```

## Column Modifiers

```typescript
col.primaryKey()              // Primary key
col.notNull()                 // NOT NULL constraint
col.unique()                  // Unique constraint
col.defaultTo(value)          // Default value
col.defaultTo(sql`...`)       // SQL expression default
col.references('table.id')    // Foreign key
col.onDelete('restrict')      // Delete behavior
col.check(sql`...`)          // Check constraint
```

## Alter Table

```typescript
// Add column
await db.schema
  .alterTable('rooms')
  .addColumn('bed_types', sql`bed_types_enum[]`, col => col.notNull().defaultTo('{}'))
  .execute()

// Drop column
await db.schema
  .alterTable('rooms')
  .dropColumn('bed_types')
  .execute()

// Add check constraint (for STI)
await db.schema
  .alterTable('rooms')
  .addCheckConstraint(
    'rooms_not_null_bath_or_shower_style',
    sql`type != 'Bathroom' OR bath_or_shower_style IS NOT NULL`,
  )
  .execute()
```

Fixed migration-time DDL literals belong in the emitted SQL, not in bound parameters. When a check constraint contains static constants, inline controlled literals in the DDL expression or use an appropriate raw SQL helper; parameterized DDL can pass TypeScript but fail when Postgres prepares the statement.

For partial index predicates, Kysely may require the raw predicate to be typed as SQL boolean:

```typescript
import { sql, type SqlBool } from 'kysely'

await db.schema
  .createIndex('places_active_idx')
  .on('places')
  .column('host_id')
  .where(sql<SqlBool>`deleted_at IS NULL`)
  .execute()
```

## Extensions

```typescript
// Case-insensitive text
await DreamMigrationHelpers.createExtension(db, 'citext')

// Trigram similarity for fuzzy search
await DreamMigrationHelpers.createExtension(db, 'pg_trgm')
```

## Polymorphic Association Columns

For polymorphic BelongsTo, you need a type column (enum) and an ID column:

```typescript
// Create localizable type enum
await db.schema
  .createType('localized_text_localizable_types_enum')
  .asEnum(['Host', 'Place', 'Room'])
  .execute()

await db.schema
  .createTable('localized_texts')
  .addColumn('id', 'uuid', col => col.primaryKey().defaultTo(sql`uuidv7()`))
  .addColumn('localizable_type', sql`localized_text_localizable_types_enum`, col => col.notNull())
  .addColumn('localizable_id', 'uuid', col => col.notNull())
  .addColumn('locale', sql`locales_enum`, col => col.notNull())
  .addColumn('title', 'varchar')
  .addColumn('created_at', 'timestamp', col => col.notNull())
  .addColumn('updated_at', 'timestamp', col => col.notNull())
  .execute()

// Index for polymorphic lookup
await db.schema
  .createIndex('localized_texts_localizable_for_locale')
  .on('localized_texts')
  .columns(['localizable_type', 'localizable_id', 'locale'])
  .unique()
  .execute()
```

## STI Columns

STI uses a `type` column (varchar, stores child class name) plus child-specific columns with check constraints:

```typescript
// Create base table with type column
await db.schema
  .createTable('rooms')
  .addColumn('id', 'uuid', col => col.primaryKey().defaultTo(sql`uuidv7()`))
  .addColumn('type', sql`room_types_enum`, col => col.notNull())
  .addColumn('position', 'integer', col => col.notNull())
  .addColumn('place_id', 'uuid', col => col.references('places.id').onDelete('restrict').notNull())
  .addColumn('created_at', 'timestamp', col => col.notNull())
  .addColumn('updated_at', 'timestamp', col => col.notNull())
  .execute()

// Add child-specific column with check constraint
await db.schema
  .alterTable('rooms')
  .addColumn('bath_or_shower_style', sql`bath_styles_enum`)
  .execute()

await db.schema
  .alterTable('rooms')
  .addCheckConstraint(
    'rooms_not_null_bath_or_shower_style',
    sql`type != 'Bathroom' OR bath_or_shower_style IS NOT NULL`,
  )
  .execute()
```

## Soft Delete Column

```typescript
.addColumn('deleted_at', 'timestamp')  // nullable, no default needed
```

## Sortable Column

```typescript
// Position column
.addColumn('position', 'integer', col => col.notNull())

// Deferrable unique constraint (separate migration or same)
await DreamMigrationHelpers.addDeferrableUniqueConstraint(db, 'room_position_constraint', {
  table: 'rooms',
  columns: ['place_id', 'position'],
})
```

## Generator Column Syntax

All columns default to NOT NULL. Append `:optional` to make a column nullable.

When using `pnpm psy g:model`, `pnpm psy g:resource`, or `pnpm psy g:migration`:

```
name:string                    # varchar(255)
name:string:128                # varchar(128)
name:string:optional           # varchar(255) nullable
name:citext                    # Case-insensitive text
bio:text                       # Unlimited text
count:integer                  # integer
price:decimal:10,2             # decimal(10,2)
active:boolean                 # boolean
birthday:date                  # date
start_at:datetime              # timestamp
data:jsonb                     # jsonb
tags:string[]                  # varchar(255)[]
scores:integer[]               # integer[]
status:enum:statuses:active,inactive         # Create enum + column
status:enum:statuses                          # Reuse existing enum (factory emits 'TODO')
types:enum[]:room_types:bathroom,bedroom     # Enum array
User:belongs_to                # user_id FK
User:belongs_to:optional       # nullable FK
User@created_by:belongs_to     # created_by_id FK, createdBy assoc (aliased)
Messaging/Template@template:belongs_to   # template_id FK (namespace stripped)
encrypted_phone:encrypted      # encrypted text column
```

### Aliased BelongsTo shorthand (`Model@alias:belongs_to`)

A `@alias` suffix on segment-1 of the `belongs_to` shorthand drives three identifiers from a single snake_case token:

| Identifier | Derived from | Example |
|---|---|---|
| FK column name | `${alias}_id` | `canceled_by_id` |
| Typed FK property | `${alias}Id` (camelized) | `canceledById: DreamColumn<...>` |
| BelongsTo association | `${alias}` (camelized) | `canceledBy: InternalUser` |

```bash
pnpm psy g:model Messaging/MessageRequest \
  InternalUser@canceled_by:belongs_to:optional \
  Messaging/Template@template:belongs_to
```

Generates:

```typescript
@deco.BelongsTo('InternalUser', { on: 'canceledById', optional: true })
public canceledBy: InternalUser | null
public canceledById: DreamColumn<MessageRequest, 'canceledById'>

@deco.BelongsTo('Messaging/Template', { on: 'templateId' })
public template: MessagingTemplate
public templateId: DreamColumn<MessageRequest, 'templateId'>
```

**Canonical use case** — the `_by` pattern: `User@created_by`, `User@approved_by`, `InternalUser@canceled_by`. Lets one model carry multiple FKs to the same target without collision (`Messaging/Message@last_inbound:belongs_to`, `Messaging/Message@last_outbound:belongs_to`, etc.).

**Secondary use case** — namespace strip: `Messaging/Template@template:belongs_to` produces `template: MessagingTemplate` rather than `messagingTemplate: MessagingTemplate`. Useful when the namespace is organizational rather than semantic.

The legacy form (`Model:belongs_to[:optional]`) is unchanged: with no alias, the property/column derive from the model's last namespace segment. Works in every generator that accepts `belongs_to`: `g:model`, `g:sti-child`, `g:migration`, `g:resource`.
