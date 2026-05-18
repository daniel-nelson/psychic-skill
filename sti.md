# Single Table Inheritance (STI) - Complete Guide

## Overview

STI stores multiple model types in a single database table, discriminated by a `type` column. The parent table holds all columns for all child types; child-specific columns are nullable.

## Generating an STI Resource

### Step 1: Generate the Parent with `--sti-base-serializer`

Use `g:resource` with the `--sti-base-serializer` flag. This flag generates **base serializers specially designed for STI extension** - they accept a generic `StiChildClass` parameter and include the `type` attribute with an enum constrained to the child's class name. This is critical for clients consuming the OpenAPI spec and JSON responses to know which shape they're receiving. Children extend these base serializers and pass their own class.

```bash
pnpm psy g:resource --sti-base-serializer --owning-model=Place \
  v1/host/places/{}/rooms Room \
  type:enum:room_types:Bathroom,Bedroom,Kitchen,Den,LivingRoom \
  Place:belongs_to \
  position:integer:optional \
  deleted_at:datetime:optional
```

**Breaking down the command:**
- `--sti-base-serializer` - Generates generic base serializers that accept a `StiChildClass` param and include a `type` attribute with a per-child enum, so clients know which shape they're receiving
- `--owning-model=Place` - The parent model for authorization in controllers
- `v1/host/places/{}/rooms` - Route path (`{}` is a placeholder for the parent resource ID param)
- `Room` - Model name (the STI base class)
- `type:enum:room_types:Bathroom,Bedroom,...` - The type discriminator column as an enum with all child type values
- `Place:belongs_to` - Association column
- `position:integer:optional` - Shared column (nullable)
- `deleted_at:datetime:optional` - For soft delete

**What this generates:**
- Migration creating the `rooms` table with `type` enum column
- `Room` model with STI-aware serializers
- `RoomSerializer.ts` and `RoomSummarySerializer.ts` (base serializers for children to extend)
- Controller with CRUD actions
- Route entry
- Factory and spec files

Run after generating:
```bash
pnpm psy db:migrate
```

### Step 2: Generate STI Children with `g:sti-child`

```bash
pnpm psy g:sti-child [options] <ChildPath> extends <ParentModel> [columns...]
```

**Examples:**

Child with an enum column:
```bash
pnpm psy g:sti-child --model-name=Bathroom \
  Room/Bathroom extends Room \
  bath_or_shower_style:enum:bath_or_shower_styles:bath,shower,bath_and_shower,none
pnpm psy db:migrate
```

Child with an enum array column:
```bash
pnpm psy g:sti-child --model-name=Bedroom \
  Room/Bedroom extends Room \
  bed_types:enum[]:bed_types:twin,bunk,queen,king,cot,sofabed
pnpm psy db:migrate
```

Child with no additional columns:
```bash
pnpm psy g:sti-child --model-name=Den Room/Den extends Room
pnpm psy g:sti-child --model-name=LivingRoom Room/LivingRoom extends Room
```

After all children are generated:
```bash
pnpm psy db:migrate
pnpm psy sync
```

**`g:sti-child` flags:**
- `--model-name=ClassName` - Explicit class name (e.g., `--model-name=Bathroom` for path `Room/Bathroom`)
- `--admin-serializers` - Also generate `AdminSerializer` and `AdminSummarySerializer` variants
- `--internal-serializers` - Also generate `InternalSerializer` and `InternalSummarySerializer` variants
- `--no-serializer` - Skip serializer generation entirely
- `--connection-name` - Database connection (defaults to "default")

**What `g:sti-child` generates:**
- Dream STI child model decorated with `@STI(ParentClass)`
- Child serializers that extend parent serializers
- Migration that **ALTERs the parent table** (not a new table) to add child-specific columns. **No migration is emitted when the child declares no additional columns** — STI children share the parent's table, so a no-columns child requires no schema change.
- Check constraints ensuring child columns are NOT NULL only for that child type
- Factory and spec skeleton

STI children never receive `@SoftDelete()` — soft delete is enforced at the STI parent level — and `g:sti-child` does not accept `--no-soft-delete`.

