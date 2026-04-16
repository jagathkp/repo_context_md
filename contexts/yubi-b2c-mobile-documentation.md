# Aspero B2C Mobile & Web — Master Context

> **Repository**: yubisecurities/yubi-b2c-mobile  
> **Default Branch**: main  
> **Stack**: React Native 0.80, React 19, TypeScript, Next.js (web), React Navigation v6, Hookstate, Firebase, Axios  
> **Documentation**: CLAUDE.md (project-specific guidance)  
> **Owner**: Yubi Securities team

---

## TL;DR (30-second read)

- **What**: A React Native cross-platform (iOS/Android/Web) retail bonds/securities investment application ("Aspero Fin")
- **Primary Input**: User authentication, KYC verification, bond/FD product exploration and selection
- **Primary Output**: Investment orders placed, portfolio tracked, KYC status managed
- **Status**: Production (active on iOS, Android, and web at invest.aspero.co.in)
- **Owner/Slack**: Aspero / Yubi Securities product team

---

## Who Should Read This

| Audience | Key Sections |
|----------|--------------|
| **Product Manager** | TL;DR, Glossary, User Journey, Conditional Flows, Notifications, Integration Map |
| **Frontend Developer (new to repo)** | Architecture, Local Setup, API Reference, How to Extend, Data Model, Testing |
| **DevOps / Infra** | CI/CD, Deployment, Infrastructure, Environments |
| **QA / Tester** | User Journey, Workflows, Testing, Common Gotchas |
| **Technical Lead** | All sections |

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
15. Known Gotchas & Developer Pitfalls

---

## 1. Glossary

| Term | Definition |
|------|-----------|
| **Aspero** | Yubi's retail investment platform brand; "Fin" is the internal app name |
| **B2C** | Business-to-Consumer; retail customer facing |
| **KYC** | Know Your Customer — regulatory compliance process; includes PAN, Aadhaar, address verification |
| **Demat** | Dematerialised securities account; required to hold bonds/stocks; holds BO account |
| **BO Account** | Beneficial Owner account at NSDL/CDSL; linked to demat |
| **Bond** | Debt security; core product; has ISIN, issuer, tenor, rating, coupon |
| **ISIN** | International Securities Identification Number; unique bond identifier |
| **FD** | Fixed Deposit; recurring product; linked to partner bank |
| **SGB** | Sovereign Gold Bond; government-backed gold investment; feature-flagged |
| **Portfolio** | User's holdings; bonds and FDs owned |
| **Order** | Purchase request for a bond or FD; moves through payment → settlement |
| **Watchlist** | User-saved bonds for later purchase |
| **2FA / PIN** | Two-factor authentication; user sets secure PIN after login for sensitive operations |
| **Remote Config** | Firebase Remote Config; runtime feature flags (e.g., enable_bond_basket, enable_sgb) |
| **Deeplink** | URL scheme to route to specific screens; supported via Branch SDK, AppsFlyer, custom schemes |
| **SEBI** | Securities and Exchange Board of India; regulatory body |
| **RBI** | Reserve Bank of India; central bank; regulates FDs |
| **SSO Partner** | Third-party customer acquiring partner (e.g., Google OAuth, corporate SSO) via JWT validation |
| **Tenant / Domain** | invest.aspero.co.in is primary domain; may support multi-tenant via domain cookies |
| **Environment** | Deployment target: development, qa, uat, production (GA) |

---

## 2. What Is This Service?

### One-Line Purpose

A cross-platform mobile and web application enabling Indian retail investors to discover, evaluate, and purchase bonds, fixed deposits, and sovereign gold bonds with full regulatory compliance (KYC) and portfolio tracking.

### Why It Exists

**Regulatory**: SEBI requires KYC before trading securities. **Business**: Yubi wants direct B2C distribution; Aspero is the brand. **Technical**: Unified codebase reduces maintenance across iOS, Android, web.

### What It Does NOT Do

- **Not a broker**: Does not provide brokerage/advisory services beyond product info. **Not a bank**: FDs are partner-provided. **Not international**: India-only (NSDL/CDSL demat, SEBI regulated). **Not real-time trading**: Bonds are primary instruments; limited order types. **Not wealth management**: No portfolio rebalancing or advisory algo.

### Value to the Business

- **Revenue**: Transaction fees on order placement. **Customer Acquisition**: Direct retail channel; brand awareness. **Compliance**: Automated KYC reduces manual overhead. **Data**: User profiling → product recommendations.

---

## 3. Who Uses It and How

| Stakeholder | How They Interact | What They Need |
|-------------|------------------|----------------|
| **Retail Investor** | Download app / visit web → Sign in → Complete KYC → Explore bonds → Place order → Track portfolio | Easy discovery, clear pricing, trust (SEBI compliance, ratings) |
| **SSO Partner** (Google, corporate) | Sends JWT with customer email/phone → App validates → Auto-creates account | Seamless login, no re-KYC if pre-existing |
| **Internal Admin** | Views KYC statuses, manually approves/rejects suspicious profiles | Dashboard (Freshdesk tickets, admin endpoints) |
| **Fintech Ops** | Monitors order settlement, payment failures, FD funding | Observability (Kfuse alerts, Datadog dashboards) |
| **Customer Support** | Handles escalations: payment issues, KYC rejection, order status | Support interface (Freshdesk integration, chat widget Zoho) |

### Supported Product Types / Platforms

| Platform | Product Types | Status |
|----------|--------------|--------|
| **iOS** | Bonds, FD, SGB (feature-flagged) | Production |
| **Android** | Bonds, FD, SGB (feature-flagged) | Production |
| **Web (invest.aspero.co.in)** | Bonds, FD, SGB (feature-flagged) | Production |
| **Mobile Web (responsive)** | Bonds, FD, SGB (feature-flagged) | Production |

---

## 4. The User Journey — Plain English

### Journey 1: New Investor Sign-Up → First Bond Purchase

**Step 1: Authentication**
- User opens app or visits invest.aspero.co.in
- Chooses "Sign In" → selects phone or email
- Enters phone/email → receives OTP → verifies
- System checks if account exists; if not, creates one
- User sets secure PIN (2FA)
- **System**: Auth API creates session, stores PIN hash, issues JWT

**Step 2: KYC Verification (Regulatory Gate)**
- Dashboard banner shows "KYC Incomplete"
- User taps "Start KYC" → wizard opens
- **Step 2a: PAN & Aadhaar**
  - Enters PAN number
  - Captures/uploads Aadhaar via camera or DigiLocker (OAuth)
  - System verifies PAN → retrieves name/DOB from CKYC
  - **System**: Calls CKYC (PAN verification service) → verifies Aadhaar
- **Step 2b: Address Proof**
  - Uploads address document (Aadhaar, Driving License, Passport, Voter ID)
  - Crops/validates image quality
  - **System**: Manual review or automated checks (size, clarity)
- **Step 2c: Bank Account**
  - Enters bank name, account number, IFSC
  - User sets as primary or secondary
  - **System**: Validates IFSC, stores bank details (encrypted)
- **Step 2d: Demat Account (if applicable)**
  - Enters DP ID, Client ID
  - **System**: Links to NSDL/CDSL BO account (verified at order time)
- **Step 2e: Risk Profiling (KRA)**
  - Quiz: risk appetite, investment horizon, income level
  - System calculates risk score → recommends bond categories
- **Step 2f: E-Signature / Wet Signature**
  - Signs T&Cs via e-sign (if NSDL DDPI enabled)
  - Fallback: manual verification
  - **System**: Submits to backend → marks KYC "IN_REVIEW"
- **Step 3: Regulatory Approval**
  - Backend runs AML/KRA checks
  - SEBI compliance automation
  - User sees banner: "KYC Under Review" (2–24 hours typical)
  - **System Event**: On approval → sends notification → marks KYC "APPROVED"

**Step 4: Product Discovery**
- User browses "Explore" tab → sees curated bond lists (by sector, tenor, rating)
- Taps a bond → views:
  - Issuer details, rating, coupon, tenure, maturity date
  - Returns calculator (manual input → shows expected profit)
  - Risks & benefits
  - Similar bonds from same issuer
- Adds to watchlist or proceeds to invest

