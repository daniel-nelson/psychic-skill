# Dream Console & Development Environment

## NODE_ENV Defaults

**CRITICAL: All `pnpm psy` commands default to `NODE_ENV=test`.** To operate on the development database, you must explicitly prefix with `NODE_ENV=development`:

```bash
NODE_ENV=development pnpm psy db:reset    # Resets the DEVELOPMENT database
pnpm psy db:reset                          # Resets the TEST database (default)
```

This applies to `db:migrate`, `db:rollback`, `db:reset`, and all other `pnpm psy` commands. Running `pnpm psy db:reset` without `NODE_ENV=development` will only reset the test database, leaving stale data in the development database.

**Use caution with `NODE_ENV=development pnpm psy db:reset`** — it drops and recreates the development database, losing all data. This is safe if all needed development data is in a seed file, but can be destructive otherwise.

## Dream Console

### Launching

```bash
NODE_ENV=development pnpm console
```

**The console command is `pnpm console`, not `pnpm psy console`.** The `psy` CLI is for generators, migrations, sync, and `db:reset`. The console is a separate top-level script in `api/package.json` alongside `pnpm web:dev` and `pnpm uspec`. If you see `error: unknown command 'console'`, you've prefixed `psy` by mistake — drop it.

Wait for the console to fully load before using auto-imported names. The console does NOT live-reload — exit and re-launch after code changes.

### Exiting

Press `ESC` twice to exit.

### Auto-imported Models

Models are available by their class name, derived from the file path relative to `src/app/models/` with slashes removed and the extension dropped:

| File Path | Console Name |
|---|---|
| `src/app/models/User.ts` | `User` |
| `src/app/models/Client/Contract.ts` | `ClientContract` |
| `src/app/models/Client/Contract/Invoice.ts` | `ClientContractInvoice` |

### Auto-imported Services

Services are prefixed with `Services`, derived from the path relative to `src/app/services/`:

| File Path | Console Name |
|---|---|
| `src/app/services/NotificationService.ts` | `ServicesNotificationService` |
| `src/app/services/Host/OnboardingService.ts` | `ServicesHostOnboardingService` |

### Auto-imported Utilities

Dream utilities are available globally:
- `CalendarDate.today()`, `DateTime.now()`
- `sort(array)`, `sortBy(array, callback)`
- Other Dream utils from `@rvoh/dream/utils`

### Manual Imports

For code not auto-imported, use dynamic import with path relative to `src/`:

```js
const helpers = (await import('./src/app/services/Host/pricingHelpers.js')).default
```

### Example Queries

```js
await Place.count()
const place = await Place.find('some-uuid')
const place = await Place.preload('rooms').preload('hosts').first()
const rooms = await place.associationQuery('rooms').all()
```

## Non-interactive console use from an agent

