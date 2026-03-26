# Rebtel Wallet — Product Requirements

**Live prototype:** [View on Vercel](https://wallet-prototype.vercel.app) | **Source:** `index.html`

---

## What is this

A closed-loop digital wallet for Rebtel customers. Customers can deposit funds and pay for Rebtel services (calling, top-ups, subscriptions) from a single balance. Withdrawals are not supported.

The Wallet becomes the primary payment experience across the app — visible on every main tab, used at every checkout.

---

## Screens and what they do

### 1. Home tab
- Calling activity cards (recent calls with flags, minutes left, call-again CTA)
- No wallet-specific content on this tab

### 2. Services tab
- **Wallet card** — shows balance + "Add funds" button (taps through to Wallet detail)
- **Service grid** — Rebtel Wireless, International Calling, Mobile Top-up (2-column grid)

### 3. Account tab
- Greeting ("Hello, Maria Garcia")
- **Wallet card** — same component as Services tab, balance + "Add funds"
- **Calling card** — shows calling credits (separate from wallet balance) with amount in teal (#0C8489), country flag, and minutes remaining (e.g. "53/100 minutes left")
- Account menu: Subscriptions, Invite, Activity, Recaps, Settings, Help

### 4. Wallet detail (sub-screen of Account)
- **Balance header** — large balance display + "Add funds" button
- **Discounts section** — list of active discounts with status badges, "See all" link
- **Payment methods section** — ordered list with primary badge, "Manage" link
- **Recent transactions** — last 3 entries with type icons, amounts, "See all" link

### 5. Add Funds flow (3 steps)
- **Step 1:** Quick-select amounts ($10 / $20 / $50) or custom input. Payment method selector (radio, priority order). "Deposit cash at a store" teaser at bottom (coming soon, US only — future Incomm integration)
- **Step 2:** Confirmation screen — amount, method, projected new balance
- **Step 3:** Success or error state with clear next action

### 6. Checkout
- Product line item, wallet balance applied, discount applied (largest auto-selected), remaining charged to payment method
- Single "Pay [amount]" CTA
- Success/error states

### 7. Payment Methods management
- Ordered list with drag-to-reorder
- Add new method (tokenized via Payment Vault)
- Remove method (blocked if last method with active subscriptions)

### 8. Transaction History
- Paginated reverse-chronological list
- Tap for detail view (amount, method, date, description)
- "Request Refund" on eligible transactions — reason selector + optional note

### 9. Discounts
- Read-only list: active, used, expired
- Each shows: value, applies-to, expiry date, status badge

### 10. Subscriptions
- Active/paused list
- Pause/resume with confirmation modal
- Shows plan name, amount, next billing date

---

## For designers

### Design system
- All tokens are in `design-system/tokens.css` (colors, typography, spacing, radii, shadows)
- Fonts: **Pano** (display/headings), **KH Teka** (body/UI)
- Brand red: `#E31B3B`
- Calling credits use teal: `#0C8489`

### Tab bar
- 3 tabs: Home, Services, Account
- Icons from Rebtel 3.0 design system (SVG, 56x56 display) with distinct active (filled black) and inactive (grey outline) states
- Tab bar height: 80px

### Key components to design
- **Wallet card** — reused on Services and Account tabs (balance + CTA)
- **Calling card** — Account tab only (calling credits + flag + minutes)
- **Discount items** — icon + title + detail + status badge
- **Transaction rows** — type icon + name + detail + amount (green positive, red negative)
- **Payment method rows** — card brand icon + last four + expiry + optional "Primary" badge

### States to account for
- Empty wallet (no payment method, no balance)
- Has payment method but $0 balance
- Has balance (current prototype state)
- Add funds: success and error
- Checkout: success and error
- Refund: request submitted

### Copy rules
- Always say "customer", never "user"
- Wallet actions: "Add funds" (not "top up" or "deposit")
- Keep CTAs short: "Add funds", "Pay $X.XX", "Request refund"

---

## For engineers

### Architecture
- **Ledger service** — single source of truth for all balances. Immutable transaction log. No other service may mutate balances directly.
- **Payment Vault** — PCI-compliant tokenization. Wallet stores tokens only, never raw card data.
- **Hydra** — internal discount/offers engine. Wallet reads eligible discounts per customer.
- **Fraud Platform** — receives events on every balance-affecting transaction (method, amount, type, customer_id).

### Data models

| Entity | Key fields |
|---|---|
| Wallet | customer_id, balance (decimal), currency (ISO 4217) |
| LedgerEntry | wallet_id, type (PURCHASE/TOP_UP/REFUND/ADJUSTMENT/DISCOUNT), amount, payment_method_id, description — **immutable** |
| PaymentMethod | customer_id, type (CARD/BANK/PAYPAL), token, last_four, expiry, priority_order |
| Discount | customer_id, value, value_type (PERCENT/FIXED), applies_to, status (ACTIVE/USED/EXPIRED), expires_at, source (HYDRA) |
| Subscription | customer_id, plan_name, status (ACTIVE/PAUSED), next_billing_date, amount, currency |
| Refund | ledger_entry_id, reason, status (REQUESTED/APPROVED/REJECTED) |

### API endpoints

```
GET    /wallet/:customer_id                        → balance, currency
POST   /wallet/:customer_id/topup                  → { amount, payment_method_id } → new balance
GET    /wallet/:customer_id/transactions            → paginated ledger entries
GET    /wallet/:customer_id/payment-methods         → ordered list
POST   /wallet/:customer_id/payment-methods         → add tokenized method
DELETE /wallet/:customer_id/payment-methods/:id     → remove method
PATCH  /wallet/:customer_id/payment-methods/reorder → { payment_method_ids[] }
GET    /wallet/:customer_id/discounts               → from Hydra
GET    /wallet/:customer_id/subscriptions           → list all
PATCH  /wallet/:customer_id/subscriptions/:id       → { status: ACTIVE|PAUSED }
POST   /wallet/:customer_id/refunds                 → { ledger_entry_id, reason, note }
```

All endpoints authenticated. Error responses include `message` and `suggested_action`.

### Key rules
- **Closed-loop**: deposits only, no withdrawals, ever
- **Discount logic**: largest eligible discount auto-applied at checkout; customer can change or skip
- **Subscription fallback**: uses payment methods in priority order, no single "default"
- **Race conditions**: top-up and purchase flows must prevent double-spend or lost funds
- **Cash-in (Incomm)**: not in v1, but architecture must support plugging it in (US-only, feature-flagged)
- **Performance**: p95 < 300ms for all Wallet/Checkout/Transaction APIs
- **Privacy**: GDPR/CCPA compliant, all payment data via Vault

### Open questions
1. Min/max top-up amounts (needs fraud + finance input)
2. Refund SLA — what do we tell customers?
3. Which currencies for v1? FX handling?
4. Hydra API contract (exact fields for discount payloads)
5. Fraud platform event schema
6. Subscription pause — prorate or defer?

---

## Phasing

| Phase | Scope | ~Duration |
|---|---|---|
| 1 | Ledger service, data models, payment methods CRUD, basic wallet UI | 1 week |
| 2 | Add funds flow, top-up integration, transaction history | 1 week |
| 3 | Discounts (Hydra), checkout, subscriptions, refunds | 1–2 weeks |
| 4 | Cash-in architecture (Incomm stub), extensibility review | Future |
