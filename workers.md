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
- **NEVER pass model data as background job arguments.** Almost always pass only the model's ID and look up the record in the implementation method. This applies to any data stored in a Dream model. Passing model data as arguments has serious downsides: it bloats Redis memory (a full JSON payload vs. a single ID string), loses all type information when serialized to JSON (e.g., Dream date/time objects become plain strings, enums become untyped values), and creates a snapshot that is immediately stale if the record is updated after the job is queued. The only arguments to `this.background(...)` should be IDs and simple scalar values (strings, numbers, booleans) that are not sourced from model columns. For model instance methods, `ApplicationBackgroundedModel` handles this automatically by storing only the primary key.
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

### Imperative Form: Never Enqueue from Inside an Active Transaction

The same race applies to **services** that take a `txn` parameter and call `background()` directly. BullMQ is not transactional with Postgres — anything you enqueue from inside an open transaction is visible to workers immediately, but the records you just created via `txn(txn).create(...)` are not yet visible to other connections. The worker dequeues, calls `Model.find(id)`, gets `null`, and silently returns success. The job is marked complete, no retry, no alerting, data stranded.

```typescript
// WRONG — bgjob races the transaction commit
await ApplicationModel.transaction(async txn => {
  const photo = await Photo.txn(txn).create({ ... })
  await PhotoProcessingService.background('process', photo.id)
  // ↑ Redis sees the job NOW; Postgres won't see `photo` until commit
})

// RIGHT — enqueue after commit
let photoId: string
await ApplicationModel.transaction(async txn => {
  const photo = await Photo.txn(txn).create({ ... })
  photoId = photo.id
})
await PhotoProcessingService.background('process', photoId)

// ALSO RIGHT — let the model's @AfterCreateCommit hook do the enqueue
// (don't ALSO do it manually inside the txn)
```

**Red flag:** if a service takes a `txn` parameter AND its method body contains `background(`, it is almost always a bug.

## Never Rescue Exceptions Inside Backgrounded Services

