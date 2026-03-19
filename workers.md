# Psychic Workers - Background Jobs

## Overview

Psychic Workers is built on **BullMQ** (Redis-based job queue). It provides type-safe backgrounding of service and model methods, with automatic retry, priority levels, scheduled/cron jobs, multiple named queues, and (with BullMQ pro) queue based rate limiting.

Background work should be used to offload costly, process-intensive, or failure-prone operations from the web server, keeping API responses fast and resilient (e.g. an external service may be temporarily down, which would result in an API error, but a background job will simply retry).

### Base Classes

- **`ApplicationBackgroundedService`** - For static method backgrounding
- **`ApplicationBackgroundedModel`** (extends Dream) - For model instance method backgrounding (transparently handles serializing the model global name and id and calling the backgrounded method on the instantiated model)
- **`ApplicationScheduledService`** - For recurring/cron-based jobs

## Backgrounding Service Methods

Services are the preferred way to background work. The pattern is:

1. A **public entry method** receives rich objects (e.g., model instances), extracts serializable identifiers, and calls `this.background(...)`.
2. A **private implementation method** (prefixed with `_`) receives only serializable arguments (strings, numbers), looks up any needed models, and does the work.

This two-method pattern lets the service encapsulate its implementation and expose an expressive API — callers just write `await IntercomSyncService.syncUser(user)` without knowing or caring that the work is backgrounded, how arguments are serialized, or what the worker does internally. As a secondary benefit, it also handles the constraint that background job arguments must be serializable to Redis (model instances and complex objects cannot pass through the queue).

```typescript
import ApplicationBackgroundedService from './ApplicationBackgroundedService.js'

export class IntercomSyncService extends ApplicationBackgroundedService {
  // Public entry point — called from application code
  public static async syncUser(user: User) {
    await this.background('_syncUser', user.id)
  }

  // Private implementation — called by the worker
  public static async _syncUser(id: string) {
    const user = await User.find(id)
    if (!user) return
    // ...sync user to intercom
  }

  // Optional: configure priority and/or workstream
  public static override get backgroundJobConfig() {
    return { priority: 'not_urgent', workstream: 'Intercom' }
  }
}

// Usage from application code (e.g., a controller or model hook):
await IntercomSyncService.syncUser(user)
```

**Key rules:**
- **NEVER pass model data as background job arguments.** Almost always pass only the model's ID and look up the record in the implementation method. This applies to any data stored in a Dream model. Passing model data as arguments has serious downsides: it bloats Redis memory (a full JSON payload vs. a single ID string), loses all type information when serialized to JSON (e.g., `DateTime` becomes a plain string, enums become untyped values), and creates a snapshot that is immediately stale if the record is updated after the job is queued. The only arguments to `this.background(...)` should be IDs and simple scalar values (strings, numbers, booleans) that are not sourced from model columns. For model instance methods, `ApplicationBackgroundedModel` handles this automatically by storing only the primary key.
- **Use `find` (not `findOrFail`) in background job implementations**, and return early when the record is not found. Model deletion is a normal part of many application flows — a record may be deleted between when the job was queued and when the worker picks it up. Using `findOrFail` would throw an error, causing the job to be retried repeatedly for ~6 days before finally failing, wasting resources on a record that will never exist again.
- Always call the public entry method from application code, not `this.background(...)` directly from outside the service.

## backgroundWithDelay

`backgroundWithDelay` queues a job to run at least the specified amount of time in the future. Supports `seconds`, `minutes`, `hours`, and `days`.

```typescript
export class ImageProcessingService extends ApplicationBackgroundedService {
  public static async processUpload(uploadId: string) {
    // Wait for S3 upload to propagate before processing
    await this.backgroundWithDelay({ seconds: 15 }, '_processUpload', uploadId)
  }

  public static async _processUpload(uploadId: string) {
    const upload = await Upload.find(uploadId)
    if (!upload) return
    // ...process the image
  }
}
```

Use `backgroundWithDelay` when:
- A job may be queued before an external dependency is ready (e.g., waiting for a direct-to-S3 upload to exist before image processing begins)
- You want to add a grace period before processing

### Debounce with jobId

`backgroundWithDelay` supports debounce behavior via `jobId`. If a job with the same `jobId` is already queued with a delay, re-backgrounding with that `jobId` overwrites the previous job but resets the delay timer from the current time. This reduces duplicate work when events fire in quick succession.

```typescript
export class IntercomSyncService extends ApplicationBackgroundedService {
  public static async syncUser(user: User) {
    await this.backgroundWithDelay(
      { minutes: 2, jobId: `intercom-sync-user-${user.id}` },
      '_syncUser',
      user.id
    )
  }

  public static async _syncUser(id: string) {
    const user = await User.find(id)
    if (!user) return
    // ...sync user to intercom
  }
}
```

