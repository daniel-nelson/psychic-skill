# OpenAPI - Detailed Reference

## How Psychic builds the spec

Psychic derives the OpenAPI spec automatically. Database column types, serializers, and routes already describe most of your API, so Psychic reads them and emits the schema rather than asking you to write it. You declare only the remainder:

- **Per-action concerns** — status codes, tags, request-body overrides, query parameters, custom responses — go in the `@OpenAPI` decorator on each controller action. See [controllers.md](controllers.md#openapi-decorator-options).
- **Response schema** comes from serializer attributes. A `DreamSerializer` infers types from the model's columns; an `ObjectSerializer` carries explicit `openapi` types. See [serializers.md](serializers.md#overview).
- **Spec-wide concerns** — output paths, namespaces, default headers and responses, security schemes, validation — go in the `psy.set('openapi', ...)` block in `conf/app.ts`, covered below.

Because the spec is customizable centrally, don't hack around it in client code. If every request needs a bearer header, declare a security scheme once (below) rather than attaching the header by hand in the generated client. If an endpoint's response shape is wrong, fix the serializer rather than hand-writing a `responses` block. Run `pnpm psy sync` after any change so the spec files and generated clients update.

## Conf-level configuration

The spec-wide configuration lives in `conf/app.ts`. The default (client-facing) spec is configured with `psy.set('openapi', { ... })`; named specs use `psy.set('openapi', '<name>', { ... })`.

### outputFilepath and namespaces

`outputFilepath` is where the spec JSON is written. Each `psy.set('openapi', '<name>', ...)` call defines a separate named spec with its own output file. Use namespaces to publish separate admin, internal, and public specs so internal or admin endpoints never leak into a public API document.

```typescript
// Default (client-facing) — controllers without an openapiNames override
psy.set('openapi', {
  outputFilepath: path.join('src', 'openapi', 'openapi.json'),
  validate: { requestBody: true, headers: true, query: true, responseBody: AppEnv.isTest },
})

// Named specs — controllers opt in via the openapiNames getter
psy.set('openapi', 'admin', {
  outputFilepath: path.join('src', 'openapi', 'admin.openapi.json'),
  validate: { requestBody: true, headers: true, query: true, responseBody: AppEnv.isTest },
})

psy.set('openapi', 'internal', {
  outputFilepath: path.join('src', 'openapi', 'internal.openapi.json'),
  validate: { requestBody: true, headers: true, query: true, responseBody: AppEnv.isTest },
})

// Tests spec — surfaces include 'tests' so controller specs type-check against one spec
psy.set('openapi', 'tests', {
  outputFilepath: path.join('src', 'openapi', 'tests.openapi.json'),
  syncTypes: true,
})
```

Controllers route into a named spec by overriding the `openapiNames` getter; that side lives in [controllers.md](controllers.md#opting-controllers-into-an-openapi-namespace). Run `pnpm psy sync` after adding a new namespace so the `openapiNames` types update.

### defaults

`defaults` applies values to every endpoint in the spec unless an action overrides them. It holds `headers`, `responses`, `securitySchemes`, `security`, and `components`.

```typescript
psy.set('openapi', {
  outputFilepath: path.join('src', 'openapi', 'openapi.json'),
  defaults: {
    // applied to every endpoint
    headers: {
      locale: { type: 'string', enum: LocalesEnumValues },
    },
    // Psychic already supplies 400/401/403/404/409/422 — only override to change or add
    responses: {
      429: { description: 'Too many requests' },
    },
    securitySchemes: { bearerAuth: { type: 'http', scheme: 'bearer' } },
    security: [{ bearerAuth: [] }],
    // reusable schema components referenced elsewhere in the spec
    components: {
      schemas: {
        HealthCheck: { type: 'object', properties: { ok: { type: 'boolean' } } },
      },
    },
  },
})
```

### validate

`validate` sets the validation rules applied to every action tied to this spec, unless an `@OpenAPI` decorator overrides them. Accepts `requestBody`, `responseBody`, `headers`, and `query` booleans, or the `all: true` shorthand.

```typescript
psy.set('openapi', {
  outputFilepath: path.join('src', 'openapi', 'openapi.json'),
  validate: { requestBody: true, headers: true, query: true, responseBody: AppEnv.isTest },
})
```

Gating `responseBody` on `AppEnv.isTest` validates responses under test without paying the cost in production.

### syncTypes

When `syncTypes` is true, Psychic reads the spec with `openapi-typescript` and generates TypeScript interfaces from it. Use those to type response bodies in tests via the `OpenapiResponseBody` utility.

```typescript
psy.set('openapi', {
  outputFilepath: path.join('src', 'openapi', 'openapi.json'),
  syncTypes: true,
})
```

```typescript
import { openapiPaths } from '@src/types/openapi.js'

it('returns posts', async () => {
  const res = await request.get('/posts', 200)
  const body = res.body as OpenapiResponseBody<openapiPaths, '/posts', 'get', 200>
})
```

### The long tail

`info` (version/title/description), `servers`, `suppressResponseEnums`, and `checkDiffs` are configured in the same block. For their full shapes see the TSDoc on `PsychicOpenapiBaseOptions` in `@rvoh/psychic`.

## Declaring a security scheme

Declaring a security scheme is conf-level customization: you state the scheme once and Psychic stamps it across the spec. To require a bearer token on every endpoint, set `securitySchemes` and a top-level `security` in `defaults`:

```typescript
psy.set('openapi', {
  outputFilepath: path.join('src', 'openapi', 'openapi.json'),
  validate: { requestBody: true, headers: true, query: true, responseBody: AppEnv.isTest },
  defaults: {
    securitySchemes: { bearerAuth: { type: 'http', scheme: 'bearer' } },
    security: [{ bearerAuth: [] }],
  },
})
```

After `pnpm psy sync`, the spec gains `components.securitySchemes.bearerAuth` and a top-level `security: [{ bearerAuth: [] }]`, which Psychic stamps onto every operation. A client generated from the spec then honors the declared scheme and attaches the bearer token to requests, so you don't wire the header per call.

**Type note:** `defaults.security` is typed `OpenapiSecurity = Record<string, string[]>[]` — an array of objects. Use the array form `[{ bearerAuth: [] }]`, not the bare object `{ bearerAuth: [] }`.
