# Psychic Controllers - Detailed Reference

## Controller Hierarchy

The controller directory structure **is** the authentication and authorization architecture. Each directory branch has its own `AuthedController` (and optionally `UnauthedController`, `MaybeAuthedController`) at its root, and every controller in that branch extends downward from it. By looking at the `@BeforeAction` methods in any directory's base controller, you can know exactly what rules are in force for the entire subtree.

**The cardinal rule:** never introduce a looser authentication pattern deeper in a directory hierarchy. Authentication flows downhill — it gets stricter, never weaker.

### Directory Structure

Each application surface (client-facing, admin, internal) gets its own directory branch with its own auth base controllers:

```
controllers/
├── ApplicationController.ts          (base — defines psychicTypes and universal methods such as extracting Accept-Language header; sets openapi namespaces)
├── AuthedController.ts               (client auth — @BeforeAction, 401 if no user)
├── MaybeAuthedController.ts          (client auth — currentUser null if absent)
├── UnauthedController.ts             (no auth)
│
├── V1/                               (client API namespace)
│   ├── SignInController.ts           (extends UnauthedController)
│   ├── SignOutController.ts          (extends AuthedController)
│   ├── Visitor/                      (public browse with optional auth)
│   │   ├── BaseController.ts         (extends MaybeAuthedController)
│   │   └── PlacesController.ts       (extends Visitor/BaseController)
│   ├── Guest/                        (authed Guest endpoints)
│   │   ├── BaseController.ts         (extends AuthedController, loads currentGuest, 403 if current User has no Guest)
│   │   ├── PlacesController.ts       (extends Guest/BaseController)
│   │   └── FavoritesController.ts    (extends Guest/BaseController)
│   └── Host/                         (authed Host endpoints)
│       ├── BaseController.ts         (extends AuthedController, loads currentHost, 403 if current User has no Host)
│       ├── PlacesController.ts       (extends Host/BaseController)
│       └── Places/
│           ├── BaseController.ts     (extends Host/BaseController, loads currentPlace)
│           └── RoomsController.ts    (extends Host/Places/BaseController)
│
├── Admin/                            (admin surface — separate auth chain)
│   ├── AuthedController.ts           (admin auth — validates admin user; overrides openapi namespaces to: 'admin', 'tests')
│   ├── UnauthedController.ts         (admin unauthed; overrides openapi namespaces to: 'admin', 'tests')
│   ├── UsersController.ts            (extends Admin/AuthedController)
│   ├── CitiesController.ts           (extends Admin/AuthedController)
│   ├── SignInController.ts           (extends Admin/UnauthedController)
│   ├── SignOutController.ts          (extends Admin/AuthedController)
│   └── Cities/
│       ├── BaseController.ts         (extends Admin/AuthedController, loads currentCity)
│       └── TravelTimesController.ts  (extends Admin/Cities/BaseController)
│
├── Internal/                         (internal employee surface — separate auth chain)
│   ├── AuthedController.ts           (internal auth — validates internal user; overrides openapi namespaces to: 'internal', 'tests')
│   ├── UnauthedController.ts         (internal unauthed; overrides openapi namespaces to: 'internal', 'tests')
│   ├── PlacesController.ts           (extends Internal/AuthedController)
│   ├── SignInController.ts           (extends Internal/UnauthedController)
│   ├── SignOutController.ts          (extends Internal/AuthedController)
│   └── Places/
│       ├── BaseController.ts         (extends Internal/AuthedController, loads currentPlace)
│       ├── RoomsController.ts        (extends Internal/Places/BaseController)
```

### Key Principles

1. **Each surface has its own auth controllers.** `Admin/AuthedController` and `Internal/AuthedController` each extend `ApplicationController` directly — they do NOT extend the client `AuthedController`. Each authenticates its own user type (AdminUser, InternalUser, etc.).

2. **Every controller extends the base controller in its own directory** (or the authed/unauthed controller if at the top of its branch). Controllers never reach across branches or skip levels.

3. **Dedicated directories for each auth level.** If a surface needs both authenticated and unauthenticated endpoints, it has separate base controllers in the same directory (e.g., `Admin/AuthedController` and `Admin/UnauthedController`). If client-facing code needs both authed and maybe-authed endpoints, use separate directories (`V1/Guest/` and `V1/Visitor/`).

4. **Use generators** to create controllers. `pnpm psy g:resource` and `pnpm psy g:controller` set up the correct inheritance chain automatically.

### Verifying the Hierarchy

