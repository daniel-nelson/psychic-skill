# Changelog

## 0.81.0 — 2026-08-19

### Added

- **`workers.md`** — a new "Two Configuration Modes" section: `workersApp.set('background', { ... })` accepts one of two mutually exclusive shapes, and the type system enforces the split. Simple (workstream) mode — selected by omitting `nativeBullMQ` — is what the boilerplate ships and what every other configuration example in the file uses, and services route with `backgroundJobConfig.workstream`. Native BullMQ mode — selected by supplying `nativeBullMQ`, even `nativeBullMQ: {}` — hands Psychic raw BullMQ queue and worker options per queue, and services route with `queue` and `groupId`. `pnpm psy sync` generates the routing union from whichever mode is configured, so the mode decides which key a service can use; `priority` is legal in both.
- **`workers.md`** — a new "Native BullMQ Mode" section covering the mode's shape (`defaultQueueOptions`, `defaultWorkerCount`, `defaultWorkerOptions`, `namedQueueOptions`, `namedQueueWorkers`), how a service routes to a named queue, and three sharp edges: a queue declared in `namedQueueOptions` with no matching key in `namedQueueWorkers` is created and accepts jobs that nothing ever works, silently; a named queue's `defaultJobOptions` replaces the app-wide bag rather than merging with it, so a queue setting `attempts` drops the app-wide `backoff`, `removeOnComplete`, and `removeOnFail`; and native mode writes no `concurrency` at all, where simple mode always writes the workstream's — 10 when the workstream omits it. A "Connections in native mode" subsection adds that a named queue's *worker* connection belongs on its `namedQueueOptions` entry — a `connection` on the `namedQueueWorkers` entry typechecks as a plain `WorkerOptions` key and is ignored — and that a missing queue connection surfaces at `pnpm psy sync` time, since sync connects to background to generate types.
- **`workers.md`** — a new "App-Owned Retry Budgets" subsection: there is no per-service retry budget, so a job whose expected failure is worth retrying but not the app-wide twenty times over six days (an external service billed per attempt, say) owns the budget itself — the `_` implementation method takes an `attempt` argument defaulting to `1`, catches its one expected error type and rethrows everything else, and re-enqueues itself with the count incremented while under the threshold, reporting the exhausted failure once the budget runs out. This is the one narrow catch the "Never Rescue Exceptions" section allows, and that section now points at it; because the expected failure ends in a completed job, BullMQ's own retry never engages for it, while an unexpected failure still propagates and still gets the full app-wide budget.
- **`workers.md`** — a new "Transitional Workstreams" subsection: repointing background work at a different Redis instance strands whatever the old one still holds, so `transitionalWorkstreams` describes the legacy topology alongside the current one — the same `defaultWorkstream` / `namedWorkstreams` shape with its own connections — and Psychic builds queues and workers for both. Workers drain the legacy queues while enqueueing reaches only the top-level workstreams, so the old side can only drain and never be added to; delete the key once those queues are empty.
- **`workers.md`** — "Named Workstreams" gains the two facts its example left implicit: a named workstream can carry its own `queueConnection` and `workerConnection`, and without them it shares the default connection so the isolation is only in queue and worker counts; and a workstream's `name` is also the `group.id` of its workers, which is what moves an assigned service's priority onto `group.priority`. The example annotates `workerCount: 1` as the default and names the concurrency default alongside it.
- **`models.md`** — "Transforming a column on write" states that a `{ skipHooks: true }` query update without `lock` cannot write an `@deco.Encrypted` column: naming the backing column throws unless the value is `null`. To update an encrypted column, do not use `skipHooks`.
- **`querying.md`** — `update(attrs, { lock: true })` is documented beside the compare-and-set claim it belongs with: each batch re-selects its rows under an exclusive row lock inside its own transaction before writing, so a record another writer moved out of the query drops out of the locked read, and the count is the records actually claimed. It is what to reach for when the claim has to run hooks, custom setters, or a write through an `@deco.Encrypted` property.
- **`models.md`**, **`querying.md`** — a default (non-`skipHooks`) query update is described as issuing one `UPDATE` per matched row rather than as an N+1. It batches its reads through `findEach` and then writes per record, so the cost is a write per row and preloading does not address it. Neither file now offers `{ skipHooks: true }` as the remedy for that cost.
- **`querying.md`** — `skipHooks` is stated as a semantic choice rather than a performance knob: it removes the callback lifecycle, so every record reached that way has to be checked against the business logic those hooks carry. Two situations warrant it — intentionally suppressing the lifecycle, or updating many records in one SQL statement having established that skipping hooks is safe for them — and a slow update is not on its own a reason. In the second case `lock: true` must not be passed, since it restores the per-record path.
- **`querying.md`** — the query-level update section names which query-level writers actually bypass the model: only `update(attrs, { skipHooks: true })` and `delete()` do, while `destroy`, `reallyDestroy`, and `undestroy` instantiate each record even under `skipHooks`, so custom setters still run there.

### Changed

- **`workers.md`** — "Never Rescue Exceptions Inside Backgrounded Services" states the alerting relationship directly: throwing is how a job reports that it did not finish, not how the app tells a human. A justified catch that handles its expected error and returns normally completes the job and never reaches BullMQ's `failed` set, so it reports the event to the app's error-reporting service right there in the handling code — rethrowing to put it somewhere a human looks buys twenty backoff retries of something that already will not succeed. Retrying is the queue's job, alerting is the app's.
- **`workers.md`** — the Priority Levels section states that priority ordering and workstreams are an either/or without a BullMQ Pro license: a `backgroundJobConfig` carrying a `workstream` (or `groupId`) writes the priority number to the job's `group.priority` instead of the top-level `priority`, and open-source BullMQ ignores `group.priority`. A service gets workstream isolation or priority ordering, not both, unless the app runs the `QueuePro`/`WorkerPro` providers — and the backgrounded-service example no longer pairs a `workstream` with a `priority`.
- **`workers.md`** — the "splitting the class is the only mechanism" claim is scoped to the backgrounded-service API, and the try/catch cross-reference names Critical Rule 14 (linked to the Critical Rules section) rather than rule 13.
- **`workers.md`** — the Worker Configuration block is introduced as the simple-mode shape a typical app runs rather than as the full worker configuration, and closes by placing `defaultBullMQWorkerOptions` — the worker-side counterpart to `defaultBullMQQueueOptions` — in that block, noting that in simple mode a `concurrency` or `connection` inside it is always overwritten by the workstream's own value.
- **`models.md`** — the read-then-write race-window claim is scoped to the default query update; the `skipHooks` form's single `UPDATE ... WHERE` statement is a compare-and-set.
- **`querying.md`** — the default query update's return value is described as the number of records it visited and wrote through — one already holding the incoming values still counts — rather than the number of rows changed.
- **`SKILL.md`** — Critical Rule 15 names `pnpm build:spec` as what surfaces the holes across `src/` and `spec/` when a closed enum's value is added or removed.
- **`websockets.md`** — the Configuration example is the shape an app actually runs. The origin allowlist lives in the app: `conf/system/allowedCorsOrigins.ts` parses `CORS_HOSTS`, defaults to `[]` (so an app booted without it rejects every handshake), and throws at boot on malformed JSON, naming the variable and echoing what it received. The websockets initializer closes over that array in an inline `allowRequest`, which is also where a handshake auth-token check goes. `cors.origin` constrains HTTP long-polling only and native WebSocket upgrades bypass CORS, which is why the allowlist is re-enforced in `allowRequest` — it runs before every handshake on every transport. The adapter's Redis connection is set outside test and bounds its retries: `maxRetriesPerRequest: null` belongs on a BullMQ worker connection, and on this one it makes a broadcast or registry lookup (a worker-process emit, say) hang forever when Redis is unreachable.
- **`websockets.md`**, **`workers.md`** — both Plugin Registration sections state that the plugin registers itself. `conf/app.ts` calls `psy.load('initializers', …)`, which auto-loads `conf/initializers/websockets.ts` and `conf/initializers/workers.ts`; each file's default export takes the `PsychicApp` and registers its plugin inside `psy.plugin()`, so `initializePsychicApp` needs no wiring for either. `workers.md`'s Worker Configuration block shows that default-export shape, with the `set('background', …)` body in a separate `initializeWorkers` function.
- **`soft-delete.md`**, **`querying.md`** — `.delete()` is described as what it is: a single `DELETE` statement that bypasses the model, so it writes no `deletedAt`, runs no `dependent: 'destroy'` cascade (database-level FK cascades still fire), and leaves the rows permanently gone even on a `@SoftDelete()` model. `soft-delete.md`'s guidance for destroying a large set without compare-and-set is plain `destroy()` alone.
- **`models.md`** — the `@deco.AfterCreateCommit()` and `@deco.AfterUpdateCommit({ ifChanged: [...] })` examples declare their methods `async`, so the `await` in each body compiles.
- **`testing.md`** — the reused-enum factory placeholder names `pnpm build:spec` as what fails fast on the TS-rejecting `'TODO'` literal.
- **`testing.md`** — a backgrounded job that throws in production is retried per the queue's `defaultJobOptions` (set in `conf/initializers/workers.ts`) and lands in `failed` only once its attempts are exhausted — none of it touching the HTTP response that was already returned.
- **`i18n.md`** — the second argument to `I18nProvider.provide` names the locale file whose shape types the key paths, so pass the most complete one: `i18n()` accepts only dotted keys that exist in it. Matching a locale value to a locale file happens at call time by language prefix (`'en-US'` maps to the `en` file), and anything unmatched falls back to `en`.
- **`openapi.md`** — the `syncTypes` section states what sync writes: `pnpm psy sync` runs the spec through `openapi-typescript` and writes one declaration file per synced spec to `src/types/openapi/<spec-json-basename>.d.ts` — e.g. `src/types/openapi/tests.openapi.d.ts` — each exporting a `paths` type, with the boilerplate setting `syncTypes` only on the `tests` spec. Typing a response body against those types is `testing.md`'s subject and is no longer duplicated here.
- **`websockets.md`**, **`workers.md`** — examples read configuration through `AppEnv` per Critical Rule 13 rather than `process.env`: the websockets Redis host and port, and the separate `ws.ts` process's port (`AppEnv.integer('WS_PORT', { optional: true })`, falling back to 8889 under test and 8888 otherwise). Both worker-count examples default to `os.cpus().length`, so the configuration shown starts workers.
- **`controllers.md`** — the request-body-size section names `@koa/bodyparser` and points at `jsonLimit` / `formLimit` as the caps to override via `psy.set('json', { ... })`, without asserting particular default values.
- **`deploying.md`** — a credential still carrying the deprecated `useSsl: true` resolves to **unverified** TLS (`{ rejectUnauthorized: false }`), not the verified default that new apps get; the migration is to replace it with an explicit `ssl` value from the matrix above it.
- **`serializers.md`** — the "N+1 Prevention" section is retitled "Unloaded associations throw" and names the actual failure: serializers are synchronous and cannot query, so everything rendered has to be loaded before `this.ok(...)`, and rendering an association that was never loaded throws `NonLoadedAssociation`.
- **`sti.md`** — changing an STI record's type is a plain `update({ type: 'Bathroom', bathOrShowerStyle: 'shower' })` that sets every field the new type requires; database-level check constraints still reject invalid data.
- **`README.md`** — the topic-file table and the "instruct your agent" list for non-auto-loading tools cover every file the skill ships, adding `soft-delete.md`, `i18n.md`, `console.md`, `deploying.md`, and `utils.md` to both.
- **`serializers.md`**, **`controllers.md`**, **`openapi.md`**, **`testing.md`** — examples are BearBnB throughout: the flattening and attribute-shadowing walkthrough is a `Booking` flattening a `Guest`, the nullable-FK `update` worked example is a `Place` with an optional `City`, the public-GET `@OpenAPI` example is `Place`, the action-specific serializer walkthrough carries real `Place/Photo` model, serializer, and controller paths and passes the instance to `this.created(...)`, and the reused-enum factory placeholder uses the `place_styles` and `bed_types` enums.

### Removed

- **`models.md`** — the Dirty Tracking section's closing paragraph on reloading before assigning; the section now ends on the `@deco.Encrypted` `changedAttributes()` fact.

## 0.80.0 — 2026-08-18

### Added

- **`sti.md`** — a new STI limitation: STI is exactly one level deep, so `@STI()` always names the base even when the TypeScript `extends` chain is deeper. A `class Bunkroom extends Bedroom` decorated `@STI(Bedroom)` compiles and imports, then fails silently — `Bedroom.all()` matches only rows whose `type` is exactly `'Bedroom'`, and `Bunkroom` never joins `Room`'s child list, so it is invisible to `preloadFor` and to the generated OpenAPI.
- **`sti.md`** — the hand-added base-serializer variant section states that the variant must be registered on every child: a table's serializer keys are the intersection across the models sharing it, so a `forGuests` key reaches `Room`'s `DreamSerializerKey` only once all five children register it. The type error lands on the base's `preloadFor` call while the fix lives on the child that lacks the key.
- **`models.md`** — a new "Hook order around a `dependent: 'destroy'` cascade" subsection: `beforeDestroy` runs before the cascade and `afterDestroy` after it, so work that must read or archive the children belongs in `beforeDestroy`. `undestroy()` restores the deepest descendants first, so a child's `afterUpdate` runs while its parent still carries a non-null `deletedAt` and an association read back through the parent's default scopes finds nothing — read it with `.removeAllDefaultScopes()` or move the work to the parent's own `afterUpdate`.
- **`models.md`** — the Transactions section names what a missing `.txn(txn)` costs on a `@deco.Sortable` model: the unbound write opens a second transaction and waits on a scope lock the enclosing transaction holds, which no deadlock detector can see, so the call hangs until `sortableScopeLockTimeout` expires and throws `SortableScopeLockWaitTimedOut`. That error advises retrying, and a retry cannot help. The `@deco.Sortable` decorator block points at it.
- **`querying.md`** — preload conditions attach to the association rather than to the chain: several preload chains reaching one association share a single condition node, so a later chain repeating a key overwrites it and a later chain carrying no conditions inherits the earlier chain's.

### Changed

- **`SKILL.md`** — ecosystem version baseline: `@rvoh/dream` to 2.28.x (the other four packages unchanged).

## 0.79.0 — 2026-08-13

### Added

