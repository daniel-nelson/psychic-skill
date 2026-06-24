# Changelog

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
