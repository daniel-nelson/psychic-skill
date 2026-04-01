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

## Piping Commands from Claude Code

To run console commands non-interactively:

```bash
echo 'console.log(await Place.count()); process.exit(0)' | NODE_ENV=development pnpm console 2>&1
```

Key points:
- Must call `process.exit(0)` at the end or the console will hang
- Use `2>&1` to capture both stdout and stderr
- The console prints startup output before your code runs — look for output after the `>` prefix
- For multi-line code, use semicolons or wrap in an async IIFE
- Use `JSON.stringify()` for clean parsing of output

## Running Scripts Against the Development Database

**For ad hoc DB/model inspection** (counting records, checking data, exploring associations), use the console or pipe commands into it — not a custom script. See "Piping Commands from Claude Code" above.

For standalone scripts that need the full Dream environment (models, associations, DB connection), use this boilerplate:

```typescript
import '@conf/loadEnv.js'
import initializePsychicApp from '@conf/system/initializePsychicApp.js'
await initializePsychicApp()

// Now you can import and use models
const { default: Place } = await import('./src/app/models/Place.js')
```

Run with:

```bash
NODE_ENV=development npx tsx my-script.ts
```

**Do not hand-roll app initialization** with ad hoc `tsx -e` one-liners that manually import `loadEnv`, `initializePsychicApp`, and individual models. This skips important initialization steps and will fail. Use the boilerplate above for scripts, or `pnpm console` for interactive inspection.