**Step 5: Payment & Order Placement**
- User enters investment amount
- App shows fees (0% – Aspero's value prop)
- Selects payment method:
  - **Bank Transfer**: Virtual Account number displayed → user transfers from their bank → settlement in 2–3 hours
  - **Razorpay UPI/Card**: Direct debit (reserved)
- Payment gateway processes → awaits settlement
- **System**: Creates order, polls settlement status, updates portfolio on success

**Step 6: Order Confirmation**
- Order status: "Pending" → "Settlement In Progress" → "Confirmed"
- Bond appears in Portfolio tab
- Notification: "Investment Successful"

### Journey 2: Returning User — Check Portfolio & Maturity

- User opens Portfolio tab
- Sees holdings: current value, unrealised gains/losses, maturity date
- Taps a bond → sees maturity schedule, repayment details
- At maturity: notification → funds auto-credited to linked bank account

### Journey 3: SSO Partner (Google/Corporate)

- Partner redirects user to app with JWT
- App validates JWT signature
- If user exists: logs in automatically
- If new: auto-creates account, skips email verification, but still requires KYC
- User proceeds to KYC Journey

### Alternative Path: Fast-Track KYC via DigiLocker

- Instead of manual document upload, user grants Aadhaar + DigiLocker OAuth access
- System retrieves verified documents → auto-populates
- Speeds up KYC to 5 minutes
- Still requires backend approval for AML/KRA

---

## 5. Conditional Flows — Which Workflow Applies to Whom

### Routing Dimensions

```
User Type (Individual / HUF)
         ↓
   KYC Status (Not Started / Submitted / Approved / Rejected)
         ↓
   Product Type (Bond / FD / SGB)
         ↓
   Feature Flag State (enable_bond_basket, enable_sgb, enable_fd_widget, etc.)
         ↓
   Environment (dev / qa / uat / production)
```

### Feature Availability Matrix

| Dimension | Individual | HUF | Notes |
|-----------|-----------|-----|-------|
| **Bonds** | ✓ | ✓ (via custodian) | Core product |
| **FD** | ✓ (enable_fd_widget flag) | ✓ | Partner-provided |
| **SGB** | ✓ (enable_sgb flag) | ✓ (enable_sgb flag) | Feature-flagged |
| **Demat Link** | ✓ (dematFeatureActive flag) | ✓ | Optional; gated for BO linkage |
| **KYC Steps** | Standard: PAN, Aadhaar, Address, Bank | Extended: Guardianship proof required | HUF = Hindu Undivided Family |

### Product Type Routing (Bonds)

```
Bond Discovery
  ├─ All Bonds (explore_search)
  ├─ Curated Collections
  │   ├─ By Sector (Automobile, Banking, etc.)
  │   ├─ By Rating (AAA, AA, etc.)
  │   └─ By Tenor (Short, Long)
  ├─ Issuer Page (all bonds from one issuer)
  └─ Bond Basket (pre-configured diversified sets; if enabled)
```

### KYC Status Rules

```
KYC_PENDING
  ├─ Show banner: "Complete KYC to invest"
  └─ Disable order placement

KYC_SUBMITTED
  ├─ Show banner: "KYC Under Review (24 hours)"
  └─ Disable order placement

KYC_APPROVED
  ├─ Remove banner
  ├─ Enable order placement
  └─ Show portfolio

KYC_REJECTED
  ├─ Show error message + rejection reason
  ├─ Offer "Re-submit" option
  └─ Disable order placement until re-submitted

AML_FAILED / KRA_FAILED
  ├─ Block account (soft delete possible)
  └─ Show message: "Your account has been restricted. Please contact support."
```

### Config-Only Changes (No Code Deploy)

| Config Key | Type | Effect | Managed Via |
|-----------|------|--------|------------|
| `enable_bond_basket` | Boolean | Show/hide bond basket products | Firebase Remote Config |
| `enable_sgb` | Boolean | Show/hide SGB product tab | Firebase Remote Config |
| `enable_fd_widget` | JSON | Show/hide FD; can disable per environment | Firebase Remote Config |
| `enable_promotional_banner` | Boolean | Show/hide promotional cards on home | Firebase Remote Config |
| `enable_payments` | Boolean | Disable all payment flows (e.g., maintenance) | Firebase Remote Config |
| `enable_kra_check` | Boolean | Require KRA profiling before order | Firebase Remote Config |
| `enableRequestCallback` | Boolean | Show "Request Callback" in support | Firebase Remote Config |
| `activeChatBot` | Enum (ZOHO / NONE) | Which chat widget to load | Firebase Remote Config |
| Product pricing / fees | JSON | Update bond fees, fund charge | CMS (Strapi) |
| Feature gate per user type | JSON | Restrict features to certain user segments | Backend API (feature flags) |

### Code-Required Changes

| Change Type | Example | Reason |
|------------|---------|--------|
| New product type | Add new tab for crypto (if ever) | Navigation structure, state management |
| New KYC document | NREGA for rural verification | DB schema, workflow steps |
| New payment method | Credit card via new provider | API integration, security review |
| New user type | Institutional investors | Role-based access, compliance logic |
| New integration | Razorpay → Stripe | HTTP client, error handling, reconciliation |

### Notification Triggers

| Event | Who | Channel | Condition |
|-------|-----|---------|-----------|
| **Sign-Up OTP** | User | SMS | On new account creation |
| **KYC Approved** | User | In-app + Email | After regulatory approval |
| **KYC Rejected** | User | In-app + Email + SMS | Failed AML or manual review |
| **Order Confirmation** | User | In-app + Email + SMS | Order status → "CONFIRMED" |
| **Payment Pending** | User | In-app banner | Order awaiting settlement |
| **Bond Matured** | User | In-app + Email + SMS | Maturity date reached |
| **New Bond Listed** | User | In-app + Email | New product matches user's profile (if subscribed) |
| **Watchlist Bond Price Drop** | User | In-app + Push | If feature enabled |
| **Order Stuck** | Ops | Slack / Kfuse Alert | Order not settled after 24 hours |
| **KYC Submission Failure** | Ops | Slack | Backend KYC API call failed |

---

## 6. Architecture & Code Structure

### High-Level Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                        CLIENT LAYER                             │
├─────────────────┬──────────────────┬──────────────────────┐
│   React Native  │   React Native   │   Next.js Web       │
│   (iOS)         │   (Android)      │   + Webpack         │
│   FinGA/FinQA   │   (Gradle)       │   invest.aspero... │
└────────┬────────┴────────┬─────────┴──────────┬──────────┘
         │ React Navigation v6 Stack Navigation │
         │       (useRoute, useNavigation)      │
         │          Deeplink Manager            │
         └────────────┬────────────────────────┘
                      │
        ┌─────────────────────────────────┐
        │   STATE MANAGEMENT (Hookstate)  │
        │  • authStore (JWT, PIN)         │
        │  • profileStore (user data)     │
        │  • kycStore (KYC status)        │
        │  • remoteConfigStore (flags)    │
        │  • modalStackStore (UI state)   │
        └────────────┬────────────────────┘
                     │
        ┌────────────────────────────────┐
        │      API CLIENT (Axios)        │
        │  • APIClient.get/post/put      │
        │  • Interceptors (auth refresh, │
        │    403 PIN expiry, error log)  │
        │  • Headers: JWT, Device Info   │
        └────────────┬───────────────────┘
                     │
        ┌────────────────────────────────┐
        │    BACKEND SERVICES            │
        │  • Auth API (login, logout)    │
        │  • KYC API (submission)        │
        │  • Bond API (search, detail)   │
        │  • Order API (place, status)   │
        │  • Portfolio API (holdings)    │
        │  • Payment API (Razorpay)      │
        │  • FD API (NBFC partner)       │
        └────────────┬───────────────────┘
                     │
        ┌────────────────────────────────┐
        │  EXTERNAL SERVICES             │
        │  • Firebase (auth, config)     │
        │  • NSDL/CDSL (demat)           │
        │  • Razorpay (payments)         │
        │  • CKYC (PAN verification)     │
        │  • AML/KRA (compliance)        │
        │  • CMS Strapi (content)        │
        │  • Analytics (Amplitude, FB)   │
        │  • Chat Widget (Zoho)          │
        └────────────────────────────────┘
```

### Request Lifecycle (Synchronous)

```
User taps "Invest Now"
  ↓
Screen component calls: apiClient.post({url: '/order/place', data: {...}})
  ↓
Axios Request Interceptor:
  ├─ Adds Authorization: Bearer {JWT}
  ├─ Adds Device-Info header
  └─ Logs request (analytics)
  ↓
Backend receives request → validates JWT → processes
  ↓
Response 200 (success)
  ├─ Response Interceptor (no special action)
  └─ Component updates state: showSuccessToast()
  ↓
Response 401 (unauthorized)
  ├─ Response Interceptor detects 401
  ├─ Calls refreshToken() endpoint
  ├─ Gets new JWT
  ├─ Retries original request with new JWT
  └─ If refresh fails → logout (redirect to Sign In)
  ↓
Response 403 (PIN expired)
  ├─ Response Interceptor detects 403
  ├─ Debounces: shows "Verify PIN" modal
  └─ On close: navigates to verify_secure_pin_post_login screen
  ↓
Response 4xx/5xx (error)
  ├─ Response Interceptor extracts error message
  ├─ Logs to analytics
  └─ Component shows: showErrorToast(error.message)
```

### Async Flow (Kafka / Event-Driven)

**Note**: Kafka not present in this repo. Async flows use polling and webhooks.

```
User places order → Order status = "PENDING"
  ↓
Frontend polls: GET /order/{orderId}/status (every 2 sec for 5 min)
  ↓
Backend receives payment → settlement API call
  ↓
Settlement success (or failure webhook)
  ↓
Backend marks order status = "CONFIRMED" (or "FAILED")
  ↓
Frontend poll detects change → updates portfolio
  ↓
Notification sent to user (SMS, email, in-app)
```

### Package/Directory Navigation

```
src/
├── api/
│   ├── index.ts                 # APIClient class (all HTTP calls)
│   ├── interceptors.ts          # Request/response handlers
│   ├── ErrorModel.ts            # Error DTO
│   └── ssrClient.ts             # SSR-specific client (web)
│
├── b2c-components/              # Atomic UI components (atoms)
│   ├── Button/
│   ├── Card/
│   ├── Input/
│   ├── Typography/
│   ├── Icons/ (50+ icon SVGs)
│   └── ...
│
├── b2c-business/                # Domain logic helpers
│   └── utils.ts                 # Business calculations, mappings
│
├── molecules/                   # Composite components (atoms + layout)
│   ├── Header/
│   ├── BottomSheet/
│   ├── Modal/
│   ├── Toast/
│   ├── KycStatusBanner/
│   └── ...
│
├── screens/                     # Full-screen views (route destinations)
│   ├── bondDetail/              # Bond info, returns calculator, invest button
│   ├── bondExplore/             # Bond discovery, filters
│   ├── kycV2/                   # KYC workflow (multi-step)
│   ├── portfolio/               # Holdings, maturity tracking
│   ├── payment/                 # Payment mode selection, bank account entry
│   ├── orders/                  # Order list, order detail
│   ├── profile/                 # User account, KYC status, settings
│   ├── homeRevamp/              # Home dashboard (curated widgets)
│   └── ...
│
├── navigation/                  # React Navigation stacks
│   ├── Navigation.tsx           # Root container + deep linking config
│   ├── PublicNavigation.tsx     # Pre-auth screens (login, KYC)
│   ├── AppNavigation.tsx        # Post-auth screens (home, explore, portfolio)
│   ├── linking.ts              # Deep link route definitions
│   └── screenPathConfig.ts     # Screen name constants
│
├── store/                       # Global state (Hookstate)
│   ├── authStore.ts             # JWT, tokens, session
│   ├── profileStore.ts          # User profile, KYC status
│   ├── kycStore.ts              # KYC form data
│   ├── remoteConfigStore.ts     # Feature flags
│   ├── modalStackStore.ts       # Modal/bottom sheet state
│   └── ...
│
├── hooks/                       # Custom React hooks (50+)
│   ├── useAuth.ts               # Auth logic, login/logout
│   ├── useAppLaunch.ts          # App initialization
│   ├── useRemoteConfig.ts       # Fetch feature flags
│   ├── useWatchlist.ts          # Watchlist operations
│   ├── useProfile.tsx           # User data fetching
│   ├── useBondDetail.ts         # Bond data + calculator
│   ├── customNativeHooks/
│   │   ├── useAppRoute.ts       # Get route params (cross-platform)
│   │   ├── useAppNavigation.ts  # Navigate (cross-platform)
│   │   └── useAppIsFocused.ts   # Check if screen focused
│   └── ...
│
├── manager/                     # Singleton-style managers
│   ├── analytics/               # Analytics service wrapper
│   │   ├── AmplitudeService.ts
│   │   ├── FirebaseService.tsx
│   │   ├── AppsFlyerService.ts
│   │   ├── FBService.ts
│   │   └── WebEngageService.ts
│   ├── payment/                 # Razorpay integration
│   ├── themeManager/            # Theme context, token definitions
│   ├── remoteConfig/            # Firebase remote config fetcher
│   ├── permissionManager/       # Mobile permissions (camera, location)
│   ├── asyncStore/              # AsyncStorage (mobile) / localStorage (web)
│   ├── ticketManager/           # Freshdesk integration (support)
│   ├── contentManagementSystem/ # Strapi CMS integration
│   └── downloadDocument/        # Download PDF/docs
│
├── service/                     # API service layer
│   ├── newBankDematAccount/     # Bank account CRUD
│   ├── kycPermission.dataService.ts
│   └── token.dataservice.ts     # JWT refresh logic
│
├── constants/                   # App-wide constants
│   ├── screenNames.ts           # Screen name enums
│   ├── userKycStatus.ts         # KYC statuses, item IDs
│   ├── kycDocuments.ts          # Document type definitions
│   ├── product.ts               # Product type enums
│   ├── ssrRoutes.ts             # SSR-only route names (web)
│   └── ...
│
├── i18n/                        # Internationalization
│   └── languages/
│       ├── en.json              # English translations
│       └── hi.json              # Hindi translations
│
├── validators/                  # Form & data validators
│   ├── validateEmail.ts
│   ├── validatePANNumber.ts
│   ├── validatePincode.ts
│   └── ...
│
├── utils/                       # Utility functions
│   ├── deviceDetails.ts         # isWeb(), isAndroid(), isIOS()
│   ├── envUtils.ts              # getEnv() → returns current environment
│   ├── ssrUtils.ts              # SSR-specific utilities (web)
│   ├── deepLinkHandler.ts       # DeepLink routing logic
│   ├── dateUtils.ts             # Date formatting, parsing
│   └── ...
│
├── native/                      # Native module bridges
│   ├── att/                     # ATT (App Tracking Transparency) — iOS only
│   ├── sms/                     # SMS Auto-verify — Android only
│   └── phoneHint/               # Phone number hint picker — Android only
│
├── entities/                    # TypeScript data models
│   ├── bond/                    # Bond type definitions
│   ├── kyc/                     # KYC form types
│   └── ...
│
├── deeplinking/                 # Deep link routing
│   ├── DeepLinkingRoutes.ts     # Route-to-handler mapping
│   ├── DeepLinkManager.tsx      # Manages active deeplinks
│   └── linking.ts               # URL patterns
│
└── types/
    ├── global.d.ts              # Global type definitions
    └── hyperverge.d.ts          # Third-party type stubs

app/ (Next.js web app)
├── [dynamicRoutes]/             # App router routes (web)
│   ├── bond/
│   │   ├── [bondSlug]/
│   │   │   ├── [isin]/
│   │   │   │   ├── [bondId]/
│   │   │   │   │   └── page.tsx
│   ├── home/
│   ├── portfolio/
│   └── ...
├── api/                         # Next.js API routes
│   └── health/
│       └── route.ts             # Health check endpoint
├── layout.tsx                   # Root layout
├── error.tsx                    # Error boundary (web)
└── ...

android/
├── app/
│   ├── src/
│   │   ├── main/                # Default flavor (production)
│   │   ├── debug/               # Debug signing config
│   │   ├── qa/                  # QA build variant
│   │   ├── uat/                 # UAT build variant
│   │   ├── staging/             # Staging variant
│   │   ├── ga/                  # Production variant (GA = General Availability)
│   │   └── prod/                # Production signing config
│   ├── build.gradle             # Gradle config
│   └── proguard-rules.pro       # ProGuard rules (minification)
├── gradle/
│   └── wrapper/                 # Gradle wrapper (fixed version)
└── build.gradle                 # Root Gradle config

ios/
├── Fin.xcodeproj/               # Xcode project
│   └── xcshareddata/
│       └── xcschemes/           # Build schemes
│           ├── Fin.xcscheme     # Production
│           ├── FinQA.xcscheme   # QA
│           └── ...
├── Fin/                         # iOS source
│   ├── AppDelegate.mm           # App entry point
│   ├── Info.plist               # App configuration
│   ├── Fin.entitlements         # Capabilities (push, etc.)
│   ├── Config/                  # Environment-specific Firebase configs
│   │   ├── Prod/
│   │   ├── QA/
│   │   └── ...
│   └── ...
├── Podfile                      # CocoaPods dependencies
└── fastlane/                    # Fastlane automation (builds, releases)
    └── lanes/
        └── ios/
            └── build.rb         # iOS build lane

config/
├── apiConfiguration.ts          # API endpoint URLs per environment
├── analyticsConfig.ts           # Analytics API keys
├── FirebaseGoogleSSOConfigs.ts  # Firebase config per environment
└── ...

docs/
├── architecture/                # Architecture docs
│   ├── project-structure.md
│   └── typescript-migration.md
├── development/                 # Developer guide
│   ├── code-style.md
│   ├── testing.md
│   └── workflow.md
├── ssr/                         # SSR (web) documentation
│   ├── architecture.md
│   ├── metadata-population.md
│   └── ...
└── IOS-Build.md                 # iOS-specific build guide
```

### Key Design Patterns Used

| Pattern | Where Used | Purpose |
|---------|-----------|---------|
| **Custom Hooks** | `hooks/` (50+) | Encapsulate logic; enable reuse across screens |
| **Context + Hookstate** | `store/`, `manager/` | Global state without Redux boilerplate |
| **Manager Singleton** | Analytics, Payment, Theme, Permissions | Single instance across app lifetime |
| **Screen Navigation Stack** | `navigation/` | Type-safe routing with params |
| **Data Service Layer** | `service/`, `hooks/*.dataservice.ts` | Separate API calls from UI logic |
| **Component Composition** | `b2c-components/` + `molecules/` | Atoms (Button, Card) → molecules (Modal, Header) → screens |
| **Platform-Specific Files** | `.web.ts`, `.web.tsx` | One codebase, three platforms |
| **Theme Tokens** | `b2c-themes/`, `manager/themeManager/` | Centralized colors, typography, spacing |
| **Error Boundary** | `error.tsx` (web), wrapped screens | Graceful error handling |
| **Async/Await with Interceptors** | `api/interceptors.ts` | Auto token refresh, error normalization |
| **Feature Flags** | `remoteConfigStore` | Runtime feature gating |

---

## 7. Security & Access Control

### Authentication Mechanism: JWT (JSON Web Token)

**Flow:**
```
1. User enters phone/email + OTP
2. Backend validates OTP → issues JWT + Refresh Token
3. JWT stored in AsyncStorage (mobile) / localStorage (web)
4. Every request includes: Authorization: Bearer {JWT}
5. Backend validates JWT signature + expiry (30 min default)
6. On 401: interceptor calls refresh endpoint → gets new JWT → retries request
7. On refresh failure: logout → redirect to Sign In
```

**Required Headers for All Calls:**

```typescript
Authorization: Bearer {JWT}  // Always required (except public endpoints)
Device-Info: {              // Mobile only (added by interceptor)
  "imei": "device-imei",
  "ipAddress": "192.168.1.1",
  "deviceId": "unique-device-id"
}
Channel: invest              // Identifies domain/tenant
```

### RBAC / Permission Matrix

| Role | Endpoints | Capabilities | Restrictions |
|------|-----------|--------------|--------------|
| **Anonymous** | POST /auth/login, GET /bonds (public) | Browse bonds, read FAQs | Cannot place orders, view portfolio |
| **KYC_PENDING** | All authenticated endpoints | All (same as APPROVED) | Order placement blocked (UI gate + API validation) |
| **KYC_APPROVED** | All authenticated endpoints | Browse, order, portfolio | None |
| **KYC_REJECTED** | Limited endpoints (profile, support) | View rejection reason, re-submit KYC | Cannot place orders, portfolio hidden |
| **AML_FAILED** | Minimal (logout, support ticket) | Only support contact | Account effectively suspended |
| **Admin** | All admin endpoints (not in mobile app) | KYC review, user suspension, order cancellation | None (internal use only) |

### Service-to-Service Auth Pattern

**Mobile ↔ Backend:**
- JWT in every request
- Refresh token stored securely (Keychain/iOS, Keystore/Android)
- Interceptor handles token refresh automatically

**Backend ↔ External (NSDL, Razorpay, etc.):**
- API keys stored in secrets manager (not in code)
- HTTPS + TLS 1.2+ enforced
- Rate limiting per service

### Public vs Protected Endpoints

| Endpoint | Auth Required | Example |
|----------|--------------|---------|
| **Public** | No | `GET /bonds/search`, `POST /auth/login`, `GET /faq` |
| **Protected** | JWT | `POST /order/place`, `GET /portfolio`, `PUT /profile` |
| **Sensitive** | JWT + PIN | `POST /order/place` (if amount > threshold), `DELETE /account` |

### Token Expiry & Refresh Flow

```
JWT (Access Token)
├─ Expiry: 30 minutes
├─ Stored in: AsyncStorage (mobile) / sessionStorage (web)
└─ On 401: Automatically refreshed

Refresh Token
├─ Expiry: 7 days (or login session end)
├─ Stored in: Keychain (iOS) / Keystore (Android) / secure httpOnly cookie (web)
├─ Used to: Obtain new JWT
└─ On 401: User must re-login if expired

PIN (2FA)
├─ Expiry: Sliding window (30 min of inactivity OR session end)
├─ On 403 (PIN expired): Show PIN re-entry modal
└─ On success: Resume operation
```

### Rate Limiting / Throttling

**Mobile → Backend:**
- Per-user rate limits (e.g., 100 requests/min for search, 5 orders/hour)
- Enforced via backend middleware
- Returns 429 (Too Many Requests) if exceeded
- Interceptor shows toast: "Too many requests. Please try again in X seconds."

**Frontend Throttling:**
```typescript
// Search input: throttle 500ms (prevents accidental double-taps)
// Invest button: disable for 2 sec post-click (prevents double submission)
```

### Secrets Management

| Secret | Type | Storage | Rotation |
|--------|------|---------|----------|
| **API Keys** (Razorpay, Firebase, etc.) | String | AWS Secrets Manager (production), .env file (local) | Manual; document rotation policy |
| **JWT Secret** | String | Backend secrets manager only (never in mobile) | On key rotation, old tokens expire per policy |
| **Keystore/Keychain Passwords** | String | Developer machine (for code signing) | Per CI/CD pipeline, never in git |
| **Firebase Config** | JSON | `GoogleService-Info.plist` (iOS), `google-services.json` (Android) | Versioned per environment, fetched from Firebase console |

**Required Environment Variables (Local Setup):**

```bash
REACT_APP_API_BASE_URL=https://api.aspero.co.in     # Backend URL
REACT_APP_FIREBASE_API_KEY=AIza...                  # Firebase API key
REACT_APP_RECAPTCHA_KEY=6LeIx...                    # reCAPTCHA public key
REACT_APP_RAZORPAY_KEY=rzp_live_...                 # Razorpay public key
```

---

## 8. Local Setup & Running the Service

### Prerequisites

- **Node.js**: >= 22.18.0 (check with `node -v`)
- **Yarn**: 1.22.x Classic (install: `npm install -g yarn@1.22.21`)
- **iOS** (if building for iOS):
  - Xcode 15+ (install from App Store)
  - CocoaPods (`sudo gem install cocoapods`)
  - Minimum iOS deployment target: 13.0
- **Android** (if building for Android):
  - Android Studio 2024.x+
  - JDK 17 (set `JAVA_HOME`)
  - Android SDK with API 34 (Android 14) and build tools
  - Gradle Wrapper (included in repo)
- **Git**: For cloning and version control

### Step-by-Step: Bootstrap & Install

**1. Clone the repo:**
```bash
git clone https://github.com/yubisecurities/yubi-b2c-mobile.git
cd yubi-b2c-mobile
```

**2. Run bootstrap script (clears cache, installs dependencies):**
```bash
zsh bootstrap.sh install
```
This will:
- Clear Metro bundler cache
- Clear Gradle cache
- Install yarn dependencies
- Pod install (iOS)

**3. Verify Node/Yarn versions:**
```bash
node -v       # Should be 22.x.x
yarn -v       # Should be 1.22.x
```

**4. (iOS only) Install Pods:**
```bash
cd ios && pod install && cd ..
```

### Start Dependencies (Docker Compose)

**Note**: This repo does **not** use docker-compose for backend; backend is hosted (AWS). For local dev, you use staging/QA backend.

### Build & Run the Service

#### **Option 1: React Native Development Mode (Mobile)**

**Start Metro bundler:**
```bash
yarn start
```
Output: `Metro is running on port 8081`

**In another terminal, run on iOS:**
```bash
yarn iOS
# OR specific variant:
yarn iOSQA      # QA build on iPhone 17 Pro Max
yarn iOSUAT     # UAT build
```

**In another terminal, run on Android:**
```bash
yarn android
# OR specific variant:
yarn androidQA
yarn androidUAT
```

**First run setup:**
- On first run, Metro compiles JavaScript → app launches in simulator/emulator
- If Metro hangs, kill it (`Ctrl+C`) and restart
- If simulator/emulator doesn't start, open Xcode or Android Studio manually and launch device

#### **Option 2: Web Development Mode**

```bash
yarn start:web:qa
# Starts Next.js dev server at https://localhost:8080
# Open browser: https://invest-local.aspero.co.in:8080
```

**Troubleshoot HTTPS locally:**
- Browser will warn about self-signed cert
- Click "Advanced" → "Proceed anyway"
- Or add `/etc/hosts` entry: `127.0.0.1 invest-local.aspero.co.in`

#### **Option 3: Production Build (Release)**

**Android AAB (for Play Store):**
```bash
cd android
./gradlew bundleGaRelease
# Output: android/app/build/outputs/bundle/gaRelease/app-ga-release.aab
```

**iOS Archive (for TestFlight/App Store):**
```bash
cd ios && fastlane ios build
# Guides you through Xcode archive → export process
```

**Web Production Build:**
```bash
yarn build:web:qa
# Output: .next/standalone/ (ready to deploy)
```

### Required Environment Variables & Secrets

**Create `.env.local` (for local dev, development environment):**

```env
# API & Backend
REACT_APP_API_BASE_URL=https://api-qa.aspero.co.in

# Firebase (download from Firebase Console)
REACT_APP_FIREBASE_API_KEY=AIzaSyD...
REACT_APP_FIREBASE_PROJECT_ID=aspero-app-qa
REACT_APP_FIREBASE_APP_ID=1:123456789:web:abc123...

# Analytics
REACT_APP_AMPLITUDE_KEY=abcd1234...

# Payment
REACT_APP_RAZORPAY_KEY=rzp_test_...

# Recaptcha
REACT_APP_RECAPTCHA_KEY=6LeIxAcT...
```

**For iOS/Android native builds:**
- Place `GoogleService-Info.plist` (iOS) in `ios/Config/{Prod|QA|UAT}/`
- Place `google-services.json` (Android) in `android/app/src/{prod|qa|uat}/`
- These are versioned in repo (team has access)

### Local Config Values & Defaults

| Key | File | Default | Purpose |
|-----|------|---------|---------|
| `API_BASE_URL` | `.env.*` | https://api.aspero.co.in | Backend endpoint |
| `ENVIRONMENT` | `constants/product.ts` | derived from env file | dev/qa/uat/prod flag |
| `FIREBASE_REGION` | Firebase config | us-central1 | Cloud function region |
| `LOG_LEVEL` | `.env.development` | debug | Console logging verbosity |

### Common Local Setup Pitfalls & Solutions

| Pitfall | Symptom | Solution |
|---------|---------|----------|
| **Old Node cache** | Metro bundler crashes; `ReferenceError: primordials not defined` | Run `yarn cache clean && yarn start --reset-cache` |
| **Pod version conflict** | `Podfile.lock` mismatch; native build fails | `rm -rf ios/Pods ios/Podfile.lock && pod install` |
| **Android SDK missing** | `Gradle: Android SDK not found` | Run `echo 'sdk.dir=/path/to/android/sdk' > android/local.properties` |
| **Java version mismatch** | `Gradle: Unsupported class-file format` | Verify `java -version` returns 17.x; export `JAVA_HOME` |
| **M1/M2 Mac issues** | Android build fails; `Invalid APK` | Use `arch -x86_64 /bin/bash` before gradle commands |
| **Stale app state** | Buttons don't work; screens hang | Uninstall app from device/emulator; restart Metro |
| **Wrong Firebase config** | Auth fails; sign-up doesn't work | Verify `GoogleService-Info.plist` (iOS) and `google-services.json` (Android) are in correct folder per environment |

### Access Swagger / API Docs Locally

**Web API docs** (if backend provides):
- Navigate to: https://api-qa.aspero.co.in/swagger-ui.html (if available)
- Or check backend repo for Postman collection

**In-app API testing:**
- Use React Query Devtools (if installed)
- Or add a dev menu: `Developer → API Tester` (if implemented)

---

## 9. API Reference

### Required Headers for All Calls

```
Authorization: Bearer {JWT}                    // Always
Content-Type: application/json                 // For POST/PUT
Device-Info: {...}                            // Mobile only (auto-added)
Channel: invest                                // Tenant identifier
```

### API Versioning Strategy

**Current**: No explicit versioning (v1 implicit). **Future**: If breaking changes needed, will use `/api/v2/` prefix. **Migration**: Old version endpoints remain supported for 6 months, then deprecated.

### Sample Curl Requests (Core Operations)

#### **1. Login (Sign In with Phone)**

```bash
curl -X POST https://api-qa.aspero.co.in/auth/login \
  -H "Content-Type: application/json" \
  -d '{
    "phone": "+919876543210",
    "deviceId": "device-uuid"
  }'

# Response:
{
  "success": true,
  "data": {
    "sessionId": "sess-abc123",
    "isOtpRequired": true
  }
}
```

#### **2. Verify OTP**

```bash
curl -X POST https://api-qa.aspero.co.in/auth/verify-otp \
  -H "Content-Type: application/json" \
  -d '{
    "sessionId": "sess-abc123",
    "otp": "123456"
  }'

# Response:
{
  "success": true,
  "data": {
    "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "refreshToken": "refresh-token-xyz",
    "expiresIn": 1800,
    "user": {
      "id": "user-123",
      "phone": "+919876543210",
      "email": "user@example.com",
      "kycStatus": "PENDING"
    }
  }
}
```

#### **3. Get Bond Listing (Search)**

```bash
curl -X GET 'https://api-qa.aspero.co.in/bonds/search?sector=Banking&rating=AAA&pageSize=20&pageNo=1' \
  -H "Authorization: Bearer {JWT}"

# Response:
{
  "success": true,
  "data": {
    "totalCount": 42,
    "pageSize": 20,
    "pageNo": 1,
    "bonds": [
      {
        "id": "bond-xyz",
        "isin": "INE123A01020",
        "issuerName": "SBI Ltd",
        "couponRate": 6.5,
        "maturityDate": "2026-12-31",
        "rating": "AAA",
        "tenor": "3Y",
        "sector": "Banking"
      },
      ...
    ]
  }
}
```

#### **4. Get Bond Detail**

```bash
curl -X GET 'https://api-qa.aspero.co.in/bonds/{bondId}' \
  -H "Authorization: Bearer {JWT}"

# Response:
{
  "success": true,
  "data": {
    "id": "bond-xyz",
    "isin": "INE123A01020",
    "issuerName": "SBI Ltd",
    "issuerRating": "AAA",
    "couponRate": 6.5,
    "frequency": "ANNUAL",
    "maturityDate": "2026-12-31",
    "issuePrice": 1000,
    "currentPrice": 1005,
    "yieldToMaturity": 6.2,
    "interestPaymentDates": ["2024-12-31", "2025-12-31"],
    "documents": [
      { "name": "Prospectus", "url": "..." },
      { "name": "Rating Rationale", "url": "..." }
    ]
  }
}
```

#### **5. Place Order**

```bash
curl -X POST https://api-qa.aspero.co.in/order/place \
  -H "Authorization: Bearer {JWT}" \
  -H "Content-Type: application/json" \
  -d '{
    "bondId": "bond-xyz",
    "quantity": 10,
    "amount": 10050,
    "paymentMethod": "BANK_TRANSFER",
    "dematAccountId": "demat-123"
  }'

# Response:
{
  "success": true,
  "data": {
    "orderId": "order-abc123",
    "status": "PENDING",
    "bondId": "bond-xyz",
    "amount": 10050,
    "quantity": 10,
    "fee": 0,
    "totalAmount": 10050,
    "virtualAccountNumber": "1112220123456789",
    "bankName": "HDFC Bank",
    "ifsc": "HDFC0001234",
    "expiresAt": "2024-12-20T23:59:59Z",
    "createdAt": "2024-12-19T12:00:00Z"
  }
}
```

#### **6. Get Portfolio**

```bash
curl -X GET https://api-qa.aspero.co.in/portfolio \
  -H "Authorization: Bearer {JWT}"

# Response:
{
  "success": true,
  "data": {
    "holdings": [
      {
        "bondId": "bond-xyz",
        "quantity": 10,
        "averageCost": 1000,
        "currentValue": 10050,
        "unrealizedGain": 500,
        "maturityDate": "2026-12-31",
        "nextCouponDate": "2025-12-31"
      }
    ],
    "totalValue": 50250,
    "totalInvested": 50000,
    "totalGains": 250
  }
}
```

### Complete Endpoint Table

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| **Submission Endpoints** |
| POST | /auth/login | No | Send OTP to phone/email |
| POST | /auth/verify-otp | No | Verify OTP, issue JWT |
| POST | /auth/refresh-token | No | Refresh expired JWT |
| POST | /kyc/submit | Yes | Submit KYC form |
| POST | /kyc/upload-document | Yes | Upload KYC document |
| POST | /order/place | Yes + PIN (if amount > threshold) | Place bond/FD order |
| **Retrieval Endpoints** |
| GET | /bonds/search | No | Search bonds with filters |
| GET | /bonds/{bondId} | No | Get bond detail |
| GET | /bonds/{bondId}/similar | No | Get similar bonds |
| GET | /issuer/{cinNumber} | No | Get issuer page |
| GET | /portfolio | Yes | Get user holdings |
| GET | /watchlist | Yes | Get watchlist bonds |
| GET | /orders | Yes | Get order history |
| GET | /orders/{orderId} | Yes | Get order detail + status |
| GET | /kyc/status | Yes | Get KYC status |
| GET | /user/profile | Yes | Get user profile |
| GET | /user/bank-accounts | Yes | Get saved bank accounts |
| GET | /user/demat-accounts | Yes | Get saved demat accounts |
| **Status/Query Endpoints** |
| GET | /order/{orderId}/status | Yes | Poll order settlement status |
| GET | /kyc/{kycId}/status | Yes | Poll KYC approval status |
| GET | /payment/{paymentId}/status | Yes | Check payment status |
| **Workflow Endpoints** |
| PUT | /order/{orderId}/cancel | Yes + PIN | Cancel pending order |
| PUT | /kyc/resubmit | Yes | Resubmit rejected KYC |
| POST | /watchlist/{bondId}/add | Yes | Add bond to watchlist |
| DELETE | /watchlist/{bondId} | Yes | Remove from watchlist |
| **Admin Endpoints** |
| PUT | /admin/kyc/{kycId}/approve | Admin | Admin approve KYC (internal) |
| PUT | /admin/kyc/{kycId}/reject | Admin | Admin reject KYC (internal) |
| DELETE | /admin/user/{userId} | Admin | Suspend user account (internal) |

### Request Body Structure (Key Fields)

#### **KYC Submission**

```json
{
  "pan": "ABCDE1234F",
  "aadhaarNumber": "123456789012",
  "firstName": "John",
  "lastName": "Doe",
  "dateOfBirth": "1990-01-15",
  "address": {
    "line1": "123 Main St",
    "line2": "",
    "city": "Mumbai",
    "state": "Maharashtra",
    "postalCode": "400001",
    "country": "IN"
  },
  "addressProofType": "AADHAAR",
  "bankAccount": {
    "accountNumber": "12345678901234",
    "ifsc": "HDFC0001234",
    "bankName": "HDFC Bank",
    "accountHolder": "John Doe",
    "type": "SAVINGS"
  },
  "dematAccount": {
    "dpId": "IN302111",
    "clientId": "12345678",
    "bo": "1234567890123456"
  },
  "riskProfile": {
    "riskScore": 3,
    "investmentHorizon": "5Y",
    "annualIncome": 1000000
  }
}
```

#### **Order Placement**

```json
{
  "bondId": "bond-xyz",
  "quantity": 10,
  "amount": 10050,
  "paymentMethod": "BANK_TRANSFER",
  "dematAccountId": "demat-123",
  "termsAccepted": true
}
```

---

## 10. How to Extend the System

### **1. Adding a New Document Type (KYC)**

**Scenario**: Support NREGA for rural verification.

**Step 1: Define enum** (`src/constants/kycDocuments.ts`)
```typescript
export interface IKycDocument {
  id: string;
  sectionNo: number;
  status: string;
  rejectionReason?: string;
}

export enum KYC_DOCUMENT_TYPES {
  AADHAAR = 'AADHAAR',
  PAN = 'PAN',
  DRIVING_LICENSE = 'DRIVING_LICENSE',
  PASSPORT = 'PASSPORT',
  VOTER_ID = 'VOTER_ID',
  NREGA = 'NREGA',  // NEW
}

export const KYC_DOCUMENT_CONFIG = {
  NREGA: {
    label: 'NREGA Job Card',
    acceptedFormats: ['pdf', 'jpg', 'png'],
    maxSize: 10 * 1024 * 1024, // 10 MB
    mandatory: false,
    validationRules: {
      /* ... */
    },
  },
};
```

**Step 2: Add to KYC form** (`src/screens/kycV2/enums/kycForm.fields.ts`)
```typescript
export const KYC_FORM_FIELDS = {
  ADDITIONAL_DOCUMENTS: [
    {
      stepId: 'NREGA',
      label: 'NREGA Job Card',
      inputType: 'document-upload',
      required: false,
      validator: validateNregaJobCard,
    },
  ],
};
```

**Step 3: Create validator** (`src/validators/validateNregaJobCard.ts`)
```typescript
export const validateNregaJobCard = (file: File): { valid: boolean; error?: string } => {
  if (file.size > 10 * 1024 * 1024) {
    return { valid: false, error: 'File size must be less than 10 MB' };
  }
  // Add NREGA-specific validation logic
  return { valid: true };
};
```

**Step 4: Update backend to accept** 
- Add `nrega_document_url` field to KYC API
- Add NREGA validation in backend KYC service

**Step 5: Test**
```bash
yarn test -- src/validators/validateNregaJobCard.test.ts
```

### **2. Adding a New Async Listener (Kafka Alternative: Polling)**

**Scenario**: Poll for KYC approval status every 5 seconds.

**Step 1: Create a custom hook** (`src/hooks/useKycStatusPolling.ts`)
```typescript
import { useEffect } from 'react';
import { useHookstate } from '@hookstate/core';
import { kycPermissionStore } from '@src/store/kycPermissionStore';
import { apiClient } from '@src/api';

