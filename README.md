
# Credipay ERP API Documentation

Our APIs allow you to create customers with wallets, with or without static virtual account numbers.
These customers can receive money through a checkout API which generates a Dynamic Virtual Account (DVA).

## Authentication
All requests must be authenticated using a Bearer token.

```json
"Authorization": "Bearer <your-secret-key>"
```

## Base URL
Replace `{{ERP Base URL}}` with your actual API base URL.

## Endpoints

### Checkout
**Method:** POST  
**URL:** `{{ERP Base URL}}/api/v1/ERP/checkout`

**Headers:**
- `Content-Type: application/json`
- `Authorization: Bearer <secret key>`

**Note:** If `useWalletBalance` is set to true, the system will debit the wallet first. If the balance is insufficient, it will generate a Dynamic Virtual Account (DVA) for the remaining amount.

**Request Body:**
```json
{
  "paymentReference": "Reference-1007",
  "customerWalletId": "<customerWalletId>",
  "useWalletBalance": false, //leave this as false to generate DVA expecting the total amount in the checkoutAmount below
  "checkoutAmount": 500,
  "accountName": "Reference 1007 Account",
  "bankCode": "090629"
}
```

### Create Customer
**Method:** POST  
**URL:** `{{ERP Base URL}}/api/v1/ERP/create-customer`

**Headers:**
- `Content-Type: application/json`
- `Authorization: Bearer <secret key>`

**Note:** You can create either an `individual` or a `business` customer. Use only one of the following objects in the payload:

```json
"individualInfo": {
  "firstName": "John",
  "lastName": "Doe",
  "phoneNumber": "+1234567890",
  "gender": "M",
  "email": "john@example.com"
}

OR

"businessInfo": {
  "businessName": "Big Wholesaler",
  "businessPhone": "+2348122223333",
  "businessEmail": "contact@bigwholesaler.com"
}
```

**Request Body:**
```json
{
    "businessInfo": {
        "businessName": "Big Wholesaler",
        "businessPhone": "+2348122223333",
        "businessEmail": "contact@bigwholesaler.com"
    },
    "accountName": "Big Wholesaler",
    "partnerCustomerId": "<customerId>",
    "customerType": "Business",
    "customerGroup": "Wholesaler", // Retail, Distributor or Wholesaler
    "shouldCreateWallet": true,
    "shouldCreateStaticVirtualAccount": true,
    "parentId": "67799f2e67799f2-e685-4b75-4044-08dd5fcc9bbf", //null if no parent
    "addressObject": {
        "addressLine1": "123 Main Street",
        "addressLine2": "Suite 4B",
        "city": "New York",
        "state": "NY",
        "country": "USA",
        "postCode": "10001"
    },
    "customerCreditLimitRequest": {
        "creditLimit": 5000,
        "tenureInDays": 30,
        "spendLimit": true,
        "spendLimitAmount": 2000,
        "spendLimitScopeInDays": 7
    },
    "onlinePresenseInfo": {
        "instagramHandle": "@johndoe",
        "facebook": "https://www.facebook.com/johndoe",
        "twitter": "@johndoe",
        "whatsapp": "+1234567890"
    }
}
```

### Get Wallet Balance
**Method:** GET  
**URL:** `{{ERP Base URL}}/api/v1/ERP/wallet/balance/<customerWalletId>`

**Path Parameter:** Replace `<customerWalletId>` with the customer's wallet ID.

**Headers:**
- `Authorization: Bearer <secret key>`

### Get Available VA Banks
**Method:** GET  
**URL:** `{{ERP Base URL}}/api/v1/ERP/available-va-banks`

**Headers:**
- `Authorization: Bearer <secret key>`

### Get Transfer Banks
**Method:** GET  
**URL:** `{{ERP Base URL}}/api/v1/ERP/transfer-banks`

**Headers:**
- `Authorization: Bearer <secret key>`

### Get Customer by ID
**Method:** GET  
**URL:** `{{ERP Base URL}}/api/v1/ERP/customer/<customerId>`

**Path Parameter:** Replace `<customerId>` with the specific customer's ID.

**Headers:**
- `Authorization: Bearer <secret key>`

### Get Customers
**Method:** GET  
**URL:** `{{ERP Base URL}}/api/v1/ERP/customers`

**Headers:**
- `Authorization: Bearer <secret key>`

**Query Parameters:**
| Parameter       | Description                                    |
|-----------------|------------------------------------------------|
| `CustomerGroup` | Available Values: Retail, Wholesaler, Distributor |
| `Startdate`     | string($date-time)                             |
| `Enddate`       | string($date-time)                             |
| `PageNumber`    | integer($int32)                                |
| `PageSize`      | integer($int32)                                |

### Get Wallets
**Method:** GET  
**URL:** `{{ERP Base URL}}/api/v1/ERP/wallets`

**Headers:**
- `Authorization: Bearer <secret key>`

**Query Parameters:**
| Parameter    | Description      |
|--------------|------------------|
| `Startdate`  | string($date-time)|
| `Enddate`    | string($date-time)|
| `PageNumber` | integer($int32)   |
| `PageSize`   | integer($int32)   |

### Get Wallet Transactions
**Method:** GET  
**URL:** `{{ERP Base URL}}/api/v1/ERP/wallet-transactions`

**Headers:**
- `Authorization: Bearer <secret key>`

**Query Parameters:**
| Parameter            | Description         |
|----------------------|---------------------|
| `TransactionReference`| string              |
| `TransactionType`     | Available Values: Credit, Debit |
| `CustomerNumber`      | string              |
| `WalletId`            | string($uuid)       |
| `Startdate`           | string($date-time)  |
| `Enddate`             | string($date-time)  |
| `PageNumber`          | integer($int32)     |
| `PageSize`            | integer($int32)     |

### Get Wallet by ID
**Method:** GET  
**URL:** `{{ERP Base URL}}/api/v1/ERP/wallet/<walletId>`

**Path Parameter:** Replace `<walletId>` with the wallet ID.

**Headers:**
- `Authorization: Bearer <secret key>`

### Get Transaction by ID
**Method:** GET  
**URL:** `{{ERP Base URL}}/api/v1/ERP/transaction/<transactionId>`

**Path Parameter:** Replace `<transactionId>` with the transaction ID.

**Headers:**
- `Authorization: Bearer <secret key>`

### Wallet Payout
**Method:** POST  
**URL:** `{{ERP Base URL}}/api/v1/ERP/walletpayout`

**Headers:**
- `Content-Type: application/json`
- `Authorization: Bearer <secret key>`

**Note:** This endpoint allows payouts to either another wallet or to an external bank account. Use `destinationWalletId` or `externalAccountInfo` accordingly.

**Request Body:**
```json
{
  "transactionReference": "reference 00019",
  "sourceWalletId": "<customerWalletId>",
  "accountInfo": {
    "destinationWalletId": "<walletId>",
    "externalAccountInfo": null
  },
  "amount": 50,
  "narration": "string"
}
```
