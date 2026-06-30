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

**Ecosystem versions & staleness policy.** This skill is written against `@rvoh/dream` 2.17.x, `@rvoh/psychic` 3.8.x, `@rvoh/psychic-workers` 2.3.x, `@rvoh/psychic-websockets` 3.3.x, and `@rvoh/psychic-spec-helpers` 3.2.x. Features and generator behavior described here assume versions at or above these. **Stay current.** If something documented in this skill fails — a generator flag is unrecognized, shorthand produces malformed output (e.g. an `@alias` passing through literally into identifiers), an API is missing — the first corrective action is to update the out-of-date `@rvoh/*` packages, not to work around the skill. Minor and patch bumps within these majors are low-risk and cheap to apply; treat keeping these packages up to date as the default. This skill deliberately does **not** annotate which version each individual feature landed in — assume current, and upgrade if reality disagrees with the skill.

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
8. **Sources of truth** (priority order): TSDocs > `pnpm psy <command> --help` > psychic-skill.
9. **BDD approach**: Write failing spec first, then implement. Generated code is the only exception (generators create scaffolding for specs and implementation simultaneously).
10. **Run `pnpm psy sync`** after changing associations, serializers, OpenAPI decorators, or routes.
11. **Only use comments to explain "why", not "what"** - prefer expressive code over comments. TSDoc comments explaining "how" or "when" to use a function/method/class are welcome.
12. **Use Dream's built-in utilities** (`@rvoh/dream/utils`) instead of lodash or hand-rolled equivalents. See [utils.md](utils.md) for the full list.
13. **NEVER use `process.env` directly** - always use `AppEnv` (`api/src/conf/AppEnv.ts`) to access environment variables. `AppEnv` validates that required variables are present at boot time with clear error messages, and decouples the application from the container environment (enabling secret injection from external sources like AWS Secrets Manager). Variables that are only present in some environments (e.g., a deploy-only credential blob absent in local dev) use the `{ optional: true }` overload, which returns `string | undefined` instead of throwing — never reach for `process.env` to avoid the throw. Example: `const credentialConfig = AppEnv.string('GOOGLE_CALENDAR_CREDENTIAL_CONFIG', { optional: true })`; gate the deployed-vs-local path with `if (credentialConfig) { ... } else { ... }`. This rule governs reading *application config*. It does not cover a dev-only launcher whose job is to *compose* the environment handed to spawned child processes — reading `process.env` to spread it into a `spawn(..., { env })` child is correct there, since no `AppEnv` accessor represents "the whole environment to forward" and those values are child-process plumbing, not validated app config.
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
    - Model request bodies use `requestBody: { params: [...] }` / `{ including: [...] }`, even when the action must use `castParam` instead of `extractParams` for STI dispatch or custom validation. `params` is the OpenAPI request-body narrowing key; older code may still show `only`, but new docs and scaffolds should use `params`. **These must be literal arrays mirroring the action's `extractParams` allowlist — never backfill them by calling `Model.paramSafeColumnsOrFallback()` / `Model.paramSafeColumns` / `Model.columns()`, which dumps the model's entire writable column surface into the spec and re-creates the implicit include-all default. See [controllers.md](controllers.md#requestbody-shorthand--what-each-option-is-for).**
    - Model responses use `@OpenAPI(Model, { serializerKey })` and serializers.
    - Computed / view-model responses use an `ObjectSerializer` passed to `@OpenAPI(SerializerFn, { status })`; nested computed objects become nested `ObjectSerializer`s. If the ObjectSerializer does not exist yet, create it.
    - Hand-written JSON Schema is only for genuinely ad hoc inputs or outputs that cannot sensibly be represented by a Dream model, serializer, or newly-created ObjectSerializer. Do not use "there is no serializer yet" as a reason to hand-write `responses`. Keeping OpenAPI attached to serializers gives TypeScript a single implementation surface for the returned data and documented schema instead of letting a plain object and a duplicated schema drift independently.

22. **The controller directory tree IS the auth architecture; a surface that loosens auth is its own top-level namespace.** Authed client endpoints live under `V1/`; any surface that loosens auth — public/maybe-authed, webhooks, partner API — is its own top-level namespace with the version nested inside (`Visitor/V1/`, `Webhooks/V1/`, `Api/V1/`), never `V1/Visitor/`. `Admin/` and `Internal/` are separate top-level surfaces with their own `AuthedController`. Generate the surface, then reparent its top-level namespace base controller once. Auth is enforced by ancestry, so nesting a looser surface inside an authed one changes auth deep in the tree — the placement *is* the enforcement, there is no other correct way to express it. Full rules: [controllers.md](controllers.md#controller-hierarchy).

## Creating a New Psychic Application

If you are not already inside a Psychic project, use `create-psychic` to scaffold one:

```bash
npx @rvoh/create-psychic new <app-name> [options]
```

**Every option must be provided**, or `create-psychic` will enter interactive mode to prompt for unanswered questions. Boolean options can be negated with `--no-` (e.g., `--no-workers`).

| Option | Description |
|--------|-------------|
| `--package-manager <pm>` | `pnpm`, `yarn`, or `npm` |
| `--primary-key-type <type>` | `uuid7`, `uuid4`, `bigint`, or `integer` |
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

## Generators

- **Never hand-roll what a generator can make.** Models, resources, migrations, and STI children are always created with `pnpm psy g:*` and then edited — never written from scratch.
- **Generator order:** `g:resource` (default for almost any model) → `g:sti-child` (STI children) → `g:model` (only for models never exposed via any API) → `g:migration` (schema change without a new model).
- **Before you run any generator, read [generators.md](generators.md) and run `pnpm psy <command> --help`.** It carries the argument contract, the mandatory `--owning-model` rule for nested resources, what each generator scaffolds by default, and the post-generate workflow. Reaching for a generator without reading it produces subtly wrong scaffolding that is expensive to unwind.

## Models

A Dream model is the source of truth for one table — its columns, associations, validations, hooks, scopes, and the serializers it renders through. Reach here whenever you define or change a model, query through it, or write inside a transaction.

- **Prefer Dream's public query and association APIs**; drop to Kysely only for SQL they don't cover.
- **Inside a transaction, bind every operation with `.txn(txn)`** — creates, updates, queries, and association calls alike. Miss it and that operation runs outside the transaction and won't roll back.
- **Name a model by what it *is*, not by its route or its owner** — a nested route plus `--owning-model` does not imply a `Parent/Child` namespace.

**Before you add an association, write a hook or validation, run a multi-step or preloaded query, or open a transaction, read [models.md](models.md)** — and [querying.md](querying.md) for queries that reach past Dream's public API. It owns the column and decorator setup, the full association reference (including the required-`BelongsTo` two-way contract that throws `MissingRequiredBelongsToAssociation` at runtime when violated, `selfAnd`/`selfAndNot`, polymorphism, and `through` restrictions), hooks, scopes, find-or-create/upsert, and the transaction rules. Guessing association options or transaction binding from memory produces code that compiles and then fails or corrupts data at runtime.

## Controllers

A Psychic controller authenticates a request, pulls and validates params, does the work through Dream models, and renders a response. Reach here whenever you add or change an endpoint.

- **The controller directory tree *is* the auth architecture, and auth only ever gets stricter downhill** — never introduce a looser authentication pattern deeper in a branch.
- **A surface that loosens auth is its own top-level namespace, version nested inside** (`Visitor/V1/`, `Webhooks/V1/`, `Api/V1/`) — never `V1/Visitor/`. `V1/` is the authed client surface; `Admin/` and `Internal/` are separate top-level surfaces each with their own `AuthedController`.
- **Generate controllers; never hand-roll them.** `g:resource` / `g:controller` build the namespace base-controller chain that shared auth lives on. For an intentionally unauthenticated surface, generate normally and then re-parent that namespace's base to `UnauthedController`.
- **`extractParams` is an explicit, per-action allowlist**, intersected with the model's `paramSafeColumns`. Foreign keys, the STI `type`, and timestamps are always stripped — pull those explicitly via `castParam`.

**Before you write an action, an `@OpenAPI` decorator, a `@BeforeAction`, or any param handling, read [controllers.md](controllers.md).** It owns the hierarchy/auth rules, the CRUD patterns, the full `castParam`/`extractParams`/`requestBody` contracts, response methods, cookie/session handling, and logging. Hand-writing a controller skips the namespace base chain where auth is enforced — the most expensive mistake to unwind in this layer.

## Serializers

Serializers turn Dream models (and plain view-model objects) into JSON responses and generate the matching OpenAPI schema in one place. Reach here whenever a response shape changes.

- **Serializers are function-based and use named exports only** — never class-based, never `export default`. Named exports keep runtime global names and OpenAPI component names explicit.
- **`serializerKey` does not cascade.** A nested `rendersOne`/`rendersMany` defaults to the associated model's `'default'` serializer unless you pass the key explicitly at every level.
- **Serializers are synchronous and cannot query** — preload everything first with `preloadFor`/`loadFor`, not a hand-built `preload` chain that drifts and throws `NonLoadedAssociation`.

**Before you write or change a serializer, read [serializers.md](serializers.md)** — and [sti.md](sti.md) for STI base/child serializers. It owns the composition pattern, every method (`attribute`/`customAttribute`/`delegatedAttribute`/`rendersOne`/`rendersMany`), flattening and attribute-shadowing, passthrough context, and `ObjectSerializer` for non-Dream shapes. A hand-written STI serializer that drops the `type`/`StiChildClass` shape silently collapses every child to one schema — correct in a unit spec, broken over HTTP. Run `pnpm psy sync` after changing any serializer so the OpenAPI specs and generated clients update.

## Migrations

Migrations are Kysely-based schema changes — the only way the database shape changes (columns, tables, enums, indexes, foreign keys). Reach here whenever you add, alter, or remove one.

- **Always generate, never hand-write.** Migrations come from `g:resource`, `g:model`, `g:sti-child`, or `g:migration`, then are edited as needed — never created from scratch.
- **Adding a `NOT NULL` column to a table that already has rows needs a default or a backfill step**, or the migration fails against the existing data.

**Before you write or edit a migration, read [migrations.md](migrations.md).** It owns the column-type DSL, `DreamMigrationHelpers` (prefer these over raw Kysely calls), foreign keys and indexes, the soft-delete column, the new-transaction escape hatch, and enum handling — including the two-migration pattern required to rename an in-use enum value (PostgreSQL can't add and use an enum value in one transaction).

## Routing

```typescript
import { PsychicRouter } from '@rvoh/psychic'

export default function routes(r: PsychicRouter) {
  // Namespace groups routes and infers controller paths.
  // Authed client API — everything under v1/ is authenticated.
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

  // A surface that LOOSENS auth is its OWN top-level namespace, version nested inside —
  // never under v1/. The directory tree is the auth architecture (see controllers.md).
  r.namespace('webhooks', r => {     // unauthed external callbacks: /webhooks/v1/zoom
    r.namespace('v1', r => {
      r.post('zoom', WebhooksV1ZoomController, 'create')
    })
  })
  r.namespace('api', r => {          // server-to-server partner API: /api/v1/reservations
    r.namespace('v1', r => {
      r.resources('reservations', { only: ['index', 'show', 'create'] })
    })
  })
  // The maybe-authed Visitor surface lives top-level too (Visitor/V1); it can map to a
  // clean /v1 URL via an explicit `controller:` reference — see controllers.md.

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

  // Member route (custom action on a single record). A route declared directly in
  // the resources callback — outside `collection` — is member-scoped: Psychic
  // prepends `:id`. There is no `r.member`; use the existing verbs (r.get/r.post/…)
  // and read the id in the action with `this.castParam('id', 'uuid')` (cast the id to
  // its primary-key type — `uuid`, `bigint`, or `integer` — never `string`).
  r.resources('bookings', r => {
    r.post('cancel', BookingsController, 'cancel')   // member-scoped: POST /bookings/:id/cancel
  })
}
```

## Soft Delete

`@SoftDelete()` makes `destroy()` set `deletedAt` instead of removing the row, and a `dream:SoftDelete` default scope hides those rows from every query. Generators apply it by default. Reach here when a model needs "removed but recoverable / auditable" semantics — and don't hand-roll a `removed` / `deactivatedAt` flag, which fights the lifecycle.

- **Every model in a `dependent: 'destroy'` chain must also have `@SoftDelete()`** — otherwise destroying the parent permanently deletes those children. Dream does not check this for you.
- **STI children never carry `@SoftDelete()`** — it lives on the STI parent and all children inherit it.

**Before you add soft delete to an existing model, query soft-deleted rows, or build a `dependent: 'destroy'` chain, read [soft-delete.md](soft-delete.md).** It owns the setup, the `restrict`-not-`cascade` FK rule, `undestroy` / `reallyDestroy`, and the `removeDefaultScope` vs `removeAllDefaultScopes` distinction.

## Default Scopes

Default scopes are conditions automatically applied to every query on a model. Two are built in — **`dream:SoftDelete`** (added by `@SoftDelete()`, hides `deletedAt` rows) and **`dream:STI`** (added by `@STI()`, restricts a child query to its type); define your own with `@deco.Scope({ default: true })`.

- **Bypass one with `removeDefaultScope('name')` when querying a plurality** (it preserves other scopes) and **`removeAllDefaultScopes()` when targeting one specific record.**

Full reference: [models.md — Default Scopes](models.md#default-scopes).

## Single Table Inheritance (STI)

STI stores several model types in one table, discriminated by a `type` enum column. Reach for it when a `type` column's values carry different behavior, validations, serializers, or child-specific columns.

- **STI `type` values must exactly match the child class names** (`Bedroom`, `LivingRoom`) — the enum and the classes are one namespace.
- **Generate, never hand-roll:** `g:resource --sti-base-serializer` for the parent, `g:sti-child` per child. They emit the check constraints, per-child serializers, and type-discriminated OpenAPI that are painful to retrofit.
- **Creating an STI child dispatches on `type` through an exhaustive `switch` with a `_never` default** (Critical Rule 15) — `if/else` chains silently no-op when a type is added.

**Before you generate an STI parent or child, write an STI serializer, or build the create action, read [sti.md](sti.md).** It owns the full generation workflow, the generic base-serializer shape (drop the `StiChildClass` / `type` parts and discrimination breaks silently over HTTP), the migration / check-constraint patterns, and the controller switch. STI children also cannot use `@SoftDelete()`, `@ReplicaSafe()`, `@Sortable()`, or define their own associations — all live on the parent.

## Associations

Associations declare how models relate — `BelongsTo`, `HasMany`, `HasOne`, `through` for many-to-many, and polymorphic variants. Reach here whenever you wire two models together.

- **A `BelongsTo` lives on the model holding the foreign key, and you must declare the FK column too** (`public userId: DreamColumn<...>` beside the `@deco.BelongsTo('User')`).
- **A non-optional `BelongsTo` is a non-nullable contract on both ends** — it can't be conditionally loaded (compile error) and throws `MissingRequiredBelongsToAssociation` if an internal mechanism nulls it. The fix is a model change (`dependent: 'destroy'` on the inverse, or `optional: true`), not a looser serializer.
- **`through` associations cannot use** `dependent`, `primaryKeyOverride`, `withoutDefaultScopes`, `on`, or `polymorphic`.

**Before you add or change an association, read the full reference in [models.md](models.md)** — every option, the conditions (`and` / `andAny` / `andNot`), `selfAnd` / `selfAndNot`, `DreamConst.passthrough` / `required`, and polymorphism. Run `pnpm psy sync` after any association change.

## Internationalization (i18n)

Psychic supports two complementary translation patterns: **code-driven** (enum values and static labels via `I18nProvider` and locale files in `src/conf/locales/`) and **data-driven** (user-generated content via a polymorphic `LocalizedText` model with `DreamConst.passthrough`). Both read the `Accept-Language` header and flow to serializers via `serializerPassthrough({ locale })`.

**Before you add either pattern, read [i18n.md](i18n.md).**

## Background Workers

Background jobs (BullMQ / Redis) offload slow, costly, or failure-prone work off the request path, with automatic retry. Services are the standard surface; models can background their own methods. Reach here whenever work should not block a response.

- **Never pass model data as job arguments — pass IDs only** and re-look-up inside the implementation. Model payloads bloat Redis, lose type information through JSON, and go stale.
- **Use `find` (not `findOrFail`) in implementations and return early on null** — the record may have been deleted before the worker runs, and `findOrFail` would retry for ~6 days.
- **Enqueue only after the transaction commits** — any lifecycle hook that queues background work must use a `Commit` variant (`@deco.AfterCreateCommit`, `@deco.AfterUpdateCommit`, `@deco.AfterSaveCommit`). This applies whether the hook calls a backgrounded service or backgrounds a model method, and whether the worker needs a newly-created row or newly-updated persisted data. Never call `background(...)` from inside an open `txn`, or the worker races the commit and silently strands the record.

**Before you write a backgrounded service, a scheduled job, or a model hook that enqueues work, read [workers.md](workers.md).** It owns the two-method service pattern, priorities / workstreams, the scheduled-vs-backgrounded split, large-set fan-out, and why a `try/catch` inside a job (which fakes success and kills retry) is almost always a bug.

## Websockets

Psychic Websockets gives real-time push over Socket.IO with Redis pub/sub. Typed channels are declared with `Ws`, messages emit to a user id, and connections register a socket to a user. Reach here whenever the app pushes to clients.

- **`PsychicAppWebsockets` must initialize in every process that calls `Ws.emit()`** — web, worker, and ws server alike. Skipping it in any of them throws an obscure `cachePsychicAppWebsockets` error that looks like a framework bug.
- **Set `transports: ['websocket']` on the client** — Socket.IO's long-polling default causes subtle stale-data failures.

**Before you wire up channels, connection auth, or emit from a worker, read [websockets.md](websockets.md).** It owns the initializer / auth scaffolding, the origin allowlist, and the worker-emit pattern.

## Testing

Tests use Vitest against real database records — never mocks of Dream internals (Critical Rule 5). Unit specs cover models and controllers; feature specs cover end-to-end flows; factories build test data. Reach here whenever you add or change behavior.

- **Write the failing spec first** (Critical Rule 9), then implement. Generated scaffolding is the only exception.
- **Every bug is a missing spec** — write the regression spec *before* committing the fix, even for a one-line "couldn't regress" change.
- **Create real records with factories**, not stubs.

**Before you write a factory, a model / controller spec, or a feature spec, read [testing.md](testing.md).** It owns the factory pattern (including STI and side-effect-hook factories), the `session(...)` auth helper, the matchers (`toMatchDreamModels`, action / expectation matchers), spec organization, and worker / feature-spec specifics.

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

Psychic derives the OpenAPI spec from database column types, serializers, and routes; you declare only the remainder. Reach here whenever you document an endpoint or customize the spec.

- **Never hand-write a schema Psychic can derive** (Critical Rule 21) — model request bodies use `requestBody: { params }` / `{ including }`, model responses use `@OpenAPI(Model, { serializerKey })`, computed responses use an `ObjectSerializer`. Hand-written JSON Schema is only for genuinely ad hoc shapes.
- **Automatic error handling** maps `castParam` / validation failures → 400 and `findOrFail` misses → 404; don't document or throw these by hand.

**Start with [openapi.md](openapi.md)** for the derivation model ([How Psychic builds the spec](openapi.md#how-psychic-builds-the-spec)) and the spec-wide `psy.set('openapi', ...)` configuration in `conf/app.ts` ([Conf-level configuration](openapi.md#conf-level-configuration) — namespaces, default headers/responses, security schemes, validation, type sync). The per-action `@OpenAPI` decorator and `requestBody` shorthand live in [controllers.md](controllers.md); response schema comes from serializers, see [serializers.md](serializers.md). Run `pnpm psy sync` after any change so the specs and generated clients update.

## Deploying

For deployment guidance (runtime model, build output, health checks, required environment variables, TLS, and debugging), see [deploying.md](deploying.md).

## Troubleshooting Migrations

**"Corrupted migrations" error**: When switching between branches with different migrations, `pnpm psy db:migrate` fails with "corrupted migrations" errors. The fix is `pnpm psy db:reset`.
