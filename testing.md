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

**Always run migrations before specs.** After generating a resource or migration, run `pnpm psy db:migrate` before `pnpm uspec` or `pnpm fspec`. The integrity check will fail with "Migrations need to be run on default database" if migrations are pending.

**Feature spec server management:** `pnpm fspec` automatically starts AND stops the frontend servers (client, admin, internal). Do NOT manually start them before running fspec — that will cause fspec to fail because it can't bind the ports.

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

### Factories for AfterCreate Auto-Created Associations

When a model's AfterCreate hook automatically creates an associated record (e.g., User auto-creates Guest), the factory for the associated model should return the auto-created record, NOT create a duplicate:

```typescript
// BAD — creates a duplicate Guest, may violate unique constraint on userId
export default async function createGuest(attrs = {}) {
  return await Guest.create({ user: await createUser(), ...attrs })
}

// GOOD — returns the Guest auto-created by User's AfterCreate hook
export default async function createGuest(attrs = {}) {
  const user = attrs.user ? null : await createUser()
  const userId = attrs.user?.id ?? user!.id
  const guest = await Guest.findBy({ userId })
  if (!guest) throw new Error(`Guest not found for userId ${userId}`)
  return guest
}
```

### Factories for Models with Side-Effect Hooks

When a model's AfterCreate hook creates side-effect records that aren't a single auto-associated child (e.g., Place auto-creates default rooms, default photos, default amenities — multiple records), the factory should pass `skipHooks: true` so test data stays minimal and predictable:

```typescript
// PlaceFactory — bypasses the AfterCreate hook that auto-seeds rooms.
// Tests that want to verify the auto-seed flow should call Place.create() directly.
export default async function createPlace(attrs: UpdateableProperties<Place> = {}) {
  return await Place.create(
    {
      host: attrs.host ?? (await createHost()),
      name: 'Test Place',
      ...attrs,
    },
    { skipHooks: true },
  )
}
```

**Why:** if the factory fires the hook, every spec that creates a Place gets the auto-seeded rooms. Index queries, count assertions, and isolation tests break in confusing ways. The factory's job is to provide a minimal valid record; tests that need the full hook flow can call `Model.create()` directly without `skipHooks`.

**Test the hook itself in the model spec.** Have explicit tests for both paths:

```typescript
it('auto-seeds default rooms', async () => {
  const place = await Place.create({ host: await createHost(), name: 'Test' })
  const rooms = await place.associationQuery('rooms').all()
  expect(rooms.length).toBeGreaterThan(0)
})

it('does NOT auto-seed when skipHooks is true', async () => {
  const place = await Place.create({ host: await createHost(), name: 'Test' }, { skipHooks: true })
  const rooms = await place.associationQuery('rooms').all()
  expect(rooms).toEqual([])
})
```

**When to use which pattern:**
- **Return the auto-created record** when the hook creates a single 1:1 child with a unique constraint (e.g., User → Guest). Calling the factory directly would violate the constraint.
- **Bypass with `skipHooks`** when the hook creates multiple side-effect records that pollute test data and aren't usually needed by every spec.

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

## Feature Spec Matchers

Feature specs use assertion-style matchers from `@rvoh/psychic-spec-helpers` on the globally provided `page` object. All matchers are async and used with `await expect(page)`.

### Action Matchers

These attempt an action and fail the test if the target element isn't found:

```typescript
await expect(page).toClick('Submit')              // Click element with text
await expect(page).toClickButton('Save Changes')  // Click a <button> with text
await expect(page).toClickLink('View Details')     // Click an <a> tag with text
await expect(page).toClickSelector('.submit-btn')  // Click element by CSS selector
await expect(page).toFill('#email', 'a@b.com')    // Fill a text field
await expect(page).toSelect('#country', 'us')     // Select an option from a dropdown
await expect(page).toCheck('Terms of Service')     // Check a checkbox
await expect(page).toUncheck('Send newsletter')    // Uncheck a checkbox
```

### Expectation Matchers

These assert on the current state of the page:

