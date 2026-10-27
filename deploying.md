# Deploying Psychic Applications

## Runtime Model

A Psychic application runs multiple process roles from a single built image. Each role has its own entrypoint:

| Role | Entrypoint | Purpose |
|------|-----------|---------|
| **web** | `node ./dist/src/main.js` | HTTP API server |
| **websocket** | `node ./dist/src/ws.js` | WebSocket server (if using `@rvoh/psychic-websockets`) |
| **worker** | `node ./dist/src/worker.js` | Background job processor (if using `@rvoh/psychic-workers`) |
| **console / migrator** | `node ./dist/src/conf/repl.js` · `node ./dist/src/conf/system/cli.js db:migrate` | Dream console · database migrations and CLI tasks |

**Do not rely on `pnpm` (or any package manager runner) in production containers.** Use direct `node ./dist/...` commands in your deployment configuration (task definitions, Procfiles, Docker CMD, etc.).

## Health Checks

The API server needs an explicit health check route. The scaffold's `api/src/conf/routes.ts` carries one commented out; uncomment it and adjust the path and body to what your infra expects:

```typescript
r.get('health_check', ctx => {
  ctx.status = 200
  ctx.body = { ok: true }
})
```

The `@rvoh/psychic-websockets` server answers its own **built-in** health check, configured with `wsApp.set('healthCheck', …)` ([websockets.md](websockets.md#configuration)). Its defaults differ from the API route:

| | API | WebSocket (default) |
|--|-----|-----------|
| Path | `/health_check` (you define it) | `/healthcheck` |
| HTTP methods | `GET` and `HEAD` | `GET` only (one configurable method) |

**Practical consequence:** `curl -I` (which sends `HEAD`) works against the API health check but returns `404` from a default websocket health check; use `curl -X GET` there.

## Environment Variables

**Always use `AppEnv` to access environment variables** — never `process.env` directly. `AppEnv` (defined in `api/src/conf/AppEnv.ts`) throws `MissingRequiredEnvironmentVariable`, naming the variable, when a required one is read while unset — at boot for anything `conf/app.ts`, `conf/dream.ts` or an initializer reads — instead of handing `undefined` on.

`AppEnv` also decouples the application from the container's environment. Because all env access goes through a single entry point, the application can load secrets from external sources at boot time — e.g., AWS Secrets Manager or AWS Systems Manager Parameter Store — and inject them into `AppEnv` before anything else runs. This is a significant advantage for secret management: the container itself doesn't need every secret baked into its environment, and secrets can be rotated without redeploying.

For variables that are only present in some environments, use `AppEnv.string(name, { optional: true })` — it returns `string | undefined` rather than throwing at boot. This is the supported escape hatch for sometimes-present vars; never reach for `process.env` to avoid the boot-time throw.

**Register every new variable name in `AppEnv.ts`'s typed union.** `AppEnv` extends `Env<{ boolean: …; integer: …; string: … }>`, where each key is a closed union of the allowed variable names for that accessor. `AppEnv.string('SOME_NEW_VAR')` only compiles once `'SOME_NEW_VAR'` is added to the `string` union — so adding an env var is a two-part change: add the name to the union and read it through `AppEnv`. The gotcha is *where* the omission surfaces: a Vitest run (`pnpm uspec`) does not type-check, so a spec exercising the new var passes green, and the missing-name error (`TS2345`) only appears at the `pnpm build:spec` type-check gate. Add the union entry in the same change, and don't read a green spec run as done until the build gate has run.

## TLS Behind a Reverse Proxy

The generated `conf/app.ts` serves TLS in production: under `AppEnv.isProduction` it calls `psy.set('ssl', { key: AppEnv.string('SSL_KEY_PATH'), cert: AppEnv.string('SSL_CERT_PATH') })`, so the web and websocket servers listen with HTTPS, and boot throws unless both variables are set. That fits a reverse proxy (load balancer, ingress controller) that terminates TLS and re-encrypts traffic to the container. The pair is a self-signed certificate, separate from the one on the proxy; bake it into the image rather than generating it at runtime.

If the proxy forwards plain HTTP to the container instead, delete that `psy.set('ssl', …)` block.

## Postgres TLS

`DreamDbConfig` accepts an `ssl?: TlsConnectionOptions | false` field that flows directly to the underlying `pg.Pool` connection. The bare `ssl: true` shorthand is rejected — callers must choose explicitly between verified TLS (`{ rejectUnauthorized: true }`) and unverified TLS (`{ rejectUnauthorized: false }`). `ssl: false` is the explicit TLS-off sentinel.

`app.set('db', ...)` throws `MissingDbSslDirective` at setter time — in every environment, not just production — if a credential sets neither `ssl` nor the legacy `useSsl: true`, so the call site has to make TLS posture explicit.

The `create-psychic` boilerplate ships verified TLS via the system CA store as the default, with `DB_NO_SSL=1` as the explicit-off escape hatch:

```typescript
const dbSsl: { rejectUnauthorized: true } | false = AppEnv.boolean('DB_NO_SSL')
  ? false
  : { rejectUnauthorized: true }

app.set('db', {
  primary: { /* host, port, user, password, name */ ssl: dbSsl },
  replica: AppEnv.string('REPLICA_DB_HOST', { optional: true }) ? { /* ... */ ssl: dbSsl } : undefined,
})
```

Most managed Postgres providers (Supabase, Neon, Render, Azure Database for PostgreSQL Flexible Server) present a public-CA-signed certificate and work with the boilerplate default. Other providers need an explicit override at the call site:

| Provider | Required `ssl` value |
|---|---|
| Supabase, Neon, Render, Azure Database for PostgreSQL Flexible Server | `{ rejectUnauthorized: true }` (boilerplate default) |
| AWS RDS | `{ rejectUnauthorized: true, ca: readFileSync('rds-ca.pem') }` |
| GCP Cloud SQL | `{ rejectUnauthorized: true, ca: readFileSync('server-ca.pem') }` |
| Heroku Hobby, some local Docker images | `{ rejectUnauthorized: false }` |
| Local dev with no TLS | `ssl: false` or `DB_NO_SSL=1` |

```typescript
// Verified TLS with a pinned CA — e.g., AWS RDS
const credential: DreamDbConfig = {
  // ...host, port, user, password, name
  ssl: {
    rejectUnauthorized: true,
    ca: AppEnv.string('PG_CA_CERT'),
  },
}

// Unverified TLS — the self-signed-cert path. Only when the provider serves a cert that
// doesn't chain to a known root (Heroku Hobby, some local Docker images). Not the default.
const credential: DreamDbConfig = {
  // ...
  ssl: { rejectUnauthorized: false },
}
```

A credential still carrying the deprecated `useSsl: true` resolves to **unverified** TLS (`{ rejectUnauthorized: false }`), not the verified default. Replace it with an explicit `ssl` value from the matrix above.

**Short-lived passwords** (AWS RDS IAM auth tokens): a token fetched once at boot stops working when it expires, and every new pool connection opened after that fails authentication. Set `password` on `primary` and any `replica` to a function returning the token or a promise of it — `password: () => signer.getAuthToken()`, where `signer` is an `@aws-sdk/rds-signer` `Signer` for that credential's host — so each new connection gets a fresh token.

## Read Replicas

The generated `conf/dream.ts` already passes a `replica` credential — same shape as `primary` (`host`, `port`, `user`, `password`, `name`, `ssl`) — whenever `REPLICA_DB_HOST` is set. Set it to your read replica's host, and `REPLICA_DB_PORT` if the replica's port differs from `DB_PORT`:

```typescript
replica: AppEnv.string('REPLICA_DB_HOST', { optional: true })
  ? { host: AppEnv.string('REPLICA_DB_HOST'), /* port, user, password, name */ ssl: dbSsl }
  : undefined,
```

Configuring `replica` only makes the connection available — it doesn't route any traffic there by itself. A model reads from the replica only when it's decorated `@ReplicaSafe()` or a call asks for `.connection('replica')`, and only for `select` queries outside a transaction. See [models.md — Replica Safety](models.md#replica-safety-replicasafe) for the full routing rules (join fallback to primary, `create`/`update`/`destroy` always primary, per-call `.connection(...)` override).

Omit `replica` (or pass `undefined`) for environments with no replica — every model, `@ReplicaSafe()` or not, then reads from `primary`.

## Migrations in Production Deploys

### The migrate task is the boot smoke test

Running `node ./dist/src/conf/system/cli.js db:migrate` exercises the Psychic boot chain before applying any migrations:

1. Node boot on the deployed image.
2. `loadEnv` resolves the environment (including any external-secret injection wired through `AppEnv`).
3. Every required variable boot reads through `AppEnv` (`conf/app.ts`, `conf/dream.ts`, initializers) must be set, or boot throws naming it.
4. Models, serializers, controllers, services and initializers load, so import-time errors surface. `db:migrate` boots with integrity checks bypassed, so route building, the OpenAPI document and the check that every model's table is in the schema run only when the web, websocket and worker processes boot.
5. A database connection is established using the configured credentials, including TLS negotiation against the resolved `ssl` value.

Only then does Kysely start applying pending migrations. Any failure in steps 1–5 fails the migrate task before the schema changes — which means a no-op "boot smoke" task that only waits for the container to reach a running state is redundant. The migrate task itself is the smoke test, and it covers strictly more than container liveness: missing env vars, registry-time import errors, malformed DB credentials, TLS handshake failures, and network reachability all surface here.

### Combine `db:migrate && db:seed` in one invocation

Migrate and seed share the same image, the same execution role, the same env, and the same DB credentials. On serverless container platforms (anything where each one-off task pays a fixed scheduling and image-pull overhead before the container starts), invoking them as two separate one-off tasks pays the per-task lifecycle twice for no reason — including the case where seed is a no-op `if (AppEnv.isTest) return`.

Combine them in a single container invocation, preserving exit-zero semantics via shell-and:

```bash
sh -c "node ./dist/src/conf/system/cli.js db:migrate && node ./dist/src/conf/system/cli.js db:seed"
```

If migrate fails, seed never runs and the task exits non-zero. If migrate succeeds and seed fails, the task exits non-zero with seed's error — the same shape your deploy script would see from a separate seed invocation.

### Dropping a column takes two deploys

A rolling deploy runs the migration while containers built from the previous image are still serving. Dropping a column is therefore a two-deploy process gated on the model's `ignoredColumns` getter — see [migrations.md — Dropping a column](migrations.md#dropping-a-column-declare-it-ignored-one-deploy-ahead).
