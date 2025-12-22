# [LLD] Postpaid Activation Line

> **Document Status**: Draft
> **Author**: System Architect
> **Type**: Low Level Design (LLD)
> **Baseline**: Aligned with [Prepaid.md](file:///d:/ReposDOTD/Prepaid.md) standard.

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
    *   Map `IdNumber`, `Nationality`, `CRMCustomerId`, [Email](file:///d:/ReposDOTD/RedBullSalesPortalRestSharp/RedBullSalesPortal/RedBullSalesPortal.Web/Modules/SalesAPI/SalesAPIController.cs#100-110).

4.  **Determine Subscription**:
    *   Set `IsPostpaid = true`.
    *   Derive `SubType` = Postpaid.

5.  **TCC Eligibility**:
    *   If `BypassTCC == "0"`, call [CheckEligibility](file:///d:/ReposDOTD/RedBullSalesPortalRestSharp/RedBullSalesPortal/RedBullSalesPortal.Web/Modules/SalesAPI/SalesAPIController.cs#2626-2632).
    *   Log full interaction (`TCC_Eligibility_Log`).

6.  **Create Remote Onboarding Order**:
    *   Call `POST {{onboarding-redbull}}/api/mobile/v1/createOrder`.
    *   **Payload Extension**:
        *   `isPostpaid: true`
        *   `subType`: Postpaid
        *   `eligibilityTcn`: from TCC response
        *   `crmCustomerId`: from BSS
        *   `status`: "NEW"

**Response to Seller App**:
Standard [ValidateIdNew](file:///d:/ReposDOTD/RedBullSalesPortalRestSharp/RedBullSalesPortal/RedBullSalesPortal.Web/Modules/SalesAPI/SalesAPIController.cs#921-1095) response, but backed by the new Order Management ID.

---

## 2. Step 2 – Number Selection Flow

**Trigger**: After [ValidateIdNew](file:///d:/ReposDOTD/RedBullSalesPortalRestSharp/RedBullSalesPortal/RedBullSalesPortal.Web/Modules/SalesAPI/SalesAPIController.cs#921-1095) completion.
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
*   **Fields**: `FirstName`, `LastName`, [Email](file:///d:/ReposDOTD/RedBullSalesPortalRestSharp/RedBullSalesPortal/RedBullSalesPortal.Web/Modules/SalesAPI/SalesAPIController.cs#100-110).
*   **Action**: Update Customer Profile in `CustomerManagementService`. Set Order Status `InProgress`.

### Step B: `address-info`
*   **Fields**: `Lat`, `Lng`, [City](file:///d:/ReposDOTD/RedBullSalesPortalRestSharp/RedBullSalesPortal/RedBullSalesPortal.Web/Modules/Common/Helpers/NafithHelper.cs#65-73), `Address`.
*   **Action**: Validate Location. Update Delivery/Service Address.

### Step C: `select-number`
*   **Fields**: [MSISDN](file:///d:/ReposDOTD/RedBullSalesPortalRestSharp/RedBullSalesPortal/RedBullSalesPortal.Web/Modules/SalesAPI/SalesAPIController.cs#2983-2999), `IsMNP`, [ExtraSIM](file:///d:/ReposDOTD/RedBullSalesPortalRestSharp/RedBullSalesPortal/RedBullSalesPortal.Web/Modules/SalesAPI/SalesAPIController.cs#2897-2920) (JSON).
*   **Actions**:
    1.  **Normalize**: Standardize MSISDN format.
    2.  **Wallet Check**:
        *   *Postpaid Note*: Check wallet logic applies if there is an upfront deposit or vanity fee.
    3.  **Port-In Constraint**:
        *   > **CRITICAL**: If `IsMNP == PortIn` (4), **Block Extra SIMs**. Return specific error if present.
    4.  **Reserve**: Call Inventory `operateMsisdn` to reserve the number.

### Step D: `select-plan`
*   **Fields**: `PlanId`, [Addons](file:///d:/ReposDOTD/RedBullSalesPortalRestSharp/RedBullSalesPortal/RedBullSalesPortal.Web/Modules/SalesAPI/SalesAPIController.cs#2458-2498).
*   **Actions**:
    1.  **Validate**: ensure Plan is valid Postpaid plan.
    2.  **Commitment**: Calculate `TotalCommitmentValue` (Plan * Duration + Vanity Price).
    3.  **Save**: Persist Commitment requirements to Order context.

### Step E: `sanad-verification` (New Postpaid Step)
> **Purpose**: Mandatory Credit Risk check. The user must accept the Promissory Note (Sanad) via Nafith.

**Logic**:
1.  **Check Status**: Query `credit-check-service` for [SanadStatus](file:///d:/ReposDOTD/RedBullSalesPortalRestSharp/RedBullSalesPortal/RedBullSalesPortal.Web/Modules/Common/Helpers/NafithHelper.cs#154-207) of this Order.
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
| **Validate** | [onboarding](file:///d:/ReposDOTD/onboarding) | Create Shell Order | `isPostpaid: true` |
| **Select Number** | `inventory` | Reserve Number | Block Extra SIMs on PortIn |
| **Select Plan** | `product-catalog` | Get Plan | Return Commitment Metadata |
| **Sanad** | `credit-check` | **Create/Check Sanad** | **YES (New)** |
| **Auth** | `tcc-integration` | **Ballighny OTP** | **YES** |
| **Auth** | `billing-engine` | Start Cycle | **YES** |

