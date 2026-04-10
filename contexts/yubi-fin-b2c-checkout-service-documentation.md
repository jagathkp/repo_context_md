# yubi-fin-b2c-checkout-service — Master Context

> **Repository:** yubi-fin-b2c-checkout-service (master branch)  
> **Stack:** Java 17 · Spring Boot 3.4.3 · PostgreSQL · Redis · Kafka · AWS MSK  
> **API Docs:** `/checkout/api/v1/swagger-ui.html` (local)  
> **Related Services:** payment-service, kyc-service, notification-service, user-service, offer-service, tms-service, rfq-service, discovery-service  
> **Owner Team:** YUBI FIN B2C Team

---

## TL;DR (30-second read)

- **What:** Processes checkout and payments for YUBI FIN's B2C investment platform (bonds, fixed deposits, sovereign gold bonds)
- **Trigger:** User initiates payment for investment product
- **Output:** Payment processed, transaction recorded, settlement initiated, confirmation sent
- **Status:** Production (CSPL migration underway)
- **Owner:** YUBI FIN B2C Backend Team

---

## Who Should Read This

| Audience | Sections | Why |
|----------|----------|-----|
| **Product Manager** | Glossary, What Is This Service, User Journeys, Conditional Flows, Notifications | Understand business logic, user experience, configurable rules, feature gates |
| **Backend Developer** | All sections | Full context needed for feature development, debugging, and local setup |
| **DevOps / Infrastructure** | Local Setup, Secrets Management, CI/CD, Kubernetes, Operational Runbook | Deployment, environment config, troubleshooting |
| **QA / Test Engineer** | API Reference, User Journeys, Conditional Flows, Testing, Runbook | Test coverage, edge cases, failure scenarios |
| **New Team Member** | Glossary, TL;DR, What Is This Service, Local Setup, Architecture | Onboarding context and quick wins |

---

## Table of Contents

