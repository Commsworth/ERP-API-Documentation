# Credipay ERP API Documentation

Our APIs allow you to create customers with wallets, with or without static virtual account numbers. These customers can receive money through a checkout API which generates a Dynamic Virtual Account (DVA). Additionally, you can perform wallet-to-wallet transfers or wallet-to-external-bank payouts, and you can re-query payout status by your `transactionReference`.

---

## Authentication

All requests must be authenticated using the `x-api-key` header with your secret key.

```
"x-api-key": "<your-secret-key>"
```

The secret key format includes an environment prefix and key type indicator (e.g., `Test_sk_...` for test environment, `Live_sk_...` for production). The system determines the environment (Test vs Live) from the key itself.

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
| x-api-key      | `<your-secret-key>`        |

**Behavior:**
- Only **safeguarded wallets** are eligible for ERP checkout.
- Wallets created by `create-customer` with `shouldCreateWallet = true` already satisfy this requirement because the ERP service creates them as safeguarded wallets.
- The optional `bearer` field controls who absorbs the ERP inflow charge for the DVA funding leg:
  - `Customer` (default): the generated DVA is grossed up so the payer funds both principal and charge.
  - `Merchant`: the generated DVA expects only the requested checkout amount; the merchant absorbs the charge separately.
- If `useWalletBalance` is `false`, the system generates a DVA using the selected `bearer` rule.
- If `useWalletBalance` is `true`:
  - The system checks the customer's wallet balance.
  - If the wallet balance **covers the full amount**, the checkout succeeds immediately (no DVA is generated; `expectedAmount` will be `0`).
  - If the wallet balance is **less than** `checkoutAmount`, a DVA is generated for the remaining amount using the selected `bearer` rule.
- `checkoutAmount` is the amount the business wants to make available through the checkout flow.
  - For `bearer = Customer`, it is treated as the **target net value** after charge.
  - For `bearer = Merchant`, it is treated as the funded and credited amount; the merchant absorbs the charge separately.
- The response now returns:
  - `expectedAmount`: the **gross** amount the payer must transfer into the generated DVA
  - `chargeAmount`: the ERP inflow charge embedded in that funding target
  - `netAmount`: the amount that will actually be credited to the customer wallet from the DVA funding leg
  - `bearer`: the resolved fee bearer for the checkout
- The supplied `bankCode` must exist in the Available VA Banks list.
- When Dynamic VA routing is configured for the calling business/partner, that `bankCode` is used to resolve the eligible provider/bank mapping for account creation, and the returned `accountInfo.bankName` / `accountInfo.bankCode` reflect the routed provider response.
- If no Dynamic VA route is configured, checkout falls back to the default 9jaPay account-creation flow.
- A webhook notification is sent when the DVA account is created.

**Request Body:**

```jsonc
{
  "paymentReference": "Reference-1007",       // Required. Your unique payment reference. Must not be reused.
  "customerWalletId": "<customerWalletId>",   // Required. UUID of the customer's wallet.
  "useWalletBalance": false,                  // Required. Set to true to debit wallet first, false for full DVA.
  "checkoutAmount": 500,                      // Required. Requested checkout amount. Net target for Customer bearer; funded amount for Merchant bearer.
  "accountName": "Reference 1007 Account",    // Required. Name to display on the generated DVA.
  "bankCode": "090629",                      // Required. Bank code from the available VA banks list (see "Get Available VA Banks").
  "bearer": "Customer"                       // Optional. Enum: "Customer" or "Merchant". Defaults to "Customer".
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
    "expectedAmount": 525,
    "chargeAmount": 25,
    "netAmount": 500,
    "bearer": "Customer",
    "accountInfo": {
      "accountNumber": "1234567890",
      "bankName": "Providus Bank",
      "accountName": "Reference 1007 Account",
      "bankCode": "000023"
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
    "chargeAmount": 0,
    "netAmount": 0,
    "bearer": "Customer",
    "accountInfo": null
  }
}
```

> **Example (`bearer = Customer`):** if the requested `checkoutAmount` is `500` and the configured ERP inflow charge is `25`, the payer must fund `525` and the safeguarded customer wallet receives `500`.

