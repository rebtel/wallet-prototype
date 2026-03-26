# Rebtel Wallet PRD — Claude Code Export

### TL;DR

Rebtel Wallet is a mobile-first, closed-loop digital wallet for diaspora customers globally, enabling easy funding, streamlined payments, unified transaction history, and modern subscription management for Rebtel’s calling and mobile top-up services. The Wallet is the leading UX and payments entry point, architected for seamless extensibility (discounts, cash-in, remittance, future banking features), and built atop a core backend Ledger service as the single source of truth. v1 launches without cash-in; architectural support for future features is required.

---

## CONTEXT FOR THE LLM

This document is the authoritative Product Requirements Document (PRD) for the Rebtel Wallet feature. Rebtel is a telecom service provider selling PSTN calls and mobile top-ups to a global diaspora audience (e.g., Latinos in the US, Nigerians in the UK). Claude Code should treat this document as the definitive specification for implementing the Wallet and related payment features.

**Key constraints for Claude Code:**

* The Wallet is closed-loop: customers can deposit funds, but cannot withdraw under any circumstance.

* The UX is mobile-first (iOS/Android and web), but must maintain complete parity.

* Wallet is the primary mental model: checkout is always a reflection of wallet state, never its own logic.

* The Ledger is the backend core service and absolute source of truth for the customer's balance.

* Cash deposit via Incomm is explicitly not in scope for v1 (US-only for future phases), but architecture must support integration.

* “Customer” is the correct term for end users in all customer-facing UI and copy.

* Claude Code must **not** invent new features not described here, and should **flag any ambiguity as an open question**, never assume or extrapolate.

---

## PRODUCT SUMMARY

---

## STRATEGIC RATIONALE

1. **Unlocking Cash-In:** The Wallet enables future cash-in via retail partners (e.g., Incomm)—a foundational capability for the US diaspora, positioning Rebtel to serve cash-heavy demographics.

2. **Consistent Customer Experience:** Every transaction—regardless of funding method—flows through a unified Wallet, ensuring clarity, transparency, and simple management of balance, payments, and offers.

3. **Enabling New Products:** With the Wallet and Ledger as payment backbone, Rebtel can onboard remittance, banking, stablecoins, and other value-added services without re-architecting key flows.

---

## CORE CONSTRAINTS

1. **Closed-loop wallet:** Deposits allowed; withdrawals are *not* allowed now or in the future.

2. **Cash deposit (Incomm):** Not a v1 feature—no UI or API, but architectural support required.

3. **Wallet leads UX:** Checkout always reflects wallet state; checkout does not own balance, discounts, or preferences.

4. **Single source of truth (Ledger):** The Ledger backend manages all customer balances and histories.

5. **Payment methods are tokenized:** Never store raw card/bank data.

6. **Discount logic:** Largest eligible discount auto-applied at checkout; customer may override or opt out.

7. **Subscription fallback:** Payment method priority list drives fallback logic; no “default method” label.

8. **Customer-first copy:** Always use “customer,” never “user,” in UI.

---

## DATA MODELS

**1. Wallet / Ledger**

* customer_id

* balance (decimal)

* currency (ISO 4217 code)

* created_at

* updated_at

**2. LedgerEntry (immutable)**

* id

* wallet_id

* type (enum: PURCHASE, TOP_UP, REFUND, ADJUSTMENT, DISCOUNT)

* amount (decimal, positive or negative)

* payment_method_id (nullable)

* description

* created_at

**3. PaymentMethod**

* id

* customer_id

* type (enum: CARD, BANK, PAYPAL)

* token (vault reference)

* last_four (nullable)

* expiry (nullable)

* priority_order (integer)

* created_at

**4. Discount**

* id

* customer_id

* value

* value_type (enum: PERCENT, FIXED)

* applies_to

* status (enum: ACTIVE, USED, EXPIRED)

* expires_at (nullable)

* source (HYDRA)

**5. Subscription**

* id

* customer_id

* plan_name

* status (enum: ACTIVE, PAUSED)

* next_billing_date

* amount

* currency

**6. Refund**

* id

* ledger_entry_id

* reason

* status (enum: REQUESTED, APPROVED, REJECTED)

* created_at

---

## BACKEND ARCHITECTURE

**Ledger (Core Wallet Service):**

* Maintains spendable balance per customer, in each customer’s local currency (multi-currency support).

