# Soft Delete

The `@SoftDelete()` decorator enables a model to be hidden from all queries without being deleted from the database.

When a `@SoftDelete` model is destroyed, Dream sets the `deletedAt` column instead of removing the row. A [default scope](models.md#default-scopes) named `dream:SoftDelete` is automatically applied to hide records where `deletedAt` is not null.

Common use cases:
- **Undo** — immediately reverse an accidental deletion with `undestroy()`
- **Trash can** — hide deleted records from normal queries for a retention period (e.g. 30 days), allow users to browse and restore them, then permanently delete expired records via a scheduled job using `reallyDestroy()`
- **Data preservation** — retain records for auditing, analytics, or compliance while removing them from the application's active data

**Don't hand-roll a deactivate/delete mechanism.** When you need "removed but recoverable / auditable" semantics, that is exactly what `@SoftDelete()` provides — and generators apply it by default, so a custom `removed`/`isDeleted`/`deactivatedAt` column is almost always redundant and fights the lifecycle (your column won't be honored by `destroy()`/`undestroy()`, the `dream:SoftDelete` default scope, or `dependent: 'destroy'` cascades). A domain status flag is only warranted when it means something *other than* deletion — e.g. an `active` flag that means "currently bookable" while the row is still a live, queryable record. If the flag's real meaning is "this record is gone," delete the flag and use `@SoftDelete`.

## Setup

The `g:resource` and `g:model` generators automatically include `@SoftDelete()` and a `deleted_at` column. **STI children never receive `@SoftDelete()`** — soft delete is enforced at the STI parent level, and `g:sti-child` does not accept `--no-soft-delete`. The setup below is only needed when adding soft delete to an existing model that was generated with `--no-soft-delete`, or to an STI parent that was originally created without it.

### 1. Add a `deletedAt` column

```bash
pnpm psy g:migration add-deleted-at-to-places deleted_at:datetime:optional
pnpm psy db:migrate
```

### 2. Apply the decorator

```typescript
import { Decorators, SoftDelete } from '@rvoh/dream'
import { DreamColumn } from '@rvoh/dream/types'

const deco = new Decorators<typeof Place>()

@SoftDelete()
export default class Place extends ApplicationModel {
  public override get table() {
    return 'places' as const
  }

  public id: DreamColumn<Place, 'id'>
  public name: DreamColumn<Place, 'name'>
  public deletedAt: DreamColumn<Place, 'deletedAt'>
  public createdAt: DreamColumn<Place, 'createdAt'>
  public updatedAt: DreamColumn<Place, 'updatedAt'>
}
```

### 3. Run sync

```bash
pnpm psy sync
```

### Foreign key constraints

When using soft delete, use the default `restrict` foreign key constraint (not `cascade`). Since the parent row is not actually deleted, a `cascade` constraint would not fire, and `restrict` correctly prevents deletion of rows that still have references:

```typescript
await db.schema
  .createTable('rooms')
  .addColumn('id', 'bigint', col => col.primaryKey().generatedByDefaultAsIdentity())
  .addColumn('place_id', 'bigint', col =>
    col.references('places.id').onDelete('restrict').notNull()
  )
  .execute()
```

### Natural-key unique indexes

When you add `@SoftDelete()` to a model that has a **unique index on a natural key** — a slug, a code, a `(host_id, name)` pair, anything other than the primary key — migrate that index to a **partial** unique index with `WHERE deleted_at IS NULL`. Soft-deleted rows stay in the table, so a plain unique index keeps reserving the natural key after deletion, blocking a new live row that reuses it.

```typescript
import { sql, type SqlBool } from 'kysely'

// Replace the full unique index with one that only constrains live rows.
await db.schema.dropIndex('places_slug_unique').execute()

await db.schema
  .createIndex('places_slug_unique')
  .unique()
  .on('places')
  .column('slug')
  .where(sql<SqlBool>`deleted_at IS NULL`)
  .execute()
```

