# Sales Agent Commission & Installment Collection Flow

## Overview

This flow enables businesses, such as distributors of goods or services, to:

* Organize customers and sales agents into hierarchies under distributors.
* Collect installment payments from customers through dynamically generated virtual accounts.
* Pay commissions to sales agents through wallet-to-wallet transfers.
* Allow agents to withdraw earnings to their personal bank accounts.

---

## Actors

| Actor           | Role                                                                                                                |
| --------------- | ------------------------------------------------------------------------------------------------------------------- |
| **Business**    | The API consumer, authenticated via `x-api-key`. Orchestrates all operations.                                       |
| **Distributor** | A parent entity that owns agents and customers, for example a regional branch.                                      |
| **Sales Agent** | Earns commission on sales. Has a wallet but no static virtual account. Does not receive external payments directly. |
| **Customer**    | End-user who pays for goods/services. Has a wallet and static virtual account for receiving payments.               |
| **ERP API**     | The platform API handling wallet creation, checkout, payout, wallet balances, and transaction history.              |
| **9jaPay MFB**  | Banking partner that generates dynamic virtual accounts and processes NIP transfers.                                |

---

## High-Level Flow

```mermaid
flowchart TD
    A[Business creates Distributor] --> B[Distributor gets wallet + static VA]
    B --> C[Business creates Sales Agent under Distributor]
    C --> D[Agent gets wallet only]
    B --> E[Business creates Customer under Distributor]
    E --> F[Customer gets wallet + static VA]

    F --> G[Business initiates checkout for customer installment]
    G --> H[ERP API requests DVA from 9jaPay MFB]
    H --> I[DVA returned for unique payment reference]
    I --> J[Customer pays into DVA]
    J --> K[Bank webhook confirms payment]
    K --> L[Customer wallet credited]

    L --> M[Business pays agent commission]
    M --> N[Wallet-to-wallet transfer: Customer wallet to Agent wallet]
    N --> O[Agent wallet credited]

    O --> P[Agent requests withdrawal off-platform]
    P --> Q[Business initiates wallet payout to external bank]
    Q --> R[ERP API sends NIP transfer via 9jaPay MFB]
    R --> S[Agent receives funds in personal bank account]
```

---

## Sequence Diagram

```mermaid
sequenceDiagram
    participant B as Business API Consumer
    participant API as ERP API
    participant Bank as 9jaPay MFB
    participant C as Customer
    participant A as Sales Agent

    rect rgb(240, 248, 255)
    Note over B,API: ONE-TIME SETUP

    B->>API: POST /create-customer<br/>customerGroup: Distributor<br/>shouldCreateWallet: true<br/>shouldCreateStaticVirtualAccount: true
    API-->>B: distributorId, walletId, staticVA

    B->>API: POST /create-customer<br/>parentId: distributorId<br/>customerGroup: Retail<br/>shouldCreateWallet: true<br/>shouldCreateStaticVirtualAccount: false
    API-->>B: agentId, agentWalletId<br/>No static VA

    B->>API: POST /create-customer<br/>parentId: distributorId<br/>shouldCreateWallet: true<br/>shouldCreateStaticVirtualAccount: true
    API-->>B: customerId, customerWalletId, customerStaticVA
    end

    rect rgb(255, 248, 240)
    Note over B,C: INSTALLMENT COLLECTION

    B->>API: POST /checkout<br/>customerWalletId<br/>amount: 50000<br/>useWalletBalance: false<br/>unique paymentReference
    API->>Bank: Generate DVA for payment reference
    Bank-->>API: DVA account number
    API-->>B: DVA number + paymentReference

    B->>C: Send payment instruction<br/>Pay ₦50,000 to 900XXXXXXX
    C->>Bank: Transfer ₦50,000
    Bank->>API: Webhook: funds received
    API->>API: Credit customer wallet +₦50,000

    Note over B,API: Next installment uses a new checkout and new payment reference

    B->>API: POST /checkout<br/>same customerWalletId<br/>amount: 50000<br/>useWalletBalance: false<br/>new paymentReference
    API->>Bank: Generate new DVA
    Bank-->>API: New DVA account number
    API-->>B: New DVA number + paymentReference

    C->>Bank: Transfer ₦50,000
    Bank->>API: Webhook: funds received
    API->>API: Credit customer wallet +₦50,000

    Note over API: Customer wallet balance is now ₦100,000
    end

    rect rgb(240, 255, 240)
    Note over B,A: COMMISSION PAYOUT

    B->>API: POST /walletpayout<br/>sourceWalletId: customerWalletId<br/>destinationWalletId: agentWalletId<br/>amount: 5000
    API->>API: Debit customer wallet
    API->>API: Credit agent wallet
    API-->>B: Success<br/>Wallet-to-wallet transfer completed
    end

    rect rgb(255, 240, 255)
    Note over B,A: AGENT WITHDRAWAL

    A->>B: Agent requests withdrawal off-platform
    B->>API: POST /walletpayout<br/>sourceWalletId: agentWalletId<br/>externalAccountInfo: agent bank details<br/>amount: 5000
    API->>Bank: NIP transfer to agent bank account
    Bank-->>A: ₦5,000 received in personal account
    API-->>B: Success
    end

    rect rgb(248, 248, 248)
    Note over B,API: MONITORING

    B->>API: GET /wallet/balance/{walletId}
    API-->>B: Current wallet balance

    B->>API: GET /wallet-transactions/{walletId}
    API-->>B: Transaction history

    B->>API: GET /customers?parentId={distributorId}
    API-->>B: Agents and customers under distributor
    end
```

---

## Installment Collection Flow