* Stores *immutable* transaction history (all wallet-affecting events).

* Single authoritative backend for value across all Rebtel flows.

* **Integration Points:**

  * **Hydra:** Internal system; per-customer discount management, entitlement provisioning, etc.

  * **Internal Fraud Platform:** Hooks on every transaction, sending payment method, amount, type, and customer_id.

  * **Payment Vault:** PCI-compliant; tokenizes card/bank data (only tokens in Ledger).

  * **Future:** Incomm API for retail cash-in (feature-flagged, not active in v1).

* All balance mutations run via Ledger API; no other services may update customer balance directly.

* Ledger APIs are consumed by the Wallet UI and Checkout services.

---

## FEATURE SPECIFICATIONS

---

## USER EXPERIENCE

**Entry Point & First-Time User Experience**

* Customers encounter Wallet from main app navigation ("Wallet" tab, always visible pin).

* First-time: Short explainer ("Add funds and manage all your Rebtel payments"), then prompt to add first payment method.

* No lengthy onboarding; onboarding can be dismissed/skipped and revisited later.

**Core Experience**

* **Wallet Home Screen:**

  * *Empty State*: If no payment method or balance, show clear CTA to add payment method.

  * *Has Payment Method, No Balance*: Show balance ($0), prominent "Add Balance" button, list of payment methods, discounts row, link to transaction history.

  * *Has Balance*: Show balance, "Add Balance" button, payment methods, discounts summary, transaction history.

* **Add Balance Flow:**

  * Prompt for quick-select amount ($10, $20, $50) or custom amount entry.

  * Select payment method (list in priority order, top one pre-selected).

  * Confirm screen with amount, selected method, resulting new balance.

  * Complete action: show transaction result (success or error, with retry/alternate method if fail).

* **Checkout Flow:**

  * Purchase screen: show price, apply wallet balance as line item (if >$0), apply largest discount as line item (if eligible, with "Change" link), show remaining amount.

  * Payment method: top-priority pre-selected, can tap "Change" to select another.

  * Single prominent "Pay \[amount\]" CTA.

  * On success: show summary (amount paid, payment method, balance update); on failure, clear error and retry options.

* **Payment Methods Management:**

  * Payment methods displayed in order; up/down drag or tap to reorder.

  * Add new method (tokenization enforced by Payment Vault).

  * Remove (warn—cannot remove last active method with subscriptions).

* **Discounts Screen:**

  * Read-only list of current, used, expired discounts; each shows value, eligibility, expiry, status flag.

* **Subscription Management:**

  * List all active/paused; tap to detail.

  * Pause/resume with modals for confirmation.

  * Renewals use highest-priority available method.

* **Transaction History:**

  * Paginated reverse-chronological list; each tap opens detail.

  * "Request Refund" CTA on eligible transactions.

  * Refund: choose reason, optional note, confirmation modal.

**Advanced Features & Edge Cases**

* Transaction sync errors: Show friendly error with suggestion, log details for support/fraud.

* Removing last payment method with subscriptions: Block with modal, suggest adding a method first.

* Attempting to use expired card: Show error, prompt to update or reorder.

* Pausing a subscription with failed payment attempts: show next steps.

**UI/UX Highlights**

* Mobile-first design, responsive patterns preserved for desktop/web.

* Color contrast for balance/discounts/payment actions is WCAG-compliant.

* Prominent CTAs, minimal cognitive load at every step.

* Transactional actions always provide clear next steps in case of error.

---

## Narrative

Maria, a Nigerian living in London, uses Rebtel for affordable international calls and regular mobile top-ups for her family. Before Wallet, her payment experience was fragmented — she juggled discount codes, struggled to remember which card she used, and occasionally missed subscription charges when her primary method expired.

With Rebtel Wallet, Maria adds £20 using her debit card, sees her new balance instantly, and takes comfort knowing every purchase and subscription will use her balance first. At checkout for a top-up, her active “15% Welcome” discount is auto-applied, visibly saving her money. She can always tap in to review or skip a discount if she prefers.

When Maria’s card expires, she adds a new one and simply drags it to the top of her payment methods. Her next subscription uses the updated info automatically — no surprise missed payments. Whenever she checks her Wallet, her entire history is available in one spot, fostering trust. If something goes wrong, requesting a refund is just a couple of taps away, not a customer support ordeal. The Wallet brings clarity, security, and future readiness to Maria’s Rebtel experience.

