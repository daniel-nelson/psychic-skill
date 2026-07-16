# Psychic Websockets - Real-Time Communication

## Overview

Psychic Websockets provides real-time communication via **Socket.IO** with Redis pub/sub for distributed scaling.

### Core Classes

- **`Cable`** - WebSocket server manager
- **`Ws`** - Typed message emitter
- **`PsychicAppWebsockets`** - Configuration

## Configuration

```typescript
// conf/initializers/websockets.ts
import { allowRequestForOrigins, allowedCorsOrigins } from '@rvoh/psychic-websockets'
import resolveWebsocketUser from '../system/resolveWebsocketUser.js'

export default (psy: PsychicApp) => {
  // PsychicAppWebsockets initializes in all processes by default. Any process —
  // websocket server, web server, or worker — may call Ws.emit(), and skipping
  // init in any of them causes a runtime cachePsychicAppWebsockets error that is
  // hard to diagnose. To restrict which roles can push messages, uncomment:
  //
  // if (!['websockets', 'web', 'worker'].includes(AppEnv.serviceRole) && !AppEnv.isTest) return

  psy.plugin(async () => {
    await PsychicAppWebsockets.init(psy, initializeWebsockets)
  })
}

function initializeWebsockets(wsApp: PsychicAppWebsockets) {
  // Redis connection
  wsApp.set('connection', new Redis({
    host: process.env.REDIS_HOST,
    port: Number(process.env.REDIS_PORT),
    maxRetriesPerRequest: null,
  }))

  // Socket.IO options — `allowRequest` enforces an origin allowlist on the handshake
  // across BOTH long-polling and native WebSocket transports. Socket.IO's own
  // `cors.origin` only applies to long-polling, so use `allowRequestForOrigins` here
  // to close the gap.
  wsApp.set('socketio', {
    transports: ['websocket', 'polling'],
    allowRequest: allowRequestForOrigins(allowedCorsOrigins()),
  })

  // Health check endpoint
  wsApp.set('healthCheck', {
    path: '/healthcheck',
    method: 'GET',
    body: null,
  })

  // Handle new connections — auth via the boilerplate-scaffolded resolveWebsocketUser
  // helper. Newly-generated apps reject all WS connections until this is implemented.
  wsApp.on('ws:start', io => {
    io.of('/').on('connection', async socket => {
      const user = await resolveWebsocketUser(socket)
      if (!user) {
        socket.disconnect(true)
        return
      }

      await Ws.register(socket, user.id)

      const ws = new Ws(['/ops/connection-success'] as const)
      await ws.emit(user.id, '/ops/connection-success', { message: 'Connected' })
    })
  })
}
```

### Auth: `resolveWebsocketUser`

`create-psychic` scaffolds a boilerplate auth helper at `conf/system/resolveWebsocketUser.ts`. Newly-generated apps reject every WebSocket connection until you replace its body with real auth logic — typically pulling a token from `socket.handshake.auth`, decrypting it, and returning the matching `User` (or `null` to reject).

```typescript
// conf/system/resolveWebsocketUser.ts
import type { Socket } from 'socket.io'
import User from '../../app/models/User.js'

export default async function resolveWebsocketUser(socket: Socket): Promise<User | null> {
  const token = socket.handshake.auth.token as string | undefined
  if (!token) return null
  // ...decrypt the token and look up the user
  return User.find(/* userId */)
}
```

The "fail loudly in dev" ergonomic is intentional — if auth isn't wired up, no socket connects.