> **Example (`bearer = Merchant`):** if the requested `checkoutAmount` is `500` and the configured ERP inflow charge is `25`, the payer funds `500`, the customer wallet receives `500`, and the merchant absorbs the `25` charge.

**Error Responses:**

| Code | Scenario                        |
|------|---------------------------------|
| 403  | Wallet is not safeguarded       |
| 400  | Invalid bank code               |
| 404  | Wallet not found                |
| 500  | Payment reference already used or provider error |

> **Implementation note:** When the source wallet is not a safeguarded wallet, the service sets `responseCode` to `403` with message `Checkout is only available for safeguarded wallets`, but the current controller still returns HTTP `400 Bad Request`. Integrators should inspect the response body's `responseCode` as well as the transport status.

---

### 2. Create Customer

Creates a new customer record with optional wallet creation, static virtual account, credit limit configuration, and address.

- **Method:** `POST`
- **URL:** `{{ERP Base URL}}/api/v1/ERP/create-customer`

**Headers:**

| Header         | Value                      |
|----------------|----------------------------|
| Content-Type   | `application/json`         |
| x-api-key      | `<your-secret-key>`        |

**Customer Types:**

You can create either an **Individual** or **Business** customer. Provide **only one** of `individualInfo` or `businessInfo` in the payload. Do not supply both.

**Behavior Notes:**
- If `shouldCreateWallet` is `false`, the customer is still created successfully and `walletDetails` will be `null`.
- If `shouldCreateWallet` is `true`, the wallet created by this ERP API is a **safeguarded wallet**.
- If `shouldCreateStaticVirtualAccount` is `true`, static virtual account creation is **best effort**. The customer creation call still returns `201` even if provider account creation fails; in that case `walletDetails.virtualAccountInfo` will be `null`.

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

> **Note:** Static virtual account creation failure does **not** fail the customer creation call. The API still returns `201 Created`, but `walletDetails.virtualAccountInfo` may be `null`.

---

### 3. Get Wallet Balance

Retrieves the current balance of a specific customer wallet.

- **Method:** `GET`
- **URL:** `{{ERP Base URL}}/api/v1/ERP/wallet/balance/<customerWalletId>`
- **Path Parameter:** `customerWalletId` — UUID of the customer's wallet.

**Headers:**

| Header         | Value                      |
|----------------|----------------------------|
| x-api-key      | `<your-secret-key>`        |

**Success Response (200):**

```json
{
  "status": true,
  "responseCode": "200",
  "message": "Wallet balance retrieved successfully",
  "data": {
    "balance": 12500.75,
    "currencyCode": "NGN"
  }
}
```

**Error Responses:**

| Code | Scenario          |
|------|--------------------|
| 404  | Wallet not found   |
| 500  | Internal error     |

---

### 4. Get Available VA Banks

Returns the list of banks available for Dynamic Virtual Account (DVA) creation during checkout. Use these bank codes in the Checkout endpoint's `bankCode` field.

- The response is route-aware for the calling business/partner.
- If routing is configured, only bank mappings eligible for the current Dynamic VA routes are returned.
- If no routing is configured, the endpoint falls back to the default 9jaPay-backed bank list.

- **Method:** `GET`
- **URL:** `{{ERP Base URL}}/api/v1/ERP/available-va-banks`

**Headers:**

| Header         | Value                      |
|----------------|----------------------------|
| x-api-key      | `<your-secret-key>`        |

---

### 5. Get Transfer Banks

Returns the list of banks available for external (NIP) transfers/payouts. Use these bank codes in the Wallet Payout endpoint's `externalAccountInfo.bankCode` field.

- **Method:** `GET`
- **URL:** `{{ERP Base URL}}/api/v1/ERP/transfer-banks`

**Headers:**

| Header         | Value                      |
|----------------|----------------------------|
| x-api-key      | `<your-secret-key>`        |

---

### 6. Get Customer by ID

Retrieves full details for a single customer.