---

## Success Metrics

**User-Centric Metrics**

* Wallet adoption rate (>35% of active customers use balance in first 90 days)

* Discount redemption (>70% of active discounts used at checkout)

* 30/90-day retention lift for Wallet users (measured vs. non-Wallet cohort)

* Subscription payment success rate improvement (-25% failed charges after 6 months)

**Business Metrics**

* Checkout conversion rate (increases tracked monthly)

* Transaction success rate (>98% of successful payment/checkout attempts)

* Payment costs (tracked monthly; % shift toward lower-cost/fraud-safe methods)

* Support ticket volume regarding payments (-20% within six months post-launch)

* Average wallet balance across actively funded customers

**Technical Metrics**

* Balance consistency (zero-rollbacks, every transaction reflected in Ledger and UI)

* End-to-end Wallet API response time (<300ms p95)

* Transaction log integrity (100% of payments/ledger events persisted and non-repudiable)

* Error rate on payment API (<1% across all integrations)

**Tracking Plan**

* wallet_opened

* payment_method_added (type)

* payment_method_removed

* payment_method_reordered

* discount_applied (value, type)

* discount_skipped

* discount_changed

* balance_topup_initiated (amount)

* balance_topup_completed (amount, payment_method_type)

* subscription_paused (plan_id)

* subscription_resumed (plan_id)

* refund_requested (ledger_entry_id, reason)

* checkout_completed (wallet_balance_used: bool, discount_applied: bool, amount)

---

## Technical Considerations

### Technical Needs

* Core backend: Ledger service (stores wallet entities, immutable ledger entries, manages all balance mutations)

* Front-end: Mobile apps (iOS/Android), responsive web app

* Internal APIs for Wallet UI, Checkout flow, Payment Methods, Subscriptions, Transaction History, Refunds, Discount info

* Data model extensible for future integration with cash deposits (Incomm), remittance, and banking services

### Integration Points

* Hydra (internal discount/offers management; handles per-customer eligibility)

* Payment Vault for tokenized payment details (PCI compliance)

* Internal Fraud Platform (events on every balance-affecting transaction)

* No Incomm/cash-in in v1, but ensure pluggable integration path exists

### Data Storage & Privacy

* All sensitive payment data tokenized/handled by PCI-compliant service (Vault)

* Ledger database stores value, histories; designed for immutability and auditability

* Full compliance with GDPR, CCPA, and other regional laws for financial data

### Scalability & Performance

* Anticipate growth to hundreds of thousands of active Wallets

* Target p95 API latency <300ms for all Wallet, Checkout, and Transaction actions

* All historical ledger entries must be rapidly paginable and queryable

### Potential Challenges

* Ensuring zero race conditions/duplication on top-up and purchase flows (funds must not be double-spent or lost)

* Real-time sync for balance across multiple platforms (app, web)

* Managing multi-currency wallet ledgers for global customers

* Eventual cash-in integration: design for extensibility and US-only rollout

---

## Milestones & Sequencing

### Project Estimate

* Medium: 2–4 weeks for v1 feature-complete (lean/agile/parallelized—assumes some pre-existing payments infra)

### Team Size & Composition

* Small Team: 2–3 people

  * 1 Product Owner for requirements, flows, and acceptance

  * 1 Backend Engineer (Ledger, Payment integration)

  * 1 Fullstack or Mobile/Web Engineer (UI, integration, analytics)

  * (If bandwidth allows) 1 part-time Designer or QA

### Suggested Phases

**Phase 1: Wallet Core+Payment Methods (1 week)**

* Deliverables: Ledger service, core data models, add/remove/reorder payment methods, functional APIs; simple UI for balance and methods

* Dependencies: Payment Vault for tokenization

**Phase 2: Transactions & Add Balance (1 week)**

* Deliverables: Add Balance flow, top-up integration, wallet mutation APIs, paginated transaction history, wallet status UI

* Dependencies: Completed Ledger base, internal events bus

**Phase 3: Discounts, Checkout, Subscriptions, Refunds (1–2 weeks)**

* Deliverables: Hydra discounts integration, checkout calculation logic, subscription management, refund initiation, all UI screens web+mobile

* Dependencies: Hydra API contract, backend fraud hooks, existing subscriptions infra

