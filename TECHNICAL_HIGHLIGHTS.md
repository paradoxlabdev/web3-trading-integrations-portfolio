# Technical Highlights — Deep Dive

> Detailed writeups of six technical problems I solved during this engagement. Each includes the problem statement, the approach I evaluated (and what I rejected), the final solution, and measurable impact.

## Table of Contents

- [1. FIFO PnL Reconciliation Across Three Trade Sources](#fifo-pnl)
- [2. Protocol B Auth Migration — Session to HMAC](#hmac-migration)
- [3. Copy-Trading Latency — from 60s Polling to WebSocket-Driven](#copy-trading-latency)
- [4. AMM Sell-Output Binary Search — Local Simulation Instead of On-Chain Polling](#amm-binary-search)
- [5. Post-Trade Polling — Fixing the "Just-Closed" Flash](#post-trade-polling)
- [6. Concurrent Close Guard — Nonce Collision](#concurrent-close)

---

## <a id="fifo-pnl"></a>1. FIFO PnL Reconciliation Across Three Trade Sources

### Problem Statement

A user's position on Protocol B can accumulate trades from three independent sources:

1. **CLOB fills** — order-book trades, with an `actualSellValue` returned by the exchange
2. **AMM trades** — hybrid liquidity, shares-in / shares-out computed via on-chain binary search
3. **Raw on-chain balance delta** — user manually transferring CTF tokens in/out

A naive PnL calculation — `unrealized = shares × current_price` — produces the **wrong result** when any of the following happens:

- A SELL for more shares than the most recent BUY (matches against multiple earlier BUYs with different cost bases)
- A partial fill where `actualSellValue < expected` (slippage, skipped rounding)
- A redeem that settled but wasn't recorded as a SELL
- A user imports an existing position (on-chain balance > sum of recorded BUYs)

**Symptom:** Win Rate and PnL in the dashboard contradicted on-chain USDC flows. Users reported the numbers looked "off by 10-20%" on active positions.

### Approach Rejected: Recompute from on-chain events

One option was to subscribe to every CTF transfer event and reconstruct the cost basis from chain history. Rejected because:

- Requires a dedicated indexer (out of scope, and slow)
- Upstream platforms don't publish event ABIs in a stable format
- Redeem events don't include the original buy price
- Cross-source trades (user bought via us, sold on native UI) wouldn't have our cost basis anyway

### Solution

**Two canonical sources, reconciled at render time:**

- On-chain `CTF.balanceOf(conditionId)` = **canonical for open shares**
- DB trade history (`trade_history` table) = **canonical for cost basis and PnL**

**Algorithm (pseudocode):**

```typescript
function reconcilePosition(conditionId, userId) {
  const onChainBalance = CTF.balanceOf(conditionId, userId)
  const trades = db.select().from(trade_history)
                    .where(and(eq(userId), eq(conditionId)))
                    .orderBy(asc(timestamp))

  // Build FIFO queue of BUYs
  let buyQueue = []  // [{ shares, costPerShare }]
  let realizedPnl = 0
  let totalBoughtShares = 0

  for (const trade of trades) {
    if (trade.side === 'BUY') {
      buyQueue.push({ shares: trade.shares, costPerShare: trade.price })
      totalBoughtShares += trade.shares
    } else { // SELL or REDEEM
      let remaining = trade.shares
      const proceeds = trade.actualSellValue  // critical: use actual, not requested
      const costOfSold = matchAgainstBuyQueue(buyQueue, remaining)
      realizedPnl += proceeds - costOfSold
    }
  }

  // Unrealized: remaining open shares × current price
  const openShares = sumShares(buyQueue)
  const unrealizedPnl = openShares * currentMidpoint - sumCost(buyQueue)

  // Cross-check vs on-chain
  if (Math.abs(onChainBalance - openShares) > DUST_THRESHOLD) {
    Sentry.captureMessage('Position drift detected', { userId, conditionId, onChainBalance, openShares })
  }

  // Render on-chain balance as authoritative
  return {
    openShares: onChainBalance,
    realizedPnl,
    unrealizedPnl: onChainBalance * currentMidpoint - proportionalCost(buyQueue, onChainBalance),
  }
}
```

**Key decisions:**

1. **`actualSellValue`, not requested size.** The CLOB returns what was actually filled (accounting for slippage, rounding to tick size). Using the requested size silently overstates proceeds by 0.1-2%.
2. **REDEEM treated as a SELL with `calculated_size = original_buy_cost`.** This makes Win Rate math consistent — a winning redeem counts as a profitable round trip, not as a mysterious "non-trade".
3. **On-chain balance is canonical.** If DB says 10 shares but chain says 8, we render 8. The 2-share drift is flagged to Sentry for investigation, but the user sees reality.
4. **Dust threshold (`< 0.05 shares`).** Sub-dust positions are filtered to avoid "you own 0.0001 YES" noise in the UI.

### Effect

- Accuracy verified against on-chain USDC flows on production traffic — the on-chain delta is the test oracle, since the whole point of the FIFO reconstruction is to match reality.
- Sentry "drift detected" captures continued at a low rate (a handful per week), all traceable to legitimate cross-source activity (native-UI trades by power users who skip our flow) — useful signal, not a bug.
- Win Rate rendered consistently across dashboard, trader card, and history table (previously each surface recomputed from different inputs and disagreed on edge cases).

### Edge Cases Handled

- SELL exceeds matched BUY queue → cap at queue total, log to Sentry (shouldn't happen)
- Redeem on a position with zero recorded BUYs (user imported) → record as SELL with `calculated_size = 0`, PnL = full proceeds
- Tx-hash duplicate (WS + cron both ingested the same trade) → unique constraint on `(follower_id, tx_hash)` catches it at DB layer
- Floating-point rounding → all math in smallest units (shares × 10^6), converted to display only at UI boundary

---

## <a id="hmac-migration"></a>2. Protocol B Auth Migration — Session to HMAC

### Problem Statement

Protocol B initially required a SIWE session per user per browser session. This caused:

- **Rate-limit cascades.** The SIWE endpoint had aggressive per-IP limits; any time a user refreshed the page mid-session (or had multiple tabs), they'd hit 429 and get locked out for 60s.
- **Orphan sessions.** Session cookies didn't survive a redeploy (session store was in-memory). Users had to re-sign mid-trade.
- **Multi-device nightmare.** Signing on desktop invalidated mobile session.
- **Silent failure.** `pollUntilState(STATE_MINED)` returned success even when the underlying tx reverted — approvals would "succeed" but every subsequent trade silently broke.

### Approach: Three-Phase Migration

Migration happened in three phases over ~5 days, each phase shippable.

**Phase 1: SIWE with cooldown guard** (shipped to stabilize)

- Added a client-side 60s cooldown lockout before any SIWE call
- Added a `partner-register` fallback when login hits rate limit
- Saved session token to localStorage as backup
- Added `ensureSafeReady` that re-verifies approvals after each tx (don't trust `STATE_MINED`)

This reduced 429 incidents from ~daily to ~weekly, but didn't eliminate the root cause.

**Phase 2: HMAC auth with partner accounts** (architecture change)

- Registered a partner account with Protocol B
- All server-side calls now sign with HMAC(secret, body + timestamp)
- `x-on-behalf-of: <user-eoa>` header identifies the acting user
- Main app never touches SIWE anymore for routine reads
- Session auth kept only for user-initiated trading confirmations

**Phase 3: On-chain CTF for positions** (final cleanup)

- Position fetch no longer requires ANY auth — reads CTF balance directly from chain
- Removed session token management code
- Removed unused session storage tables

### Solution Details

**HMAC signing (pseudocode):**

```typescript
function signRequest(body: object, secret: string) {
  const timestamp = Date.now()
  const nonce = crypto.randomBytes(16).toString('hex')
  const payload = `${timestamp}.${nonce}.${JSON.stringify(body)}`
  const signature = crypto.createHmac('sha256', secret).update(payload).digest('hex')
  return {
    'x-timestamp': timestamp,
    'x-nonce': nonce,
    'x-signature': signature,
    'content-type': 'application/json',
  }
}

// Usage in a server route
export async function POST(request: NextRequest) {
  const user = await resolveUser(request)  // our own stamped-request auth
  const body = await request.json()
  const headers = {
    ...signRequest(body, process.env.PROTOCOL_B_HMAC_SECRET),
    'x-on-behalf-of': user.walletAddress,
  }
  const res = await fetch(`${PROTOCOL_B_URL}/orders`, { method: 'POST', headers, body: JSON.stringify(body) })
  // ...
}
```

**`ensureSafeReady` pattern:**

```typescript
// OLD (broken):
await pollUntilState(txHash, STATE_MINED)
// assumed success, moved on

// NEW:
await pollUntilState(txHash, STATE_MINED)
const receipt = await provider.getTransactionReceipt(txHash)
if (receipt.status !== 1) {
  throw new Error('Tx mined but reverted — retrying approval')
}
await ensureSafeReady()  // re-checks allowance, re-checks CTF approvals
```

### Effect

Qualitative, since the internal monitoring I used isn't accessible post-engagement:

- 429-cascade incidents on position fetch — had been a recurring daily support topic, stopped being one after Phase 3.
- Page-refresh "please sign again" flow — gone; HMAC is session-less for reads.
- Multi-tab / multi-device session conflicts — gone for the same reason.
- Position fetch no longer has an auth round-trip on the critical path (HMAC signing happens server-side with a static secret, not per-user SIWE).

### Lessons

- **Don't trust `STATE_MINED`.** It means the tx is in a block — not that it succeeded.
- **HMAC > session for machine-to-machine.** Session cookies are a UX pattern for human browsers; server-to-server should always be keyed.
- **Phase migrations.** We couldn't drop sessions until HMAC was proven stable. Each phase was safe to revert.

---

## <a id="copy-trading-latency"></a>3. Copy-Trading Latency — from 60s Polling to WebSocket-Driven

### Problem Statement

Copy-trading works by watching a target wallet on an upstream platform and replicating their trades for a follower. Original pipeline:

```
Trader trades → Upstream Data API → Our cron (every 60s) → Enqueue copy → Execute
```

Worst-case latency: **120s** (trader trades at T+0, our cron fires at T+60, execution takes T+60s). Average: ~60s. Traders' markets move in seconds — 60s is the difference between "same price as trader" and "10% worse price".

### Approach Rejected: Faster polling

We could poll every 5s instead of 60s. Rejected because:

- Upstream Data API rate limits cut in at ~6 req/minute per IP
- Polling 50+ wallets at 5s = instant rate limit
- Doesn't help anyway — upstream's Data API lags real trades by 10-30s itself

### Solution: Three-Source Ingestion with Dedup

**Source 1: WebSocket worker (primary, 3-5s)**

Upstream platforms usually expose a WebSocket feed for live order flow. I built a Node worker:

```
┌──────────────────────────────────────────────────────┐
│          Copy-Trading Worker (Railway)               │
│                                                       │
│  - Subscribes to upstream WS `/trades` stream        │
│  - Filters events to tracked-wallets list            │
│  - Heartbeat every 30s (detects zombie connections)  │
│  - Exponential reconnect backoff (2s → 60s)          │
│  - On reconnect: refetches tracked-wallets           │
│                                                       │
│  On trade event:                                      │
│  → POST /api/enqueue-cron (with CRON_SECRET header)  │
│  → body: { follower_id, tx_hash, asset, side, size } │
└──────────────────────────────────────────────────────┘
```

**Source 2: Browser fast-path (1-3s, bonus)**

If a follower is also watching the trader live (market detail page open), their browser sees the trade event via the user-facing WebSocket **before** the server's worker picks it up. Added a direct ping:

```typescript
// In TraderActivityWS.tsx
onTradeEvent((trade) => {
  if (trade.trader_wallet === watchedWallet) {
    fetch('/api/copy-trade/fast-path', {
      method: 'POST',
      body: JSON.stringify({ trade }),
    })
  }
})
```

Unique constraint on `(follower_id, tx_hash)` means the DB rejects the duplicate if the worker gets there first. No race.

**Source 3: Cron fallback (60s, safety net)**

Kept the original 60s cron as a fallback. If WS drops or misses events, cron catches them. Idempotent via tx-hash.

### Effect

With the 60s polling cron as the only ingestor, follower latency was bounded by the cron interval (plus execution time). After the WebSocket worker shipped, the primary ingestion path became event-driven (single-digit seconds between upstream trade and our enqueue in typical conditions). The fast-path shortens it further when the follower is live on the same market.

I no longer have access to the internal dashboards used to measure this in production, so the headline "N× faster" number is withheld here rather than written from memory.

### Infrastructure Notes

- Worker on Railway (1 small instance, ~$5/mo)
- WS heartbeat: if no pong in 60s, reconnect (catches zombie connections that silently drop)
- Dead-letter: failed enqueues log to Sentry, no data loss
- All three sources deduplicate at DB via unique constraint — zero application-level coordination

### Worker Resilience Patterns

```typescript
// Heartbeat + zombie detection
let lastPong = Date.now()

ws.on('pong', () => { lastPong = Date.now() })

setInterval(() => {
  if (Date.now() - lastPong > 60_000) {
    console.warn('WS zombie detected, reconnecting')
    ws.terminate()
    reconnect()
  } else {
    ws.ping()
  }
}, 30_000)

// Exponential reconnect backoff
let attempt = 0
async function reconnect() {
  const delay = Math.min(2000 * 2 ** attempt, 60_000)
  attempt++
  await sleep(delay)
  connect()
}

ws.on('open', () => { attempt = 0 })
```

---

## <a id="amm-binary-search"></a>4. AMM Sell-Output Binary Search — Local Simulation Instead of On-Chain Polling

### Problem Statement

Protocol B's AMM markets use a hybrid curve. To show a user "if you sell X shares, you get $Y USDC" requires solving an on-chain formula that doesn't have a closed-form inverse. The original implementation binary-searched over on-chain quotes:

```typescript
// OLD (slow)
async function findSellAmount(targetUsdc: number) {
  let lo = 0, hi = maxShares
  for (let i = 0; i < 30; i++) {
    const mid = (lo + hi) / 2
    const quote = await contract.getSellQuote(mid)  // RPC call
    if (quote.usdc < targetUsdc) lo = mid
    else hi = mid
  }
  return lo
}
```

This made **30+ RPC calls per quote**. At ~60ms per call, the UI froze for 2+ seconds before showing the output amount.

### Solution: Pre-fetch pool state, compute locally

```typescript
// NEW (30+ RPC calls per quote → 2)
async function findSellAmount(targetUsdc: number, poolState: PoolState) {
  // poolState is fetched ONCE per session and cached
  let lo = 0, hi = poolState.maxShares
  for (let i = 0; i < 30; i++) {
    const mid = (lo + hi) / 2
    // Local computation — no RPC
    const quote = simulateSellQuote(mid, poolState)
    if (quote.usdc < targetUsdc) lo = mid
    else hi = mid
  }
  // Final verification against chain
  const finalQuote = await contract.getSellQuote(lo)
  return { amount: lo, verifiedUsdc: finalQuote.usdc }
}
```

**Where `simulateSellQuote` implements the on-chain formula in TS** — reverse-engineered from the contract source. Maintained in lockstep via a unit test that asserts `simulateSellQuote(x) === contract.getSellQuote(x)` for 100 random inputs.

### Effect

- RPC calls per quote: 30+ → 2 (one for pool state, one for final verification). This is a mechanical change in the code, not an estimate.
- Pool state cached per session — amortizes to 1 RPC call per quote after the first.
- Wall-clock improvement depends on RPC latency and network conditions; cut by roughly the same factor as the RPC-call count in the prod environments I measured.

### Also added in the same pass

- Price enrichment — if current midpoint is transiently 0 during a state transition, fall back to last known good price (avoids flashing "$0.00" on the trade widget)
- AMM polling cadence reduced (orderbook 1s → 30s, chart 2s → 10s) — replaced by WS-driven updates
- Cleanup of redundant query invalidations post-trade — multiple hooks were invalidating the same keys

---

## <a id="post-trade-polling"></a>5. Post-Trade Polling — Fixing the "Just-Closed" Flash

### Problem Statement

After a user sold a position, the UI would flash the closed position back onto the screen for ~30 seconds before it finally disappeared. Root cause: our server cached position data for 30s, so even though the user's SELL executed on-chain, the next position-fetch returned the pre-trade snapshot.

### Approach Rejected: Lower cache TTL to 1s

Would cost 30× more RPC calls globally. Unacceptable.

### Solution: Optimistic Hide + Post-Trade Polling

**Two-part fix:**

**Part 1: Local "just-closed" set (optimistic)**

```typescript
// Zustand store
const useJustClosedStore = create((set) => ({
  justClosed: new Set<string>(),  // tokenIds
  markClosed: (tokenId: string) => set((state) => ({
    justClosed: new Set([...state.justClosed, tokenId]),
  })),
  clearClosed: (tokenId: string) => set((state) => {
    const next = new Set(state.justClosed)
    next.delete(tokenId)
    return { justClosed: next }
  }),
}))

// In PositionList
const positions = useQuery({...}).data
const justClosed = useJustClosedStore(s => s.justClosed)
const visible = positions.filter(p => !justClosed.has(p.tokenId))
```

When user clicks SELL and execution succeeds, we add the tokenId to `justClosed`. The position disappears from the UI instantly.

**Part 2: Post-trade polling with backoff**

```typescript
async function pollUntilClosed(tokenId: string) {
  const maxAttempts = 10
  let delay = 1000
  for (let i = 0; i < maxAttempts; i++) {
    await sleep(delay)
    delay = Math.min(delay * 1.5, 5000)  // 1s, 1.5s, 2.25s, ..., cap at 5s
    queryClient.invalidateQueries(['positions'])
    const refreshed = await queryClient.fetchQuery({ queryKey: ['positions'] })
    const stillPresent = refreshed.find(p => p.tokenId === tokenId)
    if (!stillPresent || stillPresent.shares === 0) {
      useJustClosedStore.getState().clearClosed(tokenId)
      return
    }
  }
  // Server still has stale data — clear anyway, user sees truth on next refresh
  useJustClosedStore.getState().clearClosed(tokenId)
}
```

Reusable across SELL and REDEEM. Extracted into a shared module (`post-trade-polling.ts`).

### Effect

- "Just-closed flash" stopped appearing in the production path I could exercise end-to-end.
- Cache TTL unchanged — no infrastructure cost.
- Module reused for both market orders and redeems.

---

## <a id="concurrent-close"></a>6. Concurrent Close Guard — Nonce Collision

### Problem Statement

When a user clicked "Close" on two positions in quick succession, both tx submissions used the same nonce (Account Kit smart wallet nonce), causing one to revert silently. The UI showed "both closed" but one was still on-chain.

### Attempted Fix #1: tx-level mutex (rejected)

Lock at the transaction-sending layer. Problem: the UI spinner state lives in the hook layer. Locking at tx layer made both spinners spin, then one errored — confusing UX.

### Solution: Hook-level mutex per user

```typescript
const closeInProgress = useRef<Set<string>>(new Set())

async function closePosition(tokenId: string) {
  if (closeInProgress.current.has(tokenId)) {
    throw new Error('Close already in progress for this position')
  }
  // Block concurrent closes ACROSS positions for this user
  if (closeInProgress.current.size > 0) {
    throw new Error('Another position is currently closing, please wait')
  }
  closeInProgress.current.add(tokenId)
  try {
    await executeClose(tokenId)
  } finally {
    closeInProgress.current.delete(tokenId)
  }
}
```

The second click shows a toast "another position is currently closing, please wait" — UX is clear, no silent failures.

Also added: fail-closed RPC fallback. If primary RPC drops, fall back to secondary. If both fail, show error — don't return stale data.

### Effect

The specific class of "clicked close, got no feedback, position stayed" tickets stopped landing in the support queue I was watching during ship week. Under load I couldn't reproduce it locally anymore.

---

## Bonus: Crypto Markets Cold Start — 2-phase fetch

A smaller optimization but user-visible enough to mention.

**Problem:** First visit to the live-crypto page blocked on a serial fetch chain: (a) all markets, (b) prices, (c) user positions, (d) platform metadata. The user stared at an empty page until the last step resolved.

**Solution: 2-phase fetch**

- Phase 1: fetch market list + prices in parallel, render tiles as soon as any platform loads.
- Phase 2: fetch positions + metadata in the background, show a per-tile indicator while that's populating.

Also prefetched Limitless + Myriad data regardless of the current platform filter (cheap, amortizes when the user switches filter) and lowered the server cache (30s → 10s) for faster 15-minute market rollovers.

The user-perceived "time to first meaningful content" went from "blocked on full chain" to "tiles as soon as Phase 1 returns".

---

*See [MYRIAD_INTEGRATION.md](./MYRIAD_INTEGRATION.md), [LIMITLESS_INTEGRATION.md](./LIMITLESS_INTEGRATION.md), and [COPY_TRADING_SYSTEM.md](./COPY_TRADING_SYSTEM.md) for platform-specific deep-dives.*
