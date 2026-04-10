```typescript
// Client: Generate idempotency key
const idempotencyKey = `${userId}-${bondId}-${timestamp}`;

// API call with idempotency key header
const placeOrder = async (bondId: string, amount: number) => {
  const idempotencyKey = generateIdempotencyKey(bondId);
  
  try {
    const response = await apiClient.post(
      '/orders',
      { bondId, amount },
      {
        headers: {
          'Idempotency-Key': idempotencyKey,
        },
      }
    );
    return response.data;
  } catch (error) {
    // Retry with same idempotency key
    return apiClient.post(
      '/orders',
      { bondId, amount },
      {
        headers: {
          'Idempotency-Key': idempotencyKey,
        },
      }
    );
  }
};

// Server stores idempotency key + response for 24 hours
// Duplicate requests with same key return cached response
```

### Caching Strategy

| What | Where | TTL | Eviction | Key Pattern |
|-----|-------|-----|----------|------------|
| **Bond listings** | Redis / localStorage | 1 hour | LRU | `bonds:list:{page}` |
| **Bond detail** | Redis / localStorage | 24 hours | LRU | `bond:{bondId}` |
| **User profile** | Memory (Hookstate) | Session | On logout | `user:{userId}` |
| **Portfolio holdings** | Memory (Hookstate) | 5 min (stale) | Manual refresh | `portfolio:{userId}` |
| **KYC status** | Memory (Hookstate) | 30 sec (polling) | On approval | `kyc:{userId}` |
| **Feature flags** | Memory (Hookstate) + Firebase | 1 hour | Periodic sync | `featureFlag:{flagName}` |
| **CMS content** (FAQs, blogs) | Memory + localStorage | 24 hours | LRU | `cms:content:{id}` |
| **Images** | Device storage (filesystem) | 7 days | Manual clearing | `img_cache/{hash}` |

### Audit Logging

**Events captured**:

| Event | Captured Data | Table | Retention |
|-------|---------------|-------|-----------|
| **User login** | user_id, timestamp, device_info, IP | audit_log | 7 years (regulatory) |
| **KYC submission** | kyc_id, user_id, field_values, timestamp | audit_log | 7 years |
| **KYC approval/rejection** | kyc_id, reviewer_id, reason, timestamp | audit_log | 7 years |
| **Order placed** | order_id, user_id, bond_id, amount, timestamp | orders | 7 years |
| **Payment received** | order_id, payment_id, amount, gateway_ref, timestamp | payments | 7 years |
| **Settlement completed** | order_id, debit_date, delivery_date, demat_ref | orders | 7 years |
| **Coupon paid** | holding_id, coupon_amount, payment_date, bank_ref | cashflows | 7 years |
| **Data export** | user_id, export_date, fields_exported | audit_log | 1 year (GDPR) |
| **Profile change** | user_id, field_changed, old_value, new_value, timestamp | audit_log | 3 years |

**Log entry format**:

```json
{
  "eventId": "evt_abc123",
  "eventType": "ORDER_PLACED",
  "userId": "user_12345",
  "timestamp": "2024-01-15T10:30:00Z",
  "deviceInfo": {
    "os": "ios",
    "version": "17.0",
    "appVersion": "1.2.3"
  },
  "context": {
    "orderId": "order_xyz",
    "bondId": "bond_001",
    "amount": 100000
  },
  "ipAddress": "203.0.113.42",
  "status": "SUCCESS"
}
```

---

## 12. Integration Map

### Integration Diagram

