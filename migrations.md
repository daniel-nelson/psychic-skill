# Dream Migrations - Detailed Reference

## Overview

Dream migrations use Kysely under the hood. **Migrations are always created via generators** (`g:resource`, `g:model`, `g:sti-child`, or `g:migration`) - never create migration files by hand. This reference covers the Kysely API and DreamMigrationHelpers available for editing generated migrations.

**CRITICAL: Never modify an existing migration file that has already been merged into main.** Use `pnpm psy g:migration` to create a new migration for additional changes.

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
```typescript
.addColumn('metadata', 'jsonb')
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

When using `pnpm psy g:model` or `pnpm psy g:migration`:

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
types:enum[]:room_types:bathroom,bedroom     # Enum array
User:belongs_to                # user_id FK
User:belongs_to:optional       # nullable FK
encrypted_phone:encrypted      # encrypted text column
```
