# Postpaid Activation Line - LLD Document V1

**Version:** 1.0 | **Date:** December 2024  
**Based on:** Legacy .NET Code Reverse Engineering

---

## Key Differences from Prepaid Flow

| Aspect                  | Prepaid                    | Postpaid                                        |
|:------------------------|:---------------------------|:------------------------------------------------|
| **SubType**             | 0                          | 1 (or 2 for Hybrid)                             |
| **IsMNP Values**        | 0, 1, 2                    | 3 (NewSIM), 4 (PortIn), 5 (DataSIM/MBB)         |
| **TCC Eligibility**     | Always required            | Required (except Port-In IsMNP=4)               |
| **OTP Delivery**        | SMS via SMPP               | TCC Ballighny service                           |
| **Promissory Note**     | Never                      | Required for Vanity (MSISDNCost > 0)            |
| **VAT on Plan/Addons**  | Applied (15%)              | NOT Applied                                     |
| **ExtraSIM**            | Allowed                    | Allowed (NOT for Port-In)                       |
| **POS Channel**         | Allowed                    | **NOT Allowed** (SellerChannel=1 blocked)       |
| **TCC Activation**      | `AddNumber`                | `AddNumber` or `NumberMNP` (Port-In)            |
| **BSS Order**           | `CreateSalesOrder`         | `CreateSalesOrder` or `CreateSalesMNPOrder`     |

---

## Step 1 – ValidateIdNew (Postpaid)

### 1.1 Purpose
- Start the postpaid sales onboarding flow for a customer.
- Validate ID, get customer profile from Huawei BSS, check TCC eligibility, and create an Onboarding Order via `createOrder`.
- **Existing Seller App endpoint stays:** `POST /api/Sales/ValidateIdNew`
- IsMNP values for Postpaid: `"3"` (NewSIM), `"4"` (PortIn), `"5"` (DataSIM/MBB)

---

### 1.2 External APIs Used

1. **Huawei BSS – Customer Info / Existence**
   - Same as Prepaid - use BSS as single source of truth.
   - Call `POST {{middleware-service-redbull}}/customer/queryinfo`

2. **TCC Eligibility via Integration**
   - **Port-In (IsMNP=4):** TCC eligibility is **SKIPPED** - order created with `EligibilityTCN = null`
   - **NewSIM/DataSIM:** Call `CheckEligibility` via RBMDigitalCustomIntegration
   - Inputs: `IdNumber, IdType, Nationality, SubType=1, SellerId, PartnerId, SellerChannel`

3. **Onboarding – Create Order**
   - Same as Prepaid: `POST {{onboarding-redbull}}/api/mobile/v1/createOrder`
   - Additional fields for Postpaid: `subType=1`, `isPostpaid=1`

---

### 1.3 ValidateIdNew – Detailed Flow (Java) - Postpaid Specific

**Endpoint:** `POST /api/Sales/ValidateIdNew`

1. **Read configuration and blacklist** (Same as Prepaid)

2. **Resolve seller** (Same as Prepaid)

3. **POS Channel Restriction** ⚠️ **POSTPAID SPECIFIC**
   - If `seller.SellerChannel == 1` (POS) AND `isPostpaid == true`:
     - Return HTTP 400 with code `POSCannotSellPostpaid`
   - **Code Reference:** `SalesAPIController.cs:972-973`

4. **Fetch customer from Huawei BSS** (Same as Prepaid)

5. **Determine subscription context**
   - Set `SubType = 1` (Postpaid) or `2` (Hybrid based on TCC 605 retry)
   - Set `IsPostpaid = 1`
   - From request: `IsMNP` (3, 4, or 5), `IsESim`

6. **TCC eligibility check** ⚠️ **POSTPAID SPECIFIC**
   - **If IsMNP = "4" (Port-In):**
     - **SKIP TCC eligibility check entirely**
     - Set `EligibilityTCN = null`
     - Proceed directly to order creation
   - **If IsMNP = "3" or "5":**
     - Call `CheckEligibility` with `SubType = 1`
     - If code == 605: retry with `SubType = 2` (Hybrid) for Citizens/Residents
     - If code == 600: success, capture `EligibilityTCN`

