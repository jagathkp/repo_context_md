# yubi-fin-b2c-payment-service — Master Context

> **Repo:** yubi-fin-b2c-payment-service | **Branch:** master | **Stack:** Java 17, Spring Boot 3.4.3, PostgreSQL, Razorpay | **Owner:** B2C Fin Team

## TL;DR (30-second read)

- **What it does:** Handles payment initiation, processing, and settlement for bond and equity orders in the Yubi B2C retail investment platform.
- **Primary input:** Payment initiation requests from the Checkout Service (user selects payment method and amount).
- **Primary output:** Payment status updates sent back to Checkout Service and other downstream systems; webhook events from Razorpay processed and stored.
- **Current status:** Production (CSPL migration complete).
- **Owner:** B2C Fin Squad | Slack: #b2c-fin-payments

---

## Who Should Read This

| Audience | Most Relevant Sections |
|---|---|
| **Product Manager** | TL;DR, §3 (Users), §4 (Journeys), §5 (Conditional Flows), §13 (Integration Map) |
| **Backend Developer** | §6 (Architecture), §8 (Local Setup), §9 (API Reference), §10 (How to Extend), §11 (Data Model) |
| **DevOps / Infra** | §14 (Testing, Deployment, Infrastructure), §7 (Security & Secrets) |
| **QA / Tester** | §4 (Journeys), §13 (Common Failure Modes), §9 (API Reference) |
| **New Team Member** | Read all sections in order. Start with §2, §3, §4. |

---

## Table of Contents

1. Glossary
2. What Is This Service?
3. Who Uses It and How
4. The User Journey — Plain English
5. Conditional Flows — Which Workflow Applies to Whom
6. Architecture & Code Structure
7. Security & Access Control
8. Local Setup & Running the Service
9. API Reference
10. How to Extend the System
11. Data Model & Status Rules Engine
12. Integration Map
13. Operational Runbook
14. Testing, Deployment & Infrastructure

---

## 1. Glossary

