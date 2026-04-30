# Soft Delete

The `@SoftDelete()` decorator enables a model to be hidden from all queries without being deleted from the database.

When a `@SoftDelete` model is destroyed, Dream sets the `deletedAt` column instead of removing the row. A [default scope](models.md#default-scopes) named `dream:SoftDelete` is automatically applied to hide records where `deletedAt` is not null.

Common use cases:
- **Undo** — immediately reverse an accidental deletion with `undestroy()`
- **Trash can** — hide deleted records from normal queries for a retention period (e.g. 30 days), allow users to browse and restore them, then permanently delete expired records via a scheduled job using `reallyDestroy()`
- **Data preservation** — retain records for auditing, analytics, or compliance while removing them from the application's active data

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
  .addColumn('id', 'bigserial', col => col.primaryKey())
  .addColumn('place_id', 'bigint', col =>
    col.references('places.id').onDelete('restrict').notNull()
  )
  .execute()
```

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
await Room.where({ placeId: place.id }).count() // 0

// Still in the database
await Room.removeAllDefaultScopes().where({ placeId: place.id }).count() // 3

// Undestroy also cascades — restores the place AND its rooms and hostPlaces
await place.undestroy()
await Room.where({ placeId: place.id }).count() // 3
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