7. **Create remote Onboarding Order**
   - Same structure as Prepaid with:
     - `subType = 1` (or 2 for Hybrid)
     - `isPostpaid = 1`
     - `activationType = IsMNP` (3, 4, or 5)

---

### 1.4 ValidateIdNew – Integration & Gap Summary (Postpaid)

| Frontend API (Legacy)             | Backend Microservice Endpoint               | Backend Request Change Needed?                                                               | Backend Response Change Needed? | Required Backend Enhancements                         |
|:----------------------------------|:--------------------------------------------|:---------------------------------------------------------------------------------------------|:--------------------------------|:------------------------------------------------------|
| `POST /api/Sales/ValidateIdNew`   | Blacklist: `GET /api/blacklist/{idNumber}`  | No change                                                                                    | No change                       | Same as Prepaid                                       |
|                                   | BSS: `POST /customer/queryinfo`             | No change                                                                                    | No change                       | Same as Prepaid                                       |
|                                   | TCC: `POST /api/tcc/eligibility`            | Must handle **skip for Port-In**: if `isMNP=4`, do NOT call TCC, return `eligibilityTcn=null` | No change                       | Add logic: `if (isMNP == 4) skipEligibility = true`   |
|                                   | Onboarding: `POST /api/mobile/v1/createOrder` | Same fields as Prepaid + ensure `subType=1`, `isPostpaid=1`                                 | No change                       | Same entity changes as Prepaid                        |
|                                   | **NEW: POS Restriction Check**              | Add seller channel validation                                                                | Return 400 if POS + Postpaid    | New validation rule in orchestrator                   |

---

## Step 2 – Number Selection (Postpaid)

### 2.1 GetMSISDNs

**Same as Prepaid** with one difference for Port-In:

| Order Type                  | MSISDN Source                                          |
|:----------------------------|:-------------------------------------------------------|
| PostpaidNewSIM (IsMNP=3)    | `GetMSISDNs` API → Inventory                           |
| PostpaidPortIn (IsMNP=4)    | **User enters existing number** - NO GetMSISDNs call   |
| PostpaidDataSIM (IsMNP=5)   | `GetMSISDNs` API with DATA_SIM type                    |

---

### 2.2 UpdateOrder (step = "select-number") - Postpaid Specific

| Aspect               | Prepaid                           | Postpaid                                                 |
|:---------------------|:----------------------------------|:---------------------------------------------------------|
| **ExtraSIM**         | Allowed                           | Allowed for NewSIM/DataSIM, **NOT ALLOWED for Port-In**  |
| **ExtraSIM Cost**    | First FREE, then 25 SAR + VAT     | First FREE, then 25 SAR **NO VAT**                       |
| **MSISDNCost**       | + VAT                             | + VAT (same)                                             |
| **Port-In MSISDN**   | From GetMSISDNs                   | User-provided existing number                            |
| **OperateMSISDN**    | Called for NewSIM                 | Called for NewSIM, **SKIPPED for Port-In**               |
| **MNPOperator**      | N/A                               | Required for Port-In (donor operator code)               |

**ExtraSIM Restriction for Port-In:**
- **Code Reference:** `SalesAPIController.cs:1215-1216`
- If `IsMNP == 4` (Port-In) AND ExtraSIM requested → Return 400

---

### 2.3 Number Selection – Integration & Gap Summary

| Frontend API (Legacy)                          | Backend Microservice Endpoint      | Request Change                                                 | Response Change  | Required Enhancements                                       |
|:-----------------------------------------------|:-----------------------------------|:---------------------------------------------------------------|:-----------------|:------------------------------------------------------------|
| `POST /api/Sales/GetMSISDNs`                   | `GET /inventory/availableNumbers`  | No change                                                      | No change        | Same as Prepaid                                             |
| `POST /api/Sales/UpdateOrder` (select-number)  | `PUT /inventory/operateMsisdn`     | Add validation: if `isMNP=4`, skip MSISDN reservation          | No change        | Add Port-In logic: accept user MSISDN, validate format only |
|                                                |                                    | Add validation: if `isMNP=4` AND `extraSIM` present → reject   | Error response   | Block ExtraSIM for Port-In                                  |

