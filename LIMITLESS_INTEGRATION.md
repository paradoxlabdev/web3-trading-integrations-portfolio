# Protocol B Integration — Hybrid CLOB + AMM Prediction Market

> Full integration of a hybrid order-book + AMM prediction-market platform. Scope included a three-phase auth migration, CLOB and AMM trading, limit orders, WebSocket live feeds, redeem flow, and portfolio aggregation — spanning Base and Polygon chains.

## Table of Contents

- [Platform Overview](#platform-overview)
- [Integration Scope](#integration-scope)
- [Auth Evolution](#auth-evolution)
- [CLOB Trading](#clob-trading)
- [AMM Trading](#amm-trading)
- [Limit Orders](#limit-orders)
- [WebSocket Integration](#websocket-integration)
- [Redeem Flow](#redeem-flow)
- [Portfolio & Position Reconciliation](#portfolio--position-reconciliation)
- [Live Crypto, Sports, Group Markets](#live-crypto-sports-group-markets)
- [Performance Work](#performance-work)

---

## Platform Overview

Protocol B is a hybrid prediction-market platform:

- **Liquidity model:** CLOB (central limit order book) for popular markets, AMM fallback for thin books
- **Collateral:** USDC on Base + Polygon
- **Market types:** binary, multi-outcome (NegRisk), group markets (linked sub-markets), oracle-driven (live crypto)
- **Settlement:** Conditional Token Framework (CTF) — ERC-1155 positions with on-chain redemption

The integration is substantial: roughly two months of work, four market types (binary, multi-outcome / NegRisk, group, oracle-driven live crypto), two chains (Base + Polygon).

---

## Integration Scope

### Auth (three phases, see [Auth Evolution](#auth-evolution) below)

### CLOB

- Submit market orders (FOK) and limit orders (GTC)
- Open orders list with live WS updates
- Cancel orders, cancel-all, locked-balance display
- Fill notifications, dedup by `orderID` / `txHash`
- NegRisk sell-adapter approval for no-side exits

### AMM

- Quote + execute with on-chain binary search for exact sell amount
- Revert detection (don't trust `STATE_MINED`)
- Tx-hash-based dedup, on-chain shares as source of truth

### Orderbook & Chart

- Live orderbook panel next to chart (like major Polymarket UI)
- Orderbook toggle, search, trade history inline
- Oracle chart for live-crypto markets (5m intervals)
- Multi-line charts for group markets
- Price-history 6h support added
- Price / Probability toggle (hidden for multi-market group charts where it makes no sense)

### Redeem

- On-chain payout with NegRisk adapter when applicable
- REDEEM button in portfolio, hides both YES/NO sides after settlement
- SYNCING state while on-chain payout confirms
- Dedup by tokenId+txHash in notifications

### Live Crypto / Sports / Group Markets

- Live-crypto page: 2-phase fetch (markets → tiles → positions)
- Sports leagues filter added
- Group markets (linked sub-markets): parent image inherited by sub-positions, shared cache via globalThis

### Portfolio

- On-chain CTF balance as source of truth
- Open orders aggregated across markets in portfolio
- Trade history DB with FIFO PnL
- Safe auto-deployment on first withdraw

### Performance

- Rate-limit backoff, fewer API pages, 429 handling
- Polling reduced via WS (orderbook 1s → 30s, chart 2s → 10s)
- Concurrent close guard (nonce collision fix)
- Just-closed flash fix (post-trade polling)

---

## Auth Evolution

Three phases, each shippable, each improving on the last. Kept revertible at each step — the goal was never a big-bang rewrite.

### Phase 1: EOA Direct + SIWE

**Initial baseline:**

- User signs SIWE message on first trade
- Session token stored in localStorage
- CLOB requests include `Authorization: Bearer <session>`
- Duration: ~2 hours session expiry

**Problems observed in prod:**

- Per-user SIWE rate limit — hitting 429 on refresh
- Session expired mid-trade → confusing error state
- Sign-prompt fatigue on mobile

### Phase 2: SIWE + Guards

Shipped to stabilize Phase 1 while designing Phase 3.

**Additions:**

- 60s client-side cooldown lockout before SIWE call
  ```typescript
  const SIWE_COOLDOWN_KEY = 'limitless_siwe_last_attempt'
  function canAttemptSiwe(): boolean {
    const last = Number(localStorage.getItem(SIWE_COOLDOWN_KEY)) || 0
    return Date.now() - last > 60_000
  }
  ```
- `partner-register` fallback endpoint when login hits rate limit
- Skip cancel-orders when no session exists (don't create phantom requests)
- Remove duplicate SIWE login from `handleTrade` — let `submitOrder` handle it in one place
- Hide Sell button for Limitless positions in positions panel (was failing before Phase 3 positions logic landed)

### Phase 3: HMAC + Delegated Partner Accounts (final)

**Architecture:**

- Platform issues a **partner API key + HMAC secret** to approved integrators
- All server-side calls sign requests with `HMAC(secret, body + timestamp + nonce)`
- `x-on-behalf-of: <user-eoa>` header identifies the acting user
- **No session required** — session-less position fetches, orders, cancels
- Session auth kept only for user-initiated new trades (until fully replaced)

**Server route (simplified):**

```typescript
import { hmacSignRequest } from '@/lib/limitless/auth'

export async function POST(request: NextRequest) {
  const user = await resolveUser(request)  // our own stamped-auth
  const body = await request.json()

  const headers = hmacSignRequest({
    secret: process.env.LIMITLESS_HMAC_SECRET!,
    body,
  })
  headers['x-on-behalf-of'] = user.walletAddress.toLowerCase()

  const res = await fetch(`${LIMITLESS_API}/orders`, {
    method: 'POST',
    headers,
    body: JSON.stringify(body),
  })

  if (!res.ok) {
    captureApiError(new Error(`Limitless ${res.status}`), {
      endpoint: 'submit_order',
      extra: { status: res.status, user: user.id },
      tags: { platform: 'limitless', category: 'trading' },
    })
    return NextResponse.json({ error: 'Order failed' }, { status: 500 })
  }

  return NextResponse.json(await res.json())
}
```

### Migration plan to fully session-less

Tracked as a backlog item (`project_limitless_onbehalfof.md`): migrate portfolio fetches to `x-on-behalf-of` header — cleaner than the current 409 fallback path. Non-urgent post-Phase 3, because the cross-sourcing (on-chain CTF + HMAC orders) already works correctly.

### Why this matters

After Phase 3:
- 429-cascade on position-fetch stopped being a recurring support topic.
- Session-expired-mid-trade reports stopped.
- Multi-tab and multi-device sessions work consistently (they were lossy with the in-memory session store).
- Position fetch no longer has an auth round-trip on the critical path — HMAC signing happens server-side with a static secret.

---

## CLOB Trading

### Market Orders (FOK)

Fill-or-kill. Submit at current midpoint, either fully fills or rejects. Used for copy-trading and "buy now" flows where we want instant execution.

```typescript
async function submitFOK({ tokenId, side, size, price }) {
  return api.post('/orders', {
    tokenId,
    side,         // 'BUY' | 'SELL'
    size,         // in shares, smallest units
    price,        // limit price at current midpoint
    orderType: 'FOK',
  })
}
```

### BUY minimum

Protocol B rejects BUYs below ~$1 (after rounding). We enforce a client-side minimum of **$0.90** (allows $0.99 trades, which round up to $1.01). Early versions rejected $0.99 trades that would have filled.

### SELL minimum

Only applies to BUY — not SELL. An early bug rejected SELLs under $1 (assumption: same as BUY). Fixed so SELLs can fully exit small positions.

### Fill notifications

CLOB returns fills via a combination of:
- Synchronous `orderID` in POST response
- Async via WebSocket `order_filled` event
- Async via `trade_history` poll

Dedup by `(user_id, tx_hash)` in DB, with worker-level dedup by `order_id`. This was one of the 5 security/correctness fixes — the worker was previously accepting duplicate fills, over-reporting PnL.

### Record trade on SELL

After a successful SELL, the client records the trade in our DB:

```typescript
await api.post('/record-trade', {
  tokenId,
  side: 'SELL',
  shares: executedShares,
  actualSellValue,  // from CLOB response, NOT requested size
  txHash,
})
```

This feeds the FIFO PnL calculator. Using `actualSellValue` vs requested size is the difference between accurate and off-by-2% PnL.

---

## AMM Trading

For markets without active CLOB liquidity, fallback to AMM.

### Quote via local simulation

See [TECHNICAL_HIGHLIGHTS.md §4](./TECHNICAL_HIGHLIGHTS.md#amm-binary-search) for the local-simulation rewrite.

### On-chain shares as source of truth

AMM trades commit to the CTF contract. On-chain `balanceOf(conditionId)` is the canonical source for resulting shares. We cross-check against quote expected-shares to detect slippage/failure:

```typescript
const sharesBefore = await ctf.balanceOf(user, tokenId)
const tx = await amm.sell(tokenId, sharesToSell, minUsdcOut)
const receipt = await tx.wait()

// Revert detection
if (receipt.status !== 1) {
  throw new Error('AMM sell reverted')
}

const sharesAfter = await ctf.balanceOf(user, tokenId)
const actualSharesSold = sharesBefore - sharesAfter

if (Math.abs(actualSharesSold - sharesToSell) > DUST) {
  Sentry.captureMessage('AMM slippage mismatch', { ... })
}
```

### Tx-hash dedup

AMM trades can echo through multiple paths (event log, WS, polling). Unique constraint on `(user_id, tx_hash)` in `trade_history` prevents double-counting.

### SYNCING state

Between "tx mined" and "position reconciled" there's a ~5-10s window where the server cache is stale. UI shows a `SYNCING` badge on the position row, with post-trade polling in the background ([see TECHNICAL_HIGHLIGHTS §5](./TECHNICAL_HIGHLIGHTS.md#post-trade-polling)).

---

## Limit Orders

Shipped mid-engagement as a user-requested feature.

### UI

- Price input (slider + number input, clamped to [0.01, 0.99])
- Expected cost display (price × size + fees)
- Locked balance display (shows USDC that'll be held until fill/cancel)
- Open orders panel with status + cancel button

### Backend

Limit orders go through the CLOB as `GTC` (good-till-cancelled). Locked balance tracked server-side:

```typescript
lockedBalance = openOrders.reduce((sum, o) => sum + o.price * o.remainingSize, 0)
availableBalance = walletBalance - lockedBalance
```

### Open Orders aggregation

In the portfolio view, open orders across all markets are aggregated:

```
Open Orders
├─ Trump wins 2028 · BUY YES @ $0.42 · 100 shares · $42 locked
├─ ETH > $5k by EOY · BUY YES @ $0.18 · 200 shares · $36 locked
└─ Fed rate cut March · SELL NO @ $0.71 · 50 shares · $35.50 locked

Total locked: $113.50
```

This aggregation was a separate feature — previously open orders were only visible on each market's detail page.

---

## WebSocket Integration

Protocol B exposes a WebSocket feed for:

- Orderbook updates (per-market)
- Price updates (per-token)
- Market lifecycle events (`marketResolved`, `marketExpired`)
- User-specific events (`order_filled`, `order_cancelled`)

### Client implementation

```typescript
const ws = useLimitlessWebSocket()

useEffect(() => {
  if (!ws.connected) return

  ws.subscribe('orderbook', tokenId, (update) => {
    queryClient.setQueryData(['limitless', 'orderbook', tokenId], update)
  })

  ws.subscribe('price', tokenId, (price) => {
    queryClient.setQueryData(['limitless', 'price', tokenId], price)
  })

  ws.subscribe('lifecycle', tokenId, (event) => {
    if (event.type === 'marketResolved') {
      // patch cache with winningOutcomeIndex
      queryClient.setQueryData(['limitless', 'market', tokenId], (prev) => ({
        ...prev,
        resolved: true,
        winningOutcomeIndex: event.winningOutcomeIndex,
      }))
    }
  })

  return () => { ws.unsubscribe('orderbook', tokenId); /* ... */ }
}, [ws.connected, tokenId])
```

### Market-resolved race

When a market resolves, the WS fires `marketResolved` but REST may still return the market as open for several seconds. Fixed by patching the client cache immediately on WS event — the next render reads from the patched cache, not from the stale REST response.

### Polling cadence reductions

With WS in place, we reduced polling:

| Resource | Old interval | New interval |
|---|---|---|
| Orderbook | 1s | 30s |
| Chart prices | 2s | 10s |

Net: a large reduction in background API load for Protocol B — the remaining polling is only a safety-net cadence behind the WS-driven updates.

---

## Redeem Flow

### Trigger

A position becomes redeemable when its market resolves. UI shows:

- "REDEEM" button (green, replacing the value column)
- "Awaiting resolution" label for markets with `endDate < now` but not yet resolved by oracle

### Execution

- RelayClient for gas-sponsored redeem (so users don't need BASE/MATIC for gas)
- Fetch `conditionId` from both camelCase and snake_case fields in upstream API (inconsistency)
- Skip `fetchSetup` in ethers provider for the redeem endpoint (non-default provider behavior)
- NegRisk adapter approval check (for NegRisk markets, redeem goes through a separate adapter)

### Post-redeem UX

- Hides both YES and NO sides of the resolved market across portfolio + trade-view (was hiding only one side, leading to confusion)
- Records redeem as a SELL in `trade_history` with `actual_sell_value = on_chain_payout`
- Notification dedup by `tokenId` in metadata jsonb (multiple listeners could fire on one redeem)

### "Expired" vs "Worthless" vs "Redeemable"

Three states that look similar but mean different things:

| State | Condition | UI |
|---|---|---|
| Expired | `endDate < now`, `acceptingOrders = false`, `currentPrice = 0`, outcome is lost | Red "Expired" label, hide button |
| Worthless | `value < $0.01`, not yet resolved | Hidden by default, unhide toggle |
| Awaiting | `endDate < now`, `acceptingOrders = false`, not yet resolved | "Awaiting resolution" label |
| Redeemable | Resolved, won outcome | Green "REDEEM" button |

Each state has its own derivation logic. I consolidated this into a single `derivePositionState()` function after an earlier bug where expired and awaiting were conflated.

---

## Portfolio & Position Reconciliation

See [ARCHITECTURE.md §On-Chain Position Reconciliation](./ARCHITECTURE.md#on-chain-position-reconciliation) for the full algorithm.

### Key rule

- **On-chain CTF balance = canonical source for open shares**
- **DB trade history = canonical for cost basis and PnL**

### Open orders aggregation

New in Phase 3 — previously `positions` and `openOrders` were shown separately. Now open orders roll up into the portfolio total (locked balance displayed).

### Safe auto-deployment

First-time withdrawers may not have a deployed Safe on Polygon. The withdraw flow:

1. Check if Safe is deployed
2. If not, deploy (gas-sponsored via RelayClient)
3. Fund the Safe with user's funds
4. Trigger withdraw

**Time-box to 2 minutes.** If Safe deployment hangs (rare, but we've seen it), return HTTP 202 to the frontend with a "deployment in progress" state. Frontend polls. This prevents the UI from hanging on a slow deploy.

---

## Live Crypto, Sports, Group Markets

### Live-crypto page

2-phase fetch for cold start (see [TECHNICAL_HIGHLIGHTS.md](./TECHNICAL_HIGHLIGHTS.md)):

1. Phase 1: markets + prices — render tiles ASAP (~1.3s)
2. Phase 2: positions + metadata — populate side panels (~4s more)

Loading state includes Limitless platform specifically — previously the "live" indicator only reflected the default platform.

### Sports leagues

Added sports leagues filter (NBA / NFL / soccer / etc) to Explore with league badges. Test suite covers filter combinations.

### Group markets

Group markets are multi-market events (e.g. "Who wins the election?" with 20 candidates as sub-markets). Integration specifics:

- Parent image inherited by sub-positions (instead of empty avatar)
- `globalThis` cache for group-market images — shared across views (event detail, portfolio, market list) without refetching
- Hide Price/Probability toggle for multi-market group charts (doesn't make sense with 20 lines)

### Oracle chart

Live-crypto markets pull price data from an oracle feed. Chart renders a thin line of the oracle price over the observation window, with horizontal threshold lines at each market's strike. Helpful for users to see "how far am I from the threshold?"

---

## Performance Work

Dedicated performance pass shipped mid-April, bundling several independent improvements into a single deploy.

### Rate-limit resilience

- Exponential backoff on 429 responses
- Fewer pagination pages (was fetching all 20, now fetches 10 and paginates on user scroll)
- Page 9-10 capture for weekly markets (were being dropped due to pagination overflow)
- Search vs Gold/Silver separate indices (category leak between crypto and commodities)

### Polling reductions

- Crypto-markets server cache 30s → 10s (faster 15-min rollover)
- Limitless orderbook 1s → 30s (WS-driven)
- Limitless chart 2s → 10s (WS-driven)

### 2-phase crypto fetch

- Cold start reorganized so market tiles render before positions/metadata finish fetching (previously a fully-serial chain blocked the whole page).
- Prefetch Limitless + Myriad data regardless of platform filter.

### Concurrent close guard

- Nonce collision guard (see [TECHNICAL_HIGHLIGHTS.md §6](./TECHNICAL_HIGHLIGHTS.md#concurrent-close))
- Fail-closed RPC fallback

### Just-closed flash fix

- Post-SELL polling (see [TECHNICAL_HIGHLIGHTS.md §5](./TECHNICAL_HIGHLIGHTS.md#post-trade-polling))

---

## Testing

- Comprehensive Limitless test suite — quote, order submit, cancel, orderbook parse, WS reconnect
- Position refresh tests (post-trade polling scenarios)
- Chart fix verification (price-history, interval switching)
- Filter + search tests (sports leagues, crypto, esports leak)
- Redeem state tests (tokenId dedup, hide logic)

---

*Next: [COPY_TRADING_SYSTEM.md](./COPY_TRADING_SYSTEM.md) for the upstream-WS-driven copy-trading system.*