- **Method:** `GET`
- **URL:** `{{ERP Base URL}}/api/v1/ERP/customer/<customerId>`
- **Path Parameter:** `customerId` — UUID of the customer.

**Headers:**

| Header         | Value                      |
|----------------|----------------------------|
| x-api-key      | `<your-secret-key>`        |

**Response Shape:**
- `data.customerData`: core customer profile
- `data.addresses`: delivery addresses list
- `data.walletDetails`: wallet summary, current `availableBalance`, and `virtualAccountInfo`

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
| x-api-key      | `<your-secret-key>`        |

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
| x-api-key      | `<your-secret-key>`        |

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
| x-api-key      | `<your-secret-key>`        |

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
| x-api-key      | `<your-secret-key>`        |

**Response Shape:**
- Core wallet fields (`id`, `customerId`, `customerNumber`, `accountName`, `currencyCode`)
- Credit configuration (`creditLimit`, `tenureInDays`, `limitSpend`, `spendLimit`, `limitScope`)
- `availableBalance`
- `virtualAccountInfo`

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
| x-api-key      | `<your-secret-key>`        |

**Error Responses:**

| Code | Scenario              |
|------|-----------------------|
| 404  | Transaction not found |
| 500  | Internal error        |

---

### 12. Get Transfer Status by Reference

Looks up the current status of a payout using the same `transactionReference` you originally supplied to `walletpayout`. This is safe to call repeatedly for timeout recovery and reconciliation.

- **Method:** `GET`
- **URL:** `{{ERP Base URL}}/api/v1/ERP/transfer-status/<reference>`
- **Path Parameter:** `reference` — your original payout `transactionReference`.

**Headers:**

| Header         | Value                      |
|----------------|----------------------------|
| x-api-key      | `<your-secret-key>`        |

**Success Response (200):**

```json
{
  "status": true,
  "responseCode": "200",
  "message": "Transfer status retrieved successfully",
  "data": {
    "id": "6d1f5a11-1a66-4af0-b915-6d554f72f253",
    "walletId": "f1e2d3c4-b5a6-7890-abcd-ef1234567890",
    "reference": "PAY-00020",
    "internalReference": "PAY-00020",
    "sourceAccountNumber": "1234567890",
    "sourceAccountName": "John Doe Wallet",
    "sourceBank": "Credipay",
    "destinationAccountNumber": "0123456789",
    "destinationAccountName": "Jane Doe",
    "destinationBank": "First Bank of Nigeria",
    "sessionId": "WalletPayout_PAY-00020202501011030",
    "amount": 1500,
    "narration": "Supplier payment",
    "transactionDate": "2025-01-15T10:30:00Z",
    "transactionType": "Debit",
    "status": "Pending"
  }
}
```

**Error Responses:**

| Code | Scenario                          |
|------|-----------------------------------|
| 400  | Missing or empty reference        |
| 404  | No transfer found for reference   |
| 500  | Internal error                    |

---

### 13. Wallet Payout

Transfers funds from a customer's wallet to **either** another wallet (internal) or an external bank account (NIP transfer).

- **Method:** `POST`
- **URL:** `{{ERP Base URL}}/api/v1/ERP/walletpayout`

**Headers:**

| Header         | Value                      |
|----------------|----------------------------|
| Content-Type   | `application/json`         |
| x-api-key      | `<your-secret-key>`        |

**Behavior:**
- Only **safeguarded wallets** are eligible for payout.
- **Wallet-to-Wallet:** Set `destinationWalletId` to the target wallet's UUID and leave `externalAccountInfo` as `null`. Both wallets must belong to the same business. Transfer is instant and no transfer fee applies.
- **Wallet-to-External Bank:** Set `destinationWalletId` to `null` and populate `externalAccountInfo` with the bank account details. A NIP name enquiry is performed for validation before transfer.
- External bank payouts attract a **tiered transfer fee** looked up from Credipay's `FundTransferCharges` table.
- The optional `bearer` field controls who absorbs that fee:
  - `Customer` (default): the source wallet is debited `amount + transferCharge`, while the beneficiary receives exactly `amount`.
  - `Merchant`: the source wallet is debited exactly `amount`, while the transfer fee is debited separately from the merchant's settle-able pool.
