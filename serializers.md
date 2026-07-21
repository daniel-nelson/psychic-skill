# Dream Serializers - Detailed Reference

## Overview

Serializers are **function-based** (not classes, not decorator-based). They transform Dream models into JSON for API responses and automatically generate OpenAPI schemas.

There are two serializer types:
- **DreamSerializer** - For Dream model instances (auto-infers types from DB schema)
- **ObjectSerializer** - For plain objects/view models (requires explicit OpenAPI types)

## Import Paths

```typescript
import { DreamSerializer, ObjectSerializer } from '@rvoh/dream'
```

## Basic Pattern

Serializer files must use **named exports only**. Do not use `export default` for a serializer function. Dream derives runtime serializer global names from the serializer file path plus the export name; default exports use the file path alone, which is easier to collide with generated/path-normalized names and makes OpenAPI names less explicit. Named exports keep the model's `serializers` getter, OpenAPI component names, and imports aligned.

```typescript
import Place from '@models/Place.js'
import { DreamSerializer } from '@rvoh/dream'

// Summary serializer — used for index/list views. Generators create it with
// minimal attributes (often just id); add attributes here as the index view
// needs them. Since the default serializer extends the summary, anything
// added to the summary automatically appears in the default (show) view too.
export const PlaceSummarySerializer = (place: Place) =>
  DreamSerializer(Place, place)
    .attribute('id')
    .attribute('name')

// Default serializer (extends summary via composition)
export const PlaceSerializer = (place: Place) =>
  PlaceSummarySerializer(place)
    .attribute('style')
    .attribute('sleeps')
    .attribute('deletedAt')
```

When a property should appear on both index and show views, **move** it from the default serializer to the summary serializer. Because the default extends the summary, the property still appears on show automatically. Generators create the default with all attributes and the summary with only `id` — moving attributes from default to summary is the normal way to build up the index view.

## Registering on Models

Models declare serializers via `get serializers()`:

```typescript
export default class Place extends ApplicationModel {
  public get serializers(): DreamSerializers<Place> {
    return {
      default: 'PlaceSerializer',
      summary: 'PlaceSummarySerializer',
      forGuests: 'PlaceForGuestsSerializer',
      admin: 'PlaceAdminSerializer',
    }
  }
}
```

Serializer names are **strings** matching named exported function names. They're loaded globally via `app.load('serializers', ...)` in `conf/dream.ts`.

## Serializer Methods

### .attribute(name, options?)

Include a model column:

```typescript
.attribute('id')           // Auto-typed from DB schema
.attribute('name')         // Auto-typed from DB schema
.attribute('email')        // Auto-typed from DB schema
```

Options:

```typescript
// Rename in output
.attribute('email', { as: 'userEmail' })

// Decimal precision
.attribute('price', { precision: 2 })

// Explicit OpenAPI type (needed for Virtual columns or when inference isn't enough)
.attribute('status', { openapi: { type: 'string', enum: StatusEnumValues } })
.attribute('computed', { openapi: 'string' })
.attribute('data', { openapi: { type: 'object', properties: { key: { type: 'string' } } } })

// Default value
.attribute('status', { default: 'pending' })
```

**Important**: For `@deco.Virtual()` columns with an OpenAPI type annotation, use plain `.attribute('name')` without the openapi option - the type is inferred from the Virtual decorator.

### .customAttribute(name, fn, options)

Computed/virtual field with a callback:

```typescript
.customAttribute('displayStyle', () => i18n(passthrough.locale, `places.style.${place.style}`), {
  openapi: 'string',
})

.customAttribute('fullName', () => `${user.firstName} ${user.lastName}`, {
  openapi: 'string',
})

// Flatten computed object into parent
.customAttribute('coordinates', () => ({ lat: 40.7, lng: -74.0 }), {
  flatten: true,
  openapi: { lat: { type: 'number' }, lng: { type: 'number' } },
})
```