**A throwing `ws:connect` hook is contained to the connecting socket.** If an auth hook throws (a DB or Redis blip mid-handshake, say), the framework catches it, logs at error level, and calls `socket.disconnect(true)` — the ws process stays up. So a hook may throw to reject a connection; the cost is that one socket, not process safety, and the hook does not need its own outer try/catch to protect the process. Framework-contained ws failures are also surfaced to the `ws:error` hook (below), which is where you ship them to your monitoring SDK. (Startup is stricter: a redis adapter that can't attach aborts ws startup rather than silently falling back to in-memory delivery.)

### `ws:error`: the request-aware websocket error hook

`ws:error` is the ws-layer analogue of `server:error` — the point to forward a framework-contained websocket failure to your monitoring SDK (Sentry, Datadog). Register it alongside `ws:start` and `ws:connect`, with the same positional signature `(error, context)`:

```typescript
// conf/websockets.ts
wsApp.on('ws:error', (error, context) => {
  if (context.phase === 'ws:connect') {
    // a ws:connect hook threw; context.socketId identifies the socket
    reportToSentry(error, { socketId: context.socketId })
  } else {
    // context.phase === 'ws:health-check'
    // the ws server's own HTTP handler threw; context.method, context.path
    reportToSentry(error, { method: context.method, path: context.path })
  }
})
```

The second argument is a discriminated context (`PsychicWebsocketsErrorContext`, exported alongside `PsychicWebsocketsErrorHook`) with two phases:

- **`phase: 'ws:connect'`** — a `ws:connect` hook threw while a socket was connecting. Carries `socketId`. A throw-to-reject (an auth hook throwing to refuse a connection) fires this too, so filter your own rejection sentinels inside the hook, exactly as you would in `server:error`.
- **`phase: 'ws:health-check'`** — the websocket server's own HTTP request handler (Psychic's health check and its catch-all 404) threw. Carries `method` and `path`. This is the ws process's raw `http.Server`, **not** the Koa app — errors here never reach `server:error`.

Three properties worth knowing:

- **The context is privacy-scrubbed for external shipping.** It never carries a raw socket, handshake credentials, request headers, or the raw request URL — `path` is the pathname with the query string stripped, because a query string can carry tokens or PII and the 404 branch sees arbitrary URLs. The framework's own internal error *log* still records the full URL locally; only the external-bound hook context is scrubbed. So you get full detail in your logs and safe-by-default in what you ship to Sentry.
- **Observers are isolated and never recurse.** Each `ws:error` observer runs in its own try/catch; one that throws is logged and can't block a later observer, and an observer's own failure is never re-dispatched through `ws:error`.
- **Containment wraps only `ws:connect` hooks.** If you do per-socket auth inside a `ws:start` hook's own `io.on('connection')` handler instead of in a `ws:connect` hook, a throw there is **not** contained and gets no `ws:error` coverage. Put per-socket auth in `wsApp.on('ws:connect', …)` so it's both contained and observed.

**Redis adapter connection errors are observed by app-attached listeners, by design — there is no `ws:error` phase for them.** ioredis emits `'error'` on the pub/sub clients on every reconnect attempt during an outage, which would flood an external tracker; the framework leaves throttling and shipping to you. Attach your own `.on('error', …)` to the public `wsApp.connection` and `wsApp.subConnection` getters immediately after `wsApp.set('connection', redis)` (the `subConnection` — a `duplicate()` of the pub client — only exists after that call), and dedupe before shipping. Configure the connection **once**: `attachServer` captures the clients at startup, so replacing the connection after `cable.start(...)` disconnects the old clients without rebinding the adapter, silently breaking cross-process delivery.

### Origin allowlist

`allowRequestForOrigins(origins: string[])` returns a socket.io `allowRequest` handler that rejects handshakes whose `Origin` header isn't in the allowlist. Wire it via `wsApp.set('socketio', { allowRequest: ... })` (shown above).

`allowedCorsOrigins()` reads the `CORS_HOSTS` environment variable, which must be a JSON-encoded array of origins:

```bash
CORS_HOSTS='["https://app.example.com","https://admin.example.com"]'
```

If `CORS_HOSTS` is malformed JSON, the helper throws `CORS_HOSTS must be a JSON-encoded array of origins` at boot. Override the env-parsing strategy by passing a different array to `allowRequestForOrigins(...)` directly.

## Emitting Messages