**Batch invocation for many children.** When you need to create many STI children at once (e.g., a fixed taxonomy of 9 action-item types), run the generator once per child and chain the invocations with `&&` so a failure on any one stops the batch:

```bash
pnpm psy g:sti-child Parent/ChildA extends Parent && \
pnpm psy g:sti-child Parent/ChildB extends Parent && \
pnpm psy g:sti-child Parent/ChildC extends Parent
```

Children with no additional columns produce only a model file (no migration), so the batch is fast. **Always use the generator** — never hand-craft STI child model/serializer/factory/spec files, even when "they all look the same." The generator handles enum updates, check-constraint additions, factory wiring, and serializer scaffolding consistently; hand-crafted versions drift.

### Key Differences

| Aspect | Parent (`g:resource`) | Child (`g:sti-child`) |
|--------|----------------------|----------------------|
| Command | `pnpm psy g:resource --sti-base-serializer` | `pnpm psy g:sti-child` |
| Table | Creates new table | ALTERs parent table |
| Model | Standard model with serializers getter | `@STI(Parent)` decorator, overrides serializers getter |
| Controller | Generated | Not generated (uses parent's controller) |
| Serializer | Created as extensible base | Extends parent serializer |
| Migration | CREATE TABLE with `type` enum | ALTER TABLE, add columns + check constraints |

## Generated File Structure

```
src/app/
  models/
    Room.ts                    # Parent model (STI base with generic serializers)
    Room/
      Bathroom.ts              # @STI(Room), overrides serializers getter
      Bedroom.ts
      Kitchen.ts
      Den.ts
      LivingRoom.ts
  serializers/
    RoomSerializer.ts          # Base serializers (generic, for extension)
    Room/
      BathroomSerializer.ts    # Extends RoomSerializer/RoomSummarySerializer
      BedroomSerializer.ts
      KitchenSerializer.ts
      DenSerializer.ts
      LivingRoomSerializer.ts
  controllers/
    V1/Host/Places/
      RoomsController.ts       # Single controller handles all room types
spec/
  factories/
    Room/
      BathroomFactory.ts
      BedroomFactory.ts
      ...
  unit/
    models/
      Room/
        Bathroom.spec.ts
        Bedroom.spec.ts
        ...
```

## Model Patterns

### Parent Model (generated with `--sti-base-serializer`)

The parent model has a `get serializers()` getter pointing to the STI base serializers. These base serializers are generic and accept a `StiChildClass` parameter, so each child passes its own class to get the correct `type` enum value in the OpenAPI schema and JSON output.

```typescript
import { Decorators, SoftDelete } from '@rvoh/dream'
import { DreamColumn } from '@rvoh/dream/types'
import ApplicationModel from './ApplicationModel.js'
import Place from './Place.js'

const deco = new Decorators<typeof Room>()

@SoftDelete()
export default class Room extends ApplicationModel {
  public override get table() {
    return 'rooms' as const
  }

  // STI base serializers - generic, accept child class for type discrimination

  public id: DreamColumn<Room, 'id'>
  public type: DreamColumn<Room, 'type'>
  public position: DreamColumn<Room, 'position'>
  public deletedAt: DreamColumn<Room, 'deletedAt'>
  public createdAt: DreamColumn<Room, 'createdAt'>
  public updatedAt: DreamColumn<Room, 'updatedAt'>

  @deco.BelongsTo('Place')
  public place: Place
  public placeId: DreamColumn<Room, 'placeId'>

  // scope also accepts an array, e.g. ['place', 'type'] to create independent
  // position sequences per place AND per STI type (1-n for each type).
  @deco.Sortable({ scope: 'place' })
  public declare position: DreamColumn<Room, 'position'>
}
```

### STI Child Model

```typescript
import { STI } from '@rvoh/dream'
import { DreamColumn, DreamSerializers } from '@rvoh/dream/types'
import Room from '../Room.js'

@STI(Room)
export default class Bedroom extends Room {
  public override get serializers(): DreamSerializers<Bedroom> {
    return {
      default: 'Room/BedroomSerializer',
      summary: 'Room/BedroomSummarySerializer',
      forGuests: 'Room/BedroomForGuestsSerializer',
    }
  }

  // Child-specific columns (nullable in DB, but typed for this child)
  public bedTypes: DreamColumn<Bedroom, 'bedTypes'>
}
```

### STI Limitations

- Children **cannot define new associations** - all associations must be on the parent
- Children **cannot use `@SoftDelete()`** - must be on the parent
- Children **cannot use `@ReplicaSafe()`** - must be on the parent
- Children **cannot use `@Sortable()`** - must be on the parent. If you need position sorting scoped per STI type, declare `@Sortable` on the base model with `type` in the scope array (e.g., `scope: ['place', 'type']`)
- Children can override `get serializers()` (and should)
- Children can add child-specific columns

### Changing an STI Record's Type

Dream allows changing an STI record's type after creation by updating the `type` column. Use `skipHooks: true` to bypass model-layer hooks that may interfere, and set all fields required by the new type in the same update. Database-level check constraints (generated by `g:sti-child`) will reject invalid data regardless.

```typescript
// Change a Bedroom into a Bathroom (bathOrShowerStyle is required on Bathroom)
await bedroom.update({ type: 'Bathroom', bathOrShowerStyle: 'shower' }, { skipHooks: true })

// Re-query via the base class — returns a Bathroom, but the type system doesn't know it
const room = await Room.findOrFail(bedroom.id)

// OR re-query via the target child class — type system knows it is a Bathroom
const bathroom = await Bathroom.findOrFail(bedroom.id)
```

The re-query is necessary because Dream instantiates model classes based on the `type` column at query time. After `update()`, the in-memory object is still the old class with the old serializers, associations, and behavior.

## Serializer Patterns

### Base Serializer (generic, for extension by children)

```typescript
import Room from '@models/Room.js'
import { DreamSerializer } from '@rvoh/dream'

export const RoomSummarySerializer = <T extends Room>(StiChildClass: typeof Room, room: T) =>
  DreamSerializer(StiChildClass ?? Room, room)
    .attribute('id')
    .attribute('type', {
      openapi: { type: 'string', enum: [(StiChildClass ?? Room).sanitizedName] },
    })
    .attribute('position')

export const RoomSerializer = <T extends Room>(StiChildClass: typeof Room, room: T) =>
  RoomSummarySerializer(StiChildClass, room)
    .attribute('deletedAt')
```

Key points:
- Uses generic `<T extends Room>` so any child class works
- First param is `StiChildClass: typeof Room` - the child class itself
- `.sanitizedName` returns the class name for the `type` enum value
- `StiChildClass ?? Room` fallback handles edge cases

### Child Serializer (extends base)

```typescript
import Bedroom from '@models/Room/Bedroom.js'
import { RoomSerializer, RoomSummarySerializer } from '@serializers/RoomSerializer.js'

// Summary extends base summary
export const RoomBedroomSummarySerializer = (bedroom: Bedroom) =>
  RoomSummarySerializer(Bedroom, bedroom)

// Default extends base default, adds child-specific fields
export const RoomBedroomSerializer = (bedroom: Bedroom) =>
  RoomSerializer(Bedroom, bedroom)
    .attribute('bedTypes')

// Guest-facing with passthrough for i18n
export const RoomBedroomForGuestsSerializer = (
  bedroom: Bedroom,
  passthrough: { locale: LocalesEnum }
) =>
  RoomForGuestsSerializer(Bedroom, bedroom, passthrough)
    .rendersMany('bedTypes', { serializer: BedTypeSerializer })
```

### Hand-adding a new base-serializer variant (e.g. `forGuests`)

The generator emits a summary + default base-serializer pair for the **default** variant, and *also* emits `Admin`/`AdminSummary` and/or `Internal`/`InternalSummary` variant pairs when those are requested — `g:model` via the `--admin-serializers` / `--internal-serializers` flags, `g:resource` *automatically* when the resource is generated under the `Admin/` or `Internal/` namespace (it infers `forAdmin` / `forInternal` from the controller path). Every generator-emitted variant — including admin and internal — already carries the correct STI base shape and `type` discriminator, and STI children import the matching variant so the chain stays within-variant (admin → admin, internal → internal). **You do not hand-write admin/internal variants.**

The hand-written case is a *bespoke* variant with no corresponding flag or namespace inference — a `forGuests` flavor is the canonical example (there is no guest-serializer flag). A hand-added variant **must replicate the STI base shape itself**. This is the step that's easy to get wrong: an agent adds `RoomForGuestsSerializer` by copying a plain serializer and silently breaks STI discrimination.

Contrast the two shapes. A **non-STI** serializer (e.g. `Place`) takes just the model:

```typescript
export const PlaceForGuestsSerializer = (place: Place, passthrough: { locale: LocalesEnum }) =>
  DreamSerializer(Place, place).attribute('id') // ...
```

An **STI base** serializer — including every hand-added variant — must take `StiChildClass` first, bind `DreamSerializer` to `StiChildClass ?? Parent`, and re-declare the `type` discriminator attribute:

```typescript
export const RoomForGuestsSerializer = <T extends Room>(
  StiChildClass: typeof Room,
  room: T,
  passthrough: { locale: LocalesEnum },
) =>
  DreamSerializer(StiChildClass ?? Room, room)
    .attribute('id')
    .attribute('type', { openapi: { type: 'string', enum: [(StiChildClass ?? Room).sanitizedName] } })
    .customAttribute('displayType', () => i18n(passthrough.locale, `rooms.type.${room.type}`), { openapi: 'string' })
    // ...child-agnostic guest fields
```

The three STI-load-bearing parts — the `StiChildClass` parameter, `DreamSerializer(StiChildClass ?? Room, room)`, and the `type` attribute whose OpenAPI `enum` is `[(StiChildClass ?? Room).sanitizedName]` — are exactly what a plain serializer lacks. Each child serializer then calls the variant with its concrete class (`RoomForGuestsSerializer(Kitchen, kitchen, passthrough)`), which is what narrows that child's rendered `type` to a single-value enum (`['Kitchen']`) and binds the serializer to the child's columns.

**Why it matters / what breaks:** the per-child single-value `type` enum is the discriminator that makes the generated OpenAPI a discriminated union, so clients and `fastJsonStringify` know which child shape they received. Drop the `type` attribute (or write the variant in the plain non-STI shape) and every child collapses to one indistinguishable schema. The failure is silent in a unit spec — direct serializer rendering still looks correct — and only shows up over HTTP: under `fastJsonStringify` the response omits child-specific fields. When you see that exact symptom (direct render correct, HTTP response missing STI child fields), suspect a hand-written serializer that lost the `type`/`StiChildClass` shape or the resulting OpenAPI schema — not Dream model instantiation or `preloadFor`.

## Migration Patterns

The STI `type` column **must** use a database enum type, not `varchar`. The enum values must exactly match the STI child class names.

### Parent Table Migration

```typescript
export async function up(db: Kysely<any>): Promise<void> {
  await db.schema
    .createType('room_types_enum')
    .asEnum(['Bathroom', 'Bedroom', 'Kitchen', 'Den', 'LivingRoom'])
    .execute()

  await db.schema
    .createTable('rooms')
    .addColumn('id', 'uuid', col => col.primaryKey().defaultTo(sql`uuidv7()`))
    .addColumn('type', sql`room_types_enum`, col => col.notNull())
    .addColumn('position', 'integer', col => col.notNull())
    .addColumn('place_id', 'uuid', col =>
      col.references('places.id').onDelete('restrict').notNull()
    )
    .addColumn('deleted_at', 'timestamp')
    .addColumn('created_at', 'timestamp', col => col.notNull())
    .addColumn('updated_at', 'timestamp', col => col.notNull())
    .execute()

  await db.schema.createIndex('rooms_type').on('rooms').column('type').execute()
  await db.schema.createIndex('rooms_place_id').on('rooms').column('place_id').execute()
}
```

### STI Child Migration (ALTERs parent table)

```typescript
export async function up(db: Kysely<any>): Promise<void> {
  // Create child-specific enum
  await db.schema
    .createType('bath_or_shower_styles_enum')
    .asEnum(['bath', 'shower', 'bath_and_shower', 'none'])
    .execute()

  // Add column to PARENT table (not a new table)
  await db.schema
    .alterTable('rooms')
    .addColumn('bath_or_shower_style', sql`bath_or_shower_styles_enum`)
    .execute()

  // Check constraint: column required ONLY for this child type
  await db.schema
    .alterTable('rooms')
    .addCheckConstraint(
      'rooms_not_null_bath_or_shower_style',
      sql`type != 'Bathroom' OR bath_or_shower_style IS NOT NULL`,
    )
    .execute()
}

export async function down(db: Kysely<any>): Promise<void> {
  await db.schema.alterTable('rooms').dropConstraint('rooms_not_null_bath_or_shower_style').execute()
  await db.schema.alterTable('rooms').dropColumn('bath_or_shower_style').execute()
  await db.schema.dropType('bath_or_shower_styles_enum').execute()
}
```

### Array Enum Child Migration

```typescript
export async function up(db: Kysely<any>): Promise<void> {
  await db.schema
    .createType('bed_types_enum')
    .asEnum(['twin', 'bunk', 'queen', 'king', 'cot', 'sofabed'])
    .execute()

  await db.schema
    .alterTable('rooms')
    .addColumn('bed_types', sql`bed_types_enum[]`, col => col.notNull().defaultTo('{}'))
    .execute()
}
```

### Shared Columns Across STI Children

When multiple STI children share a column name (e.g., both `Bedroom` and `LivingRoom` have a `capacity` column), include the column in each `g:sti-child` command so that each child's model and serializer are generated correctly. However, after running the generators, **remove the duplicate `addColumn` from all but the first migration** — the second migration will fail with "column already exists" since they all ALTER the same parent table. Also remove the corresponding `dropColumn` from those migrations' `down` methods.

**Check constraints for non-optional shared columns.** When the shared column is not `:optional`, and not a simple boolean, the generator adds a check constraint like `type != 'Bedroom' OR capacity IS NOT NULL` to the first migration. The constraint must cover every STI child type that requires the column:

- **If the first child's migration has not yet been deployed** (still on a feature branch), edit that migration to include all the additional types in the check constraint: `sql\`type NOT IN ('Bedroom', 'LivingRoom') OR capacity IS NOT NULL\``.
- **If the first child's migration has already been deployed**, generate a new migration that drops the existing constraint and adds a replacement covering the new type(s):
  ```typescript
  await db.schema.alterTable('rooms').dropConstraint('rooms_not_null_capacity').execute()
  await db.schema
    .alterTable('rooms')
    .addCheckConstraint(
      'rooms_not_null_capacity',
      sql`type NOT IN ('Bedroom', 'LivingRoom') OR capacity IS NOT NULL`,
    )
    .execute()
  ```

Columns shared by **all** STI children should be on the STI base model instead, added as part of the initial `g:resource` or `g:model` command.

## Controller Pattern

A single controller handles ALL STI types. The `create` action switches on `type` with a `_never` default — this is the STI application of [SKILL.md Critical Rule 14](SKILL.md#critical-rules) (exhaustive switch on closed enums); the rule applies to any closed-enum dispatch, not just STI discriminators.

```typescript
export default class V1HostPlacesRoomsController extends V1HostPlacesBaseController {
  @OpenAPI(Room, {
    status: 200,
    tags: ['rooms'],
    cursorPaginate: true,
    serializerKey: 'summary',
    fastJsonStringify: true,
  })
  public async index() {
    const rooms = await this.currentPlace
      .associationQuery('rooms')
      .preloadFor('summary')
      .cursorPaginate({
        cursor: this.castParam('cursor', 'string', { allowNull: true }),
      })
    this.ok(rooms)
  }

  @OpenAPI(Room, { status: 201, tags: ['rooms'], fastJsonStringify: true })
  public async create() {
    const roomType = this.castParam('type', 'string', { enum: RoomTypesEnumValues })
    const roomParams = this.extractParams(Room, ['name', 'position', 'bedTypes'])
    let room: Room

    switch (roomType) {
      case 'Bathroom':
        room = await Bathroom.create({ place: this.currentPlace, ...roomParams })
        break
      case 'Bedroom':
        room = await Bedroom.create({ place: this.currentPlace, ...roomParams })
        break
      case 'Kitchen':
        room = await Kitchen.create({ place: this.currentPlace, ...roomParams })
        break
      case 'Den':
        room = await Den.create({ place: this.currentPlace, ...roomParams })
        break
      case 'LivingRoom':
        room = await LivingRoom.create({ place: this.currentPlace, ...roomParams })
        break
      default: {
        // TypeScript exhaustiveness check - compiler error if a type is missed
        const _never: never = roomType
        throw new Error(`Unhandled RoomTypesEnum: ${_never as string}`)
      }
    }

    if (room.isPersisted) room = await room.loadFor('default').execute()
    this.created(room)
  }

  @OpenAPI(Room, { status: 204, tags: ['rooms'] })
  public async update() {
    const room = await this.room()
    await room.update(this.extractParams(Room, ['name', 'position', 'bedTypes']))
    this.noContent()
  }

  @OpenAPI({ status: 204, tags: ['rooms'] })
  public async destroy() {
    const room = await this.room()
    await room.destroy()
    this.noContent()
  }

  private async room() {
    return await this.currentPlace
      .associationQuery('rooms')
      .preloadFor('default')
      .findOrFail(this.castParam('id', 'string'))
  }
}
```

**Key points:**
- `RoomTypesEnumValues` is auto-generated from the enum in `types/db.ts`
- `this.extractParams(Room, [...])` uses the parent class — this works because extraction reads from the **database table**, not the model class. Since all STI children share the same table, the allowlist can name columns from any child (including child-specific ones like `bedTypes`) and a single extraction call covers every concrete subtype. This also means `update` doesn't need a switch statement — one `extractParams(Room, [...])` call handles every child's columns.
- Each `case` creates via the **child** class (e.g., `Bedroom.create(...)`)
- The `default` branch with `const _never: never = roomType` ensures compile-time exhaustiveness
- `show`, `update`, `destroy` work on the parent `Room` type - Dream returns the correct child instance

### Bulk creation from a request-body array

For endpoints that create many STI children in one request — `{ items: [{ type: 'Bedroom', position: 1, ... }, { type: 'Kitchen', position: 2, ... }] }` — use `extractParams(STIParent, [...], { key, array: true })` to validate each entry against the parent's safe columns. The framework still strips `type` (the STI discriminator) and FKs (e.g., `placeId`) per the standard exclusions, so pull `type` per-item from the raw request body and let an exhaustive switch (per [SKILL.md Critical Rule 14](SKILL.md#critical-rules)) enforce validity.

```typescript
// Body: { items: [{ type: 'Bedroom', position: 1, ... }, { type: 'Kitchen', position: 2, ... }] }
const itemsParams = this.extractParams(Room, ['name', 'position', 'bedTypes'], { key: 'items', array: true })
const rawItems = (this.params as { items?: { type?: string }[] }).items ?? []

for (const [i, params] of itemsParams.entries()) {
  const roomType = rawItems[i]?.type as RoomTypesEnum
  switch (roomType) {
    case 'Bathroom':   await Bathroom.create({ place: this.currentPlace, ...params }); break
    case 'Bedroom':    await Bedroom.create({ place: this.currentPlace, ...params }); break
    case 'Kitchen':    await Kitchen.create({ place: this.currentPlace, ...params }); break
    case 'Den':        await Den.create({ place: this.currentPlace, ...params }); break
    case 'LivingRoom': await LivingRoom.create({ place: this.currentPlace, ...params }); break
    default: {
      const _never: never = roomType
      throw new Error(`Unhandled RoomTypesEnum: ${_never as string}`)
    }
  }
}
```

`extractParams(STIParent, [...], { key, array: true })` strips `type` per item — extract it from the raw params and let the exhaustive switch (with a `_never` default) enforce validity. No separate enum-membership check is needed; the switch already exhausts valid values and the `_never` default catches anything else at compile time AND runtime. Wrap the loop in `ApplicationModel.transaction(async txn => { ... })` only when the action genuinely needs atomicity — extra transaction overhead isn't free.

## Testing Patterns

### STI Factories

Each child gets its own factory:

```typescript
// spec/factories/Room/BedroomFactory.ts
import Bedroom from '@models/Room/Bedroom.js'
import createPlace from '../PlaceFactory.js'
import { UpdateableProperties } from '@rvoh/dream/types'

export default async function createBedroom(
  attrs: UpdateableProperties<Bedroom> & { place?: Place } = {}
) {
  const place = attrs.place ?? await createPlace()
  return await Bedroom.create({
    place,
    bedTypes: ['queen'],
    ...attrs,
  })
}
```

### STI Model Specs

```typescript
describe('Bedroom', () => {
  it('is an STI child of Room', async () => {
    const bedroom = await createBedroom()
    expect(bedroom.type).toEqual('Bedroom')

    // Querying Room returns correct child type
    const room = await Room.findOrFail(bedroom.id)
    expect(room).toBeInstanceOf(Bedroom)
    expect((room as Bedroom).bedTypes).toEqual(['queen'])
  })

  it('only returns Bedrooms when querying Bedroom class', async () => {
    const bedroom = await createBedroom()
    const kitchen = await createKitchen()

    const bedrooms = await Bedroom.all()
    expect(bedrooms).toMatchDreamModels([bedroom])
    expect(bedrooms).not.toMatchDreamModels([kitchen])
  })
})
```

### STI Controller Specs

```typescript
describe('POST /v1/host/places/:placeId/rooms', () => {
  it('creates a Bedroom', async () => {
    const { body } = await request.post(
      `/v1/host/places/${place.id}/rooms`, 201,
      { data: { type: 'Bedroom', bedTypes: ['queen', 'twin'] } }
    )
    expect(body.type).toEqual('Bedroom')
    expect(body.bedTypes).toEqual(['queen', 'twin'])

    const room = await place.associationQuery('rooms').firstOrFail()
    expect(room).toBeInstanceOf(Bedroom)
  })

  it('creates a Bathroom', async () => {
    const { body } = await request.post(
      `/v1/host/places/${place.id}/rooms`, 201,
      { data: { type: 'Bathroom', bathOrShowerStyle: 'bath_and_shower' } }
    )
    expect(body.type).toEqual('Bathroom')
  })
})
```

## Complete Generation Workflow

```bash
# 1. Generate parent resource with --sti-base-serializer
pnpm psy g:resource --sti-base-serializer --owning-model=Place \
  v1/host/places/{}/rooms Room \
  type:enum:room_types:Bathroom,Bedroom,Kitchen,Den,LivingRoom \
  Place:belongs_to position:integer:optional deleted_at:datetime:optional

# 2. Migrate parent table
pnpm psy db:migrate

# 3. Generate each child
pnpm psy g:sti-child --model-name=Bathroom Room/Bathroom extends Room \
  bath_or_shower_style:enum:bath_or_shower_styles:bath,shower,bath_and_shower,none
pnpm psy db:migrate

pnpm psy g:sti-child --model-name=Bedroom Room/Bedroom extends Room \
  bed_types:enum[]:bed_types:twin,bunk,queen,king,cot,sofabed
pnpm psy db:migrate

pnpm psy g:sti-child --model-name=Kitchen Room/Kitchen extends Room \
  appliances:enum[]:appliance_types:stove,oven,microwave,dishwasher
pnpm psy db:migrate

pnpm psy g:sti-child --model-name=Den Room/Den extends Room
pnpm psy g:sti-child --model-name=LivingRoom Room/LivingRoom extends Room

# 4. Migrate remaining children and sync types
pnpm psy db:migrate
pnpm psy sync

# 5. Update controller create action with switch statement for all types
# 6. Add @Sortable and deferrable unique constraint if needed
# 7. Write specs
# 8. Run specs: pnpm uspec
```
