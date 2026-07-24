# Dream Querying and Kysely

This guide exists because app repositories usually depend on `@rvoh/dream` through `node_modules`; they do not expose Dream's own source tree to the agent. Do not tell consuming agents to go inspect package-internal files such as `src/dream/Query.ts` or `src/Dream.ts`.

Instead, treat the public Dream API itself, the TSDocs shipped with the package, and the official Psychic/Dream guides as the sources of truth.

## Rule of Thumb

Always prefer Dream's built in public query and association APIs.

Only drop to Kysely when:

- the needed SQL is not covered by Dream's public API
- or Dream can express most of the query but needs a final low-level Kysely step via `toKysely(...)`

Strongly prefer keeping everything in Dream. Hand-roll raw Kysely (or typed `db()`) only in migrations, or as a true last resort when the requirement genuinely cannot be expressed through Dream's query and association APIs.

Do not jump straight to Kysely for routine filtering, joining, eager loading, aggregation, pagination, or association traversal.

### Eject late, never early: associations carry scopes that `toKysely` does not

`Model.query().toKysely()` does carry the base model's default scopes, including `dream:SoftDelete`. The danger is associations. Association `and` clauses (`HasOne`/`HasMany` `and: {...}`) and the default scopes of associated tables (soft-delete, STI) are applied by Dream's association traversal, not by `toKysely` on the base query. So if you eject early and hand-join an associated table in Kysely — or hand-roll the whole query from typed `db()`, which drops even the base scope — those tables get no soft-delete filter and no `and` clause, and soft-deleted or otherwise excluded rows silently reappear.

The rule: if you must use `toKysely`, build the full Dream query and traverse every association first, then call `toKysely` as the final step for the one thing Dream cannot express.

## Required Agent Workflow

Before claiming that Dream cannot express a query, do all of the following:

1. Inspect the application's models, scopes, associations, and serializers.
2. Check this API inventory.
3. Check the official querying and association docs when available:
   - `psychic-guides/docs/models/querying/*`
   - `psychic-guides/docs/models/querying/loading-and-preloading.mdx`
   - `psychic-guides/docs/models/associations/usage.mdx`
4. Only then decide between more Dream, `toKysely(...)`, or typed `db()`.

## Public API Inventory

The important thing to understand is where the query surface comes from.

### Model-class entrypoints

These start a query or return a query-like builder from a Dream model class:

- `Model.query()`
- `Model.where(...)`
- `Model.whereAny(...)`
- `Model.whereNot(...)`
- `Model.scope(...)`
- `Model.removeDefaultScope(...)`
- `Model.removeAllDefaultScopes()`
- `Model.connection(...)`
- `Model.txn(...)`
- `Model.order(...)`
- `Model.limit(...)`
- `Model.offset(...)`
- `Model.distinct(...)`
- `Model.innerJoin(...)`
- `Model.leftJoin(...)`
- `Model.leftJoinPreload(...)`
- `Model.preload(...)`
- `Model.preloadFor(...)`

These are also query-oriented convenience methods on the model class:

- `Model.all()`
- `Model.find(...)`
- `Model.findBy(...)`
- `Model.findOrFail(...)`
- `Model.findOrFailBy(...)`
- `Model.findEach(...)`
- `Model.first()`
- `Model.firstOrFail()`
- `Model.last()`
- `Model.lastOrFail()`
- `Model.count()`
- `Model.countBy(...)`
- `Model.exists()`
- `Model.max(...)`
- `Model.min(...)`
- `Model.sum(...)`
- `Model.avg(...)`
- `Model.maxBy(...)`
- `Model.minBy(...)`
- `Model.sumBy(...)`
- `Model.avgBy(...)`
- `Model.pluck(...)`
- `Model.pluckEach(...)`
- `Model.paginate(...)`
- `Model.cursorPaginate(...)`
- `Model.scrollPaginate(...)`
- `Model.sql()`
- `Model.toKysely(...)`

These are related record-finding or mutation helpers that often avoid needing ad hoc SQL:

- `Model.create(...)`
- `Model.createOrFindBy(...)`
- `Model.findOrCreateBy(...)`
- `Model.updateOrCreateBy(...)`
- `Model.createOrUpdateBy(...)`

### Query-object methods