```bash
# Visually inspect the controller inheritance tree
pnpm psy inspect:controller-hierarchy

# CI check — exits with 1 if a controller extends too far up the tree
# or crosses into another branch
pnpm psy check:controller-hierarchy
```

### Routing Controllers When Directory Names Don't Map to URLs

Controller directory names reflect the auth architecture, not URL structure. When the directory name shouldn't appear in the URL (e.g., `Visitor/` is the auth context, but `/v1/visitor/places` isn't a natural public URL), use an explicit `controller` reference instead of a namespace:

```typescript
// BAD — "visitor" leaks into the URL: /v1/visitor/places
r.namespace('visitor', r => {
  r.resources('places', { only: ['index', 'show'] })
})

// GOOD — clean URL: /v1/places, controller explicitly specified
import V1VisitorPlacesController from '@controllers/V1/Visitor/PlacesController.js'
r.resources('places', { only: ['index', 'show'], controller: V1VisitorPlacesController })
```

The controller directory structure (`V1/Visitor/`) still enforces the auth inheritance chain, but the URL stays clean.

## ApplicationController

```typescript
import { PsychicController } from '@rvoh/psychic'
import psychicTypes from '@src/types/psychic.js'

export default class ApplicationController extends PsychicController {
  public override get psychicTypes() {
    return psychicTypes
  }
}
```

## Authentication Examples

### Client AuthedController

```typescript
import { BeforeAction } from '@rvoh/psychic'

export default class AuthedController extends ApplicationController {
  protected currentUser: User

  @BeforeAction()
  protected async authenticate() {
    const token = (this.header('authorization') ?? '').split(' ').at(-1)!
    const decrypted = Encrypt.decrypt(token, {
      algorithm: 'aes-256-gcm',
      key: AppEnv.string('APP_ENCRYPTION_KEY'),
    })
    const userId = JSON.parse(decrypted)?.userId
    if (!userId) return this.unauthorized()

    const user = await User.find(userId)
    if (!user) return this.unauthorized()
    this.currentUser = user
  }
}
```

### MaybeAuthedController

Attempts authentication but does NOT 401 on failure. Sets `currentUser` to null if no token or invalid token. Use for public endpoints that behave differently when logged in (e.g., showing favorite indicators on public browse pages).

Both `AuthedController` and `MaybeAuthedController` delegate to a shared `resolveCurrentUser(controller)` helper in `controllers/helpers/resolveCurrentUser.ts`. The only difference is what happens when the result is null (401 vs. silent null).

### Surface-Specific Auth (Admin, Internal)

Each surface authenticates its own user type independently and overrides `openapiNames` to route its endpoints into the correct OpenAPI spec:

```typescript
// Admin/AuthedController.ts — authenticates AdminUser
export default class AdminAuthedController extends ApplicationController {
  public static override get openapiNames(): PsychicOpenapiNames<ApplicationController> {
    return ['admin', 'tests']
  }

  protected currentAdminUser: AdminUser

  @BeforeAction()
  protected async authenticate() {
    // admin-specific auth logic (e.g., Firebase with domain restriction)
    ...
  }
}

// Internal/AuthedController.ts — authenticates InternalUser
export default class InternalAuthedController extends ApplicationController {
  public static override get openapiNames(): PsychicOpenapiNames<ApplicationController> {
    return ['internal', 'tests']
  }

  protected currentInternalUser: InternalUser

  @BeforeAction()
  protected async authenticate() {
    // internal-specific auth logic
    ...
  }
}
```

### OpenAPI Namespaces

Each surface's controllers are routed to separate OpenAPI spec files via `openapiNames`. This prevents internal or admin endpoints from being exposed when you publish a public API spec. Namespaces are configured in `src/conf/app.ts`:

```typescript
// Default (client-facing) — all controllers without an openapiNames override
psy.set('openapi', {
  outputFilepath: path.join('src', 'openapi', 'openapi.json'),
  validate: { requestBody: true, headers: true, query: true, responseBody: AppEnv.isTest },
})

// Named namespaces — controllers opt in via openapiNames
psy.set('openapi', 'admin', {
  outputFilepath: path.join('src', 'openapi', 'admin.openapi.json'),
  validate: { requestBody: true, headers: true, query: true, responseBody: AppEnv.isTest },
})

psy.set('openapi', 'internal', {
  outputFilepath: path.join('src', 'openapi', 'internal.openapi.json'),
  validate: { requestBody: true, headers: true, query: true, responseBody: AppEnv.isTest },
})

// Tests namespace — all surfaces include 'tests' so controller specs can type-check against one spec
psy.set('openapi', 'tests', {
  outputFilepath: path.join('src', 'openapi', 'tests.openapi.json'),
  syncTypes: true,
})
```

