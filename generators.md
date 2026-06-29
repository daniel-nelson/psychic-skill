# Generators - Scaffolding Workflow

How to drive Dream/Psychic's code generators (`g:resource`, `g:model`, `g:sti-child`, `g:migration`) safely. This file owns the full scaffolding arc: picking a generator, running it, and the edit/migrate/sync/commit steps that follow. For **how to choose a model's name and namespace**, see [models.md — Model Organization & Namespacing](models.md#model-organization--namespacing). For migration file anatomy and the column DSL, see [migrations.md](migrations.md).

> Examples use `pnpm psy`; substitute your project's package manager (`yarn psy`, `npm run psy`, `bun run psy`). See SKILL.md Critical Rule #2.

## Always run `--help` first

**CRITICAL: Run `pnpm psy <command> --help` and read its output BEFORE running any generator or CLI command.** Do not infer syntax from examples in this skill or from prior experience — argument formats vary between commands and between Dream/Psychic versions. This is a hard prerequisite, not a suggestion.

## Generator decision tree

Preference order:

1. **Resource generator** (`g:resource`) — the default for almost all new models.
2. **STI-child generator** (`g:sti-child`) — for STI child models building on an existing STI base.
3. **Model generator** (`g:model`) — only for models that will never be exposed via any API.
4. **Migration generator** (`g:migration`) — for database changes when not generating a new model.

**Most models should use `g:resource`.** It is rare for a model to not be exposed via *some* API — whether end-user facing, internal, or admin. `g:resource` generates the controller, controller specs, and route scaffolding that `g:model` does not. It is easier to delete unused controller actions than to retrofit them later. Only use `g:model` for models that are purely internal (e.g., join tables, audit logs).

**If a model has a `type` column, consider whether it should use STI.** If different types will have different behavior, validations, serializers, or child-specific columns, use STI (`--sti-base-serializer` on the parent resource + `g:sti-child` for each child). The STI generators handle significant subtle scaffolding (check constraints, per-type serializers, type-discriminated OpenAPI schemas) that is much easier to get right at generation time than to refactor later. See [sti.md](sti.md).

## `g:resource` argument contract

`g:resource` takes three **orthogonal** identity inputs:

- **1st arg — route path**: a plural kebab-case route path (e.g., `v1/host/places` or `internal/action-plans`), not a class name or namespace. May be nested with `{}` parent-id placeholders. Controls the controller name and the generated routes (URLs). Its **top-level segment is an auth decision** — it picks the controller's branch, and the branch *is* the auth architecture. Authed client endpoints go under `v1/...`; any surface that loosens auth (public/maybe-authed, webhooks, server-to-server) is its own top-level namespace with the version nested inside (`visitor/v1/...`, `webhooks/v1/...`, `api/v1/...`), never `v1/visitor/...`. After generating a loosen-auth surface, reparent that namespace's base controller once. See [controllers.md — Controller Hierarchy](controllers.md#controller-hierarchy).
- **2nd arg — model file path**: the path relative to `src/app/models/` without `.ts` (e.g., `Place`, `Place/Room`). Controls where the model class is written, which drives the class name, table name, and every import. The class name defaults from the path; override with `--model-name`.
- **`--owning-model=<Parent>`**: scoping only. Controls whether the generated controller scopes its queries and writes through the parent (`associationQuery` / `createAssociation`). It does **not** affect where the model file is written.

Because these are independent, **a nested route does not imply a namespaced model** — a nested route plus `--owning-model` already gives you parent-scoped queries and the right spec scaffolding without putting the model under a `Parent/` namespace. Choosing the 2nd-arg namespace is a model-identity decision: see [models.md — Model Organization & Namespacing](models.md#model-organization--namespacing).

### Nested resources MUST pass `--owning-model`

A nested resource has a parent-id placeholder in its route path (the `{}` segment, e.g. `v1/posts/{}/comments`, `internal/clients/{}/addresses`). For these, always pass `--owning-model` set to the fully-qualified nesting (parent) model name (`--owning-model=Post`, `--owning-model=Client`). This makes the generated controller scope every query and write through `associationQuery` / `createAssociation` on the owning model, and scaffolds the parent correctly throughout — including in the generated controller spec. Omitting it generates a controller that doesn't scope to the parent and a spec that references an unconstructed parent path-param variable (the symptom is a spec that 404s on the missing parent rather than exercising the action). For `admin`/`internal` paths (which drop `currentUser` scoping), `--owning-model` is also how you reintroduce ownership scoping.

## Generated defaults

`g:resource` and `g:model` automatically include `id`, `created_at`, `updated_at`, and `deleted_at` columns and decorate the model with `@SoftDelete()`. Do not specify these in the generator command. Use `--no-soft-delete` to opt out, since hard deletion should be an intentional decision.

## Generator workflow (after a generator runs)

1. **Update the migration file as needed** (e.g., add `unique()` to a column).
2. **Run migrations**: `pnpm psy db:migrate` (this also runs `sync` automatically — do NOT follow it with a separate `pnpm psy sync`).
3. **If the generator was a resource generator**:
   - Update the generated controller spec first.
   - Then update the corresponding generated controller.
   - Note: controller specs will hang if there is no response within a controller action (generated action code starts commented out).
4. **Commit generated code as its own commit** with a message in this format:
   ```
   Generate Room resource

   ```console
   pnpm psy g:resource --sti-base-serializer --owning-model=Place v1/host/places/rooms Room type:enum:room_types:Bathroom,Bedroom,Kitchen,Den
   ```
   ```

## When to run `pnpm psy sync`

Run `pnpm psy sync` whenever any of the following are added or changed:

- An association in a Dream model
- A serializer
- An OpenAPI decorator on a controller action
- A route

If controller specs have type errors about what an endpoint accepts or returns, the OpenAPI shape is out of sync — run `pnpm psy sync`. (Standalone `sync` is only needed when no migration was run; `db:migrate` already runs it.)

## Adding properties to an existing model

- **Always use the migration generator** (`pnpm psy g:migration`) to create a migration for schema changes. Never create migration files by hand.
- **`g:migration` accepts the same column shorthand as `g:resource` and `g:model`** — including `BelongsTo:type` for foreign keys. Pass column descriptors as additional positional args; the migration body is generated correctly without hand-editing for the common cases.
- The migration name is **kebab-case** (matches the migration filename convention `kebab-case-description.ts`), not snake_case.
- Examples:
  - Plain column: `pnpm psy g:migration add-timezone-to-users timezone:string`
  - Multiple columns: `pnpm psy g:migration add-fields-to-bars name:string size:integer`
  - Foreign key (notNull): `pnpm psy g:migration add-zip-code-id-to-candidates ZipCode:belongs_to`
  - Foreign key (nullable): `pnpm psy g:migration add-zip-code-id-to-candidates ZipCode:belongs_to:optional`
  - Aliased FK (`@alias`): `pnpm psy g:migration add-canceled-by-to-message-requests InternalUser@canceled_by:belongs_to:optional` produces `canceled_by_id` column, `canceledById` property, and `canceledBy` association from one token. Canonical for `_by` columns (`created_by`, `approved_by`) and multiple FKs to the same model (`Message@last_inbound`, `Message@last_outbound`). See [migrations.md — Aliased BelongsTo shorthand](migrations.md#aliased-belongsto-shorthand-modelaliasbelongs_to) for the full reference.
- After running the migration (`pnpm psy db:migrate`), add the matching `public ...: DreamColumn<Model, 'columnName'>` declaration to the model file. For a BelongsTo, also add the `@deco.BelongsTo(...)` declaration. Commit the auto-generated `src/types/db.ts` / `src/types/dream.ts` changes alongside.
- The generator scaffolding can be modified for changes that aren't expressible as column shorthand (check constraints, enum alterations, custom backfill) — use DreamMigrationHelpers methods over compound Kysely calls when available. See [migrations.md](migrations.md) for migration anatomy and helpers.