---

## Step 3 – Plan Selection (Postpaid)

### 3.1 GetPrimaryPlans - Postpaid Specific

**Filters applied (same as Prepaid except):**
- `IsPostpaid = 1` (filter for postpaid plans only)
- `SubType = 1` or `2`
- `AllowedChannels` must NOT include POS (channel 1) for postpaid

**Cost Calculation Difference:**
| Aspect               | Prepaid              | Postpaid           |
|:---------------------|:---------------------|:-------------------|
| Plan Cost            | `Price * (1 + 0.15)` | `Price` (NO VAT)   |
| Addon Cost           | `Price * (1 + 0.15)` | `Price` (NO VAT)   |
| **Code Reference**   | Lines 1270, 1276     | Lines 1270, 1276   |

---

### 3.2 Plan Selection – Integration & Gap Summary

| Frontend API                                 | Backend Endpoint              | Request Change              | Response Change | Enhancements                                   |
|:---------------------------------------------|:------------------------------|:----------------------------|:----------------|:-----------------------------------------------|
| `POST /api/Sales/GetPrimaryPlans`            | `POST /api/rbm/plans/primary` | Add `isPostpaid=1` filter   | No change       | Ensure filter includes `isPostpaid`            |
| `POST /api/Sales/UpdateOrder` (select-plan)  | OrderMgmt update              | No change                   | No change       | Remove VAT from cost calculation for Postpaid  |

---

## Step 4 – Promissory Note (Postpaid ONLY) ⚠️

### 4.1 Purpose
- **Postpaid Specific:** Create promissory note for vanity number payment via Nafith.
- **Condition:** `MSISDNCost > 0` (Vanity number selected)
- **NOT applicable for Port-In** (MSISDNCost always 0)

---

### 4.2 CreateSalesPromissoryNote

**Endpoint:** `POST /api/Sales/CreateSalesPromissoryNote`
**Code Reference:** `SalesAPIController.cs:3222-3280`

**Flow:**
1. Load SalesOrder by OrderId
2. Lookup `CommitmentMatrix` to determine Duration based on plan
3. Call `Nafith.CreateSanad()` with:
   - `IdNumber`, `MSISDN`
   - `TotalAmount = MSISDNCost`
   - `Duration` (months from CommitmentMatrix)
   - `City`
4. Create `PromissoryNote` record with `SanadId`, `SanadStatus = "pending"`
5. Return `{ status: "pending" }`

---

### 4.3 GetPromissoryNoteStatus

**Endpoint:** `POST /api/Sales/GetPromissoryNoteStatus`
**Code Reference:** `SalesAPIController.cs:3293-3304`

**Polling Flow:**
1. Call `Nafith.CheckSanadStatus(SanadId)`
2. Return status: `pending`, `approved`, `rejected`, `cancelled_by_creditor`, `closed`

---

### 4.4 Promissory Note – Integration & Gap Summary

| Frontend API                               | Backend Endpoint                          | Request                                                         | Response                             | Required Enhancements                              |
|:-------------------------------------------|:------------------------------------------|:----------------------------------------------------------------|:-------------------------------------|:---------------------------------------------------|
| `POST /api/Sales/CreateSalesPromissoryNote` | Nafith: `POST /api/nafith/sanad/create`   | `{ orderId, idNumber, msisdn, totalAmount, duration, city }`    | `{ sanadId, status, acceptanceUrl }` | NEW Nafith integration service                     |
| `POST /api/Sales/GetPromissoryNoteStatus`  | Nafith: `GET /api/nafith/sanad/{sanadId}/status` | `{ sanadId }`                                            | `{ status }`                         | NEW endpoint                                       |
|                                            | OrderMgmt:                                | Store `sanadId`, `sanadStatus` on order                         |                                      | Add `sanad_id`, `sanad_status` to CustomerProductOrder |

---

### 4.5 PromissoryNote Entity (NEW)

