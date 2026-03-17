# Razorpay Demo Payment App — Product Requirements Document

## Product Overview

A learning-oriented mobile app that demonstrates the complete Razorpay payment lifecycle. It simulates an e-commerce purchase flow where users select products, pay via Razorpay, and see real-time order status updates driven by webhooks.

## Goals

- Demonstrate the **complete payment lifecycle** (order → checkout → verify → capture → status)
- Implement **server-side signature verification** (never trust the client)
- Handle **webhooks** for async payment confirmation
- Simulate **failure scenarios** (declined payments, network drops, timeout)
- Provide a foundation transferable to production apps

## Non-Goals

- Not a real e-commerce app (no real products, inventory, or shipping)
- No user authentication (not the focus)
- No production deployment (runs locally with Razorpay test mode)
- No subscription/recurring payments (covered conceptually as advanced topics)

## User Persona

A technically capable developer learning Razorpay integration deeply, with the goal of confidently implementing payments in a real production mobile app.

## Functional Requirements

| ID    | Requirement                                                                 |
|-------|-----------------------------------------------------------------------------|
| FR-1  | Display a list of demo products with prices                                 |
| FR-2  | "Buy Now" initiates order creation on backend                               |
| FR-3  | Backend creates Razorpay order and returns `order_id`                       |
| FR-4  | App opens Razorpay checkout with the `order_id`                             |
| FR-5  | On payment success, app sends payment details to backend for verification   |
| FR-6  | Backend verifies signature using `razorpay_order_id + razorpay_payment_id + secret` |
| FR-7  | Backend receives webhook for payment confirmation                           |
| FR-8  | App shows payment status (success/failure/pending)                          |
| FR-9  | Order history screen shows all past transactions                            |
| FR-10 | Refund initiation from the app (calls backend → Razorpay refund API)        |

## Non-Functional Requirements

| Category           | Requirement                                                                                  |
|--------------------|----------------------------------------------------------------------------------------------|
| **Security**       | API keys never in frontend code. Signature verification mandatory. HTTPS in production.      |
| **Reliability**    | Idempotent order creation. Webhook retries handled without duplicate processing.             |
| **Observability**  | Log every payment state transition on the backend.                                           |
| **Data Integrity** | Payment status determined by webhook OR server verification, never by client-side callback alone. |

## Payment Flow

```
┌─────────┐     ┌─────────────┐     ┌───────────────┐     ┌──────────┐
│  Mobile  │     │   Backend   │     │   Razorpay    │     │ Webhook  │
│   App    │     │  (Express)  │     │   Servers     │     │ Endpoint │
└────┬─────┘     └──────┬──────┘     └───────┬───────┘     └────┬─────┘
     │                  │                    │                  │
     │ 1. Buy Product   │                    │                  │
     │─────────────────>│                    │                  │
     │                  │ 2. Create Order    │                  │
     │                  │───────────────────>│                  │
     │                  │ 3. order_id        │                  │
     │                  │<───────────────────│                  │
     │ 4. order_id      │                    │                  │
     │<─────────────────│                    │                  │
     │                  │                    │                  │
     │ 5. Open Checkout (order_id)           │                  │
     │──────────────────────────────────────>│                  │
     │                  │                    │                  │
     │ 6. User pays (card/UPI/etc)           │                  │
     │                  │                    │                  │
     │ 7. payment_id + signature             │                  │
     │<──────────────────────────────────────│                  │
     │                  │                    │                  │
     │ 8. Verify payment│                    │                  │
     │─────────────────>│                    │                  │
     │                  │ 9. Verify signature│                  │
     │                  │ (HMAC SHA256)      │                  │
     │                  │                    │                  │
     │ 10. Confirmed    │                    │                  │
     │<─────────────────│                    │                  │
     │                  │                    │                  │
     │                  │    11. Webhook: payment.authorized     │
     │                  │<──────────────────────────────────────│
     │                  │                    │                  │
     │                  │ 12. Capture payment│                  │
     │                  │───────────────────>│                  │
     │                  │                    │                  │
     │                  │    13. Webhook: payment.captured       │
     │                  │<──────────────────────────────────────│
```

## System Architecture

### Tech Stack

- **Frontend:** React Native (Expo SDK 54) with Expo Router v6
- **Backend:** Node.js Express server
- **Payments:** Razorpay SDK (test mode)
- **Database:** In-memory store (learning phase), upgradeable to SQLite/PostgreSQL

### Folder Structure

```
test-app/
├── app/                          # Expo Router (frontend)
│   ├── _layout.tsx               # Root layout
│   ├── (tabs)/
│   │   ├── _layout.tsx           # Tab navigation
│   │   ├── index.tsx             # Product listing → "Buy" buttons
│   │   └── orders.tsx            # Order history
│   └── payment-result.tsx        # Payment success/failure screen
│
├── services/
│   └── api.ts                    # API client (calls backend)
│
├── types/
│   └── payment.ts                # TypeScript types for orders, payments
│
├── components/
│   └── product-card.tsx          # Product display component
│
├── backend/                      # Express server
│   ├── package.json
│   ├── .env                      # RAZORPAY_KEY_ID, RAZORPAY_KEY_SECRET, WEBHOOK_SECRET
│   ├── src/
│   │   ├── index.ts              # Express app entry
│   │   ├── routes/
│   │   │   ├── orders.ts         # POST /api/create-order
│   │   │   ├── verify.ts         # POST /api/verify-payment
│   │   │   └── webhooks.ts       # POST /api/webhooks/razorpay
│   │   ├── services/
│   │   │   └── razorpay.ts       # Razorpay SDK instance + helpers
│   │   └── db/
│   │       └── orders.ts         # In-memory store (or SQLite)
│   └── tsconfig.json
│
├── .env                          # EXPO_PUBLIC_RAZORPAY_KEY_ID, EXPO_PUBLIC_API_URL
└── package.json
```