export const useKycStatusPolling = () => {
  const kycState = useHookstate(kycPermissionStore);

  useEffect(() => {
    let pollingInterval: NodeJS.Timeout;

    const startPolling = async () => {
      pollingInterval = setInterval(async () => {
        try {
          const response = await apiClient.get({
            url: '/kyc/status',
          });

          const newStatus = response.data?.kycStatus;
          if (newStatus && newStatus !== kycState.status.get()) {
            kycState.status.set(newStatus);

            // Notify user
            if (newStatus === 'APPROVED') {
              showToast('success', { info: t('KYC_APPROVED') });
            } else if (newStatus === 'REJECTED') {
              showToast('error', { info: t('KYC_REJECTED') });
            }
          }
        } catch (error) {
          console.error('KYC polling error:', error);
        }
      }, 5000); // Poll every 5 seconds
    };

    startPolling();

    return () => {
      if (pollingInterval) clearInterval(pollingInterval);
    };
  }, []);
};
```

**Step 2: Use in screen** (`src/screens/kycV2/Kyc.tsx`)
```typescript
const Kyc = () => {
  const { t } = useTranslation();
  useKycStatusPolling(); // Auto-poll on mount

  return <KycStatusBanner />;
};
```

**Step 3: DLQ & Retry Strategy**
- Max retries: 10 (exponential backoff: 5s, 10s, 20s, ...)
- On persistent failure: show error banner, disable polling
- Manual retry button: "Refresh Status"

**Step 4: Test**
```bash
yarn test -- src/hooks/useKycStatusPolling.test.ts
```

### **3. Adding a New Product Type (e.g., Corporate Bonds)**

**Scenario**: Add corporate bonds (non-government).

**Step 1: Create product enum** (`src/constants/product.ts`)
```typescript
export enum PRODUCT_TYPE {
  GOVERNMENT_BOND = 'GOVERNMENT_BOND',
  CORPORATE_BOND = 'CORPORATE_BOND',  // NEW
  FD = 'FD',
  SGB = 'SGB',
}
```

**Step 2: Add tab to bottom navigation** (`src/navigation/homeTabNavigator/ExploreNavigation.tsx`)
```typescript
const ExploreStack = () => {
  return (
    <Tab.Navigator>
      <Tab.Screen name="AllBonds" component={BondExplore} />
      <Tab.Screen name="GovernmentBonds" component={GovernmentBondsExplore} />
      <Tab.Screen name="CorporateBonds" component={CorporateBondsExplore} /> {/* NEW */}
      <Tab.Screen name="FD" component={FdScreen} />
    </Tab.Navigator>
  );
};
```

**Step 3: Create screen** (`src/screens/corporateBondExplore/CorporateBondExplore.tsx`)
```typescript
const CorporateBondExplore = () => {
  const { t } = useTranslation();
  const [bonds, setBonds] = useState([]);

  useEffect(() => {
    fetchCorporateBonds();
  }, []);

  const fetchCorporateBonds = async () => {
    try {
      const response = await apiClient.get({
        url: '/bonds/search',
        params: { type: PRODUCT_TYPE.CORPORATE_BOND, pageSize: 20 },
      });
      setBonds(response.data.bonds);
    } catch (error) {
      showToast('error', { info: t('FAILED_TO_LOAD_BONDS') });
    }
  };

  return (
    <FlatList
      data={bonds}
      renderItem={({ item }) => <BondCard bond={item} />}
      keyExtractor={(item) => item.id}
    />
  );
};
```

**Step 4: Update feature flag** (Firebase Remote Config)
- Add key: `enable_corporate_bonds` = `true`

**Step 5: Backend work**
- Add bond type field to bonds table
- Add filtering in `/bonds/search` API
- Update issuer ratings for corporate issuers

**Step 6: Test**
```bash
yarn test -- src/screens/corporateBondExplore/CorporateBondExplore.test.tsx
```

### **4. Adding a New Mandatory Requirement (Config-Only if Possible)**

**Scenario**: Require user to accept updated terms and conditions.

**Option A: Config-Only (Remote Config)**

Firebase Remote Config:
```json
{
  "require_new_tnc": {
    "active": true,
    "version": "2.0",
    "url": "https://cdn.aspero.co.in/tnc-v2.0.pdf",
    "message": "Please accept updated Terms and Conditions"
  }
}
```

**Code** (`src/molecules/tnc/useTnc.ts`):
```typescript
export const useTnc = () => {
  const configState = useRemoteConfigState();
  const userProfile = useUserProfileState();

  useEffect(() => {
    const tncConfig = configState.require_new_tnc.get();
    if (tncConfig?.active && !userProfile.tncAcceptedVersion.get().includes(tncConfig.version)) {
      // Show TNC modal
      showTncModal(tncConfig);
    }
  }, [configState.require_new_tnc.get()]);
};
```

**Option B: Code Change (If Needed)

If requirement involves new data fields or complex logic:

**Step 1: Add to user schema** (Backend)
```sql
ALTER TABLE users ADD COLUMN biometric_enabled BOOLEAN DEFAULT FALSE;
```

**Step 2: Add to UI** (`src/screens/profile/additionalDetails/AdditionalDetails.tsx`)
```typescript
const AdditionalDetails = () => {
  const [biometricEnabled, setBiometricEnabled] = useState(false);

  const handleBiometricToggle = async () => {
    try {
      await apiClient.put({
        url: '/user/profile',
        data: { biometricEnabled: !biometricEnabled },
      });
      setBiometricEnabled(!biometricEnabled);
      showToast('success', { info: t('SETTINGS_UPDATED') });
    } catch (error) {
      showToast('error', { info: t('UPDATE_FAILED') });
    }
  };

  return (
    <Switch
      value={biometricEnabled}
      onValueChange={handleBiometricToggle}
      label={t('ENABLE_BIOMETRIC')}
    />
  );
};
```

### **5. Changing a User-Facing Message or Rejection Reason**

**Scenario**: Update error message for failed KYC.

**Step 1: Update translation** (`src/i18n/languages/en.json`)
```json
{
  "KYC": {
    "REJECTION_REASONS": {
      "INVALID_PAN": "PAN number is invalid. Please check and resubmit.",
      "AADHAAR_MISMATCH": "Aadhaar data does not match PAN. Please update.",
      "ADDRESS_PROOF_UNCLEAR": "Address proof document is unclear. Please resubmit a clearer image."
    }
  }
}
```

**Step 2: Use in error display** (`src/screens/kycV2/KycForm.tsx`)
```typescript
const KycForm = () => {
  const { t } = useTranslation();
  const rejectionReason = kycState.rejectionReason.get();

  if (rejectionReason) {
    const message = t(`KYC.REJECTION_REASONS.${rejectionReason}`);
    return <ErrorBanner message={message} />;
  }
};
```

**Step 3: Backend update**
- Return rejection reason code from API (not hardcoded message)
- Frontend translates using key

### **6. Adding a New Workflow or Workflow Step**

**Scenario**: Add identity verification step after PAN check.

**Step 1: Create step component** (`src/screens/kycV2/steps/identityVerification/IdentityVerification.tsx`)
```typescript
const IdentityVerification = () => {
  const { t } = useTranslation();

  return (
    <View>
      <Typography variant="heading">{t('VERIFY_IDENTITY')}</Typography>
      <Video source={require('identity-verification-guide.mp4')} />
      <Button onPress={() => startLivenessCheck()} title={t('START_VERIFICATION')} />
    </View>
  );
};
```

**Step 2: Add to KYC form fields** (`src/screens/kycV2/enums/kycForm.fields.ts`)
```typescript
export const KYC_FORM_FIELDS = {
  IDENTITY_VERIFICATION: IdentityVerification,
};
```

**Step 3: Update step sequence** (`src/screens/kycIntermediateScreen/KycIntermediateScreen.type.ts`)
```typescript
export const SupportedKycSteps = [
  KYC_ITEM_ID.PAN,
  KYC_ITEM_ID.ADDRESS,
  'IDENTITY_VERIFICATION',  // NEW
  KYC_ITEM_ID.BANK,
  KYC_ITEM_ID.DEMAT,
];
```

**Step 4: Backend KYC service**
- Add validation for identity verification result
- Mark as part of KYC submission

### **7. Error Handling Pattern**

**Standard error structure:**

```typescript
interface ErrorResponse {
  errors?: Array<{
    errorCode: string;
    message: string;
    constraintViolations?: Array<{
      fieldName: string;
      reason: string;
    }>;
  }>;
}
```

**Handling in components:**

```typescript
const handleOrderPlacement = async () => {
  try {
    const response = await apiClient.post({
      url: '/order/place',
      data: orderData,
    });
    showToast('success', { info: t('ORDER_PLACED_SUCCESSFULLY') });
  } catch (error: any) {
    const errorMessage =
      error?.response?.data?.errors?.[0]?.message || t('ORDER_PLACEMENT_FAILED');
    showToast('error', { info: errorMessage });
    
    // Log for analytics
    logErrorToAnalytics({
      errorCode: error?.response?.data?.errors?.[0]?.errorCode,
      screen: 'ORDER_CONFIRMATION',
      operation: 'PLACE_ORDER',
    });
  }
};
```

### **8. Mapper Pattern (MapStruct → TypeScript)**

**Scenario**: Convert API bond response to UI bond model.

**Create mapper** (`src/b2c-business/mappers/BondMapper.ts`)
```typescript
import { Bond } from '@entities/bond/Bond.type';