**Table:** `promissory_note`

| Field              | Type           | Description                   |
|:-------------------|:---------------|:------------------------------|
| `id`               | BIGINT (PK)    | Primary key                   |
| `order_id`         | BIGINT         | FK to CustomerProductOrder    |
| `id_number`        | VARCHAR(50)    | Customer ID                   |
| `msisdn`           | VARCHAR(20)    | Selected MSISDN               |
| `total_amount`     | DECIMAL(10,2)  | Vanity number cost            |
| `duration`         | INT            | Commitment months             |
| `city`             | VARCHAR(100)   | Customer city                 |
| `contact_number`   | VARCHAR(20)    | Customer contact              |
| `sanad_id`         | VARCHAR(100)   | Nafith Sanad ID               |
| `sanad_status`     | VARCHAR(50)    | pending/approved/rejected     |
| `is_test`          | INT            | 0=production, 1=test          |
| `created_at`       | TIMESTAMP      | Creation time                 |
| `updated_at`       | TIMESTAMP      | Last update                   |

---

## Step 5 – OTP Delivery (Postpaid) ⚠️

### 5.1 SendOrderOTP - Postpaid Specific

**Key Difference from Prepaid:**

| Aspect                | Prepaid                                 | Postpaid                                   |
|:----------------------|:----------------------------------------|:-------------------------------------------|
| **Delivery Method**   | SMS via SMPP Gateway                    | **TCC Ballighny Service**                  |
| **Code Reference**    | `sms.SendSMS()`                         | `tcc.SendBallighny(IdNumber, message)`     |
| **Recipient**         | ContactNumber                           | **IdNumber** (via CITC)                    |

**Code Reference:** `SalesAPIController.cs:397-400`
```csharp
if (ord.SubType == 0)  // Prepaid
    sms.SendSMS("0" + contactNumber, message);
else  // Postpaid
    tcc.SendBallighny(ord.IdNumber, message);
```

**Request with Promissory Note:**
- If `WithPN = "1"`: subtract `MSISDNCost` from `OrderTotal` in message

---

### 5.2 OTP – Integration & Gap Summary

| Frontend API                    | Backend Endpoint              | Request Change                                             | Response Change          | Enhancements                          |
|:--------------------------------|:------------------------------|:-----------------------------------------------------------|:-------------------------|:--------------------------------------|
| `POST /api/Sales/SendOrderOTP`  | TCC: `POST /api/tcc/ballighny` | NEW: `{ idNumber, message }`                               | `{ success, messageId }` | NEW Ballighny integration             |
|                                 |                               | Add logic: if `subType == 1` use Ballighny, else use SMS   |                          | Conditional routing in orchestrator   |

---

## Step 6 – Authenticate (Final Activation) - Postpaid Specific

### 6.1 TCC Activation Calls

| Order Type                   | TCC Method       | Code Reference |
|:-----------------------------|:-----------------|:---------------|
| PostpaidNewSIM (IsMNP=3)     | `tcc.AddNumber()` | Lines 1356     |
| PostpaidPortIn (IsMNP=4)     | `tcc.NumberMNP()` | Lines 1350     |
| PostpaidDataSIM (IsMNP=5)    | `tcc.AddNumber()` | Lines 1356     |

---

### 6.2 BSS Order Creation

| Order Type              | BSS Method                | Code Reference |
|:------------------------|:--------------------------|:---------------|
| PostpaidNewSIM          | `bss.CreateSalesOrder()`   | Lines 1424     |
| PostpaidPortIn          | `bss.CreateSalesMNPOrder()` | Lines 1407     |
| PostpaidDataSIM         | `bss.CreateSalesOrder()`   | Lines 1424     |

---

### 6.3 Authentication Method (exceptionFlag)

| Condition                       | exceptionFlag | Auth Fields                       |
|:--------------------------------|:--------------|:----------------------------------|
| Diplomat (IdType=99)            | 1             | None                              |
| UseIAMToken + TokenType=OTP     | 8             | `otp` from `IamOTP`               |
| UseIAMToken + TokenType=Token   | 9             | `iamToken`                        |
| IAMToken present (App auth)     | 11            | `iamAppToken`                     |
| Default (Fingerprint)           | 0             | `fingerIndex`, `fingerImage`      |