## Backgrounding Model Instance Methods

Models extending `ApplicationBackgroundedModel` can background both static and instance methods. For instance methods, Psychic stores the primary key and looks up the model when the worker picks up the job.

**Use this only when the model itself needs to background its own methods.** For most use cases, prefer a backgrounded service instead.

```typescript
export default class User extends ApplicationBackgroundedModel {
  // Static method
  public static async processAll(batchSize: number) {
    // ...
  }

  // Instance method
  public async sendNotification(message: string) {
    // Runs on a specific user instance
  }

  public static override get backgroundJobConfig() {
    return { workstream: 'user-processing', priority: 'default' }
  }
}

// Static method backgrounding
await User.background('processAll', 100)

// Instance method backgrounding — Psychic stores the primary key
// and re-fetches the model when the worker picks up the job
const user = await User.findOrFail(id)
await user.background('sendNotification', 'Hello!')
```

## Critical Pattern: Use AfterCommit Hooks

**ALWAYS use after-commit lifecycle hooks (`@deco.AfterCreateCommit`, `@deco.AfterUpdateCommit`, or `@deco.AfterSaveCommit`) instead of the regular hooks (`@deco.AfterCreate`, `@deco.AfterUpdate`, or `@deco.AfterSave`) when queuing background work from a Dream lifecycle hook.** Regular hooks run inside the transaction, so the worker may pick up the job before the transaction commits — meaning the record either doesn't exist yet or has stale data from the worker's perspective.

```typescript
// CORRECT — job runs after the transaction commits, so the worker can find the record
@deco.AfterCreateCommit()
public async notifyCreation(this: Place) {
  await NotificationService.placeCreated(this)
}

// WRONG — job may run before the transaction commits
@deco.AfterCreate()
public async notifyCreation(this: Place) {
  await NotificationService.placeCreated(this)
}
```

## Priority Levels

The priority of each backgrounded service/model is customizable via `backgroundJobConfig`:

```typescript
type BackgroundQueuePriority = 'urgent' | 'default' | 'not_urgent' | 'last'
// Maps to BullMQ numeric priority: 1 (urgent), 2 (default), 3 (not_urgent), 4 (last)
```

```typescript
export class FileImportService extends ApplicationBackgroundedService {
  public static get backgroundJobConfig(): BackgroundJobConfig<ApplicationBackgroundedService> {
    return { priority: 'not_urgent' }
  }
}
```