1. [Glossary](#1-glossary)
2. [What Is This Service?](#2-what-is-this-service)
3. [Who Uses It and How](#3-who-uses-it-and-how)
4. [The User Journey — Plain English](#4-the-user-journey--plain-english)
5. [Conditional Flows — Which Workflow Applies to Whom](#5-conditional-flows--which-workflow-applies-to-whom)
6. [Architecture & Code Structure](#6-architecture--code-structure)
7. [Security & Access Control](#7-security--access-control)
8. [Local Setup & Running the Service](#8-local-setup--running-the-service)
9. [API Reference](#9-api-reference)
10. [How to Extend the System](#10-how-to-extend-the-system)
11. [Data Model & Status Rules Engine](#11-data-model--status-rules-engine)
12. [Integration Map](#12-integration-map)
13. [Operational Runbook](#13-operational-runbook)
14. [Testing, Deployment & Infrastructure](#14-testing-deployment--infrastructure)

---

## 1. Glossary

| Term | Definition |
|------|-----------|
| **Bond** | Fixed income security issued by government or corporations; user invests lump sum to receive coupons + principal at maturity |
| **Fixed Deposit (FD)** | Blostem-provided savings instrument; user deposits principal for fixed tenure at guaranteed interest rate |
| **SGB** | Sovereign Gold Bond; government-issued security backed by physical gold; users invest instead of buying physical gold |
| **Blostem** | Third-party FD provider integrated via webhook events; handles onboarding, VKYC, FD booking, maturity |
| **Demat Account** | Dematerialized securities account; required to hold bonds and SGBs; linked to Primary Demat Account (PDA) |
| **DPID** | Depository Participant ID; identifies the NSDL/CDSL holder; used in settlement |
| **KYC** | Know Your Customer; identity verification done once during onboarding |
| **VKYC** | Video KYC; identity verification via video call; required for some FD products |
| **Payment Method** | Way to fund an order (UPI, Net Banking, NEFT); enabled/disabled per bank and amount |
| **IFSC Code** | Indian Financial System Code; unique bank branch identifier; used to determine payment methods |
| **ISIN** | International Securities Identification Number; unique identifier for a bond offering |
| **Listing ID** | YUBI FIN internal ID for a specific bond/SGB offering |
| **Order ID** | YUBI FIN internal ID for a user's checkout transaction |
| **JID** | Journey ID; Blostem-side transaction ID; links Blostem FD events to YUBI FIN tracking |
| **UPI Transaction Limit** | Max amount per UPI payment; currently ₹1,00,000 (₹2,00,000 with feature flag) |
| **CSPL** | Credit Suisse Private Limited; platform migration underway (referenced in README) |
| **Offer** | Personalized investment recommendation (bond, SGB, FD); includes pricing and terms |
| **RFQ** | Request for Quote; used in bonds trading to fetch live pricing |
| **TMS** | Trade Management System; downstream service that records settled trades |
| **Razorpay** | Payment gateway for UPI and Net Banking transactions |
| **WebEngage** | Engagement platform; tracks user events for marketing analytics |
| **Event Topic** | Kafka topic where checkout events are published (e.g., `bonds-event-dev`) |
| **External Event Topic** | Kafka topic for third-party integrations (e.g., Blostem webhooks) |
| **Audit Topic** | Kafka topic where compliance-relevant events are logged |
| **Channel** | Platform identifier (e.g., "yubifin", "invest", "aspero"); controls feature availability and branding |

---

## 2. What Is This Service?

**One-line purpose:**  
Orchestrates the end-to-end checkout, payment, and settlement workflow for retail investors purchasing bonds, fixed deposits, and sovereign gold bonds on the YUBI FIN platform.

**Why it exists:**  
- **Regulatory:** KYC, VKYC, and audit logging are mandatory for securities trading in India.
- **Business:** Requires seamless, multi-product checkout experience with real-time payment confirmation and settlement tracking.
- **Technical:** Decouples payment processing, product-specific workflows (bond vs. FD), and downstream settlement systems.

**What it does NOT do:**  
- Does NOT perform identity verification (delegated to KYC service).
- Does NOT calculate bond pricing or FD interest (delegated to Offer service).
- Does NOT execute bank transfers or record trades (delegated to TMS and bank APIs).
- Does NOT store bank account credentials (delegated to Bank Account service).
- Does NOT manage user wallets or fund transfers (delegated to Payment service).

**Value to the business:**  
- Enables users to invest in bonds/FDs with a single, secure checkout flow.
- Ensures regulatory compliance (audit, settlement tracking).
- Integrates multiple product types and payment methods under one API surface.
- Provides real-time payment status and transaction reconciliation.

---

## 3. Who Uses It and How

| Stakeholder | Interaction Pattern | What They Need |
|-------------|-------------------|-----------------|
| **Retail Investor (End User)** | Uses mobile app / web → Browsing offers → Adds to cart → Checkout → Pay → Confirmation | Fast checkout, multiple payment options, clear error messages, confirmation email/SMS |
| **Frontend / Mobile Team** | Calls checkout APIs to initialize payment, fetch bank/payment methods, poll transaction status | Clear API contracts, idempotent endpoints, real-time status updates, error codes with user-friendly messages |
| **Payment Operations** | Monitors failed payments, retries, manual interventions | Transaction logs, payment status breakdown, DLQ for retries |
| **Compliance / Legal** | Audits KYC and settlement records | Audit logs with user ID, product ID, timestamp, status changes |
| **Support Team** | Investigates failed orders, reconciles mismatches | Transaction lookup by user ID / order ID, status history, payment method details |
| **Product Team** | Enables/disables features (UPI, bond basket), adjusts transaction limits | Config-driven feature flags, no code deploy needed for A/B testing |

### Supported Product Types

| Product | Description | Checkout Flow | Settlement |
|---------|-----------|---|---|
| **Bond** | Fixed-income security; 5–10 year tenure; monthly/annual coupons | Browse offer → Pay via UPI/Net Banking → T+1 settlement | TMS records trade within 24 hours |
| **Fixed Deposit** | Blostem provider; 6–24 month tenure; fixed interest | Browse offer → Initiate VKYC → Complete payment → FD booking confirmed | Blostem handles maturity; events flow back via webhooks |
| **Sovereign Gold Bond (SGB)** | Government gold security; 8-year tenure; interest on redemption | Browse offer → Pay via UPI/Net Banking → T+1 settlement | TMS records trade; physical gold backing maintained by government |

---

## 4. The User Journey — Plain English

### **Journey 1: Bond Purchase (Happy Path)**

1. **Browse & Discover**
   - User opens app, views curated bond offerings (provided by Discovery service).
   - Each offer shows interest rate, maturity date, minimum investment, tax implications.

2. **Build Cart**
   - User selects one or more bonds → Specifies quantity (units).
   - System calculates total investment amount.

3. **Initiate Checkout**
   - User clicks "Proceed to Checkout".
   - System verifies:
     - User KYC status is **APPROVED**.
     - User has at least one verified bank account.
     - User's primary demat account is available.
   - If all verified, system presents payment options (UPI, Net Banking).

4. **Select Payment Method**
   - User's verified banks are displayed with their enabled payment methods.
   - Example: HDFC Bank offers both UPI and Net Banking; ICICI offers only Net Banking.
   - System checks UPI limit: if amount > ₹1,00,000 and UPI feature enabled, UPI is available; otherwise Net Banking only.
   - **Amount-based routing:** Payment methods are filtered per bank based on per-bank transaction limits.

5. **Authorize & Pay**
   - User selects bank + payment method.
   - System calls Razorpay gateway to initiate payment.
   - Gateway redirects user to bank portal (UPI app or Net Banking login).
   - User authorizes payment.

6. **Payment Confirmation**
   - On success, Razorpay notifies checkout service (webhook or poll).
   - System updates transaction status to `PAYMENT_SUCCESSFUL`.
   - System records order in database with mapping to demat account, bank, amount.

7. **Settlement Notification**
   - Email sent: "Your investment of ₹X in Bond Y is confirmed. Settlement on [date]."
   - SMS sent with order number.
   - Engagement event sent to WebEngage for analytics.

8. **Post-Settlement**
   - Within T+1 (next trading day), TMS service records the settled trade.
   - Bonds appear in user's demat account (via depository).
   - User receives bond certificate / holding statement.

### **Journey 2: Fixed Deposit Purchase (With VKYC)**

1. **Browse & Select**
   - User selects FD offering from Blostem provider.
   - Offer shows tenure, interest rate, maturity amount.

2. **Initiate Checkout**
   - User provides investment amount.
   - System checks if VKYC is required (based on FD terms or risk profile).

3. **VKYC Process (if required)**
   - System initiates Blostem VKYC flow.
   - Blostem returns a JID (journey ID) to track this VKYC session.
   - System polls or listens for Blostem `VKYC_COMPLETED` event.
   - Until VKYC completes, FD remains in `VKYC_PENDING` status.

4. **Bank Account & Payment**
   - Once VKYC completes (or if not required), system fetches user's verified bank accounts.
   - User selects bank account for withdrawal (post-maturity).
   - User initiates payment via UPI or Net Banking (same as bond journey).

5. **FD Booking Confirmation**
   - On payment success, system sends `PAYMENT_SUCCESSFUL` event to Blostem.
   - Blostem creates FD in its system and returns status `FD_BOOKED`.
   - System updates order status to `FD_BOOKED`.
   - Email sent: "Your FD of ₹X is confirmed. Maturity date: [date]. Interest rate: Y%."

6. **VKYC Failed Path**
   - If Blostem sends `ONBOARDING_FAILED` or `VKYC_REJECTED`, transaction moves to `REJECTED` status.
   - User is notified of rejection reason and can retry.

7. **Maturity Flow**
   - FD provider monitors maturity date.
   - Blostem sends `FD_MATURED` event.
   - System updates transaction status to `MATURED`.
   - Amount + interest credited to user's linked bank account.
   - Maturity notification email sent.

### **Journey 3: Payment Failure & Retry**

1. **Payment Fails**
   - Razorpay returns error (e.g., insufficient balance, incorrect PIN, timeout).
   - System updates transaction status to `PAYMENT_FAILED`.
   - Error message shown to user on screen + via SMS/email.

2. **User Retries**
   - User can immediately retry with same or different payment method.
   - System allows multiple retry attempts (no global lock on order).
   - Each retry creates a separate payment transaction record.

3. **Refund (if applicable)**
   - If partial payment happened (rare) or user cancels within X minutes, refund initiated.
   - Razorpay handles refund to original payment method.
   - System status moves to `REFUND_INITIATED` → `REFUND_COMPLETED`.

### **Journey 4: Alternative — Bond Basket**

*(If feature flag `is-bond-basket-enabled: true`)*

1. **Create Basket**
   - User selects 2–5 bonds of different durations.
   - System calculates combined maturity ladder.

2. **Checkout**
   - Total amount calculated across all bonds.
   - All bonds settled together in single transaction.

3. **Confirmation**
   - Composite confirmation email listing all bonds.
   - Each bond appears separately in demat account once settled.

---

## 5. Conditional Flows — Which Workflow Applies to Whom

### **Routing Dimensions Matrix**

```
                    ┌─────────────────────────────────┐
                    │ Checkout Service Routing        │
                    └─────────────────────────────────┘
                                │
                ┌───────────────┼───────────────┐
                ▼               ▼               ▼
          Product Type     User Status      Amount
             (Bond,       (KYC/VKYC       (UPI limit:
             FD, SGB)     complete?)      ₹1,00,000)
                │              │              │
                └──────────────┬──────────────┘
                               ▼
                    Payment Method Available
                    (Net Banking, UPI, NEFT)
```

### **Product × User Type Decision Table**

| Product | KYC Required | VKYC Required | Bank Account | Demat Account | Payment Methods |
|---------|-------------|--------------|-------------|--------------|-----------------|
| **Bond** | ✓ APPROVED | ✗ No | ✓ APPROVED | ✓ Primary | UPI (if amount ≤ ₹1L), Net Banking |
| **SGB** | ✓ APPROVED | ✗ No | ✓ APPROVED | ✓ Primary | UPI (if amount ≤ ₹1L), Net Banking |
| **FD** | ✓ APPROVED | Conditional* | ✓ APPROVED (for withdrawal) | ✗ Not needed | UPI (if amount ≤ ₹1L), Net Banking |

*VKYC required if: user age < 18, or amount > ₹5 lakhs, or Blostem's policy.

### **Feature Flags & Behavior**

| Flag | Location | Effect When ON | Effect When OFF |
|------|----------|---|---|
| `is-upi-enabled` | `application-{env}.yml` | UPI payment option shown if amount ≤ limit | Only Net Banking available |
| `is-bond-basket-enabled` | `application-{env}.yml` | Multi-bond checkout allowed | Single bond only |
| `iccl-notes-enabled` | `application-{env}.yml` | Additional notes field shown at checkout | Notes field hidden |
| `is-trade-check-skipped` | `application-{env}.yml` | Trade settlement checks bypassed (DEV/QA) | Full settlement validation (PROD) |

### **Status Decision Logic**

```
Order Status Lifecycle (per product):

┌─────────────┐
│   CREATED   │
└──────┬──────┘
       │
       ├─ Initiate payment
       ▼
┌──────────────────────┐
│  PAYMENT_PENDING     │
└──────┬───────────────┘
       │
       ├─ Success? ──────────────┐
       │                         │
       ├─ Failure? ──┐           │
       │             │           │
       ▼             ▼           ▼
   PAYMENT_     PAYMENT_    PAYMENT_
   FAILED       PENDING     SUCCESSFUL
      │                          │
      └──────┬──────────┬────────┘
             │          │
       (Retry)   (For FD: VKYC)
             │          │
             │          ▼
             │      VKYC_PENDING ──────┐
             │          │              │
             │          ▼ (Blostem)    │
             │      VKYC_INITIATED     │
             │          │              │
             │          ▼              │
             │      VKYC_COMPLETED ────┤
             │                         │
             └────────┬────────────────┘
                      ▼
            (Bond: T+1 Settlement)
            (FD: FD_BOOKED status)
                      │
                      ▼
                 SETTLED / OPEN
```

### **Pre-Requisite Gates (Must Pass Before Checkout)**

```
User initiates checkout
    │
    ├─ KYC status == APPROVED? ─── NO ──► STOP: "Complete KYC first"
    │                               │
    │ YES                           │
    │                         (user directed to KYC service)
    ├─ At least 1 verified bank account? ── NO ──► STOP: "Add a bank account"
    │                                          │
    │ YES                                  (user directed to bank verification)
    │
    ├─ Primary demat account exists? ── NO ──► STOP: "Link a demat account"
    │                                      │
    │ YES                              (user directed to demat linking)
    │
    ├─ Trading window open? ── NO ──► STOP: "Market is closed"
    │                             │
    │ YES                      (show retry time)
    │
    └─► PROCEED to payment method selection
```

### **What Can Be Changed Without Code**

| Configuration | Location | How to Change | Example |
|---------------|----------|--|---|
| **UPI Transaction Limit** | `application-{env}.yml: yubi.fin.b2c.checkout.transaction.max-limit-for-upi` | Update env var, redeploy (no code change) | Change ₹1L to ₹2L for high-volume users |
| **Email Templates** | Notification service DB | Update via admin panel | Change "Your payment is confirmed" text |
| **Email Recipients** | `application-{env}.yml: yubi.fin.b2c.checkout.email-notification.{from,cc,bcc}-email` | Update env var | Add new CC email for ops |
| **SMS Templates** | Notification service DB | Update via admin panel | Add transaction ID to SMS |
| **Feature Flags** | `application-{env}.yml: yubi.fin.b2c.checkout.feature` | Update env var, restart service | Enable bond-basket for QA testing |
| **API Endpoints (internal)** | `application-{env}.yml: yubi.fin.b2c.api.client.internal.*` | Update env var | Change KYC service URL during migration |
| **Kafka Topics** | `application-{env}.yml: yubi.fin.b2c.event.kafka.{event-topic,external-event-topic}` | Update env var | Route to different topic during testing |
| **Redis Cluster** | `application-{env}.yml: spring.redis` | Update env var | Add new Redis node |
| **Database** | `application-{env}.yml: spring.datasource` | Update env var | Point to new PostgreSQL instance |

### **What Requires Code Change**

| Change | Reason | Impact |
|--------|--------|--------|
| Add new product type (e.g., Bonds → Mutual Funds) | Workflow logic, status enums, settlement routing differ | Medium: ~3–5 files, ~500 LOC |
| Change payment method routing logic | Hard-coded in BankService | Small: 1–2 files, ~200 LOC |
| Add new payment method (e.g., Crypto) | Status enums, payment gateway integration, UI flows differ | Large: 5–10 files, ~1000+ LOC |
| Change status transition rules | Embedded in BlostemEventService, CheckoutTransactionService | Medium: 2–3 files, ~400 LOC |
| Add new integration (e.g., new demat provider) | New controller, service, repository, entity | Large: 10+ files, ~1500+ LOC |

### **Notification Triggers**

| Event | Who Is Notified | Channel | Condition | Template |
|-------|-----------------|---------|-----------|----------|
| Payment Successful | User | Email, SMS | Transaction status = PAYMENT_SUCCESSFUL | `bond-b2c-payment-success-email-template` + `payment-success-sms` |
| Payment Failed | User | Email, SMS | Transaction status = PAYMENT_FAILED | `bond-b2c-payment-failure-email-template` + `payment-failure-sms` |
| FD Booked | User | Email | Transaction status = FD_BOOKED | (Blostem handles) |
| Bond Maturing in 30 Days | User | Email | From scheduled task via notification service | `maturing_bond_email_template` with redirect link |
| Trade Reported | User | Email, SMS | (Internal event) | `bond-b2c-trade-reported-email-template` + `trade-reported-sms` |
| Bond Basket Confirmation | User | Email | If `is-bond-basket-enabled: true` | `bond-b2c-basket-email-template` + `bond-b2c-basket-sms-template` |
| SGB Payment Success | User | Email, SMS | (SGB-specific) | `sgb-b2c-payment-success-email-template` + `sgb-b2c-payment-success-sms` |
| VKYC Completed | User | (In-app notification) | Blostem event VKYC_COMPLETED received | N/A (status update) |
| VKYC Pending | User | Email | After X hours of pending | (From notification service) |
| Order Cancellation | User | Email | User initiates refund | (Custom template) |

---

## 6. Architecture & Code Structure

### **High-Level Architecture Diagram**

```
                                ┌──────────────────────────────────┐
                                │    Frontend (Mobile / Web)        │
                                └────────────────┬───────────────────┘
                                                 │
                                    ┌────────────▼────────────┐
                                    │  API Gateway / LB       │
                                    │  (x-user-id header)     │
                                    └────────────┬────────────┘
                                                 │
                        ┌────────────────────────▼────────────────────────┐
                        │   Checkout Service (This service)               │
                        │  ┌─────────────┬─────────────┬─────────────┐    │
                        │  │  Bank       │  Blostem    │  Transaction│    │
                        │  │  Controller │  Controller │  Controller │    │
                        │  └─────┬───────┴──────┬──────┴──────┬──────┘    │
                        │        │              │             │            │
                        │  ┌─────▼──────────────▼─────────────▼─────┐     │
                        │  │  BankService  | BlostemEventService   │     │
                        │  │  FdOrderService | CheckoutTransService │     │
                        │  └────────┬──────────────┬────────────┬───┘     │
                        │           │              │            │         │
                        │  ┌────────▼──────┬───────▼────┬──────▼────┐    │
                        │  │  Repositories │  Mappers   │ Config    │    │
                        │  └─────┬──────────┘            └──────┬───┘    │
                        │        │                             │         │
                        └────────┼─────────────┬────────────────┼────────┘
                                 │             │                │
                 ┌───────────────▼──┐  ┌──────▼────────┐  ┌───▼────────┐
                 │  PostgreSQL DB   │  │  Redis Cache  │  │  Kafka     │
                 │  (checkout_db)   │  │  (events)     │  │ (events)   │
                 └──────────────────┘  └───────────────┘  └────────────┘
                        │
         ┌──────────────┼──────────────┐
         │              │              │
    ┌────▼────┐    ┌───▼────┐    ┌───▼─────┐
    │  KYC    │    │Payment  │    │ User    │
    │  Svc    │    │  Svc    │    │  Svc    │
    └────┬────┘    └────┬───┘    └───┬────┘
         │              │            │
    ┌────▼────────┐  ┌──▼────┐  ┌───▼──────┐
    │ Discovery   │  │Razorpay│  │Offer Svc │
    │   Svc       │  │Gateway │  │(RFQ, OMS)│
    └─────────────┘  └────────┘  └──────────┘

         ┌──────────────┬──────────────┬──────────────┐
         │              │              │              │
    ┌────▼────┐    ┌───▼────┐    ┌───▼──────┐   ┌───▼────┐
    │Blostem  │    │ TMS    │    │Notif Svc │   │WebEngage│
    │ (FD)    │    │(Trade) │    │(Email/SMS)  │
    └─────────┘    └────────┘    └──────────┘   └────────┘
```

### **Request Lifecycle: Checkout API Call**

```
User submits order from frontend
    │
    ▼
POST /checkout/api/v1/banks/payment-methods/v2
    Headers: x-user-id, channel
    Body: { amount, units, isin }
    │
    ▼
BankController.getAllPaymentMethods()
    │
    ├─ Extract and validate headers
    │
    ├─ Call BankService.getAllPaymentMethods()
    │  │
    │  ├─ Fetch user's verified bank accounts via KycApi
    │  │
    │  ├─ For each bank:
    │  │  │
    │  │  ├─ Get IFSC code from KYC data
    │  │  │
    │  │  ├─ Query RazorpayService.getBankPaymentModesFromIFSCCode(ifsc)
    │  │  │
    │  │  ├─ Check if bank is enabled for given amount
    │  │  │  (is UPI enabled? is NetBanking enabled? does amount exceed limit?)
    │  │  │
    │  │  └─ Return BankPaymentDetails with payment modes
    │  │
    │  ├─ Aggregate across all banks
    │  │
    │  └─ Return PaymentModesResponse with total bank count, UPI availability
    │
    ▼
HTTP 200
    Body: { banks: [...], isUPIAllowedForAmount: true, totalBankCount: 3 }
    │
    ▼
Frontend renders payment method selector
    │
    ▼
User selects bank + payment method (UPI or NetBanking)
    │
    ▼
Frontend calls payment gateway (Razorpay) directly or via backend
    │
    ▼
User authorizes payment
    │
    ▼
Razorpay returns success/failure
    │
    ▼
Backend receives webhook or frontend polls for status
    │
    └─► (See Async Flow below)
```

### **Async Flow: Blostem FD Event Processing**

```
Blostem sends webhook event
    │
    ▼
POST /checkout/api/v1/blostem/event
    Body: { jid, event: "PAYMENT", data: { paymentCompleted: true, ... } }
    │
    ▼
BlostemController.handleEvent(payload)
    │
    ▼
BlostemEventService.handleEvent(payload)
    │
    ├─ Convert JsonNode → BlostemEventEntity
    │
    ├─ Switch on event type:
    │  │
    │  ├─ PAYMENT: updateTransactionStatus(PAYMENT_SUCCESSFUL)
    │  │
    │  ├─ VKYC: handle VKYC completion (VKYC_INITIATED → VKYC_COMPLETED)
    │  │
    │  ├─ FD_CREATED: create FixedDepositTransaction with FD_BOOKED status
    │  │
    │  ├─ MATURITY: transition to MATURED, trigger notification
    │  │
    │  ├─ ONBOARDING_FAILED: set status to REJECTED, notify user
    │  │
    │  └─ (other events)
    │
    ├─ Update FixedDepositTransaction in DB
    │
    ├─ Send email/SMS notification (EmailNotificationEvent)
    │
    ├─ Save event in BlostemEventRepository (for idempotency)
    │
    └─ Return HTTP 200 OK (webhook acknowledged)
    │
    ▼
Event logged to Kafka (audit_events topic)
    │
    ▼
TMS or downstream services consume event and update their state
```

### **Package/Directory Navigation**

```
src/main/java/com/yubi/fin/b2c/checkout/
├── CheckoutApplication.java ......................... Spring Boot entry point
│
├── bank/ ............................................. Bank account & payment method selection
│   ├── BankController.java .......................... @RestController: GET /banks, GET /payment-methods
│   ├── BankService.java ............................. Fetches verified banks, determines payment methods
│   ├── domain/
│   │   └── BankAccount.java ......................... Entity: bank account details from KYC
│   ├── model/
│   │   ├── BankPaymentDetails.java ................. DTO: bank + available payment modes
│   │   ├── PaymentModes.java ........................ Enum: UPI, NETBANKING, etc.
│   │   └── PaymentModesResponse.java ............... Response: aggregated payment options
│   └── repository/ ................................... (Note: uses KycApi; no direct DB repo)
│
├── blostem/ ........................................... Fixed Deposit integration (Blostem provider)
│   ├── controller/
│   │   ├── BlostemController.java .................. @RestController: POST /blostem/event, GET /redirect-user
│   │   └── FdOrderController.java .................. @RestController: GET /fixed-deposit/transaction/{id}
│   ├── service/
│   │   ├── BlostemEventService.java ............... Handles Blostem webhook events (payment, VKYC, maturity)
│   │   └── FdOrderService.java .................... Fetches FD order details, order list
│   ├── repository/
│   │   ├── BlostemEventRepository.java ............ JpaRepository: BlostemWebhookEvent
│   │   ├── BlostemUserJourneyRepository.java ...... JpaRepository: BlostemUserJourney (tracks user sessions)
│   │   └── FixedDepositTransactionRepository.java  JpaRepository: FixedDepositTransaction
│   ├── model/
│   │   ├── BlostemEvent.java ....................... Enum: PAYMENT, VKYC, FD_CREATED, MATURITY, etc.
│   │   ├── FixedDepositTransaction.java ........... Entity: FD order tracking
│   │   ├── FixedDepositTransactionStatus.java ..... Enum: PAYMENT_SUCCESSFUL, VKYC_PENDING, FD_BOOKED, etc.
│   │   ├── BlostemWebhookEvent.java ............... Entity: Blostem webhook event log
│   │   ├── BlostemUserJourney.java ................ Entity: User session tracking
│   │   └── BlostemUserJourneyResponse.java ........ Response: JID for user
│   ├── dto/
│   │   ├── BlostemEventEntity.java ................ DTO: parsed webhook payload
│   │   └── FdOrderListResponse.java ............... Response: paginated FD order list
│   └── BlostemErrorCode.java ....................... Enum: error codes (INVALID_JID, AMOUNT_MISMATCH, etc.)
│
├── transaction/ ...................................... Bond and SGB checkout
│   ├── CheckoutTransactionController.java ......... @RestController: POST /order, GET /order/{id}
│   ├── CheckoutTransactionService.java ........... Orchestrates order lifecycle (payment → settlement)
│   ├── model/
│   │   └── CheckoutTransaction.java .............. Entity: order tracking
│   ├── repository/
│   │   └── CheckoutTransactionRepository.java .... JpaRepository: CheckoutTransaction
│   ├── mapper/
│   │   └── TransactionMapper.java ................. MapStruct: CheckoutTransaction → TransactionResponse
│   ├── dto/
│   │   ├── TransactionResponse.java ............... Response: order details
│   │   ├── TransactionRequest.java ................ Request: create order
│   │   └── PaymentStatus.java ..................... Enum: SUCCESS, FAILED
│   ├── util/
│   │   └── OrderStatusConstants.java .............. Lists of status values (BOND_OPEN_ORDERS, FD_OPEN_ORDERS, etc.)
│   ├── config/
│   │   └── EmailNotificationEvent.java ............ Publishes email events (payment success, failure, etc.)
│   ├── PaymentService.java ......................... Payment processing via Razorpay
│   ├── OfferService.java ........................... Fetches offer details via OfferAPI
│   ├── NotificationService.java ................... Sends notifications (email, SMS)
│   └── utils/
│       ├── NotificationHelper.java ................ Utilities for building notification payloads
│       └── OfferHelper.java ........................ Utilities for offer calculations
│
├── external/
│   └── razorpay/
│       ├── RazorpayService.java ................... Integrates with Razorpay payment gateway
│       └── models/
│           └── BankPaymentModes.java ............. DTO: payment modes per bank from Razorpay
│
├── common/
│   ├── domain/
│   │   ├── CheckoutErrorCode.java ................. Enum: error codes (NO_VERIFIED_BANK_ACCOUNTS, etc.)
│   │   └── Constants.java .......................... Application constants (product IDs, etc.)
│   ├── config/
│   │   ├── CheckoutTransactionConfig.java ........ @ConfigurationProperties: transaction limits, ICCL notes
│   │   ├── DiscoveryServiceConfig.java ........... @ConfigurationProperties: Discovery service API key, channel
│   │   ├── B2B2CPrivateApiConfig.java ............ @ConfigurationProperties: B2B2C private API key, channel
│   │   └── FeatureFlagConfig.java ................ @ConfigurationProperties: feature flags (is-bond-basket-enabled, etc.)
│   └── config/
│       └── (Spring Boot auto-config, exception handling, etc.)
│
└── feign/
    └── FeignHttpClientError.java .................. Exception handler for Feign client errors
```

### **Key Design Patterns**

| Pattern | Where Used | Purpose |
|---------|-----------|---------|
| **Async Event Processing** | BlostemEventService, CheckoutTransactionService | Decouples payment processing from settlement |
| **MapStruct Mapper** | TransactionMapper | Type-safe DTO ↔ Entity mapping |
| **Repository Pattern** | CheckoutTransactionRepository, FixedDepositTransactionRepository | Abstract database access |
| **Feign Clients** | PaymentAPI, KycApi, etc. | Declarative HTTP clients for internal services |
| **Configuration Properties** | CheckoutTransactionConfig, DiscoveryServiceConfig | Externalize config, support multiple environments |
| **Error Enum (implements ErrorCode)** | CheckoutErrorCode, BlostemErrorCode | Centralized error code management |
| **Idempotency Check** | BlostemEventRepository.findByJidAndEvent() | Prevent duplicate event processing |
| **Status Enums** | FixedDepositTransactionStatus, OrderStatusConstants | Model state transitions, avoid magic strings |

---

## 7. Security & Access Control

### **Authentication Mechanism**

**Type:** Header-based User Context (no JWT validation in this service; delegated to API Gateway)

**Required Headers for ALL Authenticated Calls:**

| Header | Type | Purpose | Example |
|--------|------|---------|---------|
| `x-user-id` | String (UUID) | Identifies the user making the request | `550e8400-e29b-41d4-a716-446655440000` |
| `channel` | String | Optional; identifies the platform/brand | `yubifin`, `invest`, `aspero` |

**Header Validation:**
- API Gateway validates JWT/token and injects `x-user-id` as trusted header.
- This service **does NOT validate the header** (trusts API Gateway).
- If header is missing, request returns HTTP 400 Bad Request.

**Example Request:**
```bash
curl -X GET http://localhost:8080/checkout/api/v1/banks/payment-methods/v2 \
  -H "x-user-id: 550e8400-e29b-41d4-a716-446655440000" \
  -H "channel: yubifin" \
  -H "Content-Type: application/json"
```

### **RBAC / Permission Matrix**

| Role / User Type | Can Access | Cannot Access | Notes |
|---|---|---|---|
| **Authenticated User** | Own transactions, own bank accounts, own payment methods | Other users' transactions | User ID extracted from header |
| **Admin** | All transactions, all users (if admin endpoint exists) | N/A | Not implemented in current codebase |
| **Service-to-Service** | Publish events to Kafka, call internal APIs | Direct DB access | Uses service-level API keys |

### **Service-to-Service Auth Pattern**

**Internal Service Calls (within YUBI FIN):**
- Uses **Bearer Token** or **API Key** in `Authorization` header.
- Example:
  ```
  Authorization: Bearer {token_from_config}
  ```

**External Service Calls (Razorpay, Blostem, etc.):**
- Razorpay: API key + API secret (configured in Razorpay service).
- Blostem: API key or Bearer token (injected via environment variable).
- Discovery service: `discoverServiceApiKey` (in config).

**Configuration Location:**
```yaml
yubi:
  fin:
    b2c:
      api:
        client:
          internal:
            kyc-service:
              url: ${kycServiceUrl}
          external:
            payment-gateway:
              url: ${paymentGatewayUrl}
```

### **Public vs Protected Endpoints**

**Protected Endpoints (require x-user-id):**
```
GET    /checkout/api/v1/banks
GET    /checkout/api/v1/banks/{bank-code}/payment-methods
GET    /checkout/api/v1/banks/payment-methods
GET    /checkout/api/v1/banks/payment-methods/v2
POST   /checkout/api/v1/order
GET    /checkout/api/v1/order/{id}
GET    /checkout/api/v1/fixed-deposit/transaction/{order-id}
GET    /checkout/api/v1/fixed-deposit/transactions
GET    /checkout/api/v1/blostem/redirect-user
```

**Public/Webhook Endpoints (no x-user-id required):**
```
POST   /checkout/api/v1/blostem/event ............ Blostem webhook (verified via payload signature)
```

### **Token Expiry and Refresh Flow**

**This service does NOT handle token expiry or refresh.**
- JWT validation and refresh are handled by API Gateway / authentication service.
- This service receives only valid, non-expired `x-user-id` from gateway.
- If API Gateway returns 401 Unauthorized, client must re-authenticate.

### **Rate Limiting / Throttling**

**Not explicitly implemented** in this service.
- Rate limiting should be enforced at API Gateway level.
- Razorpay has its own rate limits; retries configured in PaymentService.

### **Secrets Management**

**Storage:** AWS Secrets Manager (via environment variables)

**Required Secrets:**

| Secret | Purpose | Environment | Format |
|--------|---------|-------------|--------|
| `redisPassword` | Redis authentication | All | Plain text password |
| `springDatasourceUsername` | PostgreSQL user | All | Plain text |
| `springDatasourcePassword` | PostgreSQL password | All | Plain text |
| `codeArtifactToken` | AWS CodeArtifact Maven | All | AWS credentials |
| `trustedServerClients` | Allowed service-to-service callers | All | Comma-separated hostnames |
| `emailNotificationKey` | Notification service API key | All | Bearer token |
| `discoverServiceApiKey` | Discovery service API key | All | API key |
| `paymentGatewayUrl` | Razorpay endpoint | All | HTTPS URL |
| `webengageAuthorization` | WebEngage API key | All | Bearer token |
| `auditKafkaUrls` | Audit Kafka broker URLs | Prod/Perf | Comma-separated URLs |
| `b2b2cPrivateApiKey` | B2B2C API key | All | API key |

**Injection Method:**
1. **Local Dev:** `application-local.yml` (hardcoded for testing; never commit real secrets)
2. **K8s / Production:** Mounted as environment variables from Secrets Manager.
3. **Example:**
   ```yaml
   spring:
     datasource:
       password: ${springDatasourcePassword}  # Injected from K8s secret
   ```

**Best Practices:**
- Never commit `.env` or actual secrets to Git.
- Use `.env.example` with placeholder values.
- Rotate secrets regularly (AWS Secrets Manager supports automatic rotation).
- Audit secret access (CloudTrail logs).

---

## 8. Local Setup & Running the Service

### **Prerequisites**

- **Java 17+**: `java -version` should show `openjdk 17.x.x` or higher.
- **Docker & Docker Compose**: For running PostgreSQL, Redis, Kafka.
- **Gradle 8.x**: Build tool (included via `./gradlew`).
- **Git**: Version control.
- **Postman / cURL / HTTPie**: For API testing.

**Install on macOS:**
```bash
brew install openjdk@17
brew install docker
brew install docker-compose
```

**Install on Linux:**
```bash
sudo apt-get install openjdk-17-jdk-headless
sudo apt-get install docker.io docker-compose
```

### **Step 1: Clone & Explore Repository**

```bash
cd ~/projects
git clone https://github.com/yubisecurities/yubi-fin-b2c-checkout-service.git
cd yubi-fin-b2c-checkout-service
git checkout master
```

### **Step 2: Start Dependencies (Docker Compose)**

Create a `docker-compose.yml` in the project root (or use the one provided by DevOps):

```yaml
version: '3.8'
services:
  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: yubi-fin-b2c-kyc
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: root
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

  kafka:
    image: confluentinc/cp-kafka:7.4.0
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
    ports:
      - "9092:9092"
    depends_on:
      - zookeeper

  zookeeper:
    image: confluentinc/cp-zookeeper:7.4.0
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
    ports:
      - "2181:2181"

volumes:
  postgres_data:
```

**Start services:**
```bash
docker-compose up -d
```

**Verify:**
```bash
docker-compose ps
# Should show: postgres, redis, kafka, zookeeper all running
```

### **Step 3: Build the Service**

```bash
# Download dependencies and build JAR
./gradlew clean build

# Or skip tests for faster build (use only for local dev, not for CI/CD)
./gradlew clean build -x test
```

**Expected output:**
```
> Task :build
BUILD SUCCESSFUL in 15s
```

### **Step 4: Set Environment Variables**

Create `~/.env.local` or export directly:

```bash
# Database
export springDatasourceUrl="jdbc:postgresql://localhost:5432/yubi-fin-b2c-kyc"
export springDatasourceUsername="postgres"
export springDatasourcePassword="root"

# Redis
export redisPassword=""
export redisHost="localhost"

# API Keys (use dummy values for local dev)
export discoverServiceApiKey="XYZ"
export emailNotificationKey="Bearer a7c940244208c97f57893b768741924ca09640ec12393ba7c5a4d71cfe2a86d1"
export paymentGatewayUrl="http://localhost:8999"  # Mock server
export kycServiceUrl="http://localhost:8081"
export paymentServiceUrl="http://localhost:8082"
# ... (others)

# Kafka
export auditKafkaUrls="localhost:9092"

# Feature flags
export icclNotesEnabled="true"
```

Or use `application-local.yml` (already configured in repo):
```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/yubi-fin-b2c-kyc
    username: postgres
    password: root
  redis:
    host: localhost
    port: 6379
```

### **Step 5: Run the Service**

**Option A: Using Gradle (recommended)**
```bash
./gradlew bootRunLocal
```

**Option B: Using Java directly**
```bash
java -Dspring.profiles.active=local -jar build/libs/yubi-fin-b2c-checkout-service-0.0.1-SNAPSHOT.jar
```

**Expected output:**
```
2025-01-15 10:23:45.123 INFO  com.yubi.fin.b2c.checkout.CheckoutApplication : Starting CheckoutApplication
...
2025-01-15 10:23:52.456 INFO  org.springframework.boot.web.embedded.tomcat.TomcatWebServer : Tomcat started on port(s): 8080
```

### **Step 6: Verify Service is Running**

```bash
# Health check
curl http://localhost:8080/checkout/api/v1/health

# Swagger UI
open http://localhost:8080/checkout/api/v1/swagger-ui.html
```

### **Required Environment Variables (Full List)**

| Variable | Purpose | Example | Required | Local? |
|----------|---------|---------|----------|--------|
| `spring.profiles.active` | Spring profile (local, dev, qa, prod) | `local` | ✓ | ✓ |
| `springDatasourceUrl` | PostgreSQL connection string | `jdbc:postgresql://localhost:5432/yubi-fin-b2c-kyc` | ✓ | ✓ |
| `springDatasourceUsername` | DB user | `postgres` | ✓ | ✓ |
| `springDatasourcePassword` | DB password | `root` | ✓ | ✓ |
| `redisHost` | Redis server | `localhost` | ✓ | ✓ |
| `redisPassword` | Redis auth password | `` (empty for local) | ✗ | ✓ |
| `discoverServiceApiKey` | Discovery service API key | `XYZ` | ✓ | ✓ (dummy ok) |
| `emailNotificationKey` | Notification service bearer token | `Bearer xxx` | ✓ | ✓ (dummy ok) |
| `paymentGatewayUrl` | Razorpay or payment mock | `http://localhost:8999` | ✓ | ✓ (mock ok) |
| `kycServiceUrl` | KYC service endpoint | `http://localhost:8081` | ✓ | ✓ (mock ok) |
| `codeArtifactToken` | AWS CodeArtifact token | `aws:xxx` | ✓ | ✓ (for build) |
| `icclNotesEnabled` | Feature flag | `true` | ✗ | ✓ |
| `auditKafkaUrls` | Audit Kafka brokers | `localhost:9092` | ✓ | ✓ |
| `trustedServerClients` | Allowed callers (DNS or IPs) | `localhost, 127.0.0.1` | ✓ | ✓ |

### **Local Config Values (Key Properties & Defaults)**

**From `application-local.yml`:**

```yaml
spring:
  jpa:
    show-sql: true
    hibernate:
      ddl-auto: update
  redis:
    database: 0
    host: localhost
    port: 6379

yubi:
  fin:
    b2c:
      checkout:
        transaction:
          max-limit-for-upi: 100000
          iccl-notes-enabled: true
        offer:
          channel: "yubifin"
          sample-user-id: 632092e862ab04005f59ef6b
```

### **Common Local Setup Pitfalls & Solutions**

| Pitfall | Symptom | Solution |
|---------|---------|----------|
| PostgreSQL not running | `Connection refused: 127.0.0.1:5432` | `docker-compose up -d postgres` |
| Wrong database name | `database "checkout" does not exist` | Use `yubi-fin-b2c-kyc` in connection string |
| Redis port conflict | `Address already in use: 6379` | `lsof -i :6379` and kill the process, or use different port |
| Gradle wrapper not executable | `Permission denied: ./gradlew` | `chmod +x gradlew` |
| Java version mismatch | `Unsupported class version 61.0` (Java 17 required for 61.0) | Install Java 17: `brew install openjdk@17` |
| Maven artifacts not found | `Could not find com.yubi.fin.b2c.core.api.common:...` | Ensure AWS CodeArtifact token is valid and has network access |
| Liquibase migration fails | `liquibase migration error` | Check PostgreSQL is running and schema is writable |
| Feign client timeout | `ReadTimeoutException` | Increase timeout in config, or mock the downstream service |

### **How to Access Swagger / API Docs Locally**

Once service is running:

1. **Swagger UI:** http://localhost:8080/checkout/api/v1/swagger-ui.html
   - Interactive API documentation
   - Test endpoints directly from browser

2. **OpenAPI JSON:** http://localhost:8080/checkout/api/v1/v3/api-docs
   - Machine-readable spec

3. **Alternative: Postman**
   - Import the OpenAPI spec into Postman.
   - Create requests with `x-user-id` header.

---

## 9. API Reference

### **Base URL**
```
http://localhost:8080/checkout/api/v1  (local)
https://api.yubifin.com/checkout/api/v1  (production)
```

### **Required Headers for ALL Calls**

```
Content-Type: application/json
x-user-id: {uuid}        # Required for authenticated endpoints
channel: yubifin         # Optional; defaults to yubifin
```

**Example Header:**
```bash
curl -X GET http://localhost:8080/checkout/api/v1/banks \
  -H "Content-Type: application/json" \
  -H "x-user-id: 550e8400-e29b-41d4-a716-446655440000"
```

### **API Versioning Strategy**

- **Current Version:** `/checkout/api/v1`
- **Future Versions:** `/checkout/api/v2`, etc.
- **Breaking Changes:** New major version (v2) introduced; v1 maintained for 6 months.
- **Non-Breaking Changes:** Additive fields allowed in v1; existing fields never removed.

**Example Breaking Change Flow:**
```
v1: GET /banks returns { bankName, accountNumber, ifsc }
v2: GET /banks returns { id, bankName, accountNumber, ifsc, verificationStatus }
     (new field added, old version still available)
```

### **Complete Endpoint Table**

#### **Bank & Payment Methods**

| Method | Endpoint | Auth | Description | Request | Response |
|--------|----------|------|-------------|---------|----------|
| GET | `/banks` | ✓ | Get verified bank accounts for user | Headers: x-user-id | Array of KycItemResponse |
| GET | `/banks/{bank-code}/payment-methods` | ✓ | Applicable payment methods for bank | Path: bank-code, Query: amount (BigDecimal) | Array of PaymentMethod |
| GET | `/banks/payment-methods` | ✓ | Applicable payment methods of user | Headers: x-user-id, Query: amount | PaymentModesResponse |
| GET | `/banks/payment-methods/v2` | ✓ | Enhanced: includes ISIN, channel | Headers: x-user-id, Query: amount, units, isin, channel | PaymentModesResponse |

#### **Fixed Deposit Orders**

| Method | Endpoint | Auth | Description | Request | Response |
|--------|----------|------|-------------|---------|----------|
| GET | `/fixed-deposit/transaction/{order-id}` | ✓ | Get FD order details | Path: order-id (String), Headers: x-user-id | FixedDepositTransaction |
| GET | `/fixed-deposit/transactions` | ✓ | List user's FD orders (paginated) | Headers: x-user-id, Query: page (default 1), size (default 10) | FdOrderListResponse |

#### **Blostem Integration**

| Method | Endpoint | Auth | Description | Request | Response |
|--------|----------|------|-------------|---------|----------|
| GET | `/blostem/redirect-user` | ✓ | Initiate Blostem onboarding | Headers: x-user-id | { "jid": "{uuid}" } |
| POST | `/blostem/event` | ✗ | Receive Blostem webhook event | Body: BlostemEventEntity (JSON) | HTTP 200 OK |

---

### **Sample cURL Requests**

**1. Get Verified Banks**
```bash
curl -X GET http://localhost:8080/checkout/api/v1/banks \
  -H "Content-Type: application/json" \
  -H "x-user-id: 550e8400-e29b-41d4-a716-446655440000"
```

**Response:**
```json
[
  {
    "id": "550e8400-e29b-41d4-a716-446655440001",
    "category": "BANK_ACCOUNT",
    "itemStatus": "APPROVED",
    "itemData": {
      "bankName": "HDFC Bank",
      "accountNumber": "007601563201",
      "ifsc": "HDFC0000001"
    }
  }
]
```

**2. Get Payment Methods for Amount**
```bash
curl -X GET "http://localhost:8080/checkout/api/v1/banks/payment-methods?amount=50000" \
  -H "Content-Type: application/json" \
  -H "x-user-id: 550e8400-e29b-41d4-a716-446655440000"
```

**Response:**
```json
{
  "banks": [
    {
      "bankDetails": {
        "id": "550e8400-e29b-41d4-a716-446655440001",
        "itemData": {
          "bankName": "HDFC Bank",
          "accountNumber": "007601563201",
          "ifsc": "HDFC0000001"
        }
      },
      "paymentModes": [
        { "name": "UPI", "isEnabledForBank": true },
        { "name": "NETBANKING", "isEnabledForBank": true }
      ],
      "isBankEnabled": true
    }
  ],
  "isUPIAllowedForAmount": true,
  "totalBankCount": 1
}
```

**3. Get FD Order Details**
```bash
curl -X GET http://localhost:8080/checkout/api/v1/fixed-deposit/transaction/12345 \
  -H "Content-Type: application/json" \
  -H "x-user-id: 550e8400-e29b-41d4-a716-446655440000"
```

**Response:**
```json
{
  "id": "550e8400-e29b-41d4-a716-446655440002",
  "orderId": 12345,
  "userId": "550e8400-e29b-41d4-a716-446655440000",
  "jid": "550e8400-e29b-41d4-a716-446655440003",
  "status": "FD_BOOKED",
  "amount": 100000,
  "roi": "7.5%",
  "tenure": "12 months",
  "createdOn": "2025-01-15T10:00:00Z",
  "lastUpdatedOn": "2025-01-15T10:15:00Z"
}
```

**4. Initiate Blostem VKYC**
```bash
curl -X GET http://localhost:8080/checkout/api/v1/blostem/redirect-user \
  -H "Content-Type: application/json" \
  -H "x-user-id: 550e8400-e29b-41d4-a716-446655440000"
```

**Response:**
```json
{
  "jid": "550e8400-e29b-41d4-a716-446655440003"
}
```

**5. Receive Blostem Webhook (from Blostem server → this service)**
```bash
curl -X POST http://localhost:8080/checkout/api/v1/blostem/event \
  -H "Content-Type: application/json" \
  -d '{
    "jid": "550e8400-e29b-41d4-a716-446655440003",
    "event": "PAYMENT",
    "type": "FD",
    "data": {
      "isPaymentCompleted": true,
      "amount": "100000",
      "paymentMode": "UPI"
    }
  }'
```

**Response:**
```
HTTP 200 OK
```

---

### **Request Body Structure (Key Fields)**

**CheckoutTransaction Create:**
```json
{
  "productId": "123456",
  "listingId": 789,
  "userId": "550e8400-e29b-41d4-a716-446655440000",
  "amount": 100000,
  "units": 10,
  "paymentMethod": "UPI",
  "bankCode": "HDFC0000001",
  "isin": "IN0987654321"
}
```

**BlostemEventEntity (webhook from Blostem):**
```json
{
  "jid": "550e8400-e29b-41d4-a716-446655440003",
  "event": "PAYMENT",
  "type": "FD",
  "mobileNumber": "9876543210",
  "issuerId": "blostem001",
  "data": {
    "isPaymentCompleted": true,
    "amount": "100000",
    "paymentMode": "UPI"
  }
}
```

---

## 10. How to Extend the System

### **1. Adding a New Document Type or Entity Type**

**Scenario:** Add support for "Commodity Futures" as a new product type.

**Step-by-step:**

1. **Create new entity:**
   ```java
   // src/main/java/com/yubi/fin/b2c/checkout/commodity/model/CommodityTransaction.java
   @Entity
   @Table(name = "commodity_transactions")
   @Data
   public class CommodityTransaction {
       @Id
       private UUID id;
       
       @Column(name = "user_id")
       private UUID userId;
       
       @Column(name = "commodity_id")
       private String commodityId;
       
       @Column(name = "quantity")
       private BigDecimal quantity;
       
       @Column(name = "status")
       @Enumerated(EnumType.STRING)
       private CommodityTransactionStatus status;
       
       @Column(name = "created_on")
       private LocalDateTime createdOn;
   }
   ```

2. **Create status enum:**
   ```java
   // src/main/java/com/yubi/fin/b2c/checkout/commodity/model/CommodityTransactionStatus.java
   public enum CommodityTransactionStatus {
       INITIATED, PAYMENT_PENDING, PAYMENT_SUCCESSFUL, PAYMENT_FAILED, 
       SETTLEMENT_IN_PROGRESS, SETTLED, REJECTED;
   }
   ```

3. **Create repository:**
   ```java
   // src/main/java/com/yubi/fin/b2c/checkout/commodity/repository/CommodityTransactionRepository.java
   @Repository
   public interface CommodityTransactionRepository extends JpaRepository<CommodityTransaction, UUID> {
       Optional<CommodityTransaction> findByIdAndUserId(UUID id, UUID userId);
   }
   ```

4. **Create service:**
   ```java
   // src/main/java/com/yubi/fin/b2c/checkout/commodity/CommodityService.java
   @Service
   @Slf4j
   public class CommodityService {
       @Autowired
       private CommodityTransactionRepository repo;
       
       public CommodityTransaction createOrder(UUID userId, CommodityOrderRequest request) {
           CommodityTransaction transaction = new CommodityTransaction();
           transaction.setId(UUID.randomUUID());
           transaction.setUserId(userId);
           transaction.setCommodityId(request.getCommodityId());
           transaction.setQuantity(request.getQuantity());
           transaction.setStatus(CommodityTransactionStatus.INITIATED);
           transaction.setCreatedOn(LocalDateTime.now());
           return repo.save(transaction);
       }
   }
   ```

5. **Create controller:**
   ```java
   // src/main/java/com/yubi/fin/b2c/checkout/commodity/CommodityController.java
   @RestController
   @RequestMapping("/commodity")
   public class CommodityController {
       @Autowired
       private CommodityService commodityService;
       
       @PostMapping("/order")
       public ResponseEntity<CommodityTransaction> createOrder(
               @RequestHeader("x-user-id") UUID userId,
               @RequestBody CommodityOrderRequest request) {
           return ResponseEntity.ok(commodityService.createOrder(userId, request));
       }
   }
   ```

6. **Add Liquibase migration:**
   ```yaml
   # src/main/resources/db/changelogs/2025-01-15-add-commodity-table.yml
   databaseChangeLog:
     - changeSet:
         id: create-commodity-transactions-table
         author: dev-team
         changes:
           - createTable:
               tableName: commodity_transactions
               columns:
                 - column:
                     name: id
                     type: UUID
                     constraints:
                       primaryKey: true
                 - column:
                     name: user_id
                     type: UUID
                 - column:
                     name: commodity_id
                     type: VARCHAR(50)
                 - column:
                     name: quantity
                     type: DECIMAL(19,2)
                 - column:
                     name: status
                     type: VARCHAR(50)
                 - column:
                     name: created_on
                     type: TIMESTAMP
   ```

---

### **2. Adding a New Async Listener (Kafka)**

**Scenario:** Listen to settlement events from TMS and update commodity order status.

**Step-by-step:**

1. **Create listener class:**
   ```java
   // src/main/java/com/yubi/fin/b2c/checkout/commodity/listener/CommoditySettlementListener.java
   @Component
   @Slf4j
   public class CommoditySettlementListener {
       
       @Autowired
       private CommodityTransactionRepository repo;
       
       @Autowired
       private EmailNotificationEvent notificationEvent;
       
       @ServiceActivator(inputChannel = "commoditySettlementChannel")
       public void handleSettlementEvent(String payload) {
           try {
               log.info("Received settlement event: {}", payload);
               
               // Parse JSON
               CommoditySettlementEvent event = new ObjectMapper()
                   .readValue(payload, CommoditySettlementEvent.class);
               
               // Update order status
               Optional<CommodityTransaction> transaction = 
                   repo.findById(event.getTransactionId());
               
               if (transaction.isPresent()) {
                   CommodityTransaction tx = transaction.get();
                   tx.setStatus(CommodityTransactionStatus.SETTLED);
                   tx.setLastUpdatedOn(LocalDateTime.now());
                   repo.save(tx);
                   
                   // Send confirmation email
                   notificationEvent.sendCommoditySettledEmail(tx);
                   
                   log.info("Commodity transaction settled: {}", tx.getId());
               }
           } catch (Exception e) {
               log.error("Error processing settlement event", e);
               // Send to DLQ
               throw new RuntimeException("Settlement event processing failed", e);
           }
       }
   }
   ```

2. **Configure Kafka binding:**
   ```yaml
   # application-{env}.yml
   spring:
     cloud:
       stream:
         bindings:
           commoditySettlementChannel:
             destination: tms-settlement-topic
             group: checkout-service-commodity-group
             consumer:
               max-attempts: 3
             default-binder: kafka
         kafka:
           binder:
             brokers: ${kafkaServers}
             auto-add-partitions: false
   ```

3. **Add DLQ configuration:**
   ```yaml
   spring:
     cloud:
       stream:
         kafka:
           bindings:
             commoditySettlementChannel:
               consumer:
                 enableDlq: true
                 dlqName: commodity-settlement-dlq
                 dlqPartitions: 1
   ```

4. **Handle DLQ (retry logic):**
   ```java
   @Component
   @Slf4j
   public class CommodityDlqListener {
       
       @ServiceActivator(inputChannel = "commoditySettlementChannel-dlq")
       public void handleDlq(Message<String> message) {
           log.error("Message in DLQ: {}", message.getPayload());
           // Manual retry logic or alerting here
       }
   }
   ```

---

### **3. Adding a New Product / Platform**

**Scenario:** Add support for "Aspero" platform with different feature set (bond-basket disabled).

**Step-by-step:**

1. **Add channel-specific config:**
   ```yaml
   # application-local.yml
   yubi:
     fin:
       b2c:
         channels:
           aspero:
             feature:
               is-bond-basket-enabled: false
               is-upi-enabled: true
               is-commodity-enabled: true
           yubifin:
             feature:
               is-bond-basket-enabled: true
               is-upi-enabled: true
               is-commodity-enabled: false
   ```

2. **Create channel config class:**
   ```java
   @Data
   @Configuration
   public class ChannelFeatureConfig {
       @Value("${yubi.fin.b2c.channels.aspero.feature.is-bond-basket-enabled:false}")
       private boolean asperoBondBasketEnabled;
       
       @Value("${yubi.fin.b2c.channels.yubifin.feature.is-bond-basket-enabled:true}")
       private boolean yubifinBondBasketEnabled;
       
       public boolean isBondBasketEnabled(String channel) {
           if ("aspero".equalsIgnoreCase(channel)) {
               return asperoBondBasketEnabled;
           }
           return yubifinBondBasketEnabled;
       }
   }
   ```

3. **Update checkout service to use channel:**
   ```java
   @Service
   public class CheckoutTransactionService {
       
       @Autowired
       private ChannelFeatureConfig featureConfig;
       
       public void processOrder(CheckoutTransaction transaction, String channel) {
           if (featureConfig.isBondBasketEnabled(channel)) {
               // Process multi-bond basket
               processMultiBondOrder(transaction);
           } else {
               // Process single bond
               processSingleBondOrder(transaction);
           }
       }
   }
   ```

4. **Add migration for channel-specific data:**
   ```sql
   -- db/changelogs/add-channel-column.sql
   ALTER TABLE checkout_transactions ADD COLUMN channel VARCHAR(50) DEFAULT 'yubifin';
   CREATE INDEX idx_channel ON checkout_transactions(channel);
   ```

---

### **4. Adding a New Mandatory Requirement (Config-Only)**

**Scenario:** Require all FD orders to have an emergency contact phone number.

**Step-by-step:**

1. **Update entity:**
   ```java
   @Entity
   public class FixedDepositTransaction {
       // ... existing fields ...
       
       @Column(name = "emergency_contact")
       private String emergencyContact;
   }
   ```

2. **Add Liquibase migration:**
   ```sql
   ALTER TABLE fixed_deposit_transactions ADD COLUMN emergency_contact VARCHAR(20);
   ```

3. **Update request DTO:**
   ```java
   @Data
   public class FdOrderRequest {
       private BigDecimal amount;
       private String tenure;
       private String emergencyContact; // NEW
   }
   ```

4. **Add validation:**
   ```java
   public class FdOrderValidator {
       public static void validate(FdOrderRequest request) {
           if (request.getEmergencyContact() == null || request.getEmergencyContact().isEmpty()) {
               throw CheckoutErrorCode.getApplicationException(
                   CheckoutErrorCode.EMERGENCY_CONTACT_REQUIRED, 
                   HttpStatus.BAD_REQUEST);
           }
           if (!request.getEmergencyContact().matches("^[0-9]{10}$")) {
               throw CheckoutErrorCode.getApplicationException(
                   CheckoutErrorCode.INVALID_PHONE_NUMBER,
                   HttpStatus.BAD_REQUEST);
           }
       }
   }
   ```

5. **Update service:**
   ```java
   @Service
   public class FdOrderService {
       public void createOrder(FdOrderRequest request) {
           FdOrderValidator.validate(request); // Validation
           FixedDepositTransaction tx = new FixedDepositTransaction();
           tx.setEmergencyContact(request.getEmergencyContact());
           repo.save(tx);
       }
   }
   ```

No code change to controller; validation is transparent.

---

### **5. Changing a User-Facing Message or Rejection Reason**

**Scenario:** Change payment failure SMS text.

**Step-by-step:**

1. **Option A: Config-Driven (Recommended)**
   - Update `application-{env}.yml`:
     ```yaml
     yubi:
       fin:
         b2c:
           checkout:
             messages:
               payment-failure-sms: "Your payment failed. Please try again or contact support at 1800-123-4567"
     ```

2. **Option B: Error Code Message**
   - Update error messages in i18n bundle:
     ```properties
     # src/main/resources/locale/message.properties
     err.payment.failure.sms=Your payment failed. Please try again.
     err.payment.failure.email=Your payment of ₹{amount} for {product} failed.
     ```

3. **Use in service:**
   ```java
   @Service
   public class NotificationService {
       @Autowired
       private MessageSource messageSource;
       
       public void sendPaymentFailureSms(CheckoutTransaction tx) {
           String message = messageSource.getMessage(
               "err.payment.failure.sms", 
               null, 
               Locale.US);
           smsProvider.send(tx.getUserPhone(), message);
       }
   }
   ```

No code redeployment needed; update `message.properties` and restart service.

---

### **6. Adding a New Workflow or Workflow Step**

**Scenario:** Add KYC re-verification before settling orders > ₹10 lakhs.

**Step-by-step:**

1. **Create new status enum:**
   ```java
   public enum CheckoutTransactionStatus {
       CREATED, PAYMENT_PENDING, PAYMENT_SUCCESSFUL, PAYMENT_FAILED,
       KYC_RE_VERIFICATION_PENDING, KYC_RE_VERIFICATION_COMPLETED,
       SETTLEMENT_IN_PROGRESS, SETTLED;
   }
   ```

2. **Add workflow logic:**
   ```java
   @Service
   public class CheckoutTransactionService {
       
       private static final BigDecimal KYC_REVERIFICATION_THRESHOLD = new BigDecimal("1000000"); // ₹10L
       
       public void onPaymentSuccess(CheckoutTransaction tx) {
           if (tx.getAmount().compareTo(KYC_REVERIFICATION_THRESHOLD) > 0) {
               // Trigger KYC re-verification
               tx.setStatus(CheckoutTransactionStatus.KYC_RE_VERIFICATION_PENDING);
               triggerKycReverification(tx);
           } else {
               // Direct settlement
               tx.setStatus(CheckoutTransactionStatus.SETTLEMENT_IN_PROGRESS);
               settleOrder(tx);
           }
       }
       
       private void triggerKycReverification(CheckoutTransaction tx) {
           KycReverificationRequest req = new KycReverificationRequest();
           req.setUserId(tx.getUserId());
           req.setTransactionId(tx.getId());
           req.setAmount(tx.getAmount());
           
           kycApi.triggerReverification(req);
           log.info("KYC re-verification triggered for transaction: {}", tx.getId());
       }
   }
   ```

3. **Add listener for KYC completion:**
   ```java
   @Component
   public class KycReverificationListener {
       
       @ServiceActivator(inputChannel = "kycReverificationChannel")
       public void onKycReverificationComplete(String payload) {
           KycReverificationEvent event = parseEvent(payload);
           Optional<CheckoutTransaction> tx = repo.findById(event.getTransactionId());
           
           if (tx.isPresent()) {
               CheckoutTransaction transaction = tx.get();
               transaction.setStatus(CheckoutTransactionStatus.KYC_RE_VERIFICATION_COMPLETED);
               transaction.setLastUpdatedOn(LocalDateTime.now());
               repo.save(transaction);
               
               // Proceed to settlement
               checkoutService.settleOrder(transaction);
           }
       }
   }
   ```

4. **Update migration:**
   ```sql
   ALTER TABLE checkout_transactions ADD COLUMN kyc_reverification_completed_on TIMESTAMP;
   ```

---

### **7. Error Handling Pattern**

**How to throw and format errors:**

```java
// Define error in CheckoutErrorCode enum
public enum CheckoutErrorCode implements ErrorCode {
    EMERGENCY_CONTACT_REQUIRED("err.fd.emergency-contact-required"),
    INVALID_PHONE_NUMBER("err.fd.invalid-phone-number"),
    // ...
}

// Throw in service
@Service
public class FdOrderService {
    public void createOrder(FdOrderRequest request) {
        if (request.getEmergencyContact() == null) {
            throw Errors.builder()
                .errorCode(CheckoutErrorCode.EMERGENCY_CONTACT_REQUIRED)
                .httpStatus(HttpStatus.BAD_REQUEST)
                .args(request.getTenure())  // Optional args for message interpolation
                .applicationError();
        }
    }
}

// Global exception handler (in core-api-common)
@ControllerAdvice
public class GlobalExceptionHandler {
    @ExceptionHandler(ApplicationException.class)
    public ResponseEntity<ApiErrorResponse> handleApplicationException(ApplicationException e) {
        ApiErrorResponse response = new ApiErrorResponse();
        response.setErrorCode(e.getErrorCode());
        response.setMessage(messageSource.getMessage(e.getErrorCode(), e.getArgs(), Locale.US));
        return ResponseEntity.status(e.getHttpStatus()).body(response);
    }
}

// Response to client
HTTP 400 Bad Request
{
  "errorCode": "err.fd.emergency-contact-required",
  "message": "Emergency contact is required to proceed"
}
```

---

### **8. Mapper Pattern (MapStruct)**

**How to add a new mapper:**

```java
// Define mapper interface
@Mapper(unmappedTargetPolicy = ReportingPolicy.IGNORE)
public interface CommodityTransactionMapper {
    CommodityTransactionResponse toResponse(CommodityTransaction transaction);
    
    @Mapping(source = "commodityId", target = "commodity.id")
    CommodityOrderRequest toRequest(CommodityTransaction transaction);
}

// MapStruct generates implementation (CommodityTransactionMapperImpl.java)
// at compile time

// Use in service
@Service
public class CommodityService {
    @Autowired
    private CommodityTransactionMapper mapper;
    
    public CommodityTransactionResponse getTransaction(UUID id) {
        CommodityTransaction tx = repo.findById(id).orElseThrow();
        return mapper.toResponse(tx);  // Auto-mapped
    }
}

// Test the mapper
@Test
public void testToResponse() {
    CommodityTransaction tx = new CommodityTransaction();
    tx.setId(UUID.randomUUID());
    tx.setCommodityId("GOLD");
    
    CommodityTransactionResponse response = new CommodityTransactionMapperImpl().toResponse(tx);
    
    assertEquals(tx.getId(), response.getId());
    assertEquals(tx.getCommodityId(), response.getCommodityId());
}
```

---

### **9. Adding a New Role or Permission**

**Note:** This service does not currently implement role-based access control; all authenticated users have the same access. To add RBAC:

1. **Extend authentication model:**
   ```java
   @Data
   public class UserContext {
       private UUID userId;
       private List<String> roles;  // ["ADMIN", "TRADER", "VIEWER"]
   }
   ```

2. **Add role-based annotations:**
   ```java
   @Component
   public class RoleInterceptor implements HandlerInterceptor {
       @Override
       public boolean preHandle(HttpServletRequest request, HttpServletResponse response, 
                               Object handler) throws Exception {
           String role = request.getHeader("x-user-role");
           SecurityContext.setUserRole(role);
           return true;
       }
   }
   ```

3. **Protect endpoints:**
   ```java
   @RestController
   public class AdminController {
       
       @GetMapping("/admin/transactions")
       @RequiredRole("ADMIN")
       public ResponseEntity<List<CheckoutTransaction>> getAllTransactions() {
           // Only admins can access
           return ResponseEntity.ok(repo.findAll());
       }
   }
   ```

---

## 11. Data Model & Status Rules Engine

### **Core Database Tables**

| Table | Purpose | Key Columns | Indexes |
|-------|---------|-------------|---------|
| `checkout_transactions` | Bond/SGB orders | id (UUID), user_id (UUID), status (VARCHAR), amount (DECIMAL), listing_id (BIGINT), isin (VARCHAR), created_on (TIMESTAMP) | (user_id, status), (user_id, created_on) |
| `fixed_deposit_transactions` | FD orders | id (UUID), user_id (UUID), jid (UUID), status (VARCHAR), amount (DECIMAL), order_id (BIGINT), created_on (TIMESTAMP) | (user_id, status), (jid), (order_id, user_id) |
| `blostem_webhook_events` | Blostem event log | id (UUID), jid (UUID), event (VARCHAR), payload (JSON), created_on (TIMESTAMP) | (jid, event) |
| `blostem_user_journeys` | Blostem session tracking | id (UUID), user_id (UUID), created_on (TIMESTAMP) | (user_id) |

---

### **Key Enums and Their Values**

**CheckoutTransactionStatus:**
```java
CREATED, PAYMENT_PENDING, PAYMENT_SUCCESSFUL, PAYMENT_FAILED,
SETTLEMENT_PENDING, SETTLEMENT_IN_PROGRESS, SETTLED, REFUND_INITIATED, REFUNDED
```

**FixedDepositTransactionStatus:**
```java
PAYMENT_SUCCESSFUL, PAYMENT_FAILED, VKYC_PENDING, VKYC_INITIATED, VKYC_COMPLETED,
IN_PROGRESS, HOLD, REJECTED, DOCUMENT_UPLOAD, CLOSED,
FD_BOOKED, MATURED, REFUND_INITIATED, PREMATURE_WITHDRAW
```

**BlostemEvent:**
```java
PAYMENT, VKYC, REFUND, FD_CREATED, MATURITY, FD_UPDATE, PREMATURE_WITHDRAW, ONBOARDING, ONBOARDING_FAILED
```

**PaymentMethod:**
```java
UPI, NETBANKING, NEFT, IMPS
```

---

### **Status Rules Configuration**

**Orders are considered "open" if status is in:**
```yaml
BOND_OPEN_ORDERS:
  - PAYMENT_PENDING
  - SETTLEMENT_PENDING
  - SETTLEMENT_IN_PROGRESS
  - REFUND_IN_PROGRESS
  - PAID
  - TRADE_REPORTED
  - PAYMENT_SUCCESSFUL

FD_OPEN_ORDERS:
  - PAYMENT_SUCCESSFUL
  - VKYC_PENDING
  - VKYC_INITIATED
  - VKYC_COMPLETED
  - IN_PROGRESS
  - HOLD
  - DOCUMENT_UPLOAD
```

**Orders are considered "settled" if status is in:**
```yaml
BOND_SETTLED_ORDERS:
  - SETTLED
  - EXPIRED
  - REFUNDED
  - FAILED
  - REFUND_INITIATED
  - PAYMENT_FAILED

FD_SETTLED_ORDERS:
  - PAYMENT_FAILED
  - REJECTED
  - CLOSED
  - FD_BOOKED
  - MATURED
  - PREMATURE_WITHDRAW
  - REFUND_INITIATED
```

---

### **How Dynamic Statuses Are Derived**

**Example: Bond Settlement Status**

```java
public String calculateBondSettlementStatus(CheckoutTransaction tx) {
    // Derive from multiple sources:
    
    // 1. Check payment status
    if (tx.getStatus().equals(CheckoutTransactionStatus.PAYMENT_FAILED)) {
        return "PAYMENT_FAILED";
    }
    
    // 2. Check TMS settlement status (call TMS API)
    SettlementStatus tmsStatus = tmsApi.getSettlementStatus(tx.getId());
    if (tmsStatus == null) {
        return "SETTLEMENT_PENDING";
    }
    
    // 3. Check if trade is reported
    if (tmsStatus.isTradeReported() && tmsStatus.isSettled()) {
        return "SETTLED";
    }
    
    // 4. Default
    return tx.getStatus().name();
}
```

---

### **Idempotency Pattern**

**Problem:** Blostem may send duplicate webhook events (e.g., PAYMENT event twice). Without idempotency, the order would be processed twice.

**Solution:** Check if event already processed before updating state.

```java
@Service
public class BlostemEventService {
    
    public void handleEvent(JsonNode payload) {
        BlostemEventEntity eventEntity = objectMapper.convertValue(payload, BlostemEventEntity.class);
        UUID jid = eventEntity.getJid();
        String event = eventEntity.getEvent();
        
        // IDEMPOTENCY CHECK: Has this exact event been processed?
        BlostemWebhookEvent existingEvent = eventRepository.findByJidAndEvent(jid, event);
        if (existingEvent != null) {
            log.info("Duplicate event, ignoring: jid={}, event={}", jid, event);
            return;  // Already processed
        }
        
        // PROCESS EVENT
        FixedDepositTransaction transaction = processPaymentEvent(eventEntity);
        fdTransactionRepository.save(transaction);
        
        // SAVE EVENT LOG
        BlostemWebhookEvent newEvent = new BlostemWebhookEvent();
        newEvent.setId(UUID.randomUUID());
        newEvent.setJid(jid);
        newEvent.setEvent(event);
        newEvent.setPayload(payload.toString());
        newEvent.setCreatedOn(LocalDateTime.now());
        eventRepository.save(newEvent);
    }
}
```

**Database query:**
```sql
SELECT * FROM blostem_webhook_events WHERE jid = ? AND event = ? LIMIT 1;
```

---

### **Caching Strategy**

**What is cached:**

| Data | Key Pattern | TTL | Eviction |
|------|-------------|-----|----------|
| Payment methods per bank | `payment-methods:{bankCode}:{amount}` | 1 hour | LRU |
| User's verified banks | `user-banks:{userId}` | 30 minutes | LRU |
| Razorpay bank modes | `razorpay-bank-modes:{ifscCode}` | 2 hours | LRU |
| Feature flags | `feature-flag:{flagName}` | 5 minutes | LRU |
| Offer details | `offer:{offerId}` | 10 minutes | LRU |

**Implementation (Redis):**
```java
@Service
public class BankService {
    
    @Cacheable(value = "paymentMethods", key = "#bankCode + ':' + #amount", unless = "#result == null")
    public List<PaymentMethod> getPaymentMethods(String bankCode, BigDecimal amount) {
        return paymentAPI.getPaymentMethods(bankCode, amount);
    }
    
    @CacheEvict(value = "paymentMethods", key = "#bankCode + ':' + #amount")
    public void invalidatePaymentMethodsCache(String bankCode, BigDecimal amount) {
        log.info("Cache invalidated for bankCode: {}, amount: {}", bankCode, amount);
    }
}
```

**Configuration:**
```yaml
spring:
  cache:
    type: redis
    redis:
      time-to-live: 3600000  # 1 hour default
```

---

### **Audit Logging**

**Events captured:**

| Event | Fields Logged | Kafka Topic | Retention |
|-------|---------------|-------------|-----------|
| Order Created | user_id, order_id, product_id, amount, timestamp | audit_events | 1 year |
| Payment Success | user_id, order_id, payment_method, amount, transaction_id, timestamp | audit_events | 1 year |
| Payment Failed | user_id, order_id, payment_method, failure_reason, timestamp | audit_events | 1 year |
| VKYC Completed | user_id, fd_id, jid, timestamp | audit_events | 1 year |
| Order Settled | user_id, order_id, settlement_amount, settlement_date, timestamp | audit_events | 1 year |

**Implementation:**
```java
@Service
public class AuditService {
    
    @Autowired
    private KafkaMessageProducer kafkaProducer;
    
    public void logOrderCreated(CheckoutTransaction tx) {
        AuditEvent event = new AuditEvent();
        event.setEventType("ORDER_CREATED");
        event.setUserId(tx.getUserId());
        event.setOrderId(tx.getId());
        event.setProductId(tx.getProductId());
        event.setAmount(tx.getAmount());
        event.setTimestamp(LocalDateTime.now());
        
        kafkaProducer.send("audit_events", event);
    }
}
```

---

## 12. Integration Map

### **Integration Diagram (ASCII)**

```
                     ┌─────────────────────────────┐
                     │   Checkout Service          │
                     └──────────┬──────────────────┘
                                │
                ┌───────────────┼───────────────┐
                │               │               │
         ┌──────▼────────┐ ┌───▼────────┐ ┌───▼──────────┐
         │  KYC Service  │ │ Payment    │ │ User Service │
         │  (Verify ID)  │ │ Service    │ │              │
         └─────────────── │ Wallet)     │ └──────────────┘
                          └────────────┘
                                │
                ┌───────────────┼───────────────┐
                │               │               │
         ┌──────▼────────┐ ┌───▼────────┐ ┌───▼──────────┐
         │  Discovery    │ │ Razorpay   │ │ Notification │
         │  Service      │ │ (Payment)  │ │ Service      │
         │  (Offers)     │ └────────────┘ │ (Email/SMS)  │
         └──────────────┘                 └──────────────┘
                │
         ┌──────▼──────────┐
         │   Blostem       │
         │   (FD Provider) │
         └────────┬────────┘
                  │
         ┌────────▼────────┐
         │  TMS (Trade)    │
         │  Settlement     │
         └─────────────────┘
         
         ┌────────────────┐
         │ WebEngage      │
         │ (Analytics)    │
         └────────────────┘
```

### **Integration Table**

| System | Purpose | Protocol | Direction | Auth Method | Failure Handling |
|--------|---------|----------|-----------|-------------|------------------|
| **KYC Service** | Fetch verified user KYC data | REST / Feign | Outbound | Bearer Token | Retry 3x, then 404 error |
| **Payment Service** | Check wallet balance, record payment | REST / Feign | Outbound | Bearer Token | Retry 3x, then fail order |
| **User Service** | Fetch user profile, email, phone | REST / Feign | Outbound | Bearer Token | Cache fallback, retry 3x |
| **Discovery Service** | Fetch bond/SGB offerings | REST / Feign | Outbound | API Key | Return cached offers or empty list |
| **Offer Service (RFQ)** | Fetch live bond pricing | REST / Feign | Outbound | API Key | Return stale quote or error |
| **Razorpay** | Process UPI/NetBanking payments | REST API | Outbound | API Key + Secret | Webhook callback or poll for status |
| **Notification Service** | Send email/SMS confirmations | REST / Kafka | Outbound | API Key | Queue message, retry async |
| **Blostem** | FD onboarding, VKYC, FD creation | Webhook (inbound) + REST (outbound) | Bi-directional | API Key (signature verification) | Idempotency check, log event, DLQ |
| **TMS** | Record settled trades | Kafka (event-driven) | Outbound | Event signature | DLQ, manual reconciliation |
| **WebEngage** | Track user engagement events | REST API | Outbound | API Key | Fire-and-forget (non-blocking) |
| **PostgreSQL** | Store all persistent data | JDBC | N/A | Connection pooling | Retry with exponential backoff |
| **Redis** | Cache, session storage | TCP | N/A | Password | Failover to secondary node |
| **Kafka** | Async event streaming | Kafka Protocol | Bi-directional | SSL/TLS | Consumer groups, DLQ |

---

## 13. Operational Runbook

### **Common Failure Modes & Resolution**

| Symptom | Likely Cause | Detection | Resolution |
|---------|-------------|-----------|-----------|
| **Payment success but order not created** | Async listener missed event or crashed | Check order table for missing entries; check Kafka DLQ | Manual order creation via admin; replay Kafka events from DLQ |
| **VKYC stuck in PENDING** | Blostem webhook not delivered or service down | Check blostem_webhook_events table; no VKYC_COMPLETED event | Restart Blostem webhook delivery; manual status update if verified |
| **User sees "No payment methods available"** | No verified banks OR Razorpay integration down | Check bank account count; ping Razorpay API | Instruct user to add bank; check Razorpay service status |
| **Payment retries exhausted** | Network timeout, insufficient balance, or invalid card | Check payment_attempts table; last error message | Retry with different payment method; user adds funds |
| **Orders not settling (stuck in SETTLEMENT_IN_PROGRESS)** | TMS service down or not consuming Kafka events | Check TMS service health; check Kafka consumer lag | Restart TMS; manually trigger settlement via TMS API |
| **Email notifications not sent** | Notification service down; invalid email template | Check email_sent_status in DB; check notification service logs | Update email template; restart notification service |
| **Redis cache stale data** | TTL not set or eviction misconfigured | Redis memory usage high; cache hit ratio low | Manually flush cache key; adjust TTL / eviction policy |
| **Database connection pool exhausted** | High concurrency OR connection leak | Check active connections; query logs show wait time | Increase pool size; kill idle connections; restart service |

---

### **How to Check DLQ / Retry Queue State**

**Kafka DLQ:**
```bash
# List messages in DLQ
kafka-console-consumer.sh \
  --bootstrap-servers localhost:9092 \
  --topic checkout-settlement-dlq \
  --from-beginning \
  --max-messages 10

# Check consumer lag
kafka-consumer-groups.sh \
  --bootstrap-servers localhost:9092 \
  --group checkout-service-commodity-group \
  --describe
```

**Database retry table:**
```sql
-- Check failed payment attempts
SELECT id, order_id, user_id, status, error_message, attempt_count, last_attempted_at
FROM checkout_transactions
WHERE status = 'PAYMENT_FAILED'
AND attempt_count < 3
ORDER BY last_attempted_at ASC
LIMIT 20;

-- Reprocess failed orders
UPDATE checkout_transactions
SET status = 'PAYMENT_PENDING', attempt_count = attempt_count + 1
WHERE id IN ('order-uuid-1', 'order-uuid-2');
```

---

### **How to Manually Trigger or Unstick a Stuck Workflow**

**Unstick stuck FD VKYC:**
```java
// In admin controller
@PostMapping("/admin/fd/{fdId}/force-vkyc-complete")
@RequiredRole("ADMIN")
public ResponseEntity<?> forceVkycComplete(@PathVariable UUID fdId) {
    FixedDepositTransaction tx = repo.findById(fdId).orElseThrow();
    tx.setStatus(FixedDepositTransactionStatus.VKYC_COMPLETED);
    tx.setLastUpdatedOn(LocalDateTime.now());
    repo.save(tx);
    
    // Trigger settlement
    checkoutService.settleOrder(tx);
    
    return ResponseEntity.ok("VKYC marked complete and settlement triggered");
}
```

**Manual order settlement:**
```java
@PostMapping("/admin/order/{orderId}/settle")
@RequiredRole("ADMIN")
public ResponseEntity<?> manualSettle(@PathVariable UUID orderId) {
    CheckoutTransaction tx = repo.findById(orderId).orElseThrow();
    
    // Call TMS to record trade
    tmsApi.recordTrade(tx.toTmsRequest());
    
    tx.setStatus(CheckoutTransactionStatus.SETTLED);
    tx.setLastUpdatedOn(LocalDateTime.now());
    repo.save(tx);
    
    // Send notification
    notificationService.sendSettlementConfirmation(tx);
    
    return ResponseEntity.ok("Order manually settled");
}
```

---

### **Key Log Queries / Kibana / Datadog**

**Find failed orders:**
```
status: PAYMENT_FAILED AND created_at: [now-1d TO now]
```

**Find Blostem webhook errors:**
```
blostem_event_service AND (ERROR OR exception)
```

**Track order lifecycle:**
```
order_id: "550e8400-e29b-41d4-a716-446655440001"
```

**High latency queries:**
```
service: checkout-service AND response_time_ms > 1000
```

---

### **Rollback Procedure**

**Rollback a deployment:**

1. **Identify version to rollback to:**
   ```bash
   kubectl rollout history deployment/checkout-service -n yubi-fin
   # Outputs: REVISION  CHANGE-CAUSE
   #         3         image updated to v1.2.5
   #         2         image updated to v1.2.4
   ```

2. **Rollback to previous version:**
   ```bash
   kubectl rollout undo deployment/checkout-service -n yubi-fin --to-revision=2
   ```

3. **Verify rollback:**
   ```bash
   kubectl rollout status deployment/checkout-service -n yubi-fin
   ```

**Rollback database migrations (Liquibase):**

```sql
-- Identify changeset to rollback
SELECT * FROM databasechangelog WHERE dateexecuted > now() - interval '1 hour' ORDER BY dateexecuted DESC;

-- Rollback manually (Liquibase doesn't auto-rollback; must implement changeSet rollback tag)
-- OR restore from backup
```

---

## 14. Testing, Deployment & Infrastructure

### **How to Run Tests Locally**

```bash
# Run all tests
./gradlew test

# Run specific test class
./gradlew test --tests BankServiceTest

# Run tests with coverage report
./gradlew test jacocoTestReport

# View coverage report
open build/reports/jacoco/test/html/index.html

# Run integration tests (requires services running)
./gradlew test --tests "*IT"
```

---

### **Test Pattern Example (BankServiceTest)**

```java
@ExtendWith(MockitoExtension.class)
class BankServiceTest {
    
    @InjectMocks
    private BankService bankService;
    
    @Mock
    private PaymentAPI paymentAPI;
    
    @Mock
    private KycApi kycApi;
    
    @Mock
    private RazorpayService razorpayService;
    
    @Test
    void shouldGetVerifiedBankAccountsForUser() {
        // ARRANGE
        String userId = "user123";
        List<BankAccount> expected = Arrays.asList(
            BankAccount.builder()
                .bankName("HDFC Bank")
                .accountNumber("007601563201")
                .ifscCode("HDFC0000001")
                .build()
        );
        
        KycResponse kycResponse = KycResponse.builder()
            .productId(Constants.YUBIFIN_PRODUCT_ID)
            .kycItems(/* ... */)
            .build();
        
        when(kycApi.executeFetchKycItemData(anyString(), eq(userId), anyString(), anyBoolean()))
            .thenReturn(ResponseEntity.ok(kycResponse));
        
        // ACT
        List<BankAccount> actual = bankService.getVerifiedBankAccounts(userId);
        
        // ASSERT
        assertEquals(expected.size(), actual.size());
        assertEquals(expected.get(0).getBankName(), actual.get(0).getBankName());
    }
    
    @Test
    void shouldThrowExceptionWhenNoBankAccountsFound() {
        // ARRANGE
        String userId = "user123";
        KycResponse kycResponse = KycResponse.builder()
            .productId(Constants.YUBIFIN_PRODUCT_ID)
            .kycItems(Collections.emptyList())
            .build();
        
        when(kycApi.executeFetchKycItemData(anyString(), eq(userId), anyString(), anyBoolean()))
            .thenReturn(ResponseEntity.ok(kycResponse));
        
        // ACT & ASSERT
        assertThrows(ApplicationException.class, () -> bankService.getVerifiedBankAccounts(userId));
    }
}
```

---

### **Environment Matrix**

| Environment | Replicas | Resources (CPU / Memory) | Purpose | Backup | Retention |
|-------------|----------|--------------------------|---------|--------|-----------|
| **Local** | 1 | 1 CPU / 2 GB | Development | Manual | N/A |
| **Dev** | 1 | 0.5 CPU / 1 GB | Developer testing | Daily | 7 days |
| **QA** | 2 | 1 CPU / 2 GB | QA testing, automation | Daily | 14 days |
| **Perf** | 3 | 2 CPU / 4 GB | Performance testing | Daily | 7 days |
| **PT** | 2 | 1 CPU / 2 GB | Partner testing | Daily | 14 days |
| **UAT** | 2 | 1 CPU / 2 GB | User acceptance testing | Daily | 30 days |
| **Prod** | 5 | 2 CPU / 4 GB | Production | Hourly | 1 year |

---

### **CI/CD Pipeline Diagram (ASCII)**

```
    ┌─────────────┐
    │ Git Push    │
    │ (master)    │
    └──────┬──────┘
           │
           ▼
    ┌─────────────────────┐
    │ Trigger Jenkins Job │
    │ build-lower-envs    │
    └──────┬──────────────┘
           │
     ┌─────┴─────┐
     │           │
     ▼           ▼
  ┌──────┐   ┌──────┐
  │Lint  │   │Build │
  │Check │   │JAR   │
  └──┬───┘   └──┬───┘
     │          │
     └────┬─────┘
          │
          ▼
   ┌─────────────┐
   │  SonarQube  │
   │  Analysis   │
   └──────┬──────┘
          │
          ▼
   ┌──────────────┐
   │  Build Docker│
   │   Image      │
   └──────┬───────┘
          │
          ▼
   ┌──────────────────┐
   │ Push to ECR      │
   │ (AWS Container   │
   │  Registry)       │
   └────────┬─────────┘
            │
     ┌──────┼──────┐
     │      │      │
     ▼      ▼      ▼
  ┌─────┬─────┬─────┐
  │ Dev │ QA  │ PT  │  (via ArgoCD)
  └─────┴─────┴─────┘
     │      │      │
     └──┬───┴──┬───┘
        │      │
        ▼      ▼
  ┌────────────────┐
  │ Approval Gate  │
  │ (Manual)       │
  └────────┬───────┘
           │
           ▼
  ┌──────────────┐
  │  Production  │  (via ArgoCD)
  └──────────────┘
```

---

### **Related Repositories**

| Repository | Role | Language | Dependency Type |
|------------|------|----------|-----------------|
| **yubi-fin-b2c-core-api-common** | Shared libs (exceptions, models, configs) | Java | Maven (upstream) |
| **yubi-fin-b2c-core-api-client** | HTTP clients for internal services | Java | Maven (upstream) |
| **yubi-fin-b2c-core-integration** | Kafka, audit, event publishing | Java | Maven (upstream) |
| **yubi-fin-b2c-kyc-service** | KYC verification | Java | Internal service |
| **yubi-fin-b2c-payment-service** | Payment wallet & ledger | Java | Internal service |
| **yubi-fin-b2c-user-service** | User profile management | Java | Internal service |
| **yubi-fin-b2c-notification-service** | Email / SMS sending | Java | Internal service |
| **yubi-fin-b2c-offer-service** | Product offerings & pricing | Java | Internal service |
| **yubi-fin-b2c-discovery-service** | Product catalog | Java | Internal service |
| **yubi-fin-b2c-tms-service** | Trade settlement | Java | Internal service |
| **yubi-fin-b2c-platform** | Frontend web app | React / TypeScript | Calls checkout-service API |
| **yubi-fin-b2c-mobile-app** | Frontend mobile app | React Native / Flutter | Calls checkout-service API |

---

### **Known Gaps and Improvement Areas**

| Gap | Impact | Owner | Status |
|-----|--------|-------|--------|
| **CSPL Migration Underway** | Service architecture/branding may change; DB schema uncertain | Team | In Progress |
| **No admin endpoints** | Cannot manually manage transactions; ops requires DB access | Ops Team | Backlog |
| **No rate limiting** | DDoS/API abuse not mitigated at service level | Platform | Backlog |
| **No request signing** | Service-to-service calls not cryptographically verified | Security | Backlog |
| **Limited RBAC** | All authenticated users have same permissions; no role-based access | Product | Backlog |
| **Hardcoded UPI limit** | Feature flag exists but threshold still hardcoded in BankService | Dev Team | Backlog |
| **No distributed tracing** | Cannot trace requests across services; debugging is hard | DevOps | Backlog |
| **No graceful degradation** | If Razorpay is down, entire checkout fails (no fallback) | Product | Future |
| **Manual idempotency check** | Risk of duplicate processing if check skipped | Dev Team | Backlog |

---

### **Known Developer Gotchas**

| Behavior | Explanation | Workaround |
|----------|-------------|-----------|
| **Soft deletes** | `isDeleted` field on entities; queries must filter out deleted records | Always use custom repository methods that exclude soft-deleted; never query raw with just ID |
| **Async status updates** | Order status changes asynchronously via Kafka; frontend must poll | Poll status endpoint every 2–5 seconds; or use WebSocket for real-time updates |
| **Bank code lookups** | Razorpay requires exact IFSC code; different from bank name | Map bank name to IFSC in KYC data; never guess IFSC |
| **Amount precision** | Use `BigDecimal` for amounts; `Double` causes rounding errors | Always use `new BigDecimal("100.00")`; never `100.0` |
| **UUID vs Long IDs** | User ID is UUID; order ID can be Long; FD JID is UUID | Don't mix types; validate in tests |
| **Redis cache keys** | Keys can collide if not namespaced; old keys not auto-expired | Use prefixes like `payment-methods:{bankCode}:{amount}`; monitor Redis memory |
| **Kafka consumer offset** | If service crashes, offset might not be committed; events reprocessed | Implement idempotency check (already done for Blostem) |
| **Feign timeout** | Default timeout is very short (1s); can fail on slow downstream services | Increase timeout in Feign config; or implement circuit breaker |
| **Transaction isolation** | Database uses default isolation level; concurrent updates possible | Use pessimistic locks or explicit transaction boundaries where needed |
| **Email templates** | Hardcoded template names in code; changing requires deploy | Move template names to config file |

---

> **Note:** This documentation is auto-generated from code analysis. Verify critical sections with the team before deploying to production.