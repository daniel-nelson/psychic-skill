---
name: psychic-skill
description: >
  Comprehensive guide for developing applications with Dream ORM and Psychic web framework.
  TRIGGER when: code imports from '@rvoh/dream', '@rvoh/psychic', '@rvoh/psychic-workers', or '@rvoh/psychic-websockets',
  or project has Dream models, Psychic controllers, or uses 'psy' commands.
  Covers models, associations, validations, hooks, scopes, serializers, controllers, routing,
  migrations, background workers, websockets, OpenAPI, testing, and code generation.
user-invocable: false
allowed-tools: Read, Grep, Glob, Bash, Edit, Write
---

# Dream ORM & Psychic Web Framework Development Guide

You are working with **Dream** (a TypeScript Active Record ORM) and **Psychic** (a batteries-included TypeScript web framework built on Koa). Both are open source packages published under the `@rvoh` npm scope.

**Update check**: !`for d in "${CLAUDE_SKILL_DIR:-}" "${CODEX_SKILL_DIR:-}" "$HOME/.agents/skills/psychic-skill" "$HOME/.claude/skills/psychic-skill" "$HOME/.codex/skills/psychic-skill" ".agents/skills/psychic-skill" ".claude/skills/psychic-skill" ".codex/skills/psychic-skill"; do [ -n "$d" ] && [ -x "$d/bin/psychic-skill-update-check" ] && "$d/bin/psychic-skill-update-check" 2>/dev/null && exit 0; done`
If the output above says `UPGRADE_AVAILABLE <old> <new>`, follow the inline upgrade flow in `/psychic-update-skill`.
If the output says `JUST_UPGRADED <old> <new>`, tell the user: "psychic-skill upgraded from v{old} to v{new}!" and continue.

All CLI commands in this document are run via the local project's package manager (e.g., `pnpm psy sync`, `yarn psy sync`, `npm run psy sync`, `bun run psy sync`). Examples use `pnpm` but substitute the project's actual package manager; the package-manager Critical Rule below explains how to detect which one a project uses (and that `npm` and `bun` need the `run` verb).

