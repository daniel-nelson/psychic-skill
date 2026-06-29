# Testing Dream & Psychic Applications - Detailed Reference

## Overview

Tests use **Vitest** with real database records (not mocks). We practice **BDD, not TDD** - focus expectations on outcomes, not implementation. For code added independently of a generator, always write a failing spec first, then implement. Generated code is the only exception (generators create scaffolding for specs and implementation simultaneously).

### Every bug is a missing spec

This is the operational corollary to [SKILL.md Rule #9](SKILL.md) (BDD approach) for bugs discovered after the fact. When you discover a bug — in QA, in production logs, during an audit, by hand-testing a flow — the bug itself is evidence that an automated test was missing. **Write the regression spec before committing the fix.**

This applies even when:

- The bug is "obvious in hindsight" once you see the offending code.
- The fix is a one-line change that "couldn't possibly regress."
- A nearby spec already exercises the same component (it clearly didn't catch this case, or this wouldn't be a bug).
- You're in the middle of an audit and finding several bugs at once — each one is its own missing spec; don't let them blur together.

The discipline is: spec **before** committing the fix, not "later." Once the fix is committed, the urgency to write the spec evaporates and the gap stays open. Writing the spec also forces you to articulate the contract ("given X, the user sees Y"), which routinely surfaces a second bug while writing the spec.

**Practical patterns:**

- **Front-end bugs that round-trip data through the API:** the regression spec should assert both the UI state *and* the DB state — mirror the feature-spec pattern in the project's `CLAUDE.md`.
- **DRY the helper, then test the helper.** When a fix consolidates duplicated logic into a shared module (e.g., a `datetime.ts` helper for timezone round-trips), prefer a fast unit spec for the helper over re-testing every call site through feature specs. The helper-level coverage is tighter and faster.
- **When a render-timing flake blocks the integration spec**, write the unit spec for the helper anyway — it locks in the fix at the layer that produced the bug. Document the flake as a follow-up; don't let the flake block the regression-spec discipline.
- **Don't paper over a flaky existing spec by skipping it.** If an `it.skip` is the only thing standing between you and committing, the spec is now an unfinished item — record it explicitly as a follow-up rather than letting `it.skip` rot in the file.

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

**Never run `uspec` and `fspec` at the same time.** Both suites share the same test database lifecycle. Run them sequentially. It is fine to pass multiple file patterns to a single `pnpm uspec` or `pnpm fspec` invocation — the runner manages that suite's lifecycle internally.

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

**Reused-enum placeholder.** When a model column reuses an existing enum (shorthand `name:enum:enum_type_name`, no inline values), the generator can't see the enum's values and emits a TS-rejecting `'TODO'` placeholder paired with a comment hint:

```typescript
// TODO: replace with a value from the `messaging_channels` enum
resolvedChannel: 'TODO',

// Array form
// TODO: replace with a value from the `post_tags` enum
tags: ['TODO'],
```

`'TODO'` is not in the enum's literal union, so `pnpm build:test-app` fails fast at the factory until the placeholder is replaced — preferable to a runtime NOT NULL / enum-mismatch surprise the first time the factory runs. The declare-with-values shorthand (`name:enum:type:val1,val2`) keeps emitting the first listed value as before.

### Factories with Associations

```typescript
// spec/factories/HostPlaceFactory.ts
import HostPlace from '@models/HostPlace.js'
import createHost from './HostFactory.js'
import createPlace from './PlaceFactory.js'
import { UpdateableProperties } from '@rvoh/dream/types'

export default async function createHostPlace(
  attrs: UpdateableProperties<HostPlace> = {}
) {
  return await HostPlace.create({
    host: attrs.host ? null : await createHost(),
    place: attrs.place ? null : await createPlace(),
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
  attrs: UpdateableProperties<Bedroom> = {}
) {
  return await Bedroom.create({
    place: attrs.place ? null : await createPlace(),
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
      it('returns 400', async () => {
        await request.post('/v1/host/places', 400, {
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

### Request and response types come from the `tests` spec

The `request` returned by `session(...)` is an `OpenapiSpecRequest` parameterized by the `tests` spec's generated `paths` types. Because of that, the URI string literal and HTTP method you pass select the endpoint from the spec, and the call is typed on every side:

- path params (keys matching the `{id}` placeholders), the request body (`RequestBody<'post', '/v1/host/places'>`), and query params (`RequestQuery<...>`) are typed to what the endpoint accepts;
- the response `body` is typed to what the endpoint returns for the status you assert.

`RequestBody` / `RequestQuery` are exposed from the generated `spec/unit/helpers/authentication.ts`. Pass the URI as a literal with `{placeholder}` params, not an interpolated string, so it can index the spec.

This makes the spec and the API contract impossible to drift apart silently: change an endpoint's params or response shape, run `pnpm psy sync`, and any spec that no longer matches stops compiling under `pnpm build:spec` until it is updated. For example, an out-of-enum value or an unknown param fails the type-check:

```typescript
await request.post('/v1/host/places', 201, {
  data: { name: 'Cozy Cabin', style: 'not_a_real_style', sleeps: 4 },
  //                           ^ TS error: not assignable to the style enum union from the spec
})
```

The types resolve against the `tests` spec because every surface includes `'tests'` in its `openapiNames`; see [openapi.md](openapi.md#the-tests-spec). Any `openapiNames` override you write must keep `'tests'` in the list — an endpoint left out of the tests spec has no generated types, so its controller spec can't type-check (see [controllers.md](controllers.md#opting-controllers-into-an-openapi-namespace)).

### Generate resourceful controllers first

`pnpm psy g:resource` generates the controller spec already wired up this way — the typed `session` request, `RequestBody<...>` bodies, and per-action blocks with status-code generics. A bare controller generator emits only a placeholder spec (`it.todo(...)`) with none of the typing.

So build resourceful controllers before non-resourceful ones. It seeds the codebase with the correct, fully-typed pattern, which both people and agents then mimic when hand-writing the specs a bare controller leaves as a stub. See [generators.md](generators.md).

### Query parameters in spec requests

**Query parameters nest under `query: {...}`; path parameters are direct keys.** The third argument to `request.get/post/patch/delete` carries everything the request needs beyond URL and expected status: path params as top-level keys (matching the `{name}` placeholders in the URI), and query-string params under a `query` key. Passing a flat `{ search: 'Alice' }` will be silently treated as a path-param attempt and won't reach the controller as a query.

```typescript
// Path param only
await request.get('/v1/host/places/{id}', 200, { id: place.id })

// Query params only
await request.get('/v1/host/places', 200, { query: { search: 'Cabin', minSleeps: 4 } })

// Both — path params at top level, query params under `query`
await request.get('/v1/host/places/{placeId}/rooms', 200, {
  placeId: place.id,
  query: { type: 'Bedroom' },
})
```

The shape is enforced by the OpenAPI-derived types: query-param keys come from the action's `@OpenAPI({ query: { ... } })` declaration, so a typo on either side surfaces as a TS error.

### Transaction-callback type for helper methods

For helper methods called inside a `.transaction(...)` callback, type the txn parameter as `DreamTransaction<Dream>` (imported from `@rvoh/dream`). The same type is used by `@AfterCreate` / `@AfterUpdate` hooks (see [models.md — After Hooks](models.md)) and in [models.md — Transactions](models.md#transactions). Don't reach for `Parameters<Parameters<typeof ApplicationModel.transaction>[0]>[0]` gymnastics.

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

### Negative specs: the principal still needs its auth role row

A negative controller spec that expects a `403`/`404` from an **ownership** check (e.g. "host A cannot update host B's place") must still give the current principal whatever role row the auth layer requires — a `Host` record, a membership row, etc. If the principal lacks that role, the auth `BeforeAction` returns `403` *before* the controller's ownership lookup ever runs. The spec goes green, but for the wrong reason: it's asserting "no host role → 403", not the "wrong owner → 403" it claims to exercise. The ownership branch is never executed and silently loses coverage.

Set up the negative spec so the principal is fully authenticated and authorized *as far as the auth layer is concerned*, and only the ownership relationship is wrong. A quick check: the same request with the *correct* owner should return success in a sibling positive spec using identical role setup — if it doesn't, your negative spec is tripping the auth gate, not the ownership gate.

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

### What the text matchers actually read

`toMatchTextContent` / `toNotMatchTextContent` assert on what the page **renders**, not its DOM source. They collect each element's `innerText` (joined with spaces, with `input`/`textarea` values included), so they see CSS `text-transform`, visibility, and other rendered output. Two consequences:

**Default to a case-insensitive regex for text content.** A label whose source is `5 out of 5` but is rendered with an `uppercase` class matches as `5 OUT OF 5`. The literal string fails, and hardcoding `'5 OUT OF 5'` couples the spec to cosmetic styling. Match on meaning instead:

```typescript
await expect(page).toMatchTextContent(/5 out of 5/i)   // robust
// not '5 out of 5' (fails on the transform), not '5 OUT OF 5' (pins the spec to CSS)
```

Case carries no meaning in rendered copy — the same reason search is case-insensitive — so this is the outcomes-not-implementation stance this file opens with, applied to text. The exception is when the displayed string is an identifier whose case is part of its value: a coupon code `SAVE20`, an invite token, a case-sensitive ID. There, assert the exact case, because `/save20/i` would pass on the wrong value.

Keep this separate from **specificity**. Case-insensitivity does not make a match looser on its own; an over-broad pattern does — `/error/i` also matches `no errors found`. Choose a pattern specific enough to avoid accidental substring hits, independent of case.

**Split label/value markup matches contiguously.** Because per-element text is joined with spaces, `<dt>Sleeps</dt><dd>4</dd>` lands in the matched string as `Sleeps 4`. So `toMatchTextContent('Sleeps 4')` matches a label/value split with no `page.$eval` workaround.

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
      it('returns 400', async () => { ... })
    })
  })
})
```

OpenAPI request validation may reject invalid model params before model validations run. In controller specs that use OpenAPI spec helpers, out-of-range or missing fields derived from model validators expect 400 — the same status the framework returns for param, request-body, and model-validation failures alike.

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

### A job that throws fails the enqueuing request in tests, but not in prod

Under the default `automatic` invocation, a backgrounded method runs inline and **awaited** inside the call that enqueued it — the framework short-circuits the queue and calls the method directly, with no surrounding try/catch. So if the job throws, the error propagates back through `.background(...)` to the caller. A controller action that backgrounds a job and awaits it therefore returns **500 in tests** when the job throws.

In production the same job runs on a separate BullMQ worker. A throw there moves the job to `failed` (and retries per its config) without touching the HTTP response that was already returned. The "fire-and-forget" mental model — `await this.background(...)` returns once queued, the job's success or failure is independent of the request — holds in prod but **not** in tests.

Two practical consequences:

- A spec that drives a backgrounded job must stub or record any external I/O that job performs (HTTP, third-party clients). Because the job runs synchronously inside the request, an unstubbed call hits the network for real and, if it fails, 500s the spec. This is a frequent source of "works in isolation, flaky in the suite" feature/controller specs.
- The inline error propagation is useful — it surfaces job bugs the prod fire-and-forget path would bury — but it means a spec asserting on the request's status code is exercising a different path than production. Assert on the job's side effects, not on a status code that only differs because the job ran inline.

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

### The API server runs in-process — stub the backend boundary directly

Feature specs start the `PsychicServer` inside the same Vitest worker as the test (the generated `spec/features/setup/hooks.ts` calls `server.start(...)` in `beforeAll`, then launches the browser). The browser talks to that server over `localhost:<port>`, but the server runs the *same module instances* the spec can see. So `vi.spyOn(SomeService, 'method')` and `vi.mock(...)` on backend modules intercept server-side code that runs while the browser drives the front end — exactly as in a unit or controller spec. There's no separate process and no IPC barrier.

This matters because a common (wrong) assumption is "a feature spec runs a real server I can't reach into, so I must use a live API key or record HTTP." Not so. To make a feature spec deterministic and offline, stub the backend boundary — an external API gateway, the clock, a third-party client — with `vi.spyOn` in the feature spec, the same way you would anywhere else. Reserve HTTP recording (Polly) for cases where you genuinely want to exercise the real client code path.

### Running a real external service: wrap the command, don't touch the harness

Occasionally a feature spec needs a real external dependency running rather than a stub — most often an emulator the front end and API both talk to (e.g. a Firebase Auth emulator). The harness assumes any such dependency is already listening: `globalSetup` launches the front-end dev servers (`PsychicDevtools.launchDevServer`), then `hooks.ts` `beforeAll` starts the in-process `PsychicServer` and the browser. None of that starts external services, and all of it may depend on them being up.

Supply the service from outside the harness by wrapping the existing spec command with the service's own runner, rather than editing `globalSetup` / `hooks.ts`. Prefer an `exec`-style launcher — one that starts the service, runs the wrapped command to completion, then tears it down — over a long-running "start" command, so each run owns a fresh instance and nothing leaks between runs:

```jsonc
// package.json — wrap the whole command; the harness is untouched
"fspec": "<service> exec '<existing fspec command>'"
```

Leave `uspec` unwrapped: unit specs don't drive the browser or the dev servers, so they shouldn't pay the startup cost or depend on the service's port.

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

`toHavePath` compares the pathname only — internally `new URL(href).pathname` — so it ignores the query string. That makes it a completion signal only when the path actually changes. If the route stays the same except for query params, the assertion can pass instantly without waiting for the mutation. A spec that starts on `/places?placeId=123`, triggers a delete, and asserts `await expect(page).toHavePath('/places')` passes the moment it runs: the pathname was already `/places` before the mutation, so nothing was waited for and the row may still exist.

When the path doesn't change, wait on the eventual state instead. Poll the database for the actual outcome:

```typescript
await clickButton(page, 'Delete')
await expect
  .poll(async () => await Place.find(place.id))
  .toBeNull()                                   // waits for the delete to land
```

Or wait on a UI signal genuinely tied to the mutation finishing (a removed row, a success banner) — not on a `toHavePath` that was already true.
