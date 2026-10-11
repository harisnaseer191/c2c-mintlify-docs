# C2C Mobile API Reference

**Base URL**: `https://<host>/api`  
**Auth**: Bearer JWT from `POST /api/auth/member-login`  
**Content-Type**: `application/json`

> **Always use `https://`** — the server redirects HTTP (port 80) to HTTPS (port 443) with a 301. HTTP clients that follow the redirect convert POST to GET, causing a 405. Use HTTPS directly to avoid this.

---

## Table of Contents

1. [Authentication](#authentication)
2. [Member Profile](#member-profile)
3. [Orders](#orders)
4. [Realtime Order Delivery](#realtime-order-delivery)
5. [Availability](#availability)
6. [Payment Accounts](#payment-accounts)
7. [Recharge Requests](#recharge-requests)
8. [Withdrawal Requests](#withdrawal-requests)
9. [Platform Accounts (Public)](#platform-accounts)
10. [Error Format](#error-format)
11. [Pagination Format](#pagination-format)
12. [Enum Reference](#enum-reference)

---

## Authentication

### Login (Member)

```
POST /api/auth/member-login
```

No authentication required.

**Request**
```json
{
  "phoneNumber": "03001234567",
  "password": "YourPassword123"
}
```

**Response 200**
```json
{
  "data": {
    "accessToken": "eyJhbGciOiJIUzI1NiIs...",
    "refreshToken": "def50200...",
    "expiresAt": "2026-06-07T12:00:00Z"
  }
}
```

**Response 401** — Invalid credentials  
**Response 423** — Account locked (5 failed attempts → 15-minute lockout)

---

### Refresh Token

```
POST /api/auth/refresh
```

No authentication required.

**Request**
```json
{
  "refreshToken": "def50200..."
}
```

**Response 200** — Same shape as login response.

---

### Logout

```
POST /api/auth/logout
```

**Auth**: Bearer JWT required.

**Request**
```json
{
  "refreshToken": "def50200..."
}
```

**Response 200**
```json
{ "message": "Success." }
```

---

### Change Password

```
POST /api/auth/change-password
```

**Auth**: Bearer JWT required.

**Request**
```json
{
  "currentPassword": "OldPassword",
  "newPassword": "NewPassword123"
}
```

**Response 200**
```json
{ "message": "Success." }
```

---

## Member Profile

### Get My Profile

```
GET /api/member/me
```

**Auth**: Bearer JWT required.

**Response 200**
```json
{
  "data": {
    "id": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
    "userId": "9b1deb4d-3b7d-4bad-9bdd-2b0d7b3dcb6d",
    "fullName": "Ali Hassan",
    "phoneNumber": "03001234567",
    "country": "PK",
    "currency": "PKR",
    "payLimit": 50000.00,
    "drawLimit": 30000.00,
    "availableBalance": 12500.00,
    "frozenBalance": 2000.00,
    "isOnline": true,
    "collectionsEnabled": true,
    "status": "Active",
    "activePaymentAccounts": [
      {
        "id": "8c6b1c3a-...",
        "walletType": "JazzCash",
        "accountNumber": "03001234567",
        "accountHolderName": "Ali Hassan",
        "isActive": true,
        "status": "Approved",
        "accountTypeLabel": "JazzCash Account Number"
      }
    ],
    "createdAt": "2026-01-15T09:00:00Z"
  }
}
```

**Response 401** — Not authenticated  
**Response 404** — No member record found for this account

---

## Orders

### List My Orders

```
GET /api/member/orders
```

**Auth**: Bearer JWT required.

**Query Parameters**

| Parameter  | Type     | Default | Description                              |
|------------|----------|---------|------------------------------------------|
| `page`     | int      | 1       | Page number                              |
| `pageSize` | int      | 20      | Items per page                           |
| `statuses` | string[] | —       | Filter by status (repeatable query param) |
| `orderType`| string   | —       | `Deposit` or `Withdrawal`                |
| `fromDate` | datetime | —       | Filter from date (ISO 8601)              |
| `toDate`   | datetime | —       | Filter to date (ISO 8601)                |

**Example**
```
GET /api/member/orders?statuses=Assigned&statuses=Frozen&page=1&pageSize=10
```

**Response 200**
```json
{
  "data": {
    "items": [
      {
        "id": "abc123...",
        "memberId": "3fa85f64...",
        "memberName": "Ali Hassan",
        "paymentAccountId": "8c6b1c3a...",
        "accountNumber": "03001234567",
        "accountHolderName": "Ali Hassan",
        "walletType": "JazzCash",
        "orderType": "Deposit",
        "status": "AwaitingMemberConfirmation",
        "currency": "PKR",
        "amount": 5000.00,
        "commissionAmount": 125.00,
        "orderReference": "ORD-20260607-001",
        "txnRefNo": null,
        "memberTxnRefNo": null,
        "customerSubmittedTxnRef": "9022",
        "customerSubmittedTxnRefAt": "2026-06-07T10:01:30Z",
        "customerIdentifier": "CUST-1",
        "merchantApiKeyId": "…",
        "merchantApiKeyName": "Store key",
        "merchantName": "QA Store",
        "cancellationReason": null,
        "customerAccountNumber": null,
        "customerAccountHolderName": null,
        "customerWalletType": null,
        "accountKind": "AccountNumber",
        "accountTypeLabel": "JazzCash Account Number",
        "evidence": [
          {
            "id": "b8f0fcd9-…",
            "kind": "CustomerPaymentProof",
            "contentType": "image/png",
            "sizeBytes": 48211,
            "uploadedAt": "2026-06-07T10:01:20Z"
          }
        ],
        "allowedActions": ["Confirm", "Reject"],
        "assignedAt": "2026-06-07T10:00:00Z",
        "expiresAt": "2026-06-07T10:05:00Z",
        "confirmedAt": null,
        "cancelledAt": null,
        "frozenAt": null,
        "queuedAt": null,
        "createdAt": "2026-06-07T09:58:00Z",
        "updatedAt": "2026-06-07T10:00:00Z"
      }
    ],
    "totalCount": 42,
    "page": 1,
    "pageSize": 20,
    "totalPages": 3,
    "hasNextPage": true,
    "hasPreviousPage": false
  }
}
```

---

### The Order object — two parties, never mixed

Every order returned to the member app (list items, order detail, action responses) has the same fields. An order
has **two parties**, and their accounts are separate fields:

| Fields | Whose | Deposit | Withdrawal |
|---|---|---|---|
| `paymentAccountId`, `accountNumber`, `accountHolderName`, `walletType`, `accountKind`, `accountTypeLabel` | **the member** | the member's **receiving** account the customer pays into | the member's **source** account the payout was sent from — `null` until the member confirms |
| `customerAccountNumber`, `customerAccountHolderName`, `customerWalletType` | **the customer** | `null` | the customer's wallet the payout is **sent to** (`customerAccountHolderName` may be `null`) |

The app must show a withdrawal's destination from the `customer*` fields — never from `accountNumber`, which is the
member's own account.

| Field | Meaning |
|---|---|
| `accountKind` | `AccountNumber` \| `TillId` \| `null` — what identifies the member's account |
| `accountTypeLabel` | Display name of the member's account type, e.g. `"JazzCash Till ID"` (`null` when there is no account yet) |
| `commissionAmount` | The member's reward; `null` until it is known |
| `merchantName` | The merchant the order came from |
| `cancellationReason` | Why the order was rejected / cancelled, when it was |
| `customerSubmittedTxnRef` | **Deposit**: the last 4 digits of the TID the customer *says* they paid with. A claim to compare with your own wallet app — **not verification**. Only the order `status` says a payment was accepted |
| `customerSubmittedTxnRefAt` | When the customer submitted it |
| `memberTxnRefNo` | **Withdrawal**: the last 4 digits of the member's payout TID. **Deposit**: the member's optional reference |
| `evidence[]` | Images attached to the order: `{ id, kind, contentType, sizeBytes, uploadedAt }`. `kind` is `CustomerPaymentProof` (deposit, uploaded by the customer — not verification), `MemberPayoutProof` or `AbnormalPaymentEvidence` (withdrawal, uploaded by the member). Fetch the bytes from [Get Order Evidence](#get-order-evidence) |
| `allowedActions` | What the assigned member may do **now**: any of `Confirm`, `Reject`, `ReportAbnormalPayment`, `Cancel`. Render action buttons from this list — never from status comparisons. `Cancel` is legacy-only; the app does not offer it |

`allowedActions` by state:

| Order | Status | `allowedActions` |
|---|---|---|
| Deposit | `AwaitingMemberConfirmation` | `Confirm`, `Reject` |
| Withdrawal | `AwaitingMemberPayout` | `Confirm`, `Reject`, `ReportAbnormalPayment` |
| Either | `Assigned` / `Frozen` (legacy) | the above plus `Cancel` |
| Either | anything else (waiting for the customer, completed, closed, under admin review) | `[]` |

---

### Get Order by ID

```
GET /api/member/orders/{id}
```

**Auth**: Bearer JWT required.

The app calls this whenever Order Details gains focus and after an order hub event, so `allowedActions` is current.

**Response 200** — The [Order object](#the-order-object--two-parties-never-mixed) plus `auditLogs`  
**Response 403** — Order belongs to a different member  
**Response 404** — Order not found

---

### Confirm Order

```
POST /api/member/orders/{id}/confirm
```

**Auth**: Bearer JWT required.

**Request — deposit** (both fields optional — `{}` is valid)
```json
{
  "memberTxnRefNo": "5678"
}
```

`memberTxnRefNo` — optional for deposits: the member's own reference, e.g. the last digits of the incoming
transaction ID (at most 20 characters). The app sends it only when the member typed one. Compare
`customerSubmittedTxnRef` with your wallet app before confirming; it is the customer's claim, not proof.

**Request — withdrawal** (both fields required)
```json
{
  "sourcePaymentAccountId": "3f1c…",
  "memberTxnRefNo": "4821"
}
```

- `sourcePaymentAccountId` — the id of one of **your own** wallet accounts with `canFundWithdrawals: true` (from
  `GET /api/member/payment-accounts`). Any provider may pay any customer wallet — e.g. paying an Easypaisa
  customer from your JazzCash account — unless that route is configured as unsupported (then 400 with a message
  naming the route). An account you switched off for receiving deposits can still be used; a pending, rejected,
  admin-deactivated or on-hold one cannot.
- `memberTxnRefNo` — **exactly the last 4 digits** (0–9) of the outgoing transaction ID (TID). It is kept for
  reconciliation; it is not proof of payment.

Sending the same confirmation again (same account and TID, e.g. after a lost response) returns 200 with the
completed order without settling it twice.

**Response 200** — Confirmed order
```json
{
  "data": { "id": "abc123...", "status": "Completed", "paymentAccountId": "3f1c…", "memberTxnRefNo": "4821", ... }
}
```

**Response 400** — Order cannot be confirmed (`Confirm` is not in its `allowedActions` — the message says why), or a withdrawal completion is invalid:
  - `Select the wallet account you sent the payment from.`
  - `Enter the last 4 digits of the transaction ID (TID) of your payment.`
  - `The TID must be exactly the last 4 digits of the transaction ID (numbers only).`
  - `The selected source account is not one of your wallet accounts.`
  - `The selected source account is on hold by the administrator. Choose another account or contact support.`
  - `The selected source account is not approved for payments. Choose one of your approved accounts.`
  - `A JazzCash Account Number account cannot send to a Easypaisa wallet. Choose another of your accounts.` (only when that route is configured as unsupported)

**Response 409** — The order was updated concurrently; reload and try again  
**Response 403** — Order belongs to a different member  
**Response 404** — Order not found

---

### Reject Order

```
POST /api/member/orders/{id}/reject
```

**Auth**: Bearer JWT required.

The member declines the order: the deposit did not arrive, or they will not pay the withdrawal out. Nothing was
settled, so nothing is reversed.

**Request**
```json
{
  "reason": "Payment not received"
}
```

`reason` — **required**, at most 500 characters.

**Response 200** — Rejected order  
**Response 400** — `reason` is missing  
**Response 409** — `Reject` is not in the order's `allowedActions` (the message says why, e.g. "This order is already completed.")  
**Response 403** — Order belongs to a different member  
**Response 404** — Order not found

---

### Report Abnormal Payment

```
POST /api/member/orders/{id}/abnormal-payment
```

**Auth**: Bearer JWT required. **Withdrawals only.**

The payout could not be completed normally, or its outcome is unknown. The order goes to **`ManualReview`** for an
admin to decide — it is **not** failed or retried, because money may have left the member's account. The member
must not pay the order again.

**Request**
```json
{
  "reason": "TransactionStatusUncertain",
  "note": "Wallet app showed a timeout after I confirmed the transfer"
}
```

| Field | |
|---|---|
| `reason` | **required** — one of [`AbnormalPaymentReason`](#abnormalpaymentreason) |
| `note` | optional, at most 400 characters; **required when `reason` is `Other`** |

Idempotent: reporting again returns the order under review.

**Response 200** — Order, now `ManualReview` with `allowedActions: []`  
**Response 400** — Unknown `reason`, `note` too long, or `note` missing for `Other`  
**Response 409** — `ReportAbnormalPayment` is not in the order's `allowedActions` (e.g. a deposit, or an order already decided)  
**Response 403** — Order belongs to a different member  
**Response 404** — Order not found

---

### Upload Order Evidence

```
POST /api/member/orders/{id}/evidence
```

**Auth**: Bearer JWT required. `multipart/form-data`. Optional — a withdrawal can be confirmed without it.

| Part | |
|---|---|
| `kind` | `MemberPayoutProof` \| `AbnormalPaymentEvidence` |
| `file` | JPEG, PNG or WebP, at most 5 MB |

> The app does not upload evidence yet (it would need an image-picker dependency); the endpoint is listed for completeness.

---

### Get Order Evidence

```
GET /api/member/orders/{id}/evidence/{evidenceId}
```

**Auth**: Bearer JWT required — the request must carry the `Authorization` header (a plain image URL will not load).

Returns the image bytes of one item of the order's `evidence[]` with its `contentType`. In React Native:

```tsx
<Image source={{ uri, headers: { Authorization: `Bearer ${accessToken}` } }} />
```

(The app builds this with `getOrderEvidenceImageSource(orderId, evidenceId)` in `src/network/apis/member`.)

**Response 404** — Unknown evidence, or the order belongs to a different member

---

### Cancel Order (legacy)

```
POST /api/member/orders/{id}/cancel
```

**Auth**: Bearer JWT required.

Only for legacy orders in `Assigned` / `Frozen`. **The app no longer calls it**: for merchant orders it returns 409
("This order cannot be cancelled. Reject it if you will not process it, or report an abnormal payment if the payout
could not be completed."). Use [Reject Order](#reject-order) or [Report Abnormal Payment](#report-abnormal-payment).

**Request**
```json
{
  "reason": "Customer did not send payment"
}
```

**Response 200** — Cancelled order  
**Response 409** — Order cannot be cancelled  
**Response 403** — Order belongs to a different member  
**Response 404** — Order not found

---

## Realtime Order Delivery

When a merchant creates an order, the platform **assigns it to a member immediately** and delivers it to
that member's app. Every event is first stored in the member's **notification inbox** and then pushed
live over SignalR, so nothing is lost while the app is closed or offline — missed events are replayed
on the next connection.

### Who gets an order

A member is eligible when they are **online** (`isOnline`, set through `PATCH /api/member/availability`), have **Collections enabled**, have
an **Approved + active** payment account for the order's wallet type, the order currency matches theirs,
the amount is within their pay limit (deposits) / draw limit (withdrawals), they have a free slot under
their concurrent-order limit, and — for deposits — their **available wallet balance covers the amount**.
Among eligible members the least-loaded one is chosen, then the one who waited longest since their last
order (fair rotation).

Withdrawals the member does not pay out in time are reassigned to another member (the previous member
receives `OrderReassigned`, the new one `OrderAssigned`).

**Deposits and the customer checkout.** A deposit's account number is only shown to the customer when they
pick a wallet on the checkout page, and only an **online** member's account is ever shown: if the assigned
member is offline at that moment, the order is moved to an online member first (the previous member receives
`OrderReassigned`, the new one `OrderAssigned`). Once an account has been shown the order never moves — the
customer may already have paid — and the customer's transaction reference is accepted even if the member has
gone offline since.

**Going offline.** Only online members with Collections enabled are assigned new orders. Online / Collections
change **only** through `PATCH /api/member/availability` (or an admin) — closing the app, losing the network or
dropping the hub connection never switches a member offline; events sent meanwhile wait in the inbox. Going
offline or turning Collections off never takes orders away: you can still confirm or reject every order
assigned to you while offline. When you go offline with orders still open you receive `PendingOrdersReminder`.

### Order hub (SignalR)

```
wss://<host>/hubs/orders?access_token=<JWT>
```

Use WebSockets with `skipNegotiation: true` (works across API replicas without sticky sessions).

* **Connecting** replays up to 100 unacknowledged notifications (oldest first). It does **not** change the
  member's online status.
* **Disconnecting** does **not** mark the member offline — only `PATCH /api/member/availability` does.
* Call **`Heartbeat()`** every ~30 s — a connection without a heartbeat for 90 s is closed server-side
  (reconnect to keep receiving live events; the member's online status is unaffected).

| Direction | Method | Arguments |
|---|---|---|
| server → client | `OrderNotification` | `{ id, orderId, eventType, payload, createdAt }` |
| client → server | `AcknowledgeNotification` | `id: number` |
| client → server | `AcknowledgeNotifications` | `ids: number[]` |
| client → server | `Heartbeat` | — |

Acknowledge every notification you receive that has an inbox record (`id` > 0); unacknowledged records are
replayed on the next connection. A notification with **`id` 0** is live-only — it has no inbox record, so **do not
acknowledge it** (and do not de-duplicate by its id). `payload` is a JSON object **serialized as a string**
(camelCase, enums as strings).

**`eventType` values**

| Event | Sent when | `payload` |
|---|---|---|
| `OrderAssigned` | A new order (deposit or withdrawal) was assigned to you — also when one is reassigned *to* you | full order payload (below) |
| `OrderSubmitted` | The customer submitted their transaction reference — confirm the deposit | full order payload |
| `OrderExpired` | Your order expired or was escalated for manual review | `{ "orderId": "…" }` |
| `OrderReassigned` | The order was taken *away* from you and given to another member | `{ "orderId": "…" }` |
| `PendingOrdersReminder` | You went offline / turned Collections off (or your last device disconnected) with orders still open | `{ "pendingCount": 2, "orders": [ { "orderId", "orderReference", "orderType", "status", "amount", "currency", "expiresAt" } ] }` |
| `OrderClosedByAdmin` | An admin confirmed (and settled) or cancelled your order — it can no longer be confirmed or rejected | `{ "orderId", "orderReference", "orderType", "status", "amount", "currency", "commissionAmount" }` — `status` is `Confirmed`, `Cancelled` or `CancelledAuto` |
| `PaymentAccountUpdated` | An admin approved, rejected, held, released, deactivated, reactivated or switched one of your wallet accounts. **Live only: `id` is 0 and `orderId` is the empty GUID** — nothing to acknowledge, nothing is replayed | account payload (below) |

**`PaymentAccountUpdated` payload**
```json
{
  "accountId": "282ac528-56ce-4d80-a82d-9bbbfc132d0c",
  "change": "Approved",
  "account": "JazzCash Till ID ending 6443",
  "effectiveStatus": "SwitchedOff",
  "canReceiveDeposits": false,
  "canFundWithdrawals": true,
  "message": null,
  "rejectionReason": null
}
```

`change` is `Approved`, `Rejected`, `Held`, `HoldReleased`, `Deactivated`, `Reactivated`, `SwitchedOn` or
`SwitchedOff` (the last two only when an admin flips the switch — the member's own toggle is not echoed). `message`
is the fixed member-safe hold text while the account is on hold. On this event the app refreshes
`GET /api/member/payment-accounts` and shows a toast such as "Your JazzCash Till ID ending 6443 was approved".

**Full order payload**
```json
{
  "orderId": "3fa85f64-5717-4562-b3fc-2c963f66afa6",
  "orderReference": "ORD-20261007120000-ab12",
  "orderType": "Deposit",
  "status": "Queued",
  "amount": 1000.0,
  "currency": "PKR",
  "paymentAccountId": "…",
  "paymentAccountWalletType": "JazzCash",
  "paymentAccountNumber": "03001234567",
  "customerSubmittedTxnRef": null,
  "customerAccountNumber": null,
  "customerWalletType": null,
  "assignedAt": "2026-10-07T12:00:00Z",
  "expiresAt": null,
  "paymentPageExpiresAt": "2026-10-07T12:15:00Z",
  "createdAt": "2026-10-07T12:00:00Z"
}
```

For withdrawals, `customerAccountNumber` / `customerWalletType` are the customer account to pay out to,
`expiresAt` is the payout deadline, and `paymentAccount*` are `null` (the member picks the source account when
confirming). `orderType` is always present, so a notification can be labelled Deposit / Withdrawal.

### Get Pending Notifications & Open Orders

```
GET /api/member/orders/pending
```

**Auth**: Bearer JWT required. REST recovery path (e.g. on app start): unacknowledged notifications (latest 50)
plus **every open order assigned to you**, including deposits still awaiting customer checkout.

**Response 200**
```json
{
  "data": {
    "notifications": [
      { "id": 42, "orderId": "…", "eventType": "OrderAssigned", "payloadJson": "{…}", "createdAt": "…" }
    ],
    "activeOrders": [ /* Order objects, with allowedActions */ ]
  }
}
```

### Acknowledge Notifications

```
POST /api/member/orders/notifications/ack
```

**Auth**: Bearer JWT required. REST equivalent of the hub acknowledgement. Ids that are unknown, already
acknowledged or belong to another member are ignored.

**Request**
```json
{ "notificationIds": [42, 43] }
```

**Response 200**
```json
{ "data": { "acknowledged": 2 } }
```

**Response 400** — empty list or more than 200 ids

---

## Availability

### Update Availability

```
PATCH /api/member/availability
```

**Auth**: Bearer JWT required.

Toggle your online/offline status or enable/disable order collection.

> **Note**: Enabling `collectionsEnabled` requires at least one approved payment account.

**Request**
```json
{
  "isOnline": true,
  "collectionsEnabled": true
}
```

Both fields are optional — send only the ones you want to change.

**Response 200**
```json
{ "data": true }
```

**Response 422** — Cannot enable collections without an approved payment account

---

## Payment Accounts

A member registers wallet accounts of four types — a **provider** (JazzCash / Easypaisa) and what identifies the
account (**Account Number** / **Till ID**). Every new or amended account is `Pending` until an admin approves it.
Approval does not switch the account on: it is `SwitchedOff` until the member turns "receive deposits" on.

### The PaymentAccount object

```json
{
  "id": "460c3924-6c2d-4747-94c7-808c40f325d4",
  "memberId": "3fa85f64...",
  "memberName": "Ali Hassan",
  "walletType": "JazzCash",
  "provider": "JazzCash",
  "accountKind": "AccountNumber",
  "accountTypeLabel": "JazzCash Account Number",
  "accountNumber": "03001234567",
  "accountHolderName": "Ali Hassan",
  "status": "Approved",
  "isActive": true,
  "effectiveStatus": "Active",
  "canReceiveDeposits": true,
  "canFundWithdrawals": true,
  "isOnHold": false,
  "holdMessage": null,
  "rejectionReason": null,
  "deactivationReason": null,
  "addedAt": "2026-01-15T09:00:00Z",
  "approvedByUserId": "...",
  "approvedByName": "Admin User",
  "approvedAt": "2026-01-16T10:00:00Z",
  "totalOrdersProcessed": 47,
  "lastOrderDate": "2026-06-06T18:30:00Z",
  "createdAt": "2026-01-15T09:00:00Z",
  "updatedAt": "2026-01-16T10:00:00Z"
}
```

| Field | Meaning |
|---|---|
| `provider` | `JazzCash` \| `Easypaisa` (`Bank` only on legacy accounts) |
| `accountKind` | `AccountNumber` \| `TillId` (`null` only on a JazzCash account registered before kinds existed) |
| `accountTypeLabel` | Display name, e.g. `"Easypaisa Till ID"` |
| `walletType` | The payment channel the account is matched under (`JazzCash`, `Easypaisa`, `TillID` = Easypaisa Till ID) |
| `status` | The approval state: [`PaymentAccountStatus`](#paymentaccountstatus) |
| `isActive` | The member's own receive-deposits switch (meaningful once `status` is `Approved`) |
| `effectiveStatus` | **Backend-computed** state to display: [`PaymentAccountEffectiveStatus`](#paymentaccounteffectivestatus) |
| `canReceiveDeposits` | Whether deposits can be assigned to it right now |
| `canFundWithdrawals` | Whether it can be chosen as the source account when confirming a withdrawal |
| `isOnHold` / `holdMessage` | An admin placed the account on hold; `holdMessage` is the fixed member-safe text to show verbatim (`null` when not held). The admin's internal reason is never sent to the member app |
| `rejectionReason` / `deactivationReason` | The admin's reason, for the member to read |

Clients show `effectiveStatus` and use `canReceiveDeposits` / `canFundWithdrawals` as given — they never work
eligibility out from `status` and `isActive`.

---

### List Registrable Account Types

```
GET /api/member/payment-account-types
```

**Auth**: Bearer JWT required.

The account types a member can register, with each identifier's label and format hint.

**Response 200**
```json
{
  "data": [
    {
      "provider": "JazzCash",
      "accountKind": "AccountNumber",
      "walletType": "JazzCash",
      "label": "JazzCash Account Number",
      "identifierLabel": "Account number",
      "identifierHint": "Enter the 11-digit mobile account number, for example 03001234567."
    },
    {
      "provider": "JazzCash",
      "accountKind": "TillId",
      "walletType": "JazzCash",
      "label": "JazzCash Till ID",
      "identifierLabel": "Till ID",
      "identifierHint": "A Till ID is 4 to 12 digits, numbers only."
    },
    {
      "provider": "Easypaisa",
      "accountKind": "AccountNumber",
      "walletType": "Easypaisa",
      "label": "Easypaisa Account Number",
      "identifierLabel": "Account number",
      "identifierHint": "Enter the 11-digit mobile account number, for example 03001234567."
    },
    {
      "provider": "Easypaisa",
      "accountKind": "TillId",
      "walletType": "TillID",
      "label": "Easypaisa Till ID",
      "identifierLabel": "Till ID",
      "identifierHint": "A Till ID is 4 to 12 digits, numbers only."
    }
  ]
}
```

The app keeps a built-in copy of these four (`FALLBACK_ACCOUNT_TYPES`) for when the call fails.

---

### List My Payment Accounts

```
GET /api/member/payment-accounts
```

**Auth**: Bearer JWT required.

**Response 200** — `{ "data": [ PaymentAccount, … ] }`. Removed (archived) accounts are not returned.

---

### Add Payment Account

```
POST /api/member/payment-accounts
```

**Auth**: Bearer JWT required.

MemberId is derived from JWT — do not send it in the body.

**Request**
```json
{
  "provider": "Easypaisa",
  "accountType": "TillId",
  "accountNumber": "764310",
  "accountHolderName": "Ali Hassan"
}
```

| Field | |
|---|---|
| `provider` | `JazzCash` \| `Easypaisa` |
| `accountType` | `AccountNumber` \| `TillId` |
| `accountNumber` | The identifier. **Account number**: 11 digits `03XXXXXXXXX` (`+92…` / `92…` forms and spaces/dashes are normalised). **Till ID**: 4–12 digits |
| `accountHolderName` | Required, at most 200 characters |

(`walletType` alone is still accepted from older app versions.)

**Response 200** — Created account: `status: "Pending"`, `isActive: false`, `effectiveStatus: "PendingApproval"`  
**Response 400** — The identifier does not match the type (the message is that type's `identifierHint`), an unknown type, or a duplicate ("You already have an account with this type and number.")  
**Response 404** — Member not found

---

### Update Payment Account

```
PUT /api/member/payment-accounts/{id}
```

**Auth**: Bearer JWT required. Same body as [Add Payment Account](#add-payment-account).

Amends one of the member's own accounts. **Any change returns it to `Pending`** (switched off, approval
invalidated) until an admin approves the new details; sending the same details changes nothing.

**Response 200** — Updated account  
**Response 400** — Same validation as adding  
**Response 409** — The account is on hold (the hold message), or it has orders in progress

---

### Remove Payment Account

```
DELETE /api/member/payment-accounts/{id}
```

**Auth**: Bearer JWT required.

Archives the account: it disappears from the member's list and can no longer receive deposits or fund withdrawals.

**Response 200**  
**Response 409** — The account is on hold, or "This account has orders in progress. Remove it once they are finished."

---

### Switch Receive-Deposits On / Off

```
PUT /api/member/payment-accounts/{id}/toggle-status
```

**Auth**: Bearer JWT required.

**Request**
```json
{ "isActive": true }
```

The member's own switch. Switching off stops new deposits; the account can still fund withdrawals.

**Response 200**  
**Response 400** — The account is not `Approved`  
**Response 409** — Switching **on** an account that is on hold (the hold message)

---

## Recharge Requests

### List My Recharge Requests

```
GET /api/member/recharge-requests
```

**Auth**: Bearer JWT required.

**Query Parameters**

| Parameter  | Type | Default | Description    |
|------------|------|---------|----------------|
| `page`     | int  | 1       | Page number    |
| `pageSize` | int  | 20      | Items per page |

**Response 200**
```json
{
  "data": {
    "items": [
      {
        "id": "abc123...",
        "memberId": "3fa85f64...",
        "memberName": "Ali Hassan",
        "phoneNumber": "03001234567",
        "amount": 10000.00,
        "status": "Approved",
        "requestedAt": "2026-06-05T14:00:00Z",
        "reviewedAt": "2026-06-05T15:30:00Z",
        "transactionIdLast4": "4821",
        "platformAccountWalletType": "JazzCash",
        "platformAccountNumber": "03001112233",
        "platformAccountHolderName": "C2C Collections"
      }
    ],
    "totalCount": 8,
    "page": 1,
    "pageSize": 20,
    "totalPages": 1,
    "hasNextPage": false,
    "hasPreviousPage": false
  }
}
```

---

### Submit Recharge Request

```
POST /api/member/recharge-requests
```

**Auth**: Bearer JWT required.

**Request**
```json
{
  "amount": 10000.00,
  "platformRechargeAccountId": "9d4e1a2b-...",
  "transactionIdLast4": "4821",
  "sourceAccountDetails": "JazzCash 03001234567",
  "transactionId": "88214821",
  "screenshotProof": "base64encodedimage..."
}
```

- `platformRechargeAccountId` — **required**. The active platform account you sent money to (see [Platform Accounts](#platform-accounts)). Its wallet type, number and holder name are stored on the request at submission.
- `transactionIdLast4` — **required**. Exactly the last 4 digits of the wallet transaction ID (TID); finance matches it against the platform account statement.
- `sourceAccountDetails` — **required**. The account you paid from.
- `transactionId`, `screenshotProof` — optional. If the full `transactionId` is sent it must end in `transactionIdLast4`.

**Response 200** — Created recharge request (status: `Pending`)  
**Response 400** — Validation error

---

## Withdrawal Requests

### List My Withdrawal Requests

```
GET /api/member/withdrawal-requests
```

**Auth**: Bearer JWT required.

**Query Parameters**

| Parameter  | Type | Default | Description    |
|------------|------|---------|----------------|
| `page`     | int  | 1       | Page number    |
| `pageSize` | int  | 20      | Items per page |

**Response 200**
```json
{
  "data": {
    "items": [
      {
        "id": "abc123...",
        "memberId": "3fa85f64...",
        "memberName": "Ali Hassan",
        "phoneNumber": "03001234567",
        "amount": 5000.00,
        "walletType": "JazzCash",
        "status": "Processed",
        "requestedAt": "2026-06-04T11:00:00Z",
        "processedAt": "2026-06-04T13:45:00Z"
      }
    ],
    "totalCount": 3,
    "page": 1,
    "pageSize": 20,
    "totalPages": 1,
    "hasNextPage": false,
    "hasPreviousPage": false
  }
}
```

---

### Submit Withdrawal Request

```
POST /api/member/withdrawal-requests
```

**Auth**: Bearer JWT required.

**Request**
```json
{
  "amount": 5000.00,
  "accountNumber": "03001234567",
  "walletType": "JazzCash"
}
```

**Response 200** — Created withdrawal request (status: `Pending`)  
**Response 400** — Insufficient balance or amount exceeds draw limit  
**Response 422** — Business rule violation

---

## Platform Accounts

### List Active Platform Accounts

```
GET /api/platform-accounts/active
```

No authentication required. Shows the platform's recharge accounts that members should send money to when submitting a recharge request.

**Response 200**
```json
{
  "data": [
    {
      "id": "9d4e1a2b-...",
      "walletType": "JazzCash",
      "accountNumber": "03009998877",
      "accountHolderName": "C2C Platform",
      "isActive": true
    }
  ]
}
```

---

## Error Format

All error responses use this shape:

```json
{
  "error": "Human-readable error message.",
  "code": "optional.code"
}
```

`error` is written for the member and is what the app shows; `code` is optional.

| HTTP Status | Meaning                                        |
|-------------|------------------------------------------------|
| 400         | Validation error or business rule violation    |
| 401         | Missing or invalid Bearer token                |
| 403         | Authenticated but not authorized (wrong owner) |
| 404         | Resource not found                             |
| 409         | The action is not allowed in the current state (order already decided, account on hold or in use) |
| 422         | Unprocessable — precondition not met           |
| 500         | Internal server error                          |

---

## Pagination Format

All list endpoints return the same paginated envelope:

```json
{
  "data": {
    "items": [...],
    "totalCount": 42,
    "page": 1,
    "pageSize": 20,
    "totalPages": 3,
    "hasNextPage": true,
    "hasPreviousPage": false
  }
}
```

---

## Enum Reference

### `OrderStatus`

| Value                        | Description                                                                 |
|------------------------------|-----------------------------------------------------------------------------|
| `Awaiting`                   | Not assigned to a member yet (e.g. a withdrawal waiting for the customer's account details or for an eligible member) |
| `Queued`                     | Merchant deposit assigned to you, waiting for the customer to open checkout (legacy orders: waiting for capacity) |
| `AwaitingCustomerPayment`    | Customer is on the checkout page and should pay into your account           |
| `AwaitingMemberConfirmation` | Customer submitted their transaction reference — confirm the deposit       |
| `AwaitingMemberPayout`       | Withdrawal assigned to you — pay out to the customer account                |
| `Completed`                  | Confirmed by you (terminal)                                                 |
| `Rejected`                   | Rejected by you                                                             |
| `Expired`                    | Customer did not pay before checkout expired                                |
| `ManualReview`               | With an admin: escalated (e.g. not confirmed in time) or reported as an abnormal payment. Do not pay it again |
| `Assigned`                   | Legacy: assigned — awaiting confirmation                                    |
| `Frozen`                     | Legacy: freeze time passed — still awaiting confirmation                    |
| `Confirmed`                  | Legacy: confirmed (terminal)                                                |
| `Cancelled` / `CancelledAuto` / `CancelledTimeout` | Cancelled by member/admin, automatically, or timed out |

### `MemberOrderAction` (`allowedActions`)

| Value                   | Description |
|-------------------------|-------------|
| `Confirm`               | Deposit: the customer's payment arrived. Withdrawal: the member sent the payout |
| `Reject`                | The member declines the order (reason required) |
| `ReportAbnormalPayment` | Withdrawal only: the payout failed or its outcome is unknown — goes to admin review |
| `Cancel`                | Legacy orders only (`Assigned` / `Frozen`). Not offered by the app |

### `AbnormalPaymentReason`

| Value                         | Shown in the app as |
|-------------------------------|---------------------|
| `CustomerAccountIncorrect`    | The customer's account number is incorrect |
| `PaymentCouldNotBeSent`       | The payment could not be sent |
| `DestinationWalletRestricted` | The customer's wallet cannot receive money |
| `ProviderUnavailable`         | The wallet provider is unavailable |
| `TransactionStatusUncertain`  | I paid, but I am not sure it went through |
| `Other`                       | Something else (`note` required) |

### `OrderEvidenceKind`

| Value                     | Uploaded by | Description |
|---------------------------|-------------|-------------|
| `CustomerPaymentProof`    | customer    | Deposit: the customer's proof of payment — a claim, not verification |
| `MemberPayoutProof`       | member      | Withdrawal: proof of the payout |
| `AbnormalPaymentEvidence` | member      | Withdrawal: evidence attached to an abnormal-payment report |

### `OrderType`

| Value        | Description                          |
|--------------|--------------------------------------|
| `Deposit`    | The customer pays into the member's receiving account |
| `Withdrawal` | The member sends money to the customer's wallet       |

### `WalletType`

| Value        | Description |
|--------------|-------------|
| `JazzCash`   | JazzCash    |
| `Easypaisa`  | Easypaisa   |
| `TillID`     | Easypaisa Till ID |
| `BankAccount`| Bank Account|

### `Currency`

| Value | Description      |
|-------|------------------|
| `PKR` | Pakistani Rupee  |

### `RequestStatus` (Recharge & Withdrawal Requests)

| Value      | Description                        |
|------------|------------------------------------|
| `Pending`  | Awaiting finance team review       |
| `Approved` | Approved — balance credited        |
| `Declined` | Rejected by finance team           |
| `Processed`| Withdrawal processed and sent      |

### `PaymentAccountStatus`

| Value      | Description                                    |
|------------|------------------------------------------------|
| `Pending`  | Awaiting admin approval (new, or amended)      |
| `Approved` | Approved by an admin                           |
| `Rejected` | Rejected by admin (`rejectionReason`)          |
| `Inactive` | Deactivated by admin (`deactivationReason`)    |

### `PaymentAccountEffectiveStatus`

Backend-computed from `status`, `isActive`, the hold and archival. This is what the app shows.

| Value                | Shown in the app as | Receives deposits | Funds withdrawals |
|----------------------|---------------------|-------------------|-------------------|
| `Active`             | Active              | yes               | yes               |
| `SwitchedOff`        | Switched off        | no                | yes               |
| `OnHold`             | On Hold by Admin    | no                | no                |
| `PendingApproval`    | Pending approval    | no                | no                |
| `Rejected`           | Rejected            | no                | no                |
| `DeactivatedByAdmin` | Disabled by admin   | no                | no                |
| `Archived`           | Removed (not returned to the member app) | no | no           |

The two right-hand columns are for orientation only — read `canReceiveDeposits` / `canFundWithdrawals`.

### `UserStatus`

| Value      | Description          |
|------------|----------------------|
| `Active`   | Account is active    |
| `Inactive` | Account is suspended |
| `Locked`   | Account is locked    |