**Phase 4: Cash-In Integrability & Future-Proofing (future)**

* Deliverables: Architecture documentation, stub for Incomm/cash-in integration, extensibility review

---

## UX FLOWS (MACHINE-READABLE)

1. **Onboarding Flow**

  1. Show Wallet intro ("Easily manage Rebtel payments from your Wallet")

  2. Prompt: "Add Payment Method" (form)

  3. On submit, route to Wallet Home

2. **Wallet Home**

  * If no payment method and no balance: Show "Add Payment Method" CTA

  * If has payment method, no balance: Show "$0 balance", "Add Balance" CTA, list payment methods, discounts summary, transaction history link

  * If has balance: Show balance, "Add Balance" button, payment methods, discounts summary, link to transaction history

3. **Add Balance Flow**

  1. Prompt: Select quick amount ($10/$20/$50) or custom amount

  2. Payment method select (priority pre-selected)

  3. Confirm screen: show amount+method+projected new balance

  4. Tap "Add Balance" → API call

  5. Success: show updated balance, transaction history entry

  6. Failure: show error, option to retry or change method

4. **Checkout Flow**

  1. Display product price

  2. Subtract wallet balance as line item (if >$0)

  3. Subtract largest eligible discount as line item (if available, with "Change" link)

  4. Remaining amount charged to top-priority method (option to "Change")

  5. Show breakdown, "Pay \[amount\]" CTA

  6. On success: confirmation screen with payment details, balance, discount used

  7. On failure: show clear error, retry or edit payment option

5. **Payment Method Management**

  * List payment methods in current priority order

  * Drag-and-drop or buttons to reorder (priority for fallback/cascade)

  * Add method (secure tokenization form)

  * Remove method (if last+active subs, show warning, block action)

6. **Subscription Management**

  * List all subscriptions

  * Tap item for details

  * Pause or resume with confirmation dialog/modal (status update, impact explained)

7. **Transaction History**

  * Paginated reverse-chronological list

  * Tap row opens detail view (full breakdown, payment method, discounts)

  * Eligible transactions show "Request Refund" CTA

  * Submit refund: select reason, optional note, confirmation message

8. **Discounts Screen**

  * List all discounts

  * For each: value, applies-to, expiry, status (active/used/expired)

**All flows: Mobile-first layouts, error states provide clear next action, web parity maintained.**

---

## API SURFACE (SUGGESTED)

* GET /wallet/:customer_id — returns wallet balance, currency

* POST /wallet/:customer_id/topup — body: amount, payment_method_id; returns new balance, ledger_entry_id

* GET /wallet/:customer_id/transactions — paginated list of ledger entries

* GET /wallet/:customer_id/payment-methods — ordered list

* POST /wallet/:customer_id/payment-methods — add new tokenized method

* DELETE /wallet/:customer_id/payment-methods/:id — remove method

* PATCH /wallet/:customer_id/payment-methods/reorder — body: array of payment_method_ids (new order)

* GET /wallet/:customer_id/discounts — active, used, expired from Hydra

* GET /wallet/:customer_id/subscriptions — list all

* PATCH /wallet/:customer_id/subscriptions/:id — body: status (ACTIVE or PAUSED)

* POST /wallet/:customer_id/refunds — body: ledger_entry_id, reason, note  

  *All endpoints must be authenticated. Errors return message and suggested_action field.*

---

## OPEN QUESTIONS FOR ENGINEERING

1. **Minimum/Maximum top-up amount:** Not defined in v1, needs input from fraud and finance teams (prevents abuse/overspill).

2. **Refund SLA:** What expected resolution time do we promise to customers in the refund UI?

3. **Balance update latency:** What is the acceptable delay (ms/sec) for updated balances after a top-up or transaction?

4. **Currency support:** Which currencies are in scope for v1? How are foreign exchange rates handled for balance/top-up/purchase?

5. **Hydra API contract:** What exact fields and payloads are available for discount info, including applies_to, status, expiry?

6. **Fraud platform event schema:** What is the payload for transaction events (field list, required/optional)?

7. **Subscription pause behavior:** If pausing mid-billing, does this prorate or defer the next charge? (Impacts customer communication)

8. **Session/auth:** What authentication and authorization flow does the Wallet API use? Should match existing Rebtel mobile/web auth, but clarify integration.

---