interface ApiBondResponse {
  id: string;
  isin: string;
  issuerName: string;
  couponRate: number;
  maturityDate: string;
  rating: string;
  tenor: string;
}

export class BondMapper {
  static toBondUi(apiBond: ApiBondResponse): Bond {
    return {
      id: apiBond.id,
      isin: apiBond.isin,
      issuerName: apiBond.issuerName,
      coupon: {
        rate: apiBond.couponRate,
        frequency: 'ANNUAL',
      },
      maturityDate: new Date(apiBond.maturityDate),
      rating: apiBond.rating,
      tenor: apiBond.tenor,
      metadata: {
        lastUpdated: new Date(),
      },
    };
  }

  static toBondArray(apiBonds: ApiBondResponse[]): Bond[] {
    return apiBonds.map((bond) => this.toBondUi(bond));
  }
}
```

**Use in service** (`src/service/bond.dataservice.ts`)
```typescript
export const fetchBonds = async (): Promise<Bond[]> => {
  const response = await apiClient.get({ url: '/bonds' });
  return BondMapper.toBondArray(response.data.bonds);
};
```

### **9. Adding a New Role or Permission**

**Scenario**: Add "Fund Manager" role who can approve orders.

**Step 1: Define role** (`src/store/authStore.ts`)
```typescript
export enum USER_ROLE {
  USER = 'USER',
  ADMIN = 'ADMIN',
  FUND_MANAGER = 'FUND_MANAGER',  // NEW
}