Use `last` for check-in/heartbeat jobs (e.g., Dead Man's Snitch) so they only run after all other work is processed, giving confidence that the queue is healthy.

## Scheduled/Cron Jobs

Classes extending `ApplicationScheduledService` can schedule recurring work using cron expressions via BullMQ:

```typescript
import ApplicationScheduledService from '../ApplicationScheduledService.js'

export default class ScheduledJobs extends ApplicationScheduledService {
  public static async scheduleAllJobs() {
    await this.schedule('0 * * * *', 'processHour')     // Every hour
    await this.schedule('0 9 * * *', 'dailyReport')     // 9am daily
    await this.schedule('0 9 * * 1', 'weeklyReport')    // Monday 9am
    await this.schedule('*/5 * * * *', 'backgroundCheckin')  // Every 5 minutes
  }

  public static async processHour() {
    await HourProcessor.process(DateTime.now())
  }

  public static async dailyReport() {
    await ReportService.generateDaily()
  }
}
```

## Named Workstreams

A workstream is a BullMQ queue with its own set of workers. Most apps only need the default workstream, but named workstreams are useful for isolating specific work (e.g., external API calls that need rate limiting).

Configure in `conf/initializers/workers.ts`:

```typescript
workersApp.set('background', {
  defaultWorkstream: {
    workerCount: os.cpus().length,
    concurrency: 100,
  },

  namedWorkstreams: [
    {
      name: 'Intercom',
      workerCount: 1,
    },
  ],
  // ...
})
```

After adding a named workstream, run `pnpm psy sync` to update types. Then assign services to the workstream via `backgroundJobConfig`:

```typescript
export class IntercomSyncService extends ApplicationBackgroundedService {
  public static get backgroundJobConfig(): BackgroundJobConfig<ApplicationBackgroundedService> {
    return { workstream: 'Intercom' }
  }
}
```

### Rate Limiting (BullMQ Pro)

Named workstreams can be rate limited when using a BullMQ Pro license. Requires `QueuePro` and `WorkerPro` providers:

```typescript
import { QueuePro, WorkerPro } from '@taskforcesh/bullmq-pro'

workersApp.set('background', {
  providers: {
    Queue: QueuePro,
    Worker: WorkerPro,
  },

  namedWorkstreams: [
    {
      name: 'Twilio',
      workerCount: 1,
      concurrency: 10,
      rateLimit: { max: 20, duration: 1000 },  // 20 requests/sec
    },
  ],
})
```

## Automatic Retry

Failed jobs (those that throw an unhandled error) are automatically retried with exponential backoff. The default configuration retries up to 20 times over ~6.1 days:

```typescript
defaultBullMQQueueOptions: {
  defaultJobOptions: {
    removeOnComplete: 1000,
    removeOnFail: 20000,
    // 524,288,000 ms (~6.1 days) using "2 ^ (attempts - 1) * delay"
    attempts: 20,
    backoff: {
      type: 'exponential',
      delay: 1000,
    },
  },
},
```

This config is sent directly to BullMQ and can be customized in `conf/initializers/workers.ts`.

**This is why using `find` instead of `findOrFail` matters in background jobs.** If a record has been deleted and the job uses `findOrFail`, the thrown error triggers 20 retries over 6 days — all of which will also fail, wasting resources. Using `find` and returning early when the record is `null` allows the job to exit cleanly.

## Job Logging

Background methods can optionally receive a BullMQ `Job` parameter as their last argument to access logging:

```typescript
import { Job } from 'bullmq'

export class DataProcessingService extends ApplicationBackgroundedService {
  public static async processDataset(datasetId: string) {
    await this.background('_processDataset', datasetId)
  }

  public static async _processDataset(datasetId: string, job: Job) {
    await job.log(`Starting processing of dataset ${datasetId}`)
    // ...do work...
    await job.log(`Completed processing`)
  }
}
```

The `Job` parameter is optional and always comes last. Job logs are accessible through BullMQ dashboards and can be retrieved programmatically.

## Worker Configuration

The full worker configuration lives in `conf/initializers/workers.ts`:

```typescript
import os from 'os'
import { PsychicAppWorkers } from '@rvoh/psychic-workers'
import Redis from 'ioredis'
import AppEnv from '../AppEnv.js'

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
      // Add named workstreams here as needed
    ],

    // Any instance can push onto the queue
    defaultQueueConnection: new Redis({
      host: AppEnv.string('BG_JOBS_REDIS_HOST', { optional: true }) || 'localhost',
      port: AppEnv.integer('BG_JOBS_REDIS_PORT', { optional: true }) || 6379,
    }),

    // Only establish the worker connection on instances that process jobs
    defaultWorkerConnection: !AppEnv.boolean('WORKER_SERVICE')
      ? undefined
      : new Redis({
          host: AppEnv.string('BG_JOBS_REDIS_HOST', { optional: true }) || 'localhost',
          port: AppEnv.integer('BG_JOBS_REDIS_PORT', { optional: true }) || 6379,
          maxRetriesPerRequest: null,
        }),
  })

  workersApp.on('workers:shutdown', () => {
    // Add cleanup logic here
  })
}
```

## Testing Workers

### Default: Immediate Invocation

By default, backgrounded methods execute immediately and synchronously in tests — no Redis or worker infrastructure needed:

```typescript
// This runs the method immediately in tests
await IntercomSyncService.syncUser(user)
```

### Manual Mode

For tests that need to verify queuing behavior or test timing, switch to manual invocation:

```typescript
import { WorkerTestUtils, PsychicAppWorkers } from '@rvoh/psychic-workers'

beforeEach(async () => {
  PsychicAppWorkers.getOrFail().set('testInvocation', 'manual')
})

afterEach(async () => {
  PsychicAppWorkers.getOrFail().set('testInvocation', 'automatic')
})

it('processes the job', async () => {
  await IntercomSyncService.syncUser(user)  // Queued only, not executed

  await WorkerTestUtils.work()              // Process all queued jobs

  // Assert on side effects after jobs have run
})
```

### Test Utilities (for manual mode)

```typescript
import { WorkerTestUtils } from '@rvoh/psychic-workers'

await WorkerTestUtils.clean()                          // Reset all queues between test runs
await WorkerTestUtils.work()                           // Work off all queued jobs
await WorkerTestUtils.workScheduled()                  // Run all delayed/scheduled jobs now
await WorkerTestUtils.workScheduled({ for: User })     // Run delayed jobs for a specific class
await WorkerTestUtils.workScheduled({ queue: 'cool' }) // Run delayed jobs for a specific queue
```

## Plugin Registration

Workers is a Psychic plugin registered during app initialization:

```typescript
// initializePsychicApp.ts
import { PsychicAppWorkers } from '@rvoh/psychic-workers'
import workersConf from '../../conf/workers.js'

export default async function initializePsychicApp(opts = {}) {
  const psychicApp = await PsychicApp.init(psychicConf, dreamConf, opts)
  await PsychicAppWorkers.init(psychicApp, workersConf)
  return psychicApp
}
```
