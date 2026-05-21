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
> BullMQ marks the job failed and retries — the error looks like a framework cache problem rather than an application configuration problem. The fix is to include `'worker'` in the initializer guard (see Configuration above) so `PsychicAppWebsockets.init()` runs in the worker process. The dedicated websocket server is a separate process and still owns `Cable.start()` and socket connection handling; the worker only needs the Redis connection that `Ws.emit()` uses.

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

## Dedicated WebSocket Host: transport configuration

When the websocket server runs as a separate process/host (the typical production topology), Socket.IO clients default to starting with long-polling before upgrading to native WebSocket. In this topology, polling requests to the dedicated websocket host can conflict with the websocket server's HTTP handler, producing header-sent failures or stale browser state: the background job completes and the event is published, but the browser never receives a live update until manual refresh.

**Diagnostic signals:**

- Browser: network tab shows `transport=polling` requests to the websocket host.
- Server: websocket process logs HTTP/header errors even though jobs complete successfully.

**Fix:** force native WebSocket transport on the Socket.IO client:

```typescript
import { io } from 'socket.io-client'

const socket = io(websocketHost, {
  path: '/ws',
  transports: ['websocket'],
  auth: { token },
})
```

Do not add `'polling'` unless you have explicitly verified that long-polling works through the same proxy and websocket route stack. The failure is easy to misdiagnose as queue, worker, or channel subscription trouble — check the client transport first.

## Plugin Registration

```typescript
// conf/app.ts or initializePsychicApp.ts
import { PsychicAppWebsockets } from '@rvoh/psychic-websockets'
import websocketsConf from '../../conf/websockets.js'

await PsychicAppWebsockets.init(psychicApp, websocketsConf)
```
