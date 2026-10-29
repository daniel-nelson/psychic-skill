# Dream Utilities

Dream exports utility functions at `@rvoh/dream/utils`. Prefer these over lodash or hand-rolled equivalents. Consult the TSDocs on each function for exact signatures and behavior.

```typescript
import { camelize, compact, groupBy, sort, uniq /* etc. */ } from '@rvoh/dream/utils'
```

## Available Utilities

- **Encrypt** - Encryption/decryption (AES-256-GCM)
- **camelize** - Convert to camelCase
- **capitalize** / **uncapitalize** - Capitalize or uncapitalize first letter
- **cloneDeepSafe** - Deep clone
- **compact** - Remove `null`/`undefined` from an array or object (keeps `false`, `0`, `''`)
- **groupBy** - Group array items by key
- **hyphenize** - Convert to hyphen-case
- **intersection** - Array intersection
- **isEmpty** - Check if value is empty
- **normalizeUnicode** - Normalize unicode strings
- **pascalize** - Convert to PascalCase
- **percent** - Calculate percentage
- **range** / **Range** - A bounded or open-ended range (`range(begin, end?, excludeEnd?)`) for `where` predicates. See [querying.md — Range Predicates](querying.md#range-predicates).
- **round** - Round numbers
- **sanitizeString** - Escape `< > & " ' / \` as literal `\uXXXX` text (`null`/`undefined` pass through)
- **snakeify** - Convert to snake_case
- **sort** / **sortBy** - Sort arrays
- **sortObjectByKey** / **sortObjectByValue** - Sort object entries
- **uniq** - Deduplicate array
