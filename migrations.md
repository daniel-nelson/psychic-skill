# Dream Migrations - Detailed Reference

## Overview

Dream migrations use Kysely under the hood. **Migrations are always created via generators** (`g:resource`, `g:model`, `g:sti-child`, or `g:migration`) - never create migration files by hand. This reference covers the Kysely API and DreamMigrationHelpers available for editing generated migrations.

**Inviolable: never modify a migration file that has been merged into `main`.** Once merged, any change to that migration's behavior must be expressed as a new migration. Production data already reflects the original migration's effect; rewriting history would skip the new behavior entirely on machines that have already recorded the original as applied. (This restates Critical Rule #6 from `SKILL.md` because it's the foundation for everything below.)

Migrations that are still on a feature branch — i.e., not yet merged into `main` — may be freely edited in place. There is no production data to worry about, so create columns with the correct constraints (e.g., `NOT NULL`) directly; don't write backfill `UPDATE` statements for data that doesn't exist yet.

When you edit an unmerged migration in place after it has already been applied locally, run `pnpm psy db:reset` to bring the DB back in sync — `pnpm psy db:migrate` won't re-run an already-recorded migration, so your edits won't take effect. `db:reset` drops, recreates, runs all migrations from scratch, and re-syncs types in one command. (Per `console.md`, all `psy` commands default to `NODE_ENV=test`; prefix with `NODE_ENV=development` to reset the dev DB instead — but that prefix skips the type-sync step, so regenerating types means running `pnpm psy db:reset` under the test default as well, and be aware of the dev-DB caveats in `console.md`.)

This applies to **any** in-place edit that changes the resulting schema — adding a column to a `createTable`, retrofitting `@SoftDelete()`, tightening a constraint, adjusting `asEnum([...])` values for an STI discriminator, anything. The single rule covers the single situation: if you edited a migration that has already been applied locally, `db:reset`. Do NOT chain a follow-up migration to "fix" an unmerged one — edit the original.

This reset is especially important after hand-editing Kysely DDL fragments such as check constraints, partial indexes, or raw `sql` expressions. TypeScript can pass while Postgres rejects the generated DDL, so a fresh `pnpm psy db:reset` is the verification step for an edited unmerged migration.

Use `git diff --name-only origin/main -- api/src/db/migrations/` to confirm a migration is still on your branch before editing.

Migrations `main` brings in may sort between your branch's; `pnpm psy db:migrate` still runs every pending one. Never rename a migration to reorder it: a database that already ran it (the user's dev DB, a deployed branch) keeps the old name recorded and fails `db:migrate` with `corrupted migrations`. That error means the database recorded a migration whose file is missing, usually after switching branches; on the test DB, the fix is `pnpm psy db:reset`.

Every `pnpm psy` command boots the full app, importing all models, before any type-regeneration step — so a stale `src/types/db.ts` crashes the CLI at import, before even `pnpm psy db:reset` or `pnpm psy sync` can reach the fix. Recover with `git checkout HEAD -- src/types/db.ts src/types/dream.ts`, then `pnpm psy db:reset` (or `sync`) to regenerate from a clean baseline. This only helps when `db.ts`/`dream.ts` themselves regressed — overwritten by a fresh scaffold's version, for example. If the real cause is a model that depends on a migration that never ran, `HEAD`'s version is equally stale and the checkout is a no-op — run the missing migration (or revert the model edit) first, then sync.

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