**Code Reference:** `TCCApiHelper.cs:719-720`

---

### 6.4 Authenticate – Integration & Gap Summary

| Frontend API                                | Backend Endpoint               | Request Change                                                       | Response Change | Enhancements                          |
|:--------------------------------------------|:-------------------------------|:---------------------------------------------------------------------|:----------------|:--------------------------------------|
| `POST /api/Sales/UpdateOrder` (authenticate) | TCC: `POST /api/tcc/activate` | Add `isMNP` to determine AddNumber vs NumberMNP                      | No change       | Route to correct TCC method           |
|                                             | BSS: `POST /bss/order/create` | Add `isMNP` to determine CreateSalesOrder vs CreateSalesMNPOrder     | No change       | Route to correct BSS method           |
|                                             | Nafith:                       | If PN exists, verify `sanadStatus == approved` before proceeding     |                 | Add PN status validation              |

---

## Step 7 – ExtraSIM (Postpaid) ⚠️

### 7.1 ExtraSIM Rules for Postpaid

| Rule                        | Value                                     |
|:----------------------------|:------------------------------------------|
| Allowed for NewSIM/DataSIM  | Yes                                       |
| Allowed for Port-In         | **NO**                                    |
| First ExtraSIM              | FREE                                      |
| Additional ExtraSIM         | 25 SAR (NO VAT for Postpaid)              |
| ExtraSIM TCC                | `tcc.AddExtraSIM()` (async)               |
| ExtraSIM BSS                | `bss.CreateExtraSIMOrder()` (async)       |

**Code Reference:** `SalesAPIController.cs:1215-1216, 1255-1256`

---

### 7.2 ExtraSIM – Integration & Gap Summary

| Frontend API                        | Backend Endpoint                                             | Request Change                               | Response Change       | Enhancements                  |
|:------------------------------------|:-------------------------------------------------------------|:---------------------------------------------|:----------------------|:------------------------------|
| (Part of UpdateOrder select-number) | Inventory: `GET /inventory/availableNumbers?simType=DATA_SIM` | No change                                    | No change             | Same as Prepaid               |
|                                     | OrderMgmt: Create ExtraSIM records                           | Add validation: reject if `isMNP=4`          | Error if Port-In      | Block ExtraSIM for Port-In    |
|                                     | TCC: `POST /api/tcc/extra-sim` (async)                       | NEW endpoint                                 |                       | NEW background job            |
|                                     | BSS: `POST /bss/extra-sim/order` (async)                     | NEW endpoint                                 |                       | NEW background job            |

---

## UpdateOrder Steps Summary (Postpaid)

| Step            | Fields                                         | Postpaid-Specific Logic                                                                           |
|:----------------|:-----------------------------------------------|:--------------------------------------------------------------------------------------------------|
| `basic-info`    | FirstName, LastName, Email                     | Same as Prepaid                                                                                   |
| `address-info`  | Lat, Lng, City, Address                        | Same as Prepaid                                                                                   |
| `select-number` | MSISDN, MSISDNCost, IsMNP, MNPOperator, ExtraSIM | **Port-In:** Skip OperateMSISDN, block ExtraSIM, require MNPOperator                             |
| `select-plan`   | PlanId, PlanName, Addons                       | **No VAT** on plan/addon costs                                                                    |
| `authenticate`  | ICCID, Auth fields, ExtraSIM ICCIDs            | **TCC:** AddNumber or NumberMNP based on IsMNP; **BSS:** CreateSalesOrder or CreateSalesMNPOrder  |

---

## Entity Changes Summary (Postpaid-Specific)

### Additional Fields on CustomerProductOrder

| Field                  | Type          | Description                              |
|:-----------------------|:--------------|:-----------------------------------------|
| `mnp_operator`         | VARCHAR(20)   | Donor operator code for Port-In          |
| `sanad_id`             | VARCHAR(100)  | Promissory note ID from Nafith           |
| `sanad_status`         | VARCHAR(50)   | Promissory note status                   |
| `commitment_duration`  | INT           | Months from CommitmentMatrix             |

