# Credipay ERP API Documentation

Our APIs allow you to create customers with wallets, with or without static virtual account numbers. These customers can receive money through a checkout API which generates a Dynamic Virtual Account (DVA). Additionally, you can perform wallet-to-wallet transfers or wallet-to-external-bank payouts.

---

## Authentication

All requests must be authenticated using a Bearer token.

```
"Authorization": "Bearer <your-secret-key>"
```

---

## Base URL

Replace `{{ERP Base URL}}` with your actual API base URL.

---

## Endpoints

### 1. Checkout

Generates a Dynamic Virtual Account (DVA) for a customer to receive payments. Optionally deducts from wallet balance first.

- **Method:** `POST`
- **URL:** `{{ERP Base URL}}/api/v1/ERP/checkout`

**Headers:**

| Header         | Value                      |
|----------------|----------------------------|
| Content-Type   | `application/json`         |
| Authorization  | `Bearer <secret key>`      |

**Behavior:**
- If `useWalletBalance` is `false`, the system generates a DVA expecting the **full** `checkoutAmount`.
- If `useWalletBalance` is `true`:
  - The system checks the customer's wallet balance.
  - If the wallet balance **covers the full amount**, the checkout succeeds immediately (no DVA is generated; `expectedAmount` will be `0`).
  - If the wallet balance is **less than** `checkoutAmount`, a DVA is generated for the **remaining difference** (`checkoutAmount - walletBalance`).
- A webhook notification (`collection.initiated`) is sent when the DVA account is created.

**Request Body:**

```jsonc
{
  "paymentReference": "Reference-1007",       // Required. Your unique payment reference. Must not be reused.
  "customerWalletId": "<customerWalletId>",   // Required. UUID of the customer's wallet.
  "useWalletBalance": false,                  // Required. Set to true to debit wallet first, false for full DVA.
  "checkoutAmount": 500,                      // Required. Total amount expected for this checkout (in Naira).
  "accountName": "Reference 1007 Account",    // Required. Name to display on the generated DVA.
  "bankCode": "090629"                        // Required. Bank code from the available VA banks list (see "Get Available VA Banks").
}
```

**Success Response (DVA Created):**

```json
{
  "status": true,
  "responseCode": "200",
  "message": "Payment account created successfully",
  "data": {
    "paymentReference": "Reference-1007",
    "expectedAmount": 500,
    "accountInfo": {
      "accountNumber": "1234567890",
      "bankName": "9jaPay MFB",
      "accountName": "Reference 1007 Account",
      "bankCode": "999916"
    }
  }
}
```

**Success Response (Wallet Fully Covers Amount):**

```json
{
  "status": true,
  "responseCode": "200",
  "message": "Checkout successful",
  "data": {
    "paymentReference": "Reference-1007",
    "expectedAmount": 0,
    "accountInfo": null
  }
}
```

**Error Responses:**

| Code | Scenario                        |
|------|---------------------------------|
| 400  | Invalid bank code               |
| 404  | Wallet not found                |
| 500  | Payment reference already used or provider error |

---

### 2. Create Customer

Creates a new customer record with optional wallet creation, static virtual account, credit limit configuration, and address.

- **Method:** `POST`
- **URL:** `{{ERP Base URL}}/api/v1/ERP/create-customer`

**Headers:**

| Header         | Value                      |
|----------------|----------------------------|
| Content-Type   | `application/json`         |
| Authorization  | `Bearer <secret key>`      |

**Customer Types:**

You can create either an **Individual** or **Business** customer. Provide **only one** of `individualInfo` or `businessInfo` in the payload. Do not supply both.

**Request Body (Full Example — Business Customer):**