```typescript
import { Ws } from '@rvoh/psychic-websockets'

// Create typed Ws instance with allowed paths
const ws = new Ws([
  '/notifications/new',
  '/data/update',
  '/chat/message',
] as const)

// Emit to a user (by ID)
await ws.emit(userId, '/notifications/new', {
  message: 'You have a new booking!',
  timestamp: new Date(),
})

// Emit to a Dream model instance
const user = await User.findOrFail(id)
await ws.emit(user, '/data/update', { field: 'status', value: 'active' })
```

## Registering Users

```typescript
// In websocket configuration (ws:start hook)
wsApp.on('ws:start', io => {
  io.of('/').on('connection', async socket => {
    const userId = extractUserId(socket.handshake.auth)
    const user = await User.find(userId)

    if (user) {
      // Register socket -> stored in Redis key 'user:{userId}'
      await Ws.register(socket, user.id)

      // With custom prefix
      await Ws.register(socket, user.id, 'admin-user')
    }
  })
})
```

## WebSocket Server Startup

```typescript
// ws.ts (separate process)
import { Cable } from '@rvoh/psychic-websockets'
import initializePsychicApp from './src/cli/helpers/initializePsychicApp.js'

let cable: Cable | null = null

async function startWs() {
  process.env.WS_SERVICE = '1'
  await initializePsychicApp()

  cable = new Cable()
  await cable.start(Number(process.env.WS_PORT || 8888))
}

process.on('SIGINT', async () => {
  if (cable) await cable.stop()
  process.exit(0)
})

startWs()
```

## Using with Background Workers

> **Process initialization required.** `Ws.emit()` boots a `PsychicAppWebsockets` instance on its first call via `PsychicAppWebsockets.getOrFail()`. If the worker process skipped the websocket initializer, that call throws:
> ```
> must call `cachePsychicAppWebsockets` before loading cached psychic application websockets
> ```
> BullMQ marks the job failed and retries — the error looks like a framework cache problem rather than an application configuration problem. The generated initializer runs in all processes by default (see Configuration above); if a role guard was added that excludes `'worker'`, that is the cause.
>
> **If the job completes but the browser shows stale data until refresh:** the initializer is not the problem, and neither is the client transport. The likely cause is cross-process delivery: an emit from a web or worker process only reaches a socket held by the websocket-server process through the Redis adapter. A process emitting on the in-process adapter (the `test` default, or a misconfigured non-`test` process) fans out only within its own process, so the browser's socket never receives it.

Common pattern: Background job sends websocket notification:

```typescript
export class NotificationService extends ApplicationBackgroundedService {
  public static async placeBooked(placeId: string, guestId: string) {
    await this.background('_placeBooked', placeId, guestId)
  }

  public static async _placeBooked(placeId: string, guestId: string) {
    const place = await Place.findOrFail(placeId)
    const hosts = await place.associationQuery('hosts').all()

    const ws = new Ws(['/notifications/booking'] as const)
    for (const host of hosts) {
      await ws.emit(host.userId, '/notifications/booking', {
        placeName: place.name,
        guestId,
      })
    }
  }
}

// Triggered from model hook
@deco.AfterCreateCommit()
public async notifyBooking(this: Booking) {
  await NotificationService.placeBooked(this.placeId, this.guestId)
}
```

## Client Transport

Set `transports: ['websocket']` on the Socket.IO client. Socket.IO defaults to trying long-polling first for historical reasons (WebSocket support was patchy in 2012); that fallback is unnecessary today — WebSocket is universally supported by modern browsers and mobile apps. Skipping the polling phase means faster connection establishment. This is a connection-latency recommendation: the websocket server serves polling and WebSocket clients correctly either way, so it is not required for correctness — it just avoids the slower initial handshake.

```typescript
import { io } from 'socket.io-client'

const socket = io(websocketHost, {
  transports: ['websocket'],
  auth: { token },
})
```

## Plugin Registration

```typescript
// conf/app.ts or initializePsychicApp.ts
import { PsychicAppWebsockets } from '@rvoh/psychic-websockets'
import websocketsConf from '../../conf/websockets.js'

await PsychicAppWebsockets.init(psychicApp, websocketsConf)
```