export interface AuthState {
  role: USER_ROLE;
  permissions: string[];
}
```

**Step 2: Add permissions** (Backend grants these to role)
```typescript
const PERMISSIONS = {
  USER: ['READ_BONDS', 'PLACE_ORDER', 'VIEW_PORTFOLIO'],
  FUND_MANAGER: [
    'READ_BONDS',
    'PLACE_ORDER',
    'VIEW_PORTFOLIO',
    'APPROVE_ORDER',  // NEW
    'VIEW_ALL_ORDERS',  // NEW
  ],
  ADMIN: ['*'],  // All permissions
};
```

**Step 3: Gate features** (`src/screens/orders/Orders.tsx`)
```typescript
const Orders = () => {
  const authState = useAuthStore();
  const canApproveOrders = authState.permissions.get().includes('APPROVE_ORDER');

  return (
    <View>
      <OrdersList />
      {canApproveOrders && <ApprovalPanel />}
    </View>
  );
};
```

**Step 4: Backend authorization**
- Check role before allowing order approval
- Log approval action for audit

---

## 11. Data Model & Status Rules Engine

### Core Database Tables (Backend Schema Reference)

| Table | Purpose | Key Indexes | Key Fields |
|-------|---------|------------|-----------|
| **users** | User accounts | `phone`, `email` (unique) | id, phone, email, role, kycStatus, pinHash, createdAt |
| **kyc_submissions** | KYC form submissions | `userId`, `status` | id, userId, pan, aadhaarNumber, address, bankAccount, dematAccount, kycStatus, rejectionReason, submittedAt, approvedAt |
| **bonds** | Bond master data | `isin` (unique), `sector`, `rating` | id, isin, issuerCin, issuerName, couponRate, maturityDate, tenor, rating, sector, listingDate |
| **orders** | User investment orders | `userId`, `status`, `createdAt` | id, userId, bondId, quantity, amount, fee, orderStatus, paymentMethod, paymentId, virtualAccountNumber, settledAt |
| **portfolio_holdings** | User current holdings | `userId`, `bondId` | id, userId, bondId, quantity, averageCost, currentValue, acquiredAt, maturityDate |
| **watchlist** | User watchlist bonds | `userId`, `bondId` | id, userId, bondId, addedAt |
| **bank_accounts** | User saved bank details | `userId`, `accountNumber` (unique) | id, userId, accountNumber, ifsc, bankName, accountHolder, isPrimary |
| **demat_accounts** | User demat details | `userId`, `dpId`, `clientId` (unique) | id, userId, dpId, clientId, boNumber, isLinked, linkedAt |
| **payments** | Payment transactions | `orderId`, `status` | id, orderId, amount, paymentMethod, paymentGatewayId, status, transactionId, createdAt |
| **audit_logs** | Audit trail | `userId`, `eventType`, `createdAt` | id, userId, eventType, entityType, entityId, beforeValue, afterValue, createdAt |

### Key Enums and Their Values

| Enum | Values | Explanation |
|------|--------|-------------|
| **KYC_STATUS** | PENDING, SUBMITTED, IN_REVIEW, APPROVED, REJECTED, CKYC_NOT_FOUND | Current KYC state |
| **ORDER_STATUS** | PENDING, PAYMENT_INITIATED, SETTLEMENT_IN_PROGRESS, CONFIRMED, FAILED, CANCELLED | Investment order lifecycle |
| **PAYMENT_STATUS** | INITIATED, PROCESSING, SUCCESS, FAILED, PENDING_SETTLEMENT | Payment gateway state |
| **USER_ROLE** | USER, ADMIN, FUND_MANAGER | User permission level |
| **PRODUCT_TYPE** | GOVERNMENT_BOND, CORPORATE_BOND, FD, SGB | Investment product category |
| **RATING** | AAA, AA, A, BBB, BB, B, C, D, NR (Not Rated) | Credit rating by CRISIL/ICRA |
| **BOND_STATUS** | ACTIVE, MATURED, DELISTED, SUSPENDED | Bond trading status |

### Full Status/State Rules Config

**KYC Status Derivation Logic:**

```javascript
// pseudocode
function deriveKycStatus(submission) {
  if (!submission.exists()) {
    return 'PENDING';
  }

  if (submission.status === 'REJECTED') {
    return 'REJECTED';
  }

  if (submission.manualReviewNeeded) {
    return 'IN_REVIEW';
  }

  // Run automated checks
  if (!validatePan(submission.pan)) {
    return 'REJECTED';
  }

  if (!validateAadhaar(submission.aadhaar)) {
    return 'REJECTED';
  }

  // Run AML/KRA checks
  const amlResult = runAmlCheck(submission.pan);
  if (amlResult.failed) {
    return 'AML_FAILED';
  }

  const kraResult = runKraCheck(submission);
  if (kraResult.risk === 'VERY_HIGH') {
    return 'KRA_FAILED';
  }

  // All checks passed
  return 'APPROVED';
}
```

**Order Status Derivation Logic:**

```javascript
function deriveOrderStatus(order, payment, settlement) {
  if (!order.exists()) return null;

  if (order.cancelledAt) {
    return 'CANCELLED';
  }

  if (!payment.exists()) {
    return 'PENDING';  // Awaiting payment
  }

  if (payment.status === 'FAILED') {
    return 'FAILED';
  }

  if (payment.status === 'SUCCESS' && !settlement.exists()) {
    return 'SETTLEMENT_IN_PROGRESS';  // Waiting for clearing
  }

  if (settlement.status === 'CONFIRMED') {
    return 'CONFIRMED';
  }

  if (settlement.status === 'FAILED') {
    return 'FAILED';
  }

  return 'PAYMENT_INITIATED';
}
```

### How Dynamic Statuses Are Derived from Non-DB Sources

**Example: Real-time bond price (not stored in DB)**

```typescript
// Hook that derives current bond value from market data service
export const useBondCurrentPrice = (bondId: string) => {
  const [currentPrice, setCurrentPrice] = useState(null);

  useEffect(() => {
    const subscription = marketDataService.subscribe(bondId, (newPrice) => {
      setCurrentPrice(newPrice);
    });
    return () => subscription.unsubscribe();
  }, [bondId]);

  const yieldToMaturity = deriveYtmFromPrice(currentPrice, bond);

  return { currentPrice, yieldToMaturity };
};
```

### Idempotency Pattern

**Duplicate Request Prevention:**

```typescript
// APIClient adds idempotency key header for POST/PUT
const apiClient = {
  post: async (config) => {
    const idempotencyKey = `${config.url}-${Date.now()}-${Math.random()}`;
    return axios.post(config.url, config.data, {
      headers: {
        'Idempotency-Key': idempotencyKey,
      },
    });
  },
};

