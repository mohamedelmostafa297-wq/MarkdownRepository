# [LLD] Prepaid Activation Line for Visitor using Absher (UC-6)

**Document Version:** 1.0
**Author:** Based on Reverse Engineering from RedbullSalesPortalRestSharp
**Date:** December 2024
**Status:** Draft

---

## Table of Contents
1. [Purpose](#1-purpose)
2. [Scope](#2-scope)
3. [Visitor ID Types](#3-visitor-id-types)
4. [Key Differences from Citizen/Resident Flow](#4-key-differences-from-citizenresident-flow)
5. [Step-by-Step Flow](#5-step-by-step-flow)
6. [External APIs Used](#6-external-apis-used)
7. [Sequence Diagrams](#7-sequence-diagrams)
8. [Integration & Gap Summary](#8-integration--gap-summary)
9. [Microservice Entity Changes](#9-microservice-entity-changes)
10. [Error Handling](#10-error-handling)

---

## 1. Purpose

This LLD describes the activation flow for **Prepaid lines for Visitors** using **Absher (Nafath)** biometric authentication.

A **Visitor** is any customer who is:
- Not a Saudi Citizen (IdType ≠ 1)
- Not a Resident/Iqama holder (IdType ≠ 2)

Visitors include tourists, pilgrims, temporary visa holders, and GCC nationals visiting Saudi Arabia.

---

## 2. Scope

| Aspect | Details |
|--------|---------|
| **Activation Type** | `PrepaidNewSIM` (0), `PrepaidDataSIM` (2) |
| **Customer Type** | Visitor (non-Citizen, non-Resident) |
| **SIM Type** | Physical SIM (eSIM may be supported based on plan) |
| **Authentication** | Absher / Nafath App with Passport + Nationality |
| **ID Document** | Passport Number (not National ID or Iqama) |

---

## 3. Visitor ID Types

The system supports the following visitor ID types:

| IdType Value | Description (EN) | Description (AR) | Identifier Used |
|--------------|------------------|------------------|-----------------|
| 3 | Visitor | زائر | Passport Number |
| 4 | GCC Passport | جواز خليجي | Passport Number |
| 5 | GCC National ID | هوية خليجية | GCC National ID |
| 6 | Pilgrim Passport | جواز حاج | Passport Number |
| 7 | Pilgrim Border | رقم حدود حاج | Border Number |
| 8 | Umrah Passport | جواز معتمر | Passport Number |
| 9 | Visitor Visa | تأشيرة زيارة | Visa Number |
| 10 | Umrah Visa | تأشيرة عمرة | Visa Number |
| 11 | Hajj Visa | تأشيرة حج | Visa Number |
| 99 | Diplomat | دبلوماسي | Passport Number |

### IdType Enum (from SalesOrder.cs)
```csharp
 public enum IdType 
 {
     Citizen = 1,
     Resident = 2,
     Visitor = 3,
     GCC_Passport = 4,
     GCC_National_Id = 5,
     Pilgrim_Passport = 6,
     Pilgrim_Border = 7,
     Umrah_Passport = 8,
     Visitor_Visa = 9,
     Umrah_Visa = 10,
     Haj_Visa = 11,
     Diplomat = 99
 }
```

---

## 4. Key Differences from Citizen/Resident Flow

| Aspect | Citizen/Resident | Visitor |
|--------|------------------|---------|
| **ID Field** | `IdNumber` (National ID / Iqama) | `Passport` (Passport Number) |
| **Nationality** | Auto-detected from ID | **Required** - Must be provided explicitly |
| **Nafath Request** | `id` only | `id` + `nationality` (AlphaCode) |
| **Validation Logic** | TCC CheckEligibility with IdNumber | TCC CheckEligibility with Passport + Nationality |
| **AlphaCode Lookup** | Not required | Required from `TCCNationality` table |
| **Eligible Sellers** | All authorized sellers | Specific sellers (e.g., UserId == 2095 for visitor handling) |

### Visitor Detection Logic (from SalesAPIController.cs:3072)
```csharp
bool IsVisitor = (user.Id == "2095")
    && (param.OrderData?.idType != null)
    && ((int)param.OrderData?.idType != 1)   // Not Citizen
    && ((int)param.OrderData?.idType != 2);  // Not Resident
```

---

## 5. Step-by-Step Flow

### Step 1 – ValidateIdNew (Visitor Variant)

#### 1.1 Purpose
Start the sales onboarding flow for a **Visitor** customer.

Validate Passport, get customer profile from Huawei BSS, check TCC eligibility with nationality, and create an Onboarding Order via `createOrder`.

#### 1.2 Request Body (Extended for Visitor)

```json
{
  "idNumber": null,
  "passport": "AB1234567",
  "idType": 3,
  "nationality": 156,
  "isMNP": false,
  "isESim": false
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `idNumber` | string | No | Not used for visitors (null) |
| `passport` | string | **Yes** | Passport number of the visitor |
| `idType` | int | **Yes** | Must be one of: 3, 4, 5, 6, 7, 8, 9, 10, 11, 99 |
| `nationality` | int | **Yes** | TCC Nationality code (maps to AlphaCode) |
| `isMNP` | boolean | No | Port-in flag (usually false for visitors) |
| `isESim` | boolean | No | eSIM flag |

#### 1.3 ValidateIdNew – Detailed Flow (Java)

**Endpoint:** `POST /api/Sales/ValidateIdNew`

1. **Read configuration and blacklist**
   - Check `DisableActivation` from config
   - Check BlackList table by Passport number (not IdNumber)

2. **Resolve seller**
   - Extract seller from JWT
   - **Important:** Verify seller is authorized for visitor activations
   - Load seller record to get: SellerId, SellerChannel, PartnerId

3. **Determine if Visitor**
   ```java
   boolean isVisitor = (idType != IdType.CITIZEN) && (idType != IdType.RESIDENT);
   ```

4. **Fetch Nationality AlphaCode**
   ```java
   if (isVisitor) {
       TCCNationality nat = tccNationalityRepository.findById(nationalityCode);
       String alphaCode = nat.getAlphaCode(); // e.g., "EGY", "PAK", "IND"
   }
   ```

5. **Fetch customer from Huawei BSS**
   - For visitors, use Passport + Nationality for lookup
   - Call: `POST {{middleware-service-redbull}}/customer/queryinfo`
   - Request:
     ```json
     {
       "idNumber": "AB1234567",
       "idType": 3,
       "nationality": 156
     }
     ```

6. **TCC eligibility check (Visitor-specific)**
   - If `BypassTCC == "0"`:
     ```java
     tccResponse = tcc.CheckEligibility(
         passport,           // Passport number instead of IdNumber
         nationalityCode,    // Nationality code
         idType,             // Visitor IdType (3, 4, 5, etc.)
         subType,            // 0 for Prepaid
         seller
     );
     ```

7. **Create remote Onboarding Order**
   - Call `POST {{onboarding-redbull}}/api/mobile/v1/createOrder`
   - Include visitor-specific fields:
     ```json
     {
       "idNumber": null,
       "passport": "AB1234567",
       "idType": 3,
       "nationality": 156,
       "nationalityAlphaCode": "EGY",
       "isVisitor": true,
       "subType": 0,
       "isPostpaid": 0,
       "sellerId": 2095,
       "partnerId": 1
     }
     ```

#### 1.4 Response to Seller App
```json
{
  "custInfo": {
    "passport": "AB1234567",
    "idType": 3,
    "nationality": 156,
    "nationalityName": "Egypt",
    "firstName": "Ahmed",
    "lastName": "Mohamed",
    "isVisitor": true
  },
  "tccEligibility": {
    "code": 600,
    "message": "success",
    "eligibilityTCN": "e81e370a-c021-49ca-af46-62e4285fe3d7"
  },
  "order": {
    "orderId": 12345,
    "subType": 0,
    "isPostpaid": 0,
    "isESim": 0,
    "otp": null,
    "otpExpiry": null
  }
}
```

---

### Step 2 – Number Selection (Same as Citizen/Resident)

Same as standard flow - no visitor-specific changes.

---

### Step 3 – Plan Selection (Same as Citizen/Resident)

Same as standard flow - no visitor-specific changes.

**Note:** Some plans may have `AllowedIDTypes` restrictions that exclude visitors. The filtering logic should check:
```java
if (!plan.getAllowedIDTypes().equals("*")) {
    List<Integer> allowedTypes = parseAllowedIDTypes(plan.getAllowedIDTypes());
    if (!allowedTypes.contains(order.getIdType())) {
        // Exclude this plan for visitor
    }
}
```

---

### Step 4 – UpdateOrder Steps (basic-info, address-info, select-number, select-plan)

Same as standard flow with these visitor-specific considerations:

#### 4.1 basic-info Step
- For visitors, the name fields may come from Passport data
- May need to handle Arabic/English name variants

#### 4.2 address-info Step
- Visitors may have temporary addresses (hotel, etc.)
- Location validation applies same as citizens

#### 4.3 select-number Step
- Same MSISDN selection logic
- Same wallet check

#### 4.4 select-plan Step
- Filter plans by `AllowedIDTypes` containing visitor IdType
- Same pricing and addon logic

---

### Step 5 – Authenticate (Nafath/Absher for Visitor) - CRITICAL DIFFERENCE

#### 5.1 Purpose
Final activation step using **Nafath App** with **Passport + Nationality**.

#### 5.2 Nafath Request for Visitor

**Endpoint:** `POST /api/Sales/NafathAppRequest`

**Request Processing (from SalesAPIController.cs:3056-3087):**

```java
// Determine if visitor
boolean isVisitor = (idType != IdType.CITIZEN) && (idType != IdType.RESIDENT);

String nationality = null;
String idNumber = order.getIdNumber();

if (isVisitor) {
    // Get AlphaCode from TCCNationality table
    TCCNationality nat = tccNationalityRepository.findById(order.getNationality());
    nationality = nat.getAlphaCode();  // e.g., "EGY", "PAK", "IND"
    idNumber = order.getPassport();     // Use passport instead of IdNumber
}

// Create Nafath request
NafathAppRequest req = new NafathAppRequest();
req.setId(idNumber);                    // Passport for visitors
req.setNationality(nationality);        // AlphaCode (only for visitors)
req.setService("AddNumber");            // Or appropriate service
req.setAction("SpRequest");
```

#### 5.3 NafathAppRequest DTO

```java
public class NafathAppRequest {
    private String id;           // Passport number for visitors
    private String nationality;  // AlphaCode (e.g., "EGY") - ONLY for visitors
    private String action = "SpRequest";
    private String service;      // "AddNumber", "MNP", "LoginWithBio", etc.
}
```

| Field | Citizen/Resident | Visitor |
|-------|------------------|---------|
| `id` | National ID / Iqama | Passport Number |
| `nationality` | `null` | AlphaCode from TCCNationality |
| `service` | "AddNumber" | "AddNumber" |
| `action` | "SpRequest" | "SpRequest" |

#### 5.4 Nafath Flow for Visitor

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  Seller App │────>│  Onboarding │────>│   Nafath    │────>│ Customer's  │
│             │     │   Service   │     │   Service   │     │ Nafath App  │
└─────────────┘     └─────────────┘     └─────────────┘     └─────────────┘
      │                   │                   │                   │
      │ NafathAppRequest  │                   │                   │
      │ {passport,        │                   │                   │
      │  nationality:     │                   │                   │
      │  "EGY"}          │                   │                   │
      │──────────────────>│                   │                   │
      │                   │ CreateNafathReq   │                   │
      │                   │ {id: passport,    │                   │
      │                   │  nationality:     │                   │
      │                   │  "EGY"}          │                   │
      │                   │──────────────────>│                   │
      │                   │                   │ Push notification │
      │                   │                   │──────────────────>│
      │                   │   {random: 45}    │                   │
      │                   │<──────────────────│                   │
      │   {random: 45}    │                   │                   │
      │<──────────────────│                   │                   │
      │                   │                   │                   │
      │ [Seller shows random number to customer]                  │
      │                   │                   │                   │
      │                   │                   │ Customer confirms │
      │                   │                   │ random in app     │
      │                   │                   │<──────────────────│
      │                   │                   │                   │
      │ CheckNafathStatus │                   │                   │
      │──────────────────>│                   │                   │
      │                   │ GetNafathCallback │                   │
      │                   │──────────────────>│                   │
      │                   │ {status:COMPLETED,│                   │
      │                   │  token: "xxx"}    │                   │
      │                   │<──────────────────│                   │
      │  {status: 50}     │                   │                   │
      │<──────────────────│                   │                   │
      │                   │                   │                   │
      │ Continue to TCC AddNumber...          │                   │
```

#### 5.5 TCC AddNumber for Visitor

After Nafath authentication is complete, the TCC AddNumber call includes:

```java
// For visitors, use Nafath App Token with exceptionFlag = 11
TccAddNumberRequest request = new TccAddNumberRequest();
request.setIdNumber(order.getPassport());
request.setIdType(order.getIdType());
request.setNationality(order.getNationality());
request.setMsisdn(order.getMsisdn());
request.setIccid(order.getIccid());

// Biometric info - using Nafath App Token
request.setBiometricInfo(new BiometricInfo()
    .setFingerIndex(null)
    .setFingerImage(null)
    .setExceptionFlag(11)           // Flag for Nafath App authentication
    .setIamAppToken(nafathAppToken) // Token from Nafath callback
);
```

---

## 6. External APIs Used

### 6.1 TCCNationality Lookup (Internal Database)

**Purpose:** Get AlphaCode for visitor's nationality

**Table:** `TCCNationality`

| Field | Type | Description |
|-------|------|-------------|
| `Id` | short | Nationality code (e.g., 156 for Egypt) |
| `NameEn` | string | English name |
| `NameAr` | string | Arabic name |
| `AlphaCode` | string | ISO Alpha code (e.g., "EGY", "PAK") |

**Query:**
```sql
SELECT AlphaCode FROM TCCNationality WHERE Id = @nationalityCode
```

### 6.2 Nafath/Absher API (TCC Integration)

**Endpoints (from appsettings.json:122-125):**
- **Production:** `POST https://iamservices.semati.sa/nafath/api/v1/client/authorize/`
- **Test:** `POST https://test-iamservices.semati.sa/nafath/api/v2/client/authorize/`

**Authentication:** `Authorization: apikey {Nafath2APIKey}`

**Request (Visitor):**
```json
{
  "id": "AB1234567",
  "nationality": "EGY",
  "action": "SpRequest",
  "service": "AddNumber"
}
```

**Response (from ExternalIntegrations_Ar.md:928-933):**
```json
{
  "random": "string (TransactionId)",
  "message": "string",
  "code": "int"
}
```

### 6.3 Nafath Callback Status

**Method:** `GetNafathCallbackStatus(string TransactionId)`

**Endpoint (from ExternalIntegrations_Ar.md:956):** `POST https://api.redbullmobile.sa/api/Semati/checkStatus`

**Note:** This is an internal endpoint in the system.

**Request (from ExternalIntegrations_Ar.md:961-964):**
```json
{
  "TransactionId": "string"
}
```

**Response:** Returns the status of the Nafath authentication request.

### 6.4 TCC CheckEligibility (Visitor Variant)

**Method (from TCCApiHelper.cs:1722):** `CheckEligibility(string idNumber, int nationality, int idType, int subType, Seller seller, bool IsTest = false)`

**Endpoints (from appsettings.json:91-94):**

- **Production:** `POST https://185.23.125.108:8085/TCC-Web/api/eligibility`
- **Test:** `POST https://185.23.125.108:8083/TCC-Web/api/eligibility`

**Request Model (from TCCApiHelper.cs:1730-1750):**
```json
{
  "apiKey": "string",
  "mobileNumber": {
    "subscriptionType": 0
  },
  "person": {
    "personId": "AB1234567",
    "nationality": 156,
    "IdType": 3
  },
  "Operator": {
    "employeeId": "string",
    "employeeIdType": 1,
    "operatorTCN": "yyyyMMddHHmmssfff"
  }
}
```

**Response:**
```json
{
  "tcn": "e81e370a-c021-49ca-af46-62e4285fe3d7",
  "code": 600,
  "message": "success"
}
```

**For Visitor:**

- `personId`: Passport number (not National ID)
- `nationality`: Nationality code (e.g., 156 for Egypt)
- `IdType`: Visitor IdType (3, 4, 5, etc.)

### 6.5 TCC AddNumber (with Nafath Token)

**Method (from TCCApiHelper.cs:1001):** `AddNumber(UserDefinition user, Seller seller, SalesOrder ord, int TccRegion, int SourceType, List<ExtraSIM> extraSIMs = null, bool IsTest = false)`

**Endpoints (from appsettings.json:91-94):**

- **Production:** `POST https://185.23.125.108:8085/TCC-Web/api-tcc/individual/v2/verify`
- **Test:** `POST https://185.23.125.108:8083/TCC-Web/api-tcc/individual/v2/verify`

**Request Model (from TCCApiHelper.cs:1035-1073):**
```json
{
  "apiKey": "string",
  "token": "seller.TccToken",
  "requestType": 1,
  "person": {
    "personId": "AB1234567",
    "IdType": 3,
    "nationality": 156,
    "fingerIndex": null,
    "fingerImage": null,
    "exceptionFlag": 11,
    "otp": null,
    "iamToken": null,
    "iamAppToken": "nafath-app-token-here"
  },
  "mobileNumber": {
    "msisdn": "966512345678",
    "msisdnType": "V",
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

**Exception Flag Logic (from TCCApiHelper.cs:1032-1033):**
```csharp
int exceptionFlag = (ord.IdType == IdType.Diplomat) ? 1
    : (ord.UseIAMToken == 1 ? ((ord.TokenType == 0) ? 8 : 9) : 0);
exceptionFlag = (ord.IAMToken == null) ? exceptionFlag : 11;
```

**For Visitor with Nafath App:**

- `exceptionFlag`: 11 (when `ord.IAMToken` is not null)
- `iamAppToken`: The Nafath App token received from authentication
- `fingerIndex`: null
- `fingerImage`: null

---

## 7. Sequence Diagrams

### 7.1 Complete Visitor Activation Flow

```
┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐
│ Seller   │  │Onboarding│  │ Customer │  │   TCC    │  │   BSS    │  │  Nafath  │
│   App    │  │ Service  │  │  Mgmt    │  │ Service  │  │ Service  │  │ Service  │
└────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘
     │             │             │             │             │             │
     │ ValidateIdNew             │             │             │             │
     │ {passport,idType:3,       │             │             │             │
     │  nationality:156}         │             │             │             │
     │────────────>│             │             │             │             │
     │             │             │             │             │             │
     │             │ GetAlphaCode│             │             │             │
     │             │ (nationality:156 -> "EGY")│             │             │
     │             │─────────────│             │             │             │
     │             │             │             │             │             │
     │             │ GetCustomerInfo            │             │             │
     │             │ {passport, idType:3}       │             │             │
     │             │────────────────────────────────────────>│             │
     │             │             │             │             │             │
     │             │ CheckEligibility           │             │             │
     │             │ {passport, nationality, idType:3}       │             │
     │             │───────────────────────────>│             │             │
     │             │             │ {tcn, code:600}           │             │
     │             │<──────────────────────────────────────────           │
     │             │             │             │             │             │
     │             │ CreateOrder │             │             │             │
     │             │ {passport, idType:3, isVisitor:true}    │             │
     │             │────────────>│             │             │             │
     │             │  {orderId}  │             │             │             │
     │             │<────────────│             │             │             │
     │ {custInfo, order}         │             │             │             │
     │<────────────│             │             │             │             │
     │             │             │             │             │             │
     │ ... [GetMSISDNs, GetPlans, UpdateOrder steps] ...     │             │
     │             │             │             │             │             │
     │ NafathAppRequest          │             │             │             │
     │ {passport, nationality:"EGY"}           │             │             │
     │────────────>│             │             │             │             │
     │             │ CreateNafathAppRequest    │             │             │
     │             │ {id:passport, nationality:"EGY"}        │             │
     │             │─────────────────────────────────────────────────────>│
     │             │             │             │             │{random:45}  │
     │             │<─────────────────────────────────────────────────────│
     │ {random:45} │             │             │             │             │
     │<────────────│             │             │             │             │
     │             │             │             │             │             │
     │ [Seller shows "45" to customer, customer confirms in Nafath app]   │
     │             │             │             │             │             │
     │ CheckNafathStatus         │             │             │             │
     │────────────>│             │             │             │             │
     │             │ GetNafathCallbackStatus   │             │             │
     │             │─────────────────────────────────────────────────────>│
     │             │             │             │             │ {COMPLETED, │
     │             │             │             │             │  token}     │
     │             │<─────────────────────────────────────────────────────│
     │ {status:50, token}        │             │             │             │
     │<────────────│             │             │             │             │
     │             │             │             │             │             │
     │ UpdateOrder(authenticate) │             │             │             │
     │ {orderId, nafathToken}    │             │             │             │
     │────────────>│             │             │             │             │
     │             │             │             │             │             │
     │             │ TCC AddNumber              │             │             │
     │             │ {passport, idType:3, nafathToken}       │             │
     │             │───────────────────────────>│             │             │
     │             │             │ {tcn, code:600}           │             │
     │             │<──────────────────────────────────────────           │
     │             │             │             │             │             │
     │             │ BSS CreateSalesOrder       │             │             │
     │             │────────────────────────────────────────>│             │
     │             │             │ {orderId, success}        │             │
     │             │<────────────────────────────────────────│             │
     │             │             │             │             │             │
     │             │ Deduct Wallet, Generate eContract, Send SMS          │
     │             │             │             │             │             │
     │ {status: Completed}       │             │             │             │
     │<────────────│             │             │             │             │
     │             │             │             │             │             │
```

---

## 8. Integration & Gap Summary

### 8.1 ValidateIdNew Changes

| Frontend API | Backend Endpoint | Request Changes | Response Changes | Required Enhancements |
|--------------|------------------|-----------------|------------------|----------------------|
| `POST /api/Sales/ValidateIdNew` | Onboarding → `POST /api/mobile/v1/validateIdNew` | Add fields: `passport`, `nationalityAlphaCode`, `isVisitor` | Add: `isVisitor`, `nationalityName`, `passport` to custInfo | OrderMgmt: Store `passport`, `nationalityAlphaCode`, `isVisitor` fields |

### 8.2 NafathAppRequest Changes

| Frontend API | Backend Endpoint | Request Changes | Response Changes | Required Enhancements |
|--------------|------------------|-----------------|------------------|----------------------|
| `POST /api/Sales/NafathAppRequest` | TCC Service → Nafath | For visitors: use `passport` as `id`, add `nationality` AlphaCode | No change | TCC Service must resolve AlphaCode from nationality and include in Nafath request |

### 8.3 UpdateOrder (authenticate) Changes

| Frontend API | Backend Endpoint | Request Changes | Response Changes | Required Enhancements |
|--------------|------------------|-----------------|------------------|----------------------|
| `POST /api/Sales/UpdateOrder` (step=authenticate) | OrderMgmt → TCC → BSS | For visitors: Use `passport`, set `exceptionFlag=11` with Nafath token | No change | TCC AddNumber call must use passport + nafathToken for visitors |

### 8.4 New/Modified Endpoints Summary

| Endpoint | Service | Change Type | Description |
|----------|---------|-------------|-------------|
| `GET /api/nationalities` | Reference Data | EXISTING | Must return full list with AlphaCode |
| `GET /api/nationalities/{code}/alphacode` | Reference Data | **NEW** | Get AlphaCode for nationality |
| `POST /api/tcc/nafath/request` | TCC Service | MODIFY | Add `nationality` param for visitors |
| `POST /api/rbm/orders/validate-id-new` | OrderMgmt | MODIFY | Handle passport and isVisitor flag |

---

## 9. Microservice Entity Changes

### 9.1 Order Management Service (`ordermgmtservice`)

#### Entity: `CustomerProductOrder` - Additional Fields for Visitor

| Field | Type | New? | Description |
|-------|------|------|-------------|
| `passport` | `VARCHAR(50)` | **NEW** | Passport number for visitors |
| `nationality_alpha_code` | `VARCHAR(5)` | **NEW** | ISO Alpha code (e.g., "EGY") |
| `is_visitor` | `BOOLEAN` | **NEW** | Flag indicating visitor customer |

### 9.2 Customer Management Service (`digitalcustomermanagement`)

#### Entity: `Customer` / `PartyIdentification` - Additional Fields

| Field | Type | New? | Description |
|-------|------|------|-------------|
| `passport` | `VARCHAR(50)` | **NEW** | Passport number for visitors |
| `nationality_alpha_code` | `VARCHAR(5)` | **NEW** | ISO Alpha code |
| `is_visitor` | `BOOLEAN` | **NEW** | Visitor flag |

### 9.3 Reference Data Service

#### Entity: `Nationality` (or migrate from `TCCNationality`)

| Field | Type | New? | Description |
|-------|------|------|-------------|
| `id` | `SHORT` | EXISTING | Nationality code |
| `name_en` | `VARCHAR(100)` | EXISTING | English name |
| `name_ar` | `VARCHAR(100)` | EXISTING | Arabic name |
| `alpha_code` | `VARCHAR(5)` | **NEW** (if not present) | ISO Alpha code |

### 9.4 Onboarding Service

#### Entity: `OnboardingTracker` - Additional Fields

| Field | Type | New? | Description |
|-------|------|------|-------------|
| `isVisitor` | `BOOLEAN` | **NEW** | Track if flow is for visitor |
| `passport` | `VARCHAR(50)` | **NEW** | Passport for visitor tracking |

---

## 10. Error Handling

### 10.1 Visitor-Specific Error Codes

| Error Code | Message | Cause | Action |
|------------|---------|-------|--------|
| `VISITOR_NOT_ALLOWED` | "Visitor activation not allowed for this seller" | Seller not authorized for visitor sales | Redirect to authorized seller |
| `NATIONALITY_REQUIRED` | "Nationality is required for visitor" | Nationality not provided | Request nationality from user |
| `INVALID_PASSPORT` | "Invalid passport number format" | Passport validation failed | Correct passport format |
| `NATIONALITY_NOT_FOUND` | "Nationality code not found" | AlphaCode lookup failed | Verify nationality code |
| `NAFATH_VISITOR_FAILED` | "Nafath authentication failed for visitor" | Nafath rejected visitor request | Retry or use alternative auth |

### 10.2 TCC Error Handling

Same as standard flow, but with visitor-specific context in error messages.

### 10.3 Compensation/Rollback

If activation fails after TCC AddNumber succeeds:
```java
// Cancel TCC registration
tcc.CancelNumberGeneric(
    order.getMsisdn(),
    order.getPassport(),    // Use passport for visitor
    order.getIdType(),
    order.getNationality(),
    order.getId()
);
```

---

## Appendix A: Sample API Payloads

### A.1 ValidateIdNew Request (Visitor)
```json
{
  "passport": "AB1234567",
  "idType": 3,
  "nationality": 156,
  "isMNP": false,
  "isESim": false
}
```

### A.2 NafathAppRequest (Visitor)
```json
{
  "id": "AB1234567",
  "nationality": "EGY",
  "action": "SpRequest",
  "service": "AddNumber"
}
```

### A.3 UpdateOrder Authenticate (Visitor)
```json
{
  "orderId": 12345,
  "step": "authenticate",
  "nafathToken": "eyJhbGciOiJIUzI1NiIs...",
  "nafathTransactionId": "abc-123-def"
}
```

---

## Appendix B: TCCNationality Sample Data

| Id | NameEn | NameAr | AlphaCode |
|----|--------|--------|-----------|
| 156 | Egypt | مصر | EGY |
| 586 | Pakistan | باكستان | PAK |
| 356 | India | الهند | IND |
| 784 | United Arab Emirates | الإمارات | ARE |
| 682 | Saudi Arabia | السعودية | SAU |
| 400 | Jordan | الأردن | JOR |
| 818 | Egypt | مصر | EGY |

---

## 11. Source Code Reference (RedbullSalesPortalRestSharp)

This section contains actual code from the original project with detailed explanations for each part.

### 11.1 Visitor Detection Logic

**File:** `SalesAPIController.cs` - Lines 1015 and 3072

This code determines whether the customer is a Visitor or not:

```csharp
// From SalesAPIController.cs:1015 - in ValidateIdNew
bool IsVisitor = (seller.UserId == 2095)
    && ((int)req.IdType.Value != 1)    // Not Citizen
    && ((int)req.IdType.Value != 2);   // Not Resident
```

```csharp
// From SalesAPIController.cs:3072 - in NafathAppRequest
bool IsVisitor = (user.Id == "2095")
    && (param.OrderData?.idType != null)
    && ((int)param.OrderData?.idType != 1)   // Not Citizen
    && ((int)param.OrderData?.idType != 2);  // Not Resident
```

**Explanation:**
- `seller.UserId == 2095`: Special condition for a specific seller authorized for visitor activations
- `IdType != 1`: Not a Saudi Citizen
- `IdType != 2`: Not a Resident (Iqama holder)
- Any other IdType (3-11, 99) is considered a Visitor

---

### 11.2 IdType Enum Definition

**File:** `Models/Tables/SalesOrder.cs` - Lines 36-50

```csharp
public enum IdType
{
    Citizen = 1,           // Saudi Citizen - National ID
    Resident = 2,          // Resident - Iqama Number
    Visitor = 3,           // Visitor - Passport Number
    GCC_Passport = 4,      // GCC Passport
    GCC_National_Id = 5,   // GCC National ID
    Pilgrim_Passport = 6,  // Pilgrim Passport
    Pilgrim_Border = 7,    // Pilgrim Border Number
    Umrah_Passport = 8,    // Umrah Passport
    Visitor_Visa = 9,      // Visitor Visa
    Umrah_Visa = 10,       // Umrah Visa
    Haj_Visa = 11,         // Hajj Visa
    Diplomat = 99          // Diplomat
}
```

**Explanation:**

- Values 1 and 2 are for Saudi Citizens and Residents (use National ID or Iqama)
- Values 3-11 and 99 are for Visitors of various types (use Passport or Visa Number)

---

### 11.3 TCCNationality Model

**File:** `Models/Tables/SalesOrder.cs` - Lines 272-279

```csharp
[Table("TCCNationality")]
public class TCCNationality
{
    public short Id { get; set; }       // Nationality code (e.g., 156 for Egypt)
    public string NameEn { get; set; }  // English name
    public string NameAr { get; set; }  // Arabic name
    public string AlphaCode { get; set; } // ISO Alpha code (e.g., "EGY", "PAK")
}
```

**Explanation:**

- This table contains the list of nationalities approved by TCC
- The `AlphaCode` field is critical for visitors - it is sent with the Nafath request
- Example: Egyptian nationality = Id: 156, AlphaCode: "EGY"

---

### 11.4 NafathAppRequest DTO

**File:** `Models/TCC/TccVerifyRequest.cs` - Lines 11-17

```csharp
public class NafathAppRequest
{
    public string id { get; set; }           // National ID or Passport number
    public string nationality { get; set; }   // Nationality code (AlphaCode) - for visitors only
    public string action { get; set; } = "SpRequest";
    public string service { get; set; }       // Service type: "AddNumber", "MNP", "LoginWithBio"
}
```

**Explanation:**

- `id`: For Citizen/Resident = National ID/Iqama, For Visitor = Passport number
- `nationality`: **null for Citizen/Resident**, **AlphaCode for Visitor** (the key difference!)
- `action`: Always "SpRequest"
- `service`: The type of operation requested

---

### 11.5 NafathAppRequest Handler - Complete Code

**File:** `SalesAPIController.cs` - Lines 3056-3113

```csharp
[HttpPost, JsonRequest]
public IActionResult NafathAppRequest([FromServices] IUserRetrieveService userRetriever, dynamic param)
{
    // 1. Log incoming request
    Log.ForContext("Type", "NafathAppRequest")
       .ForContext("OrderId", param.Id?.Value)
       .Information($"Order: {JsonConvert.SerializeObject(param)}");

    using (var db = new SalesDbContext(_config))
    {
        try
        {
            // 2. Get current user information
            UserDefinition user = (UserDefinition)userRetriever.ByUsername(
                User.FindFirst(ClaimTypes.NameIdentifier)?.Value);

            // 3. Initialize variables
            string nationality = null;  // null by default for Citizen/Resident
            string idNumber = param.IdNumber?.Value;

            // 4. *** Check if Visitor ***
            bool IsVisitor = (user.Id == "2095")
                && (param.OrderData?.idType != null)
                && ((int)param.OrderData?.idType != 1)   // Not Citizen
                && ((int)param.OrderData?.idType != 2);  // Not Resident

            // 5. *** Special handling for Visitor ***
            if (IsVisitor)
            {
                // Fetch nationality code (AlphaCode) from database
                TCCNationality nat = db.TCCNationalities.Find(
                    (short)param.OrderData?.nationality?.Value);
                nationality = nat.AlphaCode;  // e.g., "EGY", "PAK", "IND"

                // Use passport number instead of ID number
                idNumber = param.OrderData?.passport;
            }

            // 6. Create Nafath request
            NafathAppRequest req = new NafathAppRequest
            {
                service = param.ServiceType?.Value,  // "AddNumber", "MNP", etc.
                nationality = nationality,  // null for Citizen, AlphaCode for Visitor
                id = idNumber               // ID for Citizen, Passport for Visitor
            };

            // 7. Send request to Nafath
            var respStr = tcc.CreateNafathAppRequest(req);
            dynamic resp = JsonConvert.DeserializeObject(respStr);

            // 8. Validate response
            if (resp.random?.Value != null)
            {
                // Success - return random number to display to customer
                Log.ForContext("Type", "NafathAppResponse")
                   .ForContext("OrderId", param.Id?.Value)
                   .Information($"Response: {JsonConvert.SerializeObject(resp)}");
                return Ok(resp);
            }
            else
            {
                // Failure - return error message
                Log.ForContext("Type", "NafathAppResponse")
                   .ForContext("OrderId", param.Id?.Value)
                   .Warning($"Response: {JsonConvert.SerializeObject(resp.message?.Value)}");
                return BadRequest(resp.message?.Value);
            }
        }
        catch (Exception e)
        {
            Log.ForContext("Type", "NafathAppRequest")
               .ForContext("OrderId", param.Id?.Value)
               .Error(e, "An error occurred while processing the request.");
            return BadRequest($"Error: {e.Message}");
        }
    }
}
```

**Detailed Explanation:**

1. **Line 3072**: Check if user is a Visitor
2. **Lines 3074-3078**: If Visitor:
   - Fetch `AlphaCode` from `TCCNationality` table
   - Use `passport` instead of `IdNumber`
3. **Lines 3080-3085**: Create request with `nationality` (null for Citizen, AlphaCode for Visitor)

---

### 11.6 CreateNafathAppRequest - TCC Helper

**File:** `Modules/Common/Helpers/TCCApiHelper.cs` - Lines 101-124

```csharp
public string CreateNafathAppRequest(NafathAppRequest req, bool IsTest = false)
{
    // 1. Setup connection options
    RestClientOptions options = new RestClientOptions
    {
        BaseUrl = new Uri((!IsTest)
            ? _config["ApiSettings:Nafath2Endpoint"]      // Production
            : _config["ApiSettings:Nafath2EndpointTest"]), // Test
        RemoteCertificateValidationCallback = (sender, certificate, chain, sslPolicyErrors) => true
    };

    // 2. Log request
    Log.ForContext("API", "CreateNafathAppRequest").Information(JsonConvert.SerializeObject(req));

    // 3. Create and send request
    RestClient client = new RestClient(options);
    RestRequest request = new RestRequest();
    request.Method = Method.Post;
    request.AddHeader("Accept", "application/json");
    request.AddHeader("content-type", "application/json");

    // 4. Add API Key
    request.AddHeader("Authorization", (!IsTest)
        ? $"apikey {_config["ApiSettings:Nafath2APIKey"]}"
        : $"apikey {_config["ApiSettings:Nafath2APIKeyTest"]}");

    // 5. Add request body (contains nationality for visitors)
    request.AddJsonBody(req);

    // 6. Execute request
    RestResponse resp = client.Execute(request);

    // 7. Log response
    Log.ForContext("API", "CreateNafathAppRequest").Information(JsonConvert.SerializeObject(resp));

    if (resp.StatusCode != System.Net.HttpStatusCode.OK)
        return "";

    return resp.Content;
}
```

**Explanation:**

- This method sends the request to Nafath API
- The `req` contains `nationality` only if the customer is a Visitor

---

### 11.7 TCC AddNumber - Exception Flag for Visitor

**File:** `Modules/Common/Helpers/TCCApiHelper.cs` - Lines 719-738

```csharp
// Calculate exceptionFlag based on authentication type
int exceptionFlag = (ord.IdType == IdType.Diplomat)
    ? 1                                    // Diplomat
    : (ord.UseIAMToken == 1
        ? ((ord.TokenType == 0) ? 8 : 9)   // IAM Token
        : 0);                              // Normal

// *** If using Nafath App Token ***
exceptionFlag = (ord.IAMToken == null) ? exceptionFlag : 11;

TccUpdateRequest addNumberRequest = new TccUpdateRequest
{
    apiKey = (!IsTest) ? _config["ApiSettings:SematiApiKey"] : _config["ApiSettings:SematiApiKeyTest"],
    requestType = 18,
    token = seller.TccToken,
    person = new TccPersonReq
    {
        personId = ord.IdNumber,  // For Visitor: Passport number
        IdType = (int)((ord.IdType == IdType.Diplomat) ? IdType.Resident : ord.IdType),
        nationality = ord.Nationality,

        // *** Biometric Info ***
        fingerIndex = (int?)(ord.UseIAMToken == 1 ? null : ord.FingerIndex),
        fingerImage = ord.UseIAMToken == 1 ? null : ord.FingerImage,
        exceptionFlag = exceptionFlag,  // 11 for Nafath App

        // OTP and IAM Token
        otp = (exceptionFlag == 8) ? (int?)int.Parse(ord.IamOTP) : null,
        iamToken = (exceptionFlag == 9) ? ord.IAMToken : null,
        iamAppToken = (exceptionFlag == 11) ? ord.IAMToken : null  // * Nafath App Token
    },
    // ... rest of request
};
```

**Exception Flags Explanation:**

| Flag | Meaning | Usage |
|------|---------|-------|
| 0 | Normal | Fingerprint |
| 1 | Diplomat | Diplomat exception |
| 8 | OTP | IAM OTP |
| 9 | IAM Token | IAM Token |
| **11** | **Nafath App** | **For Visitors with Absher** |

---

### 11.8 Authenticate Step - Visitor Detection

**File:** `SalesAPIController.cs` - Line 1345

```csharp
// In step "authenticate"
bool IsVisitor = (seller.UserId == 2095)
    && (ord.IdType != IdType.Resident)
    && (ord.IdType != IdType.Citizen);

// Then call TCC AddNumber
if ((ord.IsMNP == ActivationType.PrepaidPortIn) || (ord.IsMNP == ActivationType.PostpaidPortIn))
{
    // Port-In
    tccResponse = tcc.NumberMNP(user, seller, ord, TccRegion, channel.TCCSourceType, ordSIMs);
}
else
{
    // New SIM
    tccResponse = tcc.AddNumber(user, seller, ord, TccRegion, channel.TCCSourceType, ordSIMs);
}
```

---

### 11.9 Compensation - Cancel Number for Visitor

**File:** `SalesAPIController.cs` - Lines 1410-1411, 1427-1428

```csharp
// If BSS fails after TCC success, cancel TCC registration
if (crmResult.code != 0)
{
    // *** Cancel registration - For Visitor uses Passport number ***
    tcc.CancelNumberGeneric(
        ord.MSISDN,           // Mobile number
        ord.IdNumber,         // For Visitor: Passport number (updated from TCC response)
        (int)ord.IdType,      // ID Type
        ord.Nationality,      // Nationality
        ord.Id                // Order ID
    );
    return BadRequest($"CRM: {crmResult.code}: {crmResult.message}");
}
```

**Explanation:**

- If TCC AddNumber succeeds but BSS CreateSalesOrder fails
- Must cancel the TCC registration (Compensation pattern)
- For Visitor: Uses Passport number in `ord.IdNumber`

---

### 11.10 Summary of Code Differences

| Location | Citizen/Resident | Visitor |
|----------|------------------|---------|
| **IdNumber Source** | `req.IdNumber` | `param.OrderData?.passport` |
| **Nationality in Nafath** | `null` | `TCCNationality.AlphaCode` |
| **Exception Flag** | 0, 8, 9 | **11** (Nafath App) |
| **Biometric** | Fingerprint/IAM | **iamAppToken** |
| **Seller Check** | Any authorized | `UserId == 2095` |

---

## Document History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | Dec 2024 | Auto-generated from RE | Initial draft |
| 1.1 | Dec 2024 | Auto-generated from RE | Added Source Code Reference section with detailed code explanations |