| Term | Definition |
|---|---|
| **B2C** | Business-to-Consumer retail investment platform (Yubi's retail division) |
| **Payment Gateway** | Razorpay (third-party payment processor that handles card, UPI, and bank transfers) |
| **VBA** | Virtual Bank Account — unique account number assigned by Razorpay to collect payments for a specific order |
| **UPI** | Unified Payments Interface — real-time bank-to-bank transfer mechanism in India, supports up to ₹100,000 per transaction |
| **Netbanking** | Direct bank transfer via online banking; supports higher limits |
| **RFQ** | Request for Quote — used in bond/fixed-income trading; order number for bonds |
| **Order** | User's request to purchase a security (bond or equity) |
| **Trade Date** | Date on which the trade is executed; impacts settlement and payment validity windows |
| **Settlement** | Process of transferring securities to user's demat account and funds to broker's account |
| **Demat** | Dematerialized securities account at NSDL (depository); required to hold securities |
| **Member ID** | Razorpay business identifier; set to "6810" |
| **CC Name** | Credit Card name / merchant identifier in Razorpay; set to "ICL" |
| **Entity / Channel** | Request headers that identify which product and platform made the payment request (e.g., entity=YUBIFIN, channel=retail) |
| **Idempotency** | Ensuring duplicate requests do not create duplicate payments; enforced via reference ID uniqueness |
| **Webhook** | HTTP callback from Razorpay when payment status changes (e.g., captured, failed, refunded) |
| **Event ID** | Unique identifier for a webhook event from Razorpay; prevents duplicate processing |
| **Signature Verification** | HMAC-SHA256 validation of webhook payloads using webhook secret; prevents forgery |
| **DLQ** | Dead Letter Queue — Kafka topic where failed messages are sent for manual review |
| **DDL/DML** | Data Definition/Manipulation Language SQL statements; Liquibase manages all DB changes |

---

## 2. What Is This Service?

**One-line purpose:**  
Manage the complete payment lifecycle for retail bond and equity purchases: initiate orders, collect payments from users via multiple channels, verify payment status, and communicate outcomes to downstream systems.

**Why it exists:**  
Retail investors need a reliable, secure way to fund their purchases on the Yubi platform. This service abstracts away payment processing complexity, fraud prevention, and regulatory compliance (PCI-DSS via Razorpay) from individual product teams.

**What it does NOT do:**
- Does **not** execute trades or settle securities (Trade Service and Settlement Service handle this).
- Does **not** validate user KYC or demat account status (KYC Service does this).
- Does **not** manage refunds initiated by end users (Refund Service handles this if it exists; currently unclear).
- Does **not** handle recurring/subscription payments.
- Does **not** manage forex or international payments.

**Value to the business:**
- Enables all retail product orders (bonds, equities) to monetize transactions.
- Reduces fraud and chargeback risk via Razorpay's underwriting and monitoring.
- Centralizes payment logic: changes to fee structures, payment method availability, or regulatory rules are made here once, not repeated across products.

---

## 3. Who Uses It and How

| Stakeholder | How They Interact | What They Need |
|---|---|---|
| **End User (Investor)** | Uses Yubi web/app to buy bonds or equities; selects payment method (UPI, Netbanking) at checkout | Fast, secure payment; multiple options; clear failure messages |
| **Checkout Service** | Calls Payment Service to initiate and finalize payment; waits for status updates | Reliable payment creation; clear status transitions; webhook callbacks on completion |
| **Trade Service** | Listens for payment completion events to know when to settle the order | Payment confirmation events via Kafka or synchronous callback |
| **Offer Service** | May need to fetch payment-related offer terms (e.g., no-cost payment options) | Payment method availability and limits |
| **Notification Service** | Receives payment completion events to send SMS/email receipts and deal slips | Event payloads with payment ID, amount, status |
| **Razorpay** | External payment gateway; sends webhook callbacks when payments are captured or fail | Webhook signature headers; event payload structure |
| **Finance/Ops Team** | Views payment settlement reports, resolves failed payments, investigates chargebacks | Payment logs, settlement reconciliation, manual intervention endpoints |
| **Compliance / Legal** | Needs audit trail of all payments for regulatory reporting | Audit events, payment status history, KYC linkage |

### Supported Product Types

| Product | Supported? | Notes |
|---|---|---|
| **Bonds (Fixed Income)** | ✅ Yes | Primary use case; uses RFQ order flow |
| **Equities** | ✅ Yes | Supported; uses standard order flow |
| **Mutual Funds** | ❓ Unclear | Not explicitly mentioned; may be planned |
| **Commodities** | ❌ No | Out of scope |

---

## 4. The User Journey — Plain English

### Journey 1: User Buys a Bond via Yubi (Happy Path)

**Actors:** Investor, Yubi (checkout), Payment Service, Razorpay, Trade Service

1. **User selects bond** on Yubi platform and clicks "Buy Now".
2. **Checkout Service shows payment options:**  
   - Display available banks and payment methods for the order amount.  
   - For amounts ≤ ₹100,000: show UPI + Netbanking.  
   - For amounts > ₹100,000: show Netbanking only (UPI has lower limit).
3. **User selects payment method** (e.g., "UPI via HDFC").
4. **Checkout calls Payment Service:** `POST /initiate`  
   - Sends: order details, amount, user ID, trade date, product type (bond/equity).  
   - Receives: payment initiation response with payment ID and Razorpay order ID.
5. **Checkout redirects user to Razorpay checkout** (mobile/web payment interface) with the order ID.
6. **User authenticates and completes payment** (via UPI app, netbanking portal, or card).
7. **Razorpay captures the payment** and sends a webhook to Payment Service: `POST /event`  
   - Headers: `x-razorpay-event-id` (idempotency), `x-razorpay-signature` (HMAC verification).  
   - Payload: payment ID, amount, status=captured.
8. **Payment Service** validates webhook signature and updates payment status to CAPTURED in DB.
9. **Payment Service publishes event** to Kafka topic `bonds-event-dev` (or prod equivalent):  
   - Message: `{paymentId, status: CAPTURED, orderId, userId, ...}`.
10. **Trade Service consumes event** and proceeds to settle the order (transfer securities, collect settlement funds).
11. **Notification Service consumes event** and sends SMS + Email receipt with deal slip to user.
12. **User receives confirmation** email within seconds; order is now live.

**Status at each step:**
- After step 4: `CREATED` (no payment captured yet)
- After step 8: `CAPTURED` (payment fully received)
- User sees: "Payment successful, your bond is now yours!"

---

### Journey 2: User's Payment Fails or Exceeds Limit

**Trigger:** User attempts a payment that cannot be processed.

1. **Checkout Service checks payment method availability:** `GET /banks/{bank-code}/payment-methods?amount=X`
   - If amount > ₹100,000 and bank supports only UPI → error: "Amount too high for UPI, use Netbanking".
   - If bank has no supported methods → error: "Bank not supported for this amount".
2. **User sees error on Yubi UI** and can retry with a different method or amount.
3. Alternatively, **user initiates payment but Razorpay declines it** (e.g., insufficient balance, fraud check):
   - Razorpay sends webhook with status=`failed`.
   - Payment Service updates DB: status=`FAILED`, error JSON contains failure reason from Razorpay.
   - Notification Service sends SMS: "Payment failed. Retry or contact support."
4. **User retries** (new payment ID generated, old one remains in DB as FAILED).

**Status progression:**
- Initial: `CREATED`
- If declined: `FAILED`
- User retries: new payment record with `CREATED` status

---

### Journey 3: Payment Refund (Post-Order)

**Trigger:** User cancels order or ops team initiates refund.

1. **Refund Service (or ops manual process) calls Payment Service:** `POST /{payment-id}/refund` (if endpoint exists; unclear in provided code).
   - Alternatively, ops team manually initiates refund in Razorpay dashboard.
2. **Payment Service creates refund request** in Razorpay via API.
3. **Razorpay processes refund** (typically 3-5 business days to user's account).
4. **Razorpay sends webhook** with status=`refunded`.
5. **Payment Service updates DB:** status=`REFUNDED`.
6. **Notification Service notifies user:** "Your ₹X refund has been initiated and will reach your account in 3-5 business days."

**Status progression:**
- Before refund: `CAPTURED`
- After refund initiated: `REFUND_IN_PROGRESS`
- After refund completed: `REFUNDED`

---

### Journey 4: Virtual Account Collection (NEFT/IMPS Transfers)

**Trigger:** User is offered a Virtual Bank Account (VBA) option for bulk or institutional purchases (less common in retail, but supported).

1. **Checkout creates a Virtual Account** via Payment Service endpoint (to be confirmed).
2. **User receives bank account number** and reference ID.
3. **User transfers funds** via NEFT/IMPS to that account.
4. **Razorpay webhook notifies** of funds received: `POST /event/vba-credit`.
5. **Payment Service marks payment as CAPTURED** and triggers trade settlement.

---

## 5. Conditional Flows — Which Workflow Applies to Whom

### Dimension 1: Order Amount and Payment Method

```
                     Amount ≤ ₹100,000        Amount > ₹100,000
                    (UPI Allowed)             (UPI Blocked)
┌─────────────────┬────────────────┬──────────────────────┐
│ Bank Supports:  │   Methods      │   Methods            │
├─────────────────┼────────────────┼──────────────────────┤
│ UPI + Netbank   │ ✅ UPI + NB    │ ✅ NB only (UPI off) │
│ UPI only        │ ✅ UPI         │ ❌ ERROR: NB needed  │
│ Netbank only    │ ✅ NB          │ ✅ NB                │
└─────────────────┴────────────────┴──────────────────────┘
```

**Rule enforced in code:**  
`application-*.yml` config: `yubi.fin.b2c.bank.max-amount-allowed-for-upi: 100000` (or 200000 in prod/perf).

---

### Dimension 2: Product Type and Feature Availability

| Feature | Bonds | Equities | Notes |
|---|---|---|---|
| UPI payment | ✅ | ✅ | Subject to amount limits |
| Netbanking | ✅ | ✅ | No amount limit |
| Virtual Account | ✅ | ✅ | For institutional buyers |
| Payment links | ✅ | ✅ | Can disable UPI for links: `disable-upi-for-payment-link: true` |
| Settlement callback | ✅ | ✅ | Via Kafka event or HTTP callback |

---

### Dimension 3: Entity / Channel (Platform)

| Entity | Channel | Notes |
|---|---|---|
| YUBIFIN | retail | Primary retail investment platform |
| DISTRIBUTOR | retail | Distributor/channel partner platform |

Each has its own callback URL and notification template (configured in `CallbackConfig`).

---

### Payment Status State Machine

```
┌─────────┐
│ CREATED │  (Payment initiated, awaiting capture)
└────┬────┘
     │
     ├──→ [Razorpay webhook: captured] → ┌──────────┐
     │                                     │CAPTURED  │ (Payment complete, trade can settle)
     │                                     └──────────┘
     │                                            │
     │                                            └──→ [Refund initiated] → ┌──────────────────┐
     │                                                                       │REFUND_IN_PROGRESS│
     │                                                                       └──────┬───────────┘
     │                                                                              │
     │                                                                              └──→ ┌─────────┐
     │                                                                                   │REFUNDED │
     │                                                                                   └─────────┘
     │
     └──→ [Razorpay webhook: failed] → ┌───────┐
                                        │ FAILED│  (User can retry with new payment)
                                        └───────┘
```

---

### Pre-requisite Gates (Before Payment Can Start)

A payment **cannot** be initiated unless:

1. ✅ **User exists and is authenticated** (checked by Checkout Service before calling Payment Service).
2. ✅ **User has a valid demat account** (checked by KYC Service; Payment Service assumes it's true).
3. ✅ **Trade date is valid** (not a market holiday, not in the past). Validated by Trade Service and passed to Payment Service.
4. ✅ **Amount > 0 and within order limits** (checked by Checkout Service).
5. ✅ **Bank supports the payment method for this amount** (checked in `BankService.getPaymentModes()`; throws `NO_PAYMENT_METHODS_SUPPORTED_FOR_THE_AMOUNT` error if violated).

---

### What Can Be Changed WITHOUT Code Deploy

| Item | Config File | Key Property | Example |
|---|---|---|---|
| **Max UPI amount** | `application-*.yml` | `yubi.fin.b2c.bank.max-amount-allowed-for-upi` | Change from 100000 to 200000 |
| **Razorpay API keys** | `application-*.yml` | `yubi.fin.b2c.payment.payment-gateway.api-key` | Rotate key on Razorpay dashboard, update env var |
| **Webhook secret** | `application-*.yml` | `yubi.fin.b2c.payment.payment-gateway.webhook-secret` | Regenerate on Razorpay dashboard |
| **Callback URLs** | `application-*.yml` | `yubi.fin.b2c.payment.event.callback.product-urls` | Point to new Checkout Service endpoint |
| **Email templates** | `application-*.yml` | `yubi.fin.b2c.email-notification.*-template` | Change template name in notification service |
| **Email recipients** | `application-*.yml` | `yubi.fin.b2c.email-notification.live-trade-to-email` | Ops team mailbox |
| **Offer service config** | `application-*.yml` | `yubi.fin.b2c.payment.offer.hour`, `minute`, `holiday` | Change offer fetch schedule |
| **Disable UPI for payment links** | `application-*.yml` | `yubi.fin.b2c.payment.disable-upi-for-payment-link` | Set to `false` to enable UPI links |
| **Trade date URL** | `application-*.yml` | `yubi.fin.b2c.payment.trade-date-url` | Point to new validation service |
| **Kafka brokers** | `application-*.yml` | `yubi.fin.b2c.event.kafka.kafka-servers` | Add/remove broker IPs for cluster scaling |
| **Kafka topics** | `application-*.yml` | `yubi.fin.b2c.event.kafka.event-topic` | Rename topic (must update all consumers too) |
| **ICCL notes enabled** | `application-*.yml` | `yubi.fin.b2c.payment.iccl-notes-enabled` | Enable/disable ICCL-specific notes in payments |

---

### What Requires a Code Change

| Scenario | Why | Example |
|---|---|---|
| **Add a new payment method** | Must add support in `BankService`, update DB schema, add bank payment mode mappings | Add CARD payment method |
| **Change payment status enum** | Affects state machine; all dependent services must handle new status | Add AUTHORIZING state between CREATED and AUTHORIZED |
| **Modify idempotency key logic** | Idempotency is built into service; changing how duplicates are detected requires code | Use `order_id + user_id` instead of `reference_id` |
| **Add new bank** | Must seed new bank into `bank_payment_modes` table; can be done via data migration | Add BANK_OF_INDIA bank |
| **Add new product type** | Must add handling for new product in payment initiation logic | Support Mutual Funds |
| **Change webhook signature verification algorithm** | Security-critical; affects how Razorpay payloads are validated | Switch from HMAC-SHA256 to HMAC-SHA512 |
| **Add new downstream service integration** | Must add REST client, error handling, retry logic | Call new Settlement Service for real-time settlement |

---

### Notification Trigger Table

| Event | Recipient | Channel | Condition | Template |
|---|---|---|---|---|
| **Payment captured** | User | Email + SMS | `status == CAPTURED` | `bond-b2c-order-receipt-template` |
| **Payment failed** | User | Email + SMS | `status == FAILED` | (inferred; TBD) |
| **Payment refunded** | User | Email + SMS | `status == REFUNDED` | (inferred; TBD) |
| **VBA credit received** | User | Email + SMS | `vba-credit webhook received` | (inferred; TBD) |
| **Live trade execution** | Ops team | Email | `manual trigger by ops` | `live-trade-ops-team-template` |
| **Deal slip generated** | User | Email + SMS | After payment capture | `bond-b2c-deal-slip-template` + `bond-b2c-deal-sms-template` |

> **Note:** Payment Service publishes events to Kafka; Notification Service consumes them. Exact templates inferred from config — verify with Notification Service team.

---

## 6. Architecture & Code Structure

### High-Level Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                      Yubi B2C Platform                          │
│  ┌──────────────┐     ┌─────────────────┐      ┌──────────────┐ │
│  │ Checkout Svc │────→ │ Payment Service │      │ Trade Svc    │ │
│  └──────────────┘     └────────┬────────┘      └──────────────┘ │
│                                │                                 │
│                                ├→ Kafka (bonds-event-dev)        │
│                                │   ├→ Trade Service              │
│                                │   ├→ Notification Service       │
│                                │   └→ Other subscribers          │
│                                │                                 │
└────────────────────────────────┼─────────────────────────────────┘
                                 │
                    ┌────────────┴────────────┐
                    │                         │
              ┌─────▼──────┐         ┌────────▼──────┐
              │ PostgreSQL │         │ Razorpay API  │
              │ (payments  │         │ (webhooks)    │
              │  table)    │         └───────────────┘
              └────────────┘
```

---

### Request Lifecycle: Payment Initiation Flow

```
1. Checkout Service
   │
   ├─→ POST /initiate
   │   (entity, channel, amount, product, trade-date)
   │
2. PaymentController
   │
   ├─→ PaymentService.initiatePayment()
   │
3. PaymentService
   │
   ├─→ Validate trade date (call Trade Date URL)
   ├─→ Validate user KYC (call KYC Service)
   ├─→ Create Razorpay customer (REST call to Razorpay API)
   ├─→ Create Razorpay order (REST call: POST /orders)
   │
4. PaymentGatewayLinkRepository
   │
   ├─→ Save Payment record in DB (status=CREATED)
   ├─→ Save PaymentGatewayLink record (maps local ID to Razorpay ID)
   │
5. Return PaymentInitResponse
   │   (paymentId, razorpayOrderId, amount, ...)
   │
6. Checkout Service
   │
   └─→ Redirect user to Razorpay Checkout (web/mobile)
       User completes payment on Razorpay → Razorpay captures payment

```

---

### Async Flow: Webhook Processing

```
Razorpay (external)
│
├─→ POST /event
   (x-razorpay-event-id header, x-razorpay-signature header, JSON body)
│
PaymentEventController
│
├─→ Validate signature (HMAC-SHA256 using webhook-secret)
├─→ Parse event payload
│
PaymentEventService.handle(PaymentEvent)
│
├─→ Look up PaymentGatewayLink by razorpay_payment_id
├─→ Fetch Payment record from DB
├─→ Update Payment.status based on webhook (e.g., CAPTURED, FAILED)
├─→ Update Payment.lastUpdatedOn
│
EventPusher (Kafka)
│
├─→ Publish PaymentEvent to Kafka topic (bonds-event-dev)
│   Message format: {paymentId, status, razorpayPaymentId, userId, ...}
│
Subscribers (Trade Service, Notification Service, etc.)
│
├─→ Consume event and take action
   Trade Service → Settle order
   Notification Service → Send SMS/email receipt
```

---

### Request Lifecycle: Get Payment Methods for a Bank

```
1. Checkout / Client
   │
   ├─→ GET /banks/{bank-code}/payment-methods?amount=X&currency=INR
   │
2. BankController
   │
   ├─→ BankService.getPaymentModes(bankCode, BigMoney)
   │
3. BankService
   │
   ├─→ BankRepository.findByBankCode(bankCode)
   │
4. BankRepository
   │
   ├─→ BankPaymentModeRepository.findById(bankCode)
   │
5. Database (bank_payment_modes table)
   │
   ├─→ Fetch: bank_code, supports_net_banking, supports_upi
   │
6. BankService
   │
   ├─→ Build List<PaymentMethod>
   ├─→ Check amount against UPI max limit
   │   (if amount > max AND supports_upi: remove UPI from list)
   │
7. BankController
   │
   ├─→ Return List<PaymentMethod> as JSON
   │
Response: [UPI, NETBANKING] or [NETBANKING] or [UPI] depending on config & amount
```

---

### Package Structure & Directory Navigation

```
src/main/java/com/yubi/fin/b2c/payment/
│
├── controller/
│   ├── BankController.java               ← GET /banks/{code}/payment-methods
│   ├── PaymentController.java            ← POST /initiate, /payments, status updates
│   └── PaymentEventController.java       ← POST /event (webhook handler)
│
├── service/
│   ├── BankService.java                  ← Interface for bank logic
│   ├── DefaultBankService.java           ← Implementation (filter methods by amount)
│   ├── PaymentService.java               ← Payment initiation, status updates
│   └── PaymentEventService.java          ← Webhook event processing
│
├── repository/
│   ├── BankRepository.java               ← Query bank_payment_modes
│   ├── BankPaymentModeRepository.java   ← JPA repo for bank_payment_modes entity
│   ├── PaymentGatewayLinkRepository.java ← JPA repo for payment_gateway_link entity
│   └── ... (other repo interfaces)
│
├── model/
│   ├── entity/
│   │   ├── Payment.java                  ← JPA entity, maps to payments table
│   │   ├── BankPaymentMode.java          ← JPA entity, maps to bank_payment_modes table
│   │   └── PaymentGatewayLink.java       ← Links local payment_id to razorpay_payment_id
│   ├── dto/
│   │   ├── PaymentMethod.java            ← Enum: UPI, NETBANKING
│   │   ├── PaymentUpdateStatus.java      ← Enum: SUCCESS, FAILED, REFUNDED
│   │   └── ... (other DTOs)
│   ├── request/
│   │   ├── PaymentInitRequest.java       ← Incoming /initiate request
│   │   ├── PaymentUpdateRequest.java     ← Incoming status update request
│   │   └── UpdatePaymentNotesRequest.java ← Update notes on payment
│   ├── response/
│   │   ├── PaymentInitResponse.java      ← Response to /initiate
│   │   ├── PaymentResponse.java          ← Response for /payments GET
│   │   └── ... (other response types)
│   ├── PaymentEvent.java                 ← Webhook event structure
│   ├── PaymentEventData.java             ← Webhook payload content
│   ├── PaymentStatus.java                ← Enum: CREATED, CAPTURED, FAILED, REFUNDED, ...
│   └── PaymentEventStatus.java           ← Enum: RECEIVED, PROCESSED, FAILED
│
├── config/
│   ├── PaymentServiceConfig.java         ← Bean definitions (RestTemplate, signature verifiers)
│   ├── PaymentGatewayConfig.java         ← @ConfigurationProperties for Razorpay keys
│   ├── BankConfigProperties.java         ← @ConfigurationProperties for UPI max amount
│   ├── DiscoveryServiceConfig.java       ← Config for discovery service
│   ├── OfferServiceConfig.java           ← Config for offer service (hour, minute, holiday)
│   ├── CallbackConfig.java               ← Product callback URLs for payment completion
│   ├── PaymentMapperConfig.java          ← MapStruct bean
│   ├── PaymentsErrorCode.java            ← Enum of all error codes
│   ├── PaymentEventConfigProperties.java ← Config for event headers & verification
│   └── email/
│       └── EmailNotificationEvent.java   ← Config for email templates & senders
│
├── mapper/
│   └── PaymentMapper.java                ← MapStruct interface: Payment → PaymentResponse
│
├── util/
│   ├── PaymentSignatureVerifier.java     ← HMAC-SHA256 verification for webhooks
│   └── Constants.java                    ← Strings like CURRENCY_INR
│
├── PaymentApplication.java               ← Spring Boot main class
│
└── resources/
    ├── application.yml                   ← Base config
    ├── application-dev.yml               ← Dev overrides (Kafka URLs, API endpoints, keys)
    ├── application-local.yml             ← Local dev overrides (localhost URLs)
    ├── application-prod.yml              ← Prod overrides
    ├── application-qa.yml                ← QA overrides
    ├── application-uat.yml               ← UAT overrides
    ├── application-perf.yml              ← Perf overrides
    ├── application-pt.yml                ← PT (Performance Testing) overrides
    ├── db/
    │   └── changelogRoot.yml             ← Liquibase changelog (entry point)
    └── locale/
        └── messages-payment.properties   ← Localized error & notification messages
```

---

### Key Design Patterns Used

| Pattern | Where Used | Purpose |
|---|---|---|
| **REST over HTTP** | All endpoints, client integrations | Standard API communication |
| **Repository Pattern** | `BankRepository`, `PaymentGatewayLinkRepository` | Abstract data access; enable testing via mocks |
| **Service Layer** | `BankService`, `PaymentService`, `PaymentEventService` | Business logic encapsulation; reusable across controllers |
| **MapStruct Mapper** | `PaymentMapper` | Type-safe entity → DTO mapping; compile-time safety |
| **Builder Pattern** | `Payment.builder()`, `Bank.builder()` | Fluent object construction (via Lombok `@Builder`) |
| **Configuration Properties** | `@ConfigurationProperties` on `PaymentGatewayConfig`, `BankConfigProperties` | External config management; env-aware values |
| **Signature Verification** | `PaymentSignatureVerifier` with strategy pattern | HMAC-based webhook authenticity; prevent MITM attacks |
| **Idempotency via Unique Constraint** | `reference_id` in payments table | Prevent duplicate payments from replay attacks |
| **Event-Driven via Kafka** | `PaymentEventService` publishes to Kafka | Async, loosely-coupled downstream notifications |
| **Template Method Pattern** | Error handling in `PaymentsErrorCode` | Consistent error response structure |

---

## 7. Security & Access Control

### Authentication & Authorization

**Current Model:**  
Request headers `entity` and `channel` identify the calling service (Checkout Service, etc.). No JWT or OAuth2 used internally; assumes all internal services are trusted (service-to-service via private network in K8s).

```
POST /initiate
Headers:
  entity: YUBIFIN
  channel: retail
  Content-Type: application/json

Body:
  {
    "productId": "PRODUCT_123",
    "userId": "uuid",
    "amount": 100000,
    "currency": "INR",
    ...
  }
```

**Access Control:**
- **Public endpoints:** None. All endpoints assume caller is an internal Yubi service.
- **Webhook endpoint (`POST /event`):** Only Razorpay can call it; signature verification ensures authenticity (not authorization, but integrity check).
- **Protected endpoints:** All others require `entity` + `channel` headers to determine callback URL and template routing.

> **Note:** No RBAC matrix defined. Inferred from code — verify with security team whether this model is acceptable.

---

### Secrets Management

| Secret | Storage | How Used | Rotation |
|---|---|---|---|
| **Razorpay API Key** | AWS Systems Manager Parameter Store (env var: `paymentGatewayApiKey`) | REST client auth when calling Razorpay API | Quarterly or on compromise |
| **Razorpay API Secret** | AWS Systems Manager Parameter Store (env var: `paymentGatewayApiSecret`) | HMAC-SHA256 signing for API requests | Quarterly or on compromise |
| **Razorpay Webhook Secret** | AWS Systems Manager Parameter Store (env var: `paymentGatewayWebhookSecret`) | Verify webhook signatures from Razorpay | Quarterly or on compromise; must be updated on Razorpay dashboard |
| **Encryption Secret Key** | AWS KMS (env var: `encryptionSecretKey`) | Encrypt/decrypt sensitive fields in payments (if used) | TBD; depends on KMS rotation policy |
| **Database Password** | AWS RDS Secret Manager (env var: `springDatasourcePassword`) | PostgreSQL connection | Automatic rotation via AWS Secrets Manager |
| **Email Notification API Key** | AWS Systems Manager Parameter Store (env var: `emailNotificationKey`) | Authenticate with Notification Service | Quarterly |
| **CodeArtifact Token** | AWS CodeArtifact (env var: `codeArtifactToken`) | Fetch internal libraries from artifact repo | Auto-expiring; renewed on each build |
| **Trusted Server Clients** | Config file (env var: `trustedServerClients`) | Allowlist of servers allowed to call this service | Manual; listed as comma-separated IPs/domains |

**Injection Method:** Environment variables via AWS ECS / K8s ConfigMap + Secrets.  
**Local Development:** `.env` file (not committed); see `application-local.yml` for hardcoded example values.

---

### Signature Verification (Webhooks)

**Algorithm:** HMAC-SHA256 over the JSON request body using `webhook-secret`.

```java
// PaymentSignatureVerifier.java
public boolean verifySignature(String payload, String signature) {
    String computed = hmacSha256(payload, webhookSecret);
    return computed.equals(signature);
}
```

**Header extraction in `PaymentEventController`:**
```java
String eventId = headers.get("x-razorpay-event-id");      // Idempotency key
String signature = headers.get("x-razorpay-signature");   // HMAC-SHA256
```

**If signature invalid:**
- Return HTTP 400 Bad Request.
- Do **not** process the event.
- Log the failed verification for audit.

**Idempotency:**  
Even if the same webhook is received twice (Razorpay retries), `event_id` must be unique. Store processed `event_id` in `razorpay_webhook_events` table with status `PROCESSED` to skip re-processing.

---

### Rate Limiting & Throttling

**Configured:** Not explicitly mentioned in code snapshot.

> **Note:** Inferred from code — payment endpoints likely have no built-in rate limits. Recommend adding Spring Security `@RateLimiter` or AWS API Gateway throttling before production scale-out. **Action item:** Confirm rate limit policy with ops.

---

### Service-to-Service Auth

**Internal Calls (Payment Service → other services):**  
No explicit bearer token or OAuth2 in provided code. Calls use `RestTemplate` with no auth headers.

```java
// Example: calling Trade Service
restTemplate.getForObject("https://partner-qa-api.myyubiinvest.in/tms/...", Object.class);
```

**Assumption:** All internal services run in same VPC/K8s cluster and are protected by network policies (no external access). If cross-region or external SaaS calls are needed, add bearer token or API key headers.

> **Note:** Verify with security team whether current model meets compliance requirements.

---

### Encryption

**At Rest:**  
- PostgreSQL connection uses SSL (configured in JDBC URL).
- Sensitive fields (e.g., bank account numbers) in DB: check if encrypted; not explicit in schema.

**In Transit:**  
- All endpoints use HTTPS (enforced at API Gateway / load balancer level, not in code).
- Razorpay API calls use HTTPS with TLS 1.2+.

**Data Masking in Logs:**  
- Payment amounts and user IDs may appear in logs; ensure PII is masked in log output or sent to secure log aggregator (Kibana/Datadog with appropriate retention policies).

---

## 8. Local Setup & Running the Service

### Prerequisites

- **Java 17** (download from oracle.com or use sdkman: `sdk install java 17.0.x`)
- **Gradle 8.x** (included via `gradlew` wrapper; no separate install needed)
- **Docker** (for PostgreSQL and other containers)
- **Docker Compose** (for orchestrating local stack)
- **IDE:** IntelliJ IDEA or VS Code with Java extensions
- **Git** (for cloning the repo)
- **Postman or cURL** (for testing endpoints)

### Step 1: Clone Repository

```bash
git clone https://github.com/yubisecurities/yubi-fin-b2c-payment-service.git
cd yubi-fin-b2c-payment-service
```

### Step 2: Start Dependencies (PostgreSQL via Docker Compose)

```bash
docker-compose up -d

# Verify containers are running
docker ps
# Output should show:
# - postgres_b2c (port 5432)
# - pgadmin4_postgres_b2c (port 5050)
```

**Verify PostgreSQL is ready:**
```bash
psql -h localhost -U postgres -d yubi-fin-b2c-payment-db -c "SELECT 1;"
# Enter password: root
# Output: 1 (success)
```

**Access PgAdmin (optional):**
- URL: http://localhost:5050
- Email: (see docker-compose.yml; defaults may be blank)
- Password: (see docker-compose.yml)
- Add server manually: hostname=postgres_b2c, port=5432, user=postgres, password=root

### Step 3: Set Up Local Environment Variables

Create `.env` file in project root (NOT committed to git):

```bash
# .env (not committed)

# Database
SPRING_DATASOURCE_URL=jdbc:postgresql://localhost:5432/yubi-fin-b2c-payment-db
SPRING_DATASOURCE_USERNAME=postgres
SPRING_DATASOURCE_PASSWORD=root

# Razorpay (use test keys from Razorpay dashboard)
PAYMENT_GATEWAY_API_KEY=rzp_test_XXXX...
PAYMENT_GATEWAY_API_SECRET=XXXX...
PAYMENT_GATEWAY_WEBHOOK_SECRET=XXXX...

# Email notification (local mock or test key)
EMAIL_NOTIFICATION_KEY=Bearer test-key-12345

# Encryption
ENCRYPTION_SECRET_KEY=your-32-char-secret-key-here-12345

# Amplitude (analytics; can be dummy for local)
AMPLITUDE_URL=https://api.amplitude.com
AMPLITUDE_API_KEY=dummy-local-key

# Trusted servers (allow localhost for testing)
TRUSTED_SERVER_CLIENTS=localhost,127.0.0.1

# Other service URLs (point to local mocks or test servers)
TRADE_SERVICE_URL=http://localhost:8083
OFFER_SERVICE_URL=http://localhost:8084
CHECKOUT_SERVICE_URL=http://localhost:8082
# ... (fill in others as needed; see application-local.yml for all)
```

Load `.env` into your shell:
```bash
export $(cat .env | grep -v '^#' | xargs)
```

Or add to IDE run configuration (IntelliJ: Run → Edit Configurations → add Environment variables).

### Step 4: Build the Service

```bash
# Using gradle wrapper
./gradlew clean build -x test

# Or with tests (slower)
./gradlew clean build
```

**Troubleshooting build failures:**
- **Missing CodeArtifact token:** Ensure `codeArtifactToken` env var is set. If building without internal dependencies, run `./gradlew build -x test`.
- **Checkstyle errors:** Run `./gradlew checkstyleMain --continue` to see all violations; fix code style or adjust `config/checkstyle/checkstyle.xml`.
- **Java version mismatch:** Confirm `java -version` shows 17.x.

### Step 5: Run the Service

```bash
# Profile: local
./gradlew bootRunLocal

# Or specify profile manually
./gradlew bootRun --args='--spring.profiles.active=local'

# Service should start on http://localhost:8081
# (port 8081 is configured in application-local.yml; prod uses 8080)
```

**Expected console output:**
```
Started PaymentApplication in X.XXX seconds
Server is running on port 8081
```

### Step 6: Verify Service is Running

**Health endpoint:**
```bash
curl http://localhost:8081/actuator/health
# Response: {"status":"UP"}
```

**Swagger/OpenAPI docs:**
```
http://localhost:8081/payment/api/v1/swagger-ui.html
```

### Step 7: Common Local Setup Pitfalls

| Pitfall | Symptom | Solution |
|---|---|---|
| **PostgreSQL not running** | Connection refused on localhost:5432 | Run `docker-compose up -d` and verify `docker ps` |
| **Port 5432 already in use** | Bind failed: Address already in use | Stop other PostgreSQL: `lsof -i :5432 \| grep LISTEN \| awk '{print $2}' \| xargs kill -9` |
| **Liquibase migration fails** | Error: "liquibase.*" in logs | Ensure DB exists: `psql -c "CREATE DATABASE \"yubi-fin-b2c-payment-db\";"` |
| **Missing .env variables** | `Cannot resolve property` or `null pointer` | Add all vars from `application-local.yml` to `.env` and export |
| **Port 8081 in use** | Address already in use | Change port in `application-local.yml`: `server.port: 8085` |
| **Gradle build hangs** | Stuck on "Resolving dependencies" | Add gradle daemon flag: `./gradlew build --no-daemon` |
| **CodeArtifact auth fails** | Unauthorized downloading dependencies | Ensure AWS credentials are configured: `aws sts get-caller-identity` should work. Also see if build uses `codeArtifactToken`. |

---

## 9. API Reference

### Required Headers for All Requests

```http
GET /banks/HDFC0001234/payment-methods?amount=100000
  entity: YUBIFIN          # Product identifier (e.g., YUBIFIN, DISTRIBUTOR)
  channel: retail          # Platform identifier (e.g., retail, partner)
  Content-Type: application/json
```

### Endpoint Summary

| Method | Path | Auth | Description | Status |
|---|---|---|---|---|
| **GET** | `/banks/{bank-code}/payment-methods` | entity, channel | Fetch payment methods for a bank/amount | ✅ Prod |
| **GET** | `/payments` | entity, channel | List payments (filters: start-time, end-time, payment-mode, status) | ✅ Prod |
| **POST** | `/initiate` | entity, channel | Initiate a payment (returns razorpay order ID) | ✅ Prod |
| **POST** | `/payments` | entity, channel | Create a payment record | ✅ Prod |
| **POST** | `/{payment-id}/status` | entity, channel | Update payment status | ✅ Prod |
| **PATCH** | `/{payment-id}/status` | entity, channel | Patch update payment status | ✅ Prod |
| **POST** | `/{payment-id}/notes` | entity, channel | Update notes on a payment | ✅ Prod |
| **POST** | `/event` | (Razorpay webhook) | Receive payment webhook (signature verified) | ✅ Prod |
| **POST** | `/event/route` | (Razorpay webhook) | Receive route/settlement webhook | ✅ Prod |
| **POST** | `/event/vba-credit` | (Razorpay webhook) | Receive VBA credit webhook | ✅ Prod |

---

### 1. Get Payment Methods for Bank

**Endpoint:** `GET /banks/{bank-code}/payment-methods`

**Purpose:** Determine which payment methods (UPI, Netbanking) are available for a given bank and amount.

**Request Headers:**
```http
entity: YUBIFIN
channel: retail
Content-Type: application/json
```

**Query Parameters:**
| Param | Type | Required | Example | Notes |
|---|---|---|---|---|
| `amount` | BigDecimal | Yes | 100000 | Amount in base currency units (paise or smallest unit) |
| `currency-code` | String | No (default: INR) | INR | ISO 4217 code |

**Example Request:**
```bash
curl -X GET "http://localhost:8081/payment/api/v1/banks/HDFC0001234/payment-methods?amount=100000&currency-code=INR" \
  -H "entity: YUBIFIN" \
  -H "channel: retail" \
  -H "Content-Type: application/json"
```

**Response (200 OK):**
```json
[
  "UPI",
  "NETBANKING"
]
```

**Response (404 Not Found):** Bank not found
```json
{
  "errors": [
    {
      "errorCode": "err.payment.no-bank-found",
      "message": "Bank not found",
      "violations": [
        {
          "fieldName": "bank code",
          "rejectedValue": "INVALID",
          "message": "Bank not found"
        }
      ]
    }
  ]
}
```

**Response (404 Not Found):** No payment methods supported for amount
```json
{
  "errors": [
    {
      "errorCode": "err.payment.no-payment-methods-supported-for-the-amount",
      "message": "No payment mode supported for the amount by your bank",
      "violations": [
        {
          "fieldName": "bankCode",
          "rejectedValue": "HSBC0001234",
          "message": "bank does not support any mode for given amount : INR 100005.0"
        }
      ]
    }
  ]
}
```

---

### 2. Initiate Payment

**Endpoint:** `POST /initiate`

**Purpose:** Create a payment record and request a Razorpay order ID for checkout redirect.

**Request Headers:**
```http
entity: YUBIFIN
channel: retail
Content-Type: application/json
```

**Request Body:**
```json
{
  "productId": "BOND",
  "userId": "550e8400-e29b-41d4-a716-446655440000",
  "referenceId": "RFQ123_USER456_2026-04-10",
  "amount": 100000,
  "currency": "INR",
  "paymentMethod": "UPI",
  "selectedBankCode": "HDFC",
  "selectedBankIfsc": "HDFC0001234",
  "rfqOrderNumber": "RFQ123",
  "orderNumber": "ORD789",
  "tradeDate": "2026-04-10",
  "category": "BOND",
  "isBasket": false,
  "dematNumber": "DEMAT123"
}
```

**Example Request:**
```bash
curl -X POST "http://localhost:8081/payment/api/v1/initiate" \
  -H "entity: YUBIFIN" \
  -H "channel: retail" \
  -H "Content-Type: application/json" \
  -d '{
    "productId": "BOND",
    "userId": "550e8400-e29b-41d4-a716-446655440000",
    "referenceId": "RFQ123_USER456_2026-04-10",
    "amount": 100000,
    "currency": "INR",
    "paymentMethod": "UPI",
    "selectedBankCode": "HDFC",
    "selectedBankIfsc": "HDFC0001234",
    "rfqOrderNumber": "RFQ123",
    "orderNumber": "ORD789",
    "tradeDate": "2026-04-10",
    "category": "BOND",
    "isBasket": false,
    "dematNumber": "DEMAT123"
  }'
```

**Response (200 OK):**
```json
{
  "paymentId": "550e8400-e29b-41d4-a716-446655440001",
  "razorpayOrderId": "order_1Z8a9B7c6D5e4F3g2H1i0J9",
  "amount": 100000,
  "currency": "INR",
  "status": "CREATED",
  "createdOn": "2026-04-10T10:30:45.123Z"
}
```

**Response (400 Bad Request):** Validation failed or amount exceeds UPI limit
```json
{
  "errors": [
    {
      "errorCode": "err.payment.upi-limit-exceeds",
      "message": "UPI limit exceeds for this amount",
      "violations": []
    }
  ]
}
```

**Response (400 Bad Request):** Invalid trade date
```json
{
  "errors": [
    {
      "errorCode": "err.payment-trade-date-is-null",
      "message": "Trade date is required",
      "violations": []
    }
  ]
}
```

---

### 3. Update Payment Status

**Endpoint:** `POST /{payment-id}/status` or `PATCH /{payment-id}/status`

**Purpose:** Manually update payment status (typically called by Checkout or ops after webhook processing or manual intervention).

**Request Headers:**
```http
entity: YUBIFIN
channel: retail
Content-Type: application/json
```

**Path Parameter:**
| Param | Type | Example |
|---|---|---|
| `payment-id` | UUID | 550e8400-e29b-41d4-a716-446655440001 |

**Request Body:**
```json
{
  "status": "CAPTURED",
  "failureReason": null
}
```

**Example Request:**
```bash
curl -X POST "http://localhost:8081/payment/api/v1/550e8400-e29b-41d4-a716-446655440001/status" \
  -H "entity: YUBIFIN" \
  -H "channel: retail" \
  -H "Content-Type: application/json" \
  -d '{
    "status": "CAPTURED",
    "failureReason": null
  }'
```

**Response (200 OK):**
```json
(no body, just 200)
```

**Response (404 Not Found):** Payment ID doesn't exist
```json
{
  "errors": [
    {
      "errorCode": "err.payment.no-payment-id-found",
      "message": "Payment not found",
      "violations": []
    }
  ]
}
```

---

### 4. Get Payments (List with Filters)

**Endpoint:** `GET /payments`

**Purpose:** Retrieve payments matching filters (time range, payment method, status).

**Request Headers:**
```http
entity: YUBIFIN
channel: retail
```

**Query Parameters:**
| Param | Type | Required | Example |
|---|---|---|---|
| `start-time` | String (ISO 8601) | Yes | 2026-04-09T00:00:00Z |
| `end-time` | String (ISO 8601) | Yes | 2026-04-10T23:59:59Z |
| `payment-mode` | List (comma-separated) | Yes | UPI,NETBANKING |
| `status` | List (comma-separated) | Yes | CAPTURED,FAILED |

**Example Request:**
```bash
curl -X GET "http://localhost:8081/payment/api/v1/payments?start-time=2026-04-09T00:00:00Z&end-time=2026-04-10T23:59:59Z&payment-mode=UPI,NETBANKING&status=CAPTURED,FAILED" \
  -H "entity: YUBIFIN" \
  -H "channel: retail"
```

**Response (200 OK):**
```json
[
  {
    "id": "550e8400-e29b-41d4-a716-446655440001",
    "productId": "BOND",
    "userId": "550e8400-e29b-41d4-a716-446655440000",
    "referenceId": "RFQ123_USER456_2026-04-10",
    "status": "CAPTURED",
    "amount": 100000,
    "currency": "INR",
    "paymentGateway": "RAZORPAY",
    "paymentGatewayOrderId": "order_1Z8a9B7c6D5e4F3g2H1i0J9",
    "paymentGatewayPaymentId": "pay_1Z8a9B7c6D5e4F3g2H1i0J9",
    "selectedPaymentMode": "UPI",
    "selectedBankAccount": "9876543210",
    "selectedBankIfsc": "HDFC0001234",
    "selectedBankName": "HDFC Bank",
    "selectedBankAccountType": "SAVINGS",
    "createdOn": "2026-04-10T10:30:45Z",
    "lastUpdatedOn": "2026-04-10T10:31:02Z",
    "tradeDate": "2026-04-10",
    "dematNumber": "DEMAT123"
  }
]
```

---

### 5. Update Payment Notes

**Endpoint:** `POST /{payment-id}/notes`

**Purpose:** Add or update internal notes on a payment (e.g., ops team annotation for troubleshooting).

**Request Headers:**
```http
entity: YUBIFIN
channel: retail
Content-Type: application/json
```

**Path Parameter:**
| Param | Type | Example |
|---|---|---|
| `payment-id` | UUID | 550e8400-e29b-41d4-a716-446655440001 |

**Request Body:**
```json
{
  "notes": "Dispute resolved; payment confirmed by customer service"
}
```

**Example Request:**
```bash
curl -X POST "http://localhost:8081/payment/api/v1/550e8400-e29b-41d4-a716-446655440001/notes" \
  -H "entity: YUBIFIN" \
  -H "channel: retail" \
  -H "Content-Type: application/json" \
  -d '{"notes": "Dispute resolved"}'
```

**Response (200 OK):**
```json
{
  "paymentId": "550e8400-e29b-41d4-a716-446655440001",
  "notes": "Dispute resolved; payment confirmed by customer service",
  "lastUpdatedOn": "2026-04-10T11:00:00Z"
}
```

---

### 6. Receive Payment Webhook (from Razorpay)

**Endpoint:** `POST /event`

**Purpose:** Webhook receiver for payment status updates from Razorpay (automatically called by Razorpay, not by human).

**Request Headers (sent by Razorpay):**
```http
x-razorpay-event-id: evt_1Z8a9B7c6D5e4F3g2H1i0J9
x-razorpay-signature: d41d8cd98f00b204e9800998ecf8427e
Content-Type: application/json
```

**Request Body (example: payment captured):**
```json
{
  "event": "payment.captured",
  "payload": {
    "payment": {
      "entity": {
        "id": "pay_1Z8a9B7c6D5e4F3g2H1i0J9",
        "entity": "payment",
        "amount": 100000,
        "currency": "INR",
        "status": "captured",
        "order_id": "order_1Z8a9B7c6D5e4F3g2H1i0J9",
        "invoice_id": null,
        "international": false,
        "method": "upi",
        "amount_refunded": 0,
        "refund_status": null,
        "captured": true,
        "description": "Bond purchase",
        "card_id": null,
        "bank": null,
        "wallet": null,
        "vpa": "user@hdfc",
        "email": "user@example.com",
        "contact": "+919876543210",
        "notes": {},
        "fee": 590,
        "tax": 90,
        "error_code": null,
        "error_description": null,
        "error_source": null,
        "error_reason": null,
        "acquirer_data": {
          "rrn": "123456789012"
        },
        "created_at": 1712746245
      }
    }
  }
}
```

**Processing:**
1. Payment Service validates signature using `webhook-secret`.
2. Extracts `payment.entity` from payload.
3. Looks up payment in DB using Razorpay `payment_id`.
4. Updates local payment record: `status = CAPTURED`, `lastUpdatedOn = now()`.
5. Publishes event to Kafka: `{paymentId, status, userId, orderId, ...}`.

**Response (200 OK):**
```json
(no body, just HTTP 200 to acknowledge receipt)
```

**Response (400 Bad Request):** Signature invalid or headers missing
```json
(no body, just HTTP 400 to tell Razorpay to retry later)
```

---

### API Versioning Strategy

**Current version:** `v1` (in path: `/payment/api/v1`)

**Breaking change handling:**
- New major versions (v2, v3) introduced as new paths: `/payment/api/v2/...`.
- v1 endpoints deprecated (but kept for backward compatibility) for 2-3 quarters.
- Notification sent to all consumers 1 month before deprecation.
- Clients must migrate to v2 before v1 sunset.

> **Note:** No documented v2 roadmap provided. Verify with team.

---

## 10. How to Extend the System

### Example 1: Add a New Document Type (e.g., Equity Options)

**Scenario:** Product team wants to support options trading; each option has a price, expiry, and strike price. Payment Service must handle it like bonds.

**Steps:**

1. **Add new product type to config:**
   ```yaml
   # application.yml
   yubi:
     fin:
       b2c:
         payment:
           supported-products:
             - BOND
             - EQUITY
             - OPTIONS  # ← NEW
   ```

2. **Update PaymentInitRequest validation** (if product-specific rules needed):
   ```java
   // model/request/PaymentInitRequest.java
   @Data
   public class PaymentInitRequest {
       private String productId;  // e.g., "OPTIONS"
       private String symbolCode; // e.g., "NIFTY50CALL"
       private BigDecimal strikePrice;
       private LocalDate expiryDate;
       // ... other fields
   }
   ```

3. **Add product-specific validation in PaymentService:**
   ```java
   // service/PaymentService.java
   public PaymentInitResponse initiatePayment(...) {
       if ("OPTIONS".equals(request.getProductId())) {
           validateOptionsOrder(request);  // ← NEW METHOD
       }
       // ... rest of initiation logic
   }

   private void validateOptionsOrder(PaymentInitRequest request) {
       // Check expiry date is in future
       // Check strike price is within valid range
       // etc.
   }
   ```

4. **Update database migration** (if new columns needed):
   ```sql
   -- liquibase/002-add-options-fields.postgresql.sql
   ALTER TABLE payment.payments ADD COLUMN option_symbol_code VARCHAR(20);
   ALTER TABLE payment.payments ADD COLUMN option_strike_price DECIMAL;
   ALTER TABLE payment.payments ADD COLUMN option_expiry_date DATE;
   ```

5. **Update Payment entity:**
   ```java
   // model/entity/Payment.java
   @Entity
   @Table(name = "payments", schema = "payment")
   public class Payment {
       // ... existing fields
       @Column(name = "option_symbol_code")
       private String optionSymbolCode;
       @Column(name = "option_strike_price")
       private BigDecimal optionStrikePrice;
       @Column(name = "option_expiry_date")
       private LocalDate optionExpiryDate;
   }
   ```

6. **Update PaymentMapper:**
   ```java
   // model/mapper/PaymentMapper.java
   @Mapper(unmappedTargetPolicy = ReportingPolicy.IGNORE)
   public interface PaymentMapper {
       @Mapping(source = "optionSymbolCode", target = "optionSymbolCode")
       @Mapping(source = "optionStrikePrice", target = "optionStrikePrice")
       @Mapping(source = "optionExpiryDate", target = "optionExpiryDate")
       PaymentResponse toDomain(Payment payment);
       // ... other mappings
   }
   ```

7. **Test new product type:**
   ```java
   // test/java/com/yubi/fin/b2c/payment/OptionsPaymentTest.java
   @Test
   void shouldInitiateOptionsPayment() {
       PaymentInitRequest request = PaymentInitRequest.builder()
           .productId("OPTIONS")
           .symbolCode("NIFTY50CALL")
           .strikePrice(BigDecimal.valueOf(25000))
           .expiryDate(LocalDate.now().plusDays(30))
           .amount(BigDecimal.valueOf(50000))
           .build();
       
       PaymentInitResponse response = paymentService.initiatePayment(request, "YUBIFIN", "retail");
       
       assertNotNull(response.getPaymentId());
       assertEquals("CREATED", response.getStatus());
   }
   ```

8. **Deploy:**
   - Merge PR to `master`.
   - Jenkins pipeline auto-triggers: build → SonarQube scan → push to ECR → ArgoCD deploys to k8s.
   - Monitor: check payment logs for OPTIONS transactions; verify settlement works end-to-end.

---

### Example 2: Add a New Async Listener (Kafka Consumer)

**Scenario:** A new Settlement Service needs to listen for payment completion events to trigger immediate settlement (instead of current async approach).

**Steps:**

1. **Add Kafka consumer listener in PaymentService:**
   ```java
   // service/PaymentService.java (or create SettlementKafkaListener.java)
   
   @Component
   @Slf4j
   public class SettlementKafkaListener {
       
       @Autowired
       private RestTemplate restTemplate;
       
       @KafkaListener(
           topics = "bonds-event-dev",
           groupId = "payment-settlement-group",
           containerFactory = "kafkaListenerContainerFactory"
       )
       public void onPaymentCaptured(PaymentEvent event) {
           log.info("Received payment event: {}", event.getPaymentId());
           
           if ("CAPTURED".equals(event.getStatus())) {
               try {
                   // Call Settlement Service
                   SettlementRequest settlementRequest = SettlementRequest.builder()
                       .paymentId(event.getPaymentId())
                       .userId(event.getUserId())
                       .orderId(event.getOrderId())
                       .amount(event.getAmount())
                       .build();
                   
                   restTemplate.postForObject(
                       "http://settlement-service:8085/settle",
                       settlementRequest,
                       SettlementResponse.class
                   );
                   
                   log.info("Settlement triggered for payment: {}", event.getPaymentId());
               } catch (Exception e) {
                   log.error("Settlement failed for payment: {}", event.getPaymentId(), e);
                   // Send to DLQ (automatic retry or manual intervention)
               }
           }
       }
   }
   ```

2. **Add DLQ configuration:**
   ```java
   // config/KafkaConfig.java
   @Configuration
   public class KafkaConfig {
       
       @Bean
       public ConcurrentKafkaListenerContainerFactory kafkaListenerContainerFactory(
               ConsumerFactory<String, PaymentEvent> consumerFactory) {
           ConcurrentKafkaListenerContainerFactory<String, PaymentEvent> factory =
               new ConcurrentKafkaListenerContainerFactory<>();
           factory.setCommonErrorHandler(new DefaultErrorHandler(
               new FixedBackOff(1000L, 3L)  // Retry 3 times with 1-second backoff
           ));
           return factory;
       }
   }
   ```

3. **Add retry and DLQ topic setup in Liquibase/config:**
   ```sql
   -- Kafka topics created via script or managed by ops team
   -- Topic: bonds-event-dev (existing)
   -- Topic: bonds-event-dev.DLT (auto-created for failed messages after retries)
   ```

4. **Monitor DLQ:**
   ```bash
   # Check DLQ using Kafka console consumer
   kafka-console-consumer --bootstrap-servers localhost:9094 \
     --topic bonds-event-dev.DLT \
     --from-beginning
   ```

5. **Handle DLQ manually:**
   - Ops team periodically reviews DLQ messages.
   - If settlement request was transient failure (e.g., Settlement Service was down), manually reprocess.
   - If permanent failure (e.g., invalid settlement request), log for audit and skip.

6. **Test:**
   ```java
   @Test
   void shouldRetrySettlementOnFailureAndSendToDLQ() {
       // Mock Settlement Service to fail first, then succeed
       when(restTemplate.postForObject(...))
           .thenThrow(new RestClientException("Service unavailable"))
           .thenReturn(successResponse);
       
       PaymentEvent event = PaymentEvent.builder()
           .paymentId(UUID.randomUUID())
           .status("CAPTURED")
           .userId(UUID.randomUUID())
           .build();
       
       settlementListener.onPaymentCaptured(event);
       
       // Verify retry happened
       verify(restTemplate, times(2)).postForObject(anyString(), any(), any());
   }
   ```

---

### Example 3: Add a New Role or Permission

**Scenario:** Operations team needs read-only access to view payment logs, but not modify payments.

**Current state:** No RBAC implemented; all endpoints assume caller is trusted internal service.

**Proposed change:**

1. **Add role enum:**
   ```java
   // config/PaymentRole.java
   public enum PaymentRole {
       ADMIN("admin", "Full access to all endpoints"),
       OPS("ops", "Read-only access; can view payments, cannot modify"),
       SERVICE("service", "Full access; internal service calls"),
       PUBLIC("public", "Webhook receiver; no auth needed");
       
       private final String code;
       private final String description;
       
       PaymentRole(String code, String description) {
           this.code = code;
           this.description = description;
       }
   }
   ```

2. **Add role annotation:**
   ```java
   // config/RequireRole.java
   @Target(ElementType.METHOD)
   @Retention(RetentionPolicy.RUNTIME)
   public @interface RequireRole {
       PaymentRole[] value() default {};
   }
   ```

3. **Add interceptor to check role:**
   ```java
   // config/PaymentRoleInterceptor.java
   @Component
   public class PaymentRoleInterceptor implements HandlerInterceptor {
       
       @Override
       public boolean preHandle(HttpServletRequest request, 
                                HttpServletResponse response, 
                                Object handler) throws Exception {
           if (handler instanceof HandlerMethod) {
               RequireRole roleAnnotation = 
                   ((HandlerMethod) handler).getMethodAnnotation(RequireRole.class);
               
               if (roleAnnotation != null) {
                   String userRole = request.getHeader("x-user-role");
                   PaymentRole[] allowedRoles = roleAnnotation.value();
                   
                   boolean hasRole = Arrays.stream(allowedRoles)
                       .anyMatch(role -> role.code.equals(userRole));
                   
                   if (!hasRole) {
                       response.setStatus(HttpServletResponse.SC_FORBIDDEN);
                       response.getWriter().write("Insufficient permissions");
                       return false;
                   }
               }
           }
           return true;
       }
   }
   ```

4. **Apply annotation to endpoints:**
   ```java
   // controller/PaymentController.java
   
   @GetMapping("/payments")
   @RequireRole({PaymentRole.ADMIN, PaymentRole.OPS, PaymentRole.SERVICE})
   public ResponseEntity<List<PaymentResponse>> getPayments(...) {
       // Read-only; OPS role allowed
       ...
   }
   
   @PostMapping("/{payment-id}/status")
   @RequireRole({PaymentRole.ADMIN, PaymentRole.SERVICE})  // OPS NOT allowed
   public ResponseEntity<Void> updatePaymentStatus(...) {
       // Modification endpoint; OPS role NOT allowed
       ...
   }
   ```

5. **Test:**
   ```java
   @Test
   void shouldDenyOpsRoleFromModifyingPayment() throws Exception {
       mockMvc.perform(post("/payments/{id}/status")
           .header("x-user-role", "ops")
           .contentType(MediaType.APPLICATION_JSON)
           .content("..."))
           .andExpect(status().isForbidden());
   }
   ```

---

### Example 4: Add a New Error Code

**Scenario:** Settlement Service needs to reject payments for invalid demat accounts; Payment Service should return a user-friendly error.

**Steps:**

1. **Add error code enum:**
   ```java
   // config/PaymentsErrorCode.java
   public enum PaymentsErrorCode implements ErrorCode {
       // ... existing codes
       INVALID_DEMAT_ACCOUNT("err.payment.invalid-demat-account"),
       DEMAT_ACCOUNT_BLOCKED("err.payment.demat-account-blocked");
       
       private String code;
       
       PaymentsErrorCode(String code) {
           this.code = code;
       }
       
       @Override
       public String get() {
           return code;
       }
   }
   ```

2. **Add i18n message:**
   ```properties
   # resources/locale/messages-payment.properties
   err.payment.invalid-demat-account=The demat account associated with your profile is invalid. Please update your account details and retry.
   err.payment.demat-account-blocked=Your demat account is currently blocked. Please contact support.
   ```

3. **Use in service:**
   ```java
   // service/PaymentService.java
   public PaymentInitResponse initiatePayment(...) {
       // Validate demat account
       if (!isValidDematAccount(request.getDematNumber())) {
           throw Errors.builder()
               .errorCode(PaymentsErrorCode.INVALID_DEMAT_ACCOUNT)
               .violation("dematNumber", request.getDematNumber(), 
                          "Demat account is invalid")
               .httpStatus(HttpStatus.BAD_REQUEST)
               .applicationError();
       }
       // ... rest
   }
   ```

4. **Test:**
   ```java
   @Test
   void shouldThrowErrorForInvalidDematAccount() {
       PaymentInitRequest request = PaymentInitRequest.builder()
           .dematNumber("INVALID123")
           .build();
       
       ApplicationException exception = assertThrows(ApplicationException.class,
           () -> paymentService.initiatePayment(request, "YUBIFIN", "retail"));
       
       assertEquals(PaymentsErrorCode.INVALID_DEMAT_ACCOUNT.get(), 
                    exception.getErrorCode());
   }
   ```

---

### Example 5: Error Handling Pattern

**General pattern for throwing errors:**

```java
// util/ErrorHandlingExample.java

// Validation error (400 Bad Request)
throw Errors.builder()
    .errorCode(PaymentsErrorCode.UPI_LIMIT_EXCEEDS)
    .violation("amount", request.getAmount(), 
               "Amount exceeds UPI limit of 100000 INR")
    .httpStatus(HttpStatus.BAD_REQUEST)
    .applicationError();

// Not found error (404)
throw Errors.builder()
    .errorCode(PaymentsErrorCode.NO_BANK_FOUND)
    .violation("bankCode", bankCode, "Bank not found in database")
    .httpStatus(HttpStatus.NOT_FOUND)
    .applicationError();

// Server error (500)
throw Errors.builder()
    .errorCode(PaymentsErrorCode.FAILED_CREATING_RAZORPAY_CUSTOMER)
    .args(razorpayException.getMessage())
    .httpStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    .applicationError();
```

---

## 11. Data Model & Status Rules Engine

### Core Database Tables

| Table | Purpose | Key Indexes | Notes |
|---|---|---|---|
| `payment.payments` | Main payment record per order | PK: id; FK: user_id; UQ: reference_id | Stores capture state, error details, timestamps |
| `payment.bank_payment_modes` | Supported payment methods per bank | PK: bank_code | Seeded data; rarely changes |
| `payment.razorpay_webhook_events` | Webhook event log | PK: event_id | Prevents duplicate processing via event_id |
| `payment.payment_gateway_link` | Maps local payment to Razorpay payment | PK: id; FK: payment_id; UQ: payment_gateway_payment_id | Enables idempotency; recovers on retries |

### Schema (Inferred from Liquibase Migration)

```sql
-- payment.payments
CREATE TABLE payment.payments (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    product_id VARCHAR(8) NOT NULL,
    user_id UUID NOT NULL,
    reference_id VARCHAR(50) NOT NULL UNIQUE,
    status VARCHAR(20) NOT NULL DEFAULT 'CREATED',
    amount DECIMAL NOT NULL,
    currency VARCHAR(20) NOT NULL,
    payment_gateway VARCHAR(20) NOT NULL,
    payment_gateway_order_id VARCHAR(20) NOT NULL,
    payment_gateway_payment_id VARCHAR(20),
    error JSONB,
    created_on TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    last_updated_on TIMESTAMP,
    deleted BOOLEAN DEFAULT FALSE,
    deleted_on TIMESTAMP,
    
    -- Additional fields (inferred from Payment entity)
    rfq_order_number VARCHAR(50),
    internal_order_number VARCHAR(50),
    order_number VARCHAR(50),
    selected_payment_mode VARCHAR(20),
    selected_bank_account VARCHAR(50),
    selected_bank_ifsc VARCHAR(50),
    selected_bank_name VARCHAR(100),
    selected_bank_account_type VARCHAR(20),
    is_email_notified BOOLEAN DEFAULT FALSE,
    is_settled BOOLEAN DEFAULT FALSE,
    trade_date DATE,
    rfq_deal_id VARCHAR(50),
    category VARCHAR(20),
    is_basket BOOLEAN DEFAULT FALSE,
    demat_number VARCHAR(50),
    
    INDEX idx_user_created (user_id, created_on),
    INDEX idx_reference_id (reference_id),
    INDEX idx_status (status),
    INDEX idx_payment_gateway_payment_id (payment_gateway_payment_id)
);

-- payment.bank_payment_modes
CREATE TABLE payment.bank_payment_modes (
    bank_code VARCHAR(4) PRIMARY KEY,
    bank_name VARCHAR(100) NOT NULL,
    supports_net_banking BOOLEAN NOT NULL,
    supports_upi BOOLEAN NOT NULL
);

-- payment.razorpay_webhook_events
CREATE TABLE payment.razorpay_webhook_events (
    event_id VARCHAR(50) PRIMARY KEY,
    event_status VARCHAR(20) NOT NULL,
    event_data JSONB,
    attempts INTEGER DEFAULT 0
);
```

### Key Enums

```java
// PaymentStatus — Payment state machine
CREATED          // Payment initiated; awaiting capture
AUTHORIZED       // Payment authorized (pre-capture state; unclear if used)
CAPTURED         // Payment captured; funds received
FAILED           // Payment declined or failed
REFUNDED         // Refund completed
REFUND_IN_PROGRESS // Refund initiated; awaiting confirmation from Razorpay

// PaymentEventStatus — Webhook processing state
RECEIVED         // Webhook received; awaiting processing
PROCESSED        // Webhook fully processed; payment status updated
FAILED           // Processing failed; retry later

// PaymentUpdateStatus — Callback status (from checkout/downstream services)
SUCCESS          // Payment update succeeded
FAILED           // Update failed
REFUNDED         // Payment refunded

// PaymentMethod — Available payment methods per bank
UPI              // Unified Payments Interface (≤ ₹100,000)
NETBANKING       // Direct bank transfer (unlimited)
```

### Status/State Rules

**Payment Status State Machine (Deterministic):**

```
CREATED
  ├─→ [Razorpay webhook: payment.captured] ──→ CAPTURED
  │
  └─→ [Razorpay webhook: payment.failed] ──────→ FAILED

CAPTURED
  └─→ [Refund initiated] ──────────────────────→ REFUND_IN_PROGRESS
                                                    │
                                                    └─→ [Razorpay webhook: refund.created] ──→ REFUNDED

FAILED
  └─→ (User retries or ops re-initiates) ──────→ [New payment record with CREATED status]
```

**Idempotency Rules:**

- **Unique constraint:** `reference_id` must be unique per user + order.
  - If same `reference_id` sent twice → second request returns existing payment (no new record).
  - Rationale: Prevents duplicate charges from retried API calls.

- **Webhook idempotency:**
  - Same `event_id` (from Razorpay header) processed only once.
  - If duplicate webhook received → skip processing; return 200 OK to Razorpay (ack receipt without re-processing).

**Caching Strategy:**

- **Bank payment modes** (static data): Cached in memory or Redis with 24-hour TTL. Rarely changes; reduces DB queries.
- **Payment records**: Not cached; always read from DB to avoid stale status.
- **Razorpay order ID**: Stored in DB; not cached externally.

> **Note:** Caching strategy inferred from code. Verify with team; Redis config not explicitly shown in provided files.

### Audit Logging

**Captured events:**
1. Payment creation (amount, user, product, status=CREATED)
2. Payment status transition (old status → new status)
3. Webhook receipt and processing
4. Error/failure reasons
5. Refund initiation and completion

**Stored in:**
- PostgreSQL `payment.payments` table (status changes via `last_updated_on` and audit columns if present).
- Kafka topic `audit_events` (integration with core audit service).
- Application logs (structured JSON logs to Kibana/CloudWatch).

**Retention:**
- DB: indefinite (or per data retention policy; TBD).
- Kafka: 7-30 days (depends on Kafka broker config).
- Logs: 90 days (CloudWatch default).

---

## 12. Integration Map

### Visual Integration Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                        Payment Service                          │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │                   Core Business Logic                   │   │
│  └─────────────────────────────────────────────────────────┘   │
└──────────┬──────────────────┬──────────────────────┬────────────┘
           │                  │                      │
           │ REST APIs        │ Kafka Publish       │ Kafka Subscribe
           │                  │                      │
     ┌─────▼──────┐    ┌─────▼──────┐      ┌───────▼────────┐
     │ Checkout   │    │ bonds-event│      │ Audit Service  │
     │ Service    │    │ -dev       │      │ (audit_events) │
     │            │    │            │      │                │
     │ • Initiate │    ├──→ Trade   │      └────────────────┘
     │   payment  │    │    Service │
     │ • Status   │    │ • Settle   │
     │   updates  │    │   order    │
     └────────────┘    │            │
                       ├──→ Notif   │
                       │    Service │
                       │ • Email    │
                       │ • SMS      │
                       └────────────┘

           │
      ┌────▼─────────────────────────────────────────┐
      │         Razorpay Payment Gateway             │
      │  ┌──────────────────────────────────────┐   │
      │  │ • Create Customer                    │   │
      │  │ • Create Order                       │   │
      │  │ • Capture Payment                    │   │
      │  │ • Create Refund                      │   │
      │  │ • Webhook Callbacks (payment.*,      │   │
      │  │   route.*, settlement.*)             │   │
      │  └──────────────────────────────────────┘   │
      └────────────────────────────────────────────┘

           │
    ┌──────▼──────────────────────┐
    │   Internal Yubi Services    │
    │  (via REST calls)           │
    │  • KYC Service              │
    │  • Trade Service            │
    │  • User Service             │
    │  • Discovery Service        │
    │  • Offer Service            │
    │  • Notification Service     │
    │  • Trade Date Service       │
    │  • ... (15+ services)       │
    └─────────────────────────────┘
```

---

### Integration Table

| System | Purpose | Protocol | Direction | Auth | Failure Handling |
|---|---|---|---|---|---|
| **Razorpay API** | Create orders, capture payments, refunds | REST + TLS | Bidirectional | API Key + HMAC signature | Retry with exponential backoff; log error; alert ops |
| **Razorpay Webhooks** | Payment status callbacks | HTTP POST + TLS | Razorpay → Payment Svc | HMAC-SHA256 header verification | Return 400 if invalid; Razorpay retries; eventually → ops manual intervention |
| **Kafka (bonds-event-dev)** | Publish payment completion events | Kafka binary | Publish | SSL + SASL | Async; failed messages go to DLQ; ops review & reprocess |
| **Trade Service** | Settlement & trade execution | REST | REST calls; async via event | Trusted network | Async event; Trade Service handles retry/backoff |
| **Checkout Service** | Receive payment requests | REST | REST calls | Trusted network | Synchronous; caller retries; Payment Svc logs error |
| **KYC Service** | Validate user KYC status | REST | REST call during payment init | Trusted network | Fail payment if KYC invalid; return error |
| **Notification Service** | Send SMS/email receipts | Kafka + REST | Async event consumer | Trusted network | Event in DLQ if Notification Svc is down; retried later |
| **Discovery Service** | Query available offers | REST | REST call during init (optional) | API Key | Non-critical; skip if unavailable; log warning |
| **Trade Date Service** | Validate trade dates | REST | REST call during init | Trusted network | Fail payment if date invalid |
| **Audit Service** | Log all events for compliance | Kafka | Event publish | SSL + SASL | Async; failed messages logged; ops review periodically |
| **Amplitude** | Analytics & user behavior tracking | REST (HTTPS) | Async event tracking | API Key | Non-critical; skip if unavailable; log warning |

---

## 13. Operational Runbook

### Common Failure Modes

| Failure Mode | Symptoms | Likely Cause | Resolution |
|---|---|---|---|
| **Razorpay API down** | All payment initiations fail; errors like "Connection refused to api.razorpay.com" | Razorpay infrastructure issue or network outage | 1. Check Razorpay status page. 2. Check DNS resolution: `nslookup api.razorpay.com`. 3. Notify Razorpay support. 4. Redirect users to retry after 30min. |
| **Database connection pool exhausted** | Payment Service becomes unresponsive; logs: "Cannot get a connection, pool error Timeout waiting for idle object" | High load or DB connection leak | 1. Check active connections: `SELECT count(*) FROM pg_stat_activity;` 2. Kill idle sessions: `SELECT pg_terminate_backend(pid) FROM pg_stat_activity WHERE state = 'idle';` 3. Restart Payment Service pod (K8s). 4. Monitor connection pool size. |
| **Webhook signature verification fails** | Webhooks rejected; logs: "EVENT: X has invalid signature" | Webhook secret mismatch or compromised payload | 1. Verify webhook secret in Razorpay dashboard. 2. Verify webhook secret in Payment Service config (env var). 3. If recent Razorpay key rotation: update env var in K8s secret. 4. Restart Payment Service. |
| **Payment stuck in CREATED state** | User reports payment taken but order not processed | Webhook never received or processing failed | 1. Check `razorpay_webhook_events` table: did webhook arrive? 2. If no → Razorpay failed to call webhook; check Razorpay logs. If yes → check Payment Service logs for processing error. 3. Manually update payment.status to CAPTURED (if verified with user). 4. Publish event to Kafka to trigger settlement. |
| **Virtual Account (VBA) transfer not received** | User initiated bank transfer to VBA; no payment captured | VBA credit webhook not processed | 1. Check DLQ for VBA credit events. 2. If in DLQ → manual retry or fix processing logic. 3. Check Razorpay VBA logs for credit received timestamp. |
| **Refund stuck in REFUND_IN_PROGRESS** | User refund initiated 3+ days ago; status not updated to REFUNDED | Razorpay refund webhook failed or lost | 1. Check Razorpay dashboard: is refund status "processed" there? 2. If yes → webhook likely lost; manually trigger event or update DB status. 3. If no → refund still processing; wait another 2 days (refunds can take up to 5 business days). |
| **Email notifications not sent** | Payment captured but user didn't receive receipt email | Notification Service down or email template misconfigured | 1. Check if notification event in DLQ. 2. Check Notification Service health: `curl https://notification-svc/health`. 3. Check email template name in config: `yubi.fin.b2c.email-notification.order-receipt-email-template`. 4. Verify email sender address is whitelisted. |
| **UPI payment fails for amount ≤ 100,000** | User selects UPI but Payment Service returns "no payment methods supported" | UPI limit config set to 0 or misconfigured | 1. Check config: `yubi.fin.b2c.bank.max-amount-allowed-for-upi` (should be 100000 or 200000). 2. If wrong value → update in K8s ConfigMap. 3. Restart Payment Service. |
| **Trade date validation error on every payment** | All payments rejected with "Trade date is null" or "invalid trade date" | Trade Date Service returning errors | 1. Check Trade Date Service health. 2. Check network connectivity. 3. Manually verify trade date format in Payment Service: should be ISO 8601 (YYYY-MM-DD). 4. Disable trade date validation temporarily (if allowed). |
| **Memory leak / OOM (out of memory)** | Payment Service pod crashes; logs: "java.lang.OutOfMemoryError: Java heap space" | Connection leak, large object in memory, or memory config too low | 1. Check heap dump: `jmap -dump:live,format=b,file=heap.bin <pid>`. 2. Analyze with Eclipse MAT. 3. Increase JVM heap: `JAVA_OPTS=-Xmx2048m` in K8s deployment. 4. Restart pod. 5. Review code for leak (e.g., unclosed connections). |

---

### How to Check DLQ / Retry Queue State

**Kafka DLQ (Dead Letter Topic):**

```bash
# SSH into Kafka pod or use kafka-console-consumer CLI

# List topics (look for .DLT suffix)
kafka-topics --bootstrap-servers localhost:9094 --list | grep DLT

# Consume from DLQ (show last 10 messages)
kafka-console-consumer \
  --bootstrap-servers kafka-broker:9094 \
  --topic bonds-event-dev.DLT \
  --from-beginning \
  --max-messages 10 \
  --security-protocol SSL

# Check DLQ size (message count)
kafka-run-class kafka.tools.JmxTool \
  --object-name kafka.server:type=ReplicaManager,name=UnderReplicatedPartitions
```

**Database: Check webhook retries**

```sql
-- PostgreSQL: check failed webhook events
SELECT event_id, event_status, attempts, event_data
FROM payment.razorpay_webhook_events
WHERE event_status = 'FAILED'
ORDER BY event_id DESC
LIMIT 10;

-- Check how many times event was processed
SELECT event_id, COUNT(*) as attempt_count
FROM payment.razorpay_webhook_events
GROUP BY event_id
HAVING COUNT(*) > 1;
```

---

### How to Manually Trigger or Unstick a Stuck Workflow

**Scenario 1: Payment stuck in CREATED, user funds were captured**

1. **Verify in Razorpay dashboard:** Check if payment status is "captured" there.

2. **If captured in Razorpay:**
   ```sql
   -- Manually update payment status (CAREFUL: backup first!)
   UPDATE payment.payments
   SET status = 'CAPTURED',
       last_updated_on = CURRENT_TIMESTAMP
   WHERE id = 'payment-uuid-here'
   AND status = 'CREATED';
   ```

3. **Publish event to Kafka manually** (trigger settlement):
   ```java
   // In a temporary script or Spring Boot admin console:
   PaymentEvent event = PaymentEvent.builder()
       .paymentId(UUID.fromString("payment-uuid-here"))
       .status("CAPTURED")
       .build();
   eventPusher.publish(event);  // Kafka publish
   ```

4. **Verify settlement triggered:** Check Trade Service logs and order status.

---

**Scenario 2: Webhook in DLQ; need to reprocess**

1. **Extract message from DLQ:**
   ```bash
   kafka-console-consumer \
     --bootstrap-servers kafka:9094 \
     --topic bonds-event-dev.DLT \
     --from-beginning \
     --max-messages 1 \
     > dlq-message.json
   ```

2. **Analyze message:** Check error field and payload.

3. **Reproduce locally or in staging:** Verify fix applies.

4. **Republish to main topic:**
   ```bash
   kafka-console-producer \
     --broker-list kafka:9094 \
     --topic bonds-event-dev \
     < dlq-message.json
   ```

5. **Monitor reprocessing:** Check logs and DB for successful update.

---

### Key Log Queries / Kibana Saved Searches

**Critical logs to monitor (CloudWatch / Kibana):**

```
# Payments created today
fields @timestamp, payment_id, user_id, amount, status
| filter @message like /PaymentService.*initiatePayment/
| stats count() by status

# Payment failures
fields @timestamp, payment_id, error_code, error_message
| filter @message like /PaymentService.*FAILED/
| stats count() by error_code

# Webhook processing
fields @timestamp, event_id, event_status, duration_ms
| filter @message like /PaymentEventService.*handle/
| stats avg(duration_ms), max(duration_ms) by event_status

# API latency
fields @timestamp, endpoint, http_status, response_time_ms
| filter @message like /PaymentController/
| stats pct(response_time_ms, 95), pct(response_time_ms, 99) by endpoint

# Razorpay API errors
fields @timestamp, error_code, attempt
| filter @message like /RazorpayClient.*error/
| stats count() by error_code
```

---

### Rollback Procedure

**If new Payment Service deployment causes payment failures:**

1. **Identify bad deployment:**
   ```bash
   kubectl rollout history deployment/payment-service -n b2c
   # Output shows revisions; find bad one (newest)
   ```

2. **Rollback to previous stable version:**
   ```bash
   kubectl rollout undo deployment/payment-service -n b2c
   # Or specific revision:
   kubectl rollout undo deployment/payment-service -n b2c --to-revision=5
   ```

3. **Monitor after rollback:**
   ```bash
   kubectl logs -f deployment/payment-service -n b2c
   curl https://payment-service/actuator/health
   ```

4. **Verify payments work again:**
   - Run smoke test: initiate test payment, verify status updates.
   - Check error rates in Datadog/Kibana: should drop within 5min.

5. **Post-mortem:** Debug the bad deployment; fix; re-test before re-deploying.

---

## 14. Testing, Deployment & Infrastructure

### How to Run Tests Locally

```bash
# Run all tests
./gradlew test

# Run specific test class
./gradlew test --tests BankServiceTest

# Run specific test method
./gradlew test --tests BankServiceTest.shouldGetPaymentModesUpiAndNetBanking

# Run with coverage report
./gradlew test jacocoTestReport
# Open: build/reports/jacoco/test/html/index.html

# Run tests in parallel (faster)
./gradlew test --parallel
```

---

### Test Pattern Example (from BankControllerTest.java)

```java
@WebMvcTest(controllers = BankController.class)
public class BankControllerTest {

    private final ObjectMapper mapper = new ObjectMapper();

    @Autowired
    private MockMvc mockMvc;

    @MockBean
    private BankService bankService;

    @Test
    public void shouldFetchUpiAndNetBankingAsPaymentModes() throws Exception {
        // Arrange: set up test data
        String bankCode = "HDFC";
        String currencyCode = "INR";
        BigDecimal amount = BigDecimal.valueOf(100000d);
        List<PaymentMethod> expected = Arrays.asList(PaymentMethod.UPI, PaymentMethod.NETBANKING);

        when(bankService.getPaymentModes(bankCode,
                BigMoney.of(CurrencyUnit.of(currencyCode), amount)))
            .thenReturn(expected);

        // Act: make HTTP request
        MvcResult mvcResult = mockMvc
                .perform(get("/banks/" + bankCode + "/payment-methods?amount=" + amount))
                .andExpect(status().isOk())
                .andReturn();

        // Assert: verify response
        List<PaymentMethod> actual = Arrays.asList(
            mapper.readValue(mvcResult.getResponse().getContentAsString(), PaymentMethod[].class));
        assertTrue(actual.containsAll(expected));
    }
}
```

**Pattern breakdown:**
- **@WebMvcTest:** Load only controller layer; mock services.
- **@MockBean:** Replace real service with mock.
- **Arrange-Act-Assert:** Organize test clearly.
- **MockMvc.perform():** Simulate HTTP request.
- **andExpect():** Assert HTTP response code/body.

---

### Environment Matrix

| Env | Replicas | Resources (CPU/Mem) | Purpose | Kafka Topic | DB |
|---|---|---|---|---|---|
| **Local** | 1 | 1 CPU / 1GB | Developer machine | localhost (embedded or docker) | localhost:5432 |
| **Dev** | 2 | 0.5 CPU / 512MB | Feature branch testing | bonds-event-dev | RDS dev-replica |
| **QA** | 3 | 1 CPU / 1GB | Pre-prod testing; ops validation | bonds-event-qa | RDS qa-replica |
| **UAT** | 3 | 1 CPU / 1GB | User acceptance testing | bonds-event-uat | RDS uat-replica |
| **Perf** | 4 | 2 CPU / 2GB | Load testing; performance baseline | bonds-event-perf | RDS perf-replica |
| **Prod** | 5+ | 2 CPU / 2GB | Production traffic | bonds-event-prod | RDS prod (Multi-AZ) |

---

### CI/CD Pipeline Diagram

```
┌──────────┐
│ Developer│ Pushes code to master
│ commits  │
└─────┬────┘
      │
      ▼
┌──────────────────────────────────────────────────────────────────┐
│ Jenkins Pipeline (triggered on master push)                      │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ 1. Checkout code                                        │    │
│  │ 2. Build JAR: ./gradlew clean build                     │    │
│  │ 3. Run checkstyle & SonarQube analysis                  │    │
│  │ 4. Run unit + integration tests                         │    │
│  │ 5. Build Docker image & push to ECR                     │    │
│  │ 6. Update ArgoCD manifests in git                       │    │
│  └─────────────────────────────────────────────────────────┘    │
│                           │                                      │
│                    ✓ All checks pass                             │
│                           │                                      │
└──────────────┬────────────┘
               │
               ▼
┌──────────────────────────────────────────────────────────────────┐
│ ArgoCD (Continuous Deployment)                                   │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ 1. Detect image tag update in K8s manifests git repo   │    │
│  │ 2. Sync desired state: deploy new pod to cluster       │    │
│  │ 3. Rolling update: 1 pod at a time (0 downtime)        │    │
│  │ 4. Health check: wait for pod ready (15s-30s)          │    │
│  │ 5. Signal ready: route traffic to new pod              │    │
│  └─────────────────────────────────────────────────────────┘    │
│                           │                                      │
│                    ✓ Deployment complete                         │
│                           │                                      │
└──────────────┬────────────┘
               │
               ▼
        ┌──────────────┐
        │ Kubernetes   │
        │ Production   │
        │ Cluster      │
        │              │
        │ payment-svc  │
        │ :8080 ✓✓✓✓✓ │
        └──────────────┘
```

---

### Related Repositories

| Repo | Role | Maintained By |
|---|---|---|
| `yubi-fin-b2c-core-api-client` | Shared DTOs, API client interfaces (dependency) | B2C Core Team |
| `yubi-fin-b2c-core-api-common` | Common exceptions, error codes, logging (dependency) | B2C Core Team |
| `yubi-fin-b2c-core-integration` | Event publishing, Kafka clients (dependency) | B2C Core Team |
| `yubi-fin-b2c-audit-integration` | Audit event publishing (dependency) | Audit Team |
| `yubi-fin-b2c-checkout-service` | Payment Service consumer; calls `/initiate` and listens for status updates | Checkout Team |
| `yubi-fin-b2c-trade-service` | Consumes payment events from Kafka; settles orders | Trade Team |
| `yubi-fin-b2c-notification-service` | Consumes payment events; sends SMS/email receipts | Notification Team |
| `yubi-fin-b2c-kyc-service` | Called during payment init to verify KYC status | KYC Team |
| `yubi-fin-b2c-infrastructure` | Terraform configs, Helm charts, K8s manifests | DevOps Team |

---

### Known Gaps & Improvement Areas

| Gap | Impact | Owner (if known) | Recommended Fix |
|---|---|---|---|
| **No rate limiting** | Service vulnerable to DOS attacks; no throttling configured | DevOps / Security | Add Spring `@RateLimiter` or AWS API Gateway throttling (100 req/s per IP) |
| **No RBAC** | All internal services treated equally; cannot restrict OPS to read-only | Payment Service Team | Implement role-based access control (see §10, Example 3) |
| **Refund handling incomplete** | Code mentions refunds but endpoint not shown in API spec; refund flow unclear | Payment Service Team | Clarify refund flow; expose `/refund` endpoint if needed |
| **No circuit breaker** | Calls to external services (Razorpay, KYC) may hang indefinitely | Payment Service Team | Add Spring Cloud Circuit Breaker; timeout after 5 seconds |
| **Webhook retry policy unclear** | Razorpay docs don't specify retry schedule; could miss events | Payment Service Team | Confirm retry policy with Razorpay; implement exponential backoff storage |
| **No database encryption at rest** | Sensitive data (amounts, user IDs) stored in plaintext in RDS | Security / DevOps | Enable RDS encryption via AWS KMS |
| **Manual settlement triggering** | No easy way to retry settlement if Trade Service is temporarily down | Payment Service Team | Add `/trigger-settlement/{payment-id}` endpoint for ops |
| **Limited observability** | No metrics exported (request latency, error rates by reason) | DevOps / Monitoring | Add Micrometer + Prometheus metrics |
| **No async refund notification** | Refund completion emails may not be sent if Notification Svc is slow | Notification Team | Decouple refund notification from Payment Service response |

---

### Known Developer Gotchas

| Surprising Behavior | Explanation | Workaround |
|---|---|---|
| **Entity vs Channel headers required** | All endpoints expect `entity` and `channel` headers; easy to forget during testing | Always include headers in curl/Postman; add to test fixtures |
| **reference_id must be unique per payment** | Duplicate reference IDs will fail insertion (unique constraint); important for idempotency | Generate reference_id as `rfq_order_id + user_id + timestamp` (collision-proof) |
| **Payment statuses are strings in DB, not enums** | Status stored as VARCHAR; Java code uses enum but DB has string | Always use exact enum value names (e.g., "CAPTURED", not "Captured"); test against real DB, not H2 |
| **Trade date validation happens twice** | Trade Service validates trade date; Payment Service also validates during init | If validation logic changes in one place, update both (or centralize) |
| **Razorpay test vs live keys switch at runtime** | Different Razorpay accounts for dev/prod; easy to use test key in prod | Env vars are environment-specific; verify in K8s ConfigMap before deploy |
| **Virtual Account (VBA) webhook is separate endpoint** | `/event/vba-credit` is not the standard `/event` endpoint; easy to miss | Ensure Razorpay sends VBA credits to correct webhook URL in dashboard |
| **Email templates are named, not inlined** | Email templates live in Notification Service, not Payment Service | Changes to email content require Notification Service deploy, not Payment Service |
| **Soft deletes in DB** | `deleted` flag is never null; filtering requires `WHERE deleted = FALSE` | Always include deletion filter in SELECT queries; risk of showing deleted payments |
| **Timestamps in database are UTC** | Created_on / last_updated_on are stored in UTC; may be confusing in logs | Convert to user's timezone when displaying in UI; clarify in API docs |

---

## Appendix

### Additional Configuration Reference

**Offer Service Schedule** (from `application-*.yml`):
```yaml
yubi:
  fin:
    b2c:
      payment:
        offer:
          hour: 5           # Fetch offers at 5:30 AM
          minute: 30
          holiday: false    # Skip if market holiday
```

**Trade Date URL Pattern:**
```
URL: https://partner-qa-api.myyubiinvest.in/rfq/api/v1/invalid_trade_dates
Purpose: Returns list of dates when trades cannot be executed (market closed)
```

**Trusted Server Clients** (comma-separated IP/domain allowlist):
```
Local:   localhost, 127.0.0.1
Dev:     b-3.intcentralizedkafka01.pv5xpn.c2.kafka.ap-south-1.amazonaws.com, ...
QA:      partner-qa-api.myyubiinvest.in, retail-qa-api.aspero.co.in, ...
Prod:    partner-api.myyubiinvest.in, retail-api.aspero.co.in, ...
```

---

### Contact & Escalation

**Payment Service Owner:** B2C Fin Squad

**Slack Channels:**
- **#b2c-fin-payments** — General discussions & alerts
- **#b2c-fin-payments-oncall** — On-call escalations (production issues)
- **#b2c-payment-service-deploy** — Deployment notifications

**Key Contacts:**
- Product Lead: [TBD — check JIRA]
- Tech Lead: [TBD — check JIRA]
- DevOps: [TBD — check JIRA]

**Runbook Link:** [TBD — confluence or internal wiki]

**Monitoring Dashboards:**
- **Grafana:** grafana.internal/d/payment-svc
- **Datadog:** datadog.com/dashboard/payment-service

---

**Document Generated:** 2026-04-10  
**Last Updated:** 2026-04-10  
**Version:** 1.0  
**Status:** Draft (Ready for Team Review)

> **Note:** This document is inferred from code and configuration files. Verify with team before using as source of truth for critical decisions. Update this document whenever significant architecture changes occur.