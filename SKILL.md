---
name: dream-psychic
description: >
  Comprehensive guide for developing applications with Dream ORM and Psychic web framework.
  TRIGGER when: code imports from '@rvoh/dream', '@rvoh/psychic', '@rvoh/psychic-workers', or '@rvoh/psychic-websockets',
  or project has Dream models, Psychic controllers, or uses 'pnpm psy' commands.
  Covers models, associations, validations, hooks, scopes, serializers, controllers, routing,
  migrations, background workers, websockets, OpenAPI, testing, and code generation.
user-invocable: false
allowed-tools: Read, Grep, Glob, Bash, Edit, Write
---

# Dream ORM & Psychic Web Framework Development Guide

You are working with **Dream** (a TypeScript Active Record ORM) and **Psychic** (a batteries-included TypeScript web framework built on Koa). Both are open source packages published under the `@rvoh` npm scope.

## Critical Rules

1. **ALWAYS read the project's `AGENTS.md` or `CLAUDE.md` first** before doing any work - these contain project-specific conventions that override general patterns.
2. **ALWAYS run `pnpm psy <command> --help`** before using any generator - never guess syntax.
3. **NEVER use JavaScript `Date`** - always use `DateTime` or `CalendarDate` from `@rvoh/dream`.
4. **NEVER stub or mock Dream internals** in tests - use factories to create real model instances.
5. **Sources of truth** (priority order): TSDocs > `pnpm psy <command> --help` > existing code patterns.
6. **Use `pnpm`** as the package manager (not npm or yarn).
7. **BDD approach**: Write failing spec first, then implement.
8. **Run `pnpm psy sync`** after changing associations or model structure to regenerate types.

## Project Structure

```
api/
  src/
    app/
      models/           # Dream models (ApplicationModel base)
      controllers/      # Psychic controllers
      serializers/      # Response serializers (DreamSerializer / ObjectSerializer)
      services/         # Business logic, backgrounded services
      view-models/      # Complex data transformation models
    conf/
      app.ts            # Psychic app config
      dream.ts          # Dream ORM config
      routes.ts         # Route definitions
      AppEnv.ts         # Typed environment variables
      initializers/     # Boot-time initializers (websockets, workers, etc.)
    db/
      migrations/       # Kysely migrations
    types/
      db.ts             # Auto-generated Kysely database types
      dream.ts          # Auto-generated Dream type config
  spec/
    unit/               # Unit tests (models, controllers)
    features/           # E2E tests
    factories/          # Test data factories
```

## Key Commands

```bash
pnpm psy db:migrate              # Run migrations
pnpm psy db:rollback             # Rollback last migration batch
pnpm psy db:reset                # Drop + create + migrate
pnpm psy sync                    # Sync types + OpenAPI specs
pnpm psy sync:openapi            # Regenerate OpenAPI only
pnpm psy routes                  # Display all routes

# Generators (ALWAYS run --help first)
pnpm psy g:model ModelName field:type
pnpm psy g:resource path/Name ModelName field:type
pnpm psy g:controller Path/Name action1 action2
pnpm psy g:migration description
pnpm psy g:sti-child Model/Child extends Parent field:type

# Testing
pnpm uspec                       # Unit specs
pnpm fspec                       # Feature specs (headless)
pnpm fspec:visible               # Feature specs (visible browser)
```

## Models

For detailed model patterns, see [models.md](models.md).

### Quick Reference