// Backend checks for duplicate Idempotency-Key:
// If key exists in cache → return cached response (don't re-process)
// If new key → process & cache response
```

**Idempotency Rules:**

| Operation | Idempotent? | Handling |
|-----------|------------|----------|
| Place Order | Yes | Backend stores orderId per idempotency key; retry returns same orderId |
| Payment Callback | Yes | Check if payment already processed before updating |
| KYC Submission | Yes | If same submission → reject "Already submitted"; if new → accept |
| Update Profile | Yes | Timestamp-based (only accept if newer than last update) |

### Caching Strategy

| What | Key | TTL | Eviction |
|-----|-----|-----|----------|
| **Bond List** | `bonds:search:{filters_hash}` | 5 minutes | LRU (least recently used) |
| **Bond Detail** | `bond:{bondId}` | 10 minutes | LRU |
| **User Profile** | `user:{userId}:profile` | 1 hour | LRU + manual invalidation on update |
| **Remote Config** | `remote_config:all` | 5 minutes (or manual refresh) | Manual refresh on app focus |
| **Watchlist** | `user:{userId}:watchlist` | 1 hour | Manual invalidation on add/remove |
| **Portfolio** | `user:{userId}:portfolio` | 2 minutes | LRU + manual invalidation on order |
| **Images (CDN)** | `image:{cdn_url}` | 7 days (configurable) | File-based; disk quota based |

**Caching Implementation:**

```typescript
// Frontend caching via AsyncStorage + TTL
export const useCachedData = (key: string, fetcher: () => Promise<any>, ttl: number) => {
  const [data, setData] = useState(null);

  useEffect(() => {
    const getCachedOrFetch = async () => {
      const cached = await AsyncStorage.getItem(key);
      if (cached) {
        const { value, timestamp } = JSON.parse(cached);
        if (Date.now() - timestamp < ttl * 1000) {
          setData(value);
          return;
        }
      }

      const freshData = await fetcher();
      await AsyncStorage.setItem(key, JSON.stringify({ value: freshData, timestamp: Date.now() }));
      setData(freshData);
    };

    getCachedOrFetch();
  }, [key]);

  return data;
};
```

### Audit Logging

**What Events Are Captured:**

| Event | Table | Fields Logged |
|-------|-------|---|
| User Sign In | audit_logs | userId, eventType: LOGIN, ipAddress, device |
| KYC Submitted | audit_logs | userId, eventType: KYC_SUBMITTED, kycId, submittedAt |
| Order Placed | audit_logs | userId, eventType: ORDER_PLACED, orderId, amount, bondId |
| Order Cancelled | audit_logs | userId, eventType: ORDER_CANCELLED, orderId, reason |
| KYC Approved | audit_logs | userId, eventType: KYC_APPROVED, kycId, approvedBy (admin) |
| Password Changed | audit_logs | userId, eventType: PASSWORD_CHANGED, timestamp |

**Retention**: 7 years (regulatory requirement in India)

**Structured Log Pattern:**

```json
{
  "id": "audit-xyz",
  "userId": "user-123",
  "eventType": "ORDER_PLACED",
  "entityType": "order",
  "entityId": "order-abc",
  "beforeValue": null,
  "afterValue": {
    "orderId": "order-abc",
    "bondId": "bond-xyz",
    "amount": 10000,
    "status": "PENDING"
  },
  "ipAddress": "203.0.113.45",
  "userAgent": "AsperoApp/1.0.0 (iOS)",
  "timestamp": "2024-12-19T12:00:00Z"
}
```

---

## 12. Integration Map

### Integration Diagram (ASCII)

```
┌─────────────────────────────────────────────────────────────────┐
│                   ASPERO B2C MOBILE APP                         │
├─────────────────────────────────────────────────────────────────┤
│  ├─ NSDL/CDSL Demat  ──→ Link BO Account (bonds settlement)    │
│  ├─ CKYC              ──→ Verify PAN against CKYC database     │
│  ├─ AML Provider      ──→ Screen users against sanctions list  │
│  ├─ Razorpay          ──→ Process payments (UPI, Card, etc.)   │
│  ├─ Firebase          ──→ Remote config, analytics, auth       │
│  ├─ Email Service     ──→ Send order confirmations, KYC status │
│  ├─ SMS Service       ──→ Send OTP, payment alerts, maturity   │
│  ├─ CMS (Strapi)      ──→ Blog, FAQ, product content          │
│  ├─ Chat Widget (Zoho)──→ Live support chat                   │
│  ├─ Analytics         ──→ Event tracking (Amplitude, FB)       │
│  ├─ AppsFlyer         ──→ Deep linking, user acquisition       │
│  └─ Kfuse             ──→ Observability, error tracking        │
└─────────────────────────────────────────────────────────────────┘
        │
        ↓
