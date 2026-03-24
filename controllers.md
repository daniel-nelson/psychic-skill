# Psychic Controllers - Detailed Reference

## Controller Hierarchy

Psychic controllers use inheritance for authentication and resource loading:

```
ApplicationController (base - defines psychicTypes)
├── AuthedController (@BeforeAction for auth, 401 if no user)
│   ├── V1BaseController
│   │   ├── V1HostBaseController (@BeforeAction loads currentHost)
│   │   │   ├── V1HostPlacesController (CRUD)
│   │   │   └── V1HostPlacesBaseController (@BeforeAction loads currentPlace)
│   │   │       └── V1HostPlacesRoomsController (nested CRUD)
│   │   └── V1GuestBaseController
│   │       └── V1GuestPlacesController (read-only)
│   ├── Admin/AuthedController
│   └── Internal/AuthedController
├── MaybeAuthedController (attempts auth, currentUser null if absent)
│   └── V1MaybeAuthedGuestBaseController
│       └── V1MaybeAuthedGuestPlacesController (public browse with optional auth)
└── UnauthedController
```

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

## Authentication Pattern

`create-psychic` generates three controller base classes for authentication:

- **AuthedController** — requires authentication, returns 401 if no user found
- **MaybeAuthedController** — attempts authentication but does NOT 401 on failure. Sets `currentUser` to null if no token or invalid token. Use for public endpoints that behave differently when logged in (e.g., showing favorite indicators on public browse pages).
- **UnauthedController** — no authentication at all

Both `AuthedController` and `MaybeAuthedController` delegate to a shared `resolveCurrentUser(controller)` helper in `controllers/helpers/resolveCurrentUser.ts`. The only difference is what happens when the result is null (401 vs. silent null).

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

### MaybeAuthed Namespace Pattern

When a namespace has both authenticated and maybe-authenticated endpoints, use separate controller directories with different base controllers:

```
V1/Guest/           → extends AuthedController (bookings, favorites, reviews)
V1/MaybeAuthedGuest/ → extends MaybeAuthedController (public browse with optional auth)
```

Do NOT put authenticated actions under a MaybeAuthed base controller. The controller hierarchy enforces auth by default — authenticated actions should always be under AuthedController to prevent accidentally exposing endpoints without auth.

### Routing MaybeAuthed Controllers Without Leaking Internal Names

When the controller directory name is an internal concern (like `MaybeAuthedGuest`) that shouldn't appear in public URLs, use an explicit controller reference instead of a namespace:

```typescript
// BAD — "maybe-authed-guest" leaks into the URL: /v1/maybe-authed-guest/places
r.namespace('maybe-authed-guest', r => {
  r.resources('places', { only: ['index', 'show'] })
})

// GOOD — clean URL: /v1/places, controller explicitly specified
import V1MaybeAuthedGuestPlacesController from '@controllers/V1/MaybeAuthedGuest/PlacesController.js'
r.resources('places', { only: ['index', 'show'], controller: V1MaybeAuthedGuestPlacesController })
```

The controller directory structure (`V1/MaybeAuthedGuest/`) still enforces the auth inheritance chain, but the URL stays clean.
```

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
this.castParam('status', 'string', { enum: ['active', 'inactive'], allowNull: true })
```

### paramsFor

Extracts and validates params matching model's safe columns:

```typescript
// Basic
this.paramsFor(Place)

// Restrict to specific fields
this.paramsFor(Place, { only: ['name', 'style'] })

// Include extra fields beyond paramSafeColumns
this.paramsFor(Place, { including: ['specialField'] })

// Extract from nested key in body
this.paramsFor(Place, { key: 'place' })
```

`paramsFor` provides default protection by automatically excluding certain columns that should never be set from user input. **This cannot be overridden** by `paramSafeColumns` or `paramUnsafeColumns`:

- The primary key (defaults to `id`)
- `createdAt` / `updatedAt` / `deletedAt`
- The `type` field of STI models
- Foreign keys of BelongsTo associations
- The polymorphic type field of polymorphic BelongsTo associations

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
  requestBody: {                        // Request body config
    including: ['name', 'style'],       // Fields to include in body schema
  },

  // Headers and security
  headers: {
    'X-Custom': { required: false },
  },

  // Custom responses
  responses: {
    400: { description: 'Bad request' },
    404: { description: 'Not found' },
  },
})
```

## Automatic Error Handling

Psychic automatically converts certain errors to HTTP responses:

| Error Source | HTTP Status | When |
|---|---|---|
| `castParam` fails | 400 | Invalid parameter type/value |
| `paramsFor` fails | 400 | Invalid model attributes |
| `findOrFail` no match | 404 | Record not found |
| `firstOrFail` no match | 404 | Record not found |
| Model validation fails | 422 | `isInvalid` with errors |

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