```typescript
import { Decorators, DreamConst, SoftDelete, STI } from '@rvoh/dream'
import { DreamColumn, DreamSerializers } from '@rvoh/dream/types'
import ApplicationModel from './ApplicationModel.js'

const deco = new Decorators<typeof Place>()

@SoftDelete()
export default class Place extends ApplicationModel {
  public override get table() {
    return 'places' as const
  }

  public get serializers(): DreamSerializers<Place> {
    return { default: 'PlaceSerializer', summary: 'PlaceSummarySerializer' }
  }

  // Columns - use DreamColumn<Model, 'columnName'> for type safety
  public id: DreamColumn<Place, 'id'>
  public name: DreamColumn<Place, 'name'>
  public style: DreamColumn<Place, 'style'>
  public sleeps: DreamColumn<Place, 'sleeps'>
  public createdAt: DreamColumn<Place, 'createdAt'>
  public updatedAt: DreamColumn<Place, 'updatedAt'>
  public deletedAt: DreamColumn<Place, 'deletedAt'>

  // Encrypted columns
  @deco.Encrypted()
  public phone: DreamColumn<Place, 'encryptedPhone'>

  // Virtual (non-database) columns
  @deco.Virtual('integer')
  public age: number

  // Associations
  @deco.HasMany('Room', { order: 'position', dependent: 'destroy' })
  public rooms: Room[]

  @deco.HasMany('HostPlace', { dependent: 'destroy' })
  public hostPlaces: HostPlace[]

  @deco.HasMany('Host', { through: 'hostPlaces' })
  public hosts: Host[]

  @deco.BelongsTo('User')
  public user: User
  public userId: DreamColumn<Place, 'userId'>

  // Hooks
  @deco.AfterCreate()
  public async createDefaultRoom(this: Place) {
    await this.createAssociation('rooms', { type: 'LivingRoom' })
  }

  @deco.AfterUpdateCommit({ ifChanged: ['status'] })
  public async notifyStatusChange(this: Place) {
    await NotificationService.background('placeStatusChanged', this.id)
  }

  // Validations
  @deco.Validates('presence')
  public declare name: DreamColumn<Place, 'name'>

  @deco.Validates('numericality', { min: 1, max: 50 })
  public declare sleeps: DreamColumn<Place, 'sleeps'>

  // Scopes
  @deco.Scope()
  public static active(query: Query<Place>) {
    return query.where({ status: 'active' })
  }

  @deco.Scope({ default: true })
  public static hideDeleted(query: Query<Place>) {
    return query.where({ deletedAt: null })
  }

  // Sortable (requires deferrable unique constraint in migration)
  @deco.Sortable({ scope: 'place' })
  public position: DreamColumn<Place, 'position'>
}
```

### Key Model Operations

```typescript
// Create
const place = await Place.create({ name: 'Cozy Cabin', style: 'cabin', sleeps: 4 })

// Find
const place = await Place.find(id)              // or null
const place = await Place.findOrFail(id)         // throws -> 404 in controller
const place = await Place.findBy({ name: 'x' }) // by attributes

// Update
await place.update({ name: 'New Name' })

// Destroy (soft delete if @SoftDelete)
await place.destroy()
await place.reallyDestroy()  // permanent

// Query
const places = await Place.where({ style: 'cabin' }).order('createdAt', 'desc').all()
const count = await Place.where({ style: 'cabin' }).count()
const exists = await Place.where({ id: placeId }).exists()

// Operators
import { ops } from '@rvoh/dream'
await Place.where({ name: ops.like('%cabin%') }).all()
await Place.where({ sleeps: ops.greaterThan(4) }).all()
await Place.whereAny([{ style: 'cabin' }, { style: 'cottage' }]).all()

// Associations
const rooms = await place.associationQuery('rooms').where({ type: 'Bedroom' }).all()
const room = await place.createAssociation('rooms', { type: 'Bedroom' })
await place.load('rooms').execute()

// Preloading (prevents N+1)
const places = await Place.preload('rooms').all()
const places = await Place.preloadFor('summary').all()  // based on serializer needs

// Pagination
const result = await Place.cursorPaginate({ cursor: cursorString })
// Returns: { cursor: string | null, results: Place[] }

// Transactions
await ApplicationModel.transaction(async txn => {
  const place = await Place.txn(txn).create({ ... })
  await HostPlace.txn(txn).create({ host, place })
})
```

## Controllers

For detailed controller patterns, see [controllers.md](controllers.md).

### Quick Reference