Inside any class extending `ApplicationBackgroundedService` or `ApplicationBackgroundedModel`, the bar for adding a `try/catch` is much higher than [SKILL.md rule #13](SKILL.md). The default is **no catch, ever**, and you need a named, justified reason to deviate.

BullMQ relies on thrown exceptions to detect failure. A caught-and-not-rethrown error inside a backgrounded method causes the job to be marked successful, which means:

- No failed-job marker in BullMQ
- No automatic retry
- No alerting hook fires
- The work is silently lost and only discoverable by reading worker logs

```typescript
// WRONG — log-and-continue inside a backgrounded service
private static async _importAll(records: Record[]) {
  for (const record of records) {
    try {
      await this.importOne(record)
    } catch (error) {
      console.error('Failed:', error)  // ← BullMQ never sees this; job marks success
    }
  }
}

// RIGHT — let the first failure propagate, let BullMQ retry the whole job
private static async _importAll(records: Record[]) {
  for (const record of records) {
    await this.importOne(record)
  }
}

// ALSO RIGHT — fan out to per-record jobs, each retried independently by BullMQ
public static async importAll(records: Record[]) {
  for (const record of records) {
    await this.background('_importOne', record.id)
  }
}
```

If your motivation for adding a catch is "I want to be resilient to one bad input out of many", the right answer is **separate background jobs per input**. A try/catch loop inside a single job that fakes per-iteration success gives up the retry property entirely.

## Fanning Out Background Jobs for Very Large Record Sets

When you need to background work across a very large number of records (hundreds of thousands to millions), don't enqueue all individual jobs up front. Creating a million jobs at once has several problems:

- **Redis memory pressure** from holding a million job payloads at once
- **Interruption risk** — the enqueuing loop itself can be killed by a deployment, `SIGTERM`, or a Node process crash, and if it restarts from the beginning it will create duplicate jobs
- **Queue observability collapses** — dashboards become unusable

The idiomatic pattern is a **two-level fan-out** using `pluckEach` and priority levels:

1. A kickoff job uses `pluckEach` to pluck IDs in batches (default batch size is 1000).
2. For each batch, it enqueues an **expander job** (priority `not_urgent`) with just that batch of IDs.
3. Each expander job iterates its batch and enqueues an **individual worker job** (priority `default`) per ID.
4. Each individual worker job loads the record and does the real work.

Because expanders run at `not_urgent` priority, they only claim worker slots when no `default`-priority individual jobs are pending. With 10 workers, that means at most ~10 batches are expanded at a time (producing ~10,000 individual jobs in flight), and the individual jobs are drained before more batches are expanded. The queue depth stays bounded regardless of the total record count.

```typescript
export default class ReprocessAllPhotosService extends ApplicationBackgroundedService {
  public static override get backgroundJobConfig() {
    return { priority: 'not_urgent' as const }
  }

  // Step 1: kickoff — iterates the table in batches and enqueues one expander per batch
  public static async reprocessAll() {
    await this.background('_reprocessAll')
  }

  public static async _reprocessAll() {
    let batch: string[] = []
    await Photo.pluckEach('id', async (id: string) => {
      batch.push(id)
      if (batch.length >= 1000) {
        await this.background('_expandBatch', batch)
        batch = []
      }
    })
    if (batch.length > 0) await this.background('_expandBatch', batch)
  }

  // Step 2: expander — fans the batch into individual high-priority jobs
  public static async _expandBatch(ids: string[]) {
    for (const id of ids) {
      await PhotoProcessingService.processOne(id)
    }
  }
}

export default class PhotoProcessingService extends ApplicationBackgroundedService {
  public static override get backgroundJobConfig() {
    return { priority: 'default' as const }
  }

  // Step 3: individual worker — loads the record and does the actual work
  public static async processOne(id: string) {
    await this.background('_processOne', id)
  }

  public static async _processOne(id: string) {
    const photo = await Photo.find(id)
    if (!photo) return
    // ...do the real work
  }
}
```

Key points:

- **`pluckEach` selects only the `id` column** — no hydration, minimal memory.
- **Priorities create natural backpressure.** Expanders (`not_urgent`) yield to individual jobs (`default`), so the in-flight count of individual jobs never exceeds roughly `worker_count * concurrency * batch_size`.
- **If the kickoff is interrupted,** only the expanders already enqueued will run, and if the kickoff job itself is retried it will re-pluck from the beginning — but because each individual job is independent and idempotent (via the `find`/early-return pattern), re-runs are safe.
- **Individual jobs still follow the standard rule of passing IDs, not model instances.** Hydrate inside `_processOne`.

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

### Scheduled services are thin orchestrators, not workers

A scheduled method should do almost nothing itself: select what needs to happen now and **fan out to backgrounded services** that do the real work. Heavy lifting inside a scheduled method is a design smell. The goal is to keep the number of permanently-registered schedulers in the BullMQ "Delayed" queue to a small fixed set — typically just `hourly`, `daily`, and `weekly` — each of which kicks off whatever backgrounded jobs that cadence requires. Three clean schedulers fanning out to dedicated services is the target shape; a sprawling list of per-task schedulers is not.

### A class is either scheduled OR backgrounded — never both

`ApplicationScheduledService` exposes `schedule()` but **not** `background()`. `ApplicationBackgroundedService` exposes `background()` but **not** `schedule()`. They are sibling base classes; a service extends one or the other. This is a hard architectural constraint, not a style choice:

- **A scheduled method cannot enqueue background jobs from within itself.** To fan out, the scheduled (orchestrator) service calls the public entry method of a *separate* backgrounded (worker) service.
- **Import direction is one-way:** the orchestrator imports the worker, never the reverse. A mutual import is a cycle and risks class-init / `globalName`-registration ordering problems.

```typescript
// Orchestrator — extends ApplicationScheduledService (has schedule(), no background())
export default class ScheduledJobs extends ApplicationScheduledService {
  public static async scheduleAllJobs() {
    await this.schedule('0 * * * *', 'hourly')
  }

  public static async hourly() {
    await ReconcileService.reconcileAll()   // delegate to a backgrounded service
  }
}

// Worker — extends ApplicationBackgroundedService (has background(), no schedule())
export default class ReconcileService extends ApplicationBackgroundedService {
  public static async reconcileAll() {
    for (const config of FORM_CONFIGS) {
      await this.background('_reconcileForm', config.key)  // one job per item
    }
  }
  public static async _reconcileForm(key: string) {
    // ...the actual work
  }
}
```

### Pitfall: `schedule()` keys by class+method only — looping silently drops jobs

`schedule()` registers the BullMQ job scheduler under the id `` `${globalName}:${method}` `` — **the args are not part of the id.** Internally it calls `upsertJobScheduler`, which is create-or-replace. So scheduling the same method more than once with different args does **not** create multiple schedulers; each call overwrites the last, and only the final args survive:

```typescript
// WRONG — registers exactly ONE scheduler (for FORM_CONFIGS[last]); the rest silently never run
for (const config of FORM_CONFIGS) {
  await ReconcileService.schedule('0 13 * * *', 'reconcileForm', config.key)
}
```

This fails silently: no error, no warning, the worker boots clean, and you only discover it when N−1 of N jobs never happened in production. To schedule N variants you need either **N distinct `(class, method)` pairs**, or — the idiomatic fix — **one scheduled method registered once that fans out at runtime**:

```typescript
// RIGHT — schedule the fan-out method ONCE; it loops at run time, not at schedule time
await ReconcileService.schedule('0 13 * * *', 'reconcileAll')
```

### Per-user cadence across time zones: schedule hourly, select by zone, fan out

"Daily" and "weekly" work for users spread across time zones is not a daily/weekly cron. Register the orchestrator on an **hourly** cron and have each run select the users whose *local* end-of-day (or end-of-week) falls in the current hour, then fan out one backgrounded job per selected user. Bake the time zone — and any end-of-week preference — into the query so the database efficiently returns just the relevant user IDs, rather than loading all users and filtering in code.

```typescript
// Orchestrator — runs every hour; figures out whose local end-of-day this hour is
export default class ScheduledJobs extends ApplicationScheduledService {
  public static async scheduleAllJobs() {
    await this.schedule('0 * * * *', 'endOfDayFanOut')   // hourly, despite being "daily" work
  }

  public static async endOfDayFanOut() {
    await EndOfDayService.fanOut(DateTime.now())
  }
}

// Worker — selects matching users by time zone, enqueues one job each
export default class EndOfDayService extends ApplicationBackgroundedService {
  public static async fanOut(now: DateTime) {
    // The query filters by timezone so only users whose local hour is end-of-day
    // are returned — selecting IDs only, never hydrating full records.
    await User.where({ endOfDayHourUtc: now.hour }).pluckEach('id', async (id: string) => {
      await this.background('_runEndOfDay', id)
    })
  }

  public static async _runEndOfDay(id: string) {
    const user = await User.find(id)
    if (!user) return
    // ...the per-user end-of-day work
  }
}
```

End-of-week works the same way, with the user's chosen end-of-week day folded into the query alongside the time zone, so the single hourly orchestrator covers every user's preference without a separate scheduler per variant.

### Scheduled and backgrounded methods run inline in tests

In `NODE_ENV=test`, both `schedule(...)` and `background(...)` invoke the underlying method immediately and synchronously (see the [Testing Workers](#testing-workers) section). A spec that calls either executes the work with no queue flush needed. The flip side: any environment guard inside the method (e.g. `if (serverEnvironment !== 'production') return`) also fires in tests, so a guarded method needs a `force`-style override to be exercised in a spec.

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

### Redis TLS

Pass `tls: {}` (or a full `tls.ConnectionOptions` object) to the `ioredis` constructor in both `defaultQueueConnection` and `defaultWorkerConnection` to opt into TLS:

```typescript
new Redis({
  host: AppEnv.string('BG_JOBS_REDIS_HOST'),
  port: AppEnv.integer('BG_JOBS_REDIS_PORT'),
  tls: {},
})
```

See the ioredis README and Node's `tls.connect` docs for the option shape and the connection-trust defaults.

### Retry and failure handling

The boilerplate `defaultJobOptions` ships `attempts: 20` with exponential `backoff` and `removeOnFail: 20000`. BullMQ's `failed` set is the dead-letter queue; the boilerplate defaults are the canonical retry-and-DLQ pattern. See the BullMQ docs for `QueueEvents` and bespoke DLQ patterns beyond this default.

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