```mermaid
flowchart TD
    A[Business starts checkout] --> B[POST /checkout]
    B --> C{useWalletBalance = false?}
    C -->|Yes| D[Ignore existing wallet balance]
    D --> E[Generate new payment reference]
    E --> F[ERP API requests DVA from 9jaPay MFB]
    F --> G[DVA returned]
    G --> H[Business sends DVA to customer]
    H --> I[Customer transfers installment amount]
    I --> J[Bank sends payment webhook]
    J --> K[ERP API verifies payment]
    K --> L[Customer wallet credited]
    L --> M[Installment accumulated in wallet]

    C -->|No| N[Wallet balance may be considered for checkout]
    N --> O[Not recommended for installment accumulation]
```

---

## Commission and Withdrawal Flow

```mermaid
flowchart LR
    A[Customer Wallet] -->|Commission walletpayout| B[Sales Agent Wallet]
    B -->|Withdrawal walletpayout| C[9jaPay MFB]
    C -->|NIP Transfer| D[Agent Personal Bank Account]

    E[Business] -->|Initiates commission payout| A
    E -->|Initiates agent withdrawal| B
```

---

## Key Design Decisions

| Concern                       | Solution                                                                                                               |
| ----------------------------- | ---------------------------------------------------------------------------------------------------------------------- |
| **Installment accumulation**  | Always pass `useWalletBalance: false` on checkout. Existing balance is ignored and a new DVA is always generated.      |
| **Unique payment references** | Each checkout call must use a new `paymentReference`. Reusing one returns `"payment reference already exists"`.        |
| **Agent does not need VA**    | Agents receive commission via wallet-to-wallet transfers only. No static VA means no accidental external deposits.     |
| **Commission timing**         | The business decides when to pay commission. It could be per sale, weekly, monthly, or triggered after full payment.   |
| **Agent withdrawal**          | Agent signals the business off-platform, then the business fires `walletpayout` with the agent’s bank details via NIP. |
| **Customer payment tracking** | Customer wallet balance and transaction history show accumulated installment payments.                                 |
| **Distributor hierarchy**     | Distributor acts as the parent entity for both agents and customers.                                                   |

---

## API Calls Summary

| Step | Endpoint                                  | Purpose                              | Key Params                                                                                                   |
| ---: | ----------------------------------------- | ------------------------------------ | ------------------------------------------------------------------------------------------------------------ |
|    1 | `POST /create-customer`                   | Create distributor                   | `customerGroup: "Distributor"`, `shouldCreateWallet: true`, `shouldCreateStaticVirtualAccount: true`         |
|    2 | `POST /create-customer`                   | Create sales agent under distributor | `parentId`, `customerGroup: "Retail"`, `shouldCreateWallet: true`, `shouldCreateStaticVirtualAccount: false` |
|    3 | `POST /create-customer`                   | Create customer under distributor    | `parentId`, `shouldCreateWallet: true`, `shouldCreateStaticVirtualAccount: true`                             |
|    4 | `POST /checkout`                          | Generate DVA for installment         | `customerWalletId`, `checkoutAmount`, `useWalletBalance: false`, unique `paymentReference`                   |
|    5 | External transfer                         | Customer pays installment            | Customer transfers to generated DVA                                                                          |
|    6 | Bank webhook                              | Confirm payment                      | Bank notifies ERP API that funds were received                                                               |
|    7 | `POST /walletpayout`                      | Pay agent commission                 | `sourceWalletId`, `destinationWalletId`, `amount`                                                            |
|    8 | `POST /walletpayout`                      | Withdraw agent earnings              | `sourceWalletId`, `externalAccountInfo: { bankCode, accountNumber }`, `amount`                               |
|    9 | `GET /wallet/balance/{walletId}`          | Check wallet balance                 | `walletId`                                                                                                   |
|   10 | `GET /wallet-transactions/{walletId}`     | View transaction history             | `walletId`                                                                                                   |
|   11 | `GET /customers?parentId={distributorId}` | View distributor hierarchy           | `distributorId`                                                                                              |

---

## Example End-to-End Scenario

A distributor has one customer and one sales agent.

The customer needs to pay **₦100,000** in two installments of **₦50,000** each.

### First installment

```text
Business calls POST /checkout
Amount: ₦50,000
useWalletBalance: false
paymentReference: INSTALLMENT-001
```

ERP API generates a DVA.

```text
Customer pays ₦50,000
Bank webhook confirms payment
Customer wallet balance becomes ₦50,000
```

### Second installment

```text
Business calls POST /checkout again
Amount: ₦50,000
useWalletBalance: false
paymentReference: INSTALLMENT-002
```

ERP API generates a new DVA.

```text
Customer pays ₦50,000
Bank webhook confirms payment
Customer wallet balance becomes ₦100,000
```

### Commission payout

```text
Business calls POST /walletpayout
sourceWalletId: customerWalletId
destinationWalletId: agentWalletId
amount: ₦5,000
```

Result:

```text
Customer wallet debited ₦5,000
Agent wallet credited ₦5,000
```

### Agent withdrawal

```text
Business calls POST /walletpayout
sourceWalletId: agentWalletId
externalAccountInfo:
  bankCode: "..."
  accountNumber: "..."
amount: ₦5,000
```

Result:

```text
Agent receives ₦5,000 in personal bank account via NIP transfer.
```

---

## Final Notes

The important part of this design is that **customer payments and agent earnings are separated**:

```text
Customers pay into generated payment accounts.
Customer wallets accumulate installment payments.
Agents receive commission internally through wallet-to-wallet transfers.
Agents withdraw externally only when needed.
```

This keeps installment collection traceable, prevents accidental deposits into agent wallets, and gives the business full control over commission timing.