```jsonc
{
  // --- Customer Identity (provide ONE of individualInfo or businessInfo) ---

  // Option A: Individual customer
  "individualInfo": null,
  // "individualInfo": {
  //   "firstName": "John",               // Required. Max 200 chars.
  //   "lastName": "Doe",                 // Required. Max 200 chars.
  //   "phoneNumber": "+2348012345678",   // Optional. Max 15 chars. E.164 format recommended.
  //   "gender": "M",                     // Optional. Single character: M or F.
  //   "email": "john@example.com"        // Required. Valid email. Max 200 chars.
  // },

  // Option B: Business customer
  "businessInfo": {
    "businessName": "Big Wholesaler",         // Required. Max 200 chars.
    "businessPhone": "+2348122223333",        // Optional. Max 15 chars.
    "businessEmail": "contact@bigwholesaler.com" // Required. Valid email. Max 200 chars.
  },

  // --- Core Fields ---
  "accountName": "Big Wholesaler",            // Required. Display name for the wallet/virtual account.
  "partnerCustomerId": "<your-internal-id>",  // Required. Your own system's unique customer identifier.
  "customerType": "Business",                 // Required. Enum: "Individual" or "Business".
  "customerGroup": "Wholesaler",              // Required. Enum: "Retail", "Wholesaler", or "Distributor".
  "shouldCreateWallet": true,                 // Required. Whether to create a wallet for this customer.
  "shouldCreateStaticVirtualAccount": true,   // Required. Whether to assign a permanent virtual account number.

  // --- Hierarchy (Optional) ---
  "parentId": "67799f2e-e685-4b75-4044-08dd5fcc9bbf", // Optional (nullable UUID). The ID of a parent customer.
                                                        // Parent MUST be of CustomerGroup "Distributor".
                                                        // Set to null if no parent hierarchy.

  // --- Address (Optional but validated if provided) ---
  "addressObject": {
    "addressLine1": "123 Main Street",   // Required if addressObject is provided.
    "addressLine2": "Suite 4B",          // Optional.
    "city": "Lagos",                     // Required if addressObject is provided.
    "state": "Lagos",                    // Required if addressObject is provided.
    "country": "Nigeria",                // Required if addressObject is provided.
    "postCode": "100001"                 // Required if addressObject is provided.
  },

  // --- Credit Limit Configuration (Optional) ---
  "customerCreditLimitRequest": {
    "creditLimit": 5000,            // The maximum credit the customer can hold (in Naira).
    "tenureInDays": 30,             // How many days the credit remains valid before repayment.
    "spendLimit": true,             // Whether to enforce a periodic spend limit.
    "spendLimitAmount": 2000,       // Optional (nullable). Max spend within the scope period. Required if spendLimit is true.
    "spendLimitScopeInDays": 7      // Optional (nullable). Scope window for the spend limit (in days). Required if spendLimit is true.
  },

  // --- Online Presence (Optional — all fields nullable) ---
  "onlinePresenseInfo": {
    "instagramHandle": "@bigwholesaler",
    "facebook": "https://www.facebook.com/bigwholesaler",
    "twitter": "@bigwholesaler",
    "whatsapp": "+2348122223333"
  }
}
```

**Request Body (Individual Customer — Minimal Example):**

```jsonc
{
  "individualInfo": {
    "firstName": "John",
    "lastName": "Doe",
    "phoneNumber": "+2348012345678",
    "gender": "M",
    "email": "john@example.com"
  },
  "businessInfo": null,
  "accountName": "John Doe Wallet",
  "partnerCustomerId": "CUST-001",
  "customerType": "Individual",
  "customerGroup": "Retail",
  "shouldCreateWallet": true,
  "shouldCreateStaticVirtualAccount": false,
  "parentId": null,
  "addressObject": null,
  "customerCreditLimitRequest": null,
  "onlinePresenseInfo": null
}
```

**Success Response (201):**

```json
{
  "status": true,
  "responseCode": "201",
  "message": "Customer created successfully.",
  "data": {
    "customerId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "partnerCustomerId": "<your-internal-id>",
    "customerType": "Business",
    "customerGroup": "Wholesaler",
    "dateCreated": "2025-01-15T10:30:00Z",
    "walletDetails": {
      "walletId": "f1e2d3c4-b5a6-7890-abcd-ef1234567890",
      "currencyCode": "NGN",
      "virtualAccountInfo": {
        "accountNumber": "9876543210",
        "bankName": "9jaPay MFB",
        "accountName": "Big Wholesaler",
        "bankCode": "999916"
      }
    }
  }
}
```

**Error Responses:**

| Code | Scenario                                                 |
|------|----------------------------------------------------------|
| 400  | Missing required fields, invalid email/phone, duplicate email |
| 404  | Parent customer not found or parent is not a Distributor |
| 500  | Virtual account creation failure (customer still created)|

---

### 3. Get Wallet Balance

Retrieves the current balance of a specific customer wallet.

- **Method:** `GET`
- **URL:** `{{ERP Base URL}}/api/v1/ERP/wallet/balance/<customerWalletId>`
- **Path Parameter:** `customerWalletId` — UUID of the customer's wallet.

