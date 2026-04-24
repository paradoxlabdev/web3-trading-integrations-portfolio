# Security & Correctness Fixes

> Five production-grade security or correctness issues identified, investigated, and fixed during the engagement. Each writeup includes the issue, how it was found, exploitation scenario (where relevant), and the specific fix. Severity ranges from Critical to Low — these are internal fixes in a closed-source product, not publicly-disclosed CVEs.

## Table of Contents

- [1. Private-Key Scope Leak](#1-private-key-scope-leak)
- [2. Unauthenticated Enqueue Endpoint](#2-unauthenticated-enqueue-endpoint)
- [3. Stamped-Request Auth Broken by Body Consumption](#3-stamped-request-auth-broken-by-body-consumption)
- [4. EOA Lookup Case-Sensitivity](#4-eoa-lookup-case-sensitivity)
- [5. FOK Fill Without `orderID` Retried Indefinitely](#5-fok-fill-without-orderid-retried-indefinitely)
- [Process & Methodology](#process--methodology)

---

## 1. Private-Key Scope Leak

### Severity
High — could expose a signing key to an unrelated worker context.

### Discovery
Code review during a refactor pass. Spotted a closure holding a reference to a key that outlived its intended scope.

### Issue

A helper function in the copy-trading worker loaded a signing key at module level and embedded it in a closure passed to downstream processors. The closure was retained in memory across requests due to a shared processor instance.

Simplified reproduction:

```typescript
// BEFORE — key leaks into module-scope closure
const signer = new Wallet(process.env.PRIVATE_KEY!)  // module scope

export function createTradeProcessor() {
  return async (trade: Trade) => {
    const tx = await signer.sendTransaction(...)  // closure holds `signer`
    return tx
  }
}

// The processor instance was cached globally:
const globalProcessor = createTradeProcessor()
// → `signer` kept alive forever, accessible via JS debugger / heap dump
```

### Exploitation Scenario

Not remotely exploitable directly, but:

1. A compromised dependency could read `signer` from the module scope
2. A heap dump (debug endpoint, crash report) could leak the key in plaintext

Neither is a high-probability vector on its own, but defense-in-depth: keys shouldn't outlive the operation that needs them.

### Fix

Scope the key per-invocation, retrieve from secret store on each call:

```typescript
// AFTER — key retrieved per call, no closure leak
async function getSigner(): Promise<Wallet> {
  const key = await secretStore.get('TRADE_PRIVATE_KEY')
  return new Wallet(key)
}

export function createTradeProcessor() {
  return async (trade: Trade) => {
    const signer = await getSigner()
    try {
      const tx = await signer.sendTransaction(...)
      return tx
    } finally {
      // Key goes out of scope when function returns
    }
  }
}
```

Added a test that asserts the processor function's closure doesn't contain a `Wallet` instance on inspection.

### Hardening in the same pass

- Added deduplication on `original_trade_id` in `copy_trade_queue` (orthogonal, but in the same commit because touching the same files)
- Added size-scaling upper bounds (someone could game a trader's reported size otherwise)

---

## 2. Unauthenticated Enqueue Endpoint

### Severity
Critical — any caller could enqueue arbitrary trades for any follower.

### Discovery
Security review pass before production ship. Route was exposed but relied on a "secret" that had leaked.

### Issue

The `/api/copy-trade/enqueue` endpoint accepted any caller with a known `INTERNAL_API_SECRET`. Problems:

1. The secret was referenced in client-side code (for dev debugging), which got committed and deployed. Anyone could read it in browser devtools.
2. Even without the leak, "secret in env + shared code paths" is a weak auth model.
3. No rate limiting on the endpoint.

### Exploitation Scenario

1. Attacker reads `INTERNAL_API_SECRET` from client bundle
2. Attacker scripts requests to `/api/copy-trade/enqueue` with arbitrary `follower_id` and trade payload
3. Follower's Safe executes the attacker-crafted copy trade
4. Attacker could exhaust follower's balance on bad markets, or fill their own orders with follower's money

### Fix

1. **Remove `INTERNAL_API_SECRET` entirely.** Rotate + delete.
2. **Make enqueue cron-only.** Only callable by Vercel cron with `x-vercel-cron-signature` header (verified against a private key).
3. **For browser fast-path**, create a separate endpoint `/api/copy-trade/enqueue-fast` that uses the user's stamped-request auth (user can only enqueue for themselves).
4. **Rate-limit both endpoints** at the edge (max 10 req/sec per IP for fast-path, cron has its own throttle).

```typescript
// AFTER
export async function POST(request: NextRequest) {
  // Cron-only verification
  const isCron = verifyCronSignature(request)
  if (!isCron) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })
  }
  // ... enqueue logic
}
```

For the fast-path route:

```typescript
export async function POST(request: NextRequest) {
  // Require stamped-request auth, user can only enqueue for themselves
  const user = await resolveUser(request)
  const body = await request.json()
  if (body.follower_id !== user.id) {
    return NextResponse.json({ error: 'Forbidden' }, { status: 403 })
  }
  // ... enqueue logic
}
```

### Commit reference
`security: remove exposed INTERNAL_API_SECRET, enqueue now cron-only`

---

## 3. Stamped-Request Auth Broken by Body Consumption

### Severity
Medium — caused 401s in production for legitimate users on POST endpoints; easy path to confusion and erroneously opening the endpoint.

### Discovery
Production error report — users in "production mode" (not dev-auth-bypass) hit 401 on create/edit relationship.

### Issue

The `resolveUser(request)` helper reads the request body to verify the stamped signature. But the POST handler was also reading the body via `await request.json()`. The **first** read consumed the underlying stream; the second returned empty.

Sequence:

```typescript
// BROKEN
export async function POST(request: NextRequest) {
  const body = await request.json()  // reads stream — consumed
  const user = await resolveUser(request)  // tries to read again — empty
  //      ↑ returns null → we return 401
  // ...
}
```

### Exploitation Scenario

Not exploitable for privilege escalation. The concern was that, in trying to fix the 401s quickly, a developer could loosen auth ("maybe just skip auth if body parse fails"). I wanted to lock this down permanently.

### Fix

Resolve user **before** body parse. `resolveUser` reads + caches the body once:

```typescript
// FIXED
export async function POST(request: NextRequest) {
  const { user, body } = await resolveUserWithBody(request)
  if (!user) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })
  }
  // `body` already parsed inside resolveUser, passed back
  // ... logic
}
```

The `resolveUserWithBody` helper:

```typescript
export async function resolveUserWithBody(request: NextRequest): Promise<{ user: User | null; body: any }> {
  const raw = await request.text()  // single read
  const body = raw ? JSON.parse(raw) : {}

  const stamp = request.headers.get('x-stamped-signature')
  if (!stamp) return { user: null, body }

  const verified = verifyStamp(stamp, raw)
  if (!verified) return { user: null, body }

  const user = await db.query.users.findFirst({
    where: eq(users.walletAddress, verified.signer.toLowerCase()),
  })

  return { user, body }
}
```

### Hardening in the same pass

- Refactored all CT API routes to use `resolveUserWithBody` (replace ALL remaining inline auth with shared helper)
- Added integration tests for the body-consumption edge case specifically

### Commit reference
`fix: resolveUser before request.json() — body was consumed, causing 401 on POST`

---

## 4. EOA Lookup Case-Sensitivity

### Severity
Medium — silent auth failure on withdraw for a subset of users (those with checksummed address format in the DB).

### Discovery
Support ticket — user reported "withdraw returns 401 but I'm logged in". Reproduced by checking the DB for their row.

### Issue

The withdraw endpoint looked up users by EOA address:

```typescript
// BROKEN
const user = await db.query.users.findFirst({
  where: eq(users.walletAddress, signedAddress),
})
```

Ethereum addresses are case-insensitive (the hex representation). But some wallet providers return the **checksummed** form (mixed case), and some DB rows had checksummed addresses stored. The SQL `=` comparison is case-sensitive by default on PostgreSQL.

Result: if user's row had `0xAbCd...` and their wallet returned `0xabcd...`, the lookup returned `null` → 401.

### Exploitation Scenario

Not exploitable for privilege escalation — legitimate users just couldn't use the endpoint. But trust-eroding (users blame the product for broken withdraws).

### Fix

Normalize addresses to lowercase at **both ends** — at insert and at query:

```typescript
// Normalize at insert
await db.insert(users).values({
  walletAddress: address.toLowerCase(),  // always store lowercase
  // ...
})

// Normalize at query
const user = await db.query.users.findFirst({
  where: eq(users.walletAddress, signedAddress.toLowerCase()),
})
```

Also wrote a one-time migration to lowercase all existing `wallet_address` values:

```sql
UPDATE users SET wallet_address = LOWER(wallet_address);
```

### Lesson

Ethereum addresses are case-insensitive by spec. Never rely on mixed-case form for lookups. Enforce lowercase normalization at the boundary (insert, API input) and everywhere that reads addresses.

### Commit reference
`fix(copy-trading): withdraw 401 after intent-sig lookup dropped checksum`

---

## 5. FOK Fill Without `orderID` Retried Indefinitely

### Severity
Low (correctness bug, not auth) — queue grew without bound on specific error shapes, increasing infra cost and making retries look "pending forever" to users.

### Discovery
Queue size monitoring — saw items stuck in `pending` with 100+ attempts.

### Issue

After submitting a FOK order, the response shape is usually:

```json
{ "orderID": "abc123", "status": "filled", "fills": [...] }
```

But the upstream occasionally returns a fill-success response **without** `orderID`:

```json
{ "status": "ok", "fills": [{ ... }] }
```

Our retry classification treated "no orderID" as a transient failure → retry. But the trade had already executed. Result:

1. Copy trade executes successfully on-chain
2. Our code sees no `orderID`, marks pending, retries
3. Retry creates a *second* copy trade (with different `original_trade_id`? or not deduped here), or fails at CLOB with "already filled"
4. Either way, queue item looked stuck to the user

### Exploitation Scenario

Not exploitable externally. But:

- Unbounded retry of a "filled" order = 2× position for follower (rare, but costly)
- Queue grew without bound → DB size creep
- Monitoring noise drowned out real failures

### Fix

Classify "fill success without orderID" as **non-retriable**, treat it as a success:

```typescript
if (result.status === 'ok' || result.status === 'filled') {
  // Success, even without orderID
  if (!result.orderID) {
    // Log the full response for upstream investigation
    captureApiError(new Error('CLOB ok without orderID'), {
      endpoint: 'submit_fok',
      extra: { response: result },
      tags: { category: 'correctness', severity: 'low' },
    })
  }

  await db.update(copy_trade_queue).set({
    status: 'completed',
    tx_hash: result.txHash ?? result.fills?.[0]?.txHash ?? null,
    actual_sell_value: calculateFromFills(result.fills),
    completed_at: Date.now(),
  }).where(eq(id, item.id))

  return
}
```

Also added explicit non-retriable shape for the related "min size" / "invalid amount" errors that showed the same pattern.

### Commit references
`fix: log full CLOB response when order has no orderID`
`fix: add 'invalid amount' and 'min size' to non-retriable errors`

---

## Process & Methodology

### How these were found

- **1, 2:** Proactive security review pre-ship (went through every sensitive endpoint, asked "what if")
- **3:** Production error report, reproduced locally
- **4:** Support ticket, reproduced by DB inspection
- **5:** Queue monitoring alerts, investigated stuck items

### What I'd do the same way

- **Never skip-ship to fix a 401.** Fix the root cause. Fix #3's initial report was "auth is broken in prod" — tempting to disable auth temporarily. Instead I reproduced, identified body-consumption, fixed properly.
- **Log full responses on unexpected shapes.** Fix #5 added Sentry logging for the "ok without orderID" case — future upstream changes will surface quickly.
- **Normalize at the boundary.** Address case sensitivity (#4) is one of many boundary issues; also normalize timezones (UTC), decimals (bigint), and JSON keys (camelCase vs snake_case) at the boundary.

### What I'd improve next

- Add automated integration tests for stamped-request auth on every new API route (would have caught #3 before prod)
- Add a `verifyCronSignature` wrapper that can't be disabled with a flag (would have prevented #2 from existing in the first place)
- Add a static analysis check for `address` string comparisons (would have caught #4)
- Formalize error-classification table as a single source of truth (retriable vs non-retriable)

### Sentry tagging convention

For every captured error:

```typescript
captureApiError(error, {
  endpoint: 'route_name',
  extra: { /* relevant payload, no secrets */ },
  tags: {
    platform: 'limitless' | 'myriad' | 'copy-trading' | 'polymarket',
    category: 'auth' | 'trading' | 'correctness' | 'performance',
    severity: 'critical' | 'high' | 'medium' | 'low',
    userFacing: 'true' | 'false',
  },
})
```

Lets us filter Sentry by "which integration is misbehaving today" or "all user-facing auth errors this week" in one click.

---

*Back to [README.md](./README.md).*
