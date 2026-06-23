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
   - [List My Orders](#list-my-orders)
   - [Get Order by ID](#get-order-by-id)
   - [Confirm Order](#confirm-order)
   - [Reject Order](#reject-order)
   - [Cancel Order](#cancel-order)
4. [Availability](#availability)
5. [Payment Accounts](#payment-accounts)
6. [Recharge Requests](#recharge-requests)
7. [Withdrawal Requests](#withdrawal-requests)
8. [Platform Accounts (Public)](#platform-accounts)
9. [Error Format](#error-format)
10. [Pagination Format](#pagination-format)
11. [Enum Reference](#enum-reference)

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
        "status": "Approved"
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
        "status": "Assigned",
        "currency": "PKR",
        "amount": 5000.00,
        "commissionAmount": 125.00,
        "orderReference": "ORD-20260607-001",
        "merchantId": "f7b3c2a1-...",
        "merchantName": "Acme Games",
        "merchantOrderRef": "ORD-20260618-7890",
        "txnRefNo": "1234",
        "memberTxnRefNo": null,
        "customerSubmittedTxnRef": "5678",
        "customerReceiptImageUrl": null,
        "customerAccountNumber": null,
        "customerWalletType": null,
        "customerIdentifier": "player_12345",
        "merchantApiKeyId": "d2e1f0a9-...",
        "merchantApiKeyName": "Game Checkout Key",
        "cancellationReason": null,
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

**New fields in the order object**:

| Field | Type | Description |
|-------|------|-------------|
| `merchantId` | uuid\|null | The merchant who created this order |
| `merchantName` | string\|null | Merchant's display name |
| `merchantOrderRef` | string\|null | Merchant's own order reference number |
| `customerSubmittedTxnRef` | string\|null | Last 4 digits submitted by the customer on the payment page (deposit orders) |
| `customerReceiptImageUrl` | string\|null | Optional receipt image URL uploaded by the customer |
| `customerAccountNumber` | string\|null | **Withdrawal orders only** — the customer's account number you must send money to |
| `customerWalletType` | string\|null | **Withdrawal orders only** — the wallet type for `customerAccountNumber` |

---

### Get Order by ID

```
GET /api/member/orders/{id}
```

**Auth**: Bearer JWT required.

**Response 200** — Full order detail including audit logs  
**Response 403** — Order belongs to a different member  
**Response 404** — Order not found

---

### Confirm Order

```
POST /api/member/orders/{id}/confirm
```

**Auth**: Bearer JWT required.

Use this when you have **received** money (deposit) or **sent** money (withdrawal) and want to approve the order.

**Request**
```json
{
  "memberTxnRefNo": "5678"
}
```

`memberTxnRefNo` — The last 4–6 digits of the transaction ID from your wallet app.
- **Deposit orders**: the incoming transaction ID (the money the customer sent you)
- **Withdrawal orders**: the outgoing transaction ID (the money you sent to the customer)

This field is optional but strongly recommended for audit purposes.

**Response 200** — Confirmed order
```json
{
  "data": { "id": "abc123...", "status": "Confirmed", ... }
}
```

**Response 400** — Order cannot be confirmed (wrong status)  
**Response 403** — Order belongs to a different member  
**Response 404** — Order not found

---

### Reject Order

```
POST /api/member/orders/{id}/reject
```

**Auth**: Bearer JWT required.

Use this when you did **not** receive the money (deposit order) and want to reject the order. This is different from cancellation — rejection indicates the customer's payment was not received in your account.

**Request**
```json
{
  "reason": "No payment received in my JazzCash account"
}
```

`reason` — Optional. A brief note explaining why you are rejecting. Shown to the admin and returned to the merchant in their status polling.

**Response 200** — Rejected order
```json
{
  "data": { "id": "abc123...", "status": "Rejected", ... }
}
```

**Response 400** — Order cannot be rejected in its current state  
**Response 403** — Order belongs to a different member  
**Response 404** — Order not found

---

### Cancel Order

```
POST /api/member/orders/{id}/cancel
```

**Auth**: Bearer JWT required.

System-level cancellation (e.g. freeze timeout). For rejecting a payment not received, use [Reject Order](#reject-order) instead.

**Request**
```json
{
  "reason": "Customer did not respond"
}
```

**Response 200** — Cancelled order  
**Response 400** — Order cannot be cancelled (wrong status)  
**Response 403** — Order belongs to a different member  
**Response 404** — Order not found

---

## Availability

### Update Availability

```
PATCH /api/member/availability
```

**Auth**: Bearer JWT required.

Toggle your online/offline status or enable/disable order collection.

> **Note**: Enabling `collectionsEnabled` requires at least one `Approved` and `IsActive` payment account.

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

### List My Payment Accounts

```
GET /api/member/payment-accounts
```

**Auth**: Bearer JWT required.

**Response 200**
```json
{
  "data": [
    {
      "id": "8c6b1c3a...",
      "memberId": "3fa85f64...",
      "memberName": "Ali Hassan",
      "walletType": "JazzCash",
      "accountNumber": "03001234567",
      "accountHolderName": "Ali Hassan",
      "status": "Approved",
      "isActive": true,
      "addedAt": "2026-01-15T09:00:00Z",
      "approvedByUserId": "...",
      "approvedByName": "Admin User",
      "approvedAt": "2026-01-16T10:00:00Z",
      "rejectionReason": null,
      "deactivationReason": null,
      "totalOrdersProcessed": 47,
      "lastOrderDate": "2026-06-06T18:30:00Z",
      "createdAt": "2026-01-15T09:00:00Z",
      "updatedAt": "2026-01-16T10:00:00Z"
    }
  ]
}
```

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
  "walletType": "Easypaisa",
  "accountNumber": "03211234567",
  "accountHolderName": "Ali Hassan"
}
```

**Response 200** — Created payment account (status: `Pending` until approved by admin)  
**Response 400** — Duplicate account number for the same wallet type  
**Response 404** — Member not found

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
        "reviewedAt": "2026-06-05T15:30:00Z"
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
  "screenshotProof": "base64encodedimage...",
  "transactionId": "TXN20260607001",
  "sourceAccountDetails": "JazzCash 03001234567",
  "amount": 10000.00,
  "platformRechargeAccountId": "9d4e1a2b-..."
}
```

- At least one of `screenshotProof` or `transactionId` is required.
- `platformRechargeAccountId` — the platform account you sent money to (see [Platform Accounts](#platform-accounts)).

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
  "error": "Human-readable error message."
}
```

| HTTP Status | Meaning                                        |
|-------------|------------------------------------------------|
| 400         | Validation error or business rule violation    |
| 401         | Missing or invalid Bearer token                |
| 403         | Authenticated but not authorized (wrong owner) |
| 404         | Resource not found                             |
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

| Value              | Description                                                          |
|--------------------|----------------------------------------------------------------------|
| `Queued`           | Waiting to be assigned to a member                                   |
| `Assigned`         | Assigned to a member — awaiting member confirmation                  |
| `Frozen`           | Freeze time expired — still awaiting member confirmation             |
| `Confirmed`        | Confirmed by member — payment received (deposit) or sent (withdrawal)|
| `Rejected`         | Member rejected — did not receive payment in their account           |
| `Cancelled`        | Cancelled by admin                                                   |
| `CancelledAuto`    | Auto-cancelled after freeze timeout                                  |
| `CancelledTimeout` | Cancelled due to awaiting timeout                                    |

### `OrderType`

| Value        | Description                                                                   |
|--------------|-------------------------------------------------------------------------------|
| `Deposit`    | Customer sends money to the member's wallet account                           |
| `Withdrawal` | Member sends money to the customer's account (`customerAccountNumber` field)  |

### `WalletType`

| Value        | Description |
|--------------|-------------|
| `JazzCash`   | JazzCash    |
| `Easypaisa`  | Easypaisa   |
| `TillID`     | TillID      |
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
| `Pending`  | Awaiting admin approval                        |
| `Approved` | Approved — eligible to receive orders          |
| `Rejected` | Rejected by admin                              |
| `Inactive` | Deactivated                                    |

### `UserStatus`

| Value      | Description          |
|------------|----------------------|
| `Active`   | Account is active    |
| `Inactive` | Account is suspended |
| `Locked`   | Account is locked    |