```
┌────────────────────────────────────────────────────────────────┐
│                    Aspero App (Client)                         │
├────────────────────────────────────────────────────────────────┤
│                                                                │
│  ┌──────────────┬─────────────┬──────────────────────────────┐ │
│  │   iOS        │   Android   │   Web (Next.js)              │ │
│  └──────────────┴─────────────┴──────────────────────────────┘ │
│                       │                                        │
└───────────────────────┼────────────────────────────────────────┘
                        │
         ┌──────────────┼──────────────────────────────────────┐
         ↓              ↓              ↓                        ↓
    ┌─────────┐  ┌─────────────┐  ┌──────────┐         ┌─────────────┐
    │Firebase │  │Razorpay     │  │HyperVerge│         │Strapi CMS   │
    │(Remote  │  │(Payment)    │  │(Liveness)│         │(Content)    │
    │Config,  │  │             │  │          │         │             │
    │Analytics)  │             │  │          │         │             │
    └─────────┘  └─────────────┘  └──────────┘         └─────────────┘
         │             │                │                       │
         │             │                │                       │
    ┌────────────────────────────────────────────────────────────────┐
    │              Yubi Backend Services                             │
    │  /user-service  /kyc-service  /bond-service  /order-service   │
    │  /payment-service  /portfolio-service                         │
    └────────────────────────────────────────────────────────────────┘
         │             │              │              │
         │             │              │              │
    ┌────┴───┐  ┌──────┴──────┐  ┌───┴────┐  ┌───────┴────────┐
    │ Firebase│  │Razorpay     │  │NSDL/   │  │Freshdesk       │
    │Database │  │Settlement   │  │CDSL    │  │(Support)       │
    │         │  │(T+1)        │  │Demat   │  │               │
    └─────────┘  └─────────────┘  └────────┘  └────────────────┘
         │
    ┌────┴──────────────┐
    │                   │
 ┌──┴──┐           ┌────┴───┐
 │Kafka│ (if used) │Analytics│
 │(Events)         │Warehouse│
 └─────┘           └─────────┘
```

### Integration Details Table

| System | Purpose | Protocol | Direction | Auth Method | Failure Handling | Notes |
|--------|---------|----------|-----------|-------------|------------------|-------|
| **Firebase** | Remote config, analytics, crash reporting | HTTPS REST | Bidirectional | API key (client-side) | Graceful degradation; features disabled | Non-critical; app works offline |
| **Razorpay** | Payment gateway | HTTPS REST | Bidirectional | API key (server-side) + webhook | User retry; manual reconciliation | PCI-compliant; handles card data |
| **HyperVerge** | KYC liveness check + face match | HTTPS REST | Unidirectional (client → server) | API key | User can retry; manual review | Called during KYC flow |
| **NSDL/CDSL** | Demat account validation + settlement | HTTPS REST + custom protocol | Bidirectional | mTLS + API key (backend) | Retry queue; escalation | Settlement is async (T+1 or T+2) |
| **Strapi CMS** | Blog, FAQ, video links | HTTPS REST | Unidirectional (read-only) | API token (public) | Cache fallback; stale content OK | Content is non-critical |
| **Freshdesk** | Support ticketing | HTTPS REST | Unidirectional (write-only) | API key (server-side) | Queue ticket; retry | Non-blocking; async |
| **AppsFlyer** | Attribution + deep linking | HTTPS REST | Unidirectional (write-only) | API key (SDK) | Event loss acceptable | Analytics only |
| **Amplitude** | Event analytics | HTTPS REST | Unidirectional (write-only) | API key (SDK) | Event loss acceptable | Analytics only |
| **WebEngage** | Push notifications | HTTPS REST | Bidirectional | API key (SDK) | Notify via SMS fallback | Non-critical; user engagement |

---

## 13. Operational Runbook

### Common Failure Modes

| Symptom | Likely Cause | Resolution |
|---------|-------------|------------|
| **Users see "Network Error" on bonds listing** | Backend service down OR CDN misconfigured | Check backend service status. Verify CDN cache headers. Rollback recent changes. |
| **KYC status stuck on "Under Review" for 24+ hours** | NSDL API timeout OR webhook not firing | Check NSDL API logs. Check webhook queue on backend. Manual review if stuck. |
| **Orders not settling after 48 hours** | Settlement instruction not sent OR NSDL delay | Check settlement queue. Verify demat account active. Contact NSDL support. |
| **"Unauthorized" on API calls** | JWT expired and refresh failed | User re-login required. Check token refresh endpoint. |
| **Payment stuck after user pays** | Razorpay webhook lost OR order not updated | Check Razorpay settlement account. Manual order status sync. |
| **App crashes on bond detail screen** | Unmapped API response field | Check error logs (Sentry). Verify API contract. Deploy fix. |
| **High latency (> 5s) on portfolio load** | N+1 query OR missing index on holdings table | Check backend query plan. Add index if missing. Cache portfolio summary. |
| **Watchlist not syncing across devices** | CloudSync disabled OR lastModified timestamp stale | Check user's CloudSync settings. Force sync via API. |

