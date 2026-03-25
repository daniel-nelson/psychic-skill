---
name: dream-psychic
description: >
  Comprehensive guide for developing applications with Dream ORM and Psychic web framework.
  TRIGGER when: code imports from '@rvoh/dream', '@rvoh/psychic', '@rvoh/psychic-workers', or '@rvoh/psychic-websockets',
  or project has Dream models, Psychic controllers, or uses 'psy' commands.
  Covers models, associations, validations, hooks, scopes, serializers, controllers, routing,
  migrations, background workers, websockets, OpenAPI, testing, and code generation.
user-invocable: false
allowed-tools: Read, Grep, Glob, Bash, Edit, Write
---

# Dream ORM & Psychic Web Framework Development Guide

You are working with **Dream** (a TypeScript Active Record ORM) and **Psychic** (a batteries-included TypeScript web framework built on Koa). Both are open source packages published under the `@rvoh` npm scope.

**Update check**: !`for d in "${CLAUDE_SKILL_DIR:-}" "${CODEX_SKILL_DIR:-}" "$HOME/.codex/skills/psychic-skill" "$HOME/.claude/skills/psychic-skill" ".codex/skills/psychic-skill" ".claude/skills/psychic-skill"; do [ -n "$d" ] && [ -x "$d/bin/psychic-skill-update-check" ] && "$d/bin/psychic-skill-update-check" 2>/dev/null && exit 0; done`
If the output above says `UPGRADE_AVAILABLE <old> <new>`, follow the inline upgrade flow in `/psychic-update-skill`.
If the output says `JUST_UPGRADED <old> <new>`, tell the user: "psychic-skill upgraded from v{old} to v{new}!" and continue.

All CLI commands in this document are run via the local project's package manager (e.g., `pnpm psy sync`, `yarn psy sync`, `npm run psy sync`). Examples use `pnpm` but substitute the project's actual package manager.

