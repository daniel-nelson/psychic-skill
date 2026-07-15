# Deploying Psychic Applications

## Runtime Model

A Psychic application runs multiple process roles from a single built image. Each role has its own entrypoint:

| Role | Entrypoint | Purpose |
|------|-----------|---------|
| **web** | `node ./dist/src/main.js` | HTTP API server |
| **websocket** | `node ./dist/src/ws.js` | WebSocket server (if using `@rvoh/psychic-websockets`) |
| **worker** | `node ./dist/src/worker.js` | Background job processor (if using `@rvoh/psychic-workers`) |
| **console / migrator** | `node ./dist/src/conf/system/cli.js db:migrate` | Database migrations and CLI tasks |

**Do not rely on `pnpm` (or any package manager runner) in production containers.** Use direct `node ./dist/...` commands in your deployment configuration (task definitions, Procfiles, Docker CMD, etc.).

## Build Output

- If the project uses TypeScript path aliases (e.g., `@app/*`), the production build **must** run `tsc-alias` after `tsc`. Without this step, compiled JS will contain unresolved path alias imports and fail at runtime.
- When building in Docker, explicitly creating the `node_modules` directory before `pnpm install` (or equivalent) can avoid intermittent install failures:
  ```dockerfile
  RUN mkdir -p /app/node_modules && pnpm install --frozen-lockfile
  ```

## Health Checks

The API server needs an explicit health check route registered in `api/src/conf/routes.ts`:

```typescript
r.get('health_check', HealthCheckController, 'index')
```

The `@rvoh/psychic-websockets` package has a **built-in** health check, but be aware of two differences from the API health check:

| | API | WebSocket |
|--|-----|-----------|
| Default path | `/health_check` (you define it) | `/healthcheck` (built-in) |
| HTTP methods | Responds to `GET` and `HEAD` | Responds to `GET` only |

**Practical consequence:** `curl -I` (which sends `HEAD`) works against the API health check but returns `404` from the websocket service. When verifying websocket health externally, use `curl -X GET` instead.

## Environment Variables

**Always use `AppEnv` to access environment variables** — never `process.env` directly. `AppEnv` (defined in `api/src/conf/AppEnv.ts`) validates that required variables are present at boot time and provides a clear error message when one is missing, rather than failing silently deep in application code.

`AppEnv` also decouples the application from the container's environment. Because all env access goes through a single entry point, the application can load secrets from external sources at boot time — e.g., AWS Secrets Manager or AWS Systems Manager Parameter Store — and inject them into `AppEnv` before anything else runs. This is a significant advantage for secret management: the container itself doesn't need every secret baked into its environment, and secrets can be rotated without redeploying.

For variables that are only present in some environments, use `AppEnv.string(name, { optional: true })` — it returns `string | undefined` rather than throwing at boot. This is the supported escape hatch for sometimes-present vars; never reach for `process.env` to avoid the boot-time throw.

**Register every new variable name in `AppEnv.ts`'s typed union.** `AppEnv` extends `Env<{ boolean: …; integer: …; string: … }>`, where each key is a closed union of the allowed variable names for that accessor. `AppEnv.string('SOME_NEW_VAR')` only compiles once `'SOME_NEW_VAR'` is added to the `string` union — so adding an env var is a two-part change: add the name to the union and read it through `AppEnv`. The gotcha is *where* the omission surfaces: a Vitest run (`pnpm uspec`) does not type-check, so a spec exercising the new var passes green, and the missing-name error (`TS2345`) only appears at the `pnpm build:spec` type-check gate. Add the union entry in the same change, and don't read a green spec run as done until the build gate has run.

## TLS Behind a Reverse Proxy

If your reverse proxy (load balancer, ingress controller, etc.) terminates TLS but re-encrypts traffic to the container, the Psychic app must speak TLS on its container port. In this case, baking a self-signed certificate into the image is more reliable than generating one dynamically at runtime.

If the proxy terminates TLS and forwards plain HTTP to the container, no additional TLS configuration is needed in the app.

## Postgres TLS

`SingleDbCredential` accepts an `ssl?: TlsConnectionOptions | false` field that flows directly to the underlying `pg.Pool` connection. The bare `ssl: true` shorthand is rejected — callers must choose explicitly between verified TLS (`{ rejectUnauthorized: true }`) and unverified TLS (`{ rejectUnauthorized: false }`). `ssl: false` is the explicit TLS-off sentinel.

`app.set('db', ...)` throws `MissingDbSslDirective` at setter time — in every environment, not just production — if neither `ssl` nor the legacy `useSsl` is set on a credential. Silently defaulting to TLS-off was the previous behavior; removing that default is intentional, so the call site has to make TLS posture explicit.

The `create-psychic` boilerplate ships verified TLS via the system CA store as the default, with `DB_NO_SSL=true` as the explicit-off escape hatch:

```typescript
const dbSsl: { rejectUnauthorized: true } | false = AppEnv.boolean('DB_NO_SSL')
  ? false
  : { rejectUnauthorized: true }

app.set('db', {
  primary: { /* host, port, user, password, database */ ssl: dbSsl },
  replica: hasReplica ? { /* ... */ ssl: dbSsl } : undefined,
})
```