### How to Check DLQ / Retry Queue State

**Backend Dead Letter Queue** (not in mobile app; backend service):

```bash
# Check Redis DLQ (if using Redis)
redis-cli --scan --pattern "dlq:*"
redis-cli LLEN "dlq:settlement-instruction"

# Check Kafka DLQ (if using Kafka)
kafka-consumer-groups --bootstrap-server localhost:9092 \
  --group order-settlement-consumer \
  --reset-offsets --to-earliest --execute

# Check SQS (if AWS-based)
aws sqs get-queue-attributes \
  --queue-url https://sqs.us-east-1.amazonaws.com/.../dlq-settlement \
  --attribute-names ApproximateNumberOfMessages
```

### How to Manually Trigger or Unstick a Stuck Workflow

**Example**: Order stuck on PENDING_PAYMENT; user claims they paid but order not confirmed.

```bash
# 1. Check order status in backend DB
SELECT order_id, status, created_at, payment_id 
FROM orders 
WHERE order_id = 'order_xyz';

# 2. Check Razorpay settlement account
# Via Razorpay dashboard: Payments → Search by order_id
# Verify payment_id exists and status is 'captured' or 'authorized'

# 3. If Razorpay shows payment captured but order not updated:
# Manually trigger webhook
curl -X POST https://api-qa.aspero.co.in/webhooks/razorpay/payment-captured \
  -H "Content-Type: application/json" \
  -d '{"payment_id": "pay_xyz", "order_id": "order_xyz", "amount": 100000}'

# 4. Verify order status updated
SELECT order_id, status FROM orders WHERE order_id = 'order_xyz';
# Expected: status = CONFIRMED or SETTLED

# 5. If still stuck, escalate to support team
```

### Key Log Queries / Kibana Searches

```
# All failed orders in last 24 hours
status:REJECTED OR status:FAILED
timestamp:[now-24h TO now]

# All KYC rejections with reason
event_type:KYC_REJECTED
timestamp:[now-7d TO now]

# Payment timeouts (expiry without payment)
event_type:ORDER_EXPIRED
timestamp:[now-24h TO now]

# Network errors (user-facing)
error_code:NETWORK_ERROR
timestamp:[now-1h TO now]

# Slow API calls (> 1 second)
response_time_ms:[1000 TO *]
timestamp:[now-1h TO now]

# AML screening results
event_type:AML_SCREENING_RESULT
timestamp:[now-7d TO now]
```

### Rollback Procedure

**If a bad deployment breaks the app**:

1. **Identify bad commit**: Check CI/CD pipeline logs or recent git commits.
2. **Revert on backend** (if backend change): `git revert <commit-hash>`, push to main, CI redeploys.
3. **Revert on mobile**:
   - **iOS**: Previous IPA available in App Store. Downgrade via TestFlight or submit hotfix build.
   - **Android**: Previous APK available on Play Store. User downgrades or wait for hotfix.
   - **Web**: Rollback to previous Next.js build or S3 deployment snapshot.
4. **Notify users**: Post to support channel; update status page.
5. **Root cause analysis**: Post-mortem within 24 hours; add tests to prevent recurrence.

---

## 14. Testing, Deployment & Infrastructure

### How to Run Tests Locally

```bash
# All tests
yarn test

# Watch mode (re-run on file change)
yarn test:watch

# Coverage report
yarn coverage
open coverage/lcov-report/index.html

# Single test file
NODE_OPTIONS=--experimental-vm-modules jest --config=jest.config.cjs \
  src/screens/kycV2/steps/AadhaarPan/AadhaarPan.test.tsx

# Tests matching a pattern
yarn test --testNamePattern="KYC"
```

### Test Pattern Example (Critical Service)

**File**: `src/screens/kycV2/hooks/useKyc.test.ts`