┌─────────────────────────────────────────────────────────────────┐
│                    YUBI BACKEND SERVICES                        │
│  ├─ Auth Service      (JWT issue/refresh)                      │
│  ├─ KYC Service       (submission, approval)                   │
│  ├─ Bond Service      (catalog, pricing)                       │
│  ├─ Order Service     (order placement, status)                │
│  ├─ Portfolio Service (holdings, performance)                  │
│  ├─ Payment Service   (Razorpay webhook handling)              │
│  ├─ Settlement Service(bond settlement tracking)               │
│  └─ Notification Service (email, SMS)                          │
└─────────────────────────────────────────────────────────────────┘
        │
        ↓
┌─────────────────────────────────────────────────────────────────┐
│                   DATABASES & CACHES                            │
│  ├─ PostgreSQL (user data, KYC, orders, audit)                 │
│  ├─ Redis (session cache, rate limits, feature flags)          │
│  ├─ S3 (document storage: PAN, Aadhaar, KYC)                   │
│  └─ Elasticsearch (order history search)                       │
└─────────────────────────────────────────────────────────────────┘
```

### Integration Table

| System | Purpose | Protocol | Direction | Auth | Failure Handling |
|--------|---------|----------|-----------|------|-----------------|
| **NSDL Demat** | Link securities BO to order | HTTPS REST | Outbound | API Key | Retry (3x), manual follow-up |
| **CDSL Demat** | Alternative demat provider | HTTPS REST | Outbound | API Key | Fallback to NSDL; manual review |
| **CKYC** | PAN verification (identity) | HTTPS XML-RPC | Outbound | Cert-based mTLS | Reject KYC if CKYC unavailable |
| **AML Provider** | Sanctions list screening | HTTPS REST | Outbound | API Key | Block account; alert ops team |
| **Razorpay** | Payment processing | HTTPS REST | Bi-directional | API Key | Webhook retry (5x); manual reconciliation |
| **Firebase** | Remote config, analytics, auth | HTTPS REST | Bi-directional | SDK | Fallback to cached config |
| **Email Service** | Send confirmations, notifications | HTTPS REST / SMTP | Outbound | API Key | Queue; retry next hour |
| **SMS Service** | Send OTP, alerts | HTTPS REST | Outbound | API Key | SMS gateway failover; email fallback |
| **Strapi CMS** | Content delivery (blogs, FAQs) | HTTPS GraphQL/REST | Outbound | API Token | Cache locally; stale content acceptable |
| **Zoho Chat** | Live support widget | WebSocket | Bi-directional | OAuth | Graceful degradation (disable chat) |
| **Amplitude** | Event analytics | HTTPS REST | Outbound | API Key | Queue locally; send on next app start |
| **Facebook Analytics** | Conversion tracking | HTTPS REST | Outbound | SDK | Silent fail (not critical) |
| **Kfuse** | Error tracking & observability | HTTPS REST | Outbound | API Key | Buffer errors; flush on next successful connection |
| **AppsFlyer** | Attribution & deep linking | HTTPS REST | Outbound | SDK | Local queue; retry on app resume |

---

## 13. Operational Runbook

### Common Failure Modes & Resolution

| Symptom | Likely Cause | Resolution |
|---------|-------------|-----------|
| **Users cannot sign up (OTP fails)** | SMS gateway overloaded or credential issue | Check SMS provider status → verify API key in secrets → test with test number |
| **KYC approval stuck (still IN_REVIEW after 48h)** | Backend KYC service crashed or manual queue blocked | SSH to backend → check KYC service logs → check Redis for stuck jobs → manually trigger reprocessing |
| **Orders not settling (stuck in PAYMENT_INITIATED)** | Payment gateway webhook not received or backend processing hung | Check Razorpay dashboard for payment status → check webhook retry logs → trigger manual settlement reconciliation script |
| **Bonds not loading (explore page blank)** | Bond catalog API down or data corrupted | Check backend bond service health → query PostgreSQL for bond count → restart service if needed |
| **Portfolio shows 0 holding (data lost)** | Database query timeout or cache stale | Invalidate user portfolio cache → force refresh from DB → check for any failed writes in audit logs |
| **Deeplinks not routing (user clicks link, app doesn't open)** | Deep linking config missing or URL scheme not registered | Verify `linking.ts` has correct route → check native build includes URL scheme (iOS `Info.plist`, Android `AndroidManifest.xml`) |
| **App crashes on startup (black screen)** | Corrupted AsyncStorage or infinite loop on init | Clear app cache/data → reinstall app → check init logs for errors |
| **Payment gateway returns 500 error** | Razorpay outage or account misconfiguration | Check Razorpay status page → verify merchant account active → reach out to Razorpay support |
| **User sees "KYC Rejected" but reason unclear** | Backend didn't store rejection reason properly | Check `kyc_submissions` table for this user → manually update with reason → notify user via support |
| **Orders appear in portfolio but user wasn't charged** | Settlement success but payment not confirmed | Manual reconciliation: check if payment ID exists → if not, refund logic should trigger automatically |

### How to Check DLQ / Retry Queue State

**Razorpay Payment Webhook DLQ (Dead Letter Queue):**

```bash
# SSH to backend
ssh user@backend-prod.aspero.co.in

# Check Redis for failed webhooks
redis-cli KEYS "razorpay:webhook:retry:*" | wc -l  # Count failed webhooks

# Inspect specific failed webhook
redis-cli HGETALL razorpay:webhook:retry:payment-abc123

# Manual reprocessing (if safe)
redis-cli DEL razorpay:webhook:retry:payment-abc123  # Remove from DLQ
# Then manually trigger webhook handler with payload
```

**KYC Submission Queue:**

```bash
# Check if KYC submissions are stuck
psql -h db-prod.aspero.co.in -U aspero_user -d aspero_db \
  -c "SELECT COUNT(*) as stuck_kyc FROM kyc_submissions WHERE status='IN_REVIEW' AND submitted_at < NOW() - INTERVAL '48 hours'"

# Trigger reprocessing for specific KYC
curl -X POST https://api-prod.aspero.co.in/admin/kyc/{kycId}/reprocess \
  -H "Authorization: Bearer {admin-jwt}"
```

### Manual Trigger / Unstick Stuck Workflow

**Stuck Order (SETTLEMENT_IN_PROGRESS for 24h+):**

```bash
# Check order status in DB
psql -c "SELECT * FROM orders WHERE id='order-abc' AND status='SETTLEMENT_IN_PROGRESS';"

# Manually check settlement status with NSDL
curl -X GET https://nsdl-api.example.com/settlement/order-abc \
  -H "Authorization: Bearer {nsdl-api-key}"

# If settled: manually update order in DB
psql -c "UPDATE orders SET status='CONFIRMED' WHERE id='order-abc';"

# Notify user of successful settlement
curl -X POST https://api-prod.aspero.co.in/notification/send \
  -d '{"userId":"user-123","type":"ORDER_CONFIRMED","orderId":"order-abc"}'
```

**Stuck KYC (IN_REVIEW for 72h+):**

```bash
# Trigger manual KYC reprocessing
curl -X POST https://api-prod.aspero.co.in/admin/kyc/{kycId}/reprocess \
  -H "Authorization: Bearer {admin-jwt}" \
  -H "Content-Type: application/json"