```typescript
import { BeforeAction, OpenAPI } from '@rvoh/psychic'

export default class V1HostPlacesController extends V1HostBaseController {
  @OpenAPI(Place, {
    status: 200,
    tags: ['places'],
    cursorPaginate: true,
    serializerKey: 'summary',
    fastJsonStringify: true,
  })
  public async index() {
    const places = await this.currentHost
      .associationQuery('places')
      .preloadFor('summary')
      .cursorPaginate({ cursor: this.castParam('cursor', 'string', { allowNull: true }) })
    this.ok(places)
  }

  @OpenAPI(Place, { status: 201, tags: ['places'], fastJsonStringify: true })
  public async create() {
    let place = await ApplicationModel.transaction(async txn => {
      const place = await Place.txn(txn).create(this.paramsFor(Place))
      await HostPlace.txn(txn).create({ host: this.currentHost, place })
      return place
    })
    if (place.isPersisted) place = await place.loadFor('default').execute()
    this.created(place)
  }

  @OpenAPI(Place, { status: 200, tags: ['places'], fastJsonStringify: true })
  public async show() {
    this.ok(await this.place())
  }

  @OpenAPI(Place, { status: 204, tags: ['places'] })
  public async update() {
    await (await this.place()).update(this.paramsFor(Place))
    this.noContent()
  }

  @OpenAPI({ status: 204, tags: ['places'] })
  public async destroy() {
    await (await this.place()).destroy()
    this.noContent()
  }

  private async place() {
    return await this.currentHost
      .associationQuery('places')
      .preloadFor('default')
      .findOrFail(this.castParam('id', 'string'))
  }
}
```

### Controller Key Methods

| Method | Purpose |
|--------|---------|
| `this.params` | Merged URL + body + query params |
| `this.castParam('name', 'type', opts?)` | Validate and cast a single param. Types: `uuid`, `string`, `integer`, `bigint`, `date`, `datetime`, `number`, `enum`. Array variants: `uuid[]`, `string[]`, etc. Options: `{ allowNull: true, enum: [...] }` |
| `this.paramsFor(Model, opts?)` | Extract safe params for model. Options: `{ only: [...], including: [...], key: 'bodyKey' }` |
| `this.ok(data)` | 200 response |
| `this.created(data)` | 201 response |
| `this.noContent()` | 204 response |
| `this.unauthorized()` | 401 response |
| `this.forbidden()` | 403 response |
| `this.notFound()` | 404 response |
| `this.serializerPassthrough(obj)` | Pass context (e.g., locale) to serializers |
| `this.header('name')` | Read request header |
| `this.startSession(user)` / `this.endSession()` | Session management |

### @BeforeAction Pattern

```typescript
export default class AuthedController extends ApplicationController {
  protected currentUser: User

  @BeforeAction()
  protected async authenticate() {
    const userId = this.authedUserId()
    if (!userId) return this.unauthorized()
    const user = await User.find(userId)
    if (!user) return this.unauthorized()
    this.currentUser = user
  }

  // With only/except filters
  @BeforeAction({ only: ['create', 'update'] })
  protected validatePermissions() { ... }

  @BeforeAction({ except: ['index'] })
  protected loadResource() { ... }
}
```

## Serializers

For detailed serializer patterns, see [serializers.md](serializers.md).

### Quick Reference

Serializers are **function-based** (not class-based or decorator-based):

```typescript
import { DreamSerializer, ObjectSerializer } from '@rvoh/dream'

// Summary serializer (base layer)
export const PlaceSummarySerializer = (place: Place) =>
  DreamSerializer(Place, place)
    .attribute('id')
    .attribute('name')

// Default serializer (extends summary)
export const PlaceSerializer = (place: Place) =>
  PlaceSummarySerializer(place)
    .attribute('style')
    .attribute('sleeps')
    .attribute('deletedAt')
    .rendersMany('rooms', { serializerKey: 'summary' })

// Serializer with passthrough context
export const PlaceForGuestsSerializer = (place: Place, passthrough: { locale: LocalesEnum }) =>
  PlaceSummarySerializer(place)
    .customAttribute('displayStyle', () => i18n(passthrough.locale, `places.style.${place.style}`), {
      openapi: 'string',
    })
    .delegatedAttribute('currentLocalizedText', 'title', { openapi: 'string' })
    .rendersMany('rooms', { serializerKey: 'forGuests' })

// ObjectSerializer for non-Dream objects
export const BedTypeSerializer = (bedType: BedTypesEnum, passthrough: { locale: LocalesEnum }) =>
  ObjectSerializer({ bedType }, passthrough)
    .attribute('bedType', { as: 'value', openapi: { type: 'string', enum: BedTypesEnumValues } })
    .customAttribute('label', () => i18n(passthrough.locale, `rooms.Bedroom.bedTypes.${bedType}`), {
      openapi: 'string',
    })
```

### Serializer Methods

