# Testing Dream & Psychic Applications - Detailed Reference

## Overview

Tests use **Vitest** with real database records (not mocks). We practice **BDD, not TDD** - focus expectations on outcomes, not implementation. For code added independently of a generator, always write a failing spec first, then implement. Generated code is the only exception (generators create scaffolding for specs and implementation simultaneously).

## Test Types

- **Unit specs**: Model logic, controller responses (`spec/unit/`)
- **Feature specs**: End-to-end browser tests (`spec/features/`)
- **Factories**: Test data creation (`spec/factories/`)

## Running Tests

```bash
pnpm uspec                                    # All unit specs
pnpm uspec spec/unit/models/Place.spec.ts    # Single file
pnpm fspec                                    # Feature specs (headless)
pnpm fspec:visible                            # Feature specs (visible browser)
```

**Run specific specs within a file** using `.only` or `.skip`:
```typescript
it.only('returns the index of Places', async () => { ... })  // Run only this
it.skip('returns the index of Places', async () => { ... })  // Skip this
// Also works on describe and context blocks
```

## Factory Pattern

Factories create real database records with sensible defaults:

```typescript
// spec/factories/PlaceFactory.ts
import Place from '@models/Place.js'
import { UpdateableProperties } from '@rvoh/dream/types'

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

Factory conventions:
- Export a default async function named `create{ModelName}`
- Accept optional `attrs` with `UpdateableProperties<Model>` type
- Use counter for unique values
- Provide sensible defaults for all required fields
- Accept overrides via spread

### Factories with Associations

```typescript
// spec/factories/HostPlaceFactory.ts
import HostPlace from '@models/HostPlace.js'
import createHost from './HostFactory.js'
import createPlace from './PlaceFactory.js'
import { UpdateableProperties } from '@rvoh/dream/types'

export default async function createHostPlace(
  attrs: UpdateableProperties<HostPlace> & { host?: Host; place?: Place } = {}
) {
  const host = attrs.host ?? await createHost()
  const place = attrs.place ?? await createPlace()

  return await HostPlace.create({
    host,
    place,
    ...attrs,
  })
}
```

### STI Factories

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

## Model Specs

```typescript
import Place from '@models/Place.js'
import createPlace from '@factories/PlaceFactory.js'
import createHost from '@factories/HostFactory.js'
import createHostPlace from '@factories/HostPlaceFactory.js'

describe('Place', () => {
  // Test associations
  describe('associations', () => {
    it('has many hosts through hostPlaces', async () => {
      const host = await createHost()
      const place = await createPlace()
      await createHostPlace({ host, place })

      expect(await place.associationQuery('hosts').all()).toMatchDreamModels([host])
    })

    it('has many rooms', async () => {
      const place = await createPlace()
      const bedroom = await createBedroom({ place })
      const kitchen = await createKitchen({ place })

      const rooms = await place.associationQuery('rooms').all()
      expect(rooms).toMatchDreamModels([bedroom, kitchen])
    })
  })

  // Test hooks
  describe('hooks', () => {
    context('upon creation', () => {
      it('creates default LocalizedText', async () => {
        const place = await createPlace({ style: 'cottage' })
        const text = await place.associationQuery('localizedTexts').firstOrFail()
        expect(text.locale).toEqual('en-US')
        expect(text.title).toEqual('My cottage')
      })
    })
  })

  // Test soft deletes
  describe('soft delete', () => {
    it('soft deletes and cascades to dependents', async () => {
      const place = await createPlace()
      const hostPlace = await createHostPlace({ place })
      const room = await createBedroom({ place })

      await place.destroy()

      // Soft deleted - hidden from default queries
      expect(await Place.where({ id: place.id }).exists()).toBe(false)
      // Still in database
      expect(await Place.where({ id: place.id }).removeAllDefaultScopes().exists()).toBe(true)

      // Cascaded to dependents
      expect(await HostPlace.where({ id: hostPlace.id }).exists()).toBe(false)
    })
  })

  // Test validations
  describe('validations', () => {
    it('requires name', async () => {
      const place = Place.new({ style: 'cottage', sleeps: 1 })
      expect(place.isInvalid).toBe(true)
      expect(place.errors.name).toBeDefined()
    })

    it('validates sleeps is positive', async () => {
      const place = Place.new({ name: 'Test', style: 'cottage', sleeps: -1 })
      expect(place.isInvalid).toBe(true)
      expect(place.errors.sleeps).toBeDefined()
    })
  })

  // Test scopes
  describe('scopes', () => {
    it('.active returns only active places', async () => {
      const active = await createPlace({ status: 'active' })
      const inactive = await createPlace({ status: 'inactive' })

      const results = await Place.scope('active').all()
      expect(results).toMatchDreamModels([active])
    })
  })

  // Test virtual attributes
  describe('#age', () => {
    it('computes age from birthYear', async () => {
      const candidate = await createCandidate({ birthYear: 1990 })
      const reloaded = await Candidate.findOrFail(candidate.id)
      expect(reloaded.age).toEqual(CalendarDate.today().year - 1990)
    })
  })
})
```

## Controller Specs

Controller specs use `OpenapiSpecRequest` for type-safe HTTP testing:

```typescript
import { PsychicServer } from '@rvoh/psychic'
import { OpenapiSpecRequest } from '@rvoh/psychic-spec-helpers'
import { session } from '@spec/unit/helpers/authentication.js'
import createUser from '@factories/UserFactory.js'
import createHost from '@factories/HostFactory.js'
import createPlace from '@factories/PlaceFactory.js'
import createHostPlace from '@factories/HostPlaceFactory.js'