### New Entity: CommitmentMatrix

| Field              | Type         | Description         |
|:-------------------|:-------------|:--------------------|
| `id`               | BIGINT (PK)  | Primary key         |
| `plan_id`          | VARCHAR(50)  | Plan/offer ID       |
| `msisdn_tier`      | INT          | Vanity level        |
| `duration_months`  | INT          | Commitment period   |

---

## Microservice Endpoint Changes (Postpaid-Specific)

### 1. TCC Service

| Endpoint                          | New/Existing | Purpose                                  |
|:----------------------------------|:-------------|:-----------------------------------------|
| `POST /api/tcc/eligibility`       | Existing     | Add logic to SKIP for Port-In            |
| `POST /api/tcc/activate`          | Existing     | Route to AddNumber or NumberMNP          |
| `POST /api/tcc/number-mnp`        | NEW          | Port-In activation                       |
| `POST /api/tcc/ballighny`         | NEW          | Postpaid OTP delivery                    |
| `POST /api/tcc/extra-sim`         | NEW          | ExtraSIM activation (async)              |

### 2. Nafith Service (NEW)

| Endpoint                              | Purpose                 |
|:--------------------------------------|:------------------------|
| `POST /api/nafith/sanad/create`       | Create promissory note  |
| `GET /api/nafith/sanad/{id}/status`   | Check PN status         |
| `POST /api/nafith/sanad/{id}/cancel`  | Cancel PN               |

### 3. BSS Integration

| Endpoint                        | New/Existing | Purpose                                                      |
|:--------------------------------|:-------------|:-------------------------------------------------------------|
| `POST /bss/order/create`        | Existing     | Route to CreateSalesOrder or CreateSalesMNPOrder             |
| `POST /bss/order/mnp`           | NEW          | Port-In order creation                                       |
| `POST /bss/extra-sim/order`     | NEW          | ExtraSIM order (async)                                       |

### 4. Order Management

| Endpoint                         | Change                                                               |
|:---------------------------------|:---------------------------------------------------------------------|
| `POST /api/rbm/orders/update`    | Add PN validation before authenticate step                           |
|                                  | Add Port-In validations (skip MSISDN lock, block ExtraSIM)           |
|                                  | Remove VAT from Postpaid cost calculations                           |

---

## Items Needing Further Investigation 🔍

Based on the Prepaid LLD gaps, these need confirmation for Postpaid:

| #   | Item                                | What's Needed                                      |
|:----|:------------------------------------|:---------------------------------------------------|
| 1   | `GetAddons` for Postpaid            | Confirm same service or different filtering        |
| 2   | `EligibleAddons` for Postpaid       | Confirm addon eligibility rules                    |
| 3   | Nafith integration details          | Request/response payload from integration team     |
| 4   | TCC `NumberMNP` vs `AddNumber`      | Confirm request payload differences                |
| 5   | BSS `CreateSalesMNPOrder`           | Confirm request payload vs CreateSalesOrder        |
| 6   | CommitmentMatrix data source        | Strapi CMS or separate table?                      |
| 7   | ExtraSIM async activation flow      | Background job or saga pattern?                    |
| 8   | Ballighny message format            | Template for OTP message                           |

---

## Sequence Diagram (Postpaid New SIM with Vanity)