| Method | Purpose |
|--------|---------|
| `.attribute('name')` | Include a model column |
| `.attribute('name', { as: 'alias' })` | Rename in output |
| `.attribute('name', { precision: 2 })` | Decimal precision |
| `.attribute('name', { openapi: { type: 'string', enum: [...] } })` | OpenAPI type hint |
| `.customAttribute('name', fn, { openapi: 'string' })` | Computed field |
| `.delegatedAttribute('assocName', 'field', { openapi: 'string' })` | Nested field |
| `.rendersOne('assoc')` | Single association |
| `.rendersOne('assoc', { serializerKey: 'summary' })` | With specific serializer |
| `.rendersMany('assoc')` | Array association |
| `.rendersMany('items', { serializer: CustomFn })` | Custom serializer function |

### STI Serializer Pattern

```typescript
// Base room serializer (generic)
export const RoomSummarySerializer = <T extends Room>(StiChildClass: typeof Room, room: T) =>
  DreamSerializer(StiChildClass ?? Room, room)
    .attribute('id')
    .attribute('type', { openapi: { type: 'string', enum: [(StiChildClass ?? Room).sanitizedName] } })

// STI child serializer
export const RoomBedroomSerializer = (bedroom: Bedroom) =>
  RoomSerializer(Bedroom, bedroom)
    .attribute('bedTypes')
```

## Migrations

For detailed migration patterns, see [migrations.md](migrations.md).

### Quick Reference

```typescript
import { DreamMigrationHelpers } from '@rvoh/dream/db'
import { Kysely, sql } from 'kysely'

export async function up(db: Kysely<any>): Promise<void> {
  // Extensions
  await DreamMigrationHelpers.createExtension(db, 'citext')

  // Create enum
  await db.schema.createType('place_styles_enum')
    .asEnum(['cottage', 'cabin', 'treehouse', 'tent', 'cave'])
    .execute()

  // Create table
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
  await db.schema.dropType('place_styles_enum').execute()
}
```

### Common Column Types

| Type | Kysely Syntax | Notes |
|------|--------------|-------|
| UUID PK | `'uuid', col => col.primaryKey().defaultTo(sql\`uuidv7()\`)` | Default PK pattern |
| bigserial PK | `'bigserial', col => col.primaryKey()` | Alternative PK |
| String | `'varchar'` or `'varchar(128)'` | With optional length |
| Case-insensitive | `sql\`citext\`` | Requires citext extension |
| Text | `'text'` | Long text |
| Integer | `'integer'` | |
| Decimal | `sql\`decimal(10, 2)\`` | With precision |
| Boolean | `'boolean'` | |
| Timestamp | `'timestamp'` | For datetime columns |
| Date | `'date'` | Date only |
| Enum | `sql\`enum_name\`` | Must create type first |
| Enum array | `sql\`enum_name[]\`, col => col.defaultTo('{}')` | Array of enums |
| JSON | `'jsonb'` | Prefer jsonb over json |
| FK reference | `col => col.references('table.column').onDelete('restrict')` | Foreign keys |

### Migration Helpers

```typescript
// Deferrable unique constraint (required for @Sortable)
await DreamMigrationHelpers.addDeferrableUniqueConstraint(db, 'constraint_name', {
  table: 'rooms', columns: ['place_id', 'position'],
})

// GIN index for full-text search
await DreamMigrationHelpers.createGinIndex(db, 'index_name', {
  table: 'users', column: 'name',
})

// Add value to existing enum
await DreamMigrationHelpers.addEnumValue(db, { enumName: 'styles_enum', value: 'mansion' })

// Drop enum value (complex - requires data migration)
await DreamMigrationHelpers.dropEnumValue(db, {
  enumName: 'styles_enum', value: 'dump',
  replacements: [{ table: 'places', column: 'style', replaceWith: 'cave' }],
})
```

## Routing