Note: a `customAttribute` whose function body reads an association is invisible to `preloadFor` — see [preloadFor Integration](#preloadfor-integration). This includes a fallback expression like `() => current?.title ?? fallback.title`: reach for [layered `delegatedAttribute`s](#layering-two-delegatedattributes-onto-the-same-output-key) instead, so both associations get preloaded automatically.

### .delegatedAttribute(associationName, propertyName, options?)

Access a property on a loaded association:

```typescript
// Gets `place.currentLocalizedText.title`
.delegatedAttribute('currentLocalizedText', 'title', { openapi: 'string' })

// Gets `user.profile.avatarUrl`
.delegatedAttribute('profile', 'avatarUrl', { openapi: 'string' })

// Key always present; null when association is missing (HasOne that isn't always present).
.delegatedAttribute('profile', 'avatarUrl', { openapi: 'string', optional: true })

// Key omitted from response when association/path is missing.
.delegatedAttribute('profile', 'avatarUrl', { openapi: 'string', required: false })
```

When the delegated-through path may resolve to `undefined`/`null` (a missing `HasOne`, an absent JSON sub-key, etc.), choose between `optional: true` and `required: false` based on what consumers should see. They are not aliases; they govern different layers and can be combined.

| Option | Runtime | OpenAPI |
|---|---|---|
| `optional: true` | No effect — key always rendered (`null` when missing). | Schema wrapped in `anyOf: [schema, { type: 'null' }]`. |
| `required: false` | Key **omitted** from the response when the resolved value is `undefined`. | Field excluded from the containing schema's `required[]`. |
| `default: <value>` | Substitutes the value when the resolved path is `undefined`. | (No effect.) |

Resolution order at render time: first non-`null`/non-`undefined` value from `target?.[name]`; else `default` if provided; else omit if `required: false`; else render `null`.

Both options are accepted uniformly across regular columns, virtual columns, JSON/JSONB columns, STI `type` discriminators, and non-Dream targets — pick by what you want consumers to see, not by what kind of column the target is. `@deco.BelongsTo('Foo', { optional: true })` paths auto-infer the OpenAPI nullability, so `optional: true` is most often needed for `HasOne` or other non-`BelongsTo` nullable paths. On the STI `type` branch, `default:` is rejected by design — substituting a discriminator string would produce a response indistinguishable from "this is genuinely a record of that type." Use `required: false` to honestly signal absence.

#### Layering two `delegatedAttribute`s onto the same output key

Two `delegatedAttribute` calls can target the same output name — a general
"default value, optionally overridden by something more specific" pattern for
any model with a fallback association and a more-specific one (e.g. a
locale-specific translation over a default-locale one, an org-level setting
over a global default). The builder folds attributes in declaration order, so
a later directive overwrites an earlier one — unless it's skipped:

```typescript
.delegatedAttribute('fallbackAssociation', 'title', { openapi: 'string' })
.delegatedAttribute('specificAssociation', 'title', { openapi: 'string', required: false })
```

- The first (fallback) directive is required, so it always writes `title`.
- The second directive uses `required: false`, not `optional: true`: when
  `specificAssociation` is absent, the key is **omitted** rather than
  overwritten with `null`, leaving the fallback's value in place. `optional:
  true` would still write `title: null` and blank out the fallback.
- Because the fallback directive is required, the merged OpenAPI schema keeps
  `title` as a required string — adding the fallback doesn't change the
  generated client type.
- `preloadFor` resolves both `delegatedAttribute` targets automatically, so no
  controller change is needed to load both associations.

See [i18n.md — Falling Back to a Default Locale](i18n.md#falling-back-to-a-default-locale)
for a worked example using locale-specific vs. default-locale `LocalizedText` rows.

### .rendersOne(name, options?)

Single associated object:

```typescript
// Auto-infer serializer from association's model
.rendersOne('profile')

// Use specific serializer key
.rendersOne('user', { serializerKey: 'summary' })

// Custom serializer function
.rendersOne('owner', { serializer: CustomOwnerSerializer })

// Flatten into parent (merges properties)
.rendersOne('profile', { flatten: true })

// Allow null — only needed when the association is NOT a BelongsTo (e.g., HasOne
// that may be absent). For BelongsTo, the model's `optional: true` declaration is
// auto-inferred; restating it in the serializer is a DRY violation, and changing
// it is almost always a mistake (the model governs runtime nullability).
.rendersOne('approver', { optional: true })
```

**Don't pass `optional: true` to `rendersOne` for a `BelongsTo` association.** The `BelongsTo` declaration on the model — `optional: true` or the default `false` — is the canonical source of truth for whether the association can be null. The renderer auto-infers nullability from that metadata: a `rendersOne('approver')` whose `BelongsTo('User', { optional: true })` declaration says nullable produces exactly the same OpenAPI shape (`anyOf: [{ $ref: ... }, { type: 'null' }]`) as if you had passed `optional: true` explicitly.

```typescript
// Model
@deco.BelongsTo('User', { optional: true })
public approver: User | null

// Serializer — no `optional` needed; auto-inferred from the BelongsTo declaration.
.rendersOne('approver')
```

Restating `optional: true` in the serializer duplicates the declaration; **changing** it — passing `optional: true` when the BelongsTo is non-optional, or vice versa — is almost always a mistake, because the model's declaration governs whether the value can actually be `null` at runtime. Explicit `optional: true` on `rendersOne` is only meaningful when the association is **not** a BelongsTo (e.g., a `HasOne` that may be absent), where there is no model-side nullability for the renderer to infer.

Note: `optional` on `rendersOne` is purely an OpenAPI nullability marker — the key is always rendered. `rendersOne` does not support `required: false`. If you need the key absent from the response, reshape the serializer (a custom attribute, or a different serializer variant) rather than reaching for an option that does not exist.

The non-optional → non-nullable mapping is a contract Dream defends on both sides, so you never need to loosen a serializer to "maybe null" to stay safe. A non-optional `BelongsTo` cannot be conditionally loaded (a trailing constraint on it is a compile error in `preload`/`load`/`leftJoinPreload`/`leftJoinLoad`), and a required parent that goes missing at runtime throws `MissingRequiredBelongsToAssociation` rather than serializing as `null`. Both are signals to fix the model — add `dependent: 'destroy'` to the inverse `HasOne`/`HasMany`, or make the `BelongsTo` `optional: true` — not to mark the serializer field nullable. See [models.md — required BelongsTo contract](models.md).

### .rendersMany(name, options?)

Array of associated objects:

```typescript
// Auto-infer serializer
.rendersMany('rooms')

// Use specific serializer key
.rendersMany('rooms', { serializerKey: 'summary' })
.rendersMany('rooms', { serializerKey: 'forGuests' })

// Custom serializer function (for non-Dream objects)
.rendersMany('bedTypes', { serializer: BedTypeSerializer })
.rendersMany('appliances', { serializer: ApplianceSerializer })
```

### serializerKey Does Not Cascade

The `serializerKey` chosen by the `@OpenAPI` decorator (or by a parent serializer's `rendersOne`/`rendersMany`) does **not** automatically propagate to nested `rendersOne`/`rendersMany` calls. Each nested render defaults to the `'default'` serializer unless an explicit `serializerKey` is provided.

For example, if a controller action uses `serializerKey: 'internal'`, the top-level model's `'internal'` serializer is selected, but any associations it renders will use their `'default'` serializer — not `'internal'` — unless explicitly specified:

```typescript
// The 'internal' serializer for Place
export const PlaceInternalSerializer = (place: Place) =>
  PlaceInternalSummarySerializer(place)
    .rendersMany('rooms')                                  // uses Room's 'default' serializer
    .rendersMany('rooms', { serializerKey: 'internal' })   // uses Room's 'internal' serializer
```

When building a family of serializers around a key (e.g., `'summary'`, `'internal'`, `'forGuests'`), pass the `serializerKey` explicitly at every level.

## Flattening Associations

Flattening merges a `rendersOne` association's attributes directly into the parent response (instead of nesting them under a key). This is useful when a model wants to present a HasOne or BelongsTo association's data as if it were part of itself.

```typescript
.rendersOne('candidate', { serializerKey: 'summary', flatten: true })
```

### Attribute Shadowing

**When flattening, the flattened association's attributes overwrite any same-named attributes on the parent serializer.** The most common collision is `id` — both the parent and the flattened association will typically have an `id` attribute, and the flattened one wins.

**Symptoms:**
- The `id` in the API response is the associated record's ID, not the parent's
- Frontend routing or controller specs use the wrong identity
- Other attributes with the same name (e.g., `name`, `createdAt`) have unexpected values

### Fixing Attribute Shadowing

**Fix 1 (preferred when the shadowing attribute from the flattened association is not needed):** Create a dedicated flattenable serializer that omits the conflicting attribute (e.g., `id`). Build the default serializer from the flattenable serializer, adding the omitted attribute back. This changes the normal composition pattern where the default serializer extends the summary serializer, so some attributes may need to be duplicated.

```typescript
// CandidateSerializers
export const CandidateSummarySerializer = (candidate: Candidate) =>
  DreamSerializer(Candidate, candidate)
    .attribute('id')
    .attribute('name')

export const FlattenableCandidateSerializer = (candidate: Candidate) =>
  DreamSerializer(Candidate, candidate)
    .attribute('name')
    .attribute('bio')

export const CandidateSerializer = (candidate: Candidate) =>
  FlattenableCandidateSerializer(candidate)
    .attribute('id')

// ClientSerializer — flattens candidate data without the candidate's id
export const ClientSerializer = (client: Client) =>
  DreamSerializer(Client, client)
    .attribute('id')
    .rendersOne('candidate', { serializer: FlattenableCandidateSerializer, flatten: true })
```

**Fix 2 (preferred when the shadowed attribute from the parent is not needed — common for join models):** Simply omit the shadowed attribute from the parent serializer. This is typical when the parent is a join model (e.g., `HostPlace`) where the join model's own `id` is not useful to the consumer, and the real identity is the flattened association's `id`.

```typescript
// HostPlace only contributes `position`; Place's attributes (including id) are flattened in
const HostPlaceSerializer = (hostPlace: HostPlace) =>
  DreamSerializer(HostPlace, hostPlace)
    .attribute('position')
    .rendersOne('place', { flatten: true })
```

### Controller Spec Expectations with Flattening

When a serializer uses flattening, controller specs should assert the serialized API contract, not the raw model shape:

```typescript
// If ClientSerializer flattens candidate data and exposes:
//   - client's id as 'id' would be shadowed, so client omits it or renames it
//   - candidate's 'id', 'name', 'bio' flattened into the response
// Then specs should assert against the serialized shape, not model property names
```

## Composition Pattern

Build complex serializers by extending simpler ones:

```typescript
// Layer 1: Summary
export const PlaceSummarySerializer = (place: Place) =>
  DreamSerializer(Place, place)
    .attribute('id')
    .attribute('name')

// Layer 2: Default (extends summary)
export const PlaceSerializer = (place: Place) =>
  PlaceSummarySerializer(place)
    .attribute('style')
    .attribute('sleeps')
    .attribute('deletedAt')

// Layer 3: Admin (extends default)
export const PlaceAdminSerializer = (place: Place) =>
  PlaceSerializer(place)
    .attribute('createdAt')
    .attribute('updatedAt')
    .rendersMany('hostPlaces')
```

## Action-Specific Serializer Keys

When a model has virtual attributes that are only populated in certain contexts (e.g., upload URLs set during creation but absent when loading from DB), don't include those attributes in the general-purpose serializer. Instead, create a separate serializer that extends the base one and register it under its own key.

This avoids OpenAPI response validation failures when the virtuals are undefined in other actions (e.g., show/index).

```typescript
// DocumentSerializer.ts — base serializer omits context-specific fields
export const PlacePhotoSerializer = (PlacePhoto: PlacePhoto) =>
  PlacePhotoSummarySerializer(PlacePhoto)

// Create-specific serializer adds upload fields only present after creation
export const PlacePhotoCreateSerializer = (PlacePhoto: PlacePhoto) =>
  PlacePhotoSerializer(PlacePhoto)
    .attribute('uploadUrl')
    .attribute('uploadHeaders')
```

Register the action-specific serializer under its own key:

```typescript
// Document.ts model
public get serializers(): DreamSerializers<PlacePhoto> {
  return {
    default: 'Place/PhotoSerializer',
    create: 'Place/PhotoCreateSerializer',
  }
}
```

Use that key in the controller action:

```typescript
// DocumentsController.ts
@OpenAPI(PlacePhoto, { status: 201, serializerKey: 'create' })
public async create() {
  // ... creation logic that populates uploadUrl/uploadHeaders ...
  this.created(Document)
}
```

After adding a new serializer key, run `pnpm psy sync` so the global types pick up the new key.

## Passthrough Context

Serializers can receive context data (locale, current user, etc.):

```typescript
export const PlaceForGuestsSerializer = (place: Place, passthrough: { locale: LocalesEnum }) =>
  DreamSerializer(Place, place)
    .attribute('id')
    .attribute('name')
    .customAttribute('displayStyle',
      () => i18n(passthrough.locale, `places.style.${place.style}`),
      { openapi: 'string' }
    )
    .rendersMany('rooms', { serializerKey: 'forGuests' })
```

In controllers, set passthrough via:
```typescript
this.serializerPassthrough({ locale: this.locale })
```

Or via query passthrough:
```typescript
await Place.passthrough({ locale: 'en-US' }).preloadFor('forGuests').all()
```

## ObjectSerializer (for non-Dream objects)

For plain objects, use ObjectSerializer. **All attributes require explicit `openapi` type.** When a controller returns a stable computed / view-model object, create an ObjectSerializer and pass that serializer function to `@OpenAPI(SerializerFn, { status })` instead of hand-writing `responses[status].properties`.

```typescript
import { ObjectSerializer } from '@rvoh/dream'

export const BedTypeSerializer = (bedType: BedTypesEnum, passthrough: { locale: LocalesEnum }) =>
  ObjectSerializer({ bedType }, passthrough)
    .attribute('bedType', {
      as: 'value',
      openapi: { type: 'string', enum: BedTypesEnumValues },
    })
    .customAttribute('label',
      () => i18n(passthrough.locale, `rooms.Bedroom.bedTypes.${bedType}`),
      { openapi: 'string' }
    )
```

For OpenAPI-visible nested computed/view-model response shapes, export every ObjectSerializer that is passed to another serializer's `rendersOne` or `rendersMany`. Exported serializers are registered as named OpenAPI schemas. Local non-exported nested `const` serializers can generate anonymous `Unnamed` schemas; multiple nested shapes may collapse into the same anonymous type and cause generated clients to lose fields.

Runtime serializer global names include the serializer path, so the same exported function name in different directories does not by itself cause `SerializerNameConflict`. OpenAPI component names for named exports are based on the export name, though, so two OpenAPI-visible serializers with the same exported function name can still collide in the generated schema. Give computed/view-model serializers distinct export names, such as a `ViewSerializer` suffix, when the domain noun overlaps a Dream model serializer.

### A compound response is still one serializer

When an action returns a hand-shaped envelope — say a record plus a related collection, `{ place, nearby }`, or a computed array alongside serialized models — model the whole envelope as one composing `ObjectSerializer` that renders each part, and pass it to `@OpenAPI(SerializerFn)`. Don't reach for a hand-written `responses` block. `rendersOne` / `rendersMany` take a serializer function via `{ serializer }`, or render a Dream-model field by key via `{ dreamClass, serializerKey }`:

```typescript
// Action returns { place: Place; nearby: Place[] }
export const PlaceWithNearbySerializer = (place: Place, nearby: Place[]) =>
  ObjectSerializer({ place, nearby })
    .rendersOne('place', { serializer: PlaceSummarySerializer })
    .rendersMany('nearby', { serializer: PlaceSummarySerializer })
```

The action is then `@OpenAPI(PlaceWithNearbySerializer, { status: 200 })` over `this.ok(PlaceWithNearbySerializer(place, nearby))`. The schema is derived and validated under test, with no hand-maintained JSON Schema to drift. Even a one-off envelope is worth this — a composing serializer stays the single source of truth where a hand-written `responses` block does not.

### Rendering an async-computed shape on the model

`rendersOne` / `rendersMany` accept any declared property, not only associations, so a shape that must be computed asynchronously can be assigned in the controller and rendered as a field of the model itself. Prefer the compound envelope above; reach for this only when the computed shape has to sit *inside* the model's own object — typically when those models render as a collection, where an envelope cannot reach individual items. Either way, don't hand-write `openapi` for the nested shape ([Critical Rule 21](SKILL.md#critical-rules)).

```typescript
// Place.ts — a declared property, not a column (columns stay bare — see models.md)
public availabilitySummary: PlaceAvailabilitySummaryView | null = null

// PlaceSerializer.ts — `serializer` is required for a non-Dream, non-ViewModel value
.rendersOne('availabilitySummary', { serializer: PlaceAvailabilitySummaryViewSerializer, optional: true })

// PlacesController.ts
place.availabilitySummary = await PlaceAvailabilityService.summarize(place)
this.ok(place)
```

`optional: true` keeps actions that leave it unassigned from failing response validation. Export the nested ObjectSerializer so it registers as a named OpenAPI component.

## STI Serializers

For Single Table Inheritance, use generic type parameter:

```typescript
import Room from '@models/Room.js'
import { DreamSerializer } from '@rvoh/dream'

// Base serializer uses generic
export const RoomSummarySerializer = <T extends Room>(StiChildClass: typeof Room, room: T) =>
  DreamSerializer(StiChildClass ?? Room, room)
    .attribute('id')
    .attribute('type', { openapi: { type: 'string', enum: [(StiChildClass ?? Room).sanitizedName] } })
    .attribute('position')

export const RoomSerializer = <T extends Room>(StiChildClass: typeof Room, room: T) =>
  RoomSummarySerializer(StiChildClass, room)
    .attribute('deletedAt')
```

STI child serializers pass their class:

```typescript
import Bedroom from '@models/Room/Bedroom.js'
import { RoomSerializer, RoomSummarySerializer } from '@serializers/RoomSerializer.js'

export const RoomBedroomSummarySerializer = (bedroom: Bedroom) =>
  RoomSummarySerializer(Bedroom, bedroom)

export const RoomBedroomSerializer = (bedroom: Bedroom) =>
  RoomSerializer(Bedroom, bedroom)
    .attribute('bedTypes')  // Bedroom-specific field
```

### Generic Parameter on `rendersOne`, `rendersMany`, and `delegatedAttribute`

When an STI child serializer extends an STI base serializer and calls `rendersOne`, `rendersMany`, or `delegatedAttribute`, it must pass the STI child class as a generic parameter. Without it, TypeScript cannot resolve the association types correctly because the base serializer's generic context is lost at the call site.

`.attribute` infers types correctly without a generic argument, and `.customAttribute` doesn't do type inference, so neither needs this.

```typescript
// CORRECT — pass <Bedroom> so rendersMany can resolve Bedroom's associations
export const RoomBedroomForGuestsSerializer = (bedroom: Bedroom, passthrough: { locale: LocalesEnum }) =>
  RoomForGuestsSerializer(Bedroom, bedroom, passthrough)
    .rendersMany<Bedroom>('bedTypes', { serializer: BedTypeSerializer })

// CORRECT — pass <Bathroom> for rendersOne
export const RoomBathroomForGuestsSerializer = (bathroom: Bathroom, passthrough: { locale: LocalesEnum }) =>
  RoomForGuestsSerializer(Bathroom, bathroom, passthrough)
    .rendersOne<Bathroom>('bathOrShowerStyle', { serializer: BathOrShowerStyleSerializer })

// WRONG — omitting the generic causes type errors
  .rendersMany('bedTypes', { serializer: BedTypeSerializer })
```

## preloadFor Integration

**Prefer `preloadFor` or `loadFor`** over manual `preload(...)` chains. These methods automatically load everything the serializer needs, including nested `rendersOne`, `rendersMany`, and `delegatedAttribute` associations. Manual preload is fragile — it misses nested dependencies, causing `NonLoadedAssociation` errors at serialization time. Don't switch to manual preload to avoid an extra DB query without data showing it's a real problem. `preloadFor` automatically adapts as serializers evolve — if you add a `rendersMany` to a serializer, every controller using `preloadFor` picks it up automatically. Manual preload chains don't, leading to production `NonLoadedAssociation` errors and harder-to-understand code. Let data lead optimization decisions, not intuition about query counts. Manual `preload(...)` and `load(...)` are still useful outside of serialization contexts (e.g., loading associations for business logic in a service).

To put a value from an associated record on a serializer, use `delegatedAttribute` or `rendersOne(name, { flatten: true })` — both are discovered by `preloadFor`, so the association is auto-preloaded. (`customAttribute` is for values computed from the model's own columns and passthrough, like localized labels; it's the right tool there.) `preloadFor` does not inspect `customAttribute` bodies, so reading an association inside one leaves it unloaded and rendering throws `NonLoadedAssociation` — read that error as the signal to switch to a delegate or a flattened `rendersOne`, not to hand-`.preload()` the association.

**STI-aware preloading:** `preloadFor` on an STI base class discovers associations from every STI child's serializer, not just the base. If different children declare different `rendersOne`, `rendersMany`, or `delegatedAttribute` associations, `preloadFor` loads them all. At query time, Dream partitions the loaded records by actual STI child class and preloads each group using that child's own association definitions (including any `and` clauses or overrides). This means a single `preloadFor` call on the base class correctly handles mixed result sets of different STI children.

```typescript
// In controller
const places = await Place.preloadFor('forGuests').all()
// This automatically preloads 'rooms', 'currentLocalizedText', etc.

// Through association query
const places = await host
  .associationQuery('places')
  .preloadFor('summary')
  .cursorPaginate({ cursor })

// Load on existing instance
const place = await Place.findOrFail(id)
await place.loadFor('default').execute()

// After a transaction — loadFor outside the transaction
let place = await ApplicationModel.transaction(async txn => {
  const place = await Place.txn(txn).create({ ... })
  await HostPlace.txn(txn).create({ host: this.currentHost, place })
  return place
})
if (place.isPersisted) place = await place.loadFor('default').execute()
this.created(place)
```

### Overriding Individual Associations in preloadFor

**This pattern adds complexity and should only be used when data shows it is warranted.** In the example below, collapsing Place and Room `currentLocalizedText` into one query saves only a single DB call — likely not worth the added code. But when 10 different models in the preload tree each have their own `currentLocalizedText`, replacing 10 queries with 1 may justify the complexity. Let data guide the decision.

`preloadFor` accepts a second argument — a modifier callback that runs for each association in the preload tree. The callback can return `undefined` (use default behavior), `{ and?, andAny?, andNot? }` (add conditions), or `'omit'` (skip this association entirely so you can load it yourself).

Use `'omit'` when the default preload would issue repeated queries that could be collapsed into one — commonly for polymorphic associations like `currentLocalizedText` where the lookup depends on request-scoped context (e.g., locale):

```typescript
import { LoadForModifierFn } from '@rvoh/dream/types'

// 1. preloadFor, but skip currentLocalizedText so we can batch-load it across
//    multiple polymorphic types (Place + Room) in a single query
const skipCurrentLocalizedText: LoadForModifierFn = (associationName) =>
  associationName === 'currentLocalizedText' ? 'omit' : undefined

const places = await Place.query()
  .preloadFor('forGuests', skipCurrentLocalizedText)
  .all()

// 2. Batch-load in ONE query across all places AND their rooms
const rooms = places.flatMap(place => place.rooms)
await assignCurrentLocalizedText(places, rooms, this.locale)
```

```typescript
// Helper — loads LocalizedTexts for multiple polymorphic types in one query
async function assignCurrentLocalizedText(places: Place[], rooms: Room[], locale: string) {
  if (places.length === 0 && rooms.length === 0) return

  // Single query across both polymorphic types
  const texts = await LocalizedText.where({ locale })
    .whereAny([
      { localizableType: 'Place', localizableId: places.map(p => p.id) },
      { localizableType: 'Room', localizableId: rooms.map(r => r.id) },
    ]).all()

  // Index by (type, id) for O(1) lookup
  const byKey = new Map(texts.map(t => [`${t.localizableType}:${t.localizableId}`, t]))

  // Direct assignment marks the association as loaded — serializers see it as preloaded
  for (const place of places) {
    place.currentLocalizedText = byKey.get(`Place:${place.id}`) ?? null
  }
  for (const room of rooms) {
    room.currentLocalizedText = byKey.get(`Room:${room.id}`) ?? null
  }
}
```

The purpose of this pattern is to collapse queries across **multiple polymorphic types** into one. If only a single type is involved, `preloadFor` already handles it — the `omit` override is for cases where the same polymorphic association (like `currentLocalizedText`) appears on multiple models in the preload tree (Place, Room, Host) and you want one query instead of one per type.

Key points:
- Assign `null` (not `undefined`) for missing associations — `undefined` causes `NonLoadedAssociation` errors.
- The modifier callback receives `(associationName, dreamClass)`. Use `dreamClass.typeof(Place)` to scope the omit when needed. `typeof` is Dream's class-level equivalent of `instanceof` — use it when comparing a **class** against another class (since `instanceof` only works on instances):
  ```typescript
  const skip: LoadForModifierFn = (assoc, dreamClass) =>
    dreamClass.typeof(Place) && assoc === 'currentLocalizedText' ? 'omit' : undefined
  ```
- If the omitted association itself has nested associations via `rendersOne`/`rendersMany` in its serializer, those are also pruned — load them in your batch helper too.

## File Organization

```
src/app/serializers/
  PlaceSerializer.ts          # PlaceSummarySerializer, PlaceSerializer, PlaceForGuestsSerializer
  HostSerializer.ts           # HostSummarySerializer, HostSerializer
  RoomSerializer.ts           # RoomSummarySerializer, RoomSerializer (generic base)
  Room/
    BedroomSerializer.ts      # RoomBedroomSummarySerializer, RoomBedroomSerializer
    KitchenSerializer.ts      # RoomKitchenSummarySerializer, RoomKitchenSerializer
  LocalizedTextSerializer.ts  # LocalizedTextSerializer
```

## N+1 Prevention

Serializers are **synchronous by design** - they cannot make database queries. This forces you to preload all needed data upfront:

```typescript
// CORRECT: Preload before serializing
const places = await Place.preloadFor('forGuests').all()
this.ok(places)  // Serializer accesses already-loaded associations

// WRONG: Would cause N+1 (serializers can't query)
const places = await Place.all()
this.ok(places)  // Associations not loaded -> empty/missing data
```

## Sync After Serializer Changes

**Always run `pnpm psy sync` after changing serializer attributes, field names, or OpenAPI-visible response shapes.** This regenerates:

- OpenAPI specs
- Controller spec request/response typings
- Generated frontend API clients (if configured via `setup:sync:openapi-*`)

**Symptoms of a missing sync:**
- Controller specs have type errors against stale response fields
- Generated frontend clients don't reflect renamed or added serializer attributes