Most managed Postgres providers (Supabase, Neon, Render, Azure Database for PostgreSQL Flexible Server) present a public-CA-signed certificate and work with the boilerplate default. Other providers need an explicit override at the call site:

| Provider | Required `ssl` value |
|---|---|
| Supabase, Neon, Render, Azure Database for PostgreSQL Flexible Server | `{ rejectUnauthorized: true }` (boilerplate default) |
| AWS RDS | `{ rejectUnauthorized: true, ca: readFileSync('rds-ca.pem') }` |
| GCP Cloud SQL | `{ rejectUnauthorized: true, ca: readFileSync('server-ca.pem') }` |
| Heroku Hobby, some local Docker images | `{ rejectUnauthorized: false }` |
| Local dev with no TLS | `ssl: false` or `DB_NO_SSL=true` |

```typescript
// Verified TLS with a pinned CA — e.g., AWS RDS
const credential: SingleDbCredential = {
  // ...host, port, user, password, database
  ssl: {
    rejectUnauthorized: true,
    ca: AppEnv.string('PG_CA_CERT'),
  },
}

// Unverified TLS — the self-signed-cert path. Only when the provider serves a cert that
// doesn't chain to a known root (Heroku Hobby, some local Docker images). Not the default.
const credential: SingleDbCredential = {
  // ...
  ssl: { rejectUnauthorized: false },
}
```

Existing apps generated under the previous boilerplate may still set `useSsl: true`; that keeps working but is deprecated and will be removed in a future major. The migration is to switch the credential to an explicit `ssl` value chosen from the matrix above — new apps already opt into verified TLS by default, so this only affects apps scaffolded before this change.

## Read Replicas

`app.set('db', { primary, replica })` accepts an optional `replica` credential alongside `primary`, same shape (`host`, `port`, `user`, `password`, `name`, `ssl`) pointed at your read replica instance:

```typescript
app.set('db', {
  primary: { host: AppEnv.string('DB_HOST'), /* port, user, password, database */ ssl: dbSsl },
  replica: hasReplica
    ? { host: AppEnv.string('DB_REPLICA_HOST'), /* port, user, password, database */ ssl: dbSsl }
    : undefined,
})
```

Configuring `replica` only makes the connection available — it doesn't route any traffic there by itself. A model only reads from the replica when it's decorated `@ReplicaSafe()`, and even then only for `select` queries outside a transaction. See [models.md — Replica Safety](models.md#replica-safety-replicasafe) for the full routing rules (join fallback to primary, `create`/`update`/`destroy` always primary, per-call `.connection(...)` override).

Omit `replica` (or pass `undefined`) for environments with no replica — every model, `@ReplicaSafe()` or not, then reads from `primary`.

## Migrations in Production Deploys

### The migrate task is the boot smoke test

Running `node ./dist/src/conf/system/cli.js db:migrate` exercises the full Psychic boot chain before applying any migrations:

1. Node boot on the deployed image.
2. `loadEnv` resolves the environment (including any external-secret injection wired through `AppEnv`).
3. `AppEnv` materializes — every required env var is validated and the application halts with a clear error if anything is missing.
4. The Psychic registry initializes — every module that registers on import (models, serializers, controllers, routes, workers, websocket channels) runs its top-level side effects.
5. A database connection is established using the configured credentials, including TLS negotiation against the resolved `ssl` value.

Only then does Kysely start applying pending migrations. Any failure in steps 1–5 fails the migrate task before the schema changes — which means a no-op "boot smoke" task that only waits for the container to reach a running state is redundant. The migrate task itself is the smoke test, and it covers strictly more than container liveness: missing env vars, registry-time import errors, malformed DB credentials, TLS handshake failures, and network reachability all surface here.

### Combine `db:migrate && db:seed` in one invocation

Migrate and seed share the same image, the same execution role, the same env, and the same DB credentials. On serverless container platforms (anything where each one-off task pays a fixed scheduling and image-pull overhead before the container starts), invoking them as two separate one-off tasks pays the per-task lifecycle twice for no reason — including the case where seed is a no-op `if (AppEnv.isTest) return`.

Combine them in a single container invocation, preserving exit-zero semantics via shell-and:

```bash
sh -c "node ./dist/src/conf/system/cli.js db:migrate && node ./dist/src/conf/system/cli.js db:seed"
```

If migrate fails, seed never runs and the task exits non-zero. If migrate succeeds and seed fails, the task exits non-zero with seed's error — the same shape your deploy script would see from a separate seed invocation.

## Debugging a Deployed Psychic Service

When a deployed Psychic service is not responding correctly:

1. **Check the public endpoint** — is it returning the expected status code?
2. **Check health check status** — is the target healthy from the load balancer/ingress perspective?
3. **Check orchestrator events** — container restarts, OOM kills, failed deployments
4. **Check the running configuration** — correct image, correct command, correct environment variables, correct log destination
5. **Check application logs** — look for startup errors, missing env vars, failed migrations
6. **Verify HTTP method** — if health checks pass but manual verification fails, confirm you're using `GET` (not `HEAD`) for websocket endpoints