type SpecRequestType = Awaited<ReturnType<typeof session>>

describe('V1/Host/PlacesController', () => {
  let request: SpecRequestType
  let user: User
  let host: Host

  beforeEach(async () => {
    user = await createUser()
    host = await createHost({ user })
    request = await session(user)
  })

  describe('GET /v1/host/places', () => {
    it('returns paginated places for this host', async () => {
      const place = await createPlace()
      await createHostPlace({ host, place })

      const { body } = await request.get('/v1/host/places', 200)
      expect(body.results).toEqual([
        expect.objectContaining({ id: place.id, name: place.name }),
      ])
    })

    context('places created by another host', () => {
      it('are omitted', async () => {
        const otherHost = await createHost()
        const otherPlace = await createPlace()
        await createHostPlace({ host: otherHost, place: otherPlace })

        const { body } = await request.get('/v1/host/places', 200)
        expect(body.results).toEqual([])
      })
    })
  })

  describe('GET /v1/host/places/:id', () => {
    it('returns the place', async () => {
      const place = await createPlace()
      await createHostPlace({ host, place })

      const { body } = await request.get(`/v1/host/places/${place.id}`, 200)
      expect(body.id).toEqual(place.id)
      expect(body.name).toEqual(place.name)
    })

    context('when place does not exist', () => {
      it('returns 404', async () => {
        await request.get('/v1/host/places/nonexistent-id', 404)
      })
    })
  })

  describe('POST /v1/host/places', () => {
    it('creates a Place', async () => {
      const { body } = await request.post('/v1/host/places', 201, {
        data: { name: 'Cozy Cabin', style: 'cabin', sleeps: 4 },
      })

      const place = await host.associationQuery('places').firstOrFail()
      expect(place.name).toEqual('Cozy Cabin')
      expect(place.style).toEqual('cabin')
      expect(place.sleeps).toEqual(4)
      expect(body).toEqual(expect.objectContaining({ id: place.id }))
    })

    context('with invalid params', () => {
      it('returns 422', async () => {
        await request.post('/v1/host/places', 422, {
          data: { style: 'cabin', sleeps: 4 },  // Missing required 'name'
        })
      })
    })
  })

  describe('PATCH /v1/host/places/:id', () => {
    it('updates the place', async () => {
      const place = await createPlace()
      await createHostPlace({ host, place })

      await request.patch(`/v1/host/places/${place.id}`, 204, {
        data: { name: 'Updated Name' },
      })

      await place.reload()
      expect(place.name).toEqual('Updated Name')
    })
  })

  describe('DELETE /v1/host/places/:id', () => {
    it('soft deletes the place', async () => {
      const place = await createPlace()
      await createHostPlace({ host, place })

      await request.delete(`/v1/host/places/${place.id}`, 204)

      expect(await Place.where({ id: place.id }).exists()).toBe(false)
    })
  })
})
```

## Authentication Helper

```typescript
// spec/unit/helpers/authentication.ts
import { Encrypt } from '@rvoh/dream'
import { OpenapiSpecRequest } from '@rvoh/psychic-spec-helpers'
import { PsychicServer } from '@rvoh/psychic'
import AppEnv from '@conf/AppEnv.js'