- **`models.md`** — the `createOrFindBy` section states the unique-violation fallback: Dream re-finds with the same first argument, so that argument must hold exactly the unique index's attributes and nothing more — an extra attribute narrows the lookup, and if the submitted value differs from the stored row the re-find comes back empty and `CreateOrFindByFailedToCreateAndFind` turns the duplicate case into a 500. Everything else goes in `createWith`, except a field carrying its own unique constraint — any unique violation lands in the same fallback and the first argument can't identify the row that field collided with. Includes a BearBnB `Booking.createOrFindBy` example.
- **`models.md`** — "Passing associations: use the instance, not the foreign key" gains a fourth reason: generated factories guard on the association (`host: attrs.host ?? (await createHost())`) with `...attrs` spread after, so `createHostPlace({ hostId: host.id })` creates a second `Host` and then overwrites `hostId` with the one passed — a stray row and an extra insert per call, silently.
- **`models.md`** — the `@deco.Encrypted` block adds the column-type decision: a column needing both encryption and structured content uses the `:encrypted` type (which generates a `text` backing column), not `:jsonb`. JSONB earns its place by being queryable and ciphertext is opaque, so nothing survives that a JSONB column exists for; the encrypted column carries structured content directly, round-tripping objects and arrays with no model-layer parse/stringify — the generated declaration types the plaintext property from its `text` backing column, so widen it by hand to the structured type — and the real question is whether the data warrants encryption at all. The not-queryable note also names the consequence that no single-statement compare-and-set `UPDATE ... WHERE` can filter on the value being replaced.
- **`models.md`** — the Dirty Tracking section notes that a persisted instance with nothing dirty issues no `UPDATE` on `save()` or `update()` and leaves `updatedAt` unstamped, so `update({})` — or an `update()` assigning values equal to the current ones — is a no-op rather than a touch. Re-assigning the same plaintext to an `@deco.Encrypted()` property is always a real write, since each assignment re-encrypts to fresh ciphertext; before-save hooks and validations still run first.
- **`testing.md`** — the Factory Pattern conventions add the boundary with `Model.new()`: factories are for records that exist, `Model.new()` for a spec whose subject is an instance that deliberately has no row behind it (validation state before a save, an identifier with no account).
- **`testing.md`** — a ninth testing principle: exercise an environment-dependent branch by spying on the accessor the code actually calls (`vi.spyOn(AppEnv, 'isTest', 'get').mockReturnValue(false)`), since `vi.stubEnv` also changes what Dream reads, including its test-database machinery.

### Changed