Any query object returned by `Model.query()`, `Model.where(...)`, `Model.scope(...)`, `associationQuery(...)`, and similar entrypoints exposes the richer query surface.

Chaining and query-shaping methods include:

- `query.where(...)`
- `query.whereAny(...)`
- `query.whereNot(...)`
- `query.order(...)`
- `query.limit(...)`
- `query.offset(...)`
- `query.distinct(...)`
- `query.innerJoin(...)`
- `query.leftJoin(...)`
- `query.leftJoinPreload(...)`
- `query.preload(...)`
- `query.preloadFor(...)`
- `query.removeDefaultScope(...)`
- `query.removeAllDefaultScopes()`
- `query.connection(...)`
- `query.txn(...)`
- `query.passthrough(...)`
- `query.nestedSelect(...)`

Terminal or result-producing query methods include:

- `query.all()`
- `query.find(...)`
- `query.findBy(...)`
- `query.findOrFail(...)`
- `query.findOrFailBy(...)`
- `query.findEach(...)`
- `query.first()`
- `query.firstOrFail()`
- `query.last()`
- `query.lastOrFail()`
- `query.count()`
- `query.countBy(...)`
- `query.exists()`
- `query.max(...)`
- `query.min(...)`
- `query.sum(...)`
- `query.avg(...)`
- `query.maxBy(...)`
- `query.minBy(...)`
- `query.sumBy(...)`
- `query.avgBy(...)`
- `query.pluck(...)`
- `query.pluckEach(...)`
- `query.paginate(...)`
- `query.cursorPaginate(...)`
- `query.scrollPaginate(...)`
- `query.sql()`
- `query.toKysely(...)`

### Client-controlled page size is capped, not unbounded

`paginate`/`cursorPaginate`/`scrollPaginate` resolve the effective page size as `requestedPageSize || paginationPageSize` (default 25 when the caller passes none), then clamp the result to `paginationMaxPageSize` (default 200). So forwarding a request's `pageSize` straight into one of these methods is already safe against a client asking for the whole table in one page — the framework caps it regardless. Configure either default as a Dream-app option in `conf/dream.ts`:

```typescript
app.set('paginationPageSize', 25)
app.set('paginationMaxPageSize', 200)
```

Lower `paginationMaxPageSize` if an endpoint's per-row cost (large payloads, expensive preloads) warrants a stricter cap than the app-wide default.

Query-level write methods also exist and are easy to miss:

- `query.update(...)`
- `query.delete()`
- `query.destroy(...)`
- `query.reallyDestroy(...)`
- `query.undestroy(...)`

```typescript
// Update matching records — loads each record and calls instance update(), running lifecycle hooks and validations
await LocalizedText.where({ localizable: host, locale: 'en-US' }).update({ title: 'New Title' })

// Update in a single SQL call (skips lifecycle hooks)
await LocalizedText.where({ localizable: host, locale: 'en-US' }).update({ title: 'New Title' }, { skipHooks: true })

// Delete matching records
await Tag.where({ post }).delete()
```

