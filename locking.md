# Locking and Concurrent Writes

Every production deployment runs the same code on more than one server, so every write races some
other write. That on its own is not a reason to lock.

## Most writes need no lock

Two requests updating different attributes of the same record both land. Two requests racing to set
the *same* attribute resolve to a single winner — and they resolve to a single winner whether or not
you lock. A plain update is the right tool for both:

```typescript
await booking.update({ notes })
await Booking.where({ place }).update({ cleaningFee: 4500 })
```

Uniqueness and interval-overlap invariants are also not a locking problem. Enforce them with a
database constraint — a unique index, or a PostgreSQL exclusion constraint for overlapping bookings —
and let the offending write fail. See [querying.md — Range Predicates](querying.md#range-predicates)
for the overlap case and [models.md — Find-or-Create and Upsert Methods](models.md#find-or-create-and-upsert-methods)
for the create race.

## When a lock is warranted

A lock earns its place when **the value being written depends on a value just read**, and another
writer changing that value in between would produce a wrong result. Claiming a record out of a state
is the canonical case: a booking is confirmed only if it is still pending; a payout is released only
if it is still unreleased. Two servers running that code at the same instant must not both succeed.

Dream expresses this as a **guarded (compare-and-set) write**: a record is written only if it still
matches the query at the moment an exclusive row lock on it is acquired. Records are claimed in
batches. Each batch opens its own transaction, re-selects that batch's rows `FOR UPDATE`, re-checks
the query's conditions against the locked rows, then writes and commits. A record another transaction
has already moved out of the query drops out of the locked read and is left alone.

The return value is the count of records actually claimed, so the caller detects a lost race by
comparing it against what it expected.

## `update(attributes, { lock: true })`

Reach for this when the guard is expressible in the query's `where` clause and every claimed record
gets the same attributes.

```typescript
const claimed = await Booking.where({ id: booking.id, status: 'pending' })
  .update({ status: 'confirmed' }, { lock: true })

if (claimed === 0) return this.conflict()   // another request confirmed it first
```

`status: 'pending'` is the guard; `status: 'confirmed'` is the write. Each claimed record is written
through the ordinary per-instance update, so hooks, validations, and custom setters run exactly as
they would on `booking.update(...)`.

What this form cannot express: a guard the database can't compare — an `@deco.Encrypted` property has
no queryable column, and re-encrypts to fresh ciphertext on every write — and attributes that differ
per record.

## `update(callback, { lock: true })`

Reach for this when the guard, or the new value, has to be computed from the record itself. The
callback runs once per claimed record, under that record's row lock, inside the batch's transaction,
and returns the attributes to write, or `undefined` to skip that record.

```typescript
// Booking#doorCode is @deco.Encrypted: the plaintext property is virtual and the
// ciphertext is fresh on every write, so there is no column for a `where` to compare
const rotated = await Booking.where({ id: booking.id }).update(
  booking => (booking.doorCode === issuedCode ? { doorCode: newCode } : undefined),
  { lock: true }
)

if (rotated === 0) return this.conflict()   // the code had already been rotated
```

The guard compares a value the database cannot see, so it has to run in the callback, against the
freshly locked read. A new value derived from the old one is the other case only this form reaches.

`lock` is mandatory with a callback — omit it and Dream throws
`MissingRequiredLockOptionForUpdateCallback` — so every call site states out loud whether it takes
locks. Only the attributes the callback returns are written; assignments made on the record it
receives are discarded. The count is the number of records the callback returned attributes for;
skipped records are not counted.

**The callback must not touch the database.** The record it receives is not bound to the batch's
transaction, so a write through it (`await booking.update(...)`) goes out on a different pooled
connection and blocks on the batch's own row locks while the batch waits on the callback — a cycle
across two connections that Postgres's deadlock detector cannot see, so it hangs rather than aborting.
Reads through the record (`reload`, `associationQuery`) likewise run on a separate connection and
snapshot. Derive the attributes and return them; Dream performs the write inside the batch's
transaction.

## `destroy({ lock: true })`

The same guarantee for removal:

```typescript
const cancelled = await Booking.where({ place, status: 'pending' }).destroy({ lock: true })
```

Each batch re-selects its rows `FOR UPDATE OF` inside its own transaction and destroys them, so a
record another transaction has moved out of the query is left alone. Destroy hooks and the
`dependent: 'destroy'` cascade run per record, and on a `@SoftDelete()` model this is still a soft
delete — see [soft-delete.md](soft-delete.md). `reallyDestroy` takes the same option.

`lock` lives on the query, never on an instance. Destroying an already-loaded instance has no
select-then-destroy window to close, so there is no instance-level equivalent and none is needed.

## What a lock costs

- **`batchSize` defaults to 10**, not the 1000 an unlocked batch uses, and the small default is
  deliberate. Every row in a batch stays locked for the whole batch — each record's hooks, each
  record's `dependent: 'destroy'` cascade, and, on the callback form, the query's `preload` queries,
  which are fetched inside the lock window. An oversized batch blocks unrelated writers touching those
  rows and surfaces as timeouts elsewhere in the application, very hard to trace back here. Lower it
  further for models with expensive hooks or deep cascades.
- **A batch waits, without bound, for whoever holds the row.** The locked read is a plain
  `FOR UPDATE` — no `NOWAIT`, no `SKIP LOCKED`. If another transaction holds a lock on one of the
  batch's rows, the call blocks until that transaction ends, holding its own transaction and pooled
  connection open the whole time. Set a `lock_timeout` on the connection where a stuck batch failing
  is better for the application than a stuck batch hanging.
- **The guarantee is per batch, not set-wide.** A run spanning more than one batch can win a race in
  batch 1 and lose one in batch 2; nothing holds the whole matched set still. Threading a transaction
  onto the query makes the run all-or-nothing and holds every batch's locks until you commit, but it
  does not extend the compare-and-set forward: under READ COMMITTED each batch takes a fresh snapshot,
  so a record moved out of the query between batches still drops out of the later batch's locked read.
  A genuinely set-wide guarantee needs REPEATABLE READ or SERIALIZABLE.

  ```typescript
  await ApplicationModel.transaction(async txn => {
    await Booking.txn(txn).where({ place, status: 'pending' }).destroy({ lock: true })
  })
  ```
- **Thread the transaction with `.txn(txn)`; wrapping alone is worse than not wrapping.**
  `ApplicationModel.transaction(...)` establishes no ambient transaction — it hands your callback a
  `txn`. Without `.txn(txn)`, each batch opens its own transaction on a different connection; if the
  surrounding transaction has already written to any of the matched rows, every batch blocks on locks
  that transaction holds while it waits on the run — again a two-connection cycle the deadlock
  detector cannot see, so it hangs.
- **Above READ COMMITTED the loss is loud.** The drop-out-of-the-locked-read behavior described here
  is READ COMMITTED, Postgres's default. Under REPEATABLE READ or SERIALIZABLE the same race raises a
  serialization failure instead of a quieter lower count. Both are safe.
- **A `limit` or `offset` on the query throws** `BatchingIncompatibleWithLimitOrOffset`. A batched run
  would re-apply either inside every batch window rather than bounding the run as a whole.
- **An unlocked update racing a deleter aborts mid-run.** Without `lock`, a matched record another
  transaction destroys between a batch's read and that record's write throws: earlier batches stay
  committed and no count is returned. Where deleters race the update, `lock: true` holds claimed rows
  under their locks until they are written.

**If you are writing a large set and do not need compare-and-set, do not pass `lock`.** It is slower
than the plain form by construction.