```typescript
import { PsychicRouter } from '@rvoh/psychic'

export default function routes(r: PsychicRouter) {
  // Namespace groups routes and infers controller paths
  r.namespace('v1', r => {
    r.namespace('host', r => {
      // Full CRUD: index, show, create, update, destroy
      r.resources('places', r => {
        r.resources('rooms')  // Nested: /v1/host/places/:placeId/rooms
      })
      r.resources('localized-texts', { only: ['update', 'destroy'] })
    })

    r.namespace('guest', r => {
      r.resources('places', { only: ['index', 'show'] })
    })
  })

  // Simple routes
  r.get('ping', PingController, 'ping')
  r.post('login', AuthController, 'login')

  // Singular resource (no index, no :id in path)
  r.resource('profile', { only: ['show', 'update'] })

  // Collection routes (no :id)
  r.resources('items', r => {
    r.collection(r => {
      r.post('bulk-create', ItemsController, 'bulkCreate')
    })
  })
}
```

## Single Table Inheritance (STI)

STI allows multiple model types to share a single database table. For detailed generation, model, serializer, migration, and controller patterns see [sti.md](sti.md)

### Quick Reference

```typescript
// Parent model
export default class Room extends ApplicationModel {
  public override get table() { return 'rooms' as const }
  public type: DreamColumn<Room, 'type'>
}

// Child model
@STI(Room)
export default class Bedroom extends Room {
  public bedTypes: DreamColumn<Bedroom, 'bedTypes'>
  public override get serializers(): DreamSerializers<Bedroom> {
    return { default: 'Room/BedroomSerializer', summary: 'Room/BedroomSummarySerializer' }
  }
}
```

**STI limitations**: STI children cannot define new associations, cannot use `@SoftDelete()` (must be on parent), and cannot use `@ReplicaSafe()`.

**Creating STI instances in controllers** - MUST use switch on type:

```typescript
const roomType = this.castParam('type', 'string', { enum: RoomTypesEnumValues })
let room: Room
switch (roomType) {
  case 'Bathroom': room = await Bathroom.create({ place, ...params }); break
  case 'Bedroom':  room = await Bedroom.create({ place, ...params }); break
  case 'Kitchen':  room = await Kitchen.create({ place, ...params }); break
  default: { const _never: never = roomType; throw new Error(`Unhandled: ${_never}`) }
}
```

## Associations

### Types

```typescript
// BelongsTo (FK on this model)
@deco.BelongsTo('User')
public user: User
public userId: DreamColumn<Post, 'userId'>

// BelongsTo optional
@deco.BelongsTo('User', { optional: true })
public approver: User | null

// HasMany
@deco.HasMany('Post', { dependent: 'destroy' })
public posts: Post[]

// HasOne
@deco.HasOne('Profile')
public profile: Profile

// Through (many-to-many via join table)
@deco.HasMany('Host', { through: 'hostPlaces' })
public hosts: Host[]

// Polymorphic HasMany
@deco.HasMany('LocalizedText', { polymorphic: true, on: 'localizableId', dependent: 'destroy' })
public localizedTexts: LocalizedText[]

// Polymorphic BelongsTo
@deco.BelongsTo(['Host', 'Place', 'Room'], { polymorphic: true, on: 'localizableId' })
public localizable: Host | Place | Room

// With conditions
@deco.HasOne('LocalizedText', {
  polymorphic: true, on: 'localizableId',
  and: { locale: DreamConst.passthrough },
})
public currentLocalizedText: LocalizedText

// With order and distinct
@deco.HasMany('Tag', { through: 'postTags', distinct: true, order: 'name' })
public tags: Tag[]
```

## Background Workers

```typescript
import { ApplicationBackgroundedService } from './ApplicationBackgroundedService.js'

export class EmailService extends ApplicationBackgroundedService {
  public static async sendWelcome(userId: string) { ... }

  public static override get backgroundJobConfig() {
    return { priority: 'urgent' }
  }
}

// Queue a background job
await EmailService.background('sendWelcome', user.id)

// With delay
await EmailService.backgroundWithDelay({ seconds: 300 }, 'sendWelcome', user.id)

// IMPORTANT: Use AfterCreateCommit/AfterUpdateCommit hooks before backgrounding
@deco.AfterCreateCommit()
public async notifyCreation(this: Place) {
  await NotificationService.background('placeCreated', this.id)
}
```

## Websockets

```typescript
import { Ws } from '@rvoh/psychic-websockets'

// Define typed channels
const ws = new Ws(['/notifications/new', '/data/update'] as const)

// Emit to user
await ws.emit(userId, '/notifications/new', { message: 'Hello' })

// Register on connection (in conf/initializers/websockets.ts)
wsApp.on('ws:start', io => {
  io.of('/').on('connection', async socket => {
    const user = await User.find(extractUserId(socket))
    if (user) await Ws.register(socket, user.id)
  })
})
```

