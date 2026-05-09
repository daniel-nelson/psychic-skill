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

export default (wsApp: PsychicAppWebsockets) => {
  if (AppEnv.serviceRole !== 'ws' && !AppEnv.isTest) return

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

## Plugin Registration

```typescript
// conf/app.ts or initializePsychicApp.ts
import { PsychicAppWebsockets } from '@rvoh/psychic-websockets'
import websocketsConf from '../../conf/websockets.js'

await PsychicAppWebsockets.init(psychicApp, websocketsConf)
```
