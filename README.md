# Payment API Design Patterns

A product manager's guide to designing reliable payment APIs, written from experience shipping 10 payment APIs across 6 East African markets at Equity Bank Group and architecting cross-border payment infrastructure at Betaling Africa.

This is not an engineering reference. It is a record of the product decisions, failure modes, and design rationale behind payment APIs that actually work in production, especially in African and emerging markets where infrastructure is unreliable, regulations vary per corridor, and users have low tolerance for opaque errors.

---

## Who this is for

- PMs designing payment products who need to write technically credible specs
- Engineers building payment infrastructure who want the product reasoning behind common patterns
- Founders scoping an MVP payment system for the first time

---

## Contents

1. [Virtual Account Reconciliation](#1-virtual-account-reconciliation)
2. [Webhook Design and Reliability](#2-webhook-design-and-reliability)
3. [Rate Locking and FX Engine Design](#3-rate-locking-and-fx-engine-design)
4. [KYC Tiering and Limit Enforcement](#4-kyc-tiering-and-limit-enforcement)
5. [Partial Payment Detection](#5-partial-payment-detection)
6. [Idempotency in Payment APIs](#6-idempotency-in-payment-apis)
7. [Transfer State Machines](#7-transfer-state-machines)
8. [Mobile Money Integration Patterns](#8-mobile-money-integration-patterns)
9. [Compliance API Integration (KYC/KYB)](#9-compliance-api-integration-kyckyb)
10. [PSP Abstraction Layer](#10-psp-abstraction-layer)
11. [API Design for B2B Supplier Payments](#11-api-design-for-b2b-supplier-payments)
12. [Transaction Limits and Velocity Controls](#12-transaction-limits-and-velocity-controls)

---

## 1. Virtual Account Reconciliation

### What it is

A virtual account is a unique bank account number generated per transaction. The user pays into it; the system detects the inbound payment and triggers the outbound transfer. It is the primary mechanism for bank-based payment collection in Nigeria (Wema, Providus), Kenya, Ghana, and several other African markets.

Virtual accounts solve a specific problem: African consumers are far more likely to pay via bank transfer than by card. In Nigeria, over 60% of digital payment volume moves via bank transfers. Building a payment product that only accepts cards is building for a minority of the market.

### The core design rule

**One virtual account per transaction_id. Never reused.**

Reusing virtual accounts is the most common reconciliation failure in African payment systems. If a virtual account is reused across transactions, an inbound payment with a missing or incorrect reference becomes impossible to attribute correctly. The operational cost of resolving misattributed payments is significant and grows non-linearly with volume.

### Reconciliation logic

Match inbound payments using two signals in priority order:

1. **Narration reference**: the transaction reference the user is asked to include in their transfer narration
2. **Virtual account number + amount**: fallback if the reference is missing or malformed

| Scenario | Action |
|---|---|
| Narration reference matches and amount matches | Auto-confirm. Proceed with outbound transfer |
| Account number matches, amount matches, reference missing | Flag for review. Do not auto-confirm |
| Amount received is less than expected | Partial payment detected. Hold and notify user |
| Amount received is more than expected | Hold excess as pending credit. Notify user |
| No payment received within 24 hours | Auto-cancel. No fees charged. Notify user |

Auto-confirming an uncertain match is worse than a delayed correct one. When in doubt, flag for manual review.

### Expiry

Virtual accounts expire after 24 hours. After expiry: the account is deactivated and cannot accept new payments; the transaction auto-cancels; the user is notified and given the option to restart; no fees are charged.

Why 24 hours and not less? In Nigeria, bank transfers between different banks can take 30 minutes to 4 hours due to NIP settlement windows. In some cases, bank network issues extend this further. 24 hours is the right balance: short enough to limit exposure, long enough that a slow interbank transfer still lands.

### Providers

In Nigeria, Wema Bank and Providus Bank are the two dominant virtual account providers for fintechs. Selection criteria: NIP webhook reliability, reconciliation API quality, and operational support responsiveness. Providus is generally preferred for high-volume consumer-facing products; Wema has historically been stronger for enterprise accounts.

### API shape

```
POST /transactions
Response: {
  transaction_id,
  virtual_account_number,
  virtual_account_bank,
  expires_at,
  reference
}

GET /transactions/{id}/virtual-account
Returns: virtual account details if still active
Returns: 410 Gone if the virtual account has expired
```

### What goes wrong in production

**User omits the reference**: This is more common than expected. Users copy the account number correctly but forget to include the reference in the narration field. Match by account number and amount. If amount matches but reference is missing, flag for review rather than auto-confirm.

**Duplicate inbound payments**: If two inbound payments arrive for the same virtual account (a user accidentally sent twice), hold both and alert the operations team. Do not auto-process either.

**Bank webhook delay**: Webhooks from Nigerian banks can be delayed by 15-30 minutes during peak hours (8am-10am, 12pm-2pm WAT). Do not rely solely on webhooks. Run a reconciliation polling job every 5 minutes as a fallback against the bank's transaction API.

**Reference field truncation**: Some Nigerian banks truncate transfer narrations beyond a certain character length. Keep transaction references short (under 12 characters) and numeric-only where possible. Alphanumeric references with special characters frequently get corrupted in transit.

---

## 2. Webhook Design and Reliability

### The failure mode nobody plans for

Webhooks fail silently. The destination server times out, returns a 500, or is simply down for maintenance. A payment system that treats webhooks as fire-and-forget will silently drop status updates, leaving users with transactions stuck in Processing indefinitely. This is not a theoretical failure: it is a common production problem that creates a disproportionate share of support tickets.

At Equity Bank, we processed 10 payment APIs across 6 East African markets. Webhook reliability was one of the top three operational problems in the first 90 days. The solution is always the same: webhooks plus polling, retries with backoff, and idempotent consumers.

### Design requirements

**Retry with exponential backoff**

Retry failed webhook deliveries at the following intervals: 1 minute, then 5 minutes, then 30 minutes, then 2 hours, then 6 hours. After 5 failed attempts, escalate to the operations queue. Do not retry indefinitely: unbounded retries create cascading load during outage recovery and can trigger rate limits on the receiving end.

**Idempotency on receipt**

Webhook consumers must handle duplicate deliveries. The same event can be delivered more than once, either from retries or from the upstream provider sending duplicates (this is standard behaviour for Safaricom Daraja, Flutterwave, and Paystack). Deduplicate on event_id before processing. If you have already processed an event_id, return a 200 without reprocessing.

**Event ordering**

Do not assume webhooks arrive in order. A Delivered event may arrive before a Processing event if there is a delivery delay on the first event. This is not a hypothetical: it happens regularly when there are network partitions between the provider and your server. Build state machines that accept out-of-order transitions gracefully (see Section 7).

**Signature verification**

Every webhook payload must be signed by the provider. Verify the HMAC signature before processing. Reject unsigned or incorrectly signed payloads with a 401. Log the rejection with the raw payload for audit purposes. Paystack, Flutterwave, and Safaricom Daraja all provide HMAC-SHA512 or HMAC-SHA256 signatures.

### Webhook vs polling strategy

Use WebSockets as the primary real-time channel and HTTP polling as the fallback for client-side status updates:

```
WS  /ws/transactions/{id}        Primary. Real-time push.
GET /transactions/{id}/status    Fallback. Poll every 15 seconds.
```

On the server side, use webhooks from PSPs as the primary update trigger and a scheduled reconciliation job as the fallback. The reconciliation job should run every 5 minutes and check for transactions that have been in Processing for longer than the expected settlement time without a status update.

### What to log

Log every webhook receipt with: event_id, provider, event_type, timestamp, processing_result (accepted/duplicate/rejected), and latency from event timestamp to receipt timestamp. This data is essential for debugging reconciliation failures and for demonstrating to regulators that your AML monitoring is functioning.

---

## 3. Rate Locking and FX Engine Design

### The problem with live rates

You cannot show a user a rate, let them go through a 3-step payment flow, and then charge them a different rate on confirmation. That is how you lose trust permanently. But you also cannot guarantee a rate indefinitely: FX rates in NGN, KES, and GHS corridors can move 0.5-2% within a single hour during periods of volatility.

The solution is a time-bounded rate lock with a transparent countdown and a defined tolerance band for silent refreshes.

### The 10-minute lock

Lock the exchange rate for 10 minutes from the point it is fetched. Return a rate_expires_at timestamp from the API and use this to drive the client-side countdown timer. Do not calculate the expiry client-side from a duration constant: clocks drift, especially on mobile devices with unreliable network connections.

```
GET /rates/{source}/{destination}
Response: {
  rate,
  fee,
  rate_expires_at,   // ISO 8601 timestamp. Use this on the client, not a calculated duration.
  estimated_delivery
}
```

Poll this endpoint every 60 seconds. Cache the response locally with expiry matching rate_expires_at.

### The silent refresh band

Not every rate movement should interrupt the user. Define a tolerance band (typically 0.5%) within which the rate can be silently refreshed without requiring user action. Outside the band, pause the flow and require the user to actively accept the new rate or cancel.

Store the tolerance band in the database, not in code. You will need to adjust it per corridor and per market condition (NGN/USD volatility in 2024 required a wider band than KES/USD during the same period) without deploying.

### Why source matters for rate pricing

Betaling sources base exchange rates from Binance and Bybit spot prices. This is a deliberate choice. In African markets, the gap between official CBN/CBK rates and market rates can be significant: 15-40% in NGN at various points in 2023-2024. Using spot prices means your rate is market-realistic, not artificially suppressed. The compliance implication is that you need to document your rate source and margin methodology clearly for regulators.

### Rate clock UX states

The countdown timer needs colour states to communicate urgency without interrupting the flow:

- Green: more than 5 minutes remaining
- Amber: 1-5 minutes remaining
- Red: under 1 minute remaining
- Expired: rate refreshed. User must accept if outside tolerance band

Display the countdown on every screen in the payment flow from amount entry through confirmation. Users need to understand that they are operating inside a time window.

### Fee and rate separation

Never blend the fee into the exchange rate. Always show them as separate line items:

```
Exchange rate:     1 USD = 1,540 NGN
Transfer fee:      NGN 2,500
You send:          NGN 157,500
Recipient gets:    USD 100.00
```

Blending the fee into the rate is the practice that makes traditional African bank wires expensive and opaque. It is also increasingly a regulatory concern: CBN's consumer protection framework and CBK's guidelines both push for fee transparency. Building transparent pricing from the start is good product design and good compliance design.

---

## 4. KYC Tiering and Limit Enforcement

### Design principle

Enforce limits at transaction submission, not at registration. Blocking users from registering because they cannot complete full KYC immediately destroys top-of-funnel conversion. Let them in at the Starter tier and upgrade them when they hit the limit naturally. The upgrade prompt at the point of limit is a far better conversion moment than a registration-time gate: the user has demonstrated intent and just needs to clear the compliance hurdle to complete what they already want to do.

### Tier structure

| Tier | Monthly limit | Requirements | Verification |
|---|---|---|---|
| Starter | $500 | Phone OTP plus BVN or national ID number | Instant API |
| Verified | $5,000 | Government photo ID plus liveness selfie | YouVerify / automated |
| Business | $50,000 | Full KYB plus compliance sign-off | Manual review mandatory |
| Custom | Above $50,000 | Account manager | No self-service path |

### API pattern

```
GET /limits/{user_id}
Response: {
  monthly_limit,
  monthly_used,
  kyc_tier,
  days_until_reset,
  upgrade_available   // bool. Used to show upgrade prompt proactively on dashboard
}
```

Check this before rendering the confirmation screen, not after submission. Showing the upgrade prompt before the user has filled in all their transfer details is a worse experience than blocking at submission, but blocking at submission without prior warning is worse still. Surface the limit status on the dashboard with a progress indicator.

### Upgrade prompt timing

When a user hits their Starter limit mid-transfer, do not discard their transfer data. Preserve the transfer state, launch the upgrade flow as an overlay, and resume the transfer after verification completes. Losing transfer data at the KYC upgrade step is a significant drop-off point and one of the most common causes of abandonment in African consumer payment products.

### BVN and NIN verification context

In Nigeria, BVN (Bank Verification Number) is the 11-digit identity anchor for the Nigerian financial system. Every Nigerian with a bank account has one. BVN validation via API returns the registered name and date of birth, which can be matched against the user's registration details. NIN (National Identification Number) is the national identity number and is increasingly required by CBN for financial accounts. For Starter tier verification, BVN lookup is the fastest and most reliable approach. For Verified tier, government ID plus liveness check via YouVerify or Smile Identity is the standard.

---

## 5. Partial Payment Detection

### Why it happens

In bank transfer flows, users sometimes send the wrong amount. This is more common than you would expect. Causes include: miscalculating the total (forgetting the fee), sending from an app that rounds to the nearest unit, or copy-pasting the wrong amount. Without partial payment detection, the transaction either auto-fails (leaving the user's money in limbo) or auto-processes at the wrong amount (creating a compliance and reconciliation problem).

### Behaviour by scenario

| Scenario | Behaviour |
|---|---|
| Full amount received | Auto-confirm. Proceed with outbound transfer |
| Short payment (any amount under expected) | Hold. Notify user of shortfall. Wait for top-up or cancellation |
| Overpayment received | Hold excess as pending credit. Notify user. Credit to wallet on confirmation |
| No payment within 24 hours | Auto-cancel. Notify user. No fees charged |

### Shortfall handling in detail

When a shortfall is detected, do not auto-cancel. Give the user 24 hours from detection to send the remaining amount to the same virtual account. Send a notification with the exact shortfall amount. If the virtual account has expired by the time the user tries to top up, generate a new virtual account for the remaining amount only (not the full original amount).

### Why overpayments need special handling

Do not auto-refund overpayments to the source account without user instruction. Refunding requires a separate outbound transaction, which has its own fee. The correct behaviour is to hold the excess as a wallet credit and let the user decide what to do with it: apply it to the next transfer or request a refund. This also avoids a class of fraud where users intentionally overpay and then request a refund to a different account.

---

## 6. Idempotency in Payment APIs

### The rule

Every payment initiation endpoint must accept an idempotency key. If the same key is submitted twice (due to a network retry, a double-tap, or a client-side bug), return the result of the first request rather than processing a second payment.

```
POST /transactions
Headers:
  Idempotency-Key: {client-generated UUID v4}

Response (first call):
  HTTP 201 Created
  { transaction_id, status, ... }

Response (duplicate call, same key):
  HTTP 200 OK
  { transaction_id, status, ... }  // Same response as first call
```

Store idempotency keys with a 24-hour TTL. Return the original response for any duplicate submission within that window.

### Why this is critical in African markets

Mobile network reliability is inconsistent across Nigeria, Kenya, and Ghana. A user on a slow 3G connection who taps "Pay" may not receive a response before their connection drops. The natural behaviour is to tap again. Without idempotency, they get charged twice. With idempotency, the second request returns the existing transaction.

This is especially critical for wallet debits and card payments where the money moves instantly on the first request. For bank transfer flows with virtual accounts, the risk is lower (the account has not been paid yet), but for card and wallet payments, idempotency is a hard requirement before launch.

### Generating idempotency keys

The client generates the key before making the request, not the server. Use UUID v4. Generate once per user intent (i.e., when the user taps Pay), not per API call. If the user cancels and starts a new transfer, generate a new key. Do not reuse keys across different user actions.

---

## 7. Transfer State Machines

### Define explicit states with no ambiguity

Every transfer should move through a defined set of states. Any state not on the list should not exist in production. Ambiguous states (Processing_maybe, Possibly_sent) are not states: they are bugs.

**Individual remittance transfer (bank transfer method):**
```
Awaiting Payment
  --> Payment Received
      --> Processing
          --> Sent to Recipient's Bank
              --> Delivered
              --> Failed
```

**Business supplier payment (bank transfer method):**
```
Awaiting Your Payment
  --> Payment Received
      --> Processing
          --> Sent to Supplier's Bank
              --> Delivered
              --> Failed
```

**Wallet-funded transfer:**
```
Processing
  --> Sent to Recipient's Bank
      --> Delivered
      --> Failed
```

### Design rules

States are append-only. Never delete or overwrite a state record. Every state transition is timestamped and logged with the triggering event (webhook receipt, manual review sign-off, timeout). Every state change triggers a notification to the user via push, WhatsApp (if opted in), and SMS as a fallback.

The state machine must accept out-of-order webhook delivery. A Delivered webhook arriving before a Sent webhook should still produce the correct final state. This happens more frequently with Airtel Money and MTN MoMo APIs than with M-Pesa or Paystack.

Failed is a terminal state. To retry a failed transfer, create a new transaction with a new transaction_id. Do not reuse transaction IDs across retry attempts. Reusing IDs creates reconciliation ambiguity when the original failed transaction and the retry both appear in bank statements.

### Notification at every state change

Every state transition should trigger a notification. The user should never have to open the app to know what happened to their money. Notification channels in priority order: push (Firebase FCM), WhatsApp Business API (pre-approved templates, opt-in required), SMS (transactional only, under 160 characters per segment).

---

## 8. Mobile Money Integration Patterns

### Key providers and APIs

| Provider | Markets | API | Approval window |
|---|---|---|---|
| M-Pesa (Safaricom) | Kenya, Tanzania | Daraja API (STK Push, B2C, C2B) | 5 minutes |
| MTN MoMo | Ghana, Uganda, Rwanda, Cote d'Ivoire, Cameroon | MTN MoMo API | Instant on approval |
| Airtel Money | Nigeria, Kenya, Uganda, Tanzania | Airtel Money API | Instant on approval |
| Orange Money | Cote d'Ivoire, Senegal, Mali | Orange Money API | Instant on approval |

### STK Push pattern (M-Pesa)

STK Push is the standard for merchant-initiated mobile money collection in Kenya. The merchant backend calls the Safaricom API with the customer's phone number and the amount. Safaricom pushes a prompt to the customer's phone. The customer enters their PIN to approve. Safaricom sends a callback to the merchant's webhook.

```
Step 1: POST to Safaricom /mpesa/stkpush/v1/processrequest
  Body: { BusinessShortCode, Password, Timestamp, TransactionType,
          Amount, PartyA (phone), PartyB (shortcode),
          PhoneNumber, CallBackURL, AccountReference, TransactionDesc }

Step 2: Customer receives prompt and enters PIN

Step 3: Safaricom sends callback to CallBackURL
  Body: { MerchantRequestID, CheckoutRequestID, ResultCode, ResultDesc,
          Amount, MpesaReceiptNumber, Balance, TransactionDate, PhoneNumber }

Step 4: Verify Amount, PhoneNumber, and CheckoutRequestID before crediting
```

The 5-minute approval window is strict. If the customer does not approve within 5 minutes, the transaction times out. Design the UI to show a countdown and offer a retry. Do not leave the customer on an indefinite loading screen.

### MTN MoMo API pattern

MTN MoMo uses a two-step request-to-pay flow for collection:

```
Step 1: POST /collection/v1_0/requesttopay
  Headers: X-Reference-Id (UUID), X-Target-Environment, Ocp-Apim-Subscription-Key
  Body: { amount, currency, externalId, payer: { partyIdType, partyId }, payerMessage, payeeNote }
  Response: 202 Accepted (async)

Step 2: GET /collection/v1_0/requesttopay/{referenceId}
  Response: { status: PENDING | SUCCESSFUL | FAILED, ... }
```

MTN MoMo uses polling rather than webhooks as the primary status mechanism in several markets. Build both a polling fallback (every 10 seconds, up to 5 minutes) and a webhook handler. The X-Reference-Id you generate in Step 1 is your idempotency key for this transaction.

### PSP abstraction for mobile money

Wrap all mobile money providers behind a shared abstraction layer. Each provider has different API auth mechanisms (OAuth2, API keys, HMAC), different payload structures, and different callback formats. The abstraction layer normalises all of this. Adding a new mobile money provider should require changes only inside the abstraction layer, not in the payment flow:

```
POST /payments/mobile-money
{
  provider: "mpesa" | "mtn_momo" | "airtel" | "orange_money",
  phone_number: string,
  amount: number,
  currency: string,
  reference: string
}
Response: {
  payment_id,
  status: "pending" | "completed" | "failed",
  provider_reference
}
```

### Handling provider API instability

Mobile money provider APIs in African markets are less stable than Stripe or Paystack. MTN MoMo in Ghana has had extended outages during major public events (elections, holidays). Airtel Money in Uganda has maintenance windows that are not always pre-announced. Build circuit breakers: if an API returns errors on 3 consecutive requests within a 2-minute window, mark that provider as degraded and surface a warning to the user at payment method selection. Do not surface the raw provider error: just indicate that this method is temporarily unavailable and offer alternatives.

---

## 9. Compliance API Integration (KYC/KYB)

### Push, do not pull

Do not build a manual review queue inside your platform. Push documents to your verification provider (YouVerify, Smile Identity, Onfido) automatically on submission. Handle the result via webhook callback. Building an internal queue adds operational overhead, increases turnaround time for users, and creates a compliance record-keeping burden.

```
On form submission:
  1. Store documents in encrypted object storage
  2. Generate presigned upload URLs (1-hour expiry)
  3. Push document references to verification provider API
  4. Return "under review" status to user

On webhook callback:
  1. Verify signature
  2. Update KYC record: status, timestamp, provider_reference
  3. Trigger account activation if approved
  4. Notify user via push and email
```

### Retry logic for API failures

Verification provider APIs go down. Build a retry queue: on submission failure, queue for retry; maximum 3 attempts at 5-minute intervals; after 3 failures, alert the compliance team and send a human-readable status to the user. Do not leave users in limbo with no explanation.

### PEP and sanctions screening

Screen every user and every business director and UBO against: OFAC SDN list, UN sanctions list, local regulatory watchlists (CBN, CBK as applicable), and PEP databases. Any match should flag the account for manual compliance review, not auto-reject. False positives are common, especially for common names in West and East African markets (Ojo, Osei, Kamau, Mwangi appear thousands of times in PEP databases due to partial name matches). Build a manual review workflow, not a hard block.

### KYB document retention

Store all KYB documents for a minimum of 7 years per AML/CFT regulatory requirements across Nigeria, Kenya, and Ghana. Documents must not be deletable by the user or by standard support staff. Access requires manager approval and every access must be logged with user_id, timestamp, action type, and the support ticket ID justifying the access.

### YouVerify integration notes

YouVerify is the most widely used compliance API for Nigerian fintechs and cross-border products targeting West Africa. Key integration points: BVN verification (returns registered name, DOB, phone number), NIN verification, CAC company verification (returns directors and registered address), liveness check (webcam capture, returns match confidence score), and document OCR (extracts data from passports, national IDs, driver's licences). YouVerify's webhook reliability is good but build retry logic anyway. The compliance team should receive an alert if all retries on a document submission fail.

---

## 10. PSP Abstraction Layer

### The problem with direct PSP integration

At Equity Bank, we built a 13-partner PSP and fintech ecosystem. Each direct integration was a separate codebase: different auth, different payload shapes, different error codes, different webhook formats. Every time a PSP changed their API (which happens frequently, especially for African providers), it required a code change and a deployment. By the time we had 5+ active integrations, the maintenance burden was significant.

The abstraction layer pattern solves this by creating a single internal interface that all payment methods route through.

### The pattern

```
// Internal payment initiation interface
POST /charge
{
  method: "card" | "bank_transfer" | "mobile_money" | "wallet",
  provider: "paystack" | "flutterwave" | "mpesa" | "mtn_momo" | "airtel",
  amount: integer,        // Always integer, smallest denomination
  currency: string,       // ISO 4217
  customer: {
    name: string,
    phone: string,
    email: string
  },
  metadata: {
    transaction_id: string,
    reference: string
  }
}

Response:
{
  charge_id: string,
  status: "pending" | "completed" | "failed",
  provider_reference: string,
  provider_raw_response: object   // Stored for debugging but not exposed to frontend
}
```

The abstraction layer handles: provider selection logic, provider-specific payload transformation, normalised status codes across providers, webhook normalisation, and retry logic. Adding a new provider requires changes only inside the abstraction layer.

### Provider routing logic

Store provider routing rules in the database. Examples:

| Payment method | Currency | Provider |
|---|---|---|
| Card | NGN | Paystack |
| Card | GHS, KES, UGX | Flutterwave |
| Mobile money | KES | Safaricom (M-Pesa) |
| Mobile money | GHS, UGX | MTN MoMo |
| Mobile money | NG | OPay or Airtel |
| Bank transfer | NGN | Providus or Wema (virtual account) |

Routing rules are database config, not code. Changing a provider (for example, switching card processing from Flutterwave to Paystack in Ghana) is a database change with no deployment required. This is especially important in African markets where provider pricing and reliability change frequently.

### Error normalisation

Each PSP has its own error codes. Flutterwave and Paystack have different error code schemas; MTN MoMo and M-Pesa have entirely different structures. The abstraction layer maps all provider errors to a normalised internal error taxonomy:

| Internal error | User-facing message |
|---|---|
| INSUFFICIENT_FUNDS | Your account has insufficient funds for this payment |
| CARD_DECLINED | Your card was declined. Please try another payment method |
| MOBILE_MONEY_TIMEOUT | Payment approval timed out. Please try again |
| PROVIDER_UNAVAILABLE | This payment method is temporarily unavailable. Please try another |
| DUPLICATE_TRANSACTION | This payment has already been processed |

Never expose raw provider error messages to users. They are often technical, inconsistent, and in some cases reveal information about your infrastructure.

---

## 11. API Design for B2B Supplier Payments

### What makes B2B different from consumer remittance

B2B supplier payments have requirements that consumer remittance products do not. Every transfer requires a commercial invoice. Suppliers need to be saved and reused across multiple payments. Purpose of payment is a regulatory requirement in every market. Transfer limits are higher (up to $50,000 per transaction). Compliance documentation needs to be retained and exportable.

These differences are not cosmetic: they change the data model, the API contract, and the compliance architecture.

### Key API endpoints for B2B

```
// Supplier management
GET    /suppliers                     // List saved supplier profiles
POST   /suppliers                     // Create new supplier
PATCH  /suppliers/{id}                // Update supplier. All changes logged in audit_trail
DELETE /suppliers/{id}                // Soft delete. Historical records preserved

// Transfer initiation
POST   /transactions
Body: {
  amount: integer,
  source_currency: string,
  destination_currency: string,
  supplier_id: string,
  payment_method: "card" | "bank_transfer" | "wallet",
  purpose_of_payment: "goods_import" | "services" | "software" | "freight" | "intracompany" | "other",
  purpose_detail: string,             // Required if purpose = "other"
  document_ids: string[],             // Array of uploaded invoice document IDs. Min 1 required
  declaration_confirmed: boolean      // AML compliance declaration. Must be true
}

// Document handling
POST   /documents                     // Upload invoice or compliance document
GET    /documents/{id}                // Returns presigned URL (1-hour expiry)

// Reporting
GET    /transactions/export           // CSV export compatible with Sage, QuickBooks, Xero
GET    /suppliers/{id}/statement      // Annual PDF statement for one supplier
GET    /statements/annual             // Annual summary across all suppliers
```

### Supplier audit trail

Every change to a supplier's account_details (bank account number, SWIFT code, beneficiary name) must be logged in an append-only audit trail with the old value, the new value, the user_id who made the change, and the timestamp. This is a regulatory requirement in most African markets with AML frameworks and a practical fraud prevention measure: changing a supplier's bank account details is the most common vector for business email compromise fraud in B2B payments.

### Purpose of payment enforcement

Purpose of payment is a regulatory requirement for all international B2B transfers in Nigeria (CBN FX regulations), Kenya (CBK FX guidelines), and Ghana (BoG FX requirements). It is not optional even for small amounts. The API must enforce it at the transaction initiation level: a POST /transactions request without a valid purpose_of_payment should return a 422 with a clear error. Storing it on the transaction record is required for AML audit trails.

---

## 12. Transaction Limits and Velocity Controls

### Limit enforcement design

Transaction limits serve two purposes: regulatory compliance (KYC tier-based limits required by CBN, CBK, BoG) and fraud prevention (velocity controls to detect unusual patterns). These are separate concerns and should be separate systems.

KYC tier limits are hard limits set by the compliance team and stored in the database per user. They reset monthly on a calendar-month basis. Velocity controls are soft limits that trigger a review flag, not an automatic block.

```
// Limit check at transaction submission
GET /limits/{user_id}
Response: {
  monthly_limit: integer,           // KYC tier limit in USD equivalent
  monthly_used: integer,            // Running total this calendar month
  monthly_remaining: integer,       // Derived: limit - used
  daily_used: integer,              // For velocity monitoring
  transaction_count_today: integer, // For velocity monitoring
  kyc_tier: "starter" | "verified" | "business" | "custom",
  days_until_reset: integer,
  upgrade_available: boolean
}
```

### AML velocity rules

Flag transactions for manual review (do not auto-block) when: any single transfer exceeds $20,000 to a supplier the business has not paid before; more than 3 transfers are initiated in a single day by the same user; the total daily volume for a user exceeds their declared expected monthly volume divided by 20; or the destination country appears on a high-risk jurisdiction list.

Flagged transactions should be held pending compliance review. The user should be notified of a delay without the reason disclosed. Disclosing the specific AML rule that triggered a hold is a tipping-off risk under most African AML regulations.

---

## Background

These patterns are drawn from direct experience:

- Designing and launching 10 payment APIs across 6 East African markets as Senior PM at **Equity Bank Group**, including Visa Cybersource and Mastercard Payment Gateway integrations, building a 13-partner PSP ecosystem, and generating $262.5K in MRR within 90 days of launch
- Architecting cross-border payment rails and KYC/KYB compliance infrastructure as CPO/Co-Founder at **Betaling Africa**, taking an FX platform from zero to $26.6M in transaction volume with Individual, Business, and Wallet product lines
- Payment product and partnership work across 25 African markets at **Cellulant** (Chief of Staff to CEO, Product Manager) and **Norebase** (Senior Strategic Partnerships and Product Manager)

---

## About

Written by [Oluseye "Oba" Obadeyi](https://linkedin.com/in/seyeobadeyi) -- Payments PM with 6+ years across African and European fintech.

- 🌐 [seyeobadeyi.com](https://seyeobadeyi.com)
- 📰 [Dash of Payment - EMEA Focus](https://linkedin.com/in/seyeobadeyi) -- LinkedIn newsletter, 1,500+ subscribers
- 🐙 [github.com/saobadeyi-lab](https://github.com/saobadeyi-lab)