## Testing

For detailed testing patterns, see [testing.md](testing.md).

### Factory Pattern

```typescript
// spec/factories/PlaceFactory.ts
let counter = 0
export default async function createPlace(attrs: UpdateableProperties<Place> = {}) {
  return await Place.create({
    name: `Place name ${++counter}`,
    style: 'cottage',
    sleeps: 1,
    ...attrs,
  })
}
```

### Controller Spec Pattern

```typescript
describe('V1/Host/PlacesController', () => {
  let request: SpecRequestType
  let user: User
  let host: Host

  beforeEach(async () => {
    user = await createUser()
    host = await createHost({ user })
    request = await session(user)
  })

  describe('POST create', () => {
    it('creates a Place', async () => {
      const { body } = await request.post('/v1/host/places', 201, {
        data: { name: 'Cozy Cabin', style: 'cabin', sleeps: 4 },
      })
      const place = await host.associationQuery('places').firstOrFail()
      expect(place.name).toEqual('Cozy Cabin')
      expect(body).toEqual(expect.objectContaining({ id: place.id }))
    })
  })

  describe('GET index', () => {
    it('returns only this host places', async () => {
      const place = await createPlace()
      await createHostPlace({ host, place })
      const otherPlace = await createPlace()  // Not associated

      const { body } = await request.get('/v1/host/places', 200)
      expect(body.results).toHaveLength(1)
      expect(body.results[0].id).toEqual(place.id)
    })
  })
})
```

### Model Spec Pattern

```typescript
describe('Place', () => {
  describe('associations', () => {
    it('has many rooms', async () => {
      const place = await createPlace()
      const room = await createBedroom({ place })
      expect(await place.associationQuery('rooms').all()).toMatchDreamModels([room])
    })
  })

  describe('soft delete', () => {
    it('soft deletes and cascades', async () => {
      const place = await createPlace()
      await place.destroy()
      expect(await Place.where({ id: place.id }).exists()).toBe(false)
      expect(await Place.where({ id: place.id }).removeAllDefaultScopes().exists()).toBe(true)
    })
  })
})
```

## Generator Workflow

1. Run generator: `pnpm psy g:resource v1/host/places Place name:citext style:enum:place_styles:cottage,cabin sleeps:integer`
2. Review and adjust migration if needed
3. `pnpm psy db:migrate`
4. `pnpm psy sync` (regenerate types)
5. Write controller spec
6. Implement controller logic
7. Run specs: `pnpm uspec`

## Naming Conventions

| Context | Convention | Example |
|---------|-----------|---------|
| Database columns | snake_case | `created_at`, `user_id` |
| Model properties | camelCase | `createdAt`, `userId` |
| Model classes | PascalCase | `HostPlace`, `LocalizedText` |
| Controller files | PascalCase | `PlacesController.ts` |
| Serializer files | PascalCase | `PlaceSerializer.ts` |
| Migration files | timestamp-kebab | `1773151915966-create-place.ts` |
| Route paths | kebab-case | `localized-texts`, `admin-users` |
| Generator columns | snake_case | `name:string`, `user_id:uuid` |
| Enum types (DB) | snake_case + `_enum` | `place_styles_enum` |
| Enum values | snake_case | `lean_to`, `bath_and_shower` |
| STI type values | PascalCase | `Bedroom`, `LivingRoom` |

## OpenAPI Integration

Every controller action should have an `@OpenAPI` decorator:

```typescript
@OpenAPI(Place, {
  status: 200,                          // HTTP status code
  tags: ['places'],                     // OpenAPI grouping
  description: 'List all places',       // Endpoint description
  serializerKey: 'summary',             // Which serializer variant
  many: true,                           // Response is array
  cursorPaginate: true,                 // Cursor pagination wrapper
  fastJsonStringify: true,              // Performance optimization
  query: {                              // Query parameters
    search: { required: false, schema: 'string' },
  },
  requestBody: { including: ['name'] }, // Request body fields
})
```

Automatic error handling:
- `castParam` invalid -> 400
- `findOrFail` not found -> 404
- Validation failure -> 422
