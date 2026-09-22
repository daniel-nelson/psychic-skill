---
name: psychic-skill
description: >
  Builds, explains, and debugs applications on Dream ORM and the Psychic web framework.
  Covers models, associations, validations, hooks, scopes, serializers, controllers, routing,
  migrations, background workers, websockets, OpenAPI, i18n, the Dream console, deploying, testing, and code generation.
  Use this skill whenever code imports '@rvoh/dream' or a '@rvoh/dream/*' sub-path, '@rvoh/psychic',
  '@rvoh/psychic-workers', '@rvoh/psychic-websockets' or '@rvoh/psychic-spec-helpers';
  whenever a project has Dream models, Psychic controllers or 'psy' commands;
  and whenever the ask arrives in plain words — "add an endpoint", "generate a resource",
  "why isn't this field in the response", "this migration won't run", "my spec is failing"
  — even when Dream, Psychic or 'psy' is never named.
  Applies equally to writing code, reading an unfamiliar Psychic codebase, inspecting or querying data,
  and working out why a query returns the records it does.
user-invocable: false
allowed-tools: Read, Grep, Glob, Bash, Edit, Write
---

**If `## Troubleshooting Migrations` is missing below, this copy was truncated by compaction — re-read this skill's `SKILL.md` in full before acting on it.**

# Dream ORM & Psychic Web Framework Development Guide

**Dream** is a TypeScript Active Record ORM and **Psychic** a batteries-included web framework on Koa, both under the `@rvoh` npm scope.

**Update check**: !`for d in "${CLAUDE_SKILL_DIR:-}" "$HOME/.agents/skills/psychic-skill" "$HOME/.claude/skills/psychic-skill" ".agents/skills/psychic-skill" ".claude/skills/psychic-skill"; do [ -n "$d" ] && [ -x "$d/bin/psychic-skill-update-check" ] && "$d/bin/psychic-skill-update-check" 2>/dev/null && exit 0; done`
On `UPGRADE_AVAILABLE <old> <new>`, follow the inline upgrade flow in `/psychic-update-skill` — a second indented line marks which copies are `(behind)`; relay it. On `JUST_UPGRADED <old> <new>`, tell the user "psychic-skill upgraded from v{old} to v{new}!" and continue.

All CLI commands run through the project's package manager. Examples here are written `pnpm psy ...`; substitute the project's own (`yarn psy ...`, `npm run psy ...`, `bun run psy ...`) — Critical Rule 2 says how to detect it.

**Examples** use BearBnB, an AirBnB-for-bears demo app (https://github.com/daniel-nelson/bearbnb), where **Guest** and **Host** are application roles, not "visitor" (unauthenticated) or "server".

**Ecosystem versions & staleness policy.** Written against `@rvoh/dream` 2.32.x, `@rvoh/psychic` 3.15.x, `@rvoh/psychic-workers` 2.7.x, `@rvoh/psychic-websockets` 3.5.x, `@rvoh/psychic-spec-helpers` 3.4.x. **Stay current:** when something here fails — an unrecognized generator flag, malformed shorthand, a missing API — update the out-of-date `@rvoh/*` packages rather than working around the skill. No feature is annotated with the version it landed in: assume current, upgrade if reality disagrees. A scoped `pnpm up -L "@rvoh/*"` leaves peers behind, so resolve every peer requirement it introduces (`kysely`, `kysely-codegen`).

## Reference Map

**Open the named file before you write code in its area — not after something breaks. Read the whole file.** Copying a neighboring file in the app is not research: it cannot tell you a helper exists.

- **[generators.md](generators.md)** — before any generator. Owns the decision tree, `g:resource`'s argument contract, `--owning-model`, the post-generate workflow.
- **[models.md](models.md)** — before an association, hook, validation, transaction, or a new variant of an existing concept. Owns columns, decorators, associations, hooks, validations, scopes, batching, upserts, date/time, `.txn(txn)`.
- **[querying.md](querying.md)** — when a query reaches past Dream's public API or returns unexpected rows. Owns the query inventory, predicates, ordering, association chaining, preloading, `toKysely`.
- **[controllers.md](controllers.md)** — before an action, `@OpenAPI` decorator, `@BeforeAction`, or param handling. Owns hierarchy/auth, routes, CRUD, params, responses, error handling, cookies, logging.
- **[serializers.md](serializers.md)** — before writing or changing a serializer. Owns the named-export function pattern, every method, flattening, `serializerKey`, passthrough, `preloadFor`, STI and `ObjectSerializer`.
- **[migrations.md](migrations.md)** — before writing or editing a migration. Owns the column-type DSL, `DreamMigrationHelpers`, keys, indexes, polymorphic/STI/soft-delete columns, enums.
- **[sti.md](sti.md)** — before generating an STI parent or child, writing an STI serializer, or building the create action. Owns the generation workflow, the base-serializer shape, check constraints, the controller `switch`.
- **[soft-delete.md](soft-delete.md)** — before adding `@SoftDelete()`, querying soft-deleted rows, or a `dependent: 'destroy'` chain. Owns setup, the `restrict`-not-`cascade` FK rule, `undestroy`/`reallyDestroy`.
- **[locking.md](locking.md)** — before a claim: a write whose new value depends on a value just read. Owns `{ lock: true }` and its forms, what a lock costs, why a table lock is not the next step up.
- **[workers.md](workers.md)** — before a backgrounded service, a scheduled job, or a hook that enqueues work. Owns the service pattern, the `AfterCommit` requirement, ID-only arguments, priorities, workstreams, fan-out, retry.
- **[websockets.md](websockets.md)** — before channels, connection auth, or emitting from a worker. Owns the `PsychicAppWebsockets` initializer, typed `Ws` channels, auth, the origin allowlist, worker emits.
- **[openapi.md](openapi.md)** — documenting an endpoint or customizing the spec. Owns spec derivation, `psy.set('openapi', ...)`, typed clients, custom error responses.
- **[testing.md](testing.md)** — before a factory, a model or controller spec, or a feature spec. Owns the factory pattern, `session(...)`, the matchers, spec organization, worker and feature specifics.
- **[i18n.md](i18n.md)** — before translating anything. Owns code-driven labels via `I18nProvider` and `src/conf/locales/`, data-driven content via the polymorphic `LocalizedText` model, locale passthrough.
- **[console.md](console.md)** — before inspecting data by hand or running a one-off script. Owns the `NODE_ENV` defaults every `psy` command inherits, the Dream console and its auto-imports, dev-database scripts.
- **[deploying.md](deploying.md)** — deploying, configuring an environment, or debugging a container. Owns the runtime model, health checks, the `AppEnv` contract, TLS, read replicas, production migrations.
- **[utils.md](utils.md)** — before reaching for lodash or hand-rolling a helper. Owns the `@rvoh/dream/utils` inventory: case conversion, array and object helpers, `range`, `isEmpty`, `cloneDeepSafe`, `sanitizeString`, `Encrypt`.

## Critical Rules

**If something is failing unexpectedly, re-read this skill before debugging.** Most common errors — type mismatches, missing associations, validation failures, generator syntax, migrations — are already documented here with solutions.

1. **Read the project's `AGENTS.md` or `CLAUDE.md` before doing any work.** They carry project-specific conventions that override this skill's patterns.
2. **Detect the project's package manager before running any command; `pnpm` in this skill is a stand-in.** Check `package.json`'s `"packageManager"` field if present (authoritative), otherwise the lockfile — `pnpm-lock.yaml` → `pnpm psy ...`, `yarn.lock` → `yarn psy ...`, `package-lock.json` → `npm run psy ...`, `bun.lock`/`bun.lockb` → `bun run psy ...`. `npm` and `bun` need the `run` verb; `pnpm` and `yarn` invoke the binary directly. Running `pnpm` in a non-pnpm project resolves against the wrong lockfile and fails quietly.
3. **ALWAYS run `pnpm psy <command> --help`** before using any generator — never guess. Argument formats vary between commands and between versions.
4. **NEVER use JavaScript `Date`** — always `DateTime`, `CalendarDate`, `ClockTime`, or `ClockTimeTz` from `@rvoh/dream` (timestamp / date / time-without-tz / time-with-tz) — these are what `castParam`/`extractParams` return and what the DB hydrates. See [models.md — Date/Time](models.md#datetime).
5. **NEVER stub or mock Dream internals** in specs — use factories to create real model instances. A stub returns what you wrote it to return, so the spec proves the stub, not the query.
6. **NEVER modify a migration file already merged into main.** Machines that recorded it as applied skip it, so schemas silently diverge — express the change as a new migration.
7. **A generator must always be used** when creating new models, controllers, or migrations.
8. **Sources of truth** (priority order): TSDocs > `pnpm psy <command> --help` > psychic-skill.
9. **BDD approach**: write the failing spec first, then implement. Generated code is the only exception (generators scaffold specs and implementation simultaneously).
10. **Run `pnpm psy sync`** after changing associations, serializers, OpenAPI decorators, or routes, and after adding a decorator declaring a virtual column (`@deco.Virtual()`, `@deco.Encrypted()`) — that sync is separate from the one a migration triggers. Without it, `create()` / `update()` reject the virtual attribute at build time while every runtime spec passes.
11. **Use Dream's built-in utilities** (`@rvoh/dream/utils`) instead of lodash or hand-rolled equivalents. See [utils.md](utils.md).
12. **Read application config through `AppEnv` (`api/src/conf/AppEnv.ts`), never `process.env`.** `AppEnv` is typed by a union of the names the app declares, so a `process.env` read is a variable nothing types, lists, or lets a spec set. Variables present in only some environments use the `{ optional: true }` overload, which returns `string | undefined` instead of throwing — never reach for `process.env` to avoid the throw. This governs *application config*: a dev-only launcher whose job is to *compose* the environment handed to spawned children is not that, and reading `process.env` to spread into a `spawn(..., { env })` child is correct there. See [deploying.md](deploying.md#environment-variables).
13. **NEVER add try/catch blocks unless handling a specific, expected error.** Dead programs tell no lies. Psychic already converts common errors to HTTP responses (`findOrFail` → 404, `castParam` → 400, validation failure → 400). Catch only the error you expect, re-throw everything else, and never wrap a large block in a catch-all. **"I'm logging, not swallowing" and "it's only a small per-iteration catch" are both rejected** — see [controllers.md](controllers.md#automatic-error-handling) and [workers.md](workers.md#never-rescue-exceptions-inside-backgrounded-services).
14. **Branching or mapping on a closed-enum value must stop compiling when a value is added or removed.** Closed enums are database enums in `@src/types/db.js` (`PlaceStylesEnum`, `BookingStatusesEnum`), STI type discriminators (`RoomTypesEnum`), and any other union of string literals. Adding a value widens the union everywhere at once; the compiler finds the sites with a hole in them, and `pnpm build:spec` surfaces them across `src/` and `spec/`. The trigger is the closed enum, not your judgment about needing every case today. Use one of the two shapes below.

    **Different code per value — exhaustive `switch` with a `const _never: never` default.** Every value is named, no-ops included; `_never` proves none is missing.

    ```ts
    const status = booking.status
    switch (status) {
      case 'confirmed': await booking.notifyGuest(); return
      case 'cancelled': await booking.releaseHold(); return
      case 'pending': return   // explicit no-op — acknowledged, not forgotten
      default: {
        const _never: never = status
        throw new Error(`Unhandled BookingStatusesEnum: ${String(_never)}`)
      }
    }
    ```

    **A value per member — `Record<Enum, T>`**, whenever the answer is data and no branch runs: it requires a key for every member, so a new place style fails to compile at the literal. (Labels are i18n — [i18n.md](i18n.md).)

    ```ts
    const NIGHTLY_MINIMUM: Record<PlaceStylesEnum, number> = {
      cabin: 2, cave: 3, cottage: 2, dump: 1, lean_to: 1, tent: 1, treehouse: 1,
    }
    ```

    **An `if` / `else if` chain does not give you the check.** It type-checks and **silently no-ops** when a value is added, including a two-branch one whose `else` absorbs every later value. A lone comparison used as a guard, with no `else`, is not what this rejects.

    **Case labels are the literal string** (`case 'confirmed':`); a constant typed as the union (`const x: BookingStatusesEnum = 'confirmed'`) widens `x` and breaks the `_never` check, as does a `satisfies` alias.

15. **The database is the source of truth for types defined at the database level.** Never hardcode database enum values — import the generated constants and types from `@src/types/db.js` (`PlaceStylesEnumValues`, `RoomTypesEnum`), which `pnpm psy sync` regenerates and propagates to types, OpenAPI specs, and clients. Outside migrations, enum literals appear only where the type system pins them to one member — a `case` label, a `Record` key, a where/create/association-condition attribute, or an assignment to a union-typed variable. Type anything holding one known value as the union type and assign the literal directly; reach for the values array only where a runtime array is needed (`castParam(..., { enum })`, OpenAPI enum schemas, select options, iteration), never to pluck out a single member.
16. **Commit all auto-generated files** after `pnpm psy db:migrate` or `pnpm psy sync` — `src/types/`, `src/openapi/`, and any configured sync output directories. Don't cherry-pick.
17. **Application code logs through `PsychicApp`, never `console.log`.** `PsychicApp.log(message, ...meta)` for general output, `PsychicApp.logWithLevel(level, message, ...meta)` to set a level (`'debug' | 'info' | 'warn' | 'error'`). A Psychic app configures one logger (Winston by default) and these are its entry point; a `console.log` line never passes through it. See [controllers.md](controllers.md#logging). REPL / `pnpm console` sessions are exempt — interactive `stdout` is the point.
18. **For Psychic surfaces that are thin wrappers over Koa, defer to upstream docs.** Where Psychic exposes a Koa-layer knob (`psy.set('json', { ... })`, `psy.use(...)`), the skill teaches the Psychic-specific shape and links upstream for options and defaults. Same posture for `ioredis`, `BullMQ`, `socket.io`, and `pg` wrappers.
19. **NEVER hand-code OpenAPI schema for a shape Psychic can derive.** Before writing `requestBody.properties`, `responses[status].properties`, or `enum: SomeEnumValues`, choose the derived path:
    - Model request bodies use `requestBody: { params: [...] }` / `{ including: [...] }`, even when the action must use `castParam` instead of `extractParams` for STI dispatch or custom validation. `params` is the OpenAPI request-body narrowing key. **These must be literal arrays mirroring the action's `extractParams` allowlist — never backfill them from the model's own `paramSafeColumns` or from `Model.columns()`. See [controllers.md](controllers.md#requestbody-shorthand--what-each-option-is-for).**
    - Model responses use `@OpenAPI(Model, { serializerKey })` and serializers. Computed / view-model responses use an `ObjectSerializer` passed to `@OpenAPI(SerializerFn, { status })`, nested objects nesting further `ObjectSerializer`s; create one if it does not exist yet.
    - Hand-written JSON Schema is only for ad hoc shapes no model, serializer, or new `ObjectSerializer` can represent; "there is no serializer yet" is not a reason. See [openapi.md](openapi.md#how-psychic-builds-the-spec).

20. **The controller directory tree IS the auth architecture; a surface that loosens auth is its own top-level namespace.** Authed client endpoints live under `V1/`; any surface that loosens auth — public/maybe-authed, webhooks, partner API — is its own top-level namespace with the version nested inside (`Visitor/V1/`, `Webhooks/V1/`, `Api/V1/`), never `V1/Visitor/`. `Admin/` and `Internal/` are separate top-level surfaces with their own `AuthedController`. Generate the surface, then reparent its top-level namespace base controller once. Auth is enforced by ancestry: the placement *is* the enforcement. Full rules: [controllers.md](controllers.md#controller-hierarchy).
21. **Reach for the simplest shape that satisfies the requirement; escalate only on evidence you can point at.** Before choosing anything heavier than a plain `update`, a `findEach`, or a database constraint, name the concrete scenario that breaks the simple shape — a specific concurrent writer, a measured row count, an invariant the database cannot express. "A race is conceivable" and "this table might get large" are not that. A one-shot data correction goes wrong most often: it is a `findEach` calling `update` per row, or the `Query#update` callback form when the new value derives from the row. When a review finding is an artifact of complexity you introduced, remove the complexity instead of hardening it. See [locking.md](locking.md#most-writes-need-no-lock).

## Project Structure and Commands

`api/src/app/` holds `models/`, `controllers/` (with `Admin/`, `Internal/`, `helpers/`), `serializers/`, `services/`; `api/src/conf/` holds `app.ts`, `dream.ts`, `AppEnv.ts`, `routes.ts`, `locales/`, `initializers/`; migrations, generated types and specs are under `api/src/db/migrations/`, `api/src/types/`, `api/spec/`.

**Every `pnpm psy` command defaults to `NODE_ENV=test`** — prefix `NODE_ENV=development` to reach the development database; type generation only runs against test ([console.md](console.md)). **NEVER use `npx tsc --noEmit`**: it fails with spurious errors on spec types a bare `tsc` cannot resolve — use `pnpm build:spec` or `pnpm build`. Scaffold a new app with `npx @rvoh/create-psychic new <app-name>`; every option left off becomes a prompt, and **the GitHub Actions question has no flag**, so `new` can never run unattended.

## Naming Conventions

Database columns, enum types (suffixed `_enum`), enum values and generator column arguments (`name:string`, `User:belongs_to`) are snake_case; model properties camelCase; model, controller and serializer classes and files PascalCase; route paths kebab-case. STI `type` values are PascalCase and **MUST match the STI child class names** (`Bedroom`). `date` columns end in `On` and `datetime` columns in `At`.

## Troubleshooting Migrations

**"Corrupted migrations"**: switching branches between different migration sets makes `pnpm psy db:migrate` fail this way; the fix is `pnpm psy db:reset`. Everything else migration-related is in [migrations.md](migrations.md).
