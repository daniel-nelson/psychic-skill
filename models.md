# Dream Models - Detailed Reference

## Base Model

Every project has an `ApplicationModel` that extends Dream:

```typescript
import Dream from '@rvoh/dream'
import { DB } from '@src/types/db.js'
import schema from '@src/types/dream.js'
import { connectionTypeConfig, globalTypeConfig } from '@src/types/dream.globals.js'

export default class ApplicationModel extends Dream {
  declare public DB: DB
  public override get schema() { return schema }
  public override get connectionTypeConfig() { return connectionTypeConfig }
  public override get globalTypeConfig() { return globalTypeConfig }
}
```

## Column Types

Use `DreamColumn<ModelClass, 'columnName'>` for all database-backed columns. The type is inferred from the database schema via Kysely codegen.

```typescript
public id: DreamColumn<User, 'id'>
public email: DreamColumn<User, 'email'>
public createdAt: DreamColumn<User, 'createdAt'>
public updatedAt: DreamColumn<User, 'updatedAt'>
```

## Decorator Setup

Each model creates its own decorator instance:

```typescript
import { Decorators } from '@rvoh/dream'
const deco = new Decorators<typeof MyModel>()
```

## Associations - Full Reference

### BelongsTo

Foreign key lives on THIS model. You must also declare the FK column.

```typescript
@deco.BelongsTo('User')
public user: User
public userId: DreamColumn<Post, 'userId'>

// Optional (nullable FK)
@deco.BelongsTo('User', { optional: true })
public approver: User | null
public approverId: DreamColumn<Post, 'approverId'>

// Custom FK name
@deco.BelongsTo('User', { on: 'authorId' })
public author: User
public authorId: DreamColumn<Post, 'authorId'>

// Polymorphic BelongsTo
@deco.BelongsTo(['Host', 'Place', 'Room'], { polymorphic: true, on: 'localizableId' })
public localizable: Host | Place | Room
public localizableType: DreamColumn<LocalizedText, 'localizableType'>
public localizableId: DreamColumn<LocalizedText, 'localizableId'>
```

### HasMany

Foreign key lives on the ASSOCIATED model.

```typescript
@deco.HasMany('Post')
public posts: Post[]

// With cascade delete
@deco.HasMany('Post', { dependent: 'destroy' })
public posts: Post[]

// With order
@deco.HasMany('Room', { order: 'position' })
public rooms: Room[]

// With conditions
@deco.HasMany('Post', { and: { published: true } })
public publishedPosts: Post[]

// Through (many-to-many)
@deco.HasMany('Host', { through: 'hostPlaces' })
public hosts: Host[]

// Nested through
@deco.HasMany('PostComment', { through: 'posts', source: 'comments' })
public postComments: PostComment[]

// Polymorphic
@deco.HasMany('LocalizedText', { polymorphic: true, on: 'localizableId', dependent: 'destroy' })
public localizedTexts: LocalizedText[]

// With distinct
@deco.HasMany('Tag', { through: 'postTags', distinct: true })
public uniqueTags: Tag[]
```

### HasOne

Like HasMany but returns single instance.

```typescript
@deco.HasOne('Profile')
public profile: Profile

// With condition using passthrough
@deco.HasOne('LocalizedText', {
  polymorphic: true,
  on: 'localizableId',
  and: { locale: DreamConst.passthrough },
})
public currentLocalizedText: LocalizedText

// Through
@deco.HasOne('CompositionAsset', { through: 'mainComposition' })
public mainCompositionAsset: CompositionAsset
```

## Hooks - Full Reference

### Before Hooks (run in same transaction, before DB write)

```typescript
@deco.BeforeCreate()
public setDefaults(this: Place) {
  if (!this.status) this.status = 'draft'
}

@deco.BeforeUpdate()
public trackChanges(this: Place) { ... }

@deco.BeforeSave()  // Runs on BOTH create and update
public normalizeEmail(this: User) {
  if (this.email) this.email = this.email.toLowerCase()
}

@deco.BeforeDestroy()
public validateCanDelete(this: Place) { ... }

// Conditional hooks
@deco.BeforeSave({ ifChanging: ['email'] })
public validateNewEmail(this: User) { ... }
```

### After Hooks (run in same transaction, after DB write)

```typescript
@deco.AfterCreate()
public createDefaults(this: User) {
  await this.createAssociation('profile', {})
}

@deco.AfterUpdate({ ifChanged: ['status'] })
public logStatusChange(this: Place) { ... }

@deco.AfterSave()  // Both create and update
public updateCache(this: Place) { ... }

@deco.AfterDestroy()
public cleanupReferences(this: Place) { ... }
```

### After Commit Hooks (run AFTER transaction commits)

**CRITICAL**: Always use commit hooks when sending models to background jobs. This prevents race conditions where the job runs before the transaction commits.

```typescript
@deco.AfterCreateCommit()
public sendWelcomeEmail(this: User) {
  await EmailService.background('sendWelcome', this.id)
}

@deco.AfterUpdateCommit({ ifChanged: ['status'] })
public notifyStatusChange(this: Place) {
  await NotificationService.background('statusChanged', this.id)
}

@deco.AfterSaveCommit()
public invalidateCache(this: Place) { ... }

@deco.AfterDestroyCommit()
public removeFromSearchIndex(this: Place) { ... }
```

## Validations