- If the provider transfer call does not return an immediate success, the payout is recorded as pending and queued for automatic requery for up to **6 retries at 10-minute intervals**.
- A distributed lock prevents concurrent payouts from the same wallet.
- Reusing the same `transactionReference` for the same source wallet is treated as an **idempotent replay**. The API returns success and does not duplicate the payout.
- Use `GET /api/v1/ERP/transfer-status/<reference>` to reconcile delayed or previously timed-out payout calls.
- A payout status webhook is sent on immediate success and after background requery resolution.

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
  "narration": "Payment for invoice #123",        // Optional. Description/memo for the transaction. Truncated to 50 chars for NIP.
  "bearer": "Customer"                           // Optional. Ignored for internal transfers.
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
  "narration": "Supplier payment",               // Optional. Truncated to 50 characters for NIP transfers.
  "bearer": "Merchant"                           // Optional. Enum: "Customer" or "Merchant". Defaults to "Customer".
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

**Success Response (Idempotent Replay):**

```json
{
  "status": true,
  "responseCode": "200",
  "message": "Transfer already processed (original status: Successful). Use GET /api/v1/ERP/transfer-status/{reference} for full details.",
  "data": "PAY-00020"
}
```

**Error Responses:**

| Code | Scenario                                                         |
|------|------------------------------------------------------------------|
| 400  | Insufficient balance, insufficient balance to cover transfer charge, invalid account info, missing required fields |
| 404  | Source wallet not found, destination wallet not found             |
| 409  | Concurrent transaction in progress (lock conflict)               |
| 500  | Internal processing error                                        |

> **Implementation note:** When the source wallet is not a safeguarded wallet, the service sets `responseCode` to `403` with message `Wallet payout is only available for safeguarded wallets`, but the current controller still returns HTTP `400 Bad Request`. Integrators should inspect the response body's `responseCode` as well as the transport status.

---

### 14. Delete Wallet

Deletes a customer's wallet **only if its balance is zero**. The wallet is soft-deleted; the API does **not** automatically sweep remaining funds to a business account.

- **Method:** `DELETE`
- **URL:** `{{ERP Base URL}}/api/v1/ERP/delete-wallet`

**Headers:**

| Header         | Value                      |
|----------------|----------------------------|
| Content-Type   | `application/json`         |
| x-api-key      | `<your-secret-key>`        |

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
  "responseCode": "200",
  "message": "Wallet deleted successfully.",
  "data": {
    "customerWalletId": "f1e2d3c4-b5a6-7890-abcd-ef1234567890",
    "transferredAmount": 0,
    "fundsTransferred": false,
    "businessAccountNumber": null,
    "message": "Wallet deleted successfully with zero balance",
    "dateDeleted": "2025-01-15T14:30:00Z"
  }
}
```

**Error Responses:**

| Code | Scenario                                                            |
|------|---------------------------------------------------------------------|
| 400  | Wallet already deleted, or wallet balance is greater than zero      |
| 404  | Wallet not found or not owned by the calling business               |
| 500  | Internal deletion error                                             |

> **Implementation note:** The current controller returns HTTP `400 Bad Request` for all non-success `delete-wallet` responses. Inspect the response body's `responseCode` to distinguish `404` vs `500` semantics.

---

### 15. Delete Customer

Deletes a customer record. If the customer has a wallet with a non-zero balance, deletion will **fail**. If the wallet has zero balance, it will be deleted along with the customer.

- **Method:** `DELETE`
- **URL:** `{{ERP Base URL}}/api/v1/ERP/delete-customer`

**Headers:**

| Header         | Value                      |
|----------------|----------------------------|
| Content-Type   | `application/json`         |
| x-api-key      | `<your-secret-key>`        |

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
  "responseCode": "200",
  "message": "Customer deleted successfully. Customer wallet was also deleted.",
  "data": {
    "customerId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "walletDeleted": true,
    "message": "Customer deleted successfully. Customer wallet was also deleted.",
    "dateDeleted": "2025-01-15T14:30:00Z",
    "reason": "Duplicate account"
  }
}
```

