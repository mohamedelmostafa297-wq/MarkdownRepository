# [LLD] Postpaid Activation Line

> **Document Status**: Draft
> **Author**: System Architect
> **Type**: Low Level Design (LLD)
> **Baseline**: Aligned with `Prepaid.md` standard.

---

## 1. Step 1 – ValidateIdNew (Postpaid Design)

### 1.1 Purpose
Start the sales onboarding flow for a **Postpaid** customer.
This step orchestrates the initial validation, fetches customer profile from Huawei BSS, checks TCC eligibility, performs initial **Credit Risk assessment**, and creates an Onboarding Order via `createOrder`.

*   **Existing Endpoint**: `POST /api/Sales/ValidateIdNew`
*   **Change**: Backend logic update to handle `IsPostpaid`.

### 1.2 External APIs Used

#### Huawei BSS – Customer Info / Existence
*   **Role**: Single source of truth for customer data.
*   **Logic**:
    1.  Call BSS "Check Customer" API.
    2.  If not found, call `POST {{middleware-service-redbull}}/customer/queryinfo`.
    3.  Persist result in local DB for caching/reporting.

#### TCC Eligibility via Integration (Ballighny)
*   **Role**: Regulatory eligibility check.
*   **Logic**:
    *   Call `RBMDigitalCustomIntegration.CheckEligibility`.
    *   **Postpaid Deviation**: The `SubType` is explicitly set to Postpaid (mapped value).
    *   **Constraint**: Check against "Max Active Postpaid Lines" limit.

### 1.3 ValidateIdNew – Detailed Flow (Java)

**Endpoint**: `POST /api/Sales/ValidateIdNew`

1.  **Configuration & Blacklist**:
    *   Check `DisableActivation` config.
    *   Check `BlackList` table by `IdNumber`.

2.  **Resolve Seller Context**:
    *   Extract Seller from JWT.
    *   **Constraint**: If `SellerChannel == POS` and `IsPostpaid == true`, **Return Error**. (Postpaid is restricted to specific channels).

3.  **Customer Fetch (BSS)**:
    *   Trust BSS data (no local shortcut).
    *   Map `IdNumber`, `Nationality`, `CRMCustomerId`, `Email`.

4.  **Determine Subscription**:
    *   Set `IsPostpaid = true`.
    *   Derive `SubType` = Postpaid.

5.  **TCC Eligibility**:
    *   If `BypassTCC == "0"`, call `CheckEligibility`.
    *   Log full interaction (`TCC_Eligibility_Log`).

#### Onboarding – Create Order

`POST {{onboarding-redbull}}/api/mobile/v1/createOrder` (newSubscriber flow).

This becomes the single place where we store the “Sales Order” data that we previously persisted in our own SalesOrder table.

> **CRITICAL**: We must extend this API request body to include several important fields (see 1.4).

### 1.3 ValidateIdNew – Detailed Flow (Java)

**Response to Seller App**:
Standard `ValidateIdNew` response, but backed by the new Order Management ID.

---

### 1.4 Explicit GAP Statement for createOrder

#### Gap – Onboarding createOrder request body vs required Order data
In the legacy .NET Sales backend, the `SalesOrder` table stored several critical fields (Seller, KYC, OTP context).
In the new architecture, the local order is owned by the `ActionOrder` (Order Management), and Onboarding only orchestrates calls.

The existing `{{onboarding-redbull}}/api/mobile/v1/createOrder` request body **does not carry all the fields** that must end up on the order.

> **Requirement**: The new `createOrder` (or subsequent `updateOrder`) must persist the following into the Microservices Ecosystem:

1.  **Customer / KYC fields** (Stored in Customer Management):
    *   `idNumber`, `idType`, `nationality`
    *   `firstName`, `lastName`, `crmCustomerId`
2.  **Subscription / line type** (Stored in Order Management):
    *   `subType` (Postpaid)
    *   **`isPostpaid`** (Critical for Billing Trigger)
3.  **Seller / Partner context**:
    *   `sellerId`, `partnerId`, `sellerChannel`
4.  **Order meta**:
    *   `status` (NEW)
    *   `orderDate`

---

## 2. Step 2 – Number Selection Flow

**Trigger**: After `ValidateIdNew` completion.
**Endpoint**: `POST /api/Sales/GetMSISDNs`