**`pnpm console` is interactive — designed for a human at a terminal who launches it, waits for full boot, then types or pastes queries.** Piping commands into stdin races the boot: the REPL begins reading before auto-imports finish, each newline is a separate evaluation (so `const x = ...` on line 1 isn't visible to line 2), and `process.exit(0)` can fire before async work resolves. Treat piping as unreliable, not as a documented option.

For non-interactive work from an agent, the answer is a **scratch script** using the boilerplate in "Running Scripts Against the Development Database" below. The boilerplate handles Psychic's load-bearing init order; do not hand-roll an alternative.

If a human is in the loop, the cleanest path is still: launch `pnpm console`, wait for the prompt, paste the query.

## Running Scripts Against the Development Database

For non-interactive use from an agent, write a scratch script using the boilerplate below. (For interactive use by a human, `pnpm console` is the right tool — see "Dream Console" above.)

The boilerplate mirrors the canonical `src/conf/repl.ts` (the file `pnpm console` runs), minus the REPL-specific `DreamCLI.loadRepl(...)` call. Same boot order — load env first, then initialize the Psychic app, then import models.

```typescript
// api/scratch-cleanup.ts
//
// Mirrors src/conf/repl.ts boot order. DreamCLI.loadRepl is REPL-specific
// and intentionally omitted — a script imports models explicitly.

import '@conf/loadEnv.js'                                      // (1) side-effect — loads .env / .env.test based on NODE_ENV
import initializePsychicApp from '@conf/system/initializePsychicApp.js'

await initializePsychicApp()                                   // (2) boots Dream + Psychic

const { default: Place } = await import('./src/app/models/Place.js')  // (3) dynamic — must be after init
console.log(await Place.count())

process.exit(0)
```

**Running the script (default: test environment):**

```bash
# From the api/ directory — defaults to the test database:
APP_NAME=$(node -p "require('./package.json').name") \
  APP_VERSION=$(node -p "require('./package.json').version") \
  npx tsx scratch-cleanup.ts
```

`loadEnv.js` defaults `NODE_ENV` to `test` when unset, so omitting the prefix targets the test DB and `.env.test`. **An agent should default to test** — it's the database the agent already controls (`pnpm psy db:reset` rebuilds it from migrations, and the agent is already authorized to do so).

**Only switch to `NODE_ENV=development` when the user explicitly asks** for development-database work. The development DB is not the agent's to manage:

- **Never run `pnpm psy db:reset` against development.** The agent does not know what data the developer has in dev — seed data, ad-hoc records, partially-imported fixtures, anything. A reset destroys all of it. The way to ensure `db:reset` targets the test DB is to invoke it with no `NODE_ENV` prefix and to confirm `NODE_ENV` is not exported in the surrounding shell (`echo $NODE_ENV` should be empty, or `test`). If `NODE_ENV=development` is set in the environment, unset it before running `psy` commands — never let an inherited environment variable silently redirect a destructive command.
- **Migrations are roughly safe to run forward, but rollback by step count is not reliable.** `pnpm psy db:rollback --steps=<N>` rolls back the last `N` recorded migrations on the dev DB; in principle `N` is the number of migration files present on the feature branch but not on `main`. In practice that count can be wrong: `main` can carry migrations whose timestamps are *later* than the branch's, so "what's on the branch but not main" doesn't match "what migrated last on dev." Verify before rolling back, or ask the user to do it.
- **Schema drift hazard when editing a branch-only migration in place.** Recall from `migrations.md`: editing a still-on-branch migration plus `pnpm psy db:reset` on the test DB will bring test back in sync, but the development DB — already migrated against the *pre-edit* shape — silently retains the old shape (e.g., the old enum values). The result is dev and test diverging in ways that confuse subsequent work. If the user has migrated dev against an unmerged migration that you then edit, you must tell them — they need to choose between resetting dev themselves, manually backing out the affected rows, or recreating dev from a seed.

When the user has explicitly asked for dev-database work, prefix the invocation:

```bash
NODE_ENV=development \
  APP_NAME=$(node -p "require('./package.json').name") \
  APP_VERSION=$(node -p "require('./package.json').version") \
  npx tsx scratch-cleanup.ts
```

**Things that bite agents who don't follow this exactly:**

- **CWD must be `api/`.** Both the path alias `@conf/...` and the relative `./src/...` resolve from `api/`.
- **`APP_NAME` / `APP_VERSION` are read at boot** by some projects' `app.ts` / `AppEnv` setup. The `console` script in `package.json` sets these from `$npm_package_name` / `$npm_package_version` automatically; standalone `tsx` doesn't, so set them explicitly. Skip this only if the project's boot doesn't reference them.
- **Mixed import paths are intentional.** Static framework-config imports use the `@conf/...` alias (resolved by `tsx` via `tsconfig.json`). Model imports go through `await import('./src/...')` because the model registry is built **during** `initializePsychicApp()` — a static `import` would resolve before init completes.
- **Do not hand-roll an alternative** — `tsx -e "..."` one-liners that import individual models without `loadEnv` + `initializePsychicApp` will fail in confusing ways that look like model bugs but are actually init-order bugs.
- Add `scratch-*.ts` to `.gitignore` if it isn't already, and delete scratch files after use — they're throwaway, not artifacts.