### API Routes

| Method | Route                      | Purpose                    | Request Body                                                              | Response                          |
|--------|----------------------------|----------------------------|---------------------------------------------------------------------------|-----------------------------------|
| POST   | `/api/create-order`        | Create Razorpay order      | `{ productId, amount }`                                                   | `{ orderId, amount, currency }`   |
| POST   | `/api/verify-payment`      | Verify payment signature   | `{ razorpay_order_id, razorpay_payment_id, razorpay_signature }`          | `{ verified: true }`             |
| POST   | `/api/webhooks/razorpay`   | Receive Razorpay webhooks  | (Razorpay POST body)                                                      | `{ status: "ok" }`              |
| GET    | `/api/orders`              | List all orders            | —                                                                         | `[{ id, amount, status, ... }]`  |
| POST   | `/api/refund`              | Initiate refund            | `{ paymentId, amount? }`                                                  | `{ refundId, status }`          |

### Environment Variables

**Frontend (`.env` in project root):**
```
EXPO_PUBLIC_RAZORPAY_KEY_ID=rzp_test_xxxxx
EXPO_PUBLIC_API_URL=http://192.168.x.x:3001
```

**Backend (`backend/.env`):**
```
RAZORPAY_KEY_ID=rzp_test_xxxxx
RAZORPAY_KEY_SECRET=xxxxx
RAZORPAY_WEBHOOK_SECRET=xxxxx
PORT=3001
```

### Secrets Handling Rules

1. `key_secret` lives ONLY on the backend, ONLY in `.env`, ONLY in `.gitignore`
2. `key_id` is safe for the frontend — it's a public identifier
3. `webhook_secret` lives ONLY on the backend
4. In production: use a secrets manager (AWS SSM, Vault, etc.)

## Razorpay Lifecycle Overview

### Order Creation
- Backend calls Razorpay `POST /v1/orders` with amount (in paise), currency, and receipt
- Amount is looked up server-side — never trusted from the client
- Orders expire after 30 minutes by default
- An order can only be paid once; failed payments can retry on the same order

### Checkout
- App opens Razorpay SDK with `key_id` and `order_id`
- Razorpay renders its own payment UI (card, UPI, netbanking, etc.)
- On success, SDK returns: `razorpay_payment_id`, `razorpay_order_id`, `razorpay_signature`

### Authorization vs. Capture
- **Authorization** = money CAN be taken (reserved)
- **Capture** = money IS taken (deducted)
- Auto-capture (default) captures immediately after authorization
- Manual capture gives a window (up to 5 days) for review

### Signature Verification
```
signature = HMAC-SHA256(razorpay_order_id + "|" + razorpay_payment_id, key_secret)
```
Backend computes this and compares to the signature from the client. Non-negotiable in production.

### Webhooks
- HTTP POST from Razorpay to your server on payment events
- Key events: `payment.authorized`, `payment.captured`, `payment.failed`, `order.paid`, `refund.processed`
- Retried for 24 hours on failure
- Must respond with 2xx within 5 seconds
- Delivered at least once — handle idempotently

### Refunds
- Full or partial via `POST /v1/payments/{id}/refund`
- Take 5-7 business days to reflect
- Multiple partial refunds allowed up to original amount

## Testing Strategy

- **Unit tests:** Signature verification logic, order creation
- **Integration tests:** Full flow with Razorpay test mode
- **Manual tests:** Test card `4111 1111 1111 1111`, test UPI `success@razorpay`
- **Failure tests:** Decline card `4000 0000 0000 0002`, network drops, webhook failures

## Failure Scenarios

| Scenario                          | Behavior                                              | Handling                                      |
|-----------------------------------|-------------------------------------------------------|-----------------------------------------------|
| User closes checkout before paying | Order stays `created`, no charge                      | Show "payment cancelled"                      |
| Payment authorized, capture fails  | Webhook retry, manual capture possible                | Retry capture, alert on repeated failure       |
| Network drops after payment        | Webhook confirms; app unaware                         | Poll for status on app resume                  |
| Duplicate webhooks                 | Same event delivered multiple times                   | Idempotent processing via `payment_id` check   |
| Signature mismatch                 | Possible forgery attempt                              | Reject payment, log alert                      |

## Security Considerations

1. **Never expose `key_secret`** on the frontend — only `key_id` goes to the client
2. **Always verify signatures server-side** before confirming payment
3. **Validate webhook signatures** using the webhook secret
4. **Use HTTPS** for all backend communication in production
5. **Don't store card details** — Razorpay handles PCI compliance
6. **Validate amounts server-side** — never trust the price from the client
7. **Use raw body for webhook routes** — parsed JSON breaks signature verification