Run `pnpm psy sync` after adding a new namespace for the `openapiNames` types to update.

Surface auth controllers (both `AuthedController` and `UnauthedController` within each surface) override `openapiNames` to include their surface name plus `'tests'`:

```typescript
public static override get openapiNames(): PsychicOpenapiNames<ApplicationController> {
  return ['admin', 'tests']
}
```

All controllers inheriting from these base controllers automatically inherit the same `openapiNames`.

## Nested Resource Base Controller Pattern

```typescript
export default class V1HostBaseController extends V1BaseController {
  protected currentHost: Host

  @BeforeAction()
  protected async loadCurrentHost() {
    const host = await this.currentUser.associationQuery('host').first()
    if (!host) return this.forbidden()
    this.currentHost = host
  }
}

// For doubly-nested resources (e.g., /places/:placeId/rooms)
export default class V1HostPlacesBaseController extends V1HostBaseController {
  protected currentPlace: Place

  @BeforeAction()
  protected async loadCurrentPlace() {
    this.currentPlace = await this.currentHost
      .associationQuery('places')
      .findOrFail(this.castParam('placeId', 'string'))
  }
}
```

## CRUD Controller Pattern

```typescript
import { OpenAPI } from '@rvoh/psychic'

export default class V1HostPlacesController extends V1HostBaseController {
  // INDEX with cursor pagination
  @OpenAPI(Place, {
    status: 200,
    tags: ['places'],
    description: 'Paginated list of host places',
    cursorPaginate: true,
    serializerKey: 'summary',
    fastJsonStringify: true,
  })
  public async index() {
    const places = await this.currentHost
      .associationQuery('places')
      .preloadFor('summary')
      .cursorPaginate({
        cursor: this.castParam('cursor', 'string', { allowNull: true }),
      })
    this.ok(places)
  }

  // SHOW
  @OpenAPI(Place, {
    status: 200,
    tags: ['places'],
    fastJsonStringify: true,
  })
  public async show() {
    this.ok(await this.place())
  }

  // CREATE
  @OpenAPI(Place, {
    status: 201,
    tags: ['places'],
    fastJsonStringify: true,
  })
  public async create() {
    let place = await ApplicationModel.transaction(async txn => {
      const place = await Place.txn(txn).create(this.paramsFor(Place))
      await HostPlace.txn(txn).create({ host: this.currentHost, place })
      return place
    })
    if (place.isPersisted) place = await place.loadFor('default').execute()
    this.created(place)
  }

  // UPDATE
  @OpenAPI(Place, { status: 204, tags: ['places'] })
  public async update() {
    const place = await this.place()
    await place.update(this.paramsFor(Place))
    this.noContent()
  }

  // DESTROY
  @OpenAPI({ status: 204, tags: ['places'] })
  public async destroy() {
    const place = await this.place()
    await place.destroy()
    this.noContent()
  }

  // Private helper to find and authorize resource
  private async place() {
    return await this.currentHost
      .associationQuery('places')
      .preloadFor('default')
      .findOrFail(this.castParam('id', 'string'))
  }
}
```

## Index with Search/Filtering

```typescript
@OpenAPI(Place, {
  status: 200,
  cursorPaginate: true,
  serializerKey: 'summary',
  query: {
    search: { required: false, schema: 'string' },
    style: { required: false, schema: 'string' },
    minSleeps: { required: false, schema: 'integer' },
  },
})
public async index() {
  let query = this.currentHost
    .associationQuery('places')
    .preloadFor('summary')

  const search = this.castParam('search', 'string', { allowNull: true })
  const style = this.castParam('style', 'string', { allowNull: true })
  const minSleeps = this.castParam('minSleeps', 'integer', { allowNull: true })

  if (search) {
    query = query.whereAny([
      { name: ops.like(`${search}%`) },
    ])
  }
  if (style) query = query.where({ style })
  if (minSleeps) query = query.where({ sleeps: ops.greaterThanOrEqualTo(minSleeps) })

  const result = await query.cursorPaginate({
    cursor: this.castParam('cursor', 'string', { allowNull: true }),
  })
  this.ok(result)
}
```

## STI Creation in Controllers

When creating STI child records, you MUST switch on the type:

```typescript
@OpenAPI(Room, { status: 201, tags: ['rooms'], fastJsonStringify: true })
public async create() {
  const roomType = this.castParam('type', 'string', { enum: RoomTypesEnumValues })
  const roomParams = this.paramsFor(Room)
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
      // TypeScript exhaustiveness check
      const _never: never = roomType
      throw new Error(`Unhandled RoomTypesEnum: ${_never as string}`)
    }
  }

  if (room.isPersisted) room = await room.loadFor('default').execute()
  this.created(room)
}
```

## Parameter Handling

### castParam Types

```typescript
// Basic types
this.castParam('id', 'uuid')
this.castParam('name', 'string')
this.castParam('count', 'integer')
this.castParam('id', 'bigint')
this.castParam('price', 'number')
this.castParam('birthday', 'date')
this.castParam('startAt', 'datetime')

// Array types
this.castParam('ids', 'uuid[]')
this.castParam('tags', 'string[]')

// With options
this.castParam('cursor', 'string', { allowNull: true })
this.castParam('style', 'string', { enum: PlaceStylesEnumValues })
this.castParam('roomType', 'string', { enum: RoomTypesEnumValues, allowNull: true })
```

When restricting to a subset of a database enum, use `Extract` to maintain compile-time type safety:

```typescript
import { RoomTypesEnum, RoomTypesEnumValues } from '@src/types/db.js'

// Subset — Extract ensures a compile error if the source enum changes
const BOOKABLE_ROOM_TYPES: Extract<RoomTypesEnum, 'Bedroom' | 'Den' | 'LivingRoom'>[] =
  ['Bedroom', 'Den', 'LivingRoom']

this.castParam('roomType', 'string', { enum: BOOKABLE_ROOM_TYPES })
```

### paramsFor

Extracts and validates params based on the model's database table columns, `@Virtual()` columns, and `@Encrypted()` columns, filtered by safe-params rules:

```typescript
// Basic
this.paramsFor(Place)

// Restrict to specific fields
this.paramsFor(Place, { only: ['name', 'style'] })

// Include extra fields beyond paramSafeColumns
this.paramsFor(Place, { including: ['specialField'] })

// Extract from nested key in body
this.paramsFor(Place, { key: 'place' })

// Extract an array of nested objects from a key in the body
// (e.g., body: { rooms: [{ type: 'Bedroom', ... }, { type: 'Kitchen', ... }] })
// Each element is validated against Room's paramSafeColumns
const roomsParams = this.paramsFor(Room, { key: 'rooms', array: true })
```

Prefer `paramsFor(Model, { key, array: true })` over `castParam('key', 'json[]')` for nested object arrays that correspond to a Dream model — it validates each item against the model's `paramSafeColumns` instead of accepting arbitrary JSON.

`paramsFor` provides default protection by automatically excluding certain columns that should never be set from user input. **This cannot be overridden** by `paramSafeColumns` or `paramUnsafeColumns`:

- The primary key (defaults to `id`)
- `createdAt` / `updatedAt` / `deletedAt`
- The `type` field of STI models
- Foreign keys of BelongsTo associations
- The polymorphic type field of polymorphic BelongsTo associations

These same columns are excluded from the auto-derived OpenAPI request body. The `requestBody.including` shorthand on `@OpenAPI` exists to re-add them — read it as "re-add what `paramsFor` excluded" rather than as a generic "add fields" knob. The exclusions exist to prevent mass-assignment on FK references and STI/polymorphic type discriminators; they cannot be overridden by `paramSafeColumns` regardless. Re-add via `including` for the spec, and pull the value explicitly via `castParam` (or via the request-body shape for bulk operations) inside the action.

Models can further restrict allowed params via `paramSafeColumns` (allowlist) or `paramUnsafeColumns` (blocklist):

```typescript
// Only allow these specific columns via paramsFor
public get paramSafeColumns(): DreamColumnNames<CoachingSession>[] {
  return ['rated', 'meetingIntent', 'topic']
}

// Block specific columns (in addition to the defaults above)
public get paramUnsafeColumns(): DreamColumnNames<Friend>[] {
  return ['bff']
}
```

Since foreign keys are excluded, find the resource explicitly and pass it as an association:

```typescript
const place = await Place.findOrFail(this.castParam('placeId', 'uuid'))
const favoritePlace = await this.currentUser.createAssociation('favoritePlaces', {
  ...this.paramsFor(FavoritePlace),
  place,
})
```

