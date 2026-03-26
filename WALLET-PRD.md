# Rebtel Wallet — Product Requirements

**Live prototype:** [View on Vercel](https://wallet-prototype.vercel.app) | **Source:** `index.html`

---

## What is this

A closed-loop digital wallet for Rebtel users. Customers deposit funds and pay for Rebtel services (calling, top-ups, subscriptions) from a single balance. No withdrawals.

The Wallet is the primary payment experience — visible on every main tab, used at every checkout.

---

## User stories

### Adding funds

**As a user, I want to deposit money into my Rebtel wallet from my preferred digital payment method** so that I have a balance ready to use for Rebtel services.

- I can pick a quick amount ($10 / $20 / $50) or enter a custom amount
- My payment methods are listed in my preferred priority order, with my top choice pre-selected
- Before confirming, I see the amount, the method being charged, and what my new balance will be
- On success, I see my updated balance immediately
- On failure, I get a clear error and can retry or pick a different method

**As a user, I want to deposit cash at a retail store** so that I can fund my wallet without a bank card.

- I see this option at the bottom of the Add Funds screen
- v1: shown as "Coming soon — US only" (Incomm integration, not yet active)

### Paying for services

**As a user, I want my wallet balance to be automatically applied when I buy something** so that I don't have to think about how to pay.

- At checkout, my wallet balance is subtracted as a line item
- The largest eligible discount is auto-applied (I can change or skip it)
- Any remaining amount is charged to my highest-priority payment method
- I can tap "Change" to pick a different method
- I see a full breakdown before I tap "Pay"

**As a user, I want to see a clear confirmation after a purchase** so that I know exactly what was charged and where.

- Success screen shows: amount paid, payment method used, discount applied, updated wallet balance
- Failure screen shows: what went wrong, option to retry or change payment method

### Managing payment methods

**As a user, I want to add, remove, and reorder my payment methods** so that I control how I pay.

- I can add cards (Visa, Mastercard), bank accounts, and PayPal
- Methods are listed in my chosen priority order — the top one is used first for subscriptions and checkout
- I can drag to reorder
- I can remove a method, unless it's my last one and I have active subscriptions (shown a warning)

### Viewing my wallet

**As a user, I want to see my wallet balance from the main app tabs** so I always know where I stand.

- The wallet card (balance + "Add funds") appears on both the Services and Account tabs
- Tapping the card takes me to the full Wallet detail screen
- Wallet detail shows: balance, discounts, payment methods, recent transactions

**As a user, I want to see my calling credits separately from my wallet balance** so I know how much calling time I have left.

- On the Account tab, a separate "Calling" card shows calling credits in teal, with a country flag and minutes remaining (e.g. "53/100 minutes left")
- This is distinct from the main wallet balance

### Transaction history

**As a user, I want to see a history of all my wallet transactions** so I can track my spending.

- Transactions are listed newest-first, paginated
- Each entry shows: type icon, description, date, and amount (green for deposits, red for charges)
- I can tap any transaction to see the full detail

**As a user, I want to request a refund on an eligible transaction** so I can get my money back if something went wrong.

- Eligible transactions show a "Request refund" button
- I select a reason and can add an optional note
- I get confirmation that the request was submitted

### Discounts

**As a user, I want to see what discounts I have available** so I know what savings I can use.

- The Wallet detail screen shows my active discounts with a "See all" link
- The full discounts screen lists all discounts: active, used, and expired
- Each shows: value, what it applies to, expiry date, and status

**As a user, I want my best discount automatically applied at checkout** so I always get the biggest saving without effort.

- The largest eligible discount is auto-applied as a line item
- I can tap "Change" to pick a different discount or skip it entirely

### Subscriptions

**As a user, I want to manage my subscriptions from the wallet** so I have control over recurring charges.

- I see all my subscriptions (active and paused) with plan name, amount, and next billing date
- I can pause or resume a subscription with a confirmation step
- Renewals are charged to my payment methods in priority order

---

## Constraints

- **Closed-loop** — deposits only, no withdrawals, ever
- **Tokenized payments** — raw card/bank data never stored, only vault tokens
- **User-facing copy** — always say "user" in UI and documentation
- **Ledger is source of truth** — no other service may mutate balances
- **Cash-in not in v1** — architecture must support it for future US rollout (Incomm)

---

## For designers

- Design tokens: `design-system/tokens.css`
- Fonts: **Pano** (display), **KH Teka** (body)
- Brand red: `#E31B3B` | Calling credits teal: `#0C8489`
- Tab icons: Rebtel 3.0 SVGs, active (filled black) / inactive (grey outline)
- CTA copy: "Add funds", "Pay $X.XX", "Request refund"
- Design all states: empty wallet, zero balance, funded, success, error

---

## For engineers

### Architecture
- **Ledger** — balance + immutable transaction log per user
- **Payment Vault** — PCI-compliant tokenization
- **Hydra** — discount/offers engine (read-only from Wallet)
- **Fraud Platform** — event on every balance-affecting transaction

### API surface

```
GET    /wallet/:user_id                        → balance, currency
POST   /wallet/:user_id/topup                  → { amount, payment_method_id }
GET    /wallet/:user_id/transactions            → paginated ledger entries
GET    /wallet/:user_id/payment-methods         → ordered list
POST   /wallet/:user_id/payment-methods         → add tokenized method
DELETE /wallet/:user_id/payment-methods/:id     → remove method
PATCH  /wallet/:user_id/payment-methods/reorder → { payment_method_ids[] }
GET    /wallet/:user_id/discounts               → from Hydra
GET    /wallet/:user_id/subscriptions           → list all
PATCH  /wallet/:user_id/subscriptions/:id       → { status: ACTIVE|PAUSED }
POST   /wallet/:user_id/refunds                 → { ledger_entry_id, reason, note }
```

### Key technical rules
- Prevent double-spend and lost funds on top-up/purchase (no race conditions)
- p95 < 300ms for all Wallet/Checkout/Transaction APIs
- GDPR/CCPA compliant, all payment data via Vault
- Subscription fallback uses priority-ordered methods, no single "default"
- Incomm cash-in: feature-flagged stub, not active in v1

### Open questions
1. Min/max top-up amounts (needs fraud + finance input)
2. Refund SLA — what do we promise users?
3. Which currencies for v1? FX handling?
4. Hydra API contract (exact discount payload fields)
5. Fraud platform event schema
6. Subscription pause — prorate or defer?

---

## Phasing

| Phase | Scope | ~Duration |
|---|---|---|
| 1 | Ledger service, data models, payment methods CRUD, basic wallet UI | 1 week |
| 2 | Add funds flow, top-up integration, transaction history | 1 week |
| 3 | Discounts (Hydra), checkout, subscriptions, refunds | 1-2 weeks |
| 4 | Cash-in architecture (Incomm stub), extensibility review | Future |
