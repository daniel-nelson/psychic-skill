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

`outputFilepath` is where the spec JSON is written. Each `psy.set('openapi', '<name>', ...)` call defines a separate named spec with its own output file. Namespaces exist for two reasons:

- **Separate by access domain.** Publish admin, internal, and public endpoints as their own specs so endpoints meant for one audience never appear in a document meant for another. Add new specs as the app grows — for example a public server-to-server `partner` spec you publish while keeping the client, admin, and internal specs private.
- **Shape the schema per consumer.** The same endpoints can emit a different schema shape into different specs to fit how each client consumes them. The built-in `mobile` spec is this case (see [The mobile spec](#the-mobile-spec-and-strongly-typed-consumers) below).

A fresh create-psychic app ships five specs: `default` (client/web), `mobile`, `admin`, `internal`, and `tests`.

```typescript
// Default (client/web) — controllers without an openapiNames override
psy.set('openapi', {
  outputFilepath: path.join('src', 'openapi', 'openapi.json'),
  validate: { requestBody: true, headers: true, query: true, responseBody: AppEnv.isTest },
})

// Mobile — same endpoints as default, enums emitted as described strings
psy.set('openapi', 'mobile', {
  outputFilepath: path.join('src', 'openapi', 'mobile.openapi.json'),
  suppressResponseEnums: true,
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

// Tests spec — every surface includes 'tests' so controller specs type-check against one spec
psy.set('openapi', 'tests', {
  outputFilepath: path.join('src', 'openapi', 'tests.openapi.json'),
  syncTypes: true,
})
```

Controllers route into a named spec by overriding the `openapiNames` getter; that side lives in [controllers.md](controllers.md#opting-controllers-into-an-openapi-namespace). Run `pnpm psy sync` after adding a new namespace so the `openapiNames` types update.

### The mobile spec and strongly-typed consumers

`mobile` is identical to `default` except it sets `suppressResponseEnums: true`: instead of emitting an enum as a closed set of values, the spec emits a plain `string` whose description lists the allowed values.

This exists because a strongly-typed mobile client compiles an enum into a closed type. When the backend later adds an enum value, an older app version that hasn't been rebuilt crashes on the unrecognized value, which makes the API rigid — every new value is a breaking change for anyone who hasn't updated. Emitting strings keeps the mobile contract open: the app ignores values it doesn't recognize, and often handles the set dynamically (rendering whatever options the backend sends in a select, for instance). Web front ends compile to JavaScript, which isn't strongly typed, so the `default` spec keeps real enums — front-end tooling benefits from them and production doesn't break when the backend adds a value.

### The tests spec

Every base controller includes `'tests'` in its `openapiNames`, so the `tests` spec aggregates endpoints from all surfaces (client, admin, internal) into one document. With `syncTypes: true`, `pnpm psy sync` generates TypeScript `paths` types from it, and those types make controller specs type-safe on both the request and the response. See [testing.md](testing.md#request-and-response-types-come-from-the-tests-spec) for how specs consume them.

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

`info` (version/title/description), `servers`, and `checkDiffs` are configured in the same block. For their full shapes see the TSDoc on `PsychicOpenapiBaseOptions` in `@rvoh/psychic`.

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