Use `requestBody: { including: ['placeId'] }` in the `@OpenAPI` decorator to add the FK to the OpenAPI request body shape (for typed spec helpers and client SDKs). This only affects the OpenAPI spec — `paramsFor` still excludes the column, so you must handle it explicitly via `castParam`.

The `paramsFor` exclusion of FKs is intentional — the OpenAPI spec advertises them via `including`, but `paramsFor` itself still strips the FK. Always extract the FK explicitly via `castParam` (or look up the parent resource and pass it as an association), even when the OpenAPI shape says it's part of the body.

## Response Methods

```typescript
// Success
this.ok(data)              // 200 - serializes Dream models automatically
this.created(data)         // 201
this.accepted(data)        // 202
this.noContent()           // 204

// Redirects
this.redirect(path)        // 302
this.movedPermanently(path) // 301

// Client errors
this.badRequest()          // 400
this.unauthorized()        // 401
this.forbidden()           // 403
this.notFound()            // 404
this.unprocessableEntity() // 422

// Server errors
this.serverError()         // 500
```

## Serializer Passthrough

Pass context data to serializers (e.g., locale, current user):

```typescript
// In base controller
@BeforeAction()
public configureSerializers() {
  this.serializerPassthrough({ locale: this.locale })
}

// In controller - query with passthrough
const places = await Place
  .passthrough({ locale: this.locale })
  .preloadFor('forGuests')
  .all()
this.ok(places)
```

## @OpenAPI Decorator Options

```typescript
@OpenAPI(ModelClass, {
  // Response configuration
  status: 200,                          // HTTP status code
  serializerKey: 'summary',             // Which serializer variant
  many: true,                           // Response is array (without pagination)
  fastJsonStringify: true,              // 2-5x faster JSON serialization

  // Pagination (use ONE of these)
  paginate: true,                       // Offset-based pagination
  cursorPaginate: true,                 // Cursor-based pagination
  scrollPaginate: true,                 // Scroll-based pagination

  // Documentation
  tags: ['places'],                     // OpenAPI tags
  description: 'List all places',       // Endpoint description

  // Request parameters
  query: {                              // Query string parameters
    search: { required: false, schema: 'string', description: 'Search term' },
    page: { required: false, schema: 'integer' },
  },
  pathParams: {                         // URL path parameters
    id: { description: 'The place ID' },
  },
  requestBody: {                        // Request body shape — prefer structured options over hand-writing a full schema
    only: ['name', 'style'],
    including: ['hostId', 'type'],
    combining: { acknowledgedTerms: { type: 'boolean' } },
    required: ['name', 'type'],
  },

  // Headers and security
  headers: {
    'X-Custom': { required: false },
  },

  // Custom responses (additional error descriptions)
  responses: {
    400: { description: 'Bad request' },
    404: { description: 'Not found' },
  },
})
```

### `requestBody` shorthand — what each option is for

The auto-derived request body shape is the model's `paramSafeColumns` — exactly the columns `paramsFor(Model)` would accept. Reach for `requestBody` only when you need to override that default. **If your action only takes `paramSafeColumns`, omit `requestBody` entirely.**

| Use | When |
|---|---|
| omit `requestBody` | Action only takes `paramSafeColumns`. |
| `only: [...]` | Narrow the auto-derived shape to a subset of the model's columns. |
| `including: [...]` | Re-add columns that ARE on the model but auto-excluded — FKs, STI `type`, polymorphic type. **Most common legitimate use.** Use this even when the FK is nullable; column nullability is inferred from the model. |
| `combining: { … }` | Add fields that are NOT columns on the model — join-table arrays (`cityIds`), one-shot tokens, upload metadata, workflow flags. |
| `required: [...]` | Mark fields required. **Typed to the model's column names**, so it cannot reference `combining` keys. |

#### Worked example — UPDATE with a nullable FK

The most-mistaken case: an `update` action whose body needs to express "the FK can be set or cleared." Reach for `including`, not `combining` — column nullability is inferred from the model's `@deco.BelongsTo('ZipCode', { optional: true })` declaration.

```typescript
// `@deco.BelongsTo('ZipCode', { optional: true })` on Venue makes zipCodeId nullable.
// `including: ['zipCodeId']` advertises the FK in the OpenAPI shape with the right nullability.
// Don't reach for `combining: { zipCodeId: ['string', 'null'] }` — `combining` shadows the
// model-derived shape with a hand-typed copy that drifts.
@OpenAPI(Venue, {
  status: 204,
  tags: ['venues'],
  requestBody: { including: ['zipCodeId'] },
})
public async update() {
  await (await this.venue()).update(this.paramsFor(Venue))
  this.noContent()
}
```