Without the predicate, soft-deleting a Place with `slug: 'cozy-cabin'` and then creating a new one with the same slug fails on the unique constraint, even though no *live* row holds it. The `sql<SqlBool>` cast on the predicate is required (see [migrations.md](migrations.md#alter-table)). One consequence to expect: `undestroy()` can now fail if a live row claimed the natural key while the row was soft-deleted — restoring would create two live rows with the same key, which the partial index correctly rejects.

### Sortable position columns

When you add `@SoftDelete()` to a model that also has `@deco.Sortable()` (see [models.md](models.md#special-decorators)), make the position column nullable. Soft-deleting a record sets every `@Sortable` field's position column to `null` in the same `UPDATE` as `deletedAt`, clearing the record's slot in its sortable scope. A `NOT NULL` position column throws a not-null violation — including when the sortable model is only a `dependent: 'destroy'` cascade target of a parent being destroyed, not just on a direct `destroy()` call.

```typescript
await db.schema
  .alterTable('rooms')
  .alterColumn('position', col => col.dropNotNull())
  .execute()
```

Retrofitting `@SoftDelete()` onto an existing `@Sortable` model requires dropping the `NOT NULL` constraint on the position column in the same migration.

## Destroying and Restoring

### Soft delete (default)

Calling `destroy()` on a `@SoftDelete` model sets `deletedAt` to the current timestamp:

```typescript
const place = await Place.findOrFail(id)
await place.destroy()
// place.deletedAt is now set; row still exists in the database
```

### Restore (undestroy)

Soft-deleted records can be restored with `undestroy()`, which sets `deletedAt` back to null:

```typescript
// Must bypass the SoftDelete scope to find the record
const place = await Place.removeAllDefaultScopes().findOrFail(id)
await place.undestroy()
// place.deletedAt is now null; record is visible again in normal queries
```

Associations can be restored with `undestroyAssociation()`:

```typescript
await place.undestroyAssociation('rooms')
```

Query-level undestroy is also available:

```typescript
await Place.removeAllDefaultScopes().where({ id }).undestroy()
```

### Permanent delete (reallyDestroy)

To permanently remove the row from the database:

```typescript
await place.reallyDestroy()
```

For associations:

```typescript
await place.reallyDestroyAssociation('rooms')
await place.reallyDestroyAssociation('rooms', { and: { name: 'my room' } })
```

### Guarded (compare-and-set) destroy

Plain `destroy()`/`reallyDestroy()` on a query read the matching records and then destroy each one. A concurrent writer can change a record in that window — moving it out of the set the query originally matched — and it gets destroyed anyway. Pass `lock: true` to close that window:

```typescript
const destroyed = await Booking.where({ status: 'pending' }).destroy({ lock: true })
// 0 — someone else already moved every matching booking out of 'pending'
```

Records are processed in batches; each batch re-selects its rows with an exclusive row lock (`FOR UPDATE OF`) inside its own transaction before destroying them. A record another transaction has already moved out of the query drops out of the locked read and is left alone. The returned count is the number of records actually claimed, so a caller can detect a lost race by comparing it to what it expected.

Four things to keep in mind:

- **The guarantee is per batch, not set-wide.** A run spanning more than one batch can win the race in one batch and lose it in another — nothing holds the whole matched set still for the duration of the run. For all-or-nothing across the whole set, wrap the call in `ApplicationModel.transaction(...)` and thread it through with `.txn(txn)` (see [models.md — Transactions](models.md#transactions)):

  ```typescript
  await ApplicationModel.transaction(async txn => {
    await Booking.txn(txn).where({ status: 'pending' }).destroy({ lock: true })
  })
  ```

  Every batch then shares that transaction and holds its locks until you commit, at the cost of blocking for the whole run.
- **`batchSize` defaults to 10 under `lock`**, not the 1000 `findEach` and unlocked `destroy`/`reallyDestroy` use. Every record in a batch stays locked for that batch's full destroy-hook and `dependent: 'destroy'` cascade duration, so an oversized batch blocks unrelated writers touching those rows. Lower `batchSize` further for models with deep cascades or expensive destroy hooks.
- **This is not a bulk-delete accelerator — it's the opposite.** If you're destroying a large set and don't need compare-and-set, pass no `lock` option; plain `destroy()` or `.delete()` are the right tools for that.
- **`lock` lives on the query**, not on an instance — `Booking.where(...).destroy({ lock: true })` or `Booking.query().destroy({ lock: true })`. There's no instance-level equivalent, and none is needed: destroying a single already-loaded instance has no select-then-destroy window to close.

## Cascading

Both `destroy()` and `undestroy()` cascade through associations declared with `dependent: 'destroy'`.

**CRITICAL: Every model in a `dependent: 'destroy'` chain MUST also have `@SoftDelete()`, or those associated records will be permanently deleted when the parent is destroyed.** Dream does not check this for you — if a `dependent: 'destroy'` association points to a model without `@SoftDelete`, calling `destroy()` on the parent will irreversibly delete those associated records from the database.

```typescript
@SoftDelete()
export default class Place extends ApplicationModel {
  // Room MUST also have @SoftDelete(), or rooms will be permanently deleted
  @deco.HasMany('Room', { dependent: 'destroy' })
  public rooms: Room[]

  // HostPlace MUST also have @SoftDelete()
  @deco.HasMany('HostPlace', { dependent: 'destroy' })
  public hostPlaces: HostPlace[]
}
```

```typescript
// Soft-deletes the place AND cascades soft-delete to rooms and hostPlaces
await place.destroy()

// Hidden from normal queries
await Room.where({ place }).count() // 0

// Still in the database
await Room.removeAllDefaultScopes().where({ place }).count() // 3

// Undestroy also cascades — restores the place AND its rooms and hostPlaces
await place.undestroy()
await Room.where({ place }).count() // 3
```

## Querying Soft-Deleted Records

The `@SoftDelete` decorator adds a [default scope](models.md#default-scopes) (`dream:SoftDelete`) that filters out records where `deletedAt` is not null. To include soft-deleted records, remove the scope. Use `removeDefaultScope` when querying for a plurality and `removeAllDefaultScopes` when targeting a specific record (see [default scopes](models.md#default-scopes) for the rationale):

```typescript
// Querying for multiple soft-deleted records — use removeDefaultScope to preserve other scopes
await Place.removeDefaultScope('dream:SoftDelete').where({ style: 'cabin' }).all()
await user.associationQuery('places').removeDefaultScope('dream:SoftDelete').all()

// Finding a specific soft-deleted record — use removeAllDefaultScopes
await Place.removeAllDefaultScopes().findOrFail(id)
```

## STI and SoftDelete

`@SoftDelete()` must be applied to the **STI parent**, not to STI children. When an STI parent has `@SoftDelete`, all children inherit it. STI children have their own default scope (`dream:STI`) in addition to the inherited `dream:SoftDelete` scope.

## Testing Soft Delete

```typescript
describe('Place', () => {
  describe('soft delete', () => {
    it('soft deletes the record', async () => {
      const place = await createPlace()
      await place.destroy()
      expect(await Place.where({ id: place.id }).exists()).toBe(false)
      expect(await Place.removeAllDefaultScopes().where({ id: place.id }).exists()).toBe(true)
    })

    it('cascades soft delete to dependent associations', async () => {
      const place = await createPlace()
      const room = await createBedroom({ place })

      await place.destroy()

      expect(await Room.where({ id: room.id }).exists()).toBe(false)
      expect(await Room.removeAllDefaultScopes().where({ id: room.id }).exists()).toBe(true)
    })
  })
})
```
