# Credipay Payment API — Card & USSD Integration Guide

> **Base URL:** `CONTACT_SUPPORT`
>
> All endpoints require authentication. See [Authentication & Headers](#2-authentication--headers).

---

## Table of Contents

1. [Overview](#1-overview)
2. [Authentication & Headers](#2-authentication--headers)
3. [Shared Types & Enums](#3-shared-types--enums)
4. [Card Payments](#4-card-payments)
   - 4.1 [Step 1: Initiate Payment](#41-step-1-initiate-payment)
   - 4.2 [Step 2: Handle Authorization Type](#42-step-2-handle-authorization-type)
   - 4.3 [Step 3: Complete Payment (OTP/PIN)](#43-step-3-complete-payment-otppin)
   - 4.4 [Step 4: Verify Payment Status](#44-step-4-verify-payment-status)
   - 4.5 [Card Flow Diagram](#45-card-flow-diagram)
5. [USSD Payments](#5-ussd-payments)
   - 5.1 [Step 1: Initiate Payment](#51-step-1-initiate-payment)
   - 5.2 [Step 2: Customer Dials USSD Code](#52-step-2-customer-dials-ussd-code)
   - 5.3 [Step 3: Verify Payment Status (Polling)](#53-step-3-verify-payment-status-polling)
   - 5.4 [USSD Flow Diagram](#54-ussd-flow-diagram)
6. [Payment Popup Integration](#6-payment-popup-integration)
7. [Webhooks](#7-webhooks)
8. [Payment Status Reference](#8-payment-status-reference)
9. [Error Handling & Best Practices](#9-error-handling--best-practices)
10. [Quick Start Examples](#10-quick-start-examples)

---

## 1. Overview

Credipay's payment gateway supports two electronic collection channels:

| Channel | Value | How It Works |
|---------|-------|-------------|
| **Card** | `card` | Customer's card details are submitted via API. Multi-step authorization: Card → PIN → OTP → 3D Secure as needed. |
| **USSD** | `ussd` | API returns bank-specific USSD dial strings. Customer dials on their phone. You poll for completion. |

Both channels use the **same three endpoints** — the `paymentChannel` field in the request body determines which flow is used.

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/v1/Gateway/initiate-payment` | POST | Start a new payment |
| `/api/v1/Gateway/complete-payment` | POST | Submit OTP or PIN for card payments |
| `/api/v1/Gateway/verify-payment-status` | POST | Check payment outcome |

---

## 2. Authentication & Headers

Credipay uses **API key authentication** — no Bearer tokens or OAuth needed.

Your API keys are available in the Credipay merchant dashboard. You receive two pairs (Test and Live), each with a **Secret Key** and a **Public Key**.

### Gateway Endpoints (server-to-server)

| Header | Value | Required |
|--------|-------|----------|
| `x-api-key` | Your **Secret Key** | Yes |
| `Content-Type` | `application/json` | Yes |

Secret keys contain `sk` in the value (e.g., `Test_sk_xxxxxxxx` or `Live_sk_xxxxxxxx`).

> **Important:** Secret keys must never be exposed in client-side code. Use these only from your backend server.

### Public Endpoints (popup/frontend)

| Header | Value | Required |
|--------|-------|----------|
| `x-api-key` | Your **Public Key** | Yes |
| `Content-Type` | `application/json` | Yes |

Public keys contain `pk` in the value (e.g., `Test_pk_xxxxxxxx` or `Live_pk_xxxxxxxx`).

### Environment Selection

The API automatically determines whether to use the **Test** or **Live** environment based on your key prefix:

| Key Prefix | Environment |
|-----------|-------------|
| `Test_sk_...` / `Test_pk_...` | Test (sandbox) |
| `Live_sk_...` / `Live_pk_...` | Live (production) |

---

## 3. Shared Types & Enums

### PaymentChannel

| Value | Description |
|-------|-------------|
| `card` | Card payment |
| `ussd` | USSD payment |
| `transfer` | Bank transfer |

### PaymentRoute

| Value | Description |
|-------|-------------|
| `direct` | Direct API integration (default) |
| `pop_up` | Popup/widget integration |
| `link` | Payment link |

### API Response Format

All responses follow this structure:

```json
{
  "isSuccess": true,
  "error": "",
  "message": "Human-readable message",
  "responseCode": "00",
  "value": { }
}
```

| Field | Type | Description |
|-------|------|-------------|
| `isSuccess` | `boolean` | `true` if the operation succeeded |
| `error` | `string` | Error description when `isSuccess` is `false` |
| `message` | `string` | Human-readable status or instruction |
| `responseCode` | `string` | `"00"` = success, `"400"` = bad request |
| `value` | `object` | Response payload (varies per endpoint) |

---

## 4. Card Payments

Card payments are a **multi-step** process because different cards require different authorization methods (PIN, OTP, 3D Secure).

### 4.1 Step 1: Initiate Payment

**`POST /api/v1/Gateway/initiate-payment`**

#### Request:

```json
{
  "amount": 1000.00,
  "firstName": "John",
  "lastName": "Doe",
  "email": "john@example.com",
  "phone": "08012345678",
  "paymentReference": "YOUR-UNIQUE-REF-001",
  "paymentChannel": "card",
  "currencyCode": "NGN",
  "paymentRoute": "direct",
  "cardHolderInfo": {
    "cardNumber": "4084084084084081",
    "expiryMonth": "12",
    "expiryYear": "30",
    "cvv": "408"
  },
  "authorization": {
    "pin": "",
    "otp": ""
  },
  "redirect_Url": "https://your-site.com/payment-callback"
}
```

#### Request Fields:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `amount` | `decimal` | Yes | Amount in Naira (min: 0.01) |
| `firstName` | `string` | No | Customer's first name |
| `lastName` | `string` | No | Customer's last name |
| `email` | `string` | No | Customer's email |
| `phone` | `string` | No | Customer's phone number |
| `paymentReference` | `string` | Yes | **Your** unique reference for this transaction |
| `paymentChannel` | `string` | Yes | `"card"` |
| `currencyCode` | `string` | No | Default: `"NGN"` |
| `paymentRoute` | `string` | No | `"direct"` (default), `"pop_up"`, or `"link"` |
| `linkId` | `GUID` | No | Required only when `paymentRoute` is `"link"` |
| `cardHolderInfo.cardNumber` | `string` | Yes | Full card number |
| `cardHolderInfo.expiryMonth` | `string` | Yes | 2-digit month (e.g., `"12"`) |
| `cardHolderInfo.expiryYear` | `string` | Yes | 2-digit year (e.g., `"30"`) |
| `cardHolderInfo.cvv` | `string` | Yes | 3-digit CVV |
| `authorization.pin` | `string` | No | Card PIN — include on the re-call if the first response returns `PROVIDE_PIN` |
| `authorization.otp` | `string` | No | OTP — include if the response returns `PROVIDE_OTP` |
| `redirect_Url` | `string` | No | URL to redirect to after 3D Secure completion |
| `splitPaymentOptions` | `array` | No | For split payments to sub-accounts (see [Split Payments](#split-payments)) |

#### Response:

```json
{
  "isSuccess": true,
  "message": "Additional authorization required: PROVIDE_PIN",
  "responseCode": "00",
  "value": {
    "amount": 1000.00,
    "charges": 10.00,
    "paymentReference": "YOUR-UNIQUE-REF-001",
    "transactionReference": "CP-TXN-ABC123",
    "currencyCode": "NGN",
    "cardInfo": {
      "authorizationType": "PROVIDE_PIN"
    },
    "ussdInfo": null
  }
}
```

| Field | Type | Description |
|-------|------|-------------|
| `value.amount` | `decimal` | Transaction amount |
| `value.charges` | `decimal` | Processing fee (Naira) |
| `value.paymentReference` | `string` | Your original reference |
| `value.transactionReference` | `string` | System-generated reference — save this |
| `value.cardInfo.authorizationType` | `string` | Next action required (see table below) |

### 4.2 Step 2: Handle Authorization Type

The `authorizationType` tells you what to do next:

| Authorization Type | Meaning | Your Action |
|-------------------|---------|-------------|
| `PROVIDE_PIN` | Card requires PIN | Re-call `initiate-payment` with `authorization.pin` set (see below) |
| `PROVIDE_OTP` | OTP sent to cardholder's phone | Call `complete-payment` with the OTP |
| `PROCESS_ACS` | 3D Secure required | Redirect customer to bank's 3DS page, then call `verify-payment-status` |
| `PAYMENT_INITIATED` | Payment created, no card submitted yet | Submit card details via `initiate-payment` |
| `PAYMENT_PENDING` | Existing payment awaiting card | Submit card details via `initiate-payment` |

#### Submitting PIN

When you receive `PROVIDE_PIN`, call the **same** `initiate-payment` endpoint again with the **same** `paymentReference`, the **same** card details, plus the PIN:

```json
{
  "amount": 1000.00,
  "paymentReference": "YOUR-UNIQUE-REF-001",
  "paymentChannel": "card",
  "cardHolderInfo": {
    "cardNumber": "4084084084084081",
    "expiryMonth": "12",
    "expiryYear": "30",
    "cvv": "408"
  },
  "authorization": {
    "pin": "1234"
  }
}
```

The system detects the existing pending payment and submits the PIN to the processor. The response will typically return `PROVIDE_OTP` next.

### 4.3 Step 3: Complete Payment (OTP/PIN)

**`POST /api/v1/Gateway/complete-payment`**

Use this endpoint to submit an OTP after the cardholder receives it.

#### Request:

```json
{
  "paymentReference": "YOUR-UNIQUE-REF-001",
  "paymentChannel": "card",
  "otp": "123456",
  "pin": "",
  "cardHolderInfo": {
    "cardNumber": "4084084084084081",
    "expiryMonth": "12",
    "expiryYear": "30",
    "cvv": "408"
  }
}
```

> **Important:** Card details must be re-sent alongside the OTP. The payment processor requires full card info on the OTP submission endpoint.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `paymentReference` | `string` | Yes | Same reference from initiation |
| `paymentChannel` | `string` | Yes | `"card"` |
| `otp` | `string` | Conditional | OTP received by the cardholder |
| `pin` | `string` | Conditional | Card PIN (if submitting PIN via this endpoint) |
| `cardHolderInfo` | `object` | Yes | Same card details from initiation |

#### Response:

```json
{
  "isSuccess": true,
  "message": "Payment completed successfully.",
  "responseCode": "00"
}
```

If additional authorization is still needed, the message will indicate the next step.

### 4.4 Step 4: Verify Payment Status

**`POST /api/v1/Gateway/verify-payment-status`**

Call this to confirm the final payment outcome. Especially important after 3D Secure redirects or after `complete-payment`.

#### Request:

```json
{
  "reference": "YOUR-UNIQUE-REF-001",
  "paymentChannel": "card"
}
```

You can use either your `paymentReference` or the system's `transactionReference`.

#### Response:

```json
{
  "isSuccess": true,
  "responseCode": "00",
  "value": {
    "reference": "YOUR-UNIQUE-REF-001",
    "status": "Successful"
  }
}
```

### 4.5 Card Flow Diagram

```
┌──────────────┐
│  Your Server │
└──────┬───────┘
       │
       │ POST /api/v1/Gateway/initiate-payment
       │ { paymentChannel: "card", cardHolderInfo: {...} }
       ▼
  ┌─────────────┐
  │ Credipay API│ ──► Creates payment request
  └──────┬──────┘
         │
         ▼  authorizationType = ?
         │
         ├── "PROVIDE_PIN"
         │     │
         │     │ Re-call initiate-payment with authorization.pin
         │     ▼
         │   authorizationType = ?
         │     │
         │     ├── "PROVIDE_OTP"
         │     │     │
         │     │     │ POST complete-payment with otp + cardHolderInfo
         │     │     ▼
         │     │   "Payment completed successfully"
         │     │     │
         │     │     ▼ POST verify-payment-status
         │     │       → { status: "Successful" }
         │     │
         │     └── "SUCCESSFUL" → Verify to confirm
         │
         ├── "PROVIDE_OTP" (card didn't require PIN)
         │     │
         │     │ POST complete-payment with otp + cardHolderInfo
         │     ▼
         │   Verify status
         │
         └── "PROCESS_ACS" (3D Secure)
               │
               │ Redirect customer to bank's 3DS page
               │ Customer completes verification
               │ Redirected back to your redirect_Url
               ▼
             POST verify-payment-status
               → { status: "Successful" }
```

---

## 5. USSD Payments

USSD payments are simpler — you initiate, display the USSD dial code for the customer's bank, then poll until they complete it.

### 5.1 Step 1: Initiate Payment

**`POST /api/v1/Gateway/initiate-payment`**

#### Request:

```json
{
  "amount": 5000.00,
  "firstName": "Jane",
  "lastName": "Doe",
  "email": "jane@example.com",
  "phone": "08098765432",
  "paymentReference": "USSD-REF-001",
  "paymentChannel": "ussd",
  "currencyCode": "NGN",
  "paymentRoute": "direct"
}
```

No card info needed. The key difference is `"paymentChannel": "ussd"`.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `amount` | `decimal` | Yes | Amount in Naira |
| `paymentReference` | `string` | Yes | Your unique reference |
| `paymentChannel` | `string` | Yes | `"ussd"` |
| `firstName` | `string` | No | Customer first name |
| `lastName` | `string` | No | Customer last name |
| `email` | `string` | No | Customer email |
| `phone` | `string` | No | Customer phone |
| `currencyCode` | `string` | No | Default: `"NGN"` |

#### Response:

```json
{
  "isSuccess": true,
  "message": "USSD codes generated. Customer should dial the code for their bank.",
  "responseCode": "00",
  "value": {
    "amount": 5000.00,
    "charges": 50.00,
    "paymentReference": "USSD-REF-001",
    "transactionReference": "CP-TXN-XYZ789",
    "currencyCode": "NGN",
    "cardInfo": null,
    "ussdInfo": {
      "reference": "CP-TXN-XYZ789",
      "expiresIn": "2026-04-10T15:30:00Z",
      "banks": [
        {
          "name": "GTBank",
          "ussdString": "*737*000*5000#",
          "bankCode": "058",
          "logo": "https://cdn.credipay.io/logos/gtbank.png"
        },
        {
          "name": "Access Bank",
          "ussdString": "*901*000*5000#",
          "bankCode": "044",
          "logo": "https://cdn.credipay.io/logos/access.png"
        },
        {
          "name": "First Bank",
          "ussdString": "*894*000*5000#",
          "bankCode": "011",
          "logo": "https://cdn.credipay.io/logos/firstbank.png"
        }
      ]
    }
  }
}
```

| Field | Type | Description |
|-------|------|-------------|
| `value.ussdInfo.banks` | `array` | Available banks with USSD codes |
| `value.ussdInfo.banks[].name` | `string` | Bank name |
| `value.ussdInfo.banks[].ussdString` | `string` | USSD code for customer to dial |
| `value.ussdInfo.banks[].bankCode` | `string` | Bank code |
| `value.ussdInfo.banks[].logo` | `string` | Bank logo URL |
| `value.ussdInfo.expiresIn` | `datetime` | When the USSD session expires |
| `value.charges` | `decimal` | Processing fee (Naira) |

### 5.2 Step 2: Customer Dials USSD Code

Display the bank list to the customer. They select their bank and dial the corresponding `ussdString` on their phone. The bank will prompt them to authorize the payment.

**There is no API call for this step** — it happens on the customer's phone.

> **Note:** The `complete-payment` endpoint does **not** apply to USSD. Calling it will return: `"USSD payments are completed by the customer. Use verify-payment-status to check."`

### 5.3 Step 3: Verify Payment Status (Polling)

**`POST /api/v1/Gateway/verify-payment-status`**

After displaying the USSD codes, poll this endpoint until the status changes to `Successful` or the session expires.

#### Request:

```json
{
  "reference": "USSD-REF-001",
  "paymentChannel": "ussd"
}
```

#### Response (pending):

```json
{
  "isSuccess": true,
  "message": "Transaction status found.",
  "value": {
    "reference": "USSD-REF-001",
    "status": "InProgress"
  }
}
```

#### Response (completed):

```json
{
  "isSuccess": true,
  "value": {
    "reference": "USSD-REF-001",
    "status": "Successful"
  }
}
```

#### Recommended Polling Strategy:

| Interval | Duration | Reason |
|----------|----------|--------|
| Every 5 seconds | First 2 minutes | Customer likely still dialing |
| Every 10 seconds | 2–5 minutes | Customer going through bank prompts |
| Every 30 seconds | 5–15 minutes | Slow bank processing |
| Stop polling | After `expiresIn` | USSD session has expired |

### 5.4 USSD Flow Diagram

```
┌──────────────┐        ┌─────────────┐        ┌────────────┐
│  Your Server │        │ Credipay API│        │  Customer  │
└──────┬───────┘        └──────┬──────┘        └─────┬──────┘
       │                       │                      │
       │ POST initiate-payment │                      │
       │ { channel: "ussd" }   │                      │
       │──────────────────────►│                      │
       │                       │                      │
       │ Response: ussdInfo    │                      │
       │ { banks: [...] }      │                      │
       │◄──────────────────────│                      │
       │                       │                      │
       │ Display bank list ────────────────────────►  │
       │                       │                      │
       │                       │  Customer dials USSD │
       │                       │  on their phone      │
       │                       │  ◄── *737*..# ───── │
       │                       │                      │
       │ POST verify-payment   │                      │
       │ (poll every 5–10s)    │                      │
       │──────────────────────►│                      │
       │ { status: "InProgress"}                      │
       │◄──────────────────────│                      │
       │                       │                      │
       │ POST verify-payment   │  Bank confirms       │
       │──────────────────────►│◄─────────────────── │
       │ { status: "Successful"}                      │
       │◄──────────────────────│                      │
       │                       │                      │
       │ Show success ─────────────────────────────►  │
```

---

## 6. Payment Popup Integration

For integrators who want to use Credipay's hosted popup instead of handling card details directly.

### Validate Popup Session

**`POST /api/v1/Public/validate-payment-popup`**

Call this before opening the popup to validate the payment.

#### Request:

```json
{
  "amount": 2000.00,
  "email": "customer@example.com"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `amount` | `decimal` | Yes | Payment amount (0.01 – 999,999,999) |
| `email` | `string` | No | Customer email |

#### Response:

```json
{
  "isSuccess": true,
  "value": {
    "id": 12345,
    "email": "customer@example.com"
  }
}
```

### Popup Endpoints

The popup uses these public endpoints (same request/response shapes as the Gateway endpoints):

| Action | Endpoint | Method |
|--------|----------|--------|
| Initiate | `/api/v1/Public/initiate-payment` | POST |
| Complete | `/api/v1/Public/complete-payment` | POST |
| Verify by reference | `/api/v1/Public/verify-payment/{paymentReference}` | POST |
| Verify status | `/api/v1/Public/verify-payment-status` | GET |

When initiating from the popup, set `paymentRoute` to `"pop_up"`.

---

## 7. Webhooks

Credipay sends webhook notifications to your configured URL when payment events occur.

### Webhook Events

| Event | Type | Triggered When |
|-------|------|----------------|
| Collection Initiated | `Collection.CollectionInitiated` | Payment request created |
| Collection Success | `Collection.CollectionSuccess` | Payment completed and confirmed |

### Collection Initiated Payload

```json
{
  "event": "Collection.CollectionInitiated",
  "message": "Collection process has started.",
  "data": {
    "amount": 1000.00,
    "paymentReference": "YOUR-UNIQUE-REF-001",
    "transactionReference": "CP-TXN-ABC123",
    "paymentChannel": "card",
    "currencyCode": "NGN",
    "paymentRoute": "direct",
    "linkId": null,
    "accountInfo": {
      "accountName": "",
      "accountNumber": "",
      "bankName": "",
      "bankCode": ""
    },
    "payerInfo": {
      "firstName": "John",
      "lastName": "Doe",
      "email": "john@example.com",
      "phone": "08012345678"
    }
  }
}
```

### Collection Success Payload

```json
{
  "event": "Collection.CollectionSuccess",
  "message": "Collection completed.",
  "data": {
    "amount": 1000.00,
    "charge": 10.00,
    "paymentReference": "YOUR-UNIQUE-REF-001",
    "transactionReference": "YOUR-UNIQUE-REF-001",
    "paymentChannel": "card",
    "currencyCode": "NGN",
    "paymentRoute": "direct",
    "linkId": null,
    "transactionType": "Credit",
    "accountInfo": {
      "accountName": "",
      "accountNumber": "",
      "bankName": ""
    },
    "payerInfo": {
      "firstName": "John",
      "lastName": "Doe",
      "email": "john@example.com",
      "phone": "08012345678"
    },
    "successPayerInfo": {
      "sourceAccountNumber": "",
      "sourceAccountName": "",
      "sourceBankName": ""
    }
  }
}
```

### Webhook Security

Webhooks include your business secret key for signature verification. Always validate the webhook signature before processing.

### Configuring Webhook URLs

Set your webhook URLs in the Credipay merchant dashboard:

- **Test Webhook URL** — receives webhooks for test environment transactions
- **Live Webhook URL** — receives webhooks for live environment transactions

---

## 8. Payment Status Reference

### Transaction Statuses

| Status | Description | Applies To |
|--------|-------------|-----------|
| `Pending` | Payment request created, no card/USSD action yet | Card |
| `InProgress` | Authorization in progress or customer dialing USSD | Card, USSD |
| `Successful` | Payment confirmed and settled | Card, USSD |
| `Failed` | Payment failed | Card |
| `Unknown` | Reference not found | Card, USSD |

### Card Authorization Types

| Type | Meaning | Your Action |
|------|---------|-------------|
| `PROVIDE_PIN` | Card requires PIN | Re-call `initiate-payment` with `authorization.pin` |
| `PROVIDE_OTP` | OTP sent to cardholder | Call `complete-payment` with `otp` |
| `PROCESS_ACS` | 3D Secure required | Redirect customer to bank page, then verify |
| `PAYMENT_INITIATED` | Payment created, card not yet submitted | Submit card via `initiate-payment` |
| `PAYMENT_PENDING` | Existing payment awaiting card | Submit card via `initiate-payment` |

---

## 9. Error Handling & Best Practices

### Common Errors

| Error | Cause |
|-------|-------|
| `"Payment request not found or has no transaction reference."` | Invalid `paymentReference` or payment not yet initiated |
| `"Payment request already exists for this reference."` | Duplicate `paymentReference` — use a unique value per transaction |
| `"Business not found"` | Invalid API key or business account not found |
| `"Failed to process card payment."` | Card was declined or processor error |
| `"Failed to generate USSD codes."` | USSD service temporarily unavailable |
| `"USSD payments are completed by the customer. Use verify-payment-status to check."` | You called `complete-payment` for a USSD transaction — use `verify-payment-status` instead |

### Best Practices

1. **Generate unique `paymentReference` values.** Reusing a reference will retrieve the existing payment instead of creating a new one.

2. **Always call `verify-payment-status`** after OTP submission or 3D Secure redirect. Don't rely solely on the `complete-payment` response.

3. **For USSD, poll with backoff.** Start at 5-second intervals, increase to 30 seconds over time. Stop after `expiresIn`.

4. **Store the `transactionReference`** from the initiation response. You can use either your `paymentReference` or the system `transactionReference` for verification.

5. **Handle webhook deduplication.** You may receive both an API response and a webhook for the same successful transaction. Use your `paymentReference` as the idempotency key.

6. **Card details must be re-sent with OTP.** The `complete-payment` endpoint requires `cardHolderInfo` alongside the OTP — this is a processor requirement.

### Split Payments

Both card and USSD support splitting the payment across sub-accounts:

```json
{
  "splitPaymentOptions": [
    {
      "subAccountId": "your-subaccount-guid",
      "splitOptions": {
        "chargeType": "By_Percent",
        "value": 10.0
      }
    }
  ]
}
```

| ChargeType | Description |
|-----------|-------------|
| `By_Percent` | Split amount is a percentage of the total |
| `By_Value` | Split amount is a fixed Naira value |

---

## 10. Quick Start Examples

### Card Payment (4 calls)

```bash
# 1. Initiate with card details
curl -X POST <BASE_URL>/api/v1/Gateway/initiate-payment \
  -H "x-api-key: Test_sk_YOUR_SECRET_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "amount": 100,
    "paymentReference": "order-001",
    "paymentChannel": "card",
    "cardHolderInfo": {
      "cardNumber": "4084084084084081",
      "expiryMonth": "12",
      "expiryYear": "30",
      "cvv": "408"
    }
  }'
# → authorizationType: "PROVIDE_PIN"

# 2. Submit PIN (re-call initiate-payment with same reference)
curl -X POST <BASE_URL>/api/v1/Gateway/initiate-payment \
  -H "x-api-key: Test_sk_YOUR_SECRET_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "amount": 100,
    "paymentReference": "order-001",
    "paymentChannel": "card",
    "cardHolderInfo": {
      "cardNumber": "4084084084084081",
      "expiryMonth": "12",
      "expiryYear": "30",
      "cvv": "408"
    },
    "authorization": { "pin": "1234" }
  }'
# → authorizationType: "PROVIDE_OTP"

# 3. Submit OTP
curl -X POST <BASE_URL>/api/v1/Gateway/complete-payment \
  -H "x-api-key: Test_sk_YOUR_SECRET_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "paymentReference": "order-001",
    "paymentChannel": "card",
    "otp": "123456",
    "cardHolderInfo": {
      "cardNumber": "4084084084084081",
      "expiryMonth": "12",
      "expiryYear": "30",
      "cvv": "408"
    }
  }'
# → "Payment completed successfully."

# 4. Verify
curl -X POST <BASE_URL>/api/v1/Gateway/verify-payment-status \
  -H "x-api-key: Test_sk_YOUR_SECRET_KEY" \
  -H "Content-Type: application/json" \
  -d '{ "reference": "order-001", "paymentChannel": "card" }'
# → { "status": "Successful" }
```

### USSD Payment (2 calls)

```bash
# 1. Initiate — get bank USSD codes
curl -X POST <BASE_URL>/api/v1/Gateway/initiate-payment \
  -H "x-api-key: Test_sk_YOUR_SECRET_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "amount": 500,
    "paymentReference": "order-002",
    "paymentChannel": "ussd",
    "email": "customer@example.com",
    "phone": "08012345678"
  }'
# → ussdInfo.banks: [{ name: "GTBank", ussdString: "*737*..#" }, ...]

# 2. Poll until complete (customer dials code on their phone)
curl -X POST <BASE_URL>/api/v1/Gateway/verify-payment-status \
  -H "x-api-key: Test_sk_YOUR_SECRET_KEY" \
  -H "Content-Type: application/json" \
  -d '{ "reference": "order-002", "paymentChannel": "ussd" }'
# Repeat until → { "status": "Successful" }
```

---

**Need help?** Contact the Credipay integration support team for test credentials and environment access.