#### Anti-patterns

- **Don't list model columns inside `combining`.** `combining` adds fields that are *not* on the model. Listing real columns there either silently duplicates the auto-derived shape or shadows it with a hand-typed copy that drifts (e.g., a `{ enum: [...] }` redeclaration that goes stale when the DB enum changes).
- **Don't write `properties:` inside `requestBody`.** It is not one of the four shorthand options. If your body has required non-model fields, drop the shorthand entirely and use the explicit `type: 'object'` / `properties` / `required` form (next subsection).

#### When the body has no model fields

Some endpoints take only non-model inputs (one-shot tokens, redemption payloads, upload metadata). The shorthand's `required: [...]` is typed to the model's column names, so you cannot mark `combining` keys required through the shorthand. For these endpoints, write the explicit JSON-Schema form:

```typescript
requestBody: {
  type: 'object',
  required: ['token', 'firebaseUid'],
  properties: { token: 'string', firebaseUid: 'string' },
},
```

This is the correct tool when no model field participates in the body — not a fallback to be "simplified" later.

### Custom Response Envelopes

When an endpoint returns a custom object shape instead of a serialized model or pagination result, use `responses` to define the response contract. Psychic uses a **shorthand format** — schema properties go directly on the status code object, not nested inside `content: { 'application/json': { schema: ... } }`:

```typescript
@OpenAPI({
  status: 200,
  fastJsonStringify: true,
  responses: {
    200: {
      type: 'object',
      required: ['place', 'nearby'],
      properties: {
        place: { $serializable: Place, $serializableSerializerKey: 'summary' },
        nearby: { $serializable: Place, $serializableSerializerKey: 'summary', many: true },
      },
    },
  },
})
```

The `$serializable` shorthand references a Dream model's serializer within a custom schema, so you don't need to redeclare every field. Options: `$serializableSerializerKey` for a specific serializer key, `many: true` for arrays, `maybeNull: true` for nullable.

**Pre-render nested models in custom envelopes.** Raw Dream model instances inside a custom object will not be automatically serialized. Explicitly render nested models before calling `this.ok(...)`:

```typescript
@OpenAPI({
  status: 200,
  fastJsonStringify: true,
  responses: {
    200: {
      type: 'object',
      required: ['place', 'nearby'],
      properties: {
        place: { $serializable: Place, $serializableSerializerKey: 'summary' },
        nearby: { $serializable: Place, $serializableSerializerKey: 'summary', many: true },
      },
    },
  },
})
public async show() {
  const place = await this.place()
  const nearby = await Place.preloadFor('summary').where({ city: place.city }).limit(5).all()

  this.ok({
    place: PlaceSummarySerializer(place).render(),
    nearby: nearby.map(p => PlaceSummarySerializer(p).render()),
  })
}
```

Alternatively, formalize the response with an `ObjectSerializer`. This generates the OpenAPI schema automatically via `rendersOne` and `rendersMany`, avoiding the hand-written `responses` block:

```typescript
// PlaceWithNearbySerializer.ts
import { ObjectSerializer } from '@rvoh/dream'
import Place from '@models/Place.js'
import { PlaceSummarySerializer } from '@serializers/PlaceSerializer.js'

export const PlaceWithNearbySerializer = (place: Place, nearby: Place[]) =>
  ObjectSerializer({ place, nearby })
    .rendersOne('place', { serializer: PlaceSummarySerializer })
    .rendersMany('nearby', { serializer: PlaceSummarySerializer })
```

```typescript
// In the controller
@OpenAPI(PlaceWithNearbySerializer, { status: 200 })
public async show() {
  const place = await this.place()
  const nearby = await Place.preloadFor('summary').where({ city: place.city }).limit(5).all()

  this.ok(PlaceWithNearbySerializer(place, nearby))
}
```

## Debugging Unexpected 400s (and OpenAPI-triggered 500s in Specs)

When an endpoint returns an unexpected 400 (or a spec fails with a 500 thrown by OpenAPI response validation) and you can't tell whether OpenAPI validation or controller logic is the cause, temporarily disable validation to isolate it:

```typescript
@OpenAPI(Place, {
  status: 200,
  validate: { all: false },   // TEMPORARY — remove once debugged
})
```