### 2.1 Logic Changes
*   **Downstream**: Calls `GET {{onboarding-redbull}}/api/inventory/availableNumbers`.
*   **Filtering**: Standard Inventory filtering (Pattern, Level, Type).
*   **No Postpaid Deviation**: Logic is shared with Prepaid.

---

## 3. Step 3 – Plan Selection

**Trigger**: After MSISDN selection.
**Endpoint**: `POST /api/Sales/GetPrimaryPlans`

### 3.1 Logic Changes for Postpaid
*   **Catalog Source**: Strapi CMS / Digital Product Catalog.
*   **Filter**: `PaymentType = POSTPAID`.
*   **Constraint**: Validate `AllowedUsers` and `AllowedOrderTypes`.

> **Important**: The Plan DTO must be enriched with **Commitment Details**:
> *   `CommitmentDuration` (e.g., 12 Months)
> *   `CommitmentAmount` (e.g., Device Price or Vanity Commit)
> *   `PenaltyDetails`

---

## 4. UpdateOrder Journey (The Core Flow)

**Purpose**: Orchestrate the multi-step activation wizard.
**Endpoint**: `POST api/Sales/UpdateOrder`

### Step A: `basic-info`
*   **Fields**: `FirstName`, `LastName`, `Email`.
*   **Action**: Update Customer Profile in `CustomerManagementService`. Set Order Status `InProgress`.

### Step B: `address-info`
*   **Fields**: `Lat`, `Lng`, `City`, `Address`.
*   **Action**: Validate Location. Update Delivery/Service Address.

### Step C: `select-number`
*   **Fields**: `MSISDN`, `IsMNP`, `ExtraSIM` (JSON).
*   **Actions**:
    1.  **Normalize**: Standardize MSISDN format.
    2.  **Wallet Check**:
        *   *Postpaid Note*: Check wallet logic applies if there is an upfront deposit or vanity fee.
    3.  **Port-In Constraint**:
        *   > **CRITICAL**: If `IsMNP == PortIn` (4), **Block Extra SIMs**. Return specific error if present.
    4.  **Reserve**: Call Inventory `operateMsisdn` to reserve the number.

### Step D: `select-plan`
*   **Fields**: `PlanId`, `Addons`.
*   **Actions**:
    1.  **Validate**: ensure Plan is valid Postpaid plan.
    2.  **Commitment**: Calculate `TotalCommitmentValue` (Plan * Duration + Vanity Price).
    3.  **Save**: Persist Commitment requirements to Order context.

### Step E: `sanad-verification` (New Postpaid Step)
> **Purpose**: Mandatory Credit Risk check. The user must accept the Promissory Note (Sanad) via Nafith.

**Logic**:
1.  **Check Status**: Query `credit-check-service` for `SanadStatus` of this Order.
2.  **If Status == APPROVED**:
    *   Proceed to next step.
3.  **If Status != APPROVED**:
    *   Generate Sanad Request: Call `POST /credit-check-service/api/sanad/create`.
    *   **Return Action Required**:
        ```json
        {
          "status": "WAITING_FOR_SANAD",
          "action": "OpenUrl",
          "url": "https://nafith.sa/sanad/sign/..."
        }
        ```
    *   The App must open the URL and poll for completion.

### Step F: `authenticate` (Final Activation)
**Purpose**: Finalize TCC, BSS, and Billing.

**Logic**:
1.  **Gatekeeper Check**:
    *   > **CRITICAL**: Verify `ord.SanadStatus == APPROVED`. If not, **REJECT** activation.
2.  **OTP Verification**:
    *   Use **TCC Ballighny** OTP (instead of generic SMS) for Postpaid binding.
3.  **Activation**:
    *   Call `TCC.NumberMNP` (Port-In) or `TCC.AddNumber`.
    *   Call `BSS.CreateSalesOrder` (Postpaid Profile).
4.  **Billing Trigger**:
    *   Initialize Billing Cycle in BSS/Billing Engine.

---

## 5. Integration Matrix

| Step | Service | Action | Postpaid Specific? |
| :--- | :--- | :--- | :--- |
| **Validate** | `onboarding` | Create Shell Order | `isPostpaid: true` |
| **Select Number** | `inventory` | Reserve Number | Block Extra SIMs on PortIn |
| **Select Plan** | `product-catalog` | Get Plan | Return Commitment Metadata |
| **Sanad** | `credit-check` | **Create/Check Sanad** | **YES (New)** |
| **Auth** | `tcc-integration` | **Ballighny OTP** | **YES** |
| **Auth** | `billing-engine` | Start Cycle | **YES** |

