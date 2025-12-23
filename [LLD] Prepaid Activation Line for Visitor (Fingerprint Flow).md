# [LLD] Prepaid Activation Line for Visitor (Fingerprint Flow)

**Document Version:** 2.0 (Corrected based on Code Analysis)
**Status:** Manually Approved 

---


## 1. Purpose

This document details the **Low-Level Design (LLD)** for the **Prepaid Activation Line for Visitor** flow.
Unlike the Citizen/Resident flow which relies on Nafath/IAM, the **Standard Visitor Flow** uses **Fingerprint Verification** directly integrated with **TCC (Semati)**.

## 2. Scope

| Aspect | Details |
|--------|---------|
| **Target Audience** | Visitors (Tourists, Pilgrims, GCC) |
| **Authentication** | **Biometric Fingerprint** (Collected via Scanner) |
| **Integration** | Direct TCC (Semati) Integration |
| **Identity Document** | Passport Number |
| **Prerequisites** | Seller must be authorized (UserId 2095) |

## 3. Visitor Identification Logic

The system identifies a "Visitor" based on the following logic in `SalesAPIController.cs`:

1.  **Seller Restriction**: The logged-in seller must have `UserId == 2095`.
2.  **ID Type Restriction**: The Customer's `IdType` must **NOT** be:
    *   `1` (Citizen)
    *   `2` (Resident)

### Supported Visitor ID Types

| IdType (Enum) | Description | Identifier Used |
|---------------|-------------|-----------------|
| `3` | Visitor | Passport Number |
| `4` | GCC Passport | Passport Number |
| `5` | GCC National ID | GCC National ID |
| `6` | Pilgrim Passport | Passport Number |
| `7` | Pilgrim Border | Border Number |
| `8` | Umrah Passport | Passport Number |
| `9` | Visitor Visa | Visa Number |
| `10` | Umrah Visa | Visa Number |
| `11` | Hajj Visa | Visa Number |
| `99` | Diplomat | Diplomatic ID |

---

## 4. Workflows

### 4.1 Step 1: Validation & Onboarding (`ValidateIdNew`)

**Endpoint:** `POST /api/Sales/ValidateIdNew`

This step validates the visitor's initial data and checks eligibility with TCC.

1.  **Input Parameters**:
    *   `IdNumber`: **Passport Number** (Passed in this field).
    *   `IdType`: Visitor Type (e.g., `3`).
    *   `Nationality`: TCC Nationality Code (Integer, e.g., `156`).
    
2.  **Process**:
    *   **Visitor Check**: Validates `IsVisitor` condition.
    *   **BSS Lookup**: Checks if customer exists in BSS (likely returns not found for new visitors, potentially creating a lead).
    *   **TCC Eligibility**: Calls `tcc.CheckEligibility`.
        *   **Payload**: Uses Passport Number + Nationality ID (Int).
        *   **Note**: Does *not* require AlphaCode for this specific TCC check in the current implementation.
    *   **Order Creation**: Creates a `SalesOrder` record.
        *   `ord.IdNumber` = Passport Number.
        *   `ord.Nationality` = Nationality ID.

### 4.2 Step 2: Order Details Update (`UpdateOrder`)

The order goes through standard steps: `basic-info` (Name/Email), `address-info` (Location), `select-number` (MSISDN selection).

**Key Visitor Specifics**:
*   **Name**: Typically entered manually or OCR-scanned from Passport (since BSS might not have it).
*   **Address**: Visitor's temporary address in Saudi Arabia.

### 4.3 Step 3: Activation & Authentication (`UpdateOrder` - `authenticate`)

**Endpoint:** `POST /api/Sales/UpdateOrder` (step: `authenticate`)

This is the critical step where biometric verification occurs.

1.  **Input Parameters**:
    *   `FingerImage`: Base64 encoded fingerprint image (captured by scanner).
    *   `FingerIndex`: Index of the finger (e.g., `1` for Right Thumb).
    *   `UseIAMToken`: **False/0** (Indicates Fingerprint flow).
    *   `ICCID`: SIM Card Serial Number.

2.  **Logic (`SalesAPIController.cs`)**:
    *   **Determine Flow**:
        *   Since `UseIAMToken` is `0`, the system prepares for Fingerprint submission.
    *   **TCC Submission**: Calls `tcc.AddNumber` (or `NumberMNP` for port-ins).

3.  **TCC Integration (`TCCApiHelper.AddNumber`)**:
    *   **Endpoint**: `/TCC-Web/api-tcc/individual/v2/verify` (Semati).
    *   **Request Payload Construction**:
        ```json
        {
          "apiKey": "string",
          "token": "seller.TccToken",
          "requestType": 1,
          "person": {
            "personId": "{PassportNumber}",
            "IdType": 3,
            "nationality": 156,
            "fingerIndex": {FingerIndex},
            "fingerImage": "{Base64Image}", 
            "exceptionFlag": 0,
            "otp": null,
            "iamToken": null,
            "iamAppToken": null
          },
          "mobileNumber": {
            "msisdn": "9665XXXXXXXX",
            "msisdnType": "V", // or 'D'
            "simList": [
              {
                "iccid": "string",
                "imsi": "string",
                "eSim": false
              }
            ],
            "subscriptionType": 0,
            "isDefault": false
          },
          "Operator": {
            "sourceType": 1,
            "sourceId": "orgNumber",
            "operatorTCN": "yyyyMMddHHmmssfff",
            "region": "01",
            "employeeIdType": 1,
            "employeeUsername": "string",
            "employeeId": "string",
            "deviceId": "string",
            "branchAddress": "lat,lng"
          }
        }
        ```
    *   **Verification**: TCC Backend validates the fingerprint against the border/entry records (NIC integration handled by TCC).

4.  **Outcome**:
    *   **Success**: TCC returns `code: 600`.
        *   System updates Border Number if returned by TCC.
        *   System proceeds to create BSS Order (`bss.CreateSalesOrder`).
    *   **Failure**: Returns error message (e.g., "Fingerprint mismatch").

---

## 5. Technical Data Model

### 5.1 `SalesOrder` Entity (Reused)

No new "Visitor" specific entity is created; the existing `SalesOrder` is used with specific data mapping:

| Field | Usage for Visitor |
|-------|-------------------|
| `IdNumber` | Stores **Passport Number** (or Border Number after update). |
| `IdType` | Stores Visitor Type enum (e.g., `3`). |
| `Nationality` | Stores Integer ID from `TCCNationality`. |
| `FingerImage` | Stores the biometric data temporarily for transmission. |
| `UseIAMToken` | Set to `0` (False). |

### 5.2 External APIs

| Service | Method | Endpoint | Purpose |
|---------|--------|----------|---------|
| **TCC** | `CheckEligibility` | `/TCC-Web/api/eligibility` | Initial vetting of Passport/Nationality. |
| **TCC** | `AddNumber` | `/TCC-Web/api-tcc/individual/v2/verify` | **Biometric Activation**. Submits fingerprint and passport. |

---

## 6. Correction from Previous Assumptions

*   **No Direct Absher/Nafath App**: The standard visitor flow does **not** trigger a Nafath App push notification or require the visitor to log in to Absher. It relies on the biometric scanner at the point of sale.
*   **No AlphaCode Dependency**: The standard flow uses the numeric `Nationality` ID, not the 3-letter ISO code (AlphaCode is only used in the secondary `NafathAppRequest` endpoint which is not part of this primary flow).
*   **Passport Field**: There is no separate `Passport` column in the database; `IdNumber` is overloaded to hold the Passport Number.