# If fails: escalate to manual review
# Assign to admin for manual decision (approve/reject)
```

### Key Log Queries / Dashboards

**Kibana (Log Search):**

```
# Failed order placements
service: order-service AND event: ORDER_PLACEMENT_FAILED AND @timestamp > now-1h

# KYC rejection trend
service: kyc-service AND kyc_status: REJECTED AND @timestamp > now-24h | stats count by rejection_reason

# Payment failures
service: payment-service AND razorpay.status: failed AND @timestamp > now-1h | stats count by error_code
```

**Datadog Dashboard (Monitoring):**

- **Orders Dashboard**: Order placement rate, avg settlement time, success rate
- **KYC Dashboard**: KYC submissions, approval rate, avg time to approval
- **Payment Dashboard**: Transaction volume, failure rate, avg transaction value
- **Error Dashboard**: Error rate by service, top error codes

### Rollback Procedure

**If a new deployment causes issues:**

```bash
# Check current deployed version
kubectl describe deployment aspero-backend -n production | grep image

# Rollback to previous version
kubectl rollout undo deployment/aspero-backend -n production

# Verify rollback
kubectl describe deployment aspero-backend -n production | grep image

# Check pod health
kubectl get pods -n production | grep aspero-backend
```

**For database migrations (if needed):**

```bash
# Backup current DB
pg_dump aspero_db > backup-2024-12-19.sql

# Rollback migration (if migration tool supports it)
flyway undo

# Or manually revert SQL changes
psql -f rollback-migration-2024-12-19.sql
```

---

## 14. Testing, Deployment & Infrastructure

### How to Run Tests Locally

```bash
# All tests
yarn test

# Watch mode (re-run on file changes)
yarn test:watch

# Single test file
NODE_OPTIONS=--experimental-vm-modules jest --config=jest.config.cjs src/screens/bondDetail/__tests__/BondDetail.test.tsx

# Coverage report
yarn coverage

# View in browser
yarn open:coverage
```

### Test Pattern Example (Most Critical Service)

**Critical Service: Order Placement** (`src/screens/bondExplore/hooks/__tests__/useBondExploreData.test.ts`)

```typescript
import { renderHook, waitFor } from '@testing-library/react-native';
import { useBondExploreData } from '../useBondExploreData';
import * as api from '@src/api';

jest.mock('@src/api');

describe('useBondExploreData', () => {
  it('should fetch bonds on mount', async () => {
    const mockBonds = [
      { id: '1', isin: 'INE001', issuerName: 'SBI', couponRate: 6.5 },
    ];

    (api.apiClient.get as jest.Mock).mockResolvedValue({
      data: { bonds: mockBonds },
    });

    const { result } = renderHook(() => useBondExploreData());

    await waitFor(() => {
      expect(result.current.bonds).toEqual(mockBonds);
    });
  });

  it('should handle API errors gracefully', async () => {
    (api.apiClient.get as jest.Mock).mockRejectedValue(new Error('API Error'));

    const { result } = renderHook(() => useBondExploreData());

    await waitFor(() => {
      expect(result.current.error).toBeDefined();
      expect(result.current.bonds).toEqual([]);
    });
  });

  it('should apply filters correctly', async () => {
    const mockBonds = [
      { id: '1', rating: 'AAA', sector: 'Banking' },
      { id: '2', rating: 'AA', sector: 'IT' },
    ];

    (api.apiClient.get as jest.Mock).mockResolvedValue({
      data: { bonds: mockBonds },
    });

    const { result, rerender } = renderHook(
      ({ filters }) => useBondExploreData(filters),
      { initialProps: { filters: { rating: 'AAA' } } }
    );

    await waitFor(() => {
      expect(api.apiClient.get).toHaveBeenCalledWith(
        expect.objectContaining({ params: expect.objectContaining({ rating: 'AAA' }) })
      );
    });
  });
});
```

### Environment Matrix

| Environment | Replicas | CPU/Memory | Purpose | Data Retention |
|-------------|----------|-----------|---------|-----------------|
| **Development** | 1 | 0.5 CPU / 1 GB RAM | Local testing | Temporary (auto-reset) |
| **QA** | 2 | 2 CPU / 4 GB RAM | Internal testing, UAT prep | 7 days |
| **UAT** | 3 | 4 CPU / 8 GB RAM | User acceptance testing | 30 days |
| **Production (GA)** | 5 | 8 CPU / 16 GB RAM | Production | Permanent (archived after 7 years) |

### CI/CD Pipeline Diagram (ASCII)

```
Developer pushes to `main` branch
        ↓
┌─────────────────────────────────────────────┐
│ Jenkins / GitHub Actions (Jenkinsfile)      │
├─────────────────────────────────────────────┤
│ Stage 1: Checkout & Build                   │
│   ├─ Clone repo                             │
│   ├─ Install dependencies (yarn)            │
│   └─ Compile TypeScript                     │
│                                             │
│ Stage 2: Test                               │
│   ├─ Run Jest tests                         │
│   ├─ Generate coverage report               │
│   └─ Fail if coverage < 60%                 │
│                                             │
│ Stage 3: Lint & Type Check                  │
│   ├─ ESLint validation                      │
│   └─ TypeScript strict mode check           │
│                                             │
│ Stage 4: Build Artifacts                    │
│   ├─ Build Android (AAB/APK)                │
│   ├─ Build iOS (IPA)                        │
│   └─ Build Web (Next.js bundle)             │
│                                             │
│ Stage 5: Push to Registry                   │
│   ├─ Docker image → ECR                     │
│   └─ Artifacts → S3                         │
│                                             │
│ Stage 6: Deploy to QA                       │
│   ├─ Deploy Docker image to ECS (QA)        │
│   └─ Run smoke tests                        │
│                                             │
│ Stage 7: (Optional) Deploy to UAT/Prod      │
│   └─ Manual approval required               │
└─────────────────────────────────────────────┘
        ↓
    Success → Notify team (Slack)
    Failure → Notify team + block merge
```

**Jenkinsfile (Key Stages):**

```groovy
pipeline {
  agent any
  
  stages {
    stage('Build') {
      steps {
        sh 'yarn install'
        sh 'yarn build:web:qa'
      }
    }
    
    stage('Test') {
      steps {
        sh 'yarn test --coverage'
      }
    }
    
    stage('Deploy to QA') {
      steps {
        sh './devops/deploy-qa.sh'
      }
    }
  }
  
  post {
    failure {
      sh 'curl -X POST https://slack.com/api/chat.postMessage -d "channel=#dev&text=Build failed"'
    }
  }
}
```

### Related Repositories

| Repository | Role | Link |
|-----------|------|------|
| `yubi-b2c-backend` | REST API, KYC, orders | https://github.com/yubisecurities/yubi-b2c-backend |
| `yubi-b2c-admin` | Admin panel (KYC review, user mgmt) | https://github.com/yubisecurities/yubi-b2c-admin |
| `yubi-infra` | Terraform, Kubernetes configs | https://github.com/yubisecurities/yubi-infra |
| `yubi-docs` | Technical documentation | https://github.com/yubisecurities/yubi-docs |

### Known Gaps & Improvement Areas

| Gap | Impact | Owner | Priority |
|-----|--------|-------|----------|
| **No A/B testing framework** | Hard to test UX variations | PM | Medium |
| **Manual KYC approval process** | Slow turnaround (24-48h) | KYC Ops | High |
| **No real-time bond price updates** | Stale pricing data (10min delay) | Trading | High |
| **Limited payment options** | Razorpay-only; no international | Payments | Medium |
| **No offline support** | App unusable without internet | Mobile | Low |
| **Incomplete error translations** | English error for non-English users | i18n | Medium |
| **No advanced portfolio analytics** | Users can't analyze performance | Analytics | Low |

### Known Developer Gotchas & Workarounds

| Gotcha | Behavior | Explanation | Workaround |
|--------|----------|-------------|-----------|
| **Stale Redux state** | Orders show as PENDING after settlement | Frontend cache not invalidated | Manually call `refreshPortfolio()` after payment success |
| **M1 Mac Android build fails** | Gradle fails with architecture mismatch | Android build tools don't support ARM natively | Prepend `arch -x86_64` to gradle commands |
| **KYC step skips forward** | User clicks button, KYC jumps 2 steps | Navigation state not properly reset | Verify `KycStepsWrapper` properly tracks `selectedStep` state |
| **Watchlist bond disappears** | Bond added, then removed without user action | Cache invalidation race condition | Clear watchlist cache on any add/remove operation |
| **Deeplink doesn't route** | User taps link, app opens but wrong screen | Transient params registry cleared on page reload | Store transient data in URL as query param, not memory-only registry |
| **TypeScript compilation hangs** | `yarn typecheck` runs forever | Circular imports in type definitions | Check for circular imports using `madge` tool; break cycles |
| **Payment webhook received twice** | Order placed twice from single payment | Webhook retry fired before first processing complete | Implement idempotency key logic (already done); ensure DB constraints prevent duplicate orders |
| **Image not loading on web** | Bond image shows broken; works on mobile | CORS policy blocks CDN domain | Add CORS headers to CDN; use `next/image` component (handles CORS) |
| **Gesture handler conflict** | Bottom sheet interferes with list scroll | React Native Gesture Handler priority issue | Use `scrollEnabled` prop on gesture responder |
| **Android notification not showing** | Notification sent but device silent | Firebase config or channel mismatch | Verify `google-services.json` is for correct Android app flavor (prod/qa/uat) |

---

## 15. Appendix: Quick Reference

### Command Cheat Sheet

```bash
# Development
yarn start                      # Metro bundler
yarn iOS                        # Run iOS simulator
yarn android                    # Run Android device/emulator
yarn start:web:qa               # Web dev server (QA env)

# Testing
yarn test                       # Run all tests
yarn test:watch                 # Watch mode
yarn coverage                   # Coverage report

# Linting & Type Checking
yarn lint                       # ESLint
yarn typecheck                  # TypeScript check
yarn typecheck:strict           # Strict mode

# Build
yarn build:web:qa               # Web production build
cd android && ./gradlew assembleQaRelease  # Android APK (QA)
cd android && ./gradlew bundleGaRelease    # Android AAB (Production)

# Cleanup
rm -rf node_modules && yarn install       # Clean install
yarn cache clean && yarn start --reset-cache  # Reset Metro cache
```

### File Path Quick Links

```
Main app entry:              src/App.tsx (mobile), app/layout.tsx (web)
Navigation stacks:           src/navigation/
Global state:                src/store/
API client:                  src/api/index.ts
Screens:                     src/screens/
Components:                  src/b2c-components/, src/molecules/
Hooks:                       src/hooks/
Constants:                   src/constants/
Translations:                src/i18n/languages/
Environment config:          .env.* files, config/
```

### Key Contacts & Escalation

| Role | Contact | Slack |
|------|---------|-------|
| **iOS Lead** | - | @ios-team |
| **Android Lead** | - | @android-team |
| **Backend/API Lead** | - | @backend-team |
| **KYC/Compliance** | - | @kyc-team |
| **DevOps/Infra** | - | @devops-team |
| **On-Call** | - | @oncall-aspero |

---

**End of Master Context Document**

> **Last Updated**: December 2024  
> **Maintained By**: Aspero Engineering Team  
> **Next Review**: Q1 2025