# Dream Querying and Kysely

This guide exists because app repositories usually depend on `@rvoh/dream` through `node_modules`; they do not expose Dream's own source tree to the agent. Do not tell consuming agents to go inspect package-internal files such as `src/dream/Query.ts` or `src/Dream.ts`.

Instead, treat the public Dream API itself, the TSDocs shipped with the package, and the official Psychic/Dream guides as the sources of truth.

## Rule of Thumb

Always prefer Dream's built in public query and association APIs.

Only drop to Kysely when:

- the needed SQL is not covered by Dream's public API
- or Dream can express most of the query but needs a final low-level Kysely step via `toKysely(...)`

Do not jump straight to Kysely for routine filtering, joining, eager loading, aggregation, pagination, or association traversal.

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
- `Model.exists()`
- `Model.max(...)`
- `Model.min(...)`
- `Model.sum(...)`
- `Model.avg(...)`
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
- `query.exists()`
- `query.max(...)`
- `query.min(...)`
- `query.sum(...)`
- `query.avg(...)`
- `query.pluck(...)`
- `query.pluckEach(...)`
- `query.paginate(...)`
- `query.cursorPaginate(...)`
- `query.scrollPaginate(...)`
- `query.sql()`
- `query.toKysely(...)`

Query-level write methods also exist and are easy to miss:

- `query.update(...)`
- `query.delete()`
- `query.destroy(...)`
- `query.reallyDestroy(...)`
- `query.undestroy(...)`

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
```

### Where clauses on joined/preloaded associations

After joining or preloading an association, you can reference its columns in `where` clauses using `'associationName.column'` syntax:

```typescript
const posts = await user
  .associationQuery('posts')
  .preload('comments')
  .whereAny([{ body: null }, { 'comments.body': null }])
  .all()
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

## What To Reach For First

Use this order when reasoning about a query:

1. Start from the natural model or parent instance.
2. Use scopes, `where`, `whereAny`, `whereNot`, joins, association queries, and eager-loading methods.
3. Prefer `preloadFor(...)` or `loadFor(...)` when the query exists to feed serialization.
4. Prefer `associationQuery(...)`, `createAssociation(...)`, `updateAssociation(...)`, and destroy helpers for association-driven work.
5. Use aggregates, `pluck`, `pluckEach`, `nestedSelect`, and pagination before assuming SQL is required.
6. Only then consider `toKysely(...)` or typed `db()`.

## Preferred Kysely Escape Hatch: `toKysely(...)`

If Dream gets you most of the way there, keep the query in Dream for as long as possible and convert at the end.

```typescript
const query = User.query()
  .where({ status: 'active' })
  .leftJoin('pets as p')

const rows = await query
  .toKysely('select')
  .select(['users.id', 'p.name as petName'])
  .where('p.name', 'ilike', '%fido%')
  .execute()
```

This keeps Dream's scopes, association-aware join setup, and types in play until the low-level SQL step.

You can also convert directly from the model class:

```typescript
await User.toKysely('select').where('email', '=', 'how@yadoin').execute()
```

Supported forms:

- `Model.toKysely('select')`
- `Model.toKysely('update')`
- `Model.toKysely('delete')`
- `query.toKysely('select' | 'update' | 'delete')`

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
