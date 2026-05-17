# Security Checklist

Apply this checklist to every review. Tune severity based on exposure (public endpoint vs internal utility).

## Injection

### SQL injection
- **Critical** if raw SQL with string concatenation or template literals containing user input: `` db.query(`SELECT * FROM users WHERE id = ${userId}`) ``
- **Critical** if `knex.raw()`, `sequelize.query()`, or `pg.query()` called with interpolated user input
- Safe patterns: parameterized queries (`db.query('SELECT ... WHERE id = $1', [userId])`), Prisma, Drizzle query builder, TypeORM query builder
- Note: template literals passed to tagged SQL helpers (e.g., Drizzle's `sql` tag, postgres.js) are parameterized. Do not flag these.

### Command injection
- **Critical** if `child_process.exec`, `execSync`, or `spawn` with shell:true receives user input directly in the command string
- Safer: `execFile`/`spawn` with `shell: false` and args as an array

### NoSQL injection
- **High** if MongoDB queries accept raw objects from user input without validation (e.g., `User.find(req.query)` letting a user send `{$ne: null}`)

### Path traversal
- **High** if user input flows into `fs.readFile`, `fs.writeFile`, or `path.join` without normalization + allowlist check
- Look for `../` sanitization that only strips once (vulnerable to `....//`)

### Prototype pollution
- **High** for recursive merge/clone functions that don't block `__proto__`, `constructor`, `prototype` keys
- Check libraries in use: older lodash, some merge utilities are known-vulnerable

### SSRF
- **High** if user-provided URLs flow into `fetch`, `axios`, `http.request` on a server without an allowlist. Especially dangerous if the server has access to cloud metadata endpoints (`169.254.169.254`) or internal networks.

## Authn and authz

- **Critical**: missing authentication check on a route that handles private data
- **Critical**: missing authorization check — authenticated but not checking the user owns the resource they're acting on (IDOR)
- **Critical**: JWT verified without checking signature (`jwt.decode` instead of `jwt.verify`)
- **Critical**: JWT signature algorithm not pinned (allows `alg: none` attacks on old libraries)
- **High**: secrets or keys hardcoded in source (look for patterns: `sk_`, `AKIA`, 40+ char hex/base64 strings assigned to const)
- **High**: password comparison with `==` or `===` instead of constant-time (`crypto.timingSafeEqual`, bcrypt.compare)
- **High**: session cookies missing `httpOnly`, `secure`, `sameSite`
- **High**: overly permissive CORS (`Access-Control-Allow-Origin: *` on endpoints that return auth'd data, or reflecting origin without allowlist)

## Input validation

- **High**: API endpoint accepts request body without schema validation (no Zod/Valibot/Yup/Joi)
- **High**: user input used in `RegExp` constructor without escaping (ReDoS or injection)
- **Medium**: missing length limits on strings that go into DB or logs (memory/log flooding)
- **Medium**: `parseInt`/`parseFloat`/`Number` on user input without NaN/bounds check

## XSS and output handling

- **High**: `dangerouslySetInnerHTML` with user-controlled content
- **High**: `innerHTML = userInput` in any frontend code
- **High**: server rendering user input into HTML without escaping (templating engines vary — Handlebars escapes by default, EJS `<%=` escapes, `<%-` does not)
- **Medium**: Content-Security-Policy not set on a user-facing HTML response

## Sensitive data

- **Critical**: secrets logged (`console.log(token)`, `logger.info({user})` where user contains password/token)
- **High**: secrets in error messages returned to clients
- **High**: PII logged without masking
- **Medium**: stack traces returned to clients in production

## Cryptography

- **Critical**: `Math.random()` used for security purposes (tokens, session IDs, password resets). Use `crypto.randomBytes` or `crypto.randomUUID`.
- **Critical**: weak hashing for passwords (`md5`, `sha1`, `sha256` without salt+KDF). Use `bcrypt`, `scrypt`, `argon2`.
- **High**: deprecated crypto (`createCipher` instead of `createCipheriv`, ECB mode, hardcoded IV)
- **High**: JWT `none` algorithm, or HS256 with a weak secret

## Race conditions and concurrency

- **High**: check-then-act on shared resources without locking (classic TOCTOU)
  - e.g., `if (!await db.exists(x)) await db.insert(x)` — two concurrent requests both pass the check
  - Fix: DB unique constraint, `INSERT ... ON CONFLICT`, or row-level lock
- **High**: missing idempotency on payment/write endpoints that could be retried

## Dependency and supply chain

- **Medium**: new dependency added in package.json/lock files with very low weekly downloads or recently published. Flag for manual inspection, do not auto-approve.
- **Low**: pinning issues (`^` vs exact version) for security-critical deps

## Denial of service

- **High**: unbounded loops driven by user input (e.g., user says `count: 1000000`)
- **High**: expensive regex on user input (ReDoS — catastrophic backtracking patterns like `(a+)+b`)
- **Medium**: missing rate limit on public endpoints, especially auth/signup/password-reset
- **Medium**: JSON body parser with no size limit (default in Express is 100kb, but some apps raise it blindly)

## File uploads

- **High**: file uploads without size limit (DoS)
- **High**: file uploads without MIME or extension validation stored in a web-served directory
- **High**: user-controlled filenames used directly on disk (path traversal)

## Cookies and headers

- **Medium**: missing security headers on responses: `X-Content-Type-Options: nosniff`, `Strict-Transport-Security` on HTTPS
- **Low**: verbose `X-Powered-By` headers exposing stack

## Environment and config

- **Critical**: `.env` files checked into git (check diff for `.env`, `.env.*` additions)
- **High**: `dotenv.config()` not called before reading `process.env.*` in critical paths
- **Medium**: default secrets like `secret: 'changeme'` or `SESSION_SECRET || 'dev'` fallbacks that could ship to production

## How to report security findings

- Always cite the exact line.
- Explain the attack: "An attacker could send X to cause Y."
- Propose a concrete fix, not "sanitize the input". Name the API or pattern.
- If you're uncertain whether something is exploitable (e.g., input might be sanitized upstream), mark it **Medium** and phrase it as "Potential X — verify input is sanitized before reaching here."