- **`testing.md`** — "The API server runs in-process" scopes its interception claim to `vi.spyOn`: the setup file's static import of the app config pulls in `conf/routes.ts` and every controller a route file names as an argument, loading them before a spec's mock registry exists, so a `vi.mock` on one is silently ignored — spy on the runtime object instead. A controller only the boot-time loader reaches does get mocked, which is why the same `vi.mock` can be live on one route namespace and inert on another.
- **`testing.md`**, **`sti.md`** — factory examples use the generated default form `attrs.x ?? (await createX())`; the auto-created-`Guest` example collapses its now-redundant `userId` line accordingly.
- **`SKILL.md`** — Critical Rule 10 names a decorator that declares a virtual column (`@deco.Virtual()`, `@deco.Encrypted()`) as its own `pnpm psy sync` trigger, separate from the sync a migration triggers, with the symptom being `create()` / `update()` rejecting the virtual attribute at build time while every runtime spec passes.
- **`SKILL.md`** — Critical Rule 13 closes with a link to [deploying.md — Environment Variables](deploying.md#environment-variables), where the full `AppEnv` treatment lives.
- **`SKILL.md`**, **`controllers.md`** — the four mentions of `paramSafeColumnsOrFallback()` describe the model's param-safe set (its declared `paramSafeColumns`, or the default otherwise) instead of naming a method app code calls; the `params` / `including` literal-array rule keeps its force and names what an app can actually reach.
- **`migrations.md`** — the `dropEnumValue` blocker sentence covers check constraints alongside a column `DEFAULT`: a check constraint comparing against the enum's values fails the retype-to-text step the same way (a type-agnostic check such as `column IS NOT NULL` never blocks), so drop each value-comparing constraint with `DreamMigrationHelpers.dropConstraint(db, constraintName, { table })` before the first `dropEnumValue` and re-add whichever should survive.
- **`SKILL.md`** — ecosystem version baseline: `@rvoh/dream` to 2.27.x (the other four packages unchanged).
- **`models.md`**, **`generators.md`** — the `pnpm psy sync` trigger list in `generators.md` matches Critical Rule 10, and the find-or-create restriction note names unique-constraint violations as what `createOrFindBy` / `createOrUpdateBy` depend on.

## 0.78.0 — 2026-08-11

### Added

- **`models.md`** — the Query Operators section gains a "Negating a value also matches NULL" paragraph: `ops.not.equal`, `ops.not.in`, `whereNot`, and `andNot` against a non-null value match rows whose column is NULL, so a `Room` whose nullable `position` was never set comes back from `Room.where({ position: ops.not.equal(1) })`. Raw SQL's three-valued logic drops that row instead, so a query written from SQL habit expects a smaller result set than Dream returns — and a `.update()` or `.destroy()` on that negated `where` writes to the NULL rows too. The null literal itself is the exception: `whereNot({ position: null })` is `IS NOT NULL`. `querying.md`'s `whereNot` semantics subsection points at it.
- **`models.md`** — "Which writes run the setter" gains a paragraph distinguishing the two `skipHooks` forms: a query-level `Booking.where(…).update(attrs, { skipHooks: true })` never instantiates a model, so no custom setter runs and the attribute names go into the `UPDATE` as given, while `instance.update(attrs, { skipHooks: true })` assigns through `assignAttributes` and still runs setters — `skipHooks` on an instance means hooks, not setters.
- **`SKILL.md`** — Critical Rule 15 gains a second sanctioned shape, `Record<Enum, T>`, for the case where a value is mapped per enum member rather than code run per value: the `Record` over the union requires a key for every member, so a new value fails to compile at the literal. This is the shape whenever the answer is data — a duration, a rate, a threshold — with user-facing labels called out as an i18n concern instead.
- **`migrations.md`** — a one-sentence naming guard under "Generating column-only migrations": a hand-written migration's name must contain neither `-to-` nor `-from-` anywhere, since either marker resolves the text after it into a real table name — a `-to-` wins wherever it appears, even when a `-from-` comes earlier, and `-from-` applies only when there is no `-to-` at all — and once the table name is resolved rather than left as a placeholder, the generation fails when no columns are passed.
- **`soft-delete.md`** — the "Run sync" setup step gains a sentence naming what the sync records: applying `@SoftDelete()` is what regenerates `src/types/dream.ts` with the model's default scopes, which is where `removeDefaultScope('dream:SoftDelete')` gets its type.

### Changed

- **`SKILL.md`** — Critical Rule 15 restated around the guarantee it buys: branching or mapping on a closed-enum value must stop compiling when a value is added or removed, in one of two shapes — an exhaustive `switch` with a `const _never: never` default when code runs per value, or a `Record<Enum, T>` when a value is mapped per member. The trigger is that the value is a closed enum, not a judgment about whether every case is needed today. The `if`/`else if` prohibition is stated against what it's standing in for: reviewers reject a chain that stands in for handling the enum's values — including a two-branch one, whose `else` silently absorbs every value added later — while a single comparison used as a guard, with no `else` arm taking the remaining values, is not what the rule rejects. Examples are BearBnB-native (`BookingStatusesEnum`, `PlaceStylesEnum`), and `sti.md`'s gloss of the rule matches.
- **`SKILL.md`** — Critical Rule 16's ban on hardcoded enum literals is reconciled with Rule 15's `Record` shape: outside migrations, enum literals appear only where the type system pins them to one member — a `case` label, a `Record` key, a where/create/association-condition attribute, or an assignment to a union-typed variable.
- **`SKILL.md`**, **`migrations.md`**, **`generators.md`**, **`console.md`** — generated types are only ever built from the **test** database. `db:migrate` and `db:rollback` sync types only when `NODE_ENV=test` (the default for `psy` commands), and `pnpm psy sync` is a no-op under any other `NODE_ENV`, so migrating a development database does not update `src/types/` and there is no standalone command that will. Picking up a schema change means running the migration under the test default as well. `migrations.md` carries the full statement, `console.md` restates it under the `NODE_ENV` defaults section (where `NODE_ENV=development pnpm psy sync` prints `skipping sync`), `generators.md` links to `migrations.md` at both points where it tells you `db:migrate` already syncs, and `SKILL.md`'s command list qualifies its `db:migrate`, `db:reset`, and `sync` lines.
- **`workers.md`** — local schedule registration is `NODE_ENV=development pnpm psy db:seed`, and, being development-database work, only on the user's explicit ask (per `console.md`). `db:seed` runs the app's own `db/seed.ts` and nothing else, so whether it is safe against development depends on whether that function is idempotent or truncates and re-creates. Under the test default the CLI skips seeding, and setting `DREAM_SEED_DB_IN_TEST=1` as it suggests still registers nothing, because the app seed then returns at `AppEnv.isTest` — which is also why an unprefixed `db:reset` registers no schedule despite running its seed step. `SKILL.md`'s `db:reset` description names that seed step.
- **`migrations.md`** — three `pnpm psy g:migration` examples renamed so they generate: `rename-host-places-table`, `add-treehouse-place-style`, and `replace-legacy-place-style-with-treehouse`.
- **`sti.md`** — the `g:resource` command (both the walkthrough and the end-to-end recipe) drops the redundant `deleted_at:datetime:optional` argument — `g:resource` includes `@SoftDelete()` and the `deleted_at` column on its own — and the post-`db:migrate` `pnpm psy sync` steps are gone, since `db:migrate` already syncs.
- **`generators.md`** — the `g:resource` STI example matches `sti.md`'s canonical form: the `{}` parent-id path placeholder, `LivingRoom` among the room types, `Place:belongs_to`, and `position:integer:optional`.
- **`SKILL.md`** — ecosystem version baseline: `@rvoh/dream` to 2.25.x (the other four packages unchanged).

## 0.77.0 — 2026-08-07

### Changed

- **`serializers.md`** — the "STI-aware preloading" note states the union behavior: wherever the serializer graph reaches an STI base — at the root of a `preloadFor` call, or through an association whose target is one — the preload set is the union across every STI child's serializer, decided before the query knows which types it will return, so every row carries the whole union, while each row is still serialized by its own child's serializer. The note attributes what is rendered to each child's serializer rather than to the child itself.
- **`SKILL.md`** — ecosystem version baseline: `@rvoh/dream` to 2.23.x (the other four packages unchanged).

## 0.76.0 — 2026-08-06

### Added

- **`testing.md`** — new "What the selector matchers actually assert" subsection: `toHaveSelector` is presence-only regardless of any `visible`/`hidden` option passed, and `toNotHaveSelector` requires true DOM absence (a present-but-hidden element fails it). No matcher in the current set asserts visibility; when a spec needs to distinguish hidden-but-mounted from truly absent, assert the hiding mechanism (e.g. a CSS class) directly instead.

### Changed

- **`SKILL.md`** — ecosystem version baseline: `@rvoh/psychic-websockets` to 3.5.x (the other four packages unchanged).
- **`serializers.md`** — the `optional`-on-`rendersOne` note gains a caveat: `optional: true` has no effect on `NonLoadedAssociation` — rendering an association that was never preloaded or loaded still throws regardless of `optional`.

## 0.75.0 — 2026-07-29

### Added

- **`migrations.md`** — new "Raw Kysely Data Queries in Migrations" section: a migration's `db` handle carries the same `CamelCasePlugin` as every other Kysely instance Dream builds, so raw selects/updates are written in camelCase like everywhere else in a Dream app, even though the rest of a migration file (DDL, `DreamMigrationHelpers` calls) is snake_case.
- **`workers.md`** — extends the "NEVER pass model data as background job arguments" rule: a plain scalar argument (e.g. a confirmation-link string) can still leak sensitive, model-sourced data if a secret or token is baked into it before enqueueing. Defer minting the sensitive part to the backgrounded method itself — pass only the id(s) needed to look the record up, not a pre-composed value carrying a credential into queue/dashboard retention or error-monitoring forwarding.
- **`testing.md`** — the "split label/value markup matches contiguously" example gets a caveat: a more complex layout (e.g. a flex container) can break contiguity between what look like adjacent elements, with a whitespace-tolerant regex (`/label\s+value/`) as the fallback when a literal contiguous match fails unexpectedly.
- **`querying.md`** — "Array column containment" is retitled "Array column containment and equality" and corrected: `where()`'s array dispatch is unconditional (it branches on `Array.isArray(value)` before consulting the schema), not scoped to non-array columns — that's why a bare array fails against an array column. Adds `ops.equal(value)` as the exact, order-sensitive array-equality tool (element types constrained by the column; `json[]` unsupported, only `jsonb[]`) alongside `ops.any` for containment, with a comparison table.
- **`soft-delete.md`** — new "Guarded (compare-and-set) destroy" subsection: `destroy({ lock: true })` / `reallyDestroy({ lock: true })` claim rows via per-batch row locking (`FOR UPDATE OF`), returning the count of records actually claimed; `batchSize` defaults to 10 under `lock` (versus 1000 otherwise) since every record in a batch stays locked for its full destroy-hook/cascade duration. The guarantee is per-batch, not set-wide — wrap in `ApplicationModel.transaction(...)` for an all-or-nothing destroy. Framed as the opposite of a bulk-delete accelerator.
- **`models.md`** — the `ops` reference table gains `ops.equal`/`ops.not.equal` rows for array columns, and the `ops.any`-only note beside it is corrected. The Batch Processing section's `findEach` example is fixed to the real `findEach(cb, opts)` argument order, translated to `Booking`/`Guest`, and now states plainly that `findEach` always iterates in ascending primary-key order and discards any `order` applied to the query.
- **`models.md`** — the Dirty Tracking section notes that for an `@deco.Encrypted()` field, `changedAttributes()` reports the persisted `encrypted<Name>` key, not the plaintext virtual property; `getAttribute('<plaintext>')` returns `undefined` rather than decrypting, so read the decrypted value via the instance property.
- **`models.md`** — the Dirty Tracking section adds an escape hatch for forcing a write against the database's current state rather than the loaded instance's in-memory value: a query-level `.update()` (which reloads before writing) or `instance.reload()` first.
- **`workers.md`** — the "runs inline in tests" note now names `backgroundWithDelay(...)` alongside `schedule(...)`/`background(...)`: all three invoke the underlying method immediately and synchronously in tests, ignoring the delay.
- **`workers.md`** — the Debounce with jobId section documents that a dedup key's TTL equals the delay, so it has expired by the time the delayed job fires and it's safe to re-arm the same `jobId` from inside the job's own running handler; a zero-second delay attaches no dedup key at all.
- **`migrations.md`** — new paragraph in the Overview: every `pnpm psy` command boots the full app, importing all models, before any type-regeneration step, so a stale `src/types/db.ts` crashes the CLI — including `pnpm psy db:reset`/`sync` — before it can reach the fix. Recovery is `git checkout HEAD -- src/types/db.ts src/types/dream.ts`, then `db:reset`/`sync` to regenerate from a clean baseline.
- **`soft-delete.md`** — new "Sortable position columns" subsection: soft-deleting unconditionally nulls every `@Sortable` field's position column in the same `UPDATE` as `deletedAt`, so a `NOT NULL` position column throws a not-null violation — including when the sortable model is only a `dependent: 'destroy'` cascade target of a parent being destroyed. Retrofitting `@SoftDelete()` onto a `@Sortable` model requires dropping the position column's `NOT NULL` constraint.
- **`models.md`** — the `@deco.Sortable` block cross-references `soft-delete.md`'s new "Sortable position columns" subsection.
- **`soft-delete.md`** — the Permanent delete (reallyDestroy) section documents that `reallyDestroy()` cascades depth-first through `dependent: 'destroy'` associations while bypassing the `dream:SoftDelete` default scope, hard-deleting already-soft-deleted children too, and throws a foreign-key violation against non-`dependent` `restrict`-FK children, which the cascade never touches.
- **`openapi.md`** — new "Generating a typed API client (Zustand)" section: `--client-config-file` must live outside `--output-dir`, since the underlying generator cleans its output directory on every `pnpm psy sync` and silently deletes a starter file placed inside it — the pattern the command's own `--help` example and interactive-prompt default both use. `--export-name` only affects the initializer function and log labels, not generated SDK/store code. `setup:sync:openapi-redux` is mentioned as a sibling command with its own, different flag set.
- **`testing.md`** — "Running Tests" gains a caution: vitest's positional path filters (including `pnpm fspec <path>`) match by prefix/substring against file paths, not real path-scoping, so a "scoped" run's file count should be checked rather than assumed (e.g. `pnpm fspec spec/features/host` also runs a sibling directory like `spec/features/host-verification/`).
- **`testing.md`** — notes that a spy shared across specs via `beforeEach`/`afterEach` should be typed `MockInstance<typeof Obj.method>` (`import type { MockInstance } from 'vitest'`), not `ReturnType<typeof vi.spyOn>`, which resolves to `any` and passes `pnpm build:spec` but fails `pnpm lint`.
- **`models.md`** — after the "Which method to use" table, notes that without a real unique index on the lookup attribute(s), `createOrFindBy`/`createOrUpdateBy` can silently insert duplicate rows, since both only detect an existing record via a uniqueness violation on the insert.

### Changed

- **`migrations.md`** — the "Renaming an Enum Value (Two-Migration Pattern)" section's intro now states the two-file split applies to any migration that adds an enum value and then uses it (an `INSERT`/`UPDATE` reconciliation, not just a rename/`dropEnumValue`-based replacement), cross-referencing "Forcing a New Transaction in Migrations" for the general escape hatch.
- **`migrations.md`** — corrects the `pnpm psy db:migrate` post-sync-failure description: a failure reverts `db.ts`/`dream.ts` back to their pre-sync content (rather than leaving them "already regenerated"); the migration ledger and database schema change are unaffected by the revert, so there's no need to `db:rollback` on the assumption the ledger is out of sync.
- **`SKILL.md`** — ecosystem version baseline corrected: `@rvoh/dream` to 2.22.x, `@rvoh/psychic-websockets` to 3.4.x, `@rvoh/psychic-spec-helpers` to 3.4.x (`@rvoh/psychic` and `@rvoh/psychic-workers` unchanged).
- **`CLAUDE.md`** — "Delete stale guidance cleanly" now states a change verified against real (even unmerged) upstream source may be documented as current, matter-of-fact, once the maintainer confirms it — narrating only genuinely *past* behavior remains off-limits. "Keep the ecosystem version baseline current" gains a cross-referencing sentence permitting the baseline to lead the actual published `origin/main` version in that case.

## 0.74.0 — 2026-07-24

### Added

- **`querying.md`** — new "Array column containment" subsection: `where()`'s bare-array dispatch (`IN` over elements) is scoped to non-array columns, so use `ops.any(value)` to check whether a Postgres array-typed column contains a given element.

## 0.73.0 — 2026-07-21

### Added

- **`migrations.md`** — new "Dropping a column: declare it ignored one deploy ahead" subsection: a rolling deploy runs the `dropColumn` migration while containers built from the previous image are still serving, and those containers still name the column in the SQL `leftJoinPreload`/`leftJoinLoad` generate, so joined reads fail with `42703` until the last old container drains — a failure a single-schema spec suite cannot catch. Documents the model's `ignoredColumns` getter and the two-deploy process (deploy 1 removes references, declares the column ignored, and syncs; deploy 2 ships the migration), plus the two preconditions: the declaration is consumed by `sync` rather than at runtime, and the column must be nullable or carry a database default before deploy 1 ships.
- **`deploying.md`** — a "Dropping a column takes two deploys" pointer under "Migrations in Production Deploys", cross-referencing the migrations.md section.

### Changed

- **`SKILL.md`** — ecosystem version baseline moves `@rvoh/dream` to 2.20.x.

## 0.72.0 — 2026-07-21

### Added

- **`testing.md`** — an error status an action answers by hand must be added to its `@OpenAPI` `responses` and synced before a spec can assert that status.

## 0.71.0 — 2026-07-21

### Added

- **`models.md`** — the required-`BelongsTo` contract now covers the save side: a non-optional `BelongsTo` registers a `requiredBelongsTo` validation for you, so saving without the parent fails Dream validation before Postgres sees it, keyed by the association name (`{ place: ['requiredBelongsTo'] }`) rather than the foreign key column. Never declare `@deco.Validates('requiredBelongsTo')` yourself; the example of doing so is removed from the Validations section.
- **`models.md`** — new "Every hop applies its own default scopes, including soft delete" subsection: a soft-deleted intermediate makes a `through` chain resolve `null`/`[]` with no error, and because `through` cannot take `withoutDefaultScopes` the only way to traverse one is walking the hops as explicit `removeDefaultScope('dream:SoftDelete')` queries. Adds a one-line note that only `BelongsTo` associations are assignable through `create`/`update` params, and annotates the `IS NOT NULL` form as the one to use on array columns (which take no `ops` comparison but `ops.any`).
- **`querying.md`** — query-level `.update()` resolves to the number of updated rows; under `{ skipHooks: true }` the filter and the write are one `UPDATE ... WHERE` statement, making a filtered update a compare-and-set claim (a `0` return means another writer won the race), while the default form's count says how many rows changed rather than that you won a race.
- **`controllers.md`** — new "409 from a database constraint" section: `pgErrorType` as the narrow, specific catch Critical Rule 14 allows, and the rule that a blocked delete arrives as `'RESTRICT_VIOLATION'` under `.onDelete('restrict')` but `'FOREIGN_KEY_VIOLATION'` under `no action`, so a delete guard must accept both or the endpoint 500s. Adds `this.conflict()` (409) to the Response Methods list.
- **`serializers.md`** — new "Rendering an async-computed shape on the model" subsection: `rendersOne`/`rendersMany` accept any declared property, so an asynchronously computed shape can be assigned in the controller and rendered as a field of the model, with `optional: true` keeping actions that skip it from failing response validation. Prefer the compound envelope; this is for shapes that must sit inside the model's own object, such as models rendered as a collection.
- **`workers.md`** — new "Call `scheduleAllJobs()` yourself, from `db/seed.ts`" subsection: nothing registers schedules for you, so a scheduled service nobody calls silently never runs. `schedule()` upserts, so registration belongs where a deploy already reconciles the database to the code — seed, which runs immediately after migrations. Notes the jobs-Redis reachability requirement and the local `db:seed` step.

## 0.70.0 — 2026-07-17

### Changed

- **`controllers.md`** — corrects the client-error Response Methods block: the 422 helper is `this.unprocessableContent()`, not `this.unprocessableEntity()` (which does not exist on `PsychicController` — matching the RFC 9110 rename of 422 to "Unprocessable Content").

### Added

- **`models.md`** — new "Parsers throw on invalid input — all four classes" subsection in Date/Time. `fromISO`/`fromSQL`/`fromFormat`/`fromObject` on `DateTime`, `CalendarDate`, `ClockTime`, and `ClockTimeTz` all **throw** on unparsable or calendar-impossible input (`'2026-02-31'` throws like `'garbage'`) — there is no "invalid instance" to inspect, so the `if (!DateTime.fromISO(v).toISODate()) …` probe idiom never reaches its guard. Documents the class-specific errors (`InvalidDateTime` / `InvalidCalendarDate` / `InvalidClockTime` / `InvalidClockTimeTz`, all from `@rvoh/dream/errors`) and the parse-and-catch-the-specific-error validation idiom for untrusted date strings.
- **`models.md`** — the Transactions section now states that `instance.txn(txn).update(attrs)` assigns the attributes to that same in-memory instance before saving (like `instance.update()`), so the caller's reference is current after the call — but other loaded instances of the same row are not synchronized and must be `reload()`ed if they must observe the change.
- **`workers.md`** — the backgrounded-service section now states that `backgroundJobConfig` is class-level: both `background()` and `backgroundWithDelay()` read it and take no per-call config argument, so a service's `workstream`/`priority` govern every backgrounded method on the class. To isolate a subset of jobs, extract them into a separate backgrounded service — splitting the class is the only mechanism.
- **`controllers.md`** — the `requestBody` "UPDATE with a nullable FK" example now carves the narrowing case out of the `combining` anti-pattern: when an action's runtime contract is deliberately stricter than a nullable column (a non-null `castParam` without `allowNull`), `params` + `required` still advertises `string | null` because `required` only adds to the schema-level required array without dropping `null` from the property type — so shadowing the column via `combining` + `required` is the correct way to document the narrowing (the anti-pattern is *redundant* shadowing, not *narrowing* shadowing).
- **`migrations.md`** — the two-migration enum-rename section gains two rollback/teardown caveats: (1) a column `DEFAULT` referencing the enum blocks `dropEnumValue`'s internal retype-to-`text` step, so drop the default before and restore it after; (2) a fully-restoring `down()` cannot re-`addEnumValue` and `dropEnumValue` in one file — `dropEnumValue`'s internal `getEnumValues` counts as *use* of a just-added value and throws PostgreSQL `55P04`, so the restoring rollback must be split across the migration pair exactly like the forward pass (and the pair rolls back together).

## 0.69.0 — 2026-07-15

### Changed

- **`testing.md`** — the "Key Test Matchers" section now states that `toMatchDreamModels` compares its two arrays as **sets** (both sides sorted by comparison key before matching), so it passes regardless of the order the query returned rows in — the right assertion for a query whose order you don't control. Adds the corresponding warning: a Dream query that declares no `order` (`.all()`, an unordered `associationQuery(...).all()`, a `pluck`) carries no SQL `ORDER BY`, so the observed row sequence is not a guarantee; don't assert it with an order-sensitive matcher, and make order explicit in the query when order is part of what you're testing.
- **`websockets.md`** — the "Client Transport" section now frames `transports: ['websocket']` as a connection-latency recommendation only, not a correctness requirement: the server handles polling and WebSocket clients correctly either way, and websocket-only transport just avoids the slower initial handshake. Reworks the "stale data until refresh" troubleshooting callout accordingly — it no longer points at the client transport, and instead names cross-process delivery (an emit from a web/worker process reaches a socket held by the websocket-server process only through the Redis adapter; the in-process adapter fans out only within its own process) as the likely cause.
- **`controllers.md`** — corrects the `server:error` section. `server:error` is the surface for every **shapeable** 5xx error (caught while the response can still be formed), but it is not the complete 5xx surface: a residual class — errors thrown after headers were sent, response-stream failures, and the router's deliberate dev/test re-throws — bypasses `server:error` and reaches only Koa's app-level `'error'` event, which Psychic's own listener logs but does not route to `server:error` hooks. Replaces the prior "do not add `koaApp.on('error')`" guidance with: add your own `koaApp.on('error')` listener if your error pipeline (not just your logs) must capture that residual class — it won't double-report the shapeable errors.

### Added

- **`websockets.md`** — new "`ws:error`: the request-aware websocket error hook" section documenting the ws-layer analogue of `server:error` (register with `wsApp.on('ws:error', (error, context) => …)`). Covers the discriminated context (`phase: 'ws:connect'` with `socketId`; `phase: 'ws:health-check'` with `method`/`path` from the ws server's own HTTP handler, which never reaches `server:error`), the privacy scrubbing (context `path` is query-stripped for safe external shipping while the internal log keeps the full URL), observer isolation and non-recursion, and the key gotcha that framework containment wraps only `ws:connect` hooks — per-socket auth done inside a `ws:start` connection handler is neither contained nor observed, so it must live in a `ws:connect` hook. Adds a companion note that Redis pub/sub adapter `'error'` events have no framework hook by design: attach your own listener to the public `wsApp.connection`/`wsApp.subConnection` getters right after `set('connection')`, dedupe before shipping (ioredis floods `'error'` during an outage), and configure the connection once (replacing it after `cable.start` silently breaks cross-process delivery).
- **`controllers.md`** — new integration caveat in the `server:error` section: an error-tracking SDK configured with `defaultIntegrations: false` does not auto-attach HTTP request context, so pull the fields you want off the hook's `ctx` and attach them to the event yourself.

### Changed

- **`SKILL.md`** — bumped the ecosystem version baseline for `@rvoh/psychic-websockets` 3.4.x → 3.5.x (the `ws:error` hook and the health-check request-handler fix ship in 3.5.0).

## 0.68.0 — 2026-07-14

### Changed

- **`migrations.md`** — corrects the `DreamMigrationHelpers.dropEnumValue` two-migration example: a `replacements` entry for a **scalar** enum column is `{ table, column, replaceWith }`, with no `behavior` key (the previous example wrongly put `behavior: 'replace'` on the scalar `places.style` column). Adds a note that `behavior` (`'remove'`/`'replace'`) and `replaceWith` apply only to array columns, where the entry also carries `array: true`, with worked examples of both array shapes.

## 0.67.0 — 2026-07-06

### Added

- **`models.md`** — documents multi-hop and `BelongsTo` `through` associations, with fully-declared examples (every intermediate association shown). Makes several previously-unstated facts explicit: `through` names an association declared on the *same* model (so a multi-hop reach is one through per model, each naming the next model's association); a `through` can point at an association that is itself a `through`, so Dream unwinds the chain recursively over `BelongsTo` intermediates to reach up the ownership tree (a `Booking → Room → Place → Host` example); and `source` defaults to the association's own property name, matched by name rather than target model, which disambiguates an intermediate with two associations of the same target type. Notes that condition/shaping clauses (`and`, `andNot`, `andAny`, `selfAnd`, `selfAndNot`, `order`, `distinct`) compose on `through` associations at any position in a chain — each hop's clauses shape the join of that hop's own target model, and stacked `order`s apply in join order (the source association's own `order` last on a shared join) — along with the two real limits: a hop whose `and` uses `DreamConst.required` cannot be bridged across (throws `MissingRequiredAssociationAndClause`), and a `distinct`-carrying through loads with `preload`/`innerJoin` but not `leftJoinPreload`.
- **`models.md`** — new "Where a hop belongs: compose across models, or stack on the origin" subsection. The two ways to build a deep `through` reach are capability-equivalent (same joins, same `source` resolution on the destination), so the choice is where the intermediate `through`s are declared. Describes composing across models as the default (each hop on its natural model, so an intermediate like `Place.guests` stays reusable) and gives an origin-stacking example — an origin-relative `recentGuests` filter that must live on the origin because it describes the origin's view, not an intrinsic fact about the middle model.
- **`models.md`** — the "Passing associations" section now documents filtering by an **array of instances** under an association key (`where({ place: places })`, also in `whereNot`/`whereAny` and join on-clauses): a non-polymorphic key expands to a foreign-key `IN`, a polymorphic key groups instances by type into id-`IN`-plus-type-match pairs that are OR'd, and the array form is query-only (`create`/`update`/`findOrCreateBy` still take a single instance).
- **`models.md`** — the `@deco.Encrypted` block now notes that an encrypted column is not queryable in a `where` clause: the plaintext property is virtual (not in the model's `Whereable`, so `where({ phone: ... })` is a compile error) and the stored ciphertext is non-deterministic, so there is nothing to match server-side. Points to an in-memory decrypt-and-compare for existence checks and a separate deterministic/blind-index column for true server-side equality lookups.
- **`deploying.md`** — the `AppEnv` section now documents that variable names are a closed typed union in `AppEnv.ts` (`Env<{ boolean; integer; string }>`), so adding an env var means adding its name to the union in the same change. Notes the surfacing gotcha: a missing name compiles fine under Vitest (`pnpm uspec` doesn't type-check) and only fails at the `pnpm build:spec` type-check gate with `TS2345`.
- **`querying.md`** — the `nestedSelect` section now notes the two subquery shapes it cannot express — **correlated** (references an outer row) and **aggregate** (`count`/`sum`/… as the value) — and a new "Correlated and aggregate subqueries: keep the inner table scoped" subsection under the `toKysely` escape hatch shows the fallback: build the subquery in Kysely but source the inner table from a scoped Dream query (`AssocModel.query().toKysely('select')`), not raw `db().selectFrom('table')`, so the association's default scopes survive, correlating with `sql.ref('outer.col')` on the `whereRef` RHS.
- **`workers.md`** — new "Process-level error semantics" subsection: `background.work()` installs the worker's own `uncaughtException`/`unhandledRejection` handlers plus a bounded graceful shutdown that exits nonzero (so don't add your own in the worker entry point); `workers:shutdown` is the only lifecycle hook (wire job-failure observability directly on the BullMQ workers/queues); a job whose class no longer resolves fails loud with `NoClassForSpecifiedGlobalName`; and the jobs Redis is a trust boundary equal to the app process (dispatch runs any registered method from the payload with no allow-list).
- **`websockets.md`** — notes that a throwing `ws:connect` hook is contained to the connecting socket (framework logs and `socket.disconnect(true)`s; the process stays up), so an auth hook may throw to reject a connection without its own outer try/catch — while a redis adapter that can't attach aborts ws startup rather than falling back to in-memory delivery.
- **`controllers.md`** — new "Bounding length and range" subsection for `castParam`/`extractParams`: `minLength`/`maxLength` on strings and `minimum`/`maximum` on `number`/`integer`/`bigint`, inclusive and checked after coercion, throwing `ParamValidationError` (`400`); array casts enforce the bounds element-wise. Frames it as bounding untrusted input at the request boundary.
- **`controllers.md`** — the `server:error` section now states that Psychic mounts an error boundary as its outermost middleware, so `server:error` is the single 5xx surface for the web process: uncaught genuine errors from the body parser, CORS callbacks, custom `psy.use(...)` middleware, after-routes mounts, and controller actions all escalate to this one hook (don't add a second `koaApp.on('error')`). 4xx-shaped errors render as their status and never reach it, and once headers are sent the error is re-thrown to Koa — so the hook only ever sees 5xx-class errors and shouldn't filter by status.

### Changed

- **`querying.md`** — the grouped-aggregate section now teaches the first-class `countBy` / `minBy` / `maxBy` / `sumBy` / `avgBy` family instead of a hand-rolled `toKysely` + `GROUP BY`. These stay entirely in Dream, return a `Map` keyed by the group value (with counts already coerced to `number`), respect the model's default scopes, and group on the foreign key in a single query. `toKysely` is now called out only for grouping the family can't express (multiple group columns, `HAVING`, distinct-count). Adds the ten new methods to the model- and query-method reference lists.
- **`models.md`** — the `DreamConst.required` bullet now states that the required value can only be supplied for the association actually named in the query, so a `DreamConst.required` association cannot serve as an intermediate hop of a `through` chain.
- **`SKILL.md`** — updated the ecosystem version baseline to the current published `latest` versions: `@rvoh/psychic` 3.10.x → 3.11.x, `@rvoh/psychic-workers` 2.3.x → 2.4.x, `@rvoh/psychic-websockets` 3.3.x → 3.4.x. `@rvoh/dream` is 2.19.x (`@rvoh/psychic-spec-helpers` unchanged at 3.2.x).

## 0.66.0 — 2026-07-06

### Added

- **`.codex` → `.agents` consolidation in the updater.** `bin/psychic-skill-update-apply` now runs a one-time migration before reconciling: any legacy `.codex` psychic-skill copy (global `~/.codex` or project `.codex`) is moved into the matching `.agents` location, or dropped if an `.agents` copy already exists there. Dev symlinks are left untouched, and `--plan` reports the migration without mutating anything. Codex reads `.agents`, so this retires the legacy location without losing an install.
- **`bin/psychic-skill-dedupe` — retrofit a dual project install.** A project that committed both a real `.claude/skills/psychic-skill` copy and a real `.agents/skills/psychic-skill` copy carries two trees that drift apart. The new helper detects that case, determines the project's package manager, and converts `.claude` (and its `psychic-update-skill` sub-skill link) into the canonical link shape — a relative POSIX symlink into `../../.agents/skills/psychic-skill` for pnpm/yarn/bun (Windows junction with a recursive-copy fallback), while leaving npm's `.claude` as a real copy (npm disables install scripts and Windows breaks committed symlinks). It is idempotent and safe to re-run, and `--plan` previews without mutating.
- **`/psychic-update-skill` now offers to dedupe a dual install.** A new post-upgrade step detects a dual real `.claude`+`.agents` project install, previews the conversion with `--plan`, asks the user, and runs the dedupe helper. The routine updater does not auto-restructure committed repos — this is an on-demand, opt-in step.

### Changed

- **`.codex` is no longer enumerated as an install root.** Removed `.codex` (and the `CODEX_SKILL_DIR` override) from the root lists in `bin/psychic-skill-update-check`, `bin/psychic-skill-update-apply`, the `SKILL.md` update-check one-liner, and the `psychic-update-skill/SKILL.md` root-detection loops. Existing `.codex` copies are consolidated into `.agents` by the migration above.
- **`README.md`** — documents the canonical shape for a project using both Claude and a `.agents`-compatible agent: one real tree at `.agents`, with the Claude copy referencing it (symlink for pnpm/yarn/bun, real copy for npm), and points at `/psychic-update-skill` to retrofit an existing dual install.

## 0.65.0 — 2026-07-03

### Added

- **`models.md`** — new canonical "Passing associations: use the instance, not the foreign key" section under Creating and Updating. When you hold a model instance, pass it (not the FK id, or id + type for a polymorphic association) to `create`/`update` and to `where`/`and` filters. Documents that the instance form works uniformly regardless of the association's shape — polymorphic or not, plain model or STI child — because Dream derives the foreign key from the primary key and, when polymorphic, the type discriminator from `referenceTypeString` (resolving an STI child to its base). Gives the three reasons it's a rule and not a preference: polymorphic id-only filters omit the type column and match across types that share an id (a correctness bug on integer/serial keys); STI type strings resolve to the base automatically while a hand-written child type matches nothing; and holding the instance keeps the authorization load in the code path. Notes the controller case is effectively absolute, and carves out the three legitimate id-form cases (bulk id arrays, cross-type polymorphic batch loads, and background-job arguments).

### Changed

- **`querying.md`** — added a query-side note next to the `where`-update examples pointing at the new rule, framing polymorphic instance-filtering as a correctness fix (the id-only form omits `localizable_type`). Fixed the examples to filter by the instance (`where({ localizable: host })`, `where({ post })`, `where({ place })`) and switched the batch `GROUP BY` example from `ops.in(placeIds)` to the idiomatic array form `where({ placeId: placeIds })`.
- **`controllers.md`** — widened the FK-exclusion guidance from "because `extractParams` strips FKs" to reference the general pass-the-instance rule, and added the authorization rationale: a param id is untrusted, verifying it means loading the record (ideally scoped to `currentUser`), and the load hands you the instance to pass.
- **`soft-delete.md`** — fixed the cascade-count examples to filter by the instance (`Room.where({ place })`) instead of `Room.where({ placeId: place.id })`.
- **`SKILL.md`** — bumped the ecosystem version baseline for `@rvoh/dream` from 2.18.x to 2.19.x.

## 0.64.0 — 2026-07-02

### Added

- **`querying.md`** — new "Distinct rows: `.distinct(...)`" subsection: `SELECT DISTINCT` is native to Dream (`.distinct(column)`, `.distinct()`/`.distinct(true)` on the primary key, `.distinct(false)` to clear), so deduplication needs no Kysely eject. Also added to the "What To Reach For First" list so an agent doesn't drop to SQL for it.
- **`querying.md`** — new "Grouped aggregates (`GROUP BY`)" subsection under the `toKysely(...)` escape hatch: Dream's aggregates (`.count()`/`.sum`/`.min`/`.max`) each return a single scalar with no `GROUP BY`, so an aggregate broken out per group (a booking count for each place) is a `toKysely` job — keep Dream for the scoped, association-aware setup and eject for the grouping, querying the associated model directly so its scopes still apply. Documents the two things it depends on: `.clearSelect()` before the aggregate, since `toKysely('select')` seeds `select "<baseAlias>".*`; and `Number(row.count)`, since Postgres returns `COUNT(*)` as a string with absent groups producing no row.

### Fixed

- **The update check now scans every installed copy instead of stopping at the first one.** psychic-skill can be installed in more than one root at once (`~/.agents`, `~/.claude`, `~/.codex`, plus project-local variants), and the host often loads a different copy than the one that sorts first. The old check ran the version check from the first directory it found and stopped there, so a current `~/.agents` copy could mask a stale `~/.claude` copy — the one the host actually loads — and report "up to date" while the running skill was many versions behind. `bin/psychic-skill-update-check` now reads the VERSION of every installed copy and reports `UPGRADE_AVAILABLE` whenever any copy is behind remote, using the lowest version as the baseline. When a copy is behind, the check prints a second line listing each copy and marking the stale ones `(behind)`, which the preamble relays so you can see exactly which install is out of date.
- **`/psychic-update-skill` now reconciles all installed copies in one pass, including sibling global installs.** The updater previously detected a single primary directory and only synced project-local vendored copies alongside it, so a stale sibling global install (e.g. `~/.claude` while `~/.agents` was primary) was left untouched — the same split that hid the staleness. A new `bin/psychic-skill-update-apply` helper upgrades every copy — global and project-local — in one pass (git installs via fetch + reset, vendored installs via a single re-clone), skips developer symlinks, rolls back a failed vendored copy, and prints a per-copy summary. Run `psychic-skill-update-apply --plan` to preview which copies would change before touching anything. This replaces the old detect → upgrade → Step 4.5 local-sync sequence.

## 0.63.0 — 2026-07-02

### Added

- **`serializers.md`** — new "Layering two `delegatedAttribute`s onto the same output key" subsection under `.delegatedAttribute()`: the general "default value, optionally overridden by something more specific" technique — render a fallback association first, then a more-specific association second with `required: false` so it overrides the fallback when present but is skipped (not written as `null`) when absent. Canonical home for the pattern; other docs link here instead of re-deriving it.
- **`serializers.md`** — `.customAttribute()` note now points readers at the layered-`delegatedAttribute` pattern instead of a `current?.title ?? fallback.title` callback, since the latter is invisible to `preloadFor`.
- **`i18n.md`** — the Data-Driven i18n model gains a `fallbackCurrentLocalizedText` `HasOne` (`and: { locale: 'en-US' }`) alongside `currentLocalizedText`, and "Using in Serializers" now shows the two layered as the canonical `PlaceSummaryForGuestsSerializer`, so a locale missing a `LocalizedText` row falls back to the always-present default-locale row instead of serializing `null`. "Combining Both Patterns" updated to build on top of that (no longer redundantly re-delegates `title`).

## 0.62.0 — 2026-07-02

### Added

- **`generators.md`** — new "Choose `--singular` by association shape, not by naming convention" note: decide `--singular` from the owning model's actual association (`HasOne` vs `HasMany`), since a plural resource generated against a `HasOne` parent produces controller/spec scaffolding shaped for a collection association that doesn't exist.
- **`generators.md`** — new "`g:resource` unconditionally overwrites existing model, spec, factory, and serializer files" note: `g:resource` regenerates those files unconditionally with no existence check and no flag to skip them, so re-running it for an already-hand-edited model (e.g., to backfill a missing controller) silently discards prior edits. Documents the commit-first/restore-and-cherry-pick workaround.
- **`controllers.md`** — "Declaring `paramSafeColumns` on the model" corrected: the cap applies to `extractParams` and the model-derived `@OpenAPI` `requestBody`'s `params` (both derive from `paramSafeColumnsOrFallback()`) — but **not** to `requestBody.including`, which is the separate mechanism for exactly the columns that cap leaves out. That's the intended pattern (same as re-adding an FK or STI `type`): `including` documents the field for the frontend while the controller pulls it explicitly via `castParam` instead of bulk extraction, which works the same way for a `paramUnsafeColumns`-blocked column. Also clarifies when the model-level guard is worth adding: every action still declares its own explicit `extractParams` allowlist regardless, so `paramSafeColumns`/`paramUnsafeColumns` are a cap on top of that for a specific column (an admin flag, a payout-account ID, a hand-managed tenancy column) that must never be bulk-assignable no matter what any action's allowlist says — not something to declare reflexively on every model.
- **`controllers.md`** — Redirect response methods now note the `redirectAllowedHosts` allowlist gate: an absolute-URL redirect target must be pre-registered or the request throws a 500 (not a silent no-op); matching is host-only, case- and port/scheme-insensitive.
- **`controllers.md`** — Session & Cookie Management now states plainly that `startSession`/`endSession` are cookie-only with no server-side session store or per-session revocation — only a full cookie-key rotation invalidates outstanding sessions — and documents the default 31-day cookie `maxAge` and how to lower it (`psy.set('cookie', { maxAge: {...} })`, since `startSession` itself takes no options).
- **`openapi.md`** — `validate` section now explains it validates a clone of the request, so `useDefaults`/`coerceTypes`/`removeAdditional` never reach the real `this.params` — except query array-wrapping, which mutates the request in place.
- **`models.md`** — `@deco.Encrypted()` example now warns that its `DoNotSetEncryptedFieldsDirectly` guard is a custom setter, so `setAttributes`/`updateAttributes` (which bypass custom setters) can write unencrypted data straight into the encrypted column with no error.
- **`querying.md`** — new "Client-controlled page size is capped, not unbounded" note documenting the `paginationPageSize` (default 25) / `paginationMaxPageSize` (default 200) `conf/dream.ts` options that cap every paginator, so forwarding a request's `pageSize` is safe by default.

### Changed

- **`SKILL.md`** — bumped the ecosystem version baseline: `@rvoh/dream` 2.17.x → 2.18.x, `@rvoh/psychic` 3.8.x → 3.10.x.
- **`controllers.md`** — corrected the `extractParams` "Other options" block, which showed two options that do not exist. `extractParams` accepts only `key` and `array` (its opts type is exactly `{ key?, array? }`): the allowlist is the required positional array, and `including` is an `@OpenAPI` `requestBody` key that reshapes only the spec, never runtime extraction. Removed the bogus `{ only }` and `{ including }` examples and added a note that nothing widens extraction past the model's `paramSafeColumnsOrFallback()` set. Also tightened the `SKILL.md` one-line summary to say `extractParams` always intersects with the model's param-safe set (declared `paramSafeColumns` or the default) and to list the primary key and polymorphic type fields among the always-stripped columns.
- **`controllers.md`** — consolidated the `including` explanation. It was re-derived from scratch four times inside the `extractParams`/`paramSafeColumns` sections; the `requestBody` shorthand table is now the single canonical explanation, and the extraction-side mentions are one-clause pointers to it (or to "Always-excluded columns"). Every corrected claim is preserved — the exclusion set, the `paramSafeColumns`/`paramUnsafeColumns` cap, and the FK-via-`castParam` rule — just stated once each instead of restated in full.

## 0.61.1 — 2026-07-01

### Changed

- **`models.md`** — corrected a fabricated Aurora PostgreSQL replica-lag figure in the "How stale is 'stale'?" note, and trimmed it to one short sentence per provider. AWS's Aurora replication docs never quantify lag under heavy write load (they only say it "increases"); the "~60 seconds" figure conflated a documented but unrelated legacy connectivity-loss auto-restart timeout with an actual lag measurement. AWS's only quoted figure is "usually much less than 100 milliseconds" for Aurora, with a separate documented caveat that Aurora Serverless v2 lags further if a reader's minimum capacity is set too low relative to the writer; RDS PostgreSQL has no AWS-documented typical figure at all. Points at the `ReplicaLag` CloudWatch metric for real per-workload numbers instead of an invented ceiling.

## 0.61.0 — 2026-07-01

### Added

- **`models.md`** — new "Replica Safety (`@ReplicaSafe`)" section: what the decorator does, that only `select` queries are ever eligible for the replica (`create`/`update`/`destroy` and anything inside a transaction always hit primary), that `innerJoin`/`leftJoin`/`leftJoinPreload` fall back to primary if any joined model isn't `@ReplicaSafe()` while `preload`/`preloadFor` route each association's separate query independently, and the per-call `.connection('primary' | 'replica')` override (available on `Model`, query objects, and `LoadBuilder`). Mark a model `@ReplicaSafe()` by its dominant read-traffic pattern, not by whether any single code path reads it right after a write — a worked BearBnB example contrasts the public, high-traffic `V1::Visitor::PlacesController` (a good `@ReplicaSafe()` candidate for `Place`/`Room`/`LocalizedText`) against `V1::Hosts::PlacesController`/`V1::Hosts::Places::RoomsController`, which force `.connection('primary')` on the narrower read-your-own-write path. Includes realistic replica-lag expectations for RDS vs. Aurora PostgreSQL. Cross-referenced from the `@SoftDelete()` entry's neighboring "Special Decorators" list and from `deploying.md`.
- **`deploying.md`** — new "Read Replicas" section documenting the `replica` credential in `app.set('db', { primary, replica })` (same shape as `primary`, optional) and that configuring it only makes the connection available — routing to it is controlled entirely by `@ReplicaSafe()`, documented in `models.md`.

## 0.60.0 — 2026-07-01

### Changed

- **`migrations.md`** — the aliased-`belongs_to` legacy-form note no longer lists `g:sti-child` among generators that accept `belongs_to`; it does not, since STI children cannot declare associations. Cross-referenced to the sti.md limitation.
- **`sti.md`** — the "cannot define new associations" limitation now notes that `g:sti-child` enforces this at generation time by rejecting `belongs_to` columns.
- **`migrations.md`** — dropped the redundant `dropIndex` call before `dropTable` in the Create Table `down` example; dropping a table in Postgres already drops its indexes.

## 0.59.0 — 2026-07-01

### Added

- **`SKILL.md`** — Critical Rule 15 now calls out that exhaustive-switch case labels must be the literal string itself, never a named constant typed as the enum union (`const x: SomeEnum = 'dispatched'` widens `x` to the whole union, silently breaking the `_never` exhaustiveness check at `default`) — and explicitly rejects working around that by aliasing each literal with `satisfies` instead, which is the same mechanical-constant anti-pattern in different syntax. Verified the underlying widening behavior against `tsc --strict`.
### Changed

## 0.58.0 — 2026-07-01

### Added

- **`SKILL.md`** — Critical Rule 16 now clarifies the distinct jobs of a generated enum's union type vs. its values array: type a variable/property/parameter holding one known enum value as the union type (e.g. `RoomTypesEnum`) and assign the literal directly; reach for the values array (e.g. `RoomTypesEnumValues`) only where a runtime array is actually needed. Prevents over-applying the "never hardcode DB enums" rule into inventing helpers that pluck a single member out of the values array.

## 0.57.0 — 2026-06-30

### Added

- **`controllers.md`** / **`SKILL.md`** — strict rule that a model-derived `requestBody`'s `params` / `including` must be **literal arrays** mirroring the action's `extractParams` allowlist, and must **never** be backfilled by calling `Model.paramSafeColumnsOrFallback()`, `Model.paramSafeColumns`, or `Model.columns()` inside the `@OpenAPI` decorator. Those return the model's entire writable column surface (every table column, names and types, for a model with no declared `paramSafeColumns`), which re-creates the implicit include-all default the explicit-`params` convention exists to remove, advertises fields the action doesn't accept, and exposes the full column surface in the public spec. Added as a clause on SKILL.md Critical Rule 21 and a strict rule in the controllers.md `requestBody` shorthand section.
- **`soft-delete.md`** — new "Natural-key unique indexes" subsection in Setup: when adding `@SoftDelete()` to a model with a natural-key unique index, migrate it to a **partial** unique index with `WHERE deleted_at IS NULL`. Soft-deleted rows stay in the table, so a plain unique index keeps reserving the natural key and blocks a live replacement that reuses it. BearBnB `places_slug_unique` example, cross-referenced to the migrations partial-index typing note, with the `undestroy()`-collision consequence called out.

## 0.56.0 — 2026-06-30

### Added

- **`models.md`** — new "Joining on a non-default key (`on` + `primaryKeyOverride`)" section. Documents the previously-undocumented capability of associating on an arbitrary column pair (e.g. a `uuid` natural key) rather than the conventional id-based FK, by combining `on` (the column that holds the reference) with `primaryKeyOverride` (the column it matches on the other side): the join is `<fk-holder>.[on] = <other-side>.[primaryKeyOverride]`. Includes a paired `BelongsTo` / `HasMany` BearBnB example and a contrast clarifying that `on` / `primaryKeyOverride` change which columns the join uses, `selfAnd` adds a condition on top of the FK join, and `and` filters against a literal.

### Changed

- **`models.md`** — rewrote the `selfAnd` / `selfAndNot` association examples in the BearBnB domain (`featuredRoom`, `siblingRooms`, `peakSeasonBookings`) — they previously used off-domain `DailyChallenge` / `UserChallengeProgress` / `TreeNode` nouns. Clarified in the examples and the options tables that `selfAnd` **adds** its condition on top of the normal FK join (the FK join is kept), rather than replacing it. Added a plain `on` FK-rename example for `HasMany`, and brought the `BelongsTo` `on` / `primaryKeyOverride` examples into the BearBnB domain as proper inverses.

## 0.55.0 — 2026-06-30

### Changed

- **`controllers.md`** — corrected the STI `requestBody` note: `@OpenAPI(BaseModel, ...)` derives its request body from the base model's shared physical table, not from the TypeScript properties on the base class. A column declared (as a `DreamColumn`) only on a child class is still a real column on the shared table, so it is part of the base model's derived request-body surface — re-add an auto-excluded one (FK or `type` discriminator) with `including`, the OpenAPI mirror of `extractParams(BaseModel, [...])`. Check generated table metadata (`src/types/db.ts` / `src/types/dream.ts`) before deciding a field is unavailable. `combining` is reserved for genuine non-column inputs (one-shot tokens, upload metadata, join-table arrays); a model's declared columns and its own virtual attributes both belong to the model-derived surface (`params` / `including`), not `combining`.
- **`controllers.md`, `sti.md`, `migrations.md`** — always show `requestBody: { params: [...] }` on a model-derived create/update, mirroring the `extractParams` allowlist, so the documented request shape is visible at the call site. Dropped the guidance to omit `requestBody` when an action "only takes `paramSafeColumns`" and the "auto-derived shape is `paramSafeColumns`" framing; the implicit include-all default is no longer referenced. Updated the worked examples accordingly (including fixing the nested multi-resource create and the nullable-FK update, which now list param-safe columns under `params` rather than `including`).

### Added

- **`sti.md`** — new "Virtual attributes don't filter up to the base class" section documenting the rift between physical and virtual columns on STI children: physical columns are table-scoped (the base class reaches them via `extractParams(Base, [...])` and `@OpenAPI(Base, ...)`), but a child's `@deco.Virtual` is class-scoped and inherits base → child only. Naming a child virtual through the base type-checks but is silently dropped at runtime; handle it on the child class instead. Cross-referenced from the STI limitations list, the STI controller key-points, and the `controllers.md` `requestBody` STI nuance.

## 0.54.0 — 2026-06-30

### Removed

- **`SKILL.md`** — dropped the `dream-psychic-rag` MCP server from the Critical Rule #2 "Sources of truth" priority list. In an installed Psychic app the authoritative API reference is the packages' shipped TSDocs (version-matched to what's running), so the sources of truth are now TSDocs > `pnpm psy <command> --help` > psychic-skill.

## 0.53.0 — 2026-06-29

### Added

- **`models.md`** — the `@deco.Encrypted` sync note now covers virtual-writability: the plaintext property isn't assignable in `.create()` / `UpdateableProperties` until `pnpm psy sync` lists it under the model's `virtualColumns` (until then `tsc` reports `TS2353 … does not exist`). Separately, at the `ifChanged` hook examples, a caveat to gate on the persisted `encrypted<Name>` column, not the plaintext virtual.
- **`serializers.md`** — to surface associated data on a serializer, use `delegatedAttribute` or `rendersOne(flatten)` (both discovered by `preloadFor`, so auto-preloaded); `preloadFor` doesn't inspect `customAttribute` bodies, so reading an association inside one throws `NonLoadedAssociation` — the signal to switch tools, not to hand-`.preload()`. Cross-referenced from the `customAttribute` docs.
- **`sti.md`** — how to scope a parent-declared association to specific STI child type(s) with an `and` clause on `type` (`@deco.HasMany('Room', { and: { type: 'Bedroom' } })`, or an array for several), without contradicting the rule that children can't declare their own associations.
- **`serializers.md`** — a custom/compound response envelope (a record plus a related collection, or a computed array alongside serialized models) is modeled as one composing `ObjectSerializer` (`rendersOne`/`rendersMany` each part — `{ serializer }`, or `{ dreamClass, serializerKey }` for a model field), passed to `@OpenAPI` — not a hand-written `responses` block.
- **`querying.md`** — sharpened the drop-to-Kysely guidance: hand-roll Kysely only in migrations or as a true last resort, and eject via `toKysely` last (after traversing associations), because association `and` clauses and associated-table default scopes (soft-delete, STI) come from association traversal, not from `toKysely` on the base query — ejecting early or hand-rolling from `db()` resurfaces soft-deleted/excluded rows. Added a `nestedSelect` example showing a scope-preserving subquery as the in-Dream alternative.
- **`testing.md`** — caveat that `toHavePath` compares pathname only, so it is not a barrier when only query params change; after a redirecting mutation, assert eventual state with `expect.poll` or wait on a UI signal tied to the mutation.
- **`SKILL.md`** — member-scoped custom-action routing: a custom action declared directly in a `resources` callback is automatically member-scoped (Psychic prepends `:id`, e.g. `POST /bookings/:id/cancel`); the action reads the id via `this.castParam('id', 'string')`.

### Changed

- **`openapi.md`** / **`controllers.md`** / **`testing.md`** — validation errors are documented and tested as **400** throughout. The framework converts param, request-body, and model-validation failures to 400; to surface field-level errors deliberately, return a 400 carrying the model's `.errors` (`if (place.isInvalid) this.badRequest({ errors: place.errors })`). Removed the prior guidance to return 422 / call `unprocessableContent` for user-facing validation.
- **`openapi.md`** — clarified that `openapiConfig` only toggles `{ omitDefaultHeaders, omitDefaultResponses, tags }` and is not a place to add `responses`; a controller-wide response is declared per-action or via conf `defaults.responses`.
- **`controllers.md`** — the "Custom Response Envelopes" section now keeps only the controller-side wiring (`@OpenAPI(SerializerFn)` + `this.ok`) and defers serializer construction to `serializers.md`; removed the hand-written `$serializable` response-envelope and manual pre-render examples, which cut against modeling a compound response as one composing serializer.
- **`migrations.md`** / **`soft-delete.md`** — the auto-incrementing primary-key example now generates `.addColumn('id', 'bigint', col => col.primaryKey().generatedByDefaultAsIdentity())` (a `generated by default as identity` column), matching the current migration generator.
- **`SKILL.md`** — `--primary-key-type` lists the canonical `uuid7`, `uuid4`, `bigint`, `integer` (`bigserial` is a still-accepted legacy alias, no longer advertised); bumped the ecosystem baseline to `@rvoh/dream` 2.17.x.

### Fixed

- **`controllers.md`** / **`sti.md`** / **`SKILL.md`** — id params are now cast to their primary-key type (`uuid`, `bigint`, or `integer`), never `'string'`. The `castParam Types` reference groups the id-type options (match the app's `primaryKeyType`: `uuid4`/`uuid7` → `uuid`, `bigserial`/`bigint` → `bigint`, `integer` → `integer`) and notes `number` is for decimal values, not ids.
- **`migrations.md`** — corrected the `:encrypted` generator example to use the bare field name (`phone:encrypted`, which yields the `encrypted_phone` column); the generator prepends `encrypted_`, so the prior `encrypted_phone:encrypted` example would have produced `encrypted_encrypted_phone`.

## 0.52.0 — 2026-06-29

### Added

- **`controllers.md`** — new "Cross-Cutting Authorization Gates" subsection: a cross-cutting authorization precondition (accepted-current-ToS, completed-onboarding, active-subscription, verified-email) belongs in one `@BeforeAction` on the authed surface's `AuthedController` (each surface — client, `Admin/`, `Internal/` — has its own), declared after `authenticate`; endpoints that clear the precondition or must answer for a not-yet-cleared user are exempted structurally by re-parenting the namespace's base controller to a looser base (`MaybeAuthedController`, or `UnauthedController` for a no-app-user surface) and self-guarding. Framed as the intentional guarantee that a base controller's hooks are authoritative for its whole subtree.
- **`controllers.md`** — new "`@BeforeAction` scoping" subsection: `{ only, except }` filter by action method name (not by controller), and there is no `skipBeforeAction` / per-subclass override — descendants inherit, redeclaring a same-named hook is a no-op, so vary auth by re-parenting.
- **`controllers.md`** — new "Error markers" subsection: `forbidden(msg)` / `unauthorized(msg)` JSON-stringify the message as the response body, and default error response components are schema-less, so a marker string is a spec-invisible, untyped runtime discriminator for two same-status causes. Plus the typed-enum upgrade (declare the status body as a string `enum`) with the caveat that it only types the spec and the thrown value must be hand-synced, and a scope rule: action-specific cause → per-action `responses`; cross-cutting cause from a shared base → redefine the shared component once at conf level. Includes a security caveat that the marker is always sent to the client at runtime, so it must stay a coarse cause code with no sensitive detail.
- **`openapi.md`** — new "Customizing default error responses" section: the default error set is uniform across auth levels; how to override one status, reshape a shared response once via conf `defaults.components.responses.*`, drop the set with `omitDefaultResponses` (all-or-nothing, then re-add), the per-status precedence order, and the note that `omitDefaultResponses` / `omitDefaultHeaders` are per-action or per-controller (`openapiConfig`) only, never conf-level.
- **`openapi.md`** — new "Relocating or renaming a controller is spec-neutral" section: the spec is keyed by path + HTTP method with no controller class name or `operationId`, so relocating a controller while keeping its route is a zero-diff, no-regen change.

### Changed

- **`controllers.md`** — reworked the controller-hierarchy guidance so any surface that loosens auth is a top-level namespace with the version nested inside (`Visitor/V1/`, `Webhooks/V1/`, `Api/V1/`), never `V1/Visitor/`; `V1/` is authed-only. Rewrote the Directory Structure example, Key Principle #3 (the namespace rule), and Key Principle #5 (generate every surface; reparent the top-level namespace base once when it loosens auth — `Api`/`Webhooks` → `UnauthedController`, `Visitor` → `MaybeAuthedController`), including that a shared base carries only shared auth: one API key on `Api/BaseController`, but each webhook provider verifies its own signature on its own controller. Updated the clean-URL routing example to `Visitor/V1/`.
- **`controllers.md`** — narrowed the `@BeforeAction` guarantee to what is actually enforced: a descendant cannot un-register or re-scope an inherited hook, but overriding the hook *method body* in a subclass still changes its behavior, so vary auth by re-parenting. Clarified that a cross-cutting gate is declared per authed surface, since `Admin/` and `Internal/` have their own `AuthedController`.
- **`SKILL.md`** — surfaced the controller-hierarchy opinion at the points an agent decides structure, not only in `controllers.md`: new Critical Rule #22 (the directory tree IS the auth architecture; loosen-auth surfaces are top-level namespaces, version nested, never `V1/Visitor/`); a matching bullet in the Controllers decision map; and the Routing example now shows top-level `webhooks/`/`api/` surfaces (version nested) instead of modeling only version-first nesting under `v1/`.
- **`generators.md`** — the `g:resource`/`g:controller` route-path argument now flags that its top-level segment is an auth decision: loosen-auth surfaces are their own top-level namespace (`visitor/v1/...`, `webhooks/v1/...`, `api/v1/...`), never `v1/visitor/...`, with a link to the controller hierarchy.
- **`controllers.md`** — added cross-references: to `generators.md` for the scaffolding/reparent workflow, and to `models.md` clarifying that the controller auth tree is not model namespacing (don't mirror it into model names).

### Fixed

- **`openapi.md`** — corrected the stale claim that the default response set includes `422`. The default set is `400/401/403/404/409/500`; Psychic does not auto-emit a `422` (the `ValidationErrors` component exists but no operation references it by default, and `validate.*` does not add one). To document a validation response, declare it yourself.

## 0.51.0 — 2026-06-28

### Added

- **`testing.md`** — New "What the text matchers actually read" subsection: `toMatchTextContent` / `toNotMatchTextContent` assert on rendered `innerText` (joined across elements, input/textarea values included), so they reflect CSS `text-transform`. Default to a case-insensitive regex for text content because case carries no meaning in rendered copy, with a carve-out for identifiers whose case is part of their value (codes, tokens, case-sensitive IDs); plus a note that case-insensitivity is separate from match specificity, and that split label/value markup (`<dt>`/`<dd>`) matches contiguously so no `page.$eval` workaround is needed.

### Changed

- **`models.md` Column Types** — added a rule that column fields must be declared bare, with no `=` initializer: Dream serves column reads through prototype accessors and deletes any shadowing instance property after construction, so a field initializer is silently discarded (the default never applies). The remedy points at a database-layer default (`col.defaultTo(value).notNull()`, per `migrations.md`), with a `BeforeCreate`/`BeforeSave` hook only as the fallback for caller-dependent values.
- **`models.md`** — new "Transforming a column on write" section documenting the custom getter/setter pattern on a real column: read via `getAttribute`, write via `setAttribute` (which bypasses the setter, avoiding recursion), and the invoke-vs-bypass split between `create`/`update`/`assignAttribute(s)`/`this.col =` (run custom setters) and `setAttribute(s)` (bypass them). Distinguished from `@deco.Virtual`.
- **`controllers.md`** — new "String params are trimmed automatically" subsection: Psychic strips leading/trailing whitespace from scalar string params (`castParam` and `extractParams` both route through `Params.cast`), covering scalar strings, enum strings, Virtual string params, and string/enum array elements — so controllers, setters, and hooks should not re-trim.
- **`SKILL.md` Rule 13** — clarified that the "always use `AppEnv`" rule governs reading *application config*; a dev-only launcher composing the environment for spawned child processes legitimately reads `process.env` to spread into `spawn(..., { env })`, since no `AppEnv` accessor represents the whole forwarded environment.

### Fixed

- **`testing.md`** — corrected a stale cross-reference: "every bug is a missing spec" now points at SKILL.md Rule #9 (BDD approach); Rule #8 is now "Sources of truth".

- `/psychic-update-skill` forced checks now bypass stale `just-upgraded-from` markers instead of letting marker state mask a newer remote release.
- Standalone update guidance now verifies the installed version against the remote even when the helper exits cleanly without `UPGRADE_AVAILABLE`, preventing false "up to date" results from cached/local marker state.
- Local vendored sync now detects and checks all project copies (`.agents`, `.claude`, and `.codex`) instead of stopping at the first one, so mixed-agent repos do not leave a stale copy behind.
- The standalone shell snippet avoids zsh's read-only `status` variable name.

## 0.50.0 — 2026-06-28

### Added

- New `openapi.md` documenting the automatic OpenAPI derivation model (Psychic builds the spec from database column types, serializers, and routes; you declare only the remainder) and the conf-level `psy.set('openapi', ...)` customization surface in `conf/app.ts`: namespaces and `outputFilepath`, `defaults` (headers, responses, security schemes, security, components), `validate`, and `syncTypes`. Declaring a bearer security scheme is now covered there as an example of conf-level customization, including the type note that `defaults.security` is an array (`OpenapiSecurity = Record<string, string[]>[]`), not the object form shown in the framework TSDoc.
- **`testing.md`** — New "Running a real external service" subsection: when a feature spec needs a real external dependency (e.g. an auth emulator) rather than a stub, wrap the existing spec command with the service's own `exec`-style runner instead of editing the `globalSetup` / `hooks.ts` harness, since the harness assumes external dependencies are already listening when it boots.
- **`openapi.md`** — Documented why OpenAPI namespaces exist (separate by access domain *and* shape the schema per consumer), the five specs a fresh create-psychic app ships (`default`, `mobile`, `admin`, `internal`, `tests`), the `mobile` spec's `suppressResponseEnums` and the enum-rigidity reasoning behind it (strongly-typed mobile clients crash on unknown enum values; JS web clients degrade gracefully and keep real enums), and the `tests` spec — it aggregates every surface and, via `syncTypes`, generates the types that make controller specs type-safe.
- **`testing.md`** — New "Request and response types come from the `tests` spec" subsection explaining how the typed `OpenapiSpecRequest` derives params and response types from the URI literal + method + status, enforced at `pnpm build:spec`; and a "Generate resourceful controllers first" subsection noting that `g:resource` emits fully-typed controller specs while bare controllers emit an `it.todo` stub, so building resources first seeds the correct pattern.
- **`controllers.md` / `testing.md`** — Explicit rule that any `openapiNames` override must keep `'tests'` in the list; an endpoint omitted from the aggregated tests spec gets no generated types, so its controller spec can't type-check.

### Changed

- Conf-level OpenAPI configuration moved out of `controllers.md` into `openapi.md`. `controllers.md` now covers only the per-action `@OpenAPI` decorator and the `openapiNames` controller override, linking to `openapi.md` for spec-wide config. `SKILL.md`'s OpenAPI Integration section now points at `openapi.md` as the entry point for the derivation model and spec-wide configuration.
- **`controllers.md`** — The `openapiNames` section now shows the real create-psychic defaults: `ApplicationController` returns `['default', 'mobile', 'tests']` (client controllers serve web and mobile from one place), and admin/internal bases return their surface plus `'tests'`. Explains that a controller is documented into every spec it lists and that `'tests'` everywhere is what lets controller specs type-check against one aggregated spec.

## 0.49.3 — 2026-06-26

### Fixed

- Factory examples with required associations now match Dream's generated conditional default pattern (`association: attrs.association ? null : await createAssociation(), ...attrs`).
- The STI child migration rollback example now matches Dream's generated shape by dropping the child column directly instead of explicitly dropping the generated check constraint first.

## 0.49.2 — 2026-06-26

### Fixed

- Factory examples now match Dream's generated factory shape by typing `attrs` as plain `UpdateableProperties<Model>`, including STI child factories.

### Changed

- Added maintainer guidance to remove stale patterns cleanly instead of preserving explanations that only contrast with corrected outdated guidance.

## 0.49.1 — 2026-06-26

### Fixed
- `setup` script post-install message reported `dream-psychic` instead of `psychic-skill`

## 0.49.0 — 2026-06-25

### Changed

- **`SKILL.md` altitude audit — collapsed task-triggered depth to decision triggers + demanding pointers.** `SKILL.md` is the always-loaded core, so every token competes with the agent's working context. Twelve sections that had re-accreted mid-task depth — `## Models`, `## Controllers`, `## Serializers`, `## Soft Delete`, `## Default Scopes`, `## Single Table Inheritance (STI)`, `## Associations`, `## Internationalization (i18n)`, `## Background Workers`, `## Websockets`, `## Testing`, and `## OpenAPI Integration` — were rewritten to the same altitude as the already-refactored `## Generators` section: a decision trigger (when does an agent reach for this?), the 1–3 always-true principles that govern the decision, and a demanding "before you do X, read `<topic>.md`" pointer that names the worst landmine to create urgency without teaching it inline. An agent reading `SKILL.md` now comes away with the core principles and a map of where to go for each task, not a manual. `SKILL.md` drops from ~1003 to ~377 lines.
- **No guidance was removed.** Every collapsed quick-reference, method table, and code example already lived in its topic file (`models.md`, `controllers.md`, `serializers.md`, `sti.md`, `soft-delete.md`, `workers.md`, `websockets.md`, `testing.md`, `i18n.md`, `querying.md`); the always-on Critical Rules, Key Commands, Project Structure, Naming Conventions, Routing, Migrations, Deploying, and Troubleshooting blocks stay resident in `SKILL.md`. Removed the standalone `g:controller` explanatory paragraph after Key Commands (the same detail lives in `controllers.md`).
- **`testing.md`** — Repointed the transaction-callback-type cross-reference from the removed `SKILL.md` Transactions example to [models.md — Transactions](models.md#transactions).

## 0.48.0 — 2026-06-25

### Added

- **`generators.md`** (new file) — A dedicated home for the scaffolding-generator workflow, previously scattered across `SKILL.md`. Owns the generator decision tree, the mandatory `--help` preflight, the `g:resource` argument contract (route path / model file path / `--owning-model` as three orthogonal inputs), the nested-resource `--owning-model` rule, generated defaults (`--no-soft-delete`), the post-generation edit/migrate/spec/commit workflow, `sync` triggers, and the "adding properties to an existing model" migration workflow. Linked from the README manifest and required-reading list.
- **`models.md`** — New "Model Organization & Namespacing" section establishing that a model's namespace should describe what it *is*, not where it is routed or what owns it. A nested route plus `--owning-model` does not imply a `Parent/Child` model namespace (a `Booking` under `v1/host/places/{}/bookings --owning-model=Place` is not `Place/Booking`). `Parent/Child` is for STI subtypes (`Room/Bedroom`) and subdomain / bounded-context modules (`Reservations/Booking`); apps are flat when small and grouped by subdomain as they grow, never organized by the owning model or route. A model that `belongsTo` two parents is its own aggregate root and belongs under neither.

### Changed

- **`SKILL.md`** — Trimmed the generator guidance to a terse, must-not-miss residue (preference order, `--help`-first, the orthogonal `g:resource` arguments, the nested-`--owning-model` rule, the `g:migration`-for-existing-models and `sync` triggers) with pointers to the new `generators.md` and to `models.md` for namespace judgment. Removed the full "Generator Workflow", "When to Run sync", and "Adding Properties to Existing Models" sections, which now live in `generators.md`.
- **`migrations.md`** — Repointed the column-shorthand cross-reference from the removed `SKILL.md` "Adding Properties" section to [generators.md — Adding properties to an existing model](generators.md#adding-properties-to-an-existing-model).

## 0.47.1 — 2026-06-24

### Fixed

- **Updated `create-psychic` CLI flag reference.** `--codex-psychic-skill` was renamed to `--agents-psychic-skill` in create-psychic 3.5.6; updated the flag table and example command in this skill accordingly.

## 0.47.0 — 2026-06-24

### Added

- **`querying.md`** — Documented that a trailing condition object on a non-optional `BelongsTo` is a compile error in the hydrating loaders (`preload` / `load` / `leftJoinPreload` / `leftJoinLoad`), because the constraint could null a field the OpenAPI spec declares non-nullable. Constraints on optional `BelongsTo`, `HasOne`, `HasMany`, and on `innerJoin` / `leftJoin` remain allowed (Dream 2.15.0).
- **`models.md`** — New "required `BelongsTo` is a two-way non-nullable contract" subsection: a non-optional `BelongsTo` may not be conditionally loaded (compile-time), and accessing it when an internal mechanism (default scope, baked-in `and`/`on`, dangling FK) leaves it null throws `MissingRequiredBelongsToAssociation` (runtime). The fix is `dependent: 'destroy'` on the inverse `HasOne`/`HasMany` or `optional: true`, not a looser spec.
- **`models.md`** — Documented the column-encryption config (`app.set('encryption', { columns: { current: { algorithm, key } } })` in `conf/dream.ts`) required for `@deco.Encrypted()`, distinct from the cookie encryption config in `conf/app.ts`.
- **`serializers.md`** — Added a note tying the existing `optional`-as-source-of-truth guidance to the two-way contract and the `MissingRequiredBelongsToAssociation` runtime backstop.
- **`SKILL.md`** — New Critical Rule #2: detect the project's package manager (from `package.json`'s `"packageManager"` field or the lockfile) before running any command, since `pnpm` in every example is a stand-in for the project's actual manager. Notes that `npm` and `bun` need the `run` verb (`npm run psy`, `bun run psy`) while `pnpm`/`yarn` invoke the binary directly. The package-manager disclaimer now also lists `bun run psy sync`. Renumbers prior Critical Rules 2–20 → 3–21 and updates the live cross-references in `SKILL.md`, `sti.md`, `controllers.md`, and `migrations.md`.

### Changed

- **`migrations.md`** — Replaced the vague "require column encryption to be configured" caveat on `encryptColumn` / `decryptColumn` with the concrete `app.set('encryption', { columns: ... })` setup in `conf/dream.ts`, plus the `pnpm psy g:encryption-key` and `pnpm psy sync` steps.
- **`controllers.md`** — Stated explicitly that URL namespace, controller file namespace, and auth inheritance chain are three independent concerns: a versioned URL (`/v1/...`) should not force a matching controller ancestry, since `g:controller` generates files without adding routes and the route file can map any URL onto the controller with the correct ancestry.

### Baseline

- Bumped the documented `@rvoh/dream` baseline from 2.14.x to 2.15.x.

## 0.46.2 — 2026-06-24

### Fixed

- **Added `~/.agents/skills/psychic-skill` to update-check and install-detection search paths.** Codex reads skills from `$HOME/.agents/skills/` per the official Codex docs, not `~/.codex/skills/`. The update-check preamble and `/psychic-update-skill` install detection now check `~/.agents/` first so the skill correctly finds itself and upgrades when installed in Codex.

## 0.46.1 — 2026-06-24

### Changed

- **`bin/psychic-skill-config`** — Made the `set` path portable. It used BSD-only `sed -i ''`, which errors on GNU/Linux, so opting into a config change there (e.g. `set auto_upgrade true` from the "always keep me up to date" prompt) would fail. Replaced with a portable temp-file rewrite that works on both macOS and Linux.

## 0.46.0 — 2026-06-24

### Changed

- **`models.md`** — Removed non-existent `'inclusion'` and `'exclusion'` validator types from the Validations section. Fixed the custom validation example to use `this.addError(column, message)` instead of direct `this.errors[col]` assignment.
- **`models.md`** — Noted in Dirty Tracking that `changedAttributes()` is populated on unpersisted `.new()` instances before the first save.

### Added

- **`testing.md`** — Added rule: never run `uspec` and `fspec` concurrently — both share the test database lifecycle, so parallel execution produces transient false failures.

## 0.45.0 — 2026-06-22

### Added

- **`migrations.md`** — documented `DreamMigrationHelpers.encryptColumn` / `decryptColumn` for converting existing persisted columns to and from `@Encrypted`, including why rename-and-decorate leaves plaintext behind.
- **`controllers.md`**, **`SKILL.md`** — documented that `g:controller` scaffolds namespace `BaseController.ts` files and the leaf controller's inheritance chain, while leaving routes for the developer to wire manually.
- **`querying.md`**, **`models.md`** — made query-level `.update()` semantics explicit: it loads each matched record and runs instance hooks/validations by default; `{ skipHooks: true }` is the raw bulk SQL path.

### Changed

- **`controllers.md`**, **`SKILL.md`** — updated OpenAPI request-body narrowing guidance from `requestBody.only` to the accepted `requestBody.params` wording, while noting `only` as the older compatibility alias.
- **`SKILL.md`** — bumped the documented ecosystem baseline to `@rvoh/dream` 2.14.x and `@rvoh/psychic` 3.8.x.

## 0.44.0 — 2026-06-19

### Added

- **`querying.md`**, **`utils.md`**, **`SKILL.md`** — documented Dream's `range` helper as a `where` predicate helper for bounded single-column comparisons on `CalendarDate`, `DateTime`, `ClockTime`, and `ClockTimeTz` columns. The guidance distinguishes readable single-column bounds from multi-column interval overlap logic, where explicit `ops.lessThan*` / `ops.greaterThan*` comparisons make boundary semantics clearer.
- **`serializers.md`**, **`SKILL.md`** — added serializer export guidance: serializer functions must use named exports, OpenAPI-visible nested `ObjectSerializer`s should be exported so generated schemas get stable names, and computed/view-model serializers should use distinct export names when their domain noun overlaps model serializers.
- **`migrations.md`** — added Kysely DDL gotchas for hand-edited migrations: run a fresh `pnpm psy db:reset` after editing raw check constraints/partial indexes, keep fixed DDL literals in emitted SQL instead of bound parameters, and type raw partial-index predicates as `sql<SqlBool>` when Kysely requires a boolean expression.

### Changed

- **`README.md`**, **`models.md`**, **`workers.md`** — expanded the no-JavaScript-`Date` guidance from `DateTime`/`CalendarDate` to all four Dream date/time classes, including `ClockTime` and `ClockTimeTz`.
- **`controllers.md`** — clarified the STI request-body edge case where a child-only field cannot be listed in base-model `including` / `required` and must be added via `combining`.
- **`testing.md`**, **`SKILL.md`** — corrected validation-layer examples and quick reference from 422 to 400, with a note that 422 is reserved for controller actions that explicitly return `unprocessableContent` / `unprocessableEntity`.

## 0.43.0 — 2026-06-16

### Added

- **`SKILL.md`** — new Critical Rule #20: never hand-code OpenAPI schema for shapes Psychic can derive. Model request bodies use `only` / `including`, model responses use serializers, and computed / view-model responses use `ObjectSerializer`; if the ObjectSerializer does not exist yet, create it rather than duplicating response JSON Schema.

### Changed

- **`controllers.md`** — OpenAPI request-body guidance now explicitly says to derive model field types, nullability, and enum constraints even when the action uses custom `castParam` logic instead of `extractParams`, such as STI discriminator dispatch. Custom response envelopes now make `ObjectSerializer` the default for stable computed response shapes and reserve hand-written `responses` schemas for genuinely ad hoc outputs.
- **`serializers.md`** — ObjectSerializer guidance now calls out stable computed / view-model controller responses as serializer-backed OpenAPI contracts, avoiding drift between returned plain objects and duplicated response schemas.

## 0.42.1 — 2026-06-16

### Fixed

- **`bin/psychic-skill-update-check`** — `--force` now bypasses cache and snooze state in memory even when sandbox permissions prevent deleting `~/.psychic-skill` cache files. State-file writes are best-effort, so an unwritable state directory no longer makes the update checker fail before it can report `UPGRADE_AVAILABLE`.

## 0.42.0 — 2026-06-12

### Added

- **`migrations.md`** — new "Adding a NOT NULL column to a table that already has rows" subsection under NOT NULL columns and defaults: add the column with a temporary default to backfill existing rows, then drop the default in a separate statement in the same migration so application code must set the value going forward (caller-dependent case). Keep a permanent default only when it genuinely fits the domain. Clarifies that column-shorthand generators omit the default by design — expected generate-then-edit, not a generator bug.
- **`testing.md`** — new "The API server runs in-process — stub the backend boundary directly" subsection under Feature Spec Pattern: feature specs start `PsychicServer` in the same Vitest worker, so `vi.spyOn`/`vi.mock` on backend modules intercept server-side code while the browser drives the front end. Corrects the assumption that feature specs require a live key or HTTP recording; reserve Polly for exercising the real client path.
- **`testing.md`** — new "A job that throws fails the enqueuing request in tests, but not in prod" subsection under Background Worker Testing: under `automatic` invocation a backgrounded method runs inline and awaited with no surrounding try/catch, so a throw propagates to the caller and 500s the request in tests, whereas in prod the same throw only moves the BullMQ job to `failed`. Specs driving a backgrounded job must stub its external I/O, and should assert on the job's side effects rather than a status code that only differs because the job ran inline.

## 0.41.0 — 2026-06-12

### Added

- **`SKILL.md`** — new Critical Rule #19: default optional parameters in the signature via a destructured options bag (`{ dryRun = true }: { dryRun?: boolean } = {}`), not by accepting a whole `options` object and re-deriving each value with `??` in the body. Defaults stay visible at the call boundary and each option is declared once. Includes correct/wrong BearBnB examples.

## 0.40.0 — 2026-06-12

### Added

- **`workers.md`** — Scheduled/Cron Jobs section expanded with the orchestrator/worker model:
  - Scheduled services are thin orchestrators that fan out to backgrounded services; keep the permanently-registered scheduler set small (typically `hourly`/`daily`/`weekly`) and do heavy work in dedicated backgrounded services.
  - A class extends `ApplicationScheduledService` (has `schedule()`, no `background()`) **or** `ApplicationBackgroundedService` (has `background()`, no `schedule()`) — never both. A scheduled method cannot enqueue background jobs itself; it delegates to a separate backgrounded service, with imports flowing orchestrator → worker only.
  - Pitfall: `schedule()` keys the BullMQ scheduler by `` `${globalName}:${method}` `` (args excluded) via `upsertJobScheduler`, so looping `schedule()` over a config list silently registers only the last entry. Fix: distinct `(class, method)` pairs, or one fan-out method scheduled once that loops at run time.
  - Per-user cadence across time zones: register the orchestrator on an hourly cron and select users whose local end-of-day/end-of-week falls in the current hour, baking time zone and end-of-week preference into the query to pluck just the matching user IDs, then fan out one backgrounded job each.
  - Note that `schedule()`/`background()` run inline in `NODE_ENV=test`, so environment-guarded methods need a `force`-style override to be exercised in a spec.

### Changed

- **`SKILL.md`** — bumped the ecosystem version baseline to `@rvoh/dream` 2.12.x, `@rvoh/psychic` 3.5.x, and `@rvoh/psychic-spec-helpers` 3.1.x to match current published versions (`@rvoh/psychic-workers` 2.3.x and `@rvoh/psychic-websockets` 3.1.x unchanged).

## 0.39.2 — 2026-05-21

### Changed

- **`SKILL.md`** — renamed skill `name:` from `dream-psychic` to `psychic-skill`. Repo name, install directory, skill registry name, and CLAUDE.md/AGENTS.md references are now a single consistent string.
- **`README.md`**, **`CLAUDE.md`** — updated all `dream-psychic` references to `psychic-skill`.

## 0.39.1 — 2026-05-21

### Changed

- **`psychic-update-skill/SKILL.md`** — standalone flow: the update-check script's exit code is now captured and checked. If the script fails (e.g. sandbox filesystem restrictions prevent removing the cache file), the flow falls back to directly fetching the remote `VERSION` via `curl` and comparing it to the installed version. A bold rule is added: never conclude "up to date" solely because the update-check script produced no output — always confirm by comparing installed vs. remote.

## 0.39.0 — 2026-05-20

### Added

- **`websockets.md`** — Configuration section: updated the initializer guard example from `serviceRole !== 'ws'` to `['websockets', 'worker'].includes(AppEnv.serviceRole)` and restructured the example to the `psy.plugin(async () => { await PsychicAppWebsockets.init(...) })` shape. Added inline comments explaining that the ws process owns `Cable.start()` and socket handling while worker processes only need the Redis connection for `Ws.emit()`, and that each Node process has its own module cache.
- **`websockets.md`** — "Using with Background Workers" section: added a callout block documenting the per-process initialization requirement, the exact error thrown when a worker skips the initializer (`must call cachePsychicAppWebsockets before loading cached psychic application websockets`), why BullMQ retries make it look like a framework cache problem, and the fix.
- **`websockets.md`** — new "Dedicated WebSocket Host: transport configuration" section: documents the Socket.IO long-polling collision failure, the two diagnostic signals (browser network tab showing `transport=polling`, websocket server logging header errors), the `transports: ['websocket']` client fix, and a warning not to add `'polling'` without end-to-end verification.

## 0.38.0 — 2026-05-19

### Added

- **`SKILL.md`** — "Ecosystem versions & staleness policy" block near the top: states the `@rvoh/*` versions the skill is written against (`@rvoh/dream` 2.11.x, `@rvoh/psychic` 3.4.x, `@rvoh/psychic-workers` 2.3.x, `@rvoh/psychic-websockets` 3.1.x, `@rvoh/psychic-spec-helpers` 3.0.x) and the policy that when documented behavior fails (unrecognized generator flag, shorthand producing malformed output, missing API) the first corrective action is to update out-of-date `@rvoh/*` packages, not to work around the skill. Minor/patch bumps are the default. The skill deliberately does not annotate per-feature "available since" versions.
- **`CLAUDE.md`** — skill-maintainer instruction to verify the version baseline block against actual current published versions before finalizing any skill change (and update it in the same PR), with the rule that the list mirrors exactly the packages the skill documents (`@rvoh/dream-plugin-json-snapshot` intentionally excluded) and no per-feature version annotations.
- **`SKILL.md`** — the staleness-policy block now also instructs agents to update peer dependencies alongside any `@rvoh/*` upgrade: a scoped `pnpm up -L "@rvoh/*"` leaves peers behind, which can leave one (in practice `kysely` / `kysely-codegen`, both `@rvoh/dream` peers) at a version the upgraded packages no longer accept. The rule is general — resolve every unmet peer requirement the upgrade introduces.
- **`SKILL.md`** — generator section now states the general rule that nested resources (route path with a `{}` parent-id placeholder, e.g. `v1/posts/{}/comments`) MUST pass `--owning-model=<fully-qualified owning model>`. This scopes the generated controller through `associationQuery`/`createAssociation` on the owner and scaffolds the parent correctly everywhere including the controller spec; omitting it yields an unscoped controller and a spec that references an unconstructed parent (404s on the missing parent instead of exercising the action). Also noted as the way to reintroduce ownership scoping on `admin`/`internal` paths.

## 0.37.0 — 2026-05-18

### Added

- **`models.md` + `SKILL.md`** — documented `ClockTime` and `ClockTimeTz` alongside the existing `DateTime`/`CalendarDate` Date/Time guidance. Adds construction examples for time-of-day values, an explicit Postgres-column → Dream-class mapping table (`timestamp`→`DateTime`, `date`→`CalendarDate`, `time`→`ClockTime`, `timetz`→`ClockTimeTz`) noting that `psy sync` makes `DreamColumn<…>` resolve automatically and that `ClockTimeTz` from SQL is interpreted as UTC, and a note that all four classes carry microsecond precision (preserved via API input / DB hydration; `.now()` is millisecond-only at creation since it is JS-`Date`-backed). Critical Rule 3 now names all four classes; its justification is mechanical (these are what `castParam`/`extractParams` return and what the DB hydrates), not capability-based.

## 0.36.0 — 2026-05-18

### Added

- **`sti.md`** — new "Hand-adding a new base-serializer variant (e.g. `forGuests`)" subsection in Serializer Patterns. Clarifies that the generator emits the default variant pair and *also* `Admin`/`Internal` variant pairs when requested (`g:model` via `--admin-serializers`/`--internal-serializers`; `g:resource` auto-inferred from the `Admin/`/`Internal/` namespace) — all generator-emitted variants already carry the correct STI shape, so admin/internal are not hand-written. Only a bespoke variant with no flag/namespace inference (e.g. `forGuests`) is hand-written, and it must replicate the STI base shape (`StiChildClass` parameter, `DreamSerializer(StiChildClass ?? Parent, model)`, and the `type` attribute whose OpenAPI `enum` is `[(StiChildClass ?? Parent).sanitizedName]`). Contrasts a plain non-STI serializer so the three STI-load-bearing parts are legible, explains that the per-child single-value `type` enum is the discriminator making the OpenAPI a discriminated union, and documents the silent failure mode: a hand-written variant that loses the shape renders correctly in a unit spec but drops child-specific fields over HTTP under `fastJsonStringify` — suspect the serializer/OpenAPI schema, not model instantiation or `preloadFor`.
- **`testing.md`** — new "Negative specs: the principal still needs its auth role row" subsection after the Authentication Helper. A negative controller spec expecting a `403`/`404` from an ownership check must still give the current principal whatever role row the auth layer requires, or the auth `BeforeAction` returns `403` before the ownership lookup runs — the spec passes for the wrong reason and silently loses coverage of the branch it claims to exercise. Includes the sibling-positive-spec sanity check.
- **`soft-delete.md`** — design-judgment note up front: don't hand-roll a `removed`/`isDeleted`/`deactivatedAt` mechanism; `@SoftDelete()` (applied by generators by default) is that mechanism, and a custom column fights `destroy()`/`undestroy()`, the `dream:SoftDelete` default scope, and `dependent: 'destroy'` cascades. A domain status flag is warranted only when it means something other than deletion (e.g. an `active` flag meaning "currently bookable" on a still-live row).
- **`controllers.md`** — new Key Principle 5: custom-token surfaces (signed public links, webhooks, partner APIs) are still scaffolded with the generators into their own namespace, then the namespace base controller is manually repointed to extend the branch's `UnauthedController` and verify the token in a `@BeforeAction` — keeping generator conventions and the one-base-controller-per-directory rule while avoiding accidental inheritance of session-auth `@BeforeAction`s.

## 0.35.0 — 2026-05-18

### Added

- **`models.md`** — new "Anchor polymorphism to a stable model" design note in the polymorphic-association section. When a record participates polymorphically in more than one direction, introduce a stable join model that owns the participant polymorphism and acts as the fixed boundary, rather than stacking polymorphic ownership onto an already-generic model. Includes a BearBnB-domain shape (`ConversationParticipant` → polymorphic `Guest | Host`; `ConversationThread.context` → polymorphic `Booking | Place | ...`) showing why each model should have exactly one polymorphic association to reason about.

### Changed

- **`controllers.md`** — corrected and deepened the decryption-error guidance. The three errors import from `@rvoh/dream/errors` (not the `@rvoh/dream` root), and the rotation error's real name is `DecryptionRotationError` (was incorrectly `DecryptionWithRotationError`). The "Decrypting Cookies Outside a Controller" example no longer blanket-catches: it uses the three-arg (current + legacy) rotation form and **rethrows `DecryptionRotationError` by default** — a broken rotation is a configuration defect that must fail loudly at deploy time (first request / smoke test / health check), not be logged-and-swallowed while every user is silently logged out; downgrading it to log-at-`error` + unauthenticated is documented as a conscious, reversible posture only after a rotation has proven stable in production for a prolonged window. `DecryptionError` (single-key stale/forged cookie) is logged at `warn` via `PsychicApp.logWithLevel` and converted to an unauthenticated response. `DecryptionParseError` and anything else propagate (cipher + auth tag validated but the decrypted plaintext was not valid JSON → a format/contract mismatch — your encoder or, with a shared cross-service key, theirs — never an auth outcome, 500). Frames the `catch` as "convert to an auth decision *and emit the security signal*," grounded in audit finding R-019 (silent-null lost incident-log signal and hid broken rotation). Documents that `@Encrypted` columns and other system-controlled ciphertext should propagate untouched (never caught/nulled), that `Encrypt.decrypt` returns the already-`JSON.parse`d value, that `null`/`undefined` ciphertext returns `null`, that a missing key throws `MissingEncryptionKey`, and the three-arg rotation semantics (legacy fallback only on `DecryptionError`; a current-key `DecryptionParseError` is neither retried nor wrapped). Fixed the `AuthedController` example's double-parse (`JSON.parse(decrypted)` on an already-parsed object).

## 0.34.0 — 2026-05-12

### Changed

- **`controllers.md`** — rewrote the "Generator output" subsection to reflect that `psy g:resource` (and related generators) now emit a shared `paramSafeColumns` const at the top of the controller file referenced from every `create` / `update` action. Admin and non-admin scaffolds emit the same shape. Replaces the prior "materialize the full safe-list into the array at each call site, delete what doesn't belong" framing. The const is shown with its real generated type (`DreamParamSafeColumnNames<Model>[]`) and the `@rvoh/dream/types` import, and the example now wires the same const into the `create` / `update` `@OpenAPI` decorators as `requestBody: { only: paramSafeColumns }` — making the documented request body and the runtime `extractParams` allowlist a single edit point that can't silently diverge. Notes the const is only emitted for controllers with `create`/`update` and a known model, the typed-empty `= []` form, that it's `requestBody` (input) not the response, and that `requestBody.including` still layers on top unchanged. `SKILL.md` `extractParams` API-table row gains a one-line note about the scaffold lockstep.
- **`deploying.md`** — Postgres TLS section updated for the framework's narrowed `SingleDbCredential.ssl` type (`TlsConnectionOptions | false`; bare `ssl: true` is no longer accepted) and the new `MissingDbSslDirective` throw at `app.set('db', ...)` time when neither `ssl` nor `useSsl` is set. Added the boilerplate `DB_NO_SSL` default, a managed-provider matrix (Supabase / Neon / Render / Azure Flexible Server work with the verified-TLS default; AWS RDS and GCP Cloud SQL need a CA bundle; Heroku Hobby is the `rejectUnauthorized: false` path), and reframed the `useSsl` paragraph as a migration note for apps scaffolded before this change rather than as new-app guidance.

### Added

- **`migrations.md`** — guardrail on the JSON column section: reaching for `json` or `jsonb` should be a stop-and-reconsider moment because Dream's column-type inference and Psychic's auto-derived OpenAPI shapes short-circuit on schemaless blobs. Calls out the relational alternatives (HasMany / BelongsTo for repeated keyed data, real columns for bounded attributes), narrows the legitimate use cases (verbatim third-party payloads, opaque per-tenant config, audit snapshots), and asks authors to document in the migration why the relational alternative was rejected.
- **`SKILL.md` Critical Rule 14** — exhaustive `switch` with a `const _never: never` default is now the top-level rule for branching on any closed-enum value: database enums regenerated into `@src/types/db.js`, STI type discriminators, and hand-written unions of string literals. `if/else if` chains type-check fine but silently no-op when a future enum value is added; the `_never` default turns "missing case" into a compile error. The STI controller switch-on-`type` pattern is reframed as one example of this rule, not the rule itself. Renumbers prior rules 14-17 → 15-18. Adds Rule 14 pointers from the STI sections in `SKILL.md` and `sti.md`.
- **`migrations.md` + `SKILL.md`** — aliased BelongsTo shorthand `Model@alias:belongs_to[:optional]` drives FK column, typed FK property, and association name from a single snake_case token (e.g., `InternalUser@canceled_by:belongs_to:optional` → `canceled_by_id` / `canceledById` / `canceledBy`). Documents the canonical `_by` use case, multi-FK disambiguation, and namespace strip; covers all generators that accept `belongs_to` (`g:model`, `g:sti-child`, `g:migration`, `g:resource`). Updates the self-referential-FK guidance to recommend the alias over post-generation hand-edits.
- **`migrations.md`** — shorthand-reference row for the reuse form of enum columns (`name:enum:enum_type_name`, no inline values), distinct from the declare-with-values form.
- **`testing.md` Factory Pattern** — reused-enum columns in generated factories emit a TS-rejecting `'TODO'` literal plus a comment hint; `pnpm build:test-app` fails fast at the factory rather than surfacing a runtime NOT NULL / enum-mismatch the first time the factory runs. Covers scalar and array forms.
- **`deploying.md` `## Migrations in Production Deploys`** — new top-level section after Postgres TLS. *The migrate task is the boot smoke test*: `db:migrate` exercises Node boot, `loadEnv`, AppEnv materialization, Psychic registry initialization, and DB connection establishment before applying any migrations, so a separate no-op boot smoke task is redundant. *Combine `db:migrate && db:seed` in one invocation*: on serverless container platforms with fixed per-task overhead, separate one-off invocations pay the per-task lifecycle twice for no reason; combine via `sh -c "… db:migrate && … db:seed"` for exit-zero semantics.

### Changed

- **`migrations.md` JSON example block** — non-optional `jsonb` columns now show the auto-default the generator emits (`col.notNull().defaultTo(sql\`'{}'::jsonb\`)`); optional `jsonb` stays bare (null is the intended initial state). Matches the boolean / array auto-default examples elsewhere in the file.
- **`models.md` Decorator Setup** — now shows both forms: the active `import { Decorators } from '@rvoh/dream'` + `const deco = …` for models with field-level `@deco.*` decorators, and the commented form (with `Decorators` dropped from the merged import) for freshly generated models without any. STI / `@SoftDelete` class-level decorators are noted as unaffected.