The `validate` option accepts `requestBody`, `responseBody`, `headers`, `query`, and `all` booleans. Setting `all: false` disables every validation segment, which is useful because:
- For **400s on requests**, it reveals whether the failure was in OpenAPI request validation (problem stops) or in controller logic (problem persists).
- For **500s in specs caused by response validation**, it lets the full response body reach the test so you can inspect what actually came back — much more useful than the opaque validation error message.

### Workflow

1. **Add `validate: { all: false }`** to the `@OpenAPI` decorator for the failing endpoint.
2. **Run the request again.**
   - **If the failure stops:** the problem was in the payload. Log the payload to see what the controller actually received:
     ```typescript
     console.dir(this.params, { depth: null })
     ```
     Check whether the shape matches what the endpoint expects. If a feature spec was the trigger and the payload looks correct, consider writing a controller spec — faster and more isolated than feature specs — and iterate from there.
   - **If the failure persists:** the problem is in controller logic (likely a `castParam`/`paramsFor` failure, a DB constraint violation, or unhandled application error). Add logs around those calls to isolate.
3. **Remove the `validate` line** from the decorator once the problem is identified. Leaving it disabled defeats the protection OpenAPI validation provides.

`NODE_DEBUG=psychic` also surfaces validation info, but `validate: { all: false }` is usually more useful because it lets the real response body through for direct inspection.

## Automatic Error Handling

Psychic automatically converts certain errors to HTTP responses:

| Error Source | HTTP Status | When |
|---|---|---|
| `castParam` fails | 400 | Invalid parameter type/value |
| `paramsFor` fails | 400 | Invalid model attributes |
| Model validation fails | 400 | `isInvalid` with errors |
| `findOrFail` no match | 404 | Record not found |
| `firstOrFail` no match | 404 | Record not found |

All validation-layer errors return 400 by design — this prevents attackers from distinguishing which layer rejected a request. To explicitly return 422 with field-level validation errors intended for the end user, call `this.unprocessableContent({ errors: { fieldName: ['message'] } })`.

## Session & Cookie Management

```typescript
// Start session (sets encrypted cookie)
this.startSession(user)

// End session
this.endSession()

// Cookies
this.getCookie<string>('theme')
this.setCookie('theme', 'dark', { expires: new Date('2025-12-31') })
```

Session cookies set by Psychic default to `SameSite=Strict` — the browser won't send them on any cross-origin request (including link-click navigations from another site), which blocks classical CSRF without needing a CSRF token. Override only when a legitimate cross-site link-follow flow needs to preserve auth (rare for JSON APIs):

```typescript
psy.set('cookie', { sameSite: 'lax' })
```

### Cookie Encryption

`this.setCookie()` **always encrypts** the cookie value — there is no option to disable this. This is the correct behavior for cookies the Psychic application both writes and reads (e.g., session data, user preferences), since `this.getCookie()` automatically decrypts.

Cookie encryption requires an encryption key configured in `conf/app.ts`:

```typescript
psy.set('encryption', {
  cookies: {
    current: {
      algorithm: 'aes-256-gcm',
      key: AppEnv.string('APP_ENCRYPTION_KEY'),
    },
    legacy: {
      algorithm: 'aes-256-gcm',
      key: AppEnv.string('LEGACY_APP_ENCRYPTION_KEY'),
    },
  },
})
```

The optional `legacy` key enables seamless key rotation: `getCookie()` first tries the `current` key, and if decryption fails, retries with the `legacy` key.

To rotate the cookie encryption key:

1. Generate a new key (`pnpm psy g:encryption-key` or `Encrypt.generateKey('aes-256-gcm')`).
2. Set `current` to the new key, `legacy` to the previously-current key. Deploy.
3. Wait for `maxAge` to elapse so all in-the-wild cookies have rolled.
4. Drop `legacy`. Deploy.

Two keys is sufficient for any sensible cookie TTL.

#### Generating keys

Two ways to produce a key for any of the encryption use cases (encrypted columns, cookie encryption, generalized `Encrypt` library use):

```bash
# CLI generator — primary; defaults to aes-256-gcm.
# --algorithm accepts aes-256-gcm | aes-192-gcm | aes-128-gcm.
pnpm psy g:encryption-key
pnpm psy g:encryption-key --algorithm aes-128-gcm
```

```typescript
// Programmatic equivalent — useful in tests, fixtures, or one-off scripts.
import { Encrypt } from '@rvoh/dream'

const key = Encrypt.generateKey('aes-256-gcm')
```

Never hand-type encryption keys. Use one of these helpers and feed the result through your secrets manager (AWS Secrets Manager, SSM Parameter Store, Vault, etc.) into `AppEnv`.

