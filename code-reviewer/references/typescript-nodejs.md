# TypeScript and Node.js Review Guide

Patterns specific to TypeScript and Node codebases. Apply these alongside the general correctness review.

## TypeScript

### Type safety

- **High**: `any` added where a concrete type is known. Narrow exceptions: third-party untyped libs, genuine dynamic payloads being validated at runtime.
- **High**: `as SomeType` casts that bypass validation (especially `as unknown as T`). Require evidence the runtime shape matches, or a schema validator.
- **Medium**: non-null assertions (`!`) on values that could actually be null in practice. Flag with the specific scenario where null is possible.
- **Medium**: `@ts-ignore`, `@ts-expect-error`, `@ts-nocheck` added without a comment explaining why
- **Low**: `Object` or `{}` used as types (they mean "anything non-null", not "an object")
- **Low**: `Function` type used instead of a specific signature

### Generics and inference

- **Medium**: generic parameter that's never constrained or used meaningfully (ceremonial generics)
- **Low**: `Array<T>` vs `T[]` inconsistency within the same file
- **Info**: could use `satisfies` instead of type assertion for literal types

### Discriminated unions and exhaustiveness

- **High**: `switch` on a discriminated union without a `default` that asserts `never` — future variants silently fall through
- **Medium**: `if/else if` chain on a union type that doesn't cover all variants

### Strict mode checks

If `tsconfig.json` has strict mode off, note it as Info but don't penalize individual findings that strict mode would have caught. If strict mode is on, these become High:
- Implicit `any` parameters
- Uninitialized class fields
- Promises returned but not awaited (`@typescript-eslint/no-floating-promises`)

## Async and promises

- **Critical**: missing `await` on a promise whose result or side effect is relied on. Especially dangerous with writes: `db.save(user)` without await returns immediately, the caller sees success before the write lands.
- **High**: unhandled promise rejections — `.then()` chain without `.catch()`, or top-level `async` function called without error handling
- **High**: `async` function in a place that expects sync (e.g., array `.filter(async ...)` always returns truthy promises, not filtering by the actual boolean)
- **High**: `Promise.all` used where one rejection should not cancel others — use `Promise.allSettled`
- **High**: sequential `await` in a loop where operations are independent — use `Promise.all`. Conversely, parallelizing where order or rate limiting matters is its own bug.
- **Medium**: `async` function that doesn't actually await anything — unnecessary wrapping
- **Medium**: creating promises inside a hot loop without awaiting, causing unbounded concurrency
- **Medium**: `setTimeout`/`setInterval` callbacks marked `async` without error handling — rejections become unhandled

## Error handling

- **High**: `catch (e) { }` empty catch blocks swallowing errors
- **High**: `catch (e)` where `e` is logged and then control continues as if nothing happened (silent failure on a path that needed to abort)
- **High**: throwing non-Error values (`throw "string"`, `throw { code: 1 }`) — loses stack traces and breaks downstream handlers
- **Medium**: catching `Error` too broadly when only a specific failure was expected (hides new bug classes)
- **Medium**: `try/catch` wrapping a block that includes lines that don't actually throw, obscuring the real error source
- **Low**: error messages that don't include enough context to debug (e.g., "Failed" with no ID, no operation name)

## Node.js runtime

### Event loop and performance
- **High**: synchronous fs calls (`readFileSync`, `writeFileSync`) in request handlers — blocks the event loop
- **High**: CPU-heavy work in the main thread for server code — use worker threads or offload
- **Medium**: unbounded in-memory accumulation (reading a whole file/stream into an array) for request handlers

### Streams
- **High**: pipe chain without error handling — a single stream error crashes the process. Use `stream.pipeline` instead of `.pipe().pipe()`.
- **Medium**: stream backpressure ignored (not checking `write()` return value in a custom producer)

### Process and signals
- **Medium**: no `SIGTERM` handler for graceful shutdown in long-running servers
- **Medium**: `process.exit()` called from library code (should throw or return error)