```typescript
await expect(page).toMatchTextContent('Welcome back')      // Page contains text
await expect(page).toNotMatchTextContent('Error')           // Page does not contain text
await expect(page).toHaveSelector('.success-banner')        // Page has CSS selector
await expect(page).toNotHaveSelector('.error-message')      // Page does not have selector
await expect(page).toHavePath('/dashboard')                 // Current path matches
await expect(page).toHaveUrl('https://example.com/dash')   // Current full URL matches
await expect(page).toHaveLink('Documentation')              // Page has a link with text
await expect(page).toHaveChecked('Remember me')             // Checkbox with text is checked
await expect(page).toHaveUnchecked('Opt out')               // Checkbox with text is unchecked
```

### Full Example

```typescript
import City from '@models/City.js'
import createAdminUser from '@spec/factories/AdminUserFactory.js'
import adminSignIn from '@spec/features/helpers/adminSignIn.js'

describe('Cities create', () => {
  it('allows an admin to create a new city from the index', async () => {
    const adminUser = await createAdminUser()

    await adminSignIn(adminUser, '/cities')

    await expect(page).toClickLink('New City')
    await expect(page).toHavePath('/cities/new')

    await expect(page).toFill('#name', 'Denver')
    await expect(page).toFill('#stateOrProvince', 'Colorado')
    await expect(page).toSelect('#country', 'united_states')
    await expect(page).toClickButton('Create City')

    await expect(page).toHavePath('/cities')
    await expect(page).toMatchTextContent('Denver')
    await expect(page).toMatchTextContent('Colorado')
  })
})
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

## Organize Feature Specs by Actor and Scenario

Follow the BDD convention from Cucumber/RSpec: organize by **actor** (role/persona), with filenames as **third-person verb phrases** that describe what the actor does. The directory provides the subject; the filename completes the sentence — read as "**guest** browses places", "**host** creates a place".

```
spec/features/
  guest/
    places/
      browses-places.spec.ts
      views-place.spec.ts
      searches-places.spec.ts
      books-place.spec.ts
    favorites/
      manages-favorites.spec.ts
  host/
    places/
      creates-place.spec.ts
      updates-place.spec.ts
      deletes-place.spec.ts
      views-places.spec.ts
    rooms/
      creates-room.spec.ts
      updates-room.spec.ts
      deletes-room.spec.ts
      views-rooms.spec.ts
  visitor/
    places/
      browses-places.spec.ts
      searches-places.spec.ts
      views-place.spec.ts
    sign-up/
      signs-up-from-booking.spec.ts
      signs-up-from-favorite.spec.ts
```

Avoid flat naming like `guest-browses-places.spec.ts` — the directory structure carries the actor context. Avoid bare CRUD names like `create.spec.ts` or `index.spec.ts` — those are resource-oriented, not behavior-oriented.


### Debugging Feature Specs Visually

When a feature spec fails, see what the browser is actually rendering — it's much faster than guessing from assertion failure messages.

1. **Human:** Run `pnpm fspec:visible` and watch the browser.
2. **AI agent:** Add `await page.screenshot({ path: '/tmp/debug.png' })` before the failing assertion, run the spec, then read the screenshot file. The agent can see validation errors, missing elements, or unexpected page state.

### Native Date Inputs

Native `<input type="date">` inputs don't accept programmatic value setting via `toFill`, `$eval` setter tricks, or React state manipulation. The browser renders separate mm/dd/yyyy segments that must be typed through individually.

Use `page.keyboard.type('MMDDYYYY')` after clicking the input:

```typescript
const dateInput = await page.$('input[name="arriveOn"]')
await dateInput!.click()
await page.keyboard.type('06012026')  // types 06/01/2026 through segments
```

### Waiting for the Browser

The test process and the browser run concurrently. Before asserting on the database or interacting with the page, you must wait for the browser to be ready. Use `page.waitForNetworkIdle({ idleTime: 500 })` as the general-purpose wait — it covers hydration, API calls, and navigation:

```typescript
// After sign-in or navigation — wait for React hydration before interacting
await hostSignIn(page, user)
await page.waitForNetworkIdle({ idleTime: 500 })

// After a browser action that triggers an API call — wait before asserting on the database
await clickButton(page, 'Add Comment')
await page.waitForNetworkIdle({ idleTime: 500 })
const comment = await post.associationQuery('comments').firstOrFail()
expect(comment.body).toEqual('Great post!')
```

When the UI visibly changes after the API call (e.g., navigation to a new path), waiting on that UI change is a cleaner signal:

```typescript
await clickButton(page, 'Create Place')
await expect(page).toHavePath('/places')        // proves the response completed
const place = await Place.firstOrFail()          // safe to query now
```