### Decrypting Cookies Outside a Controller

Koa middleware that runs outside Psychic's controller layer (e.g., protecting a bull-board dashboard) cannot use `this.getCookie()`. To decrypt a Psychic-encrypted cookie in plain middleware, use `Encrypt` from Dream directly:

```typescript
import {
  Encrypt,
  DecryptionError,
  DecryptionParseError,
  DecryptionWithRotationError,
} from '@rvoh/dream'

const encrypted = ctx.cookies.get('cookie_name')
if (encrypted) {
  try {
    const decrypted = Encrypt.decrypt(encrypted, {
      algorithm: 'aes-256-gcm',
      key: AppEnv.string('APP_ENCRYPTION_KEY'),
    })
    // ...use decrypted
  } catch (err) {
    // Tampered ciphertext, wrong/rotated key, or non-JSON plaintext.
    // Treat as auth failure.
  }
}
```

`Encrypt.decrypt` throws on failure rather than silently returning null:

- **Two-argument form** `decrypt(ciphertext, key)` throws `DecryptionError` (cipher / auth-tag / shape failure) or `DecryptionParseError` (plaintext is not JSON). `null` / `undefined` input still returns `null`.
- **Three-argument form** `decrypt(ciphertext, currentKey, legacyKey)` tries `currentKey`, falls back to `legacyKey` on `DecryptionError`. If both keys fail, throws `DecryptionWithRotationError` carrying both per-key errors. **No silent-null on total rotation failure.**
- The `@Encrypted` model decorator propagates these errors up to the app's error handler — there's no try/catch inside the decorator.

Session cookies contain user-controlled ciphertext, so a thrown error is intentional signal (tampered or wrong-key). Catch it where the policy decision belongs (typically the controller / middleware that's reading the cookie).

### Setting Cookies for External Services (Bypassing Encryption)

When setting cookies intended for a different service to consume — e.g., CloudFront signed cookies for private content access — `this.setCookie()` will not work because the consuming service cannot decrypt Psychic's encrypted values.

Instead, use `ctx.append('Set-Cookie', ...)` directly on the Koa context:

```typescript
// In a controller action:
this.ctx.append(
  'Set-Cookie',
  `CloudFront-Key-Pair-Id=${keyPairId}; Path=/; Domain=.example.com; SameSite=Lax; Secure`
)
this.ctx.append(
  'Set-Cookie',
  `CloudFront-Signature=${signature}; Path=/; Domain=.example.com; SameSite=Lax; Secure`
)
```

Key points:
- Use `ctx.append` (not `ctx.set`) to allow multiple `Set-Cookie` headers without overwriting each other
- This bypasses both Psychic's encryption and Koa's cookie signing
- The Koa context is accessible in any controller action via `this.ctx` and can be passed to service methods

### Proxy Configuration for Secure Cookies

When the app runs behind a reverse proxy that terminates TLS (load balancers, Cloud Run, Cloudflare Tunnel, dev tunnels like ngrok), the Koa server receives plain HTTP and will reject `secure: true` cookies with "Cannot send secure cookie over unencrypted connection".

Set `app.proxy = true` in `conf/app.ts` to trust `X-Forwarded-Proto` headers from the proxy:

```typescript
psy.on('server:init:after-middleware', psychicServer => {
  psychicServer.koaApp.proxy = true
})
```

This may be necessary during development with tunneling tools (e.g., ngrok) or in environments where TLS is terminated at the proxy without re-encryption to the container. However, re-encrypting traffic between the proxy and the application is recommended — and required by security frameworks like HIPAA that mandate encryption in transit — so prefer configuring TLS on the application (see [deploying.md](deploying.md#tls-behind-a-reverse-proxy)) over enabling `app.proxy`.

## Nested Resource Creation via Association

```typescript
// Create through association (automatically sets FK)
let model = await this.currentCity.createAssociation('rooms', {
  ...this.paramsFor(Room),
  additionalField: value,
})

// Load for serialization if persisted
if (model.isPersisted) {
  model = await model.loadFor('admin').execute()
}
this.created(model)
```

## Generator Gotcha: Duplicate Route Namespaces

When running `pnpm psy g:resource` for a resource in an existing namespace (e.g., generating `v1/guest/reviews` when `v1/guest/bookings` already exists), the generator may add a second `r.namespace('guest', ...)` block in `routes.ts`. Always check `routes.ts` and consolidate duplicate namespace blocks after running a generator.