```typescript
@deco.Validates('presence')
public name: DreamColumn<Place, 'name'>

@deco.Validates('length', { min: 4, max: 64 })
public email: DreamColumn<User, 'email'>

@deco.Validates('contains', '@')
public email: DreamColumn<User, 'email'>

@deco.Validates('numericality', { min: 0, max: 100 })
public volume: DreamColumn<User, 'volume'>

@deco.Validates('inclusion', { in: ['active', 'inactive'] })
public status: DreamColumn<User, 'status'>

@deco.Validates('exclusion', { in: ['admin', 'root'] })
public username: DreamColumn<User, 'username'>

@deco.Validates('requiredBelongsTo')
public user: User  // Validates the association exists

// Custom validation method
@deco.Validate()
public validateWeightConsistency(this: Sandbag) {
  if (this.weight && this.weightKgs) {
    this.errors['weight'] = ['cannot include both weight and weightKgs']
  }
}
```

Checking validity:
```typescript
const user = User.new({ email: 'bad' })
if (user.isInvalid) {
  console.log(user.errors) // { email: ['must contain @'] }
}
```

## Scopes

```typescript
// Named scope - called explicitly
@deco.Scope()
public static active(query: Query<Place>) {
  return query.where({ status: 'active' })
}

// Default scope - applied to ALL queries automatically
@deco.Scope({ default: true })
public static hideDeleted(query: Query<Place>) {
  return query.where({ deletedAt: null })
}

// Usage
const places = await Place.scope('active').all()

// Bypass default scopes
const allPlaces = await Place.removeAllDefaultScopes().all()
const withDeleted = await Place.removeDefaultScope('hideDeleted').all()
```

## Query Operators

```typescript
import { ops } from '@rvoh/dream'

// Comparison
ops.greaterThan(value)
ops.greaterThanOrEqualTo(value)
ops.lessThan(value)
ops.lessThanOrEqualTo(value)
ops.equal(value)
ops.not.equal(value)

// Collections
ops.in([1, 2, 3])
ops.not.in([1, 2, 3])

// String matching
ops.like('%pattern%')
ops.ilike('%PATTERN%')     // Case-insensitive
ops.match('^[A-Z]')        // Regex
ops.match('^[A-Z]', { caseInsensitive: true })

// Fuzzy search (requires pg_trgm extension + GIN index)
ops.similarity('search term')
ops.wordSimilarity('search term')
ops.similarity('term', { score: 0.2 })

// Array containment
ops.any(5)  // Array contains value

// NULL checks
await Model.where({ field: null }).all()      // IS NULL
await Model.whereNot({ field: null }).all()    // IS NOT NULL
```

## Special Decorators

```typescript
// Soft Delete - adds default scope excluding deleted records
@SoftDelete()
export default class Place extends ApplicationModel { ... }

// Sortable - maintains position within scope
@deco.Sortable({ scope: 'place' })
public position: DreamColumn<Room, 'position'>
// Requires deferrable unique constraint in migration

// Encrypted - auto encrypt/decrypt
@deco.Encrypted()
public phone: DreamColumn<User, 'encryptedPhone'>
// DB column is 'encrypted_phone', model property is 'phone'

// Virtual - not stored in DB, not sent to DB on save
@deco.Virtual('string')
public password: string | undefined

@deco.Virtual('integer')
public age: number

@deco.Virtual(['string', 'null'])
public computedField: string | null

// All Virtual types: 'boolean', 'date-time', 'date', 'integer', 'null', 'number',
// 'string', 'decimal', 'json', and array variants: 'boolean[]', 'date-time[]',
// 'date[]', 'decimal[]', 'integer[]', 'number[]', 'string[]'
// Use arrays for nullable: ['string', 'null']

// STI - Single Table Inheritance child
@STI(Room)
export default class Bedroom extends Room { ... }
```

## Dirty Tracking

```typescript
const user = await User.findOrFail(id)
user.name = 'New Name'

user.isDirty()                    // true
user.isClean()                    // false
user.changedAttributes()          // { name: 'New Name' }
user.hasChanges('name')           // true
user.hasChanges('email')          // false
```

## Batch Processing

```typescript
// Process records in batches (default batch size: 1000)
await User.findEach(async (user) => {
  await user.update({ processedAt: DateTime.now() })
})

// With custom batch size
await User.findEach({ batchSize: 100 }, async (user) => { ... })

// Pluck in batches
await User.pluckEach('email', async (email) => {
  await sendEmail(email)
})
```

## Transactions

```typescript
// Class-level transaction
await ApplicationModel.transaction(async (txn) => {
  const user = await User.txn(txn).create({ email: 'test@test.com' })
  const post = await Post.txn(txn).create({ userId: user.id, title: 'Test' })
  // If any error thrown, entire transaction rolls back
})

// Instance-level transaction
await user.transaction(async (txn) => {
  await user.txn(txn).update({ status: 'active' })
  await user.txn(txn).createAssociation('profile', {})
})
```

## Date/Time

**NEVER use JavaScript `Date`. Always use Dream's DateTime and CalendarDate (powered by Luxon).**

```typescript
import { CalendarDate, DateTime } from '@rvoh/dream'

// DateTime for timestamp columns
const now = DateTime.now()
const specific = DateTime.fromISO('2024-01-15T10:30:00')
const future = now.plus({ days: 7, hours: 3 })

// CalendarDate for date-only columns
const today = CalendarDate.today({ zone: 'America/New_York' })
const birthday = CalendarDate.fromISO('1990-05-15')
const nextWeek = today.plus({ days: 7 })
```