```typescript
import { renderHook, act, waitFor } from '@testing-library/react-native';
import { useKyc } from './useKyc';
import * as kyc from './KycForm.dataServices';

jest.mock('./KycForm.dataServices');

describe('useKyc', () => {
  beforeEach(() => {
    jest.clearAllMocks();
  });

  test('should submit KYC with valid data', async () => {
    const mockSubmitKYC = jest.spyOn(kyc, 'submitKYC').mockResolvedValueOnce({
      kycId: 'kyc-123',
      status: 'SUBMITTED',
    });

    const { result } = renderHook(() => useKyc());

    act(() => {
      result.current.setFormData({
        pan: 'ABCDE1234F',
        aadhaar: '123456789012',
        dob: '1990-01-15',
      });
    });

    await act(async () => {
      await result.current.submitKYC();
    });

    await waitFor(() => {
      expect(mockSubmitKYC).toHaveBeenCalledWith(
        expect.objectContaining({
          pan: 'ABCDE1234F',
          aadhaar: '123456789012',
        })
      );
    });

    expect(result.current.kycStatus).toBe('SUBMITTED');
  });

  test('should handle KYC rejection', async () => {
    const mockSubmitKYC = jest
      .spyOn(kyc, 'submitKYC')
      .mockRejectedValueOnce({
        response: {
          data: {
            errors: [{ errorCode: 'AML_FLAG', message: 'AML screening failed' }],
          },
        },
      });

    const { result } = renderHook(() => useKyc());

    await act(async () => {
      try {
        await result.current.submitKYC();
      } catch (error) {
        // Expected
      }
    });

    expect(result.current.errorMessage).toBe('AML screening failed');
  });

  test('should poll KYC status', async () => {
    const mockGetStatus = jest
      .spyOn(kyc, 'getKYCStatus')
      .mockResolvedValueOnce({ status: 'UNDER_REVIEW' })
      .mockResolvedValueOnce({ status: 'APPROVED' });

    const { result } = renderHook(() => useKyc());

    await act(async () => {
      result.current.startStatusPolling();
    });

    await waitFor(() => {
      expect(result.current.kycStatus).toBe('APPROVED');
    });
  });
});
```

---

### Environment Matrix

| Environment | Replicas | Resources | Purpose | User Access |
|-------------|----------|-----------|---------|------------|
| **Development** | 1 (local or staging) | 512 MB RAM, 1 CPU | Rapid iteration, debugging | Developers only |
| **QA** | 2 (for redundancy) | 2 GB RAM, 2 CPU each | Internal testing, feature validation | QA team + select partners |
| **UAT** | 3 (for load testing) | 4 GB RAM, 4 CPU each | User acceptance testing, performance validation | QA + business stakeholders |
| **Production (GA)** | 5–10 (auto-scale) | 8 GB RAM, 8 CPU each | Live users, full compliance | All authenticated users |

---

### CI/CD Pipeline Diagram

```
Developer commits to main
  │
  ├─ GitHub webhook → Jenkins
  │
  ├─ Run: yarn lint, yarn typecheck, yarn test
  │  ├─ If fails: Notify in Slack; block merge
  │  └─ If passes: Continue
  │
  ├─ Build artifact:
  │  ├─ Web: Next.js bundle → S3
  │  ├─ Android: Gradle build → Firebase App Distribution
  │  └─ iOS: Xcode archive → TestFlight
  │
  ├─ Deploy to QA (auto)
  │  ├─ API: Backend deployment
  │  ├─ Web: S3 + CloudFront
  │  ├─ Mobile: Firebase App Distribution
  │
  ├─ Smoke tests (automated):
  │  ├─ Health check: GET /health
  │  ├─ Auth flow: Send OTP → Verify OTP
  │  ├─ Bond listing: GET /bonds
  │  └─ If any fail: Notify engineers; block deploy
  │
  ├─ Manual approval: QA lead reviews test results
  │  └─ If approved: Deploy to UAT
  │
  ├─ Deploy to UAT
  │  ├─ Run load tests
  │  ├─ Check performance metrics (p95 latency < 2s)
  │  └─ If metrics green: Ready for production
  │
  ├─ Manual approval: Product manager or VP reviews UAT results
  │  └─ If approved: Deploy to production
  │
  └─ Deploy to Production (blue-green or canary)
     ├─ 10% traffic to new version (canary)
     ├─ Monitor error rate, latency, conversion
     ├─ If green: Route 100% traffic
     └─ If red: Automatic rollback
```

