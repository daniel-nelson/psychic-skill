# Changelog

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