When you hold the associated instance, filter by it — `where({ localizable: host })`, not `where({ localizableId: host.id })`. For a polymorphic association this is a correctness fix, not just style: the id-only form omits `localizable_type`, so it matches rows across every type that shares that id. Passing the instance sets both columns (and resolves an STI child to its base type). See [models.md — Passing associations](models.md#passing-associations-use-the-instance-not-the-foreign-key) for the full rule and its exceptions.

Query-level `.update()` is not Rails `update_all`: by default it iterates the matched
records with `findEach`, calls instance `.update()` on each one, and runs per-record
hooks and validations. Pass `{ skipHooks: true }` only when you intentionally want one
raw bulk SQL update with no hooks or validations. This distinction matters in both
directions: hook-enforced invariants are still enforced by default query updates, and
large "bulk" updates can be N+1 unless you explicitly choose `skipHooks`.

`.update()` resolves to the number of rows it updated. Under `{ skipHooks: true }` the filter
and the write are one `UPDATE ... WHERE` statement, so a filtered update is a compare-and-set
claim: a `0` return means another writer got there first. The default form updates each matched
record separately, so its count says how many rows changed, not that you won a race.

```typescript
// Claim a pending booking exactly once, even under concurrent requests
const claimed = await Booking.where({ id: booking.id, status: 'pending' })
  .update({ status: 'confirmed' }, { skipHooks: true })

if (claimed === 0) return this.conflict()   // someone else already confirmed it
```

### Range Predicates

Dream's `range` helper from `@rvoh/dream/utils` is accepted in `where` clauses for bounded comparisons on supported scalar columns, including `CalendarDate`, `DateTime`, `ClockTime`, and `ClockTimeTz` columns. Use it when a single column has a natural lower and/or upper bound. If named comparison operators make the query easier to read, use `ops` instead.

```typescript
import { ClockTime, ClockTimeTz, ops } from '@rvoh/dream'
import { range } from '@rvoh/dream/utils'

// Inclusive lower and upper bound: startsOn >= start AND startsOn <= end
await Booking.where({ startsOn: range(start, end) }).all()

// Exclusive upper bound: startsAt >= start AND startsAt < end
await Booking.where({ startsAt: range(start, end, true) }).all()

// Open-ended bounds
await Booking.where({ startsOn: range(start) }).all()       // startsOn >= start
await Booking.where({ endsOn: range(null, end) }).all()     // endsOn <= end

// Time-of-day columns work the same way
await BusinessHour
  .where({ opensAt: range(ClockTime.fromISO('09:00:00'), ClockTime.fromISO('17:00:00')) })
  .all()
await Reminder.where({ sendAt: range(null, ClockTimeTz.fromISO('12:00:00Z')) }).all()
```

For two-column interval overlap checks, choose the boundary semantics explicitly. `range` constrains one column at a time, so named comparison operators are often clearer because each boundary is visible at the call site:

```typescript
const overlaps = await Booking.where({
  place,
  startsOn: ops.lessThanOrEqualTo(requestedEndsOn),
  endsOn: ops.greaterThan(requestedStartsOn),
}).exists()
```

Use `ops.lessThan` / `ops.lessThanOrEqualTo` and `ops.greaterThan` / `ops.greaterThanOrEqualTo` based on whether touching interval boundaries count as overlap in the domain.

Query-side overlap validation is not race-safe by itself. Prevent double booking with a database constraint, such as a PostgreSQL exclusion constraint, when concurrent writes can violate the invariant.

### Array column containment

Dream's `where()` array-value dispatch (`where({ status: [...] })` → `IN`) is scoped to non-array columns. For a Postgres array-typed column like `Room.tags: text[]`, use `ops.any(value)` to check whether the column contains a given element: `where({ tags: ops.any('quiet') })`. `ops.any` throws `AnyRequiresArrayColumn` if the target column isn't an array type.

### Ordering

```typescript
// Single column, ascending (default)
Place.order('name')

// Single column, explicit direction
Place.order({ name: 'desc' })

// Multiple columns
Place.order({ position: 'asc', createdAt: 'desc' })
```

### Association and instance entrypoints

These are the methods agents most often forget when they jump too quickly to Kysely.

Querying through associations:

- `instance.associationQuery('associationName')`

Association loading and serializer-aware loading:

- `instance.load(...)`
- `instance.loadFor(...)`
- `instance.leftJoinLoad(...)`
- `instance.association(...)`
- `instance.associationOrFail(...)`

Association mutation helpers:

- `instance.createAssociation(...)`
- `instance.updateAssociation(...)`
- `instance.destroyAssociation(...)`
- `instance.reallyDestroyAssociation(...)`
- `instance.undestroyAssociation(...)`

These methods are often the correct answer for "query the children of this parent", "load nested serializer data", "update all related rows", or "delete associated rows", without writing custom SQL.

### Loading Associations on Instances

Use `await model.association('name')` to access an association. This loads the association only if it hasn't already been preloaded — it's the idiomatic, efficient approach.

```typescript
// PREFERRED — loads only if not preloaded
const hosts = await place.association('hosts')

// ALSO CORRECT — but always loads (returns a clone, not the original)
const loadedPlace = await place.load('hosts').execute()
const hosts = loadedPlace.hosts

// WRONG — load().execute() returns a clone; the original is unchanged
await place.load('hosts').execute()
place.hosts  // ERROR: association not loaded on this instance
```

Key: `load().execute()` does NOT modify the original instance. It returns a new clone with the association loaded. If you forget to use the return value, accessing the association on the original instance throws.

## Association Chaining

**This is the most important concept to understand about Dream's query API.** It differs fundamentally from Rails' ActiveRecord.

In Dream, the string arguments passed to `preload`, `leftJoinPreload`, `innerJoin`, `leftJoin`, `load`, and `leftJoinLoad` form an **association chain** — a traversal path through related models. Multiple string arguments are **NOT** parallel associations; they describe a single path from one association to the next.

### Single association

```typescript
// Preload a direct association
const users = await User.preload('posts').all()
users[0].posts // [Post{}, Post{}, ...]
```

### Association chain (nested traversal)

```typescript
// Chain: User → posts → comments
// Loads posts on each user, AND comments on each post
const users = await User.preload('posts', 'comments').all()
users[0].posts[0].comments // [Comment{}, Comment{}, ...]
```

This is NOT loading both `posts` and `comments` as separate associations on User. It traverses User → posts → comments.

### Parallel associations (separate method calls)

To load multiple unrelated associations, use separate calls:

```typescript
// Load BOTH posts and pets on User (parallel, not chained)
const users = await User.preload('posts').preload('pets').all()
users[0].posts // [Post{}, ...]
users[0].pets  // [Pet{}, ...]
```

### Branching with arrays

The last argument can be an array to branch the chain into multiple associations at the same level:

```typescript
// Chain through posts, then branch to load comments, ratings, AND images on each post
const users = await User.preload('posts', ['comments', 'ratings', 'images']).all()
users[0].posts[0].comments // loaded
users[0].posts[0].ratings  // loaded
users[0].posts[0].images   // loaded
```

### Complex nested loading

Combine chaining and parallel calls for complex loading:

```typescript
// Two separate chains through posts:
// 1. posts → comments and ratings
// 2. posts → images → credits and captions
const users = await User
  .preload('posts', ['comments', 'ratings'])
  .preload('posts', 'images', ['credits', 'captions'])
  .all()
```

### Conditions on associations in the chain

All chainable methods (`preload`, `leftJoinPreload`, `innerJoin`, `leftJoin`, `load`, `leftJoinLoad`) support `and`, `andNot`, and `andAny` condition objects placed after an association name in the chain. The condition object applies to the association immediately before it:

```typescript
// innerJoin with conditions on each association
const hosts = await Host.innerJoin(
  'places', { and: { style: 'cabin' } },
  'rooms', { and: { type: 'Bedroom' } }
).all()

// Condition on last association only
const hosts = await Host.innerJoin('places', 'rooms', {
  and: { type: 'Bedroom' },
}).all()

// preload with conditions
const users = await User.preload('posts', { and: { published: true } }, 'comments').all()

// load with shorthand conditions (plain object = and)
user = await user.load('posts', { body: null }).load('pets').execute()

// andAny follows the same semantics as whereAny: each object is OR'd,
// multiple keys within one object are AND'd
const users = await User.preload('posts', {
  andAny: [
    { title: ops.like('Dream%') },
    { title: ops.like('Psychic%'), published: true },
  ],
}).all()

// andNot follows the same semantics as whereNot
const users = await User.preload('posts', {
  andNot: { archived: true },
}).all()
```

#### Constraints on a required `BelongsTo` are forbidden

A condition object on a **non-optional** `BelongsTo` (the default) is a **compile error** in the hydrating loaders — `preload`, `load`, `leftJoinPreload`, and `leftJoinLoad`. A constraint could filter the parent row out and leave the slot null, breaking the non-nullable field the OpenAPI spec promises for a required `BelongsTo`:

```typescript
// Room.place is `@deco.BelongsTo('Place')` — required (optional defaults to false).
// COMPILE ERROR: the constraint could null a non-nullable field.
await room.load('place', { and: { name: 'Cabin 4' } }).execute()

// Allowed — load the required parent unconditionally:
await room.load('place').execute()
```

Still allowed, because these can legitimately be empty/absent (so a constraint can't violate a non-nullable contract):

- a constraint on an **optional** `BelongsTo` (`{ optional: true }`)
- a constraint on a `HasOne` or `HasMany`
- a constraint on `innerJoin` / `leftJoin` — these don't hydrate a value, so there's no nullability contract to break

`optional` on the model's `BelongsTo` declaration is the single source of truth. If a parent can legitimately be absent, declare it `optional: true` — the OpenAPI field becomes nullable and the constraint is allowed again. The runtime counterpart is `MissingRequiredBelongsToAssociation`; see [models.md — required BelongsTo contract](models.md).

### Where clauses on joined/preloaded associations

After joining or preloading an association, you can reference its columns in `where` clauses using `'associationName.column'` syntax:

```typescript
const posts = await user
  .associationQuery('posts')
  .preload('comments')
  .whereAny([{ body: null }, { 'comments.body': null }])
  .all()
```

### `whereAny` semantics

`whereAny` takes an array of condition objects. Each object in the array is OR'd together; multiple keys within the same object are AND'd together:

```typescript
// Each object in the array is OR'd; multiple keys within one object are AND'd:
// (givenName LIKE 'Anna%') OR (givenName LIKE 'Anna%' AND familyName LIKE 'Maria%')
query.whereAny([
  { givenName: ops.like('Anna%') },
  { givenName: ops.like('Anna%'), familyName: ops.like('Maria%') },
])
```

### Escaping user input in LIKE / ILIKE patterns

`%` and `_` are SQL wildcards inside `LIKE` / `ILIKE` patterns. When the pattern is built from user input, wrap the variable in `escapeLikePattern` so those characters match literally instead of acting as wildcards:

```typescript
import { ops, escapeLikePattern } from '@rvoh/dream'

// User-controlled search term — escape it before interpolating into the pattern
const term = this.castParam('search', 'string')
query = query.where({ name: ops.ilike(`%${escapeLikePattern(term)}%`) })
```

Escape only the user-controlled portion. The wildcard markers (`%` and `_`) you add yourself are not passed through the helper.

### `whereNot` semantics

`whereNot` takes a single condition object. All keys are AND'd and negated:

```typescript
// WHERE NOT (status = 'archived' AND role = 'guest')
query.whereNot({ status: 'archived', role: 'guest' })
```

### Instance-level loading

`load` and `leftJoinLoad` follow the same chaining pattern on instances:

```typescript
// Chain: load posts, then comments on each post
let user = await User.firstOrFail()
user = await user.load('posts', 'comments').execute()
user.posts[0].comments // loaded

// Parallel loads with conditions
user = await user.load('posts', { body: null }).load('pets').execute()
```

### Serializer-driven loading (preferred)

For controller/serializer use cases, `preloadFor` and `loadFor` automatically determine the full chain:

```typescript
// Automatically preloads everything the serializer needs
const users = await User.preloadFor('detail').all()

// Equivalent to manually specifying the full chain
const users = await User
  .preload('posts', 'comments', ['images', 'videos'])
  .preload('posts', ['likes', 'similarPosts'])
  .all()
```

### Incompatibilities

`leftJoinPreload` and `preload` cannot be combined in the same query. Use one or the other.

## Type Cast for Conditional Query Building with Joins

When conditionally building a query that may include `leftJoin` or `innerJoin`, TypeScript infers increasingly specific types for each join. This prevents reassigning the result back to the original `let` variable:

```typescript
let query = Place.preloadFor('summary')

// ERROR: leftJoin returns a Query with a different joinedAssociations tuple,
// which is not assignable back to the original narrow type
if (withoutPhoto)
  query = query.leftJoin('placePhotos').where({ 'placePhotos.id': null })
```

**Fix:** Cast the join expression back to the original query type using `as unknown as typeof query`:

```typescript
let query = Place.preloadFor('summary')

if (search) query = query.where({ name: ops.like(`${search}%`) })
if (withoutPhoto)
  query = query
    .leftJoin('placePhotos')
    .where({ 'placePhotos.id': null }) as unknown as typeof query
if (type?.length) query = query.where({ type })

const results = await query.cursorPaginate({ cursor })
```

The cast is safe because the runtime query object has the same shape — only the type-level join tracking differs. The narrower `typeof query` type is preserved for all subsequent method calls.

For helper functions that accept queries with varying join states, use `Query<Model, any>` for parameter and return types (since there's no `typeof query` to reference):

```typescript
import { Query } from '@rvoh/dream'

// eslint-disable-next-line @typescript-eslint/no-explicit-any
function applyFilters(query: Query<Place, any>, params: FilterParams): Query<Place, any> {
  if (params.withoutPhoto)
    query = query.leftJoin('placePhotos').where({ 'placePhotos.id': null })
  return query
}
```

**What you lose at the cast site:** Joined column name validation — `where({ 'placePhotos.id': null })` won't be type-checked to confirm the join exists. `leftJoin` itself still validates the association name.

**What you keep:** Full type safety on the model type, incompatibility guards on all other calls, and unchanged runtime behavior.

## What To Reach For First

Use this order when reasoning about a query:

1. Start from the natural model or parent instance.
2. Use scopes, `where`, `whereAny`, `whereNot`, joins, association queries, and eager-loading methods.
3. Prefer `preloadFor(...)` or `loadFor(...)` when the query exists to feed serialization.
4. Prefer `associationQuery(...)`, `createAssociation(...)`, `updateAssociation(...)`, and destroy helpers for association-driven work.
5. Use aggregates, `distinct`, `pluck`, `pluckEach`, `nestedSelect`, and pagination before assuming SQL is required.
6. Only then consider `toKysely(...)` or typed `db()`.

## Distinct rows: `.distinct(...)`

`SELECT DISTINCT` is native to Dream — no Kysely eject. `.distinct(column)` distincts on that column; `.distinct()` or `.distinct(true)` distincts on the primary key; `.distinct(false)` clears a distinct applied earlier in the chain.

```typescript
await Host.query().distinct('familyName').pluck('familyName')  // unique family names
```

## Preferred Kysely Escape Hatch: `toKysely(...)`

If Dream gets you most of the way there, keep the query in Dream for as long as possible and convert at the end.

```typescript
const query = Place.query()
  .where({ style: 'cabin' })
  .leftJoin('rooms as r')

const rows = await query
  .toKysely('select')
  .select(['places.id', 'r.type as roomType'])
  .where('r.type', '=', 'Bedroom')
  .execute()
```

This keeps Dream's scopes, association-aware join setup, and types in play until the low-level SQL step.

You can also convert directly from the model class:

```typescript
await Place.toKysely('select').where('name', '=', 'Cozy Cabin').execute()
```

Supported forms:

- `Model.toKysely('select')`
- `Model.toKysely('update')`
- `Model.toKysely('delete')`
- `query.toKysely('select' | 'update' | 'delete')`

### Grouped aggregates (`countBy` / `minBy` / `maxBy` / `sumBy` / `avgBy`)

The scalar aggregates (`.count()`, `.sum`, `.min`, `.max`, `.avg`) each collapse the whole query to one number. When you need the aggregate broken out per group — a booking count for each place on a dashboard, in one query — use the grouped counterparts. They stay entirely in Dream, so no ejection to Kysely and no `GROUP BY` by hand:

- `countBy(groupColumn)` — count per group. Returns `Map<groupValue, number>`.
- `minBy` / `maxBy` / `sumBy` / `avgBy` `(groupColumn, aggregatedColumn)` — the value aggregate per group. Returns `Map<groupValue, aggregate>`.

Each takes a single group column and runs one query, grouping on the foreign key:

```typescript
const placeIds = places.map(place => place.id)

// One query, grouped on the FK. Query the associated model directly so its own
// default scopes still apply — Booking keeps its soft-delete and STI scopes, and
// the `Place` HasMany's `and: { confirmedAt: null }` filter becomes an explicit where.
const pendingByPlaceId = await Booking.where({ placeId: placeIds, confirmedAt: null }).countBy('placeId')

// Only groups with a matching row appear in the Map, so seed absent places to 0.
places.forEach(place => (place.pendingBookingCount = pendingByPlaceId.get(place.id) ?? 0))
```

Two things to know about the returned `Map`: only groups with at least one matching row are present (seed absent keys with `map.get(key) ?? 0`), and a nullable group column produces a real `null` key. The counts arrive already coerced to `number` — no manual `Number(...)` cast. Grouping also works over a joined association column (`Place.query().innerJoin('bookings').countBy('bookings.status')`) and inside an `associationQuery`.

Reach for `toKysely('select')` only when the grouping is beyond this family — multiple group columns, `HAVING`, or a distinct-count. In that case build the full scoped Dream query first, then eject (see [the Kysely escape hatch](#preferred-kysely-escape-hatch-tokysely) above).

### Scope-preserving subqueries: `nestedSelect(...)`

Before reaching for a raw Kysely subquery, check whether `nestedSelect(...)` covers it. It lets a Dream query stand in as a subquery while keeping that query's default scopes, so it is the in-Dream alternative to a hand-rolled Kysely subquery over the raw table.

```typescript
// Places that have at least one booking. The subquery is a Dream query, so
// Booking's default scopes still apply — a soft-deleted Booking won't surface
// its Place. A hand-rolled Kysely subquery over the bookings table has no such
// filter and would resurface Places whose only booking was soft-deleted.
const bookedPlaces = await Place.where({
  id: Booking.query().nestedSelect('placeId'),
}).all()
```

`nestedSelect` projects a **single column** and cannot correlate to the outer query, so a **correlated** subquery (one that references an outer row) or an **aggregate** subquery (`count`/`sum`/… as the projected value) is out of its reach. For those, drop to Kysely while keeping the inner table scoped — see [Correlated and aggregate subqueries](#correlated-and-aggregate-subqueries-keep-the-inner-table-scoped) below.

### Correlated and aggregate subqueries: keep the inner table scoped

When a subquery must reference the outer row (**correlated**) or project an aggregate (`count`/`sum`/…) — the two shapes `nestedSelect` can't express — build it in Kysely, but source the inner table from a **scoped Dream query** (`AssocModel.query().toKysely('select')`), never a raw `db().selectFrom('table')`. This is the ["Eject late"](#eject-late-never-early-associations-carry-scopes-that-tokysely-does-not) discipline applied to the subquery: the association's own default scopes stay in the compiled SQL instead of being hand-rolled as a `deleted_at is null` that silently rots the day the model gains another default scope. Correlate the standalone subquery to the outer row with `sql.ref('outer_table.col')` (from `kysely`) as the `whereRef` right-hand side — a standalone builder's table context only includes its own FROM tables, so a bare `.whereRef('bookings.placeId', '=', 'places.id')` string won't type-check against the outer table.

```typescript
import { sql } from 'kysely'

// Places with more than five non-soft-deleted bookings. A correlated aggregate
// can't be a nestedSelect, so drop to Kysely — but source the inner count from
// Booking.query() so Booking's soft-delete scope is applied by Dream, not
// hand-written as `deleted_at is null`.
const rows = await Place.query()
  .toKysely('select')                          // carries Place's own default scopes
  .where(eb =>
    eb(
      Booking.query()                          // scoped Dream query, not db().selectFrom('bookings')
        .toKysely('select')
        .clearSelect()
        .whereRef('bookings.placeId', '=', sql.ref('places.id'))
        .select(eb2 => eb2.fn.countAll<number>().as('count')),
      '>',
      5,
    ),
  )
  .execute()
```

## Starting From Scratch With Typed `db()`

Every Psychic project provides a typed Kysely entrypoint in `src/db/index.ts` as a default export named `db`.

Use `db()` only when the query is genuinely SQL-first and does not naturally begin from a Dream model or association query.

Typical reasons include SQL that depends on database-specific functions, CTE-heavy query construction, unions, window functions, hand-written SQL expressions, or other cases where the query is not naturally model- or association-driven.

If you include an example in generated or reviewed app code, make sure it demonstrates a truly SQL-first need rather than something Dream already covers with `where`, `innerJoin`, `leftJoin`, `order`, `pluck`, `pluckEach`, aggregates, pagination, preloading, or association helpers.

Guidelines:

- Use the existing typed `db()` export provided by the app.
- Do not invent a separate DB wrapper.
- If a Dream model is the natural anchor, prefer `Model.query().toKysely(...)`.
- If the end result should be Dream models, let Dream perform model loading and association hydration when practical.
- Do not use `db()` for cases already covered cleanly by Dream APIs such as `pluck`, `pluckEach`, `where`, joins, preloading, aggregates, or association queries.

## Mixed SQL and Dream Loading

When custom SQL is needed to rank, score, or filter records but the final result should still be Dream models, use a split approach:

1. Use Kysely to compute IDs and derived values.
2. Load the models with Dream.
3. Reapply SQL ordering in memory and attach derived non-column values.

## Skill Guidance

When writing or reviewing code in a Dream/Psychic app:

- Never imply that the short examples in `SKILL.md` are the full query API.
- Do not cite package source paths as if the consuming app can inspect them.
- If the task touches advanced querying, explicitly consult `querying.md` before recommending Kysely.
- If there is any doubt about whether Dream already supports a pattern, assume it probably does and check the official querying and association docs first.
