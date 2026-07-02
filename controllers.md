# Psychic Controllers - Detailed Reference

## Controller Hierarchy

The controller directory structure **is** the authentication and authorization architecture. Each directory branch has its own `AuthedController` (and optionally `UnauthedController`, `MaybeAuthedController`) at its root, and every controller in that branch extends downward from it. By looking at the `@BeforeAction` methods in any directory's base controller, you can know exactly what rules are in force for the entire subtree.

**The cardinal rule:** never introduce a looser authentication pattern deeper in a directory hierarchy. Authentication flows downhill — it gets stricter, never weaker.

This tree is the *controller* auth architecture. Model namespacing is a separate, identity-based decision — don't mirror this controller tree into model names. See [models.md — Model Organization & Namespacing](models.md#model-organization--namespacing).

### Directory Structure

Each application surface — client (`V1/`), public browse + bootstrap (`Visitor/`), external webhooks (`Webhooks/`), partner API (`Api/`), admin, internal — gets its own top-level directory branch with its own auth base controllers. Any surface that loosens authentication is its own top-level namespace; versions nest inside it (see Key Principle #3):

```
controllers/
├── ApplicationController.ts          (base — defines psychicTypes and universal methods such as extracting Accept-Language header; sets openapi namespaces)
├── AuthedController.ts               (client auth — @BeforeAction, 401 if no user)
├── MaybeAuthedController.ts          (client auth — currentUser null if absent)
├── UnauthedController.ts             (no auth)
├── StatusController.ts               (health check — extends UnauthedController)
│
├── V1/                               (client API — AUTHED ONLY)
│   ├── BaseController.ts             (extends AuthedController)
│   ├── Guest/                        (authed Guest endpoints)
│   │   ├── BaseController.ts         (extends V1/BaseController, loads currentGuest, 403 if current User has no Guest)
│   │   ├── BookingsController.ts     (extends V1/Guest/BaseController)
│   │   └── FavoritesController.ts    (extends V1/Guest/BaseController)
│   └── Host/                         (authed Host endpoints)
│       ├── BaseController.ts         (extends V1/BaseController, loads currentHost, 403 if current User has no Host)
│       ├── PlacesController.ts       (extends V1/Host/BaseController)
│       └── Places/
│           ├── BaseController.ts     (extends V1/Host/BaseController, loads currentPlace)
│           └── RoomsController.ts    (extends V1/Host/Places/BaseController)
│
├── Visitor/                          (MAYBE-AUTHED client surface — public browse + bootstrap)
│   ├── BaseController.ts             (REPARENTED: extends MaybeAuthedController)
│   └── V1/
│       ├── BaseController.ts         (extends Visitor/BaseController)
│       ├── SignUpController.ts       (records app user + ToS consent; self-guards on currentUser)
│       ├── MeController.ts           (session probe; answers for a not-yet-cleared user)
│       └── PlacesController.ts       (public browse; varies when logged in)
│
├── Webhooks/                         (external callbacks — UNAUTHED; each provider verifies its own signature)
│   ├── BaseController.ts             (REPARENTED: extends UnauthedController — auth level only; no shared signature check)
│   └── V1/
│       ├── BaseController.ts         (extends Webhooks/BaseController)
│       ├── ZoomController.ts         (verifies the Zoom signature in a @BeforeAction)
│       └── TwilioController.ts       (verifies the Twilio signature in a @BeforeAction)
│
├── Api/                              (server-to-server partner API — UNAUTHED session, API-key auth)
│   ├── BaseController.ts             (REPARENTED: extends UnauthedController; verifies the API key in a @BeforeAction)
│   └── V1/
│       ├── BaseController.ts         (extends Api/BaseController)
│       └── ReservationsController.ts (extends Api/V1/BaseController)
│
├── Admin/                            (admin surface — separate auth chain, unversioned)
│   ├── AuthedController.ts           (admin auth — validates AdminUser; overrides openapi namespaces to: 'admin', 'tests')
│   ├── UnauthedController.ts         (admin unauthed; overrides openapi namespaces to: 'admin', 'tests')
│   ├── CitiesController.ts           (extends Admin/AuthedController)
│   ├── SignInController.ts           (extends Admin/UnauthedController)
│   └── Cities/
│       ├── BaseController.ts         (extends Admin/AuthedController, loads currentCity)
│       └── TravelTimesController.ts  (extends Admin/Cities/BaseController)
│
└── Internal/                         (internal employee surface — separate auth chain, unversioned)
    ├── AuthedController.ts           (internal auth — validates InternalUser; overrides openapi namespaces to: 'internal', 'tests')
    ├── UnauthedController.ts         (internal unauthed; overrides openapi namespaces to: 'internal', 'tests')
    ├── PlacesController.ts           (extends Internal/AuthedController)
    └── SignInController.ts           (extends Internal/UnauthedController)
```

### Key Principles

1. **Each surface has its own auth controllers.** `Admin/AuthedController` and `Internal/AuthedController` each extend `ApplicationController` directly — they do NOT extend the client `AuthedController`. Each authenticates its own user type (AdminUser, InternalUser, etc.).

2. **Every controller extends the base controller in its own directory** (or the authed/unauthed controller if at the top of its branch). Controllers never reach across branches or skip levels.

3. **Loosened auth lives in its own top-level namespace; versions nest inside it.** An authed surface may be versioned at the top (`V1/Guest/`, `V1/Host/` — all authed). Any surface that *loosens* auth — maybe-authed, unauthed, or server-to-server — is its own top-level namespace (`Visitor/`, `Webhooks/`, `Api/`) with the version nested inside (`Visitor/V1/`, `Webhooks/V1/`, `Api/V1/`), never `V1/Visitor/`, which would bury an auth change deep in the authed client branch. A separate auth surface (`Admin/`, `Internal/`) likewise gets its own top-level namespace and its own `AuthedController` / `UnauthedController`.

4. **Use generators** to create controllers. `pnpm psy g:resource` and `pnpm psy g:controller` set up the correct inheritance chain automatically. See [generators.md](generators.md) for the full scaffolding workflow (and where reparenting fits).

5. **Generate every surface; reparent the namespace base once when it loosens auth.** Always scaffold with the generators — they wire the inheritance chain and base controllers. The generator defaults each new namespace base to the client `AuthedController`, so an *authed* surface needs nothing extra:
   - `pnpm psy g:resource v1/host/places Place` → chains `V1/Host/PlacesController` → `V1/Host/BaseController` → `V1/BaseController` → `AuthedController`. No reparenting.
   - `pnpm psy g:resource admin/cities SupportedCity` → extends `Admin/AuthedController` (the generator special-cases `Admin` / `Internal`, each authenticating its own user type). No reparenting.

   A surface that *loosens* auth is reparented once, at its **top-level** namespace base (the nested version base inherits it — see #6):
   - `pnpm psy g:resource api/v1/reservations Booking` (a partner channel-manager syncing reservations) → reparent `Api/BaseController` to `UnauthedController` and verify the API key in a `@BeforeAction` there. One API, one key scheme, so it lives on the shared base.
   - `pnpm psy g:controller Webhooks/V1/Zoom create` → reparent `Webhooks/BaseController` to `UnauthedController` (auth level only). Verify the payload signature in a `@BeforeAction` on the *provider's* controller, not the shared base — a later `pnpm psy g:controller Webhooks/V1/Twilio create` reuses that base, and Twilio's signature scheme differs from Zoom's. Put on a shared base only what every endpoint under it shares.
   - `pnpm psy g:controller Visitor/V1/Places index` → reparent `Visitor/BaseController` to `MaybeAuthedController` (public reads that vary when logged in; bootstrap endpoints like `SignUp` and `/me` self-guard on `currentUser`).

   The same reparent-the-top-namespace-base move is how you exempt endpoints from a cross-cutting authorization gate — see [Cross-Cutting Authorization Gates](#cross-cutting-authorization-gates).

6. **`g:controller` is structural, not just a file stub.** It creates a `BaseController.ts` for each namespace segment that doesn't already exist (reusing the existing namespace base otherwise), wires the leaf controller through that namespace base, and creates the unit spec. Because an existing base is reused, any re-parenting of a namespace base is a one-time edit — later controllers generated into that namespace inherit it with no further change. It does not add routes. Hand-writing a controller skips the namespace base chain, which is exactly where shared webhook/public/custom-token concerns should live.

### Cross-Cutting Authorization Gates

The auth base controller is also where a cross-cutting *authorization* precondition lives — a check that must hold for every authenticated request, not just "is there a user." Accepted-current-terms-of-service, completed-onboarding, an active subscription, a verified email: any condition that gates the entire authenticated surface belongs in ONE `@BeforeAction` on the `AuthedController` at the root of that authed surface — each surface has its own (the client `AuthedController`, `Admin/AuthedController`, `Internal/AuthedController`) — declared *after* `authenticate` so `currentUser` is populated when it runs. A precondition that must hold across surfaces is declared once on each surface's auth base.

```typescript
export default class AuthedController extends ApplicationController {
  protected currentUser: User

  @BeforeAction()
  protected async authenticate() {
    // ...resolves and sets this.currentUser, or this.unauthorized()
  }

  // Declared after authenticate, so currentUser is set. Inherited by every
  // authed namespace (Guest/, Host/, and their nested bases) symmetrically.
  @BeforeAction()
  protected async requireCurrentTermsOfService() {
    if (!this.currentUser.hasAcceptedCurrentTermsOfService) {
      return this.forbidden('terms_of_service_required')
    }
  }
}
```

Because hooks inherit ancestor-to-descendant and a descendant cannot un-register or re-scope an inherited `@BeforeAction` (see [`@BeforeAction` scoping](#beforeaction-scoping)), this one declaration is the single, authoritative place the precondition is registered for the whole subtree. That is what makes the line-5 promise true: reading the base controller's `@BeforeAction`s tells you which preconditions run across the subtree. This is the architecture working as designed, not a limitation to route around.

**Exempt bootstrap endpoints structurally.** A few endpoints cannot be subject to the gate, because they are how a user *clears* it or *discovers* they have not: the endpoint that records consent / completes onboarding, and the not-yet-cleared probe (typically `GET /me`). A blocked user must still reach these, or they are locked out with no way forward. Do not weaken the gate for them — exempt them by ancestry, at the namespace's base controller:

- Put these endpoints in their own namespace and re-parent that namespace's **base controller** to a looser base — `MaybeAuthedController` for a surface that still wants optional auth (the `/me` probe, the consent-recording action that self-guards on `currentUser`), or `UnauthedController` for a surface with no app-user auth at all (e.g. webhooks). There is one base controller per directory, so this is a single edit that exempts the whole namespace; every controller in it inherits the looser base. For example, after `pnpm psy g:controller Visitor/V1/Me show`, re-parent `Visitor/BaseController.ts` to extend `MaybeAuthedController`; for `pnpm psy g:controller Webhooks/V1/Zoom create`, re-parent `Webhooks/BaseController.ts` to extend `UnauthedController`. The change is visible in the directory tree, so the exemption is auditable, not hidden in a per-action skip. This is the same reparent-the-namespace-base move used for every loosen-auth surface (see [Key Principle #5](#key-principles)), and like all such reparenting it is one-time — later controllers in the namespace reuse the existing base.
- Self-guard inside the action where the looser base allows a null user:

  ```typescript
  public async update() {
    if (!this.currentUser) return this.unauthorized()
    await this.currentUser.update({ acceptedTermsOfServiceVersion: CURRENT_TOS_VERSION })
    this.noContent()
  }
  ```

- Keep the URL clean by routing to the re-parented controller with an explicit `controller:` reference (the directory says `MaybeAuthed`, the URL should not) — see [Routing Controllers When Directory Names Don't Map to URLs](#routing-controllers-when-directory-names-dont-map-to-urls).

The result: the gate is universal and knowable in one place, and the handful of endpoints that bootstrap out of it are exempt by their position in the tree, not by an override that erodes the guarantee.

### `@BeforeAction` scoping

`@BeforeAction({ only, except })` filters by action method **name**, not by controller. The hook still runs on every controller that inherits it; `only`/`except` only decide which action methods it fires for, matched by name. So `except: ['create']` suppresses the hook for *every* descendant action named `create`, not for one controller:

```typescript
export default class AuthedController extends ApplicationController {
  // Skips the gate for any inherited action named `create`, in Guest/, Host/,
  // and every nested base below — not just one controller's create.
  @BeforeAction({ except: ['create'] })
  protected async requireCurrentTermsOfService() {
    if (!this.currentUser.hasAcceptedCurrentTermsOfService) {
      return this.forbidden('terms_of_service_required')
    }
  }
}
```

There is no `skipBeforeAction`, and a descendant cannot un-register or re-scope an inherited hook: redeclaring a `@BeforeAction` with the same method name in a descendant is a **no-op** for the *registration* — the ancestor's hook (including its `only`/`except`) is kept and the redeclaration is dropped. The one thing a subclass can still change is the hook's *behavior*, by overriding the method body itself (a same-named method runs in place of the ancestor's, since the hook is invoked by name). Don't do that — it changes what the gate does while leaving the registration intact, which is exactly the hidden override the directory model is meant to rule out. To vary auth for a subtree, re-parent it to a different base controller; the change is visible in the directory tree. This is what makes the base controller's `@BeforeAction`s authoritative for the whole subtree (see [Cross-Cutting Authorization Gates](#cross-cutting-authorization-gates)).

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
import VisitorV1PlacesController from '@controllers/Visitor/V1/PlacesController.js'
r.resources('places', { only: ['index', 'show'], controller: VisitorV1PlacesController })
```

The controller directory structure (`Visitor/V1/`) still enforces the auth inheritance chain, but the URL stays clean.

Three concerns are independent and should not be collapsed into one tree: the **URL namespace** (an API-contract concern — e.g. version-first `/v1/...`), the **controller file namespace**, and the **auth inheritance chain**. A versioned URL does not require a matching controller ancestry: don't make `Visitor/V1/BaseController` extend `V1/BaseController` merely because the URL starts with `/v1`. Versioning is a contract concern; authentication inheritance is a controller-hierarchy concern. Express auth boundaries through ancestry (a `Visitor/BaseController` extending `MaybeAuthedController` for optionally-authenticated public reads; `Guest/` and `Host/` bases staying authenticated), and let the route file map a versioned URL onto whatever controller has the correct ancestry via an explicit `controller` reference. Because `pnpm psy g:controller` generates the controller and spec but does not add routes, you're free to wire the route however the URL contract requires.

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
    // Encrypt.decrypt returns the already-parsed value — do not JSON.parse again.
    const decrypted = Encrypt.decrypt<{ userId: string }>(token, {
      algorithm: 'aes-256-gcm',
      key: AppEnv.string('APP_ENCRYPTION_KEY'),
    })
    const userId = decrypted?.userId
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

### Opting Controllers into an OpenAPI Namespace

Each surface's controllers are routed to one or more OpenAPI specs via the `openapiNames` getter, so endpoints land in the specs meant for their audience. A controller is documented into *every* spec it lists.

A fresh app sets this up on the base controllers. `ApplicationController` (the client base) defaults to all three client-relevant specs, so client controllers are documented into the web and mobile specs at once, from the same code:

```typescript
// ApplicationController — client controllers serve web and mobile from one place
public static override get openapiNames(): PsychicOpenapiNames<ApplicationController> {
  return ['default', 'mobile', 'tests']
}
```

The admin and internal surface bases (both `AuthedController` and `UnauthedController`) override it to their own surface plus `'tests'`, keeping their endpoints out of the client specs:

```typescript
// Admin/AuthedController and Admin/UnauthedController
public static override get openapiNames(): PsychicOpenapiNames<ApplicationController> {
  return ['admin', 'tests']
}
```

All controllers inheriting from these base controllers automatically inherit the same `openapiNames`. Every base includes `'tests'`, which is what lets controller specs across all surfaces type-check against one aggregated spec. Run `pnpm psy sync` after adding a new namespace so the `openapiNames` types update.

**Always keep `'tests'` in any `openapiNames` override you write.** A controller whose list omits `'tests'` is left out of the aggregated tests spec, so its endpoints get no generated types — the typed request helper won't recognize their paths and the controller spec can't type-check. When you add a new surface or namespace, the list is `['<surface>', 'tests']`, never `['<surface>']` alone.

The named specs themselves — output files, the `mobile` enum-shaping, the `tests` spec, defaults, validation — are configured in `conf/app.ts`. See [openapi.md](openapi.md#outputfilepath-and-namespaces).

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
      .findOrFail(this.castParam('placeId', 'uuid'))
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
    requestBody: { params: ['name', 'style', 'sleeps'] },
  })
  public async create() {
    let place = await ApplicationModel.transaction(async txn => {
      const place = await Place.txn(txn).create(this.extractParams(Place, ['name', 'style', 'sleeps']))
      await HostPlace.txn(txn).create({ host: this.currentHost, place })
      return place
    })
    if (place.isPersisted) place = await place.loadFor('default').execute()
    this.created(place)
  }

  // UPDATE
  @OpenAPI(Place, {
    status: 204,
    tags: ['places'],
    requestBody: { params: ['name', 'style', 'sleeps'] },
  })
  public async update() {
    const place = await this.place()
    await place.update(this.extractParams(Place, ['name', 'style', 'sleeps']))
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
      .findOrFail(this.castParam('id', 'uuid'))
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
@OpenAPI(Room, {
  status: 201,
  tags: ['rooms'],
  fastJsonStringify: true,
  requestBody: { params: ['name', 'sleeps'], including: ['type'] },
})
public async create() {
  const roomType = this.castParam('type', 'string', { enum: RoomTypesEnumValues })
  const roomParams = this.extractParams(Room, ['name', 'sleeps'])
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

### Request body size limits

`koa-bodyparser` defaults to `jsonLimit: '1mb'` and `formLimit: '56kb'`. Requests larger than these reject with a 413 before reaching the action. Override via `psy.set('json', { ... })` in `conf/app.ts`:

```typescript
psy.set('json', {
  jsonLimit: '5mb',
  formLimit: '500kb',
})
```

See the [koa-bodyparser README](https://github.com/koajs/bodyparser) for the full option surface. Tune to the largest legitimate payload your endpoints accept; DoS protection against oversized payloads belongs at the edge (see [Rate Limiting](#rate-limiting) above), not at the body-parser layer.

### castParam Types

```typescript
// Ids — cast to the column's primary-key type, never 'string'. Match your app's
// configured primaryKeyType: uuid4/uuid7 → 'uuid', bigserial/bigint → 'bigint',
// integer → 'integer'.
this.castParam('id', 'uuid')
this.castParam('id', 'bigint')
this.castParam('id', 'integer')

// Other scalars
this.castParam('name', 'string')
this.castParam('count', 'integer')
this.castParam('price', 'number')   // 'number' allows decimals — for values, not ids
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

### String params are trimmed automatically

Psychic strips leading and trailing whitespace from string params before validation and casting. Both `castParam` and `extractParams` resolve strings through the same `Params.cast` path, so `'  Cozy Cabin  '` arrives as `'Cozy Cabin'`. This applies to scalar `'string'` params, `enum` strings, Virtual string params, and the elements of string and enum arrays (`'string[]'`, array enum columns). Don't re-trim in a controller, a model setter, or a hook — it's already done.

### extractParams

`extractParams(Model, allowedParams, opts?)` validates an incoming request body against a Dream model's writable params and returns a typed, safe-to-write attributes object. The `allowedParams` array is an explicit positional allowlist, TS-checked at compile time against the model's safe params: real columns, Encrypted columns, and Virtual columns. Encrypted params are accepted as `string` or `string | null` based on the backing encrypted column's nullability; Virtual params use the type declared in `@deco.Virtual(...)`. The runtime intersects the allowlist with the model's `paramSafeColumns` (when declared) and the always-excluded set described below.

The allowlist is required and lives at the call site so that every controller's accepted params are locally visible. Adding a new model param doesn't widen any controller's surface — each controller has to opt in to accept it.

```typescript
// Standard usage
public async update() {
  await (await this.place()).update(
    this.extractParams(Place, ['name', 'style', 'sleeps']),
  )
  this.noContent()
}
```

#### Other options

```typescript
// Restrict further to a subset
this.extractParams(Place, ['name', 'style'], { only: ['name', 'style'] })

// Include extra fields beyond the allowlist
this.extractParams(Place, ['name'], { including: ['specialField'] })

// Extract from a nested key in the body
this.extractParams(Place, ['name', 'style'], { key: 'place' })

// Extract an array of nested objects from a key in the body
// (e.g., body: { rooms: [{ type: 'Bedroom', ... }, { type: 'Kitchen', ... }] })
// Each element is validated against the allowlist.
const roomsParams = this.extractParams(Room, ['type', 'name'], { key: 'rooms', array: true })
```

Prefer `extractParams(Model, allowed, { key, array: true })` over `castParam('key', 'json[]')` for nested object arrays that correspond to a Dream model — it validates each item against the allowlist instead of accepting arbitrary JSON.

#### Always-excluded columns

Some columns are always stripped, regardless of `paramSafeColumns` or the positional allowlist:

- The primary key (defaults to `id`)
- `createdAt` / `updatedAt` / `deletedAt`
- The `type` field of STI models
- Foreign keys of BelongsTo associations
- The polymorphic type field of polymorphic BelongsTo associations

These same columns are excluded from a model-derived OpenAPI request body — listing one in `params` won't surface it. The `requestBody.including` shorthand on `@OpenAPI` exists to re-add them — read it as "re-add what extraction excluded" rather than as a generic "add fields" knob. The exclusions exist to prevent mass-assignment on FK references and STI/polymorphic type discriminators. Re-add via `including` for the spec, and pull the value explicitly via `castParam` (or via the request-body shape for bulk operations) inside the action.

#### Declaring `paramSafeColumns` on the model

`paramSafeColumns` is the model-level upper bound on what `extractParams` can return. Declare it on any model where you want a single source of truth for "what a normal user can touch" — the runtime intersects the call-site allowlist with `paramSafeColumns`, so a column missing from the model's declaration cannot be extracted even if a controller lists it.

A model with no declared `paramSafeColumns` isn't left wide open in practice — a column is only ever extracted if some controller action's own call-site allowlist names it, and that allowlist is what you review at each `create`/`update` action. `paramSafeColumns`/`paramUnsafeColumns` exist for the narrower case: a specific column you want blocked at the model level regardless of what any present or future controller's allowlist says — an admin/role flag, an internal payout-account ID, a hand-managed tenancy column not backed by a declared `@BelongsTo`. Reach for `paramUnsafeColumns` to name just that column (it layers on top of the always-excluded set above); reach for `paramSafeColumns` when you want the model's entire safe surface enumerated in one place rather than trusting every call site to get its allowlist right. Neither is something to add reflexively to every model — only where a column genuinely must never be assignable no matter what a controller passes.

```typescript
// Canonical allowlist for the model — caps extractParams.
public get paramSafeColumns(): DreamColumnNames<CoachingSession>[] {
  return ['rated', 'meetingIntent', 'topic']
}

// Block specific columns (in addition to the always-excluded set above)
public get paramUnsafeColumns(): DreamColumnNames<Friend>[] {
  return ['bff']
}
```

Since foreign keys are excluded, find the resource explicitly and pass it as an association:

```typescript
const place = await Place.findOrFail(this.castParam('placeId', 'uuid'))
const favoritePlace = await this.currentUser.createAssociation('favoritePlaces', {
  ...this.extractParams(FavoritePlace, ['rating', 'note']),
  place,
})
```

Use `requestBody: { including: ['placeId'] }` in the `@OpenAPI` decorator to add the FK to the OpenAPI request body shape (for typed spec helpers and client SDKs). This only affects the OpenAPI spec — extraction still excludes the column, so you must handle it explicitly via `castParam`.

The exclusion of FKs is intentional — the OpenAPI spec advertises them via `including`, but extraction itself still strips the FK. Always extract the FK explicitly via `castParam` (or look up the parent resource and pass it as an association), even when the OpenAPI shape says it's part of the body.

#### Generator output

`psy g:resource` (and related generators) emit a shared `paramSafeColumns` const at the top of the controller file and reference it from every `create` / `update` action — and from those actions' `@OpenAPI` `requestBody` — so the allowlist stays visible at the call site without duplicating the array per action:

```typescript
import { OpenAPI } from '@rvoh/psychic'
import { DreamParamSafeColumnNames } from '@rvoh/dream/types'
import Post from '@models/Post.js'

const openApiTags = ['posts']

const paramSafeColumns: DreamParamSafeColumnNames<Post>[] = ['title', 'body']

export default class PostsController extends AuthedController {
  @OpenAPI(Post, {
    status: 201,
    tags: openApiTags,
    description: 'Create a Post',
    fastJsonStringify: true,
    requestBody: { params: paramSafeColumns },
  })
  public async create() {
    const post = await Post.create(this.extractParams(Post, paramSafeColumns))
    this.created(post)
  }

  @OpenAPI(Post, {
    status: 204,
    tags: openApiTags,
    description: 'Update a Post',
    fastJsonStringify: true,
    requestBody: { params: paramSafeColumns },
  })
  public async update() {
    await (await this.post()).update(this.extractParams(Post, paramSafeColumns))
    this.noContent()
  }
}
```

The const is typed `DreamParamSafeColumnNames<Model>[]`, which gives autocomplete of valid columns and a compile error on anything not param-safe right at the assignment. It is emitted only when the controller has a `create` and/or `update` action and the model is known (index/show/destroy-only controllers don't get it), and when no columns survive the param-safe filter it is still emitted, typed and empty: `const paramSafeColumns: DreamParamSafeColumnNames<Post>[] = []`. The real scaffold emits the action bodies commented-out as hints; they're shown uncommented here for readability. Admin and non-admin scaffolds emit the same shape.

**Single edit point:** narrowing `paramSafeColumns` updates both the runtime `extractParams` allowlist *and* the documented `@OpenAPI` request body in one place — the documented request shape and the runtime extraction can't silently diverge. Note this is `requestBody`, not the response: `create`'s 201 still returns the full serializer and `update` is 204 (no body) — the allowlist constrains *input*. The `requestBody: { params }` here is the generator pre-wiring the request body for scaffolded CRUD; the [`requestBody.including`](#always-excluded-columns) escape hatch ("re-add what extraction excluded") still applies on top of it unchanged.

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
// A same-origin relative path (starting with a single `/`) always works. An
// absolute-URL target is checked against `redirectAllowedHosts` — an empty
// allowlist by default — and throws a 500 (not a silent no-op) if the host
// isn't listed. Allowlist matching is host-only: case-insensitive and both
// port- and scheme-insensitive. Add external redirect targets explicitly:
//   psy.set('redirectAllowedHosts', ['oauth.example.com'])

// Client errors
this.badRequest()          // 400
this.unauthorized()        // 401
this.forbidden()           // 403
this.notFound()            // 404
this.unprocessableEntity() // 422

// Server errors
this.serverError()         // 500
```

### Error markers: distinguishing two same-status causes

When one endpoint can return the same status for two different reasons, the message you pass to a response helper becomes a runtime discriminator the frontend can switch on.

**Layer 1 — the marker (untyped, zero-config).** `this.forbidden(msg)` / `this.unauthorized(msg)` (and the rest) throw an `HttpError` whose argument is JSON-stringified as the response body. So `this.forbidden('not_your_place')` produces the body `"not_your_place"`. The default `Forbidden` / `Unauthorized` / etc. response components are description-only — no `content`, no schema — so the marker never appears in the generated spec or client. It is a runtime-only discriminator: two same-status causes on one endpoint, with no schema change and no regen. The frontend switches on it as an untyped string:

```typescript
// BearBnB: a Host editing a Place they don't own
if (!place.hostedBy(this.currentUser)) this.forbidden('not_your_place')
```

```typescript
// frontend
if (err.response.status === 403 && err.response.data === 'not_your_place') { /* ... */ }
```

Two caveats. Pass a non-empty string — `forbidden()` with a falsy argument sends no body, so the discriminator check fails. And the marker is always sent to the client verbatim at runtime (that is the point), independent of the spec, so keep it a coarse, stable cause code (`not_your_place`, `terms_of_service_required`) and never put sensitive detail in it.

**Layer 2 — the typed-enum upgrade.** The body stays a string; attaching an `enum` makes the generated client type it as a literal union instead of bare `string`. Declare it as a runtime OpenAPI schema shorthand (not a TS type) in the action's `@OpenAPI` `responses`:

```typescript
@OpenAPI(Place, {
  status: 204,
  responses: {
    403: { type: 'string', enum: ['not_your_place'] },
  },
})
public async update() { /* ... this.forbidden('not_your_place') ... */ }
```

Caveat, stated plainly: declaring the response only changes the spec and the generated types. It does not change runtime behavior and does not enforce the enum. You still write the `forbidden('not_your_place')` throw yourself and keep the thrown value in sync with the declared `enum` by hand — Psychic will not flag drift between them. A declared status replaces the default `$ref` for that one operation; the other default responses remain.

**Scope rule — where the typed response lives depends on where the cause is raised:**

- An *action-specific* cause (e.g. `not_your_place`, raised only inside that one Place action) → put the typed `responses` override on that action's `@OpenAPI`, as above. It applies to that operation alone, which is correct.
- A *cross-cutting* cause raised on a shared base `@BeforeAction` (e.g. `terms_of_service_required` thrown by the `AuthedController` ToS gate, returnable by *every* authed endpoint) → do **not** repeat it per action. Document it once by redefining the shared response component at conf level (`defaults.components.responses.Forbidden`), so the default 403 `$ref` resolves to the typed body across the spec. See [openapi.md — Customizing default error responses](openapi.md#customizing-default-error-responses) for the mechanism. A per-action override here would falsely imply only that one operation returns the marker, and force the same declaration onto every gated endpoint.

Honesty caveat for the cross-cutting case: conf `defaults` (including a redefined component) span the *whole* spec, not just the authed surface. If public endpoints also default a 403, they would advertise the marker body too. To scope the marker to authed controllers, give them their own named spec via `openapiNames` and redefine `Forbidden` only there (see [openapi.md](openapi.md#outputfilepath-and-namespaces)).

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
    params: ['name', 'style'],
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

**Always declare `requestBody: { params: [...] }` on a model-derived create/update.** List the model columns the action accepts, mirroring the `extractParams` allowlist — the documented request shape stays visible at the call site instead of being inferred. Reach for the other options below to layer onto that `params` list.

**`params` / `including` must be literal arrays — never compute them from the model.** Write the column names out (or share one hand-written `const paramSafeColumns: DreamParamSafeColumnNames<Place>[] = ['name', 'style']` between `extractParams` and `@OpenAPI`, as the generator does). **Never backfill them by calling `Model.paramSafeColumnsOrFallback()`, `Model.paramSafeColumns`, or `Model.columns()`** inside the decorator. Those return the model's *entire* writable surface — and for a model that declares no `paramSafeColumns`, that's every column on the table, names and types. Dumping that into the request body re-creates the implicit include-all default this convention exists to kill (the documented shape silently widens every time the model gains a column), advertises fields the action's `extractParams` allowlist doesn't actually accept (so the spec promises inputs that get dropped), and exposes the full column surface in the public OpenAPI shape. The `params` list is a deliberate, per-action allowlist; generating it from the model defeats the entire point.

**Derive before hand-writing.** If any body field is a Dream model column, use `params` / `including` / `combining` / `OpenAPI.forDream(...)` so types, nullability, and enum constraints come from the model. This still applies when the action cannot use `extractParams` directly, such as an STI create action that reads a `type` discriminator with `castParam` and then dispatches through an exhaustive `switch`. The runtime extraction can be custom; the OpenAPI request body should still be model-derived.

| Use | When |
|---|---|
| `params: [...]` | List the model columns the body accepts — its real columns, Encrypted columns, and Virtual columns. The default form on every model-derived create/update. |
| `including: [...]` | Re-add columns that ARE on the model but auto-excluded — FKs, STI `type`, polymorphic type. **Most common legitimate use.** Use this even when the FK is nullable; column nullability is inferred from the model. |
| `combining: { … }` | Add fields that are NOT columns on the model — join-table arrays (`cityIds`), one-shot tokens, upload metadata, workflow flags. |
| `required: [...]` | Mark fields required. **Typed to the model's column names**, so it cannot reference `combining` keys. |

One important STI nuance: when `@OpenAPI(BaseModel, ...)` documents an STI-dispatched create action, its `including` and `required` are typed to the base model's **shared physical table**, not to the TypeScript properties declared on the base class. Because every STI child shares that table, a column declared (as a `DreamColumn` property) only on a child class is still a real column on the base table, so it is part of the base model's derived request-body surface — re-add an auto-excluded one (a FK or the `type` discriminator) with `including` exactly as you would any base-model column. This is the OpenAPI mirror of `extractParams(BaseModel, [...])`, which names child columns off the same table (see [STI controller pattern](sti.md#controller-pattern)). Before deciding a field is unavailable to `@OpenAPI(BaseModel, ...)`, check the generated table metadata (`src/types/db.ts` / `src/types/dream.ts`): a physical STI column is derivable there even when no base-class property declares it. Reserve `combining` for a field that genuinely is *not* a model column — a non-column input such as a one-shot token, upload metadata, or a join-table array. A model's declared columns and its own virtual attributes are both part of its model-derived surface (`params` / `including`), so neither belongs in `combining`. One STI exception: a `@deco.Virtual` declared on a *child* class does not filter up to the base, so naming it in the base model's `params` / `including` type-checks but renders nothing into the spec — document a child's virtual attribute on a child-specific `@OpenAPI(ChildModel, ...)` instead (see [sti.md — Virtual attributes don't filter up to the base class](sti.md#virtual-attributes-dont-filter-up-to-the-base-class)).

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
  requestBody: { params: ['name'], including: ['zipCodeId'] },
})
public async update() {
  await (await this.venue()).update(this.extractParams(Venue, ['name', 'zipCodeId']))
  this.noContent()
}
```

#### Nested model-driven shapes — multi-resource creation

When a request body bundles multiple Dream models — typically a parent plus a one-shot array of children — use the `for: Model` sentinel inside `combining` (or `combining.<key>.items` for arrays). It produces an inline object schema derived from the model's `paramSafeColumns`. Use `OpenAPI.forDream(Model, opts)` as the typed wrapper: `params` / `including` / `required` are constrained at compile time to that model's column names, so a misspelled column raises a TS error instead of silently dropping out of the OpenAPI shape.

The nested `for:` sentinel is **request-only**; response shapes are modeled as a serializer and declared with `@OpenAPI(SerializerFn)`, not hand-written — see [Custom Response Envelopes](#custom-response-envelopes).

**BearBnB worked example.** A single `POST /host/places` that creates a `Place` and an array of `Room`s atomically:

```typescript
import { Place, Room } from '@app/models'
import { OpenAPI } from '@rvoh/psychic'

@OpenAPI(Place, {
  status: 201,
  tags: ['places'],
  fastJsonStringify: true,
  requestBody: {
    params: ['name', 'style', 'sleeps'],
    combining: {
      // Nested array of Room shapes, each derived from Room's paramSafeColumns.
      // OpenAPI.forDream's generic narrows `required` to Room column names —
      // a typo here is a compile error.
      rooms: {
        type: 'array',
        items: OpenAPI.forDream(Room, {
          required: ['type', 'name'],
        }),
      },
    },
  },
})
public async create() {
  const placeAttrs = this.extractParams(Place, ['name', 'style', 'sleeps'])
  const roomsAttrs = this.extractParams(Room, ['type', 'name'], { key: 'rooms', array: true })

  const place = await ApplicationModel.transaction(async txn => {
    const created = await Place.txn(txn).create({ ...placeAttrs, host: this.currentHost })
    for (const r of roomsAttrs) {
      await Room.txn(txn).create({ ...r, place: created })
    }
    return created
  })

  this.created(await place.loadFor('default').execute())
}
```

The OpenAPI shape and the `extractParams` calls stay aligned: the spec advertises `{ name, style, sleeps, rooms: [{ type, name }, ...] }`, and the action extracts each model's params with its own allowlist. Wrap the writes in `ApplicationModel.transaction` so a failed Room rolls back the Place — partial-creation is rarely the desired failure mode for bundled requests.

#### Anti-patterns

- **Don't list model columns inside `combining`.** `combining` adds fields that are *not* on the model. Listing real columns there either silently duplicates the model-derived shape or shadows it with a hand-typed copy that drifts (e.g., a `{ enum: [...] }` redeclaration that goes stale when the DB enum changes). Use `OpenAPI.forDream(Model, ...)` (or the nested `for:` sentinel) when the value is itself a Dream-model shape.
- **Don't write `properties:` inside `requestBody` for model fields.** It is not one of the shorthand options, and it duplicates model-owned type information. Only use explicit `type: 'object'` / `properties` / `required` when the body has no model fields at all (next subsection).

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

When an action returns a custom or compound shape — an envelope like `{ place, nearby }`, or a computed array alongside serialized models — model the whole shape as one composing `ObjectSerializer` and pass it to `@OpenAPI`; see [A compound response is still one serializer](serializers.md#a-compound-response-is-still-one-serializer) for how to build it. The action renders the plain data with `this.ok(...)`:

```typescript
@OpenAPI(PlaceWithNearbySerializer, { status: 200 })
public async show() {
  const place = await this.place()
  const nearby = await Place.preloadFor('summary').where({ city: place.city }).limit(5).all()

  this.ok(PlaceWithNearbySerializer(place, nearby))
}
```

The serializer is the single source of truth for the response schema, so don't hand-write a `responses` block to shape a success response. Even a one-off envelope is a composing serializer — it stays derived and validated where a hand-written schema drifts.

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
   - **If the failure persists:** the problem is in controller logic (likely a `castParam` / `extractParams` failure, a DB constraint violation, or unhandled application error). Add logs around those calls to isolate.
3. **Remove the `validate` line** from the decorator once the problem is identified. Leaving it disabled defeats the protection OpenAPI validation provides.

`NODE_DEBUG=psychic` also surfaces validation info, but `validate: { all: false }` is usually more useful because it lets the real response body through for direct inspection.

## Automatic Error Handling

Psychic automatically converts certain errors to HTTP responses:

| Error Source | HTTP Status | When |
|---|---|---|
| `castParam` fails | 400 | Invalid parameter type/value |
| `extractParams` fails | 400 | Invalid model attributes |
| Model validation fails | 400 | `isInvalid` with errors |
| `findOrFail` no match | 404 | Record not found |
| `firstOrFail` no match | 404 | Record not found |

All validation-layer errors return 400 by design — this prevents attackers from distinguishing which layer rejected a request. Psychic's automatic conversions return a bare 400 with no body for param, requestBody, and model validation failures, so the response is opaque about which layer rejected.

To deliberately surface field-level validation errors to the end user, return a 400 that carries the error shape — call `this.badRequest({ errors: { ... } })`. The natural source of that shape is a Dream model's `.errors` getter, which is keyed `{ field: ['message'], ... }`:

```ts
const place = Place.new(this.extractParams(Place, ['name', 'style', 'sleeps']))
if (place.isInvalid) this.badRequest({ errors: place.errors })   // place.errors → { name: ['must be present'], ... }
await place.save()
```

Without the explicit check, an invalid `save()` / `create()` still returns 400, but with no body — the framework logs the errors rather than sending them. Conveying the error shape to the client is therefore an explicit, deliberate act.

## Rate Limiting

Psychic does not bundle rate-limiting. Neither does Koa nor socket.io. **Handle this at the edge** — WAF, CDN, reverse proxy, or API gateway. Connection-exhaustion attacks (SlowLoris, SYN floods, packet-level abuse) can only be solved at the edge; by the time traffic reaches the Node event loop the damage is done.

Only fall back to a Koa-compatible middleware when you need fine-grained per-route or per-user limits the edge layer cannot express (e.g., "5 login attempts per IP+account per 15 minutes"). When you do, consult current best practices to pick a package — **don't hand-roll** a `Map`-based limiter; in-process counters don't coordinate across pods, and the failure modes are subtle.

## Logging

Use `PsychicApp.log` and `PsychicApp.logWithLevel` for all application logging. They route through Dream's configured logger (Winston by default) so output respects the log level, the configured transports, and any structured-log format the app sets.

```typescript
import { PsychicApp } from '@rvoh/psychic'

PsychicApp.log('order placed', { orderId: order.id, totalCents })
PsychicApp.logWithLevel('warn', 'inventory low', { sku, remaining })
PsychicApp.logWithLevel('error', 'payment provider error', { orderId: order.id, err })
```

Both are variadic — pass a message followed by any structured metadata. `logWithLevel` accepts `'debug' | 'info' | 'warn' | 'error'` (Dream's log levels).

### Request logging

`create-psychic` scaffolds a request logger in `conf/app.ts` mounted via the `server:init:after-middleware` hook. The default emits one structured line per request through `PsychicApp.log` under the hood:

```typescript
import requestLogger from '@conf/system/requestLogger.js'
import winston from 'winston'

psy.on('server:init:after-middleware', psychicServer => {
  if (!AppEnv.isTest || AppEnv.boolean('REQUEST_LOGGING')) {
    const SENSITIVE_FIELDS = ['password', 'token', 'authentication', 'authorization', 'secret']

    psychicServer.koaApp.use(
      requestLogger({
        transports: [new winston.transports.Console()],
        format: winston.format.combine(winston.format.json()),
        headerBlocklist: [
          'authorization', 'content-length', 'connection', 'cookie',
          'sec-ch-ua', 'sec-ch-ua-mobile', 'sec-ch-ua-platform',
          'sec-fetch-dest', 'sec-fetch-mode', 'sec-fetch-site', 'user-agent',
        ],
        ignoredRoutes: ['/health_check'],
        bodyBlocklist: SENSITIVE_FIELDS,
      }),
    )
  }
})
```

Three knobs to know:

- **`headerBlocklist`** — request headers redacted from the log line. Scaffold ships `authorization`, `cookie`, plus a handful of low-signal browser-fingerprint headers.
- **`bodyBlocklist`** — request body keys redacted from the log line. Scaffold ships `password`, `token`, `authentication`, `authorization`, `secret`. **Extend this when you add new sensitive fields** (`api_key`, `ssn`, `creditCard`, etc.) — otherwise they land in production logs.
- **`ignoredRoutes`** — request paths to skip entirely. Scaffold skips `/health_check`.

### Reacting to unhandled errors

Errors that escape the controller stack (anything Psychic doesn't auto-convert to a 4xx) fire the `server:error` hook. This is the right place to ship to Sentry, Datadog, structured alerting, or whatever your error pipeline is:

```typescript
psy.on('server:error', (err, ctx) => {
  PsychicApp.logWithLevel('error', 'unhandled server error', {
    err,
    method: ctx.method,
    path: ctx.path,
    requestId: ctx.state.requestId,
  })
  // ...ship to Sentry / Datadog / etc.

  if (!ctx.headerSent) ctx.status = 500
  else if (AppEnv.isDevelopmentOrTest) throw err
})
```

The handler signature is `(err: Error, ctx: Koa.Context) => void | Promise<void>`. The scaffold registers a default that sets `ctx.status = 500` if a response hasn't been sent and re-throws in development/test so the failure surfaces loudly.

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

`startSession` creates no server-side session record — the encrypted cookie itself is the entire credential. `endSession` only clears the cookie on the client; a cookie already captured elsewhere stays valid until it expires regardless of "logout." There is no per-session or per-user revocation — the only framework lever to invalidate outstanding sessions is rotating the cookie encryption key (see [Cookie Encryption](#cookie-encryption) below), which logs out every user, not just one. Design around this: if you need to revoke a single compromised session or a single user's sessions without logging everyone out, track that state yourself (e.g. a `sessionsInvalidatedAt` column on the user, checked against a timestamp embedded in the session payload) rather than assuming `endSession` accomplishes it server-side. The default cookie `maxAge` is 31 days; pass `{ maxAge: { hours: … } }` to `setCookie` for a shorter-lived session.

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
import { PsychicApp } from '@rvoh/psychic'
import { Encrypt } from '@rvoh/dream'
import { DecryptionError, DecryptionRotationError } from '@rvoh/dream/errors'

const encrypted = ctx.cookies.get('cookie_name')
if (encrypted) {
  let decrypted: { userId: string } | null
  try {
    // Three-arg (rotation) form: current key, with legacy fallback.
    decrypted = Encrypt.decrypt<{ userId: string }>(
      encrypted,
      { algorithm: 'aes-256-gcm', key: AppEnv.string('APP_ENCRYPTION_KEY') },
      { algorithm: 'aes-256-gcm', key: AppEnv.string('APP_LEGACY_ENCRYPTION_KEY') },
    )
  } catch (err) {
    if (err instanceof DecryptionRotationError) {
      // Both the current AND legacy keys failed. Do NOT swallow this as an
      // auth failure: the default posture is to rethrow so a misconfigured
      // rotation (wrong legacy key, legacy dropped too early) fails the
      // FIRST request — caught by smoke tests / health checks at deploy
      // time, not discovered later in a log nobody is tailing while every
      // user is being logged out. Only once a rotation has been proven
      // stable in production for a prolonged window might you deliberately
      // downgrade this to log-at-error + unauthenticated (so legitimately
      // expired ciphertext during a long rotation isn't a hard failure).
      // That downgrade is a conscious, reversible decision — never the
      // starting point.
      throw err
    }
    if (err instanceof DecryptionError) {
      // Wrong/tampered/stale-key ciphertext. Expected at low volume
      // (old cookies, forgery attempts); worth a warn for incident
      // correlation. A sustained spike is an attack or a key problem.
      PsychicApp.logWithLevel('warn', 'cookie decryption failed', { reason: err.message })
      return // treat this request as unauthenticated
    }
    // DecryptionParseError: the key was correct and the ciphertext intact
    // (cipher + auth tag validated), but the decrypted plaintext was not
    // valid JSON. Encrypt.decrypt always JSON.parses, so this is a
    // format/contract mismatch, not tampering and not a wrong key — e.g.
    // a non-JSON value passed into Encrypt.encrypt on our side, or (with a
    // shared key from another service) their payload not matching the
    // JSON-wrapped shape Encrypt expects. Never an auth outcome; let it
    // propagate to the error handler (500) so the mismatch is visible.
    throw err
  }
  // ...use decrypted (already the parsed object, not a JSON string)
}
```

The `catch` exists to convert an untrusted-input failure into an auth decision **and emit the security signal** — not to swallow it. The original finding (R-019) was that silent-null *"conflates an attacker attempting forgery with this value never being set, and obscures bugs in app code — loses operationally important signal for incident logs."* A bare `catch { return unauthenticated }` reintroduces exactly that defect: it hides both forgery attempts and a broken key rotation.

`DecryptionRotationError` is reachable only from the three-argument (current + legacy) form, and its safe default is to **rethrow**, not log-and-continue. A logged-and-swallowed rotation error still ships a broken rotation to production; a thrown one fails the first request at deploy time, which is exactly when a wrong legacy key or a prematurely-dropped legacy key must be caught. Downgrading it to log-at-error + unauthenticated is a deliberate, reversible posture you adopt *only after* a rotation has run cleanly in production for a prolonged window (so that genuinely expired ciphertext mid-rotation isn't a hard failure) — it is never the starting configuration. `DecryptionError` (single-key / no-rotation path) is the normal stale-or-forged-cookie case: log at `warn` and treat as unauthenticated. Never blanket-catch (see Critical Rule 14).

`Encrypt.decrypt` throws on failure rather than silently returning null, and the three errors export from `@rvoh/dream/errors` (not the `@rvoh/dream` root):

- `decrypt` returns the **already-`JSON.parse`d** value, not a string. Don't `JSON.parse` the result again.
- `null` / `undefined` ciphertext returns `null` (no throw). A missing key throws `MissingEncryptionKey`.
- **`DecryptionError`** — cipher op / auth-tag / payload-shape was invalid: the ciphertext was tampered with, corrupted, or encrypted with a different key. This is the untrusted-input signal — the legitimate "treat as unauthenticated" case for user-controlled ciphertext like a session cookie.
- **`DecryptionParseError`** — the cipher and auth tag validated (correct key, untampered ciphertext) but `JSON.parse` of the decrypted plaintext threw. `Encrypt.decrypt` always `JSON.parse`s the plaintext — it cannot return a raw string — so this is strictly a **format/contract mismatch**, not tampering and not a wrong key. With ciphertext you produced via `Encrypt.encrypt`, it means a non-JSON value was encrypted (your bug). With a shared key from another service, it usually means that service's payload doesn't match the JSON-wrapped shape `Encrypt` expects (their encoder, or the cross-service contract). Either way it is **never** an auth failure: let it propagate to the error handler as a 500 so the mismatch surfaces; swallowing it as "auth failure" hides the defect.
- **Three-argument form** `decrypt(ciphertext, currentOpts, legacyOpts)` tries the current key, and on `DecryptionError` only falls back to the legacy key. If both fail with `DecryptionError`, it throws **`DecryptionRotationError`** carrying `.currentKeyError` and `.legacyKeyError`. A `DecryptionParseError` from the current key is **not** retried against the legacy key and **not** wrapped — the cipher already matched (right key, intact ciphertext), so a non-JSON plaintext is a format/contract mismatch rather than a key problem, and it propagates as `DecryptionParseError` directly.
- The `@Encrypted` model decorator propagates these errors up to the app's error handler — there's no try/catch inside the decorator. Keys *and* ciphertext for an encrypted column are both system-controlled, so any decryption error there is a corrupted column or a broken rotation, not untrusted input: it should fail loudly, never be caught and nulled.

The asymmetry is deliberate. **User-controlled ciphertext** (a session cookie, a token from a header) is the only place where catching is ever correct, and even there only `DecryptionError` (the single-key stale/forged case) is caught-and-converted to unauthenticated with a `warn` log. `DecryptionRotationError` defaults to rethrow — a broken rotation is a configuration defect that must fail loudly at deploy, not a per-request auth outcome; only downgrade it to log-and-unauthenticated as a conscious, reversible decision after the rotation has proven stable in production. **System-controlled ciphertext** (`@Encrypted` columns, internal share tokens) propagates untouched in every case. And `DecryptionParseError` is always a format/contract mismatch (your encoder or a cross-service one), never an auth outcome. If you do adopt the log-and-continue downgrade for a prolonged rotation, treat a rising rate of `DecryptionRotationError` as a rotation-misconfiguration alarm wired into monitoring — but prefer the throw, which needs no one watching a dashboard.

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
  ...this.extractParams(Room, ['name', 'sleeps']),
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