---

### Related Repositories

| Repository | Role | Owner | Notes |
|------------|------|-------|-------|
| **yubi-backend** | Backend services (user, KYC, bond, order) | Platform team | Private; contains auth, DB schemas |
| **yubi-design-system** | Shared UI component library | Design + FE team | React components exported as package |
| **yubi-analytics** | Analytics pipeline + dashboards | Data team | Ingests events from clients and backend |
| **yubi-infra** | Kubernetes configs, Helm charts, Terraform | DevOps team | Private; prod secrets managed here |
| **yubi-docs** | Shared documentation (API, architecture) | Tech leads | Confluence or GitHub Wiki |

---

### Known Gaps & Improvement Areas

| Gap | Impact | Owner | Effort | Timeline |
|-----|--------|-------|--------|----------|
| **Biometric login not enabled** | Users must enter PIN every 24h; reduces convenience | Mobile team | Medium | Q2 2024 |
| **No fractional bonds** | Users cannot buy partial bonds; limits accessibility | Product | Large | Q3 2024 |
| **SSL pinning missing on Android** | Increased risk if user on compromised network | Security | Small | Q1 2024 |
| **No GraphQL API** | REST polling causes over-fetching; high data usage on slow networks | Backend | Large | Q4 2024 |
| **Manual KYC review workflow** | Scaling bottleneck; compliance team overwhelmed at peak | Compliance + Product | Medium | Q2 2024 |
| **No portfolio performance analytics** | Users cannot track returns; feature parity gap vs competitors | Product + Data | Medium | Q3 2024 |
| **No bulk order capability** | Institutions cannot invest easily; B2B revenue lost | Product | Large | 2025 |
| **Limited payment methods** | Only UPI/card; no international users | Payment team | Medium | Q3 2024 |
| **No real-time notifications** | WebSocket latency; users miss market-moving updates | Backend + Mobile | Medium | Q2 2024 |

---

### Known Developer Gotchas

| Gotcha | Explanation | Workaround |
|--------|-------------|------------|
| **Transient params on web** | Deep links via `/bond/[slug]/[isin]/[id]` lose params on page refresh; stored in memory, not URL | Use NextJS `useRouter().query` + registry pattern (see `useAppRoute.ts`). Data persists in memory during session. |
| **Platform-specific file resolution** | `.web.ts` files must exist OR export will fail | Always create `.web.tsx` pair if mobile has `.ts` version. Check `tsconfig.json` paths. |
| **Soft deletes** | Deleted users still in DB with `is_deleted = true`. Queries must filter. | Always add `.where({ is_deleted: false })` in backend queries. Ask backend for seed data without soft deletes for testing. |
| **Eventual consistency on settlements** | Order marked CONFIRMED but not yet SETTLED; takes 24+ hours. | Poll `/orders/{id}` every 30s for 48h. Show "Settlement in Progress" UX; don't block user. |
| **OTP timeout during development** | OTP expires after 5 min in QA; easy to miss while coding. | Extend OTP TTL in .env.qa to 30 min during development. Reset before committing. |
| **AsyncStorage serialization** | Complex objects must be JSON.stringify'd; dates become strings. | Always call `JSON.parse()` on retrieval. Use ISO string format for dates; parse back to Date object. |
| **Theme token enforcement** | Hardcoded colors will cause linter warnings; tests fail. | Use `lightThemeV2.semantic.colors.*` tokens. ESLint plugin catches hardcoded hex values. |
| **i18n scope** | `t('KEY')` fails if key doesn't exist; no runtime error, just blank string. | Always add both en.json + hi.json keys. Use test to validate all keys have translations. |
| **Firebase Remote Config stale** | Cached values not updated until next app launch. | Call `fetchAndActivate()` manually or set cache expiry to 0 in dev. |
| **Razorpay webhook idempotency** | Webhook may fire twice for same payment; duplicate orders possible. | Include idempotency key in order creation. Backend deduplicates. |

---

## End of Master Context

**Last updated**: 2026-04-10  
**Accuracy**: Inferred from codebase snapshot and CLAUDE.md project instructions. Verify critical sections with team before acting.

---
</code>