**Headers:**

| Header         | Value                      |
|----------------|----------------------------|
| Authorization  | `Bearer <secret key>`      |

**Error Responses:**

| Code | Scenario          |
|------|--------------------|
| 404  | Wallet not found   |
| 500  | Internal error     |

---

### 4. Get Available VA Banks

Returns the list of banks available for Dynamic Virtual Account (DVA) creation during checkout. Use these bank codes in the Checkout endpoint's `bankCode` field.

- **Method:** `GET`
- **URL:** `{{ERP Base URL}}/api/v1/ERP/available-va-banks`

**Headers:**

| Header         | Value                      |
|----------------|----------------------------|
| Authorization  | `Bearer <secret key>`      |

---

### 5. Get Transfer Banks

Returns the list of banks available for external (NIP) transfers/payouts. Use these bank codes in the Wallet Payout endpoint's `externalAccountInfo.bankCode` field.

- **Method:** `GET`
- **URL:** `{{ERP Base URL}}/api/v1/ERP/transfer-banks`

**Headers:**

| Header         | Value                      |
|----------------|----------------------------|
| Authorization  | `Bearer <secret key>`      |

---

### 6. Get Customer by ID

Retrieves full details for a single customer.

- **Method:** `GET`
- **URL:** `{{ERP Base URL}}/api/v1/ERP/customer/<customerId>`
- **Path Parameter:** `customerId` — UUID of the customer.

**Headers:**

| Header         | Value                      |
|----------------|----------------------------|
| Authorization  | `Bearer <secret key>`      |

**Error Responses:**

| Code | Scenario            |
|------|---------------------|
| 404  | Customer not found  |
| 500  | Internal error      |

---

### 7. Get Customers (Paginated)

Retrieves a paginated list of customers with optional filters.

- **Method:** `GET`
- **URL:** `{{ERP Base URL}}/api/v1/ERP/customers`

**Headers:**

| Header         | Value                      |
|----------------|----------------------------|
| Authorization  | `Bearer <secret key>`      |

**Query Parameters:**

| Parameter            | Type                | Required | Description                                                         |
|----------------------|---------------------|----------|---------------------------------------------------------------------|
| `CustomerGroup`      | string (enum)       | No       | Filter by group: `Retail`, `Wholesaler`, or `Distributor`           |
| `ParentId`           | string (UUID)       | No       | Filter customers belonging to a specific parent (Distributor)       |
| `customerWalletNumber` | string            | No       | Filter by exact wallet account number                               |
| `Startdate`          | string (date-time)  | No       | Filter customers created on or after this date                      |
| `Enddate`            | string (date-time)  | No       | Filter customers created on or before this date                     |
| `PageNumber`         | integer             | No       | Page number (default: 1)                                            |
| `PageSize`           | integer             | No       | Number of results per page (default: 50)                            |

> **Note:** The documentation previously omitted `ParentId` and `customerWalletNumber` — both are supported query filters.

---

### 8. Get Wallets (Paginated)

Retrieves a paginated list of all customer wallets.

- **Method:** `GET`
- **URL:** `{{ERP Base URL}}/api/v1/ERP/wallets`

**Headers:**

| Header         | Value                      |
|----------------|----------------------------|
| Authorization  | `Bearer <secret key>`      |

**Query Parameters:**

| Parameter    | Type               | Required | Description                                    |
|--------------|--------------------|----------|------------------------------------------------|
| `Startdate`  | string (date-time) | No       | Filter wallets created on or after this date   |
| `Enddate`    | string (date-time) | No       | Filter wallets created on or before this date  |
| `PageNumber` | integer            | No       | Page number (default: 1)                       |
| `PageSize`   | integer            | No       | Results per page (default: 50)                 |

---

### 9. Get Wallet Transactions (Paginated)

Retrieves paginated wallet transactions with flexible filters.

- **Method:** `GET`
- **URL:** `{{ERP Base URL}}/api/v1/ERP/wallet-transactions`

**Headers:**

| Header         | Value                      |
|----------------|----------------------------|
| Authorization  | `Bearer <secret key>`      |

**Query Parameters:**

| Parameter              | Type               | Required | Description                                                  |
|------------------------|--------------------|----------|--------------------------------------------------------------|
| `TransactionReference` | string             | No       | Filter by exact transaction reference                        |
| `TransactionType`      | string (enum)      | No       | Filter by type: `Credit` or `Debit`                          |
| `CustomerNumber`       | string             | No       | Filter by the customer's partner-assigned customer number    |
| `WalletId`             | string (UUID)      | No       | Filter transactions for a specific wallet                    |
| `Startdate`            | string (date-time) | No       | Filter transactions on or after this date                    |
| `Enddate`              | string (date-time) | No       | Filter transactions on or before this date                   |
| `PageNumber`           | integer            | No       | Page number (default: 1)                                     |
| `PageSize`             | integer            | No       | Results per page (default: 50)                               |

---

### 10. Get Wallet by ID

Retrieves detailed information about a specific wallet.

- **Method:** `GET`
- **URL:** `{{ERP Base URL}}/api/v1/ERP/wallet/<walletId>`
- **Path Parameter:** `walletId` — UUID of the wallet.

**Headers:**

| Header         | Value                      |
|----------------|----------------------------|
| Authorization  | `Bearer <secret key>`      |

**Error Responses:**

| Code | Scenario          |
|------|--------------------|
| 404  | Wallet not found   |
| 500  | Internal error     |

---

### 11. Get Transaction by ID

Retrieves details of a single wallet transaction.

- **Method:** `GET`
- **URL:** `{{ERP Base URL}}/api/v1/ERP/transaction/<transactionId>`
- **Path Parameter:** `transactionId` — UUID of the transaction.

**Headers:**

| Header         | Value                      |
|----------------|----------------------------|
| Authorization  | `Bearer <secret key>`      |

**Error Responses:**

| Code | Scenario              |
|------|-----------------------|
| 404  | Transaction not found |
| 500  | Internal error        |

---

### 12. Wallet Payout

Transfers funds from a customer's wallet to **either** another wallet (internal) or an external bank account (NIP transfer).

- **Method:** `POST`
- **URL:** `{{ERP Base URL}}/api/v1/ERP/walletpayout`

**Headers:**

| Header         | Value                      |
|----------------|----------------------------|
| Content-Type   | `application/json`         |
| Authorization  | `Bearer <secret key>`      |

**Behavior:**
- **Wallet-to-Wallet:** Set `destinationWalletId` to the target wallet's UUID and leave `externalAccountInfo` as `null`. Both wallets must belong to the same business. Transfer is instant.
- **Wallet-to-External Bank:** Set `destinationWalletId` to `null` and populate `externalAccountInfo` with the bank account details. A NIP name enquiry is performed for validation before transfer. If the transfer call to the payment gateway fails, the transaction is queued for automatic retry (up to 3 retries, 5 minutes apart).
- A distributed lock prevents concurrent payouts from the same wallet.
- Duplicate `transactionReference` values for the same source wallet are rejected.
- A webhook notification is sent on successful completion.

**Request Body (Wallet-to-Wallet Transfer):**

```jsonc
{
  "transactionReference": "PAY-00019",           // Required. Your unique reference for this payout. Must not be reused per source wallet.
  "sourceWalletId": "<source-customerWalletId>",  // Required. UUID of the wallet to debit.
  "accountInfo": {
    "destinationWalletId": "<target-walletId>",   // UUID of the destination wallet (must belong to same business).
    "externalAccountInfo": null                   // Must be null for internal transfers.
  },
  "amount": 50,                                   // Required. Amount to transfer (in Naira). Must not exceed source balance.
  "narration": "Payment for invoice #123"         // Optional. Description/memo for the transaction. Truncated to 50 chars for NIP.
}
```

**Request Body (Wallet-to-External Bank Account):**

```jsonc
{
  "transactionReference": "PAY-00020",           // Required. Your unique reference for this payout.
  "sourceWalletId": "<source-customerWalletId>", // Required. UUID of the wallet to debit.
  "accountInfo": {
    "destinationWalletId": null,                 // Must be null for external transfers.
    "externalAccountInfo": {
      "accountNumber": "0123456789",             // Required. 10-digit NUBAN account number.
      "bankName": "First Bank of Nigeria",       // Required. Full name of the destination bank.
      "accountName": "Jane Doe",                 // Required. Account holder's name (used for display; NIP name enquiry validates actual name).
      "bankCode": "011"                          // Required. CBN bank code. Get valid codes from "Get Transfer Banks" endpoint.
    }
  },
  "amount": 1500,                                // Required. Amount to transfer (in Naira).
  "narration": "Supplier payment"                // Optional. Truncated to 50 characters for NIP transfers.
}
```

**Important:** You must provide **either** `destinationWalletId` (internal) **or** `externalAccountInfo` (external) — never both, and never neither. If both are null, the API returns a 400 error.

**Success Response (Internal Transfer — Instant):**

```json
{
  "status": true,
  "responseCode": "200",
  "message": "Internal transfer completed successfully",
  "data": "PAY-00019"
}
```

**Success Response (External Transfer — Pending Confirmation):**

```json
{
  "status": true,
  "responseCode": "202",
  "message": "Transfer initiated and pending confirmation",
  "data": "PAY-00020"
}
```

**Success Response (External Transfer — Completed):**

```json
{
  "status": true,
  "responseCode": "200",
  "message": "Transfer completed successfully",
  "data": "PAY-00020"
}
```

**Error Responses:**

| Code | Scenario                                                         |
|------|------------------------------------------------------------------|
| 400  | Insufficient balance, duplicate reference, invalid account info, missing required fields |
| 404  | Source wallet not found, destination wallet not found             |
| 409  | Concurrent transaction in progress (lock conflict)               |
| 500  | Internal processing error                                        |

---

### 13. Delete Wallet

Deletes a customer's wallet. If the wallet has a remaining balance, funds are transferred to the business account before deletion.

- **Method:** `DELETE`
- **URL:** `{{ERP Base URL}}/api/v1/ERP/delete-wallet`

**Headers:**

| Header         | Value                      |
|----------------|----------------------------|
| Content-Type   | `application/json`         |
| Authorization  | `Bearer <secret key>`      |

**Request Body:**

```jsonc
{
  "customerWalletId": "<walletId>",  // Required. UUID of the wallet to delete.
  "reason": "Customer requested"     // Optional. Reason for deletion (for audit trail).
}
```

**Success Response:**

```json
{
  "status": true,
  "message": "Wallet deleted successfully",
  "data": {
    "customerWalletId": "f1e2d3c4-b5a6-7890-abcd-ef1234567890",
    "transferredAmount": 250.00,
    "fundsTransferred": true,
    "businessAccountNumber": "1234567890",
    "message": "Remaining funds transferred to business account",
    "dateDeleted": "2025-01-15T14:30:00Z"
  }
}
```

---

### 14. Delete Customer

Deletes a customer record. If the customer has a wallet with a non-zero balance, deletion will **fail**. If the wallet has zero balance, it will be deleted along with the customer.

- **Method:** `DELETE`
- **URL:** `{{ERP Base URL}}/api/v1/ERP/delete-customer`

**Headers:**

| Header         | Value                      |
|----------------|----------------------------|
| Content-Type   | `application/json`         |
| Authorization  | `Bearer <secret key>`      |

**Request Body:**

```jsonc
{
  "customerId": "<customerId>",      // Required. UUID of the customer to delete.
  "reason": "Duplicate account"      // Optional. Reason for deletion (for audit trail).
}
```

**Success Response:**

```json
{
  "status": true,
  "message": "Customer deleted successfully",
  "data": {
    "customerId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "walletDeleted": true,
    "message": "Customer and associated wallet deleted",
    "dateDeleted": "2025-01-15T14:30:00Z",
    "reason": "Duplicate account"
  }
}
```

**Error Responses:**

| Code | Scenario                                          |
|------|---------------------------------------------------|
| 400  | Customer has wallet with non-zero balance         |
| 422  | Unprocessable entity (service returned null)      |

---

## Webhooks

The system sends webhook notifications to your configured URL for certain events:

| Event Type              | Trigger                                              |
|-------------------------|------------------------------------------------------|
| `collection.initiated`  | When a DVA is created during checkout                |
| `payout.successful`     | When a wallet payout completes successfully          |
| `payout.failed`         | When a wallet payout fails                           |

Webhook payloads are signed using your secret key for verification.

---

## Notes

- All monetary amounts are in **Nigerian Naira (NGN)**.
- UUIDs are in standard format: `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx`.
- Pagination defaults: `PageNumber = 1`, `PageSize = 50`.
- Date-time values should be in ISO 8601 format (e.g., `2025-01-15T00:00:00Z`).
- The `bankCode` in Checkout must come from the **Available VA Banks** list; the `bankCode` in Wallet Payout must come from the **Transfer Banks** list. These are different bank pools.