```
Seller App                    Sales API                    Onboarding               TCC                BSS                Nafith
    │                             │                            │                     │                  │                    │
    │─── ValidateIdNew ──────────▶│                            │                     │                  │                    │
    │    (IsMNP=3, IsPostpaid)    │                            │                     │                  │                    │
    │                             │──── Check Blacklist ──────▶│                     │                  │                    │
    │                             │                            │                     │                  │                    │
    │                             │──── Get Customer ─────────▶│─────────────────────┼─────────────────▶│                    │
    │                             │                            │                     │                  │                    │
    │                             │──── Check Eligibility ────▶│────────────────────▶│                  │                    │
    │                             │                            │                     │                  │                    │
    │                             │──── Create Order ─────────▶│                     │                  │                    │
    │◀──────── Order Created ─────│                            │                     │                  │                    │
    │                             │                            │                     │                  │                    │
    │─── UpdateOrder (basic) ────▶│                            │                     │                  │                    │
    │─── UpdateOrder (address) ──▶│                            │                     │                  │                    │
    │                             │                            │                     │                  │                    │
    │─── GetMSISDNs ─────────────▶│────── Get Numbers ────────▶│                     │                  │                    │
    │◀──────── MSISDN List ───────│                            │                     │                  │                    │
    │                             │                            │                     │                  │                    │
    │─── UpdateOrder (number) ───▶│                            │                     │                  │                    │
    │    (Vanity MSISDNCost>0)    │──── Reserve MSISDN ───────▶│                     │                  │                    │
    │                             │                            │                     │                  │                    │
    │─── GetPrimaryPlans ────────▶│                            │                     │                  │                    │
    │─── UpdateOrder (plan) ─────▶│                            │                     │                  │                    │
    │                             │                            │                     │                  │                    │
    │─── CreatePromissoryNote ───▶│                            │                     │                  │                    │
    │                             │──────────────────────────────────────────────────────────────────────────────────────────▶│
    │                             │                            │                     │                  │    Create Sanad    │
    │◀──────── PN Pending ────────│                            │                     │                  │                    │
    │                             │                            │                     │                  │                    │
    │   [Customer signs on Nafith]│                            │                     │                  │                    │
    │                             │                            │                     │                  │                    │
    │─── GetPNStatus (poll) ─────▶│──────────────────────────────────────────────────────────────────────────────────────────▶│
    │◀──────── Approved ──────────│                            │                     │                  │                    │
    │                             │                            │                     │                  │                    │
    │─── SendOrderOTP ───────────▶│                            │                     │                  │                    │
    │                             │──────────────────────────────────────────────────▶│ SendBallighny   │                    │
    │◀──────── OTP Sent ──────────│                            │                     │                  │                    │
    │                             │                            │                     │                  │                    │
    │─── UpdateOrder (auth) ─────▶│                            │                     │                  │                    │
    │                             │─────────────────────────────────────────────────▶│ AddNumber        │                    │
    │                             │─────────────────────────────────────────────────────────────────────▶│ CreateSalesOrder   │
    │                             │──── Deduct Wallet ────────▶│                     │                  │                    │
    │                             │──── Generate eContract ───▶│                     │                  │                    │
    │◀──────── Completed ─────────│                            │                     │                  │                    │
```

---

## Sequence Diagram (Postpaid Port-In)

```
Seller App                    Sales API                    Onboarding               TCC                BSS
    │                             │                            │                     │                  │
    │─── ValidateIdNew ──────────▶│                            │                     │                  │
    │    (IsMNP=4)                │                            │                     │                  │
    │                             │──── Check Blacklist ──────▶│                     │                  │
    │                             │──── Get Customer ─────────▶│──────────────────────────────────────▶│
    │                             │                            │                     │                  │
    │                             │   *** SKIP TCC Eligibility ***                   │                  │
    │                             │                            │                     │                  │
    │                             │──── Create Order ─────────▶│                     │                  │
    │◀──────── Order Created ─────│   (EligibilityTCN=null)    │                     │                  │
    │                             │                            │                     │                  │
    │─── UpdateOrder (number) ───▶│                            │                     │                  │
    │    (User MSISDN, MNPOp)     │   *** SKIP OperateMSISDN ***                     │                  │
    │                             │   *** REJECT ExtraSIM ***  │                     │                  │
    │                             │                            │                     │                  │
    │─── UpdateOrder (auth) ─────▶│                            │                     │                  │
    │                             │─────────────────────────────────────────────────▶│ NumberMNP       │
    │                             │─────────────────────────────────────────────────────────────────────▶│ CreateSalesMNPOrder
    │◀──────── Completed ─────────│                            │                     │                  │
```

---

**Document End**
