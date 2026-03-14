# Psychic Workers & Websockets - Detailed Reference

## Workers (Background Jobs)

### Overview

Psychic Workers is built on **BullMQ** (Redis-based job queue). It provides type-safe backgrounding of service and model methods.

### Base Classes

- **`ApplicationBackgroundedService`** - For static method backgrounding
- **`ApplicationBackgroundedModel`** (extends Dream) - For model instance method backgrounding
- **`ApplicationScheduledService`** - For recurring/cron-based jobs

### Backgrounding Service Methods

```typescript
export class EmailService extends ApplicationBackgroundedService {
  public static async sendWelcome(userId: string) {
    const user = await User.findOrFail(userId)
    // Send email logic
  }

  // Optional: configure priority/workstream
  public static override get backgroundJobConfig() {
    return { priority: 'urgent' }
  }
}

// Queue the job
await EmailService.background('sendWelcome', user.id)

// Queue with delay
await EmailService.backgroundWithDelay(
  { seconds: 300, jobId: `email-${user.id}` },
  'sendWelcome',
  user.id
)
```

### Backgrounding Model Instance Methods

```typescript
export class User extends ApplicationBackgroundedModel {
  public async sendNotification(message: string) {
    // Instance method - runs on specific user
  }

  public static async processAll(batchSize: number) {
    // Static method
  }

  public static override get backgroundJobConfig() {
    return { workstream: 'user-processing', priority: 'default' }
  }
}

// Static method
await User.background('processAll', 100)

// Instance method
const user = await User.findOrFail(id)
await user.background('sendNotification', 'Hello!')
```

### Priority Levels

```typescript
type BackgroundQueuePriority = 'default' | 'urgent' | 'not_urgent' | 'last'
// Maps to numeric: 1 (urgent), 2 (default), 3 (not_urgent), 4 (last)
```

### Scheduled/Cron Jobs

```typescript
export class DailyReportService extends ApplicationScheduledService {
  public static async generateReport() {
    // Runs on schedule
  }
}

// Cron patterns
await DailyReportService.schedule('0 9 * * *', 'generateReport')   // 9am daily
await DailyReportService.schedule('0 * * * *', 'heartbeat')         // Hourly
await DailyReportService.schedule('0 9 * * 1', 'weeklyReport')     // Monday 9am
```

### Worker Configuration

```typescript
// conf/initializers/workers.ts
export default (workersApp: PsychicAppWorkers) => {
  workersApp.set('background', {
    providers: { Queue, Worker },

    defaultBullMQQueueOptions: {
      defaultJobOptions: {
        removeOnComplete: 1000,
        removeOnFail: 20000,
        attempts: 20,
        backoff: { type: 'exponential', delay: 1000 },
      },
    },

    defaultWorkstream: {
      workerCount: parseInt(process.env.WORKER_COUNT || '0'),
      concurrency: 100,
    },

    namedWorkstreams: [
      {
        name: 'email',
        workerCount: 1,
        rateLimit: { max: 10, duration: 1000 },  // 10/sec
      },
    ],

    defaultQueueConnection: new Redis({
      host: process.env.REDIS_HOST,
      port: Number(process.env.REDIS_PORT || 6379),
    }),

    defaultWorkerConnection: new Redis({
      host: process.env.REDIS_HOST,
      port: Number(process.env.REDIS_PORT || 6379),
      maxRetriesPerRequest: null,
    }),
  })

  workersApp.on('workers:shutdown', () => {
    // Cleanup when workers shut down
  })
}
```

### Critical Pattern: Use AfterCommit Hooks

**ALWAYS use `AfterCreateCommit` / `AfterUpdateCommit` hooks (not `AfterCreate` / `AfterUpdate`) when backgrounding work.** This prevents race conditions where the background job runs before the database transaction commits.

```typescript
// CORRECT
@deco.AfterCreateCommit()
public async notifyCreation(this: Place) {
  await NotificationService.background('placeCreated', this.id)
}

// WRONG - job may run before transaction commits
@deco.AfterCreate()
public async notifyCreation(this: Place) {
  await NotificationService.background('placeCreated', this.id)
}
```

### Testing Workers

```typescript
// Default: Jobs execute immediately in tests
await EmailService.background('sendWelcome', user.id)
// Method runs synchronously

// Manual mode
import { WorkerTestUtils, PsychicAppWorkers } from '@rvoh/psychic-workers'

const workersApp = PsychicAppWorkers.getOrFail()
workersApp.set('testInvocation', 'manual')

await EmailService.background('sendWelcome', user.id)  // Queued only
await WorkerTestUtils.work()                             // Process queue
WorkerTestUtils.clean()                                  // Clear all queues
await WorkerTestUtils.workScheduled()                    // Run scheduled jobs
```

---

## Websockets

### Overview

Psychic Websockets provides real-time communication via **Socket.IO** with Redis pub/sub for distributed scaling.

### Core Classes

- **`Cable`** - WebSocket server manager
- **`Ws`** - Typed message emitter
- **`PsychicAppWebsockets`** - Configuration

### Configuration

```typescript
// conf/initializers/websockets.ts
export default (wsApp: PsychicAppWebsockets) => {
  if (AppEnv.serviceRole !== 'ws' && !AppEnv.isTest) return

  // Redis connection
  wsApp.set('connection', new Redis({
    host: process.env.REDIS_HOST,
    port: Number(process.env.REDIS_PORT),
    maxRetriesPerRequest: null,
  }))

  // Socket.IO options
  wsApp.set('socketio', {
    transports: ['websocket', 'polling'],
  })

  // Health check endpoint
  wsApp.set('healthCheck', {
    path: '/healthcheck',
    method: 'GET',
    body: null,
  })

  // Handle new connections
  wsApp.on('ws:start', io => {
    io.of('/').on('connection', async socket => {
      const token = socket.handshake.auth.token as string
      const userId = decryptToken(token)
      const user = await User.find(userId)

      if (user) {
        await Ws.register(socket, user.id)

        const ws = new Ws(['/ops/connection-success'] as const)
        await ws.emit(user.id, '/ops/connection-success', { message: 'Connected' })
      }
    })
  })
}
```

### Emitting Messages

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

### Registering Users

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

### WebSocket Server Startup

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

### Using with Background Workers

Common pattern: Background job sends websocket notification:

```typescript
export class NotificationService extends ApplicationBackgroundedService {
  public static async placeBooked(placeId: string, guestId: string) {
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
  await NotificationService.background('placeBooked', this.placeId, this.guestId)
}
```

### Plugin Registration

Both workers and websockets are Psychic plugins:

```typescript
// conf/app.ts
export default async (psy: PsychicApp) => {
  // Workers plugin
  psy.plugin(async () => {
    await PsychicAppWorkers.init(psy, workersConf)
  })

  // Websockets plugin
  psy.plugin(async () => {
    await PsychicAppWebsockets.init(psy, websocketsConf)
  })
}
```