### Environment variables
- **High**: `process.env.FOO` dereferenced without checking it exists (`.toString()` on undefined crashes)
- **Medium**: env var parsed as number/bool without validation (`parseInt(process.env.PORT)` returns NaN silently)

## Express / Fastify / Node HTTP servers

- **Critical**: async handler that throws — Express 4 does not catch these automatically. Either use `express-async-errors`, wrap with a try/catch, or migrate to Express 5.
- **High**: route handler that does DB write without input validation
- **High**: CORS wildcarded on credentialed endpoints
- **Medium**: middleware ordering — body parser after route handler, auth middleware after the protected route
- **Medium**: response sent but handler continues running (`res.send(x); doMore()` — `doMore` still runs)

## Next.js

### App Router specifics
- **High**: `use server` action without authentication check — server actions are callable from anywhere
- **High**: secrets referenced with `NEXT_PUBLIC_` prefix (exposed to client bundle)
- **High**: `fetch` in a Server Component without explicit `cache` behavior when caching matters
- **Medium**: `'use client'` added to a component that doesn't need interactivity (bundle bloat)
- **Medium**: `revalidatePath`/`revalidateTag` missing after mutations in server actions

### Pages Router specifics
- **High**: `getServerSideProps` leaking server-only data into props (returned object ships to client)
- **Medium**: `getStaticProps` for data that should be dynamic

## Database layer

### Prisma
- **High**: `prisma.$queryRawUnsafe` with user input — use `$queryRaw` with template literal (parameterized)
- **Medium**: N+1 in a loop — missing `include` or batch with `findMany({ where: { id: { in: ids } } })`
- **Medium**: `findFirst` when `findUnique` would do (index usage)

### Drizzle
- **Medium**: `sql.raw()` with user input — use `sql` tagged template for parameterization
- **Low**: missing indexes implied by query patterns

### Raw pg/mysql2
- **Critical**: string concatenation in queries — always use placeholders (`$1` for pg, `?` for mysql2)
- **High**: connection pool not bounded
- **Medium**: transactions not wrapping multi-statement operations that need atomicity

## Testing

- **High**: new feature added without any test coverage (in repos that have tests as a norm)
- **Medium**: test uses `expect(x).toBeTruthy()` where a specific equality check would catch more bugs
- **Medium**: test mocks everything so thoroughly it doesn't actually test the logic
- **Low**: test description vague ("it works") — should name the specific behavior

Skip all of these for pure refactors, type-only changes, or doc changes.

## React (when Next.js or React is in the stack)

- **High**: effect with missing dependencies that cause stale closures
- **High**: effect that runs on every render due to unstable deps (object literal in deps array)
- **High**: state update in render body (infinite loop)
- **Medium**: key prop missing or using array index as key on a reorderable list
- **Medium**: uncontrolled/controlled input mixing
- **Low**: inline function passed to a memoized child, defeating memoization

## Module and package hygiene

- **Medium**: circular imports — note them, they usually indicate layering issues
- **Medium**: deep relative imports (`../../../../foo`) — suggest path aliases
- **Low**: default exports where named exports would aid refactoring
- **Low**: barrel files (`index.ts` re-exporting everything) that hurt tree-shaking

## Logging

- **High**: logging secrets or PII (covered in security checklist, worth re-mentioning)
- **Medium**: `console.log` left in production code paths (Node servers should use a proper logger)
- **Low**: log level misuse — `error` used for expected conditions, `info` for what should be `debug`

## Common TypeScript/Node pitfalls to watch for

- `Array.prototype.sort` without comparator (sorts by string representation — `[10, 2, 1].sort()` gives `[1, 10, 2]`)
- `JSON.parse` on untrusted input without try/catch
- `JSON.stringify` on objects with BigInt, circular refs, or Dates (Dates become strings, one-way)
- `Object.keys(obj).map(...)` when `Object.entries` would be cleaner
- Date math done with `Date` objects instead of a library (timezone bugs)
- `new Date(dateString)` where dateString format isn't ISO 8601 (browser-dependent parsing)
- Mutating function arguments unexpectedly
- Comparing objects/arrays with `===` expecting value equality
