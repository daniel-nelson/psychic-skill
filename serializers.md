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

```typescript
import Place from '@models/Place.js'
import { DreamSerializer } from '@rvoh/dream'

// Summary (minimal) serializer
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

Serializer names are **strings** matching the exported function names. They're loaded globally via `app.load('serializers', ...)` in `conf/dream.ts`.

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

### .delegatedAttribute(associationName, propertyName, options?)

Access a property on a loaded association:

```typescript
// Gets `place.currentLocalizedText.title`
.delegatedAttribute('currentLocalizedText', 'title', { openapi: 'string' })

// Gets `user.profile.avatarUrl`
.delegatedAttribute('profile', 'avatarUrl', { openapi: 'string' })
```

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

// Allow null
.rendersOne('approver', { optional: true })
```

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

For plain objects, use ObjectSerializer. **All attributes require explicit `openapi` type.**

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

## preloadFor Integration

The `preloadFor(serializerKey)` method analyzes the serializer and automatically preloads all needed associations:

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
```

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