**Note on examples:** Code examples throughout this skill use BearBnB, a demo app that creates an AirBnB clone for bears (https://github.com/daniel-nelson/bearbnb). In this domain, **Guest** and **Host** are application roles (a Guest books a place to stay, a Host lists a place) — not to be confused with "visitor" (unauthenticated user) or "server" (the machine).

**Ecosystem versions & staleness policy.** This skill is written against `@rvoh/dream` 2.15.x, `@rvoh/psychic` 3.8.x, `@rvoh/psychic-workers` 2.3.x, `@rvoh/psychic-websockets` 3.3.x, and `@rvoh/psychic-spec-helpers` 3.2.x. Features and generator behavior described here assume versions at or above these. **Stay current.** If something documented in this skill fails — a generator flag is unrecognized, shorthand produces malformed output (e.g. an `@alias` passing through literally into identifiers), an API is missing — the first corrective action is to update the out-of-date `@rvoh/*` packages, not to work around the skill. Minor and patch bumps within these majors are low-risk and cheap to apply; treat keeping these packages up to date as the default. This skill deliberately does **not** annotate which version each individual feature landed in — assume current, and upgrade if reality disagrees with the skill.

**Always update peer dependencies alongside `@rvoh/*`.** A scoped command like `pnpm up -L "@rvoh/*"` upgrades only the `@rvoh` scope and leaves peer dependencies behind, which can leave a peer pinned at a version the upgraded `@rvoh/*` no longer accepts. After any `@rvoh/*` upgrade, also bump the peers needed to satisfy the new peer ranges — in practice `kysely` and `kysely-codegen` (both `@rvoh/dream` peers) are the ones that bite, but the rule is general: resolve every unmet peer requirement the upgrade introduces, don't stop at the `@rvoh` scope.

## Critical Rules

**If something is failing unexpectedly, re-read this skill before debugging.** Most common errors (type mismatches, missing associations, validation failures, generator syntax issues, migration problems) are already documented here with solutions. Spending 30 seconds searching this skill is cheaper than spending 30 minutes debugging from first principles.

1. **ALWAYS read the project's `AGENTS.md` or `CLAUDE.md` first** before doing any work - these contain project-specific conventions that override general patterns.
2. **Detect the project's package manager before running any command; `pnpm` in this skill is a stand-in.** Every command example here is written `pnpm psy ...`, but `pnpm` means *whatever package manager the project uses*. Determine it before running anything and substitute: check `package.json`'s `"packageManager"` field if present (authoritative), otherwise the lockfile — `pnpm-lock.yaml` → `pnpm psy ...`, `yarn.lock` → `yarn psy ...`, `package-lock.json` → `npm run psy ...`, `bun.lock`/`bun.lockb` → `bun run psy ...`. `npm` and `bun` need the `run` verb (`npm run psy`, `bun run psy`); `pnpm` and `yarn` invoke the binary directly. Running `pnpm` literally in a non-pnpm project resolves against the wrong lockfile and tends to fail quietly rather than loudly — verify the package manager first, every time.
3. **ALWAYS run `pnpm psy <command> --help`** before using any generator - never guess syntax.
4. **NEVER use JavaScript `Date`** - always use `DateTime`, `CalendarDate`, `ClockTime`, or `ClockTimeTz` from `@rvoh/dream` (timestamp / date / time-without-tz / time-with-tz respectively). These are what `castParam`/`extractParams` return and what the DB hydrates, so a JS `Date` is never what flows through the system. See [models.md — Date/Time](models.md#datetime).
5. **NEVER stub or mock Dream internals** in tests - use factories to create real model instances.
6. **NEVER modify an existing migration file that has already been merged into main.**
7. **A generator must always be used** when creating new models, controllers, or migrations.
8. **Sources of truth** (priority order): TSDocs > `pnpm psy <command> --help` > psychic-skill > MCP server (dream-psychic-rag).
9. **BDD approach**: Write failing spec first, then implement. Generated code is the only exception (generators create scaffolding for specs and implementation simultaneously).
10. **Run `pnpm psy sync`** after changing associations, serializers, OpenAPI decorators, or routes.
11. **Only use comments to explain "why", not "what"** - prefer expressive code over comments. TSDoc comments explaining "how" or "when" to use a function/method/class are welcome.
12. **Use Dream's built-in utilities** (`@rvoh/dream/utils`) instead of lodash or hand-rolled equivalents. See [utils.md](utils.md) for the full list.
13. **NEVER use `process.env` directly** - always use `AppEnv` (`api/src/conf/AppEnv.ts`) to access environment variables. `AppEnv` validates that required variables are present at boot time with clear error messages, and decouples the application from the container environment (enabling secret injection from external sources like AWS Secrets Manager). Variables that are only present in some environments (e.g., a deploy-only credential blob absent in local dev) use the `{ optional: true }` overload, which returns `string | undefined` instead of throwing — never reach for `process.env` to avoid the throw. Example: `const credentialConfig = AppEnv.string('GOOGLE_CALENDAR_CREDENTIAL_CONFIG', { optional: true })`; gate the deployed-vs-local path with `if (credentialConfig) { ... } else { ... }`.
14. **NEVER add try/catch blocks unless handling a specific, expected error.** Dead programs tell no lies — an unhandled exception with a stack trace is far more useful than a program that silently swallows errors and continues with corrupted state. Psychic already converts common errors to appropriate HTTP responses automatically (e.g., `findOrFail` → 404, `castParam` → 400, validation failure → 400). If you must catch, handle only the specific error you expect and re-throw everything else. Never wrap large blocks of code in a catch-all try/catch. **Two rationalizations to reject explicitly:** (a) "I'm logging, not silently swallowing" — logging is for humans reading logs after the fact, not for machines deciding what to do next; if the caller is an HTTP handler, the user gets a 200 instead of a 500; if the caller is a BullMQ worker, the job is marked successful and never retried; a `console.error` line does not influence control flow. (b) "This is a small per-iteration catch, not a large block" — the size of the wrapped code is not the test; the test is whether the failure needs to propagate. A 3-line per-iteration catch inside a loop hides failures just as effectively as a 300-line function-wide catch.
15. **Branching on a closed-enum value ALWAYS uses an exhaustive `switch` with a `const _never: never` default.** Closed enums include database enums regenerated into `@src/types/db.js` (e.g., `MessagingMessageRequestStatusesEnum`), STI type discriminators (e.g., `RoomTypesEnum`), and any other union of string literals declared in code. `if/else if` chains on these values type-check fine but **silently no-op** when a future enum value is added — the addition compiles, the branch is missing, and the bug surfaces at runtime with no stack trace. The `never` default branch turns "missing case" into a compile error. No-op cases get explicit `case` arms with `return` so every enum value is named in the code; the `_never` default proves it. Reviewers should reject `if/else if` on enum values the same way they reject `try/catch` without a specific expected error. The STI controller switch-on-`type` pattern is one example of this rule, not the rule itself.

    ```ts
    const status = this.status
    switch (status) {
      case 'dispatched':
        await Bridge.handleDispatched(this.id)
        return
      case 'cancelled':
        await Bridge.handleCancelled(this.id)
        return
      case 'pending_draft':
      case 'drafted':
        return  // explicit no-op — case is acknowledged, not forgotten
      default: {
        const _never: never = status
        throw new Error(`Unhandled status: ${String(_never)}`)
      }
    }
    ```

16. **The database is the source of truth for types defined at the database level.** Never hardcode database enum values or other database-defined types in application code. Always import the auto-generated constants and types from `@src/types/db.js` (e.g., `PlaceStylesEnumValues`, `RoomTypesEnum`). These are regenerated by `pnpm psy sync` and propagate automatically to TypeScript types, OpenAPI specs, and generated frontend clients. The only place database enum literal strings belong is inside migration files.
17. **Commit all auto-generated files** after `pnpm psy db:migrate` or `pnpm psy sync`. This includes files in `src/types/`, `src/openapi/`, and any configured sync output directories (e.g., `client/api/`, `admin/api/`). Don't cherry-pick which generated files to stage.
18. **NEVER use `console.log` in application code.** Use `PsychicApp.log(message, ...meta)` for general output and `PsychicApp.logWithLevel(level, message, ...meta)` when you need to set a specific level (`'debug' | 'info' | 'warn' | 'error'`). These route through the configured logger (Winston by default) so output respects log level, transports, and structured format. `console.log` bypasses all of that and pollutes production logs with unstructured strings. See [controllers.md "Logging"](controllers.md#logging). REPL / `psy console` sessions are exempt — interactive output to `stdout` is the point.
19. **For Psychic surfaces that are thin wrappers over Koa, defer to upstream Koa docs.** When Psychic exposes a Koa-layer knob (e.g., `psy.set('json', { ... })` for `koa-bodyparser`, `psy.use(...)` for Koa middleware), the skill teaches the Psychic-specific shape — where the knob plugs in, what generators emit, what's Psychic-specific — and links to the upstream README for option shapes, defaults, and behavior. Don't restate upstream behavior in the skill; it competes with the authoritative source and goes stale. The same posture applies to thin wrappers over `ioredis`, `BullMQ`, `socket.io`, and `pg`.
20. **Default optional parameters in the signature, not in the body.** Destructure an options bag with defaults right in the parameter list, so the defaults are visible at the call boundary and each option is declared exactly once. Don't accept a whole `options` object and then re-derive each value with `??` inside the body — that splits one parameter into two declarations, hides the defaults below the signature, and drifts out of sync as options are added.

    ```typescript
    // CORRECT — defaults live in the signature, applied once
    public static async _reconcilePlace(
      placeId: string,
      { dryRun = true, notifyHost = false }: { dryRun?: boolean; notifyHost?: boolean } = {},
    ) {
      const place = await Place.find(placeId)
      if (!place) return
      // ...use dryRun / notifyHost directly
    }

    // WRONG — option bag passed through, then defaulted after the fact
    public static async _reconcilePlace(
      placeId: string,
      options: { dryRun?: boolean; notifyHost?: boolean } = {},
    ) {
      const dryRun = options.dryRun ?? true
      const notifyHost = options.notifyHost ?? false
      // ...
    }
    ```

21. **NEVER hand-code OpenAPI schema for a shape Psychic can derive.** Before writing `requestBody.properties`, `responses[status].properties`, or `enum: SomeEnumValues`, stop and choose the derived path:
    - Model request bodies use `requestBody: { params: [...] }` / `{ including: [...] }`, even when the action must use `castParam` instead of `extractParams` for STI dispatch or custom validation. `params` is the OpenAPI request-body narrowing key; older code may still show `only`, but new docs and scaffolds should use `params`.
    - Model responses use `@OpenAPI(Model, { serializerKey })` and serializers.
    - Computed / view-model responses use an `ObjectSerializer` passed to `@OpenAPI(SerializerFn, { status })`; nested computed objects become nested `ObjectSerializer`s. If the ObjectSerializer does not exist yet, create it.
    - Hand-written JSON Schema is only for genuinely ad hoc inputs or outputs that cannot sensibly be represented by a Dream model, serializer, or newly-created ObjectSerializer. Do not use "there is no serializer yet" as a reason to hand-write `responses`. Keeping OpenAPI attached to serializers gives TypeScript a single implementation surface for the returned data and documented schema instead of letting a plain object and a duplicated schema drift independently.

## Creating a New Psychic Application

If you are not already inside a Psychic project, use `create-psychic` to scaffold one:

```bash
npx @rvoh/create-psychic new <app-name> [options]
```

**Every option must be provided**, or `create-psychic` will enter interactive mode to prompt for unanswered questions. Boolean options can be negated with `--no-` (e.g., `--no-workers`).

| Option | Description |
|--------|-------------|
| `--package-manager <pm>` | `pnpm`, `yarn`, or `npm` |
| `--primary-key-type <type>` | `uuid7`, `uuid4`, `bigserial`, `bigint`, or `integer` |
| `--workers` / `--no-workers` | Include or exclude background workers (`@rvoh/psychic-workers`) |
| `--websockets` / `--no-websockets` | Include or exclude websockets (`@rvoh/psychic-websockets`) |
| `--client <type>` | Client app: `nextjs`, `react`, `vue`, `nuxt`, or `none` |
| `--admin-client <type>` | Admin client app (same options as `--client`) |
| `--internal-client <type>` | Internal client app (same options as `--client`) |
| `--claude-psychic-skill` / `--no-claude-psychic-skill` | Install or exclude psychic-skill for Claude Code |
| `--agents-psychic-skill` / `--no-agents-psychic-skill` | Install or exclude psychic-skill for Codex / .agents-compatible agents |

Example:

```bash
npx @rvoh/create-psychic new my-app --package-manager pnpm --primary-key-type uuid7 --workers --no-websockets --client react --admin-client none --internal-client none --claude-psychic-skill --no-agents-psychic-skill
```

## Project Structure

```
api/
  src/
    app/
      models/           # Dream models (ApplicationModel base)
      controllers/      # Psychic controllers
      serializers/      # Response serializers (DreamSerializer / ObjectSerializer)
      services/         # Business logic, backgrounded services
      view-models/      # Complex data transformation models
    conf/
      app.ts            # Psychic app config
      dream.ts          # Dream ORM config
      routes.ts         # Route definitions
      AppEnv.ts         # Typed environment variables
      initializers/     # Boot-time initializers (websockets, workers, etc.)
    db/
      migrations/       # Kysely migrations
    types/
      db.ts             # Auto-generated Kysely database types
      dream.ts          # Auto-generated Dream type config
  spec/
    unit/               # Unit tests (models, controllers)
    features/           # E2E tests
    factories/          # Test data factories
```

## Key Commands

**CRITICAL: All `pnpm psy` commands default to `NODE_ENV=test`.** To operate on the development database, prefix with `NODE_ENV=development`. See [console.md](console.md) for details.

```bash
pnpm psy db:migrate              # Run migrations, then automatically runs sync — do NOT follow with a separate `pnpm psy sync`
pnpm psy db:rollback             # Rollback last run migration (use --steps to specify multiple rollback steps)
pnpm psy db:reset                # Drop + create + migrate, then sync
pnpm psy sync                    # Sync types, OpenAPI specs, and cli:sync commands (only needed standalone when no migration was run)
pnpm psy routes                  # Display all routes
pnpm psy --help                  # List all psy commands

# Console (for interactive exploration and scripts)
NODE_ENV=development pnpm console   # Launch Dream console against dev DB

# Generators (ALWAYS run --help first)
pnpm psy g:resource path/to/plural-resources Model/Path field:type   # Preferred for HTTP-accessible models
pnpm psy g:model ModelName field:type                # When model won't be HTTP-accessible
pnpm psy g:controller Path/Name action1 action2
pnpm psy g:migration description                     # For schema changes without a new model
pnpm psy g:sti-child Model/Child extends Parent field:type
pnpm psy g:encryption-key [--algorithm aes-256-gcm]   # Generate a key for @Encrypted / cookie encryption

# Testing & Quality
pnpm uspec                       # Unit specs
pnpm fspec                       # Feature specs (headless)
pnpm fspec:visible               # Feature specs (visible browser)
pnpm build:spec                  # Check for type errors in src and spec directories (uses tsconfig.build-spec.json)
pnpm build                       # Production build or check for type errors only in src directory (uses tsconfig.build.json)
# NEVER use `npx tsc --noEmit` — it fails with spurious errors because the base
# tsconfig references spec types that aren't resolvable from a bare tsc invocation.
pnpm format                      # Apply standard formatting
pnpm lint                        # Check linting
```

`g:controller` scaffolds the namespace base-controller chain as well as the leaf controller and unit spec. For `Path/Name`, it creates a `Path/BaseController.ts` and makes `Path/NameController` extend that base. It does not add routes. Use it even for custom/webhook controllers, then re-parent the generated namespace base to `UnauthedController` when the surface is intentionally unauthenticated.

## Generators

- **Never hand-roll what a generator can make.** Models, resources, migrations, and STI children are always created with `pnpm psy g:*` and then edited — never written from scratch.
- **Generator order:** `g:resource` (default for almost any model) → `g:sti-child` (STI children) → `g:model` (only for models never exposed via any API) → `g:migration` (schema change without a new model).
- **Before you run any generator, read [generators.md](generators.md) and run `pnpm psy <command> --help`.** It carries the argument contract, the mandatory `--owning-model` rule for nested resources, what each generator scaffolds by default, and the post-generate workflow. Reaching for a generator without reading it produces subtly wrong scaffolding that is expensive to unwind.

## Models

For detailed model patterns, see [models.md](models.md).

### Quick Reference

```typescript
import { Decorators, DreamConst, SoftDelete, STI } from '@rvoh/dream'
import { DreamColumn, DreamSerializers } from '@rvoh/dream/types'
import ApplicationModel from './ApplicationModel.js'

const deco = new Decorators<typeof Place>()

@SoftDelete()
export default class Place extends ApplicationModel {
  public override get table() {
    return 'places' as const
  }

  public get serializers(): DreamSerializers<Place> {
    return { default: 'PlaceSerializer', summary: 'PlaceSummarySerializer' }
  }

  // Columns - use DreamColumn<Model, 'columnName'> for type safety
  public id: DreamColumn<Place, 'id'>
  public name: DreamColumn<Place, 'name'>
  public style: DreamColumn<Place, 'style'>
  public sleeps: DreamColumn<Place, 'sleeps'>
  public createdAt: DreamColumn<Place, 'createdAt'>
  public updatedAt: DreamColumn<Place, 'updatedAt'>
  public deletedAt: DreamColumn<Place, 'deletedAt'>

  // Encrypted columns
  @deco.Encrypted()
  public phone: DreamColumn<Place, 'encryptedPhone'>
  // Request bodies and OpenAPI use `phone`, not `encryptedPhone`; the accepted
  // type is string or string | null based on the encrypted column's nullability.

  // Virtual — accepted by create(), update(), and extractParams() but not stored directly in DB.
  // Decorator goes on whichever accessor is declared first.
  // Getter and setter MUST be synchronous — never async. Type argument sets the
  // OpenAPI shape in serializers and model-derived request bodies. See models.md.
  // Note the use of `DreamColumn<Place, 'grams'>` to specify the type of the
  // getter return type and setter params because these types are simply
  // mutations of what is in the database and the database is the source
  // of truth for the type.
  const GRAMS_PER_POUND = 453.592

  @deco.Virtual(['number', 'null'])
  public get pounds(): DreamColumn<Place, 'grams'> {
    const grams = this.getAttribute('grams')
    return grams === null ? null : grams / GRAMS_PER_POUND
  }

  public set pounds(pounds: DreamColumn<Place, 'grams'>) {
    this.setAttribute('grams', pounds === null ? null : pounds * GRAMS_PER_POUND)
  }

  // Associations
  @deco.HasMany('Room', { order: 'position', dependent: 'destroy' })
  public rooms: Room[]

  @deco.HasMany('HostPlace', { dependent: 'destroy' })
  public hostPlaces: HostPlace[]

  @deco.HasMany('Host', { through: 'hostPlaces' })
  public hosts: Host[]

  @deco.BelongsTo('User')
  public user: User
  public userId: DreamColumn<Place, 'userId'>

  // Hooks
  @deco.AfterCreate()
  public async createDefaultRoom(this: Place) {
    await this.createAssociation('rooms', { type: 'LivingRoom' })
  }

  @deco.AfterUpdateCommit({ ifChanged: ['status'] })
  public async notifyStatusChange(this: Place) {
    await NotificationService.background('placeStatusChanged', this.id)
  }

  // Validations
  @deco.Validates('presence')
  public declare name: DreamColumn<Place, 'name'>

  @deco.Validates('numericality', { min: 1, max: 50 })
  public declare sleeps: DreamColumn<Place, 'sleeps'>

  // Scopes
  @deco.Scope()
  public static active(query: Query<Place>) {
    return query.where({ status: 'active' })
  }

  // Note: @SoftDelete() above automatically adds a default scope (dream:SoftDelete)
  // that hides records where deletedAt is not null. No manual default scope needed.

  // Sortable (requires deferrable unique constraint in migration)
  @deco.Sortable({ scope: 'place' })
  public position: DreamColumn<Place, 'position'>
}
```

### Key Model Operations

```typescript
// Create
const place = await Place.create({ name: 'Cozy Cabin', style: 'cabin', sleeps: 4 })

// Find
const place = await Place.find(id)              // or null
const place = await Place.findOrFail(id)         // throws -> 404 in controller
const place = await Place.findBy({ name: 'x' }) // by attributes

// Update
await place.update({ name: 'New Name' })

// Destroy (soft delete if @SoftDelete)
await place.destroy()
await place.undestroy()       // restore soft-deleted record
await place.reallyDestroy()   // permanent delete

// Find-or-create / upsert (see models.md for full reference)
await User.findOrCreateBy({ email }, { createWith: { name } })       // find first, create if missing
await User.createOrFindBy({ email }, { createWith: { name } })       // create first (needs unique constraint, no txn)
await User.updateOrCreateBy({ email }, { with: { name } })           // find & update, or create
await User.createOrUpdateBy({ email }, { with: { name } })           // create first (needs unique constraint, no txn)

// Query
const places = await Place.where({ style: 'cabin' }).order('createdAt', 'desc').all()
const count = await Place.where({ style: 'cabin' }).count()
const exists = await Place.where({ id: placeId }).exists()

// Operators
import { ops } from '@rvoh/dream'
await Place.where({ name: ops.like('%cabin%') }).all()
await Place.where({ sleeps: ops.greaterThan(4) }).all()
await Place.whereAny([{ style: 'cabin' }, { style: 'cottage' }]).all()

// Associations
const rooms = await place.associationQuery('rooms').where({ type: 'Bedroom' }).all()
const room = await place.createAssociation('rooms', { type: 'Bedroom' })
await place.load('rooms').execute()

// Preloading (prevents N+1) — see querying.md for full association chaining docs
// IMPORTANT: multiple string args form a CHAIN, not parallel loads
const places = await Place.preload('rooms').all()                  // preload rooms on each place
const places = await Place.preload('rooms').preload('hosts').all() // preload rooms AND hosts (parallel)
const users = await User.preload('posts', 'comments').all()        // chain: posts → comments (nested)
const places = await Place.preloadFor('summary').all()              // serializer-driven (preferred)

// Pagination
const result = await Place.cursorPaginate({ cursor: cursorString })
// Returns: { cursor: string | null, results: Place[] }

// Transactions — EVERY operation inside must use .txn(txn)
// If you forget .txn(txn), the operation runs outside the transaction
await ApplicationModel.transaction(async txn => {
  const place = await Place.txn(txn).create({ ... })
  await HostPlace.txn(txn).create({ host, place })
  const found = await Place.txn(txn).findBy({ name: 'x' }) // queries too
})

// Extracting a transaction callback into a separate method?
// Type the txn parameter as DreamTransaction<Dream> — the same type already used
// for AfterCreate/AfterUpdate hooks (see models.md). Don't reach for
// Parameters<Parameters<typeof ApplicationModel.transaction>[0]>[0] gymnastics.
import { DreamTransaction, Dream } from '@rvoh/dream'

await ApplicationModel.transaction(async txn => {
  await this.createItems(txn, place)
})

// Helper method:
private async createItems(txn: DreamTransaction<Dream>, place: Place) {
  await Room.txn(txn).create({ place, type: 'Bedroom' })
  await Room.txn(txn).create({ place, type: 'Kitchen' })
}
```

### Querying Beyond Dream

For detailed query guidance, including when to use `toKysely(...)` and when to start from the project's typed `db()` entrypoint, see [querying.md](querying.md).

The rule is simple: always prefer Dream's built in, public query and association APIs; only drop down to Kysely when Dream's supported APIs do not cover the SQL you need.

Use `range` from `@rvoh/dream/utils` directly in `where` clauses when a single column has a natural lower and/or upper bound, including `CalendarDate`, `DateTime`, `ClockTime`, and `ClockTimeTz` columns. For multi-column interval logic, named `ops` comparisons can be clearer because the boundary semantics are visible at the call site. See [querying.md — Range Predicates](querying.md#range-predicates).

## Controllers

For detailed controller patterns, see [controllers.md](controllers.md).

### Quick Reference

```typescript
import { BeforeAction, OpenAPI } from '@rvoh/psychic'

export default class V1HostPlacesController extends V1HostBaseController {
  @OpenAPI(Place, {
    status: 200,
    tags: ['places'],
    cursorPaginate: true,
    serializerKey: 'summary',
    fastJsonStringify: true,
  })
  public async index() {
    const places = await this.currentHost
      .associationQuery('places')
      .preloadFor('summary')
      .cursorPaginate({ cursor: this.castParam('cursor', 'string', { allowNull: true }) })
    this.ok(places)
  }

  @OpenAPI(Place, { status: 201, tags: ['places'], fastJsonStringify: true })
  public async create() {
    let place = await ApplicationModel.transaction(async txn => {
      const place = await Place.txn(txn).create(this.extractParams(Place, ['name', 'style', 'sleeps']))
      await HostPlace.txn(txn).create({ host: this.currentHost, place })
      return place
    })
    if (place.isPersisted) place = await place.loadFor('default').execute()
    this.created(place)
  }

  @OpenAPI(Place, { status: 200, tags: ['places'], fastJsonStringify: true })
  public async show() {
    this.ok(await this.place())
  }

  @OpenAPI(Place, { status: 204, tags: ['places'] })
  public async update() {
    await (await this.place()).update(this.extractParams(Place, ['name', 'style', 'sleeps']))
    this.noContent()
  }

  @OpenAPI({ status: 204, tags: ['places'] })
  public async destroy() {
    await (await this.place()).destroy()
    this.noContent()
  }

  private async place() {
    return await this.currentHost
      .associationQuery('places')
      .preloadFor('default')
      .findOrFail(this.castParam('id', 'string'))
  }
}
```

### Controller Key Methods

| Method | Purpose |
|--------|---------|
| `this.params` | Merged URL + body + query params |
| `this.castParam('name', 'type', opts?)` | Validate and cast a single param. Types: `uuid`, `string`, `integer`, `bigint`, `date`, `datetime`, `number`, `enum`. Array variants: `uuid[]`, `string[]`, etc. Options: `{ allowNull: true, enum: [...] }` |
| `this.extractParams(Model, allowed, opts?)` | Extract and validate request body params against a Dream model. `allowed` is an explicit positional allowlist, TS-checked and intersected with the model's `paramSafeColumns`. Options: `{ only, including, key, array }`. |
| `this.ok(data)` | 200 response |
| `this.created(data)` | 201 response |
| `this.noContent()` | 204 response |
| `this.unauthorized()` | 401 response |
| `this.forbidden()` | 403 response |
| `this.notFound()` | 404 response |
| `this.serializerPassthrough(obj)` | Pass context (e.g., locale) to serializers |
| `this.header('name')` | Read request header |
| `this.startSession(user)` / `this.endSession()` | Session management |

`extractParams` safe params include real columns, Encrypted columns, and Virtual columns. Encrypted params are typed and validated as `string` or `string | null` from the backing encrypted column's nullability; Virtual params use the `@deco.Virtual(...)` type.

Generated CRUD scaffolds hoist a shared `paramSafeColumns` const and pass it to both `extractParams` and the `@OpenAPI` `requestBody`, keeping the documented request body and runtime allowlist in lockstep from one edit point.

### @BeforeAction Pattern

```typescript
export default class AuthedController extends ApplicationController {
  protected currentUser: User

  @BeforeAction()
  protected async authenticate() {
    const userId = this.authedUserId()
    if (!userId) return this.unauthorized()
    const user = await User.find(userId)
    if (!user) return this.unauthorized()
    this.currentUser = user
  }

  // With only/except filters
  @BeforeAction({ only: ['create', 'update'] })
  protected validatePermissions() { ... }

  @BeforeAction({ except: ['index'] })
  protected loadResource() { ... }
}
```

## Serializers

For detailed serializer patterns, see [serializers.md](serializers.md).

### Quick Reference

Serializers are **function-based** (not class-based or decorator-based):

Serializer files use named exports only; do not `export default` serializer functions. Named exports keep runtime global names and OpenAPI component names explicit.

```typescript
import { DreamSerializer, ObjectSerializer } from '@rvoh/dream'

// Summary serializer (base layer)
export const PlaceSummarySerializer = (place: Place) =>
  DreamSerializer(Place, place)
    .attribute('id')
    .attribute('name')

// Default serializer (extends summary)
export const PlaceSerializer = (place: Place) =>
  PlaceSummarySerializer(place)
    .attribute('style')
    .attribute('sleeps')
    .attribute('deletedAt')
    .rendersMany('rooms', { serializerKey: 'summary' })

// Serializer with passthrough context
export const PlaceForGuestsSerializer = (place: Place, passthrough: { locale: LocalesEnum }) =>
  PlaceSummarySerializer(place)
    .customAttribute('displayStyle', () => i18n(passthrough.locale, `places.style.${place.style}`), {
      openapi: 'string',
    })
    .delegatedAttribute('currentLocalizedText', 'title', { openapi: 'string' })
    .rendersMany('rooms', { serializerKey: 'forGuests' })

// ObjectSerializer for non-Dream objects
export const BedTypeSerializer = (bedType: BedTypesEnum, passthrough: { locale: LocalesEnum }) =>
  ObjectSerializer({ bedType }, passthrough)
    .attribute('bedType', { as: 'value', openapi: { type: 'string', enum: BedTypesEnumValues } })
    .customAttribute('label', () => i18n(passthrough.locale, `rooms.Bedroom.bedTypes.${bedType}`), {
      openapi: 'string',
    })
```

For OpenAPI-visible nested computed/view-model response shapes, export every nested `ObjectSerializer` used by `rendersOne` or `rendersMany` so OpenAPI registers a named schema instead of anonymous `Unnamed` schemas. Runtime serializer global names include the serializer path, but OpenAPI component names for named exports are based on the export name; use distinct names such as a `ViewSerializer` suffix for computed serializers that share a domain noun with a Dream model serializer.

### Serializer Methods

| Method | Purpose |
|--------|---------|
| `.attribute('name')` | Include a model column |
| `.attribute('name', { as: 'alias' })` | Rename in output |
| `.attribute('name', { precision: 2 })` | Decimal precision |
| `.attribute('name', { openapi: { type: 'string', enum: [...] } })` | OpenAPI type hint |
| `.customAttribute('name', fn, { openapi: 'string' })` | Computed field |
| `.delegatedAttribute('assocName', 'field', { openapi: 'string' })` | Nested field |
| `.rendersOne('assoc')` | Single association |
| `.rendersOne('assoc', { serializerKey: 'summary' })` | With specific serializer |
| `.rendersMany('assoc')` | Array association |
| `.rendersMany('items', { serializer: CustomFn })` | Custom serializer function |

### STI Serializer Pattern

```typescript
// Base room serializer (generic)
export const RoomSummarySerializer = <T extends Room>(StiChildClass: typeof Room, room: T) =>
  DreamSerializer(StiChildClass ?? Room, room)
    .attribute('id')
    .attribute('type', { openapi: { type: 'string', enum: [(StiChildClass ?? Room).sanitizedName] } })

// STI child serializer
export const RoomBedroomSerializer = (bedroom: Bedroom) =>
  RoomSerializer(Bedroom, bedroom)
    .attribute('bedTypes')
```

## Migrations

For detailed migration patterns (column types, helpers, etc.), see [migrations.md](migrations.md).

**Migrations are always created via generators** (`g:resource`, `g:model`, `g:sti-child`, or `g:migration`), then edited as needed. Never create migration files by hand.

## Routing

```typescript
import { PsychicRouter } from '@rvoh/psychic'

export default function routes(r: PsychicRouter) {
  // Namespace groups routes and infers controller paths
  r.namespace('v1', r => {
    r.namespace('host', r => {
      // Full CRUD: index, show, create, update, destroy
      r.resources('places', r => {
        r.resources('rooms')  // Nested: /v1/host/places/:placeId/rooms
      })
      r.resources('localized-texts', { only: ['update', 'destroy'] })
    })

    r.namespace('guest', r => {
      r.resources('places', { only: ['index', 'show'] })
    })
  })

  // Simple routes
  r.get('ping', PingController, 'ping')
  r.post('login', AuthController, 'login')

  // Singular resource (no index, no :id in path)
  r.resource('profile', { only: ['show', 'update'] })

  // Collection routes (no :id)
  r.resources('items', r => {
    r.collection(r => {
      r.post('bulk-create', ItemsController, 'bulkCreate')
    })
  })
}
```

## Soft Delete

For detailed soft delete patterns, see [soft-delete.md](soft-delete.md).

### Quick Reference

Generators automatically apply `@SoftDelete()` and include a `deleted_at` column. This makes `destroy()` set `deletedAt` instead of removing the row. A default scope (`dream:SoftDelete`) automatically hides soft-deleted records from all queries. Use `--no-soft-delete` when generating if hard deletion is intentionally desired.

```typescript
import { SoftDelete } from '@rvoh/dream'

@SoftDelete()
export default class Place extends ApplicationModel {
  public deletedAt: DreamColumn<Place, 'deletedAt'>
}
```

```typescript
await place.destroy()                  // Sets deletedAt (soft delete)
await place.undestroy()                // Restores (sets deletedAt to null)
await place.reallyDestroy()            // Permanent delete

// Query multiple soft-deleted records — removeDefaultScope preserves other scopes
await Place.removeDefaultScope('dream:SoftDelete').all()

// Find a specific soft-deleted record — removeAllDefaultScopes ensures it's found
await Place.removeAllDefaultScopes().findOrFail(id)
```

**Key rules:**
- Requires a `deleted_at` datetime column (nullable)
- Use `restrict` (the default) for foreign key constraints, not `cascade`
- **Every model in a `dependent: 'destroy'` chain MUST also have `@SoftDelete()`** — otherwise those associated records will be permanently deleted
- Both `destroy()` and `undestroy()` cascade through `dependent: 'destroy'` associations
- STI children cannot use `@SoftDelete()` — it must be on the STI parent

## Default Scopes

Default scopes are query conditions automatically applied to every query on a model. For full documentation, see [models.md - Default Scopes](models.md#default-scopes).

Dream provides built-in default scopes:
- **`dream:SoftDelete`** — added by `@SoftDelete()`, hides records where `deletedAt` is not null
- **`dream:STI`** — added by `@STI()`, restricts STI child queries to their type

Models can define custom default scopes with `@deco.Scope({ default: true })`. Use `removeDefaultScope('scopeName')` when querying for a plurality (preserves other scopes). Use `removeAllDefaultScopes()` when targeting a specific record.

## Single Table Inheritance (STI)

STI allows multiple model types to share a single database table. For detailed generation, model, serializer, migration, and controller patterns see [sti.md](sti.md)

### Quick Reference

```typescript
// Parent model
export default class Room extends ApplicationModel {
  public override get table() { return 'rooms' as const }
  public type: DreamColumn<Room, 'type'>
}

// Child model
@STI(Room)
export default class Bedroom extends Room {
  public bedTypes: DreamColumn<Bedroom, 'bedTypes'>
  public override get serializers(): DreamSerializers<Bedroom> {
    return { default: 'Room/BedroomSerializer', summary: 'Room/BedroomSummarySerializer' }
  }
}
```

**STI limitations**: STI children cannot use `@SoftDelete()` (must be on parent) and cannot use `@ReplicaSafe()`.

**Creating STI instances in controllers** — per Critical Rule 15 (exhaustive switch on closed enums), the `create` action switches on `type` with a `_never` default:

```typescript
const roomType = this.castParam('type', 'string', { enum: RoomTypesEnumValues })
let room: Room
switch (roomType) {
  case 'Bathroom': room = await Bathroom.create({ place, ...params }); break
  case 'Bedroom':  room = await Bedroom.create({ place, ...params }); break
  case 'Kitchen':  room = await Kitchen.create({ place, ...params }); break
  default: { const _never: never = roomType; throw new Error(`Unhandled: ${_never}`) }
}
```

## Associations

For the full reference including all options and option tables, see [models.md](models.md).

### Types

```typescript
// BelongsTo (FK on this model)
@deco.BelongsTo('User')
public user: User
public userId: DreamColumn<Post, 'userId'>

// BelongsTo optional
@deco.BelongsTo('User', { optional: true })
public approver: User | null

// HasMany
@deco.HasMany('Post', { dependent: 'destroy' })
public posts: Post[]

// HasOne
@deco.HasOne('Profile')
public profile: Profile

// Through (many-to-many via join table)
@deco.HasMany('Host', { through: 'hostPlaces' })
public hosts: Host[]

// Polymorphic HasMany
@deco.HasMany('LocalizedText', { polymorphic: true, on: 'localizableId', dependent: 'destroy' })
public localizedTexts: LocalizedText[]

// Polymorphic BelongsTo
@deco.BelongsTo(['Host', 'Place', 'Room'], { polymorphic: true, on: 'localizableId' })
public localizable: Host | Place | Room

// With conditions (and, andAny, andNot)
@deco.HasMany('Post', { and: { published: true } })
public publishedPosts: Post[]

@deco.HasMany('Post', { andAny: [{ featured: true }, { published: true }] })
public visiblePosts: Post[]

@deco.HasMany('Post', { andNot: { archived: true } })
public activePosts: Post[]

// selfAnd — join where associated column matches a column on THIS model
// Format: { associatedColumn: 'thisModelColumn' }
// Links to DailyChallenge by position instead of FK, allowing re-shuffling
@deco.HasOne('DailyChallenge', {
  on: 'position',
  selfAnd: { position: 'currentPosition' },
})
public currentChallenge: DailyChallenge

// selfAndNot — exclude where columns match (e.g., siblings = parent's children minus self)
@deco.HasMany('TreeNode', {
  through: 'parent', source: 'children',
  selfAndNot: { id: 'id' },
})
public siblings: TreeNode[]

// DreamConst.passthrough — value supplied at query time via .passthrough({ locale: 'en-US' })
@deco.HasOne('LocalizedText', {
  polymorphic: true, on: 'localizableId',
  and: { locale: DreamConst.passthrough },
})
public currentLocalizedText: LocalizedText

// DreamConst.required — value supplied via { and: { category: value } } in the loading chain
@deco.HasOne('Preference', {
  and: { category: DreamConst.required },
})
public preference: Preference

// With order and distinct
@deco.HasMany('Tag', { through: 'postTags', distinct: true, order: 'name' })
public tags: Tag[]

// Skip default scopes when loading
@deco.HasMany('Post', { withoutDefaultScopes: ['dream:SoftDelete'] })
public allPosts: Post[]

// Override primary key used for join
@deco.BelongsTo('Widget', { primaryKeyOverride: 'legacyId' })
public widget: Widget
```

### Association Option Summary

**BelongsTo**: `on`, `optional`, `polymorphic`, `primaryKeyOverride`, `withoutDefaultScopes`

**HasMany / HasOne**: `and`, `andAny`, `andNot`, `dependent`, `on`, `order` (HasMany only), `distinct` (HasMany only), `polymorphic`, `primaryKeyOverride`, `selfAnd`, `selfAndNot`, `through`, `source`, `withoutDefaultScopes`

**Through associations** cannot use: `dependent`, `primaryKeyOverride`, `withoutDefaultScopes`, `on`, or `polymorphic`.

## Internationalization (i18n)

For detailed i18n patterns, see [i18n.md](i18n.md). Psychic supports two complementary patterns:

- **Code-driven i18n** - Translate enum values and static labels using `I18nProvider` from `@rvoh/psychic/system` with locale files in `src/conf/locales/`
- **Data-driven i18n** - Translate user-generated content using a polymorphic `LocalizedText` model with `DreamConst.passthrough` for locale-conditioned associations

Both use the `Accept-Language` request header, passed to serializers via `serializerPassthrough({ locale })`.

## Background Workers

For detailed worker patterns (services, models, scheduling, configuration, testing), see [workers.md](workers.md).

### Quick Reference

Backgrounded services use a two-method pattern: a public entry method that exposes an expressive API to callers, and a private implementation method (prefixed with `_`) that does the work. This encapsulates backgrounding as an implementation detail — callers don't need to know the work is backgrounded. **Never pass model data as arguments** — pass only IDs and look up the record in the implementation method. Passing model data bloats Redis, loses type information when serialized to JSON, and creates immediately stale snapshots.

```typescript
import ApplicationBackgroundedService from './ApplicationBackgroundedService.js'

export class NotificationService extends ApplicationBackgroundedService {
  // Public entry point — called from application code
  public static async sendWelcome(user: User) {
    await this.background('_sendWelcome', user.id)
  }

  // Private implementation — called by the worker
  public static async _sendWelcome(userId: string) {
    // Use find (not findOrFail) — the record may have been deleted since the job was queued
    const user = await User.find(userId)
    // Exit if target model no longer exists
    if (!user) return
    // Welcome notication logic
  }

  // Optional: configure priority/workstream
  public static override get backgroundJobConfig() {
    return { priority: 'urgent' }
  }
}

// Called from application code (e.g., controller or model hook):
await NotificationService.sendWelcome(user)

// IMPORTANT: When backgrounding from a Dream model lifecycle hook, the `Commit` variant of the hook **MUST** be used, e.g., AfterCreateCommit **NOT** AfterCreate:
@deco.AfterCreateCommit()
public async notifyCreation(this: Place) {
  await NotificationService.placeCreated(this)
}
```

## Websockets

For detailed websocket patterns, see [websockets.md](websockets.md).

### Quick Reference

```typescript
import { Ws } from '@rvoh/psychic-websockets'

// Define typed channels
const ws = new Ws(['/notifications/new', '/data/update'] as const)

// Emit to user
await ws.emit(userId, '/notifications/new', { message: 'Hello' })

// Register on connection (in conf/initializers/websockets.ts)
wsApp.on('ws:start', io => {
  io.of('/').on('connection', async socket => {
    const user = await User.find(extractUserId(socket))
    if (user) await Ws.register(socket, user.id)
  })
})
```

## Testing

For detailed testing patterns, see [testing.md](testing.md).

### Factory Pattern

```typescript
// spec/factories/PlaceFactory.ts
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

### Controller Spec Pattern

```typescript
describe('V1/Host/PlacesController', () => {
  let request: SpecRequestType
  let user: User
  let host: Host

  beforeEach(async () => {
    user = await createUser()
    host = await createHost({ user })
    request = await session(user)
  })

  describe('POST create', () => {
    it('creates a Place', async () => {
      const { body } = await request.post('/v1/host/places', 201, {
        data: { name: 'Cozy Cabin', style: 'cabin', sleeps: 4 },
      })
      const place = await host.associationQuery('places').firstOrFail()
      expect(place.name).toEqual('Cozy Cabin')
      expect(body).toEqual(expect.objectContaining({ id: place.id }))
    })
  })

  describe('GET index', () => {
    it('returns only this host places', async () => {
      const place = await createPlace()
      await createHostPlace({ host, place })
      const otherPlace = await createPlace()  // Not associated

      const { body } = await request.get('/v1/host/places', 200)
      expect(body.results).toHaveLength(1)
      expect(body.results[0].id).toEqual(place.id)
    })
  })
})
```

### Model Spec Pattern

```typescript
describe('Place', () => {
  describe('associations', () => {
    it('has many rooms', async () => {
      const place = await createPlace()
      const room = await createBedroom({ place })
      expect(await place.associationQuery('rooms').all()).toMatchDreamModels([room])
    })
  })

  describe('soft delete', () => {
    it('soft deletes and cascades', async () => {
      const place = await createPlace()
      await place.destroy()
      expect(await Place.where({ id: place.id }).exists()).toBe(false)
      expect(await Place.where({ id: place.id }).removeAllDefaultScopes().exists()).toBe(true)
    })
  })
})
```

## Naming Conventions

| Context | Convention | Example |
|---------|-----------|---------|
| Database columns | snake_case | `created_at`, `user_id` |
| Model properties | camelCase | `createdAt`, `userId` |
| Model classes | PascalCase | `HostPlace`, `LocalizedText` |
| Controller files | PascalCase | `PlacesController.ts` |
| Serializer files | PascalCase | `PlaceSerializer.ts` |
| Migration files | timestamp-kebab | `1773151915966-create-place.ts` |
| Route paths | kebab-case | `localized-texts`, `admin-users` |
| Generator columns | snake_case | `name:string`, `User:belongs_to` |
| Enum types (DB) | snake_case + `_enum` | `place_styles_enum` |
| Enum values | snake_case | `lean_to`, `bath_and_shower` |
| STI type values | PascalCase (MUST match STI child class names) | `Bedroom`, `LivingRoom` |
| `date` columns | suffix `On` | `occurred_on`, `started_on`, `born_on` |
| `datetime` columns | suffix `At` | `occurred_at`, `started_at`, `deleted_at` |

## OpenAPI Integration

Every controller action should have an `@OpenAPI` decorator:

```typescript
@OpenAPI(Place, {
  status: 200,                          // HTTP status code
  tags: ['places'],                     // OpenAPI grouping
  description: 'List all places',       // Endpoint description
  serializerKey: 'summary',             // Which serializer variant
  many: true,                           // Response is array
  cursorPaginate: true,                 // Cursor pagination wrapper
  fastJsonStringify: true,              // Performance optimization
  query: {                              // Query parameters
    search: { required: false, schema: 'string' },
  },
  requestBody: { including: ['name'] }, // Request body fields
})
```

Automatic error handling:
- `castParam` invalid -> 400
- `findOrFail` not found -> 404
- Validation-layer failure -> 400

## Deploying

For deployment guidance (runtime model, build output, health checks, required environment variables, TLS, and debugging), see [deploying.md](deploying.md).

## Troubleshooting Migrations

**"Corrupted migrations" error**: When switching between branches with different migrations, `pnpm psy db:migrate` fails with "corrupted migrations" errors. The fix is `pnpm psy db:reset`.
