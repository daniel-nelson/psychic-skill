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

## TLS Behind a Reverse Proxy

If your reverse proxy (load balancer, ingress controller, etc.) terminates TLS but re-encrypts traffic to the container, the Psychic app must speak TLS on its container port. In this case, baking a self-signed certificate into the image is more reliable than generating one dynamically at runtime.

If the proxy terminates TLS and forwards plain HTTP to the container, no additional TLS configuration is needed in the app.

## Postgres TLS

`SingleDbCredential` accepts an `ssl?: boolean | tls.ConnectionOptions` field that flows directly to the underlying `pg.Pool` connection. Precedence: explicit `ssl` wins > legacy `useSsl: true` (which falls back to unverified TLS for backward compatibility with hosted Postgres) > TLS off.

```typescript
// Verified TLS using the system CA store (preferred when the DB cert chains to a public CA)
const credential: SingleDbCredential = {
  // ...host, port, user, password, database
  ssl: true,
}

// Verified TLS with a pinned CA (preferred for managed Postgres with a custom root, e.g., RDS)
const credential: SingleDbCredential = {
  // ...
  ssl: {
    rejectUnauthorized: true,
    ca: AppEnv.string('PG_CA_CERT'),
  },
}

// Unverified TLS — only when the deployment environment forces it (e.g., legacy hosted DBs
// without a published CA bundle). Document the rationale alongside.
const credential: SingleDbCredential = {
  // ...
  ssl: { rejectUnauthorized: false },
}
```

Existing apps that set `useSsl: true` keep working — the boilerplate falls back to `{ rejectUnauthorized: false }` for compatibility with RDS / Heroku / Supabase. New apps should opt into verified TLS via `ssl` instead.

## Debugging a Deployed Psychic Service

When a deployed Psychic service is not responding correctly:

1. **Check the public endpoint** — is it returning the expected status code?
2. **Check health check status** — is the target healthy from the load balancer/ingress perspective?
3. **Check orchestrator events** — container restarts, OOM kills, failed deployments
4. **Check the running configuration** — correct image, correct command, correct environment variables, correct log destination
5. **Check application logs** — look for startup errors, missing env vars, failed migrations
6. **Verify HTTP method** — if health checks pass but manual verification fails, confirm you're using `GET` (not `HEAD`) for websocket endpoints