**Note on examples:** Code examples throughout this skill use BearBnB, a demo app that creates an AirBnB clone for bears (https://github.com/daniel-nelson/bearbnb). In this domain, **Guest** and **Host** are application roles (a Guest books a place to stay, a Host lists a place) — not to be confused with "visitor" (unauthenticated user) or "server" (the machine).

## Critical Rules

1. **ALWAYS read the project's `AGENTS.md` or `CLAUDE.md` first** before doing any work - these contain project-specific conventions that override general patterns.
2. **ALWAYS run `pnpm psy <command> --help`** before using any generator - never guess syntax.
3. **NEVER use JavaScript `Date`** - always use `DateTime` or `CalendarDate` from `@rvoh/dream`.
4. **NEVER stub or mock Dream internals** in tests - use factories to create real model instances.
5. **NEVER modify an existing migration file that has already been merged into main.**
6. **A generator must always be used** when creating new models, controllers, or migrations.
7. **Sources of truth** (priority order): TSDocs > `pnpm psy <command> --help` > psychic-skill > MCP server (dream-psychic-rag).
8. **BDD approach**: Write failing spec first, then implement. Generated code is the only exception (generators create scaffolding for specs and implementation simultaneously).
9. **Run `pnpm psy sync`** after changing associations, serializers, OpenAPI decorators, or routes.
10. **Only use comments to explain "why", not "what"** - prefer expressive code over comments. TSDoc comments explaining "how" or "when" to use a function/method/class are welcome.
11. **Use Dream's built-in utilities** (`@rvoh/dream/utils`) instead of lodash or hand-rolled equivalents. See [utils.md](utils.md) for the full list.

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

**CRITICAL: All `pnpm psy` commands default to `NODE_ENV=test`.** To operate on the development database, prefix with `NODE_ENV=development`. See [console.md](console.md) for details.

```bash
pnpm psy db:migrate              # Run migrations, then sync
pnpm psy db:rollback             # Rollback last run migration (use --steps to specify multiple rollback steps)
pnpm psy db:reset                # Drop + create + migrate, then sync
pnpm psy sync                    # Sync types, OpenAPI specs, and cli:sync commands
pnpm psy routes                  # Display all routes
pnpm psy --help                  # List all psy commands

# Console (for interactive exploration and scripts)
NODE_ENV=development pnpm console   # Launch Dream console against dev DB

# Generators (ALWAYS run --help first)
pnpm psy g:resource path/Name ModelName field:type   # Preferred for HTTP-accessible models
pnpm psy g:model ModelName field:type                # When model won't be HTTP-accessible
pnpm psy g:controller Path/Name action1 action2
pnpm psy g:migration description                     # For schema changes without a new model
pnpm psy g:sti-child Model/Child extends Parent field:type

# Testing & Quality
pnpm uspec                       # Unit specs
pnpm fspec                       # Feature specs (headless)
pnpm fspec:visible               # Feature specs (visible browser)
pnpm build:spec                  # Check for type errors
pnpm format                      # Apply standard formatting
pnpm lint                        # Check linting
```

## Generator Usage Rules

- **Generator preference order**:
  1. **Resource generator** (`g:resource`) - preferred when a model may be manipulated via HTTP requests
  2. **STI-child generator** (`g:sti-child`) - for STI child models building on an existing STI base
  3. **Model generator** (`g:model`) - when a new model won't be HTTP-accessible
  4. **Migration generator** (`g:migration`) - for database changes when not generating a new model
- **When in doubt, prefer `g:resource`**. If the model's data will ever be created, edited, or viewed through any UI or API (including admin/internal tools), it needs a controller. `g:resource` generates the controller, specs, and serializer scaffolding that `g:model` does not. It is easier to delete unused controller actions than to retrofit them later.
- **CRITICAL: ALWAYS run `pnpm psy <command> --help` before running any generator**

### Generator Workflow

After a generator has run:

1. **Update the migration file as needed** (e.g., add `unique()` to a column)
2. **Run migrations**: `pnpm psy db:migrate`
3. **If the generator was a resource generator**:
   - Update the generated controller spec first
   - Then update the corresponding generated controller
   - Note: controller specs will hang if there is no response within a controller action (generated action code starts commented out)
4. **Commit generated code as its own commit** with a message in this format:
   ```
   Generate Room resource

   ```console
   pnpm psy g:resource --sti-base-serializer --owning-model=Place v1/host/places/rooms Room type:enum:room_types:Bathroom,Bedroom,Kitchen,Den
   ```
   ```

### When to Run `pnpm psy sync`

Run `pnpm psy sync` whenever any of the following are added or changed:
- An association in a Dream model
- A serializer
- An OpenAPI decorator on a controller action
- A route

If controller specs have type errors about what an endpoint accepts or returns, this means the OpenAPI shape is out of sync - run `pnpm psy sync`.

## Adding Properties to Existing Models

- **Always use the migration generator** (`pnpm psy g:migration`) to create a migration for schema changes
- The generator scaffolding can be modified to make other database changes using DreamMigrationHelpers or Kysely-native calls
- Prefer a DreamMigrationHelpers method over compound Kysely calls when one is available
- After generating and running the migration, manually add the property declaration to the model file
- Example: To add a `timezone` field to User, run `pnpm psy g:migration add_timezone_to_users`, edit the generated migration, then add `public timezone: DreamColumn<User, 'timezone'>` to the User model

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

  // Note: @SoftDelete() above automatically adds a default scope (dream:SoftDelete)
  // that hides records where deletedAt is not null. No manual default scope needed.

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
await place.undestroy()       // restore soft-deleted record
await place.reallyDestroy()   // permanent delete

// Find-or-create / upsert (see models.md for full reference)
await User.findOrCreateBy({ email }, { createWith: { name } })       // find first, create if missing
await User.createOrFindBy({ email }, { createWith: { name } })       // create first (needs unique constraint, no txn)
await User.updateOrCreateBy({ email }, { with: { name } })           // find & update, or create
await User.createOrUpdateBy({ email }, { with: { name } })           // create first (needs unique constraint, no txn)

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

// Preloading (prevents N+1) — see querying.md for full association chaining docs
// IMPORTANT: multiple string args form a CHAIN, not parallel loads
const places = await Place.preload('rooms').all()                  // preload rooms on each place
const places = await Place.preload('rooms').preload('hosts').all() // preload rooms AND hosts (parallel)
const users = await User.preload('posts', 'comments').all()        // chain: posts → comments (nested)
const places = await Place.preloadFor('summary').all()              // serializer-driven (preferred)

// Pagination
const result = await Place.cursorPaginate({ cursor: cursorString })
// Returns: { cursor: string | null, results: Place[] }

// Transactions — EVERY operation inside must use .txn(txn)
// If you forget .txn(txn), the operation runs outside the transaction
await ApplicationModel.transaction(async txn => {
  const place = await Place.txn(txn).create({ ... })
  await HostPlace.txn(txn).create({ host, place })
  const found = await Place.txn(txn).findBy({ name: 'x' }) // queries too
})
```

### Querying Beyond Dream

For detailed query guidance, including when to use `toKysely(...)` and when to start from the project's typed `db()` entrypoint, see [querying.md](querying.md).

The rule is simple: always prefer Dream's built in, public query and association APIs; only drop down to Kysely when Dream's supported APIs do not cover the SQL you need.

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

For detailed migration patterns (column types, helpers, etc.), see [migrations.md](migrations.md).

**Migrations are always created via generators** (`g:resource`, `g:model`, `g:sti-child`, or `g:migration`), then edited as needed. Never create migration files by hand.

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

## Soft Delete

For detailed soft delete patterns, see [soft-delete.md](soft-delete.md).

### Quick Reference

The `@SoftDelete()` decorator makes `destroy()` set `deletedAt` instead of removing the row. A default scope (`dream:SoftDelete`) automatically hides soft-deleted records from all queries.

```typescript
import { SoftDelete } from '@rvoh/dream'

@SoftDelete()
export default class Place extends ApplicationModel {
  public deletedAt: DreamColumn<Place, 'deletedAt'>
}
```

```typescript
await place.destroy()                  // Sets deletedAt (soft delete)
await place.undestroy()                // Restores (sets deletedAt to null)
await place.reallyDestroy()            // Permanent delete

// Query multiple soft-deleted records — removeDefaultScope preserves other scopes
await Place.removeDefaultScope('dream:SoftDelete').all()

// Find a specific soft-deleted record — removeAllDefaultScopes ensures it's found
await Place.removeAllDefaultScopes().findOrFail(id)
```

**Key rules:**
- Requires a `deleted_at` datetime column (nullable)
- Use `restrict` (the default) for foreign key constraints, not `cascade`
- **Every model in a `dependent: 'destroy'` chain MUST also have `@SoftDelete()`** — otherwise those associated records will be permanently deleted
- Both `destroy()` and `undestroy()` cascade through `dependent: 'destroy'` associations
- STI children cannot use `@SoftDelete()` — it must be on the STI parent

## Default Scopes

Default scopes are query conditions automatically applied to every query on a model. For full documentation, see [models.md - Default Scopes](models.md#default-scopes).

Dream provides built-in default scopes:
- **`dream:SoftDelete`** — added by `@SoftDelete()`, hides records where `deletedAt` is not null
- **`dream:STI`** — added by `@STI()`, restricts STI child queries to their type

Models can define custom default scopes with `@deco.Scope({ default: true })`. Use `removeDefaultScope('scopeName')` when querying for a plurality (preserves other scopes). Use `removeAllDefaultScopes()` when targeting a specific record.

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

**STI limitations**: STI children cannot use `@SoftDelete()` (must be on parent) and cannot use `@ReplicaSafe()`.

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

For the full reference including all options and option tables, see [models.md](models.md).

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

// With conditions (and, andAny, andNot)
@deco.HasMany('Post', { and: { published: true } })
public publishedPosts: Post[]

@deco.HasMany('Post', { andAny: [{ featured: true }, { published: true }] })
public visiblePosts: Post[]

@deco.HasMany('Post', { andNot: { archived: true } })
public activePosts: Post[]

// selfAnd — join where associated column matches a column on THIS model
// Format: { associatedColumn: 'thisModelColumn' }
// Links to DailyChallenge by position instead of FK, allowing re-shuffling
@deco.HasOne('DailyChallenge', {
  on: 'position',
  selfAnd: { position: 'currentPosition' },
})
public currentChallenge: DailyChallenge

// selfAndNot — exclude where columns match (e.g., siblings = parent's children minus self)
@deco.HasMany('TreeNode', {
  through: 'parent', source: 'children',
  selfAndNot: { id: 'id' },
})
public siblings: TreeNode[]

// DreamConst.passthrough — value supplied at query time via .passthrough({ locale: 'en-US' })
@deco.HasOne('LocalizedText', {
  polymorphic: true, on: 'localizableId',
  and: { locale: DreamConst.passthrough },
})
public currentLocalizedText: LocalizedText

// DreamConst.required — value supplied via { and: { category: value } } in the loading chain
@deco.HasOne('Preference', {
  and: { category: DreamConst.required },
})
public preference: Preference

// With order and distinct
@deco.HasMany('Tag', { through: 'postTags', distinct: true, order: 'name' })
public tags: Tag[]

// Skip default scopes when loading
@deco.HasMany('Post', { withoutDefaultScopes: ['dream:SoftDelete'] })
public allPosts: Post[]

// Override primary key used for join
@deco.BelongsTo('Widget', { primaryKeyOverride: 'legacyId' })
public widget: Widget
```

### Association Option Summary

**BelongsTo**: `on`, `optional`, `polymorphic`, `primaryKeyOverride`, `withoutDefaultScopes`

**HasMany / HasOne**: `and`, `andAny`, `andNot`, `dependent`, `on`, `order` (HasMany only), `distinct` (HasMany only), `polymorphic`, `primaryKeyOverride`, `selfAnd`, `selfAndNot`, `through`, `source`, `withoutDefaultScopes`

**Through associations** cannot use: `dependent`, `primaryKeyOverride`, `withoutDefaultScopes`, `on`, or `polymorphic`.

## Internationalization (i18n)

For detailed i18n patterns, see [i18n.md](i18n.md). Psychic supports two complementary patterns:

- **Code-driven i18n** - Translate enum values and static labels using `I18nProvider` from `@rvoh/psychic/system` with locale files in `src/conf/locales/`
- **Data-driven i18n** - Translate user-generated content using a polymorphic `LocalizedText` model with `DreamConst.passthrough` for locale-conditioned associations

Both use the `Accept-Language` request header, passed to serializers via `serializerPassthrough({ locale })`.

## Background Workers

For detailed worker patterns (services, models, scheduling, configuration, testing), see [workers.md](workers.md).

### Quick Reference

Backgrounded services use a two-method pattern: a public entry method that exposes an expressive API to callers, and a private implementation method (prefixed with `_`) that does the work. This encapsulates backgrounding as an implementation detail — callers don't need to know the work is backgrounded. **Never pass model data as arguments** — pass only IDs and look up the record in the implementation method. Passing model data bloats Redis, loses type information when serialized to JSON, and creates immediately stale snapshots.

```typescript
import ApplicationBackgroundedService from './ApplicationBackgroundedService.js'

export class NotificationService extends ApplicationBackgroundedService {
  // Public entry point — called from application code
  public static async sendWelcome(user: User) {
    await this.background('_sendWelcome', user.id)
  }

  // Private implementation — called by the worker
  public static async _sendWelcome(userId: string) {
    // Use find (not findOrFail) — the record may have been deleted since the job was queued
    const user = await User.find(userId)
    // Exit if target model no longer exists
    if (!user) return
    // Welcome notication logic
  }

  // Optional: configure priority/workstream
  public static override get backgroundJobConfig() {
    return { priority: 'urgent' }
  }
}

// Called from application code (e.g., controller or model hook):
await NotificationService.sendWelcome(user)

// IMPORTANT: When backgrounding from a Dream model lifecycle hook, the `Commit` variant of the hook **MUST** be used, e.g., AfterCreateCommit **NOT** AfterCreate:
@deco.AfterCreateCommit()
public async notifyCreation(this: Place) {
  await NotificationService.placeCreated(this)
}
```

## Websockets

For detailed websocket patterns, see [websockets.md](websockets.md).

### Quick Reference

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
| Generator columns | snake_case | `name:string`, `User:belongs_to` |
| Enum types (DB) | snake_case + `_enum` | `place_styles_enum` |
| Enum values | snake_case | `lean_to`, `bath_and_shower` |
| STI type values | PascalCase (MUST match STI child class names) | `Bedroom`, `LivingRoom` |
| `date` columns | suffix `On` | `occurred_on`, `started_on`, `born_on` |
| `datetime` columns | suffix `At` | `occurred_at`, `started_at`, `deleted_at` |

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

## Troubleshooting Migrations

**"Corrupted migrations" error**: When switching between branches with different migrations, `pnpm psy db:migrate` fails with "corrupted migrations" errors. The fix is `pnpm psy db:reset`.