The temporary default backfills existing rows; dropping it afterward keeps the "no silent default in caller code" guarantee for everything written from here on. (If a permanent default genuinely fits the domain, keep it and skip the drop — that's the `col.defaultTo(value).notNull()` case above.) Column shorthand supplies a default on its own only for non-optional `boolean` (`false`), array (`'{}'`) and `json`/`jsonb` (`'{}'`) columns, so adding any other `NOT NULL` column to a populated table means hand-editing the generated migration as above.

`pnpm psy db:migrate` runs migrations, then syncs types **only when `NODE_ENV=test`** — the default for `psy` commands. `db:rollback` carries the same guard. Generated types are only ever built from the **test** database — `pnpm psy sync` is a no-op under any other `NODE_ENV`, and no standalone sync can read a development database. Migrating development therefore does not update `src/types/`; run `pnpm psy db:migrate` under the default test environment as well, which migrates the test database and regenerates the types. If post-sync fails (e.g., a model references an old table name), Dream reverts `db.ts`, `dream.ts`, and any other generated type files that already existed before this sync began back to their pre-sync content. The migration itself and the database schema it applied are unaffected by this revert — the migration is recorded and its schema change is committed before sync ever starts, so there's no need to `db:rollback` on the assumption the ledger is out of sync. Fix the problem and run `pnpm psy sync` to regenerate the types.

### Generating column-only migrations

For schema changes that just add columns (or a foreign key) to an existing table, `pnpm psy g:migration` accepts the same column shorthand as `g:resource` and `g:model` — including `Model:belongs_to` and `:optional` modifiers. See the canonical examples in [generators.md — Adding properties to an existing model](generators.md#adding-properties-to-an-existing-model). Hand-edit the generated migration for what column shorthand can't express (check constraints, enum alterations, custom backfill).

A hand-written migration's name must contain neither `-to-` nor `-from-` anywhere: either marker resolves the text after it into a real table name — a `-to-` anywhere in the name wins even when a `-from-` appears earlier, and `-from-` applies only when there's no `-to-` at all — and once the table name is resolved rather than left as a `<table-name>` placeholder, the generator fails when no columns are passed.

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
  await db.schema.dropTable('places').execute()
}
```

## Rename Table

Use `DreamMigrationHelpers.renameTable` instead of Kysely's `alterTable(...).renameTo(...)` — it also renames `<table>_pkey` and, for an identity or serial primary key, `<table>_id_seq`, which Postgres leaves under the old table's name:

```typescript
export async function up(db: Kysely<any>): Promise<void> {
  await DreamMigrationHelpers.renameTable(db, 'host_places', 'host_listings')
}

export async function down(db: Kysely<any>): Promise<void> {
  await DreamMigrationHelpers.renameTable(db, 'host_listings', 'host_places')
}
```

**Full workflow for renaming a table:**

1. `pnpm psy g:migration rename-host-places-table`
2. Edit the migration to use `DreamMigrationHelpers.renameTable(db, 'host_places', 'host_listings')`
3. Update all models referencing the old table: change `return 'host_places' as const` to `return 'host_listings' as const` in each model's `table` getter
4. `pnpm psy db:migrate`

Step 3 must happen before step 4. `db:migrate` runs migrations then sync (under `NODE_ENV=test`; see above). The sync step regenerates `db.ts` from database introspection (which now sees the new table name), then runs post-sync which boots the app and loads models. If any model's `table` getter still returns the old name, post-sync fails with `InvalidTableName`.

## Primary Key Patterns

```typescript
// UUID with uuidv7 (recommended)
.addColumn('id', 'uuid', col => col.primaryKey().defaultTo(sql`uuidv7()`))

// bigint identity (auto-incrementing)
.addColumn('id', 'bigint', col => col.primaryKey().generatedByDefaultAsIdentity())
```

## Column Types

### Text Types
```typescript
.addColumn('name', 'varchar(255)')               // what name:string generates
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
.addColumn('scheduled_at', 'timestamptz')
```

### Boolean
```typescript
.addColumn('active', 'boolean', col => col.notNull().defaultTo(false))
```

### JSON

Reaching for `json` or `jsonb` should be a stop-and-reconsider moment, not a default. These column types carry no schema, so Dream's column-type inference, validation surface, and Psychic's model-derived OpenAPI request/response shapes all short-circuit at that field — the column becomes an untyped blob from the framework's perspective, and every consumer has to re-derive the shape by hand.

Reach for `jsonb` only when the data is genuinely schemaless or polymorphic in a way no relational shape captures — third-party webhook payloads stored verbatim, opaque per-tenant configuration, audit snapshots — and write down in the migration *why* the relational alternative was rejected.

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

#### Converting an existing column to or from `@Encrypted`

Do not convert an existing plaintext column to `@Encrypted` by only renaming
`phone` to `encrypted_phone` and adding `@deco.Encrypted()` to the model. A rename
moves the existing plaintext bytes unchanged; the encrypted-column getter then tries
to decrypt plaintext and raises a `DecryptionError`.

Use `DreamMigrationHelpers.encryptColumn` / `decryptColumn` so the migration renames
the column and rewrites every non-null value through Dream's encryption path:

```typescript
import { DreamMigrationHelpers } from '@rvoh/dream/db'

export async function up(db: Kysely<any>): Promise<void> {
  await DreamMigrationHelpers.encryptColumn(db, { table: 'users', column: 'phone' })
}

export async function down(db: Kysely<any>): Promise<void> {
  await DreamMigrationHelpers.decryptColumn(db, {
    table: 'users',
    column: 'phone',
    columnType: 'text',
  })
}
```

`encryptColumn` renames `column` to the backing column `@deco.Encrypted()` reads
(`encrypted_phone` for `phone`), widens it to `text`, and encrypts each non-null
value. `decryptColumn` performs the inverse; pass `columnType` when the decrypted
column should be restored to something other than `text`. Both helpers run inside
the migration transaction and preserve nulls.

They encrypt and decrypt through the same `encryption.columns` keys as the
`@deco.Encrypted()` getter/setter — set in the scaffold's `conf/dream.ts` from
`COLUMN_ENCRYPTION_KEY` — and throw `MissingColumnEncryptionOpts` without them. The
model-side `@deco.Encrypted()` setup is in [models.md](models.md); run `pnpm psy sync`
after adding the decorator so the generated types list the plaintext property (`phone`)
among the model's virtual columns.

These helpers intentionally rewrite one row at a time, reading rows in bounded
batches. That keeps memory bounded but is still not an online migration for
very large tables. Drop indexes on the converted column first; indexing ciphertext is
usually not useful, and every rewritten row pays the index-maintenance cost.

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

Array columns (enum arrays, `text[]`, `integer[]`, etc.) work seamlessly with Dream — set and read them as regular arrays. **Never mutate an array column in place** (`.push()`, `.splice()`, index assignment): the instance's snapshot from its last load or save holds the same array object, so the mutation changes the snapshot too and the save writes nothing — even if you then assign the mutated array or a copy of it. Build the new array from the old one instead.

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

PostgreSQL cannot add and use a new enum value in the same transaction, and Dream runs all pending migrations in a single transaction by default. This isn't specific to renaming — **any** migration that adds an enum value and then uses that value (an `INSERT`, an `UPDATE`, a `dropEnumValue` replacement) needs the addition committed in its own transaction before a later migration can use it, per [Forcing a New Transaction in Migrations](#forcing-a-new-transaction-in-migrations) below. A rename is just the most common shape of this, and it's the one worked through here. To rename an enum value (add the new name, migrate data from old to new, drop the old name), you must use **two separate migration files**.

**Migration 1** — add the new enum value:

```bash
pnpm psy g:migration add-treehouse-place-style
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
pnpm psy g:migration replace-legacy-place-style-with-treehouse
```

```typescript
export async function up(db: Kysely<any>): Promise<void> {
  await DreamMigrationHelpers.dropEnumValue(db, {
    enumName: 'place_styles_enum',
    value: 'lean_to',
    replacements: [
      { table: 'places', column: 'style', replaceWith: 'treehouse' },
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

A `replacements` entry for a **scalar** enum column is exactly `{ table, column, replaceWith }`. The `behavior` key (`'remove'` to strip the value from array cells, `'replace'` to swap it) applies **only** to array columns, where the entry also carries `array: true`:

```typescript
// array column: strip 'lean_to' from every cell
{ table: 'places', column: 'past_styles', array: true, behavior: 'remove' }
// array column: swap 'lean_to' for 'treehouse' in every cell
{ table: 'places', column: 'past_styles', array: true, behavior: 'replace', replaceWith: 'treehouse' }
```

Dream automatically starts a new transaction when it encounters `dropEnumValue` in a migration, ensuring the new enum value from Migration 1 is committed and visible before Migration 2 tries to use it as a replacement. If an enum is used on multiple tables/columns, add a separate entry in the `replacements` array for each.

**Anything stored against a replacement column that the retype has to re-derive — a check constraint, a partial-index predicate, an expression index, a `DEFAULT` — can block `dropEnumValue`**, which retypes each replacement column to `text`, recreates the enum, and retypes back. The error it raises names the column, lists its check constraints with a paste-ready `DreamMigrationHelpers.dropConstraint(...)` call for each, and names the other candidates; drop the blocker before the first `dropEnumValue` and re-add whatever should survive. A type-agnostic check such as `column IS NOT NULL` never blocks.

**A fully-restoring rollback must be split across the pair, exactly like the forward pass.** The minimal `down()` above only re-adds the dropped value, which is safe. But a `down()` that also maps rows back with `dropEnumValue` can't re-`addEnumValue` and `dropEnumValue` in one file (same one-transaction constraint as the forward pass). Mirror the split: re-`addEnumValue`s in Migration 2's `down()`, row-mapping `dropEnumValue`s in Migration 1's `down()`. The pair rolls back together (`--steps 2`).

### Forcing a New Transaction in Migrations

Dream runs all pending migrations in a single transaction by default. When a migration's `up` or `down` contains `DreamMigrationHelpers.dropEnumValue` or `DreamMigrationHelpers.newTransaction()`, Dream commits everything before it and starts a new transaction there; the migrations after it share that transaction until the next such migration.

The check is a plain substring search of the `up`/`down` function source, so a call made from a helper function outside them, or through a renamed import (`import { DreamMigrationHelpers as DMH }`), is not detected.

```typescript
export async function up(db: Kysely<any>): Promise<void> {
  DreamMigrationHelpers.newTransaction()
  await db.schema.alterTable('places')...
}
```

## Raw Kysely Data Queries in Migrations

A migration's `db` handle is the same Kysely instance Dream builds everywhere else, with `CamelCasePlugin` applied: builder identifiers are snake-cased on the way out, and every result key is camelCased on the way back, unconditionally ([SKILL.md — Naming Conventions](SKILL.md#naming-conventions)). The rest of a migration file — DDL, `DreamMigrationHelpers` calls — is snake_case, but rows are not: select `host_id` and the value is at `row.hostId`, while `row.host_id` is `undefined` and `Kysely<any>` won't flag it. Write builder identifiers in camelCase so they match the keys you read back:

```typescript
export async function up(db: Kysely<any>): Promise<void> {
  const rooms = await db
    .selectFrom('rooms')
    .select(['id', 'hostId'])
    .execute()

  for (const room of rooms) {
    await db
      .updateTable('bookings')
      .set({ hostId: room.hostId })
      .where('roomId', '=', room.id)
      .execute()
  }
}
```

A `sql` tagged template is the one exception: its text reaches Postgres verbatim, so it names the real snake_case columns — but its rows still come back camelCased, and `sql<T>` checks nothing, so type it camelCase; a snake_case `T` compiles and every underscored key reads `undefined`.

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

// Multi-column, unique among live rows of a soft-deleted table
await db.schema
  .createIndex('localized_texts_localizable_for_locale')
  .on('localized_texts')
  .columns(['localizable_type', 'localizable_id', 'locale'])
  .unique()
  .where(sql<SqlBool>`deleted_at IS NULL`)
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

### Dropping a column: declare it ignored one deploy ahead

In a rolling deploy the migration finishes while containers built from the previous image are still serving, and those containers still name the dropped column in the SQL `leftJoinPreload`/`leftJoinLoad` generate — so every joined read of that model fails with `42703 column "..." does not exist` until the last old container drains. A spec suite runs against a single schema and cannot catch this.

Drop a column across two deploys, using the model's `ignoredColumns` getter:

```typescript
export default class Place extends ApplicationModel {
  public override get ignoredColumns() {
    return ['legacyNightlyRateCents'] as const
  }

  // ...
}
```

**Deploy 1:** remove every code reference to the column, declare it ignored, and run `pnpm psy sync`. The column is omitted from both generated types files, so it disappears from `Place.columns()` and every select list derived from it, and any reference left behind becomes a type error. **Deploy 2:** ship the `dropColumn` migration. No image carrying the declaration ever names the column, so the drop is safe on every code path — and rolling back to a declaration-carrying image stays safe afterward.

Two preconditions:

- **`ignoredColumns` is consumed by `sync`, not at runtime.** A model that declares it without a regenerated types file is not protected — have CI verify `pnpm psy sync` produces no diff.
- **The column must be nullable or carry a database default before deploy 1 ships.** An image built with the declaration never puts the column in an `INSERT` column list, so a `NOT NULL` column with no default fails every create.

## Extensions

```typescript
// Case-insensitive text
await DreamMigrationHelpers.createExtension(db, 'citext')

// Trigram similarity for fuzzy search
await DreamMigrationHelpers.createExtension(db, 'pg_trgm')
```

## Polymorphic Association Columns

For polymorphic BelongsTo, you need a type column (here an enum of the target model names) and an ID column:

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
  .addColumn('markdown', 'text')
  .addColumn('created_at', 'timestamp', col => col.notNull())
  .addColumn('updated_at', 'timestamp', col => col.notNull())
  .addColumn('deleted_at', 'timestamp')
  .execute()

// Polymorphic lookup, one live row per locale
await db.schema
  .createIndex('localized_texts_localizable_for_locale')
  .on('localized_texts')
  .columns(['localizable_type', 'localizable_id', 'locale'])
  .unique()
  .where(sql<SqlBool>`deleted_at IS NULL`)
  .execute()
```

## STI Columns

STI uses a `type` enum column, whose values are the child class names, plus child-specific columns with check constraints:

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
phone:encrypted                # encrypted text column → DB column `encrypted_phone`
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
  InternalUser@canceled_by:belongs_to:optional
```

Generates:

```typescript
@deco.BelongsTo('InternalUser', { on: 'canceledById', optional: true })
public canceledBy: InternalUser | null
public canceledById: DreamColumn<MessageRequest, 'canceledById'>
```

**Canonical use case** — the `_by` pattern: `User@created_by`, `User@approved_by`, `InternalUser@canceled_by`. Lets one model carry multiple FKs to the same target without collision (`Messaging/Message@last_inbound:belongs_to`, `Messaging/Message@last_outbound:belongs_to`, etc.).

Without an `@alias`, the property/column derive from the model's last namespace segment — the plain `Model:belongs_to[:optional]` form. Works in every generator that accepts `belongs_to`: `g:model`, `g:migration`, `g:resource`. `g:sti-child` does not accept `belongs_to` — STI children cannot declare associations (see [sti.md — STI Limitations](sti.md#sti-limitations)); declare the association on the STI parent instead.