type OpenapiPaths = import('@src/types/openapi/tests.openapi.js').paths

function testToken(user: Dream): string {
  return Encrypt.encrypt(
    JSON.stringify({ userId: user.primaryKeyValue() }),
    { algorithm: 'aes-256-gcm', key: AppEnv.string('APP_ENCRYPTION_KEY') }
  )
}

export async function session(user: Dream) {
  const request = new OpenapiSpecRequest<OpenapiPaths>()
  await request.init(PsychicServer)
  const bearerToken = testToken(user)
  return request.setDefaultHeaders({ Authorization: `Bearer ${bearerToken}` })
}
```

## Key Test Matchers

```typescript
// Dream model matching
expect(result).toMatchDreamModels([model1, model2])  // Match by ID
expect(result).toMatchDreamModel(model)               // Single model

// Standard vitest matchers
expect(body).toEqual(expect.objectContaining({ id: place.id }))
expect(body.results).toHaveLength(1)
expect(place.name).toEqual('Expected Name')
```

## Spec Organization

### When spec'ing a function
Use `describe` for the outermost block and `context` blocks for different state:
```typescript
describe('calculatePrice', () => {
  context('when the item is on sale', () => {
    it('applies the discount', async () => { ... })
  })
})
```

### When spec'ing a class
Use `describe` for the class, `describe` for each public method (`.staticMethod` or `#instanceMethod`), and `context` for state:
```typescript
describe('Place', () => {
  describe('#destroy', () => {
    context('when the place has rooms', () => {
      it('cascades the soft delete', async () => { ... })
    })
  })
})
```

### Keep DRY with spec setup
Leverage nested `context` blocks to change only the one thing being tested:
```typescript
describe('V1/Host/PlacesController', () => {
  describe('POST create', () => {
    let options: CreatePlaceOptions

    beforeEach(async () => {
      // Set happy path defaults
      options = { name: 'Cozy Cabin', style: 'cabin', sleeps: 4 }
    })

    it('creates a Place', async () => { ... })

    context('when sleeps is negative', () => {
      beforeEach(() => { options.sleeps = -1 })
      it('returns 422', async () => { ... })
    })
  })
})
```

## Testing Principles

1. **Use real models** - Create records via factories, not mocks
2. **Don't stub Dream internals** - Never mock `.find()`, `.create()`, `.loaded()`, etc.
3. **Test behavior, not implementation** - Assert on outcomes, not internal calls
4. **Don't spec behavior of another class that is already spec'd** - Use vitest spies to return different values instead
5. **In controller specs, use factories to create real models** - Let controllers leverage real Dream queries (never mocked). If a spec'd service/view-model fetches or transforms the data, you may mock it, but ensure its own spec covers the full variety of cases
6. **Test authorization** - Verify users can only access their own resources
7. **Test soft deletes by testing behavior** - Verify record is hidden (normal query) AND still present when scopes are removed
8. **Use Polly** (`setupPolly`) for recording and replaying external API calls rather than stubbing

## Background Worker Testing

```typescript
// Default: Jobs execute immediately in tests (testInvocation = 'automatic')
await EmailService.background('sendWelcome', user.id)
// Method runs synchronously

// Manual mode for fine-grained control
import { WorkerTestUtils } from '@rvoh/psychic-workers'

const workersApp = PsychicAppWorkers.getOrFail()
workersApp.set('testInvocation', 'manual')

await EmailService.background('sendWelcome', user.id)  // Queued
await WorkerTestUtils.work()                            // Process queue
WorkerTestUtils.clean()                                 // Clear queues
```

## Feature Spec Pattern

```typescript
describe('Host creates a Place', () => {
  let page: Page

  beforeEach(async () => {
    page = await getPage()
    const user = await createUser()
    await createHost({ user })
    await hostSignIn(page, user)
  })

  it('creates a new place via the form', async () => {
    await visit(page, '/places/new')
    await fillIn(page, '#name', 'Cozy Cabin')
    await select(page, '#style', 'cabin')
    await fillIn(page, '#sleeps', '4')
    await clickButton(page, 'Create Place')

    await expect(page).toHavePath('/places')
    await expect(page).toMatchTextContent('Cozy Cabin')

    const place = await Place.firstOrFail()
    expect(place.name).toEqual('Cozy Cabin')
  })
})
```