**Error Responses:**

| Code | Scenario                                          |
|------|---------------------------------------------------|
| 400  | Customer already deleted, or customer has wallet with non-zero balance |
| 404  | Customer not found or not owned by the calling business |
| 500  | Internal deletion error                           |

> **Implementation note:** The current controller returns HTTP `400 Bad Request` for all non-success `delete-customer` responses. Inspect the response body's `responseCode` to distinguish `404` vs `500` semantics.

---

## Simulate Fund Transfer (Test Environment Only)

Simulates an inbound bank transfer to a virtual account. Use this to test the collection flow end-to-end without making a real bank transfer.

- **Method:** `POST`
- **URL:** `{{Base URL}}/api/v1/Gateway/simulate-fund-transfer`

> **⚠️ Test only.** This endpoint is hardcoded to the test environment and will reject calls if the server is running in Production mode.

**Headers:**

| Header         | Value                      |
|----------------|----------------------------|
| Content-Type   | `application/json`         |
| x-api-key      | `<your-secret-key>`        |

**Request Body:**

```json
{
  "recipientAccountNumber": "9000119338",
  "amount": 50000
}
```

| Field                    | Type    | Required | Description |
|--------------------------|---------|----------|-------------|
| `recipientAccountNumber` | string  | Yes      | The static VA or checkout-generated DVA account number to fund |
| `amount`                 | decimal | Yes      | Amount in **kobo** to send to the Pay9ja sandbox (for example, `50000` = NGN `500.00`) |

**Success Response (200 OK):**

```json
{
  "isSuccess": true,
  "value": null,
  "error": null,
  "responseCode": null
}
```

**Failure Response (if called in Production):**

```json
{
  "isSuccess": false,
  "value": null,
  "error": "99",
  "responseCode": null
}
```

**Behavior:**
- Triggers the same internal flow as a real bank transfer — the customer's wallet is credited and a `Collection.CollectionSuccess` webhook is fired.
- The request amount is forwarded to the Pay9ja sandbox as-is.
- Because inbound Pay9ja webhook amounts are normalized from **kobo to Naira** when Credipay processes the callback, callers should send the simulation amount in **kobo**. For example, sending `500` simulates NGN `5.00`, while sending `50000` simulates NGN `500.00`.
- Works for both static VAs (permanent account numbers) and checkout DVAs (transaction-scoped account numbers).

---

## Webhooks

The ERP flows trigger webhook notifications to your configured URL.

| Flow | Trigger | Current outbound `message` value |
|------|---------|----------------------------------|
| Checkout account creation | When a checkout DVA is created | `Collection process has started.` |
| Wallet payout status | When a payout completes immediately, or when background requery later resolves it | `payout_status_event` |

**Checkout webhook payload highlights:**
- `amount` (gross amount expected to be funded)
- `charge`
- `netAmount`
- `bearer`
- `paymentReference`
- `transactionReference`
- `paymentChannel`
- `currencyCode`
- `paymentRoute`
- `accountInfo`
- `payerInfo`

**Payout webhook payload highlights:**
- `amount`
- `transactionReference`
- `transactionStatus`
- `sourceWalletId`
- `benficiaryDetails`
- `transferCharge`
- `bearer`

**Collection success webhook note:**
- For ERP safeguarded wallet inflows, the incoming collection can now contain a non-zero `charge` while the wallet is credited with the net amount after that charge when `bearer = Customer`.
- For checkout DVAs with `bearer = Merchant`, the customer wallet still receives the full funded amount while the merchant absorbs the fee separately.

Webhook payloads are signed using your secret key for verification.

---

## Notes

- All monetary amounts are in **Nigerian Naira (NGN)**.
- UUIDs are in standard format: `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx`.
- Pagination defaults: `PageNumber = 1`, `PageSize = 50`.
- Date-time values should be in ISO 8601 format (e.g., `2025-01-15T00:00:00Z`).
- The `bankCode` in Checkout must come from the **Available VA Banks** list; the `bankCode` in Wallet Payout must come from the **Transfer Banks** list. These are different bank pools.
