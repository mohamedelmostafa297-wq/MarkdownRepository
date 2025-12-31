# Extra SIM Feature - Low Level Design (LLD) Document


## 1. Executive Summary

The **Extra SIM** feature allows existing Red Bull Mobile subscribers to add additional Data SIMs linked to their primary mobile number. This document provides comprehensive technical documentation extracted through reverse engineering of the existing .NET codebase.

### 1.1 Key Findings
- **Two distinct flows exist**:
  1. **Flow A**: Extra SIM added during new line activation (Postpaid only)
  2. **Flow B**: Standalone Extra SIM order for existing subscribers
- **Activation is asynchronous** via Hangfire scheduled tasks
- **OTP verification is NOT implemented** in backend (security gap)

---

## 2. Database Schema

### 2.1 Core Entities

#### 2.1.1 ExtraSIM Table
```sql
CREATE TABLE [dbo].[ExtraSIM] (
    [Id]          INT IDENTITY(1,1) PRIMARY KEY,
    [OrderId]     INT NOT NULL,           -- Links to SalesOrder.Id OR ExtraSIMOrder.OrderId
    [MSISDN]      NVARCHAR(64),           -- Assigned Data MSISDN (e.g., 580XXXXXXX)
    [ICCID]       NVARCHAR(128),          -- SIM Card ICCID
    [IsESIM]      INT NOT NULL,           -- 0=Physical, 1=eSIM
    [Status]      INT NOT NULL,           -- Status code
    [AddedDate]   DATETIME NOT NULL,
    [TCCRequest]  NVARCHAR(MAX),          -- TCC API request payload
    [TCCResponse] NVARCHAR(MAX),          -- TCC API response
    [BSSRequest]  NVARCHAR(MAX),          -- BSS SOAP request
    [BSSResponse] NVARCHAR(MAX),          -- BSS SOAP response
    [OrderType]   INT NOT NULL,           -- 0=During Activation, 1=Standalone Order
    [OrderLat]    NVARCHAR(50),           -- Geolocation latitude
    [OrderLng]    NVARCHAR(50)            -- Geolocation longitude
)
```

**Status Values:**
| Value | Meaning |
|-------|---------|
| 0 | Pending Activation |
| 1 | Activated Successfully |
| -2 | Failed / Expired |

**OrderType Values:**
| Value | Meaning |
|-------|---------|
| 0 | Created during SalesOrder activation |
| 1 | Created via standalone ExtraSIMOrder |

#### 2.1.2 ExtraSIMOrder Table
```sql
CREATE TABLE [dbo].[ExtraSIMOrder] (
    [Id]                  INT IDENTITY(1,1) PRIMARY KEY,
    [OrderId]             NVARCHAR(50) NOT NULL,    -- Generated: MSISDN.Substring(5) + DateTime
    [PrimaryMSISDN]       NVARCHAR(64) NOT NULL,    -- Customer's main voice number
    [SellerId]            INT NOT NULL,
    [ExtraMSISDN]         NVARCHAR(64),             -- New Data MSISDN assigned
    [ICCID]               NVARCHAR(128),
    [IsESIM]              INT NOT NULL,
    [TransDate]           DATETIME NOT NULL,
    [IdNumber]            NVARCHAR(50),
    [IdType]              INT NOT NULL,
    [Nationality]         INT NOT NULL,
    [UseIAMToken]         INT NOT NULL,             -- 0=Fingerprint, 1=IAM OTP
    [IAMToken]            NVARCHAR(255),
    [FingerIndex]         INT NULL,
    [FingerImage]         NVARCHAR(MAX),
    [SubType]             INT NULL,                  -- 0=Prepaid, 1=Postpaid, 2=Hybrid
    [Price]               DECIMAL(18,2) NOT NULL,
    [OTP]                 NVARCHAR(10),
    [OTPExpiry]           DATETIME,
    [Status]              INT NOT NULL,
    [WalletBalanceBefore] DECIMAL(18,2),
    [WalletBalanceAfter]  DECIMAL(18,2),
    [OrderLat]            NVARCHAR(50),
    [OrderLng]            NVARCHAR(50)
)
```

**Status Values:**
| Value | Meaning |
|-------|---------|
| 0 | Created (Pending OTP) |
| 2 | Confirmed (Pending Activation) |
| 1 | Activated Successfully |
| -1 | TCC Failed |
| -2 | BSS Failed / Expired |
| -7 | Invalid Coordinates |

### 2.2 Related Entities

#### 2.2.1 SalesOrder (relevant fields)
```csharp
public class SalesOrder {
    public int Id { get; set; }
    public decimal? ExtraSIMCost { get; set; }  // Cost for extra SIMs
    // ... other fields
}
```

### 2.3 Entity Relationship Diagram

```
┌─────────────────────┐         ┌─────────────────────┐
│     SalesOrder      │         │   ExtraSIMOrder     │
│  (Main Activation)  │         │  (Standalone Flow)  │
├─────────────────────┤         ├─────────────────────┤
│ Id (PK)             │         │ Id (PK)             │
│ ExtraSIMCost        │         │ OrderId (Generated) │
│ ...                 │         │ PrimaryMSISDN       │
└─────────┬───────────┘         │ OTP, OTPExpiry      │
          │                     │ Status              │
          │ 1:N                 └─────────┬───────────┘
          │ (OrderType=0)                 │ 1:N
          │                               │ (OrderType=1)
          ▼                               ▼
┌─────────────────────────────────────────────────────┐
│                     ExtraSIM                         │
├─────────────────────────────────────────────────────┤
│ Id (PK)                                              │
│ OrderId (FK to SalesOrder.Id OR ExtraSIMOrder.OrderId)│
│ OrderType (0=Activation, 1=Standalone)               │
│ MSISDN (Data Number)                                 │
│ ICCID                                                │
│ IsESIM                                               │
│ Status                                               │
└─────────────────────────────────────────────────────┘
```

---

## 3. Business Rules & Constraints

### 3.1 Eligibility Rules
1. **Flow A (During Activation)**: Only available for **Postpaid** orders (`IsPostpaid == 1`)
2. **Port-In Restriction**: Extra SIMs NOT allowed with Port-In requests
   ```csharp
   if ((extSims > 0) && (ord.IsMNP == ActivationType.PostpaidPortIn))
       throw new Exception("Extra SIMs are not allowed with Port-In requests");
   ```
3. **Pending Request Check**: Customer cannot have pending Extra SIM requests
   ```csharp
   if (db.ExtraSIMOrders.Where(a => (a.PrimaryMSISDN == MSISDN) && (a.Status == 2)).Count() > 0)
       throw new Exception("Customer has pending Extra SIM request");
   ```

### 3.2 Pricing Rules
- **Extra SIM Cost**: 25 SAR per additional SIM (after first free one)
  ```csharp
  const decimal ExtraSIMCost = 25m;
  ```
- **First SIM**: Free (included in plan)
- **VAT Applied**: For Prepaid orders only
  ```csharp
  ord.ExtraSIMCost = (extSimCount > 0) ? (decimal)(extSimCount - 1) * ExtraSIMCost : 0;
  ord.ExtraSIMCost = (ord.IsPostpaid == 0) ? ord.ExtraSIMCost * (1 + VAT) : ord.ExtraSIMCost;
  ```

### 3.3 Wallet Balance Validation
```csharp
if (seller.WalletBalance < sum * (double)(1 + VAT))
    throw new Exception("Insufficient Wallet Balance");
```

### 3.4 MSISDN Uniqueness
Data MSISDNs must be unique across ExtraSIM table:
```csharp
do {
    MSISDN = bss.GetDataMSISDN(slr);
} while (db.ExtraSIMs.Where(a => a.MSISDN == MSISDN).Any());
```

### 3.5 Expiration Rules
- Orders older than 1 day are auto-cancelled
- MSISDN is released back to pool via `bss.OperateMSISDN(sim.MSISDN, seller, "1030")`

---

## 4. API Endpoints

### 4.1 Flow B: Standalone Extra SIM Order

#### 4.1.1 CreateExtraSIMs
**Endpoint**: `POST /SalesAPI/CreateExtraSIMs`

**Purpose**: Create new Extra SIM order for existing subscriber

**Request Payload**:
```json
{
    "SubInfo": {
        "MSISDN": "566123456",           // Primary voice number
        "IdNumber": "1234567890",
        "IdType": 1,                      // 1=Citizen, 2=Resident, etc.
        "Nationality": 682,               // Saudi Arabia
        "SubType": 1,                     // 0=Prepaid, 1=Postpaid, 2=Hybrid
        "Lat": "24.7136",
        "Lng": "46.6753"
    },
    "ExtraSIMs": "[{\"enabled\": true, \"eSIM\": 0, \"ICCID\": \"899661...\", \"price\": 25}]"
}
```

**Response**:
```json
{
    "Order": {
        "Id": 1234,
        "OrderId": "123456Mdf",
        "PrimaryMSISDN": "566123456",
        "ExtraMSISDN": "580123456",
        "OTP": "1234",
        "OTPExpiry": "2025-12-30T10:02:00",
        "Status": 0
    }
}
```

**Process Flow**:
```
CreateExtraSIMs
├── 1. Validate ICCID presence (for physical SIM)
├── 2. Calculate total cost
├── 3. Generate OTP: rnd.Next(9999).ToString("0000")
├── 4. Check for pending requests
├── 5. Validate seller wallet balance
├── 6. For each enabled ExtraSIM:
│   ├── 6.1. Get ICCID (physical or PickESim)
│   ├── 6.2. Get SIM details from BSS
│   ├── 6.3. Get Data MSISDN from BSS
│   ├── 6.4. Operate (reserve) MSISDN
│   ├── 6.5. Validate coordinates
│   └── 6.6. Create ExtraSIMOrder record (Status=0)
└── 7. Return order details
```

**Source**: [SalesAPIController.cs:2655-2746](../RedBullSalesPortal/RedBullSalesPortal.Web/Modules/SalesAPI/SalesAPIController.cs#L2655)

---

#### 4.1.2 ConfirmExtraSIMs
**Endpoint**: `POST /SalesAPI/ConfirmExtraSIMs`

**Purpose**: Confirm Extra SIM order after OTP verification (frontend only)

**Request Payload**:
```json
{
    "OrderId": "123456Mdf",
    "SubInfo": {
        "fingerImage": "base64...",
        "fingerIndex": 1,
        "useIamToken": 0,
        "IAMToken": null
    }
}
```

**Response**:
```json
{
    // Success: empty OK response
}
```

**Process Flow**:
```
ConfirmExtraSIMs
├── 1. Get seller from JWT token
├── 2. Find all ExtraSIMOrder records with OrderId and Status=0
├── 3. For each order:
│   ├── 3.1. Update Status to 2 (Pending Activation)
│   ├── 3.2. Set authentication info (fingerprint/IAM)
│   └── 3.3. Save changes
├── 4. Call TCC API: tcc.AddExtraSIM(orders, seller)
├── 5. If TCC code == 600 (success):
│   └── 5.1. Call AddExtraSIMRecords (async)
│       ├── Create ExtraSIM record with OrderType=1
│       ├── Deduct from seller wallet
│       └── Generate Zatca invoice (if applicable)
├── 6. If TCC fails:
│   ├── Set all orders Status=-1
│   └── Return BadRequest with TCC error
└── 7. Return OK
```

**Source**: [SalesAPIController.cs:2749-2809](../RedBullSalesPortal/RedBullSalesPortal.Web/Modules/SalesAPI/SalesAPIController.cs#L2749)

---

#### 4.1.3 SendOrderOTP (for Extra SIM)
**Endpoint**: `POST /SalesAPI/SendOrderOTP`

**Purpose**: Send/Resend OTP for Extra SIM order

**Request Payload**:
```json
{
    "OrderId": "123456Mdf",
    "OrderType": 1              // OrderType.ExtraSIM = 1
}
```

**Process Flow**:
```
SendOrderOTP (OrderType=1)
├── 1. Find all ExtraSIMOrders with matching OrderId
├── 2. Build MSISDN list string
├── 3. Calculate total cost with VAT
├── 4. Determine delivery method by SubType:
│   ├── SubType=0 (Prepaid): SMS via SMPP
│   └── SubType=1,2 (Postpaid/Hybrid): TCC Ballighny
└── 5. Send OTP message
```

**OTP Message Format**:
```
Config Key: AppSettings:OTPMessageExtraSIM
Parameters: {0}=OTP, {1}=MSISDNs, {2}=Cost
```

**Source**: [SalesAPIController.cs:303-329](../RedBullSalesPortal/RedBullSalesPortal.Web/Modules/SalesAPI/SalesAPIController.cs#L303)

---

### 4.2 Flow A: Extra SIM During Activation

#### 4.2.1 UpdateOrder (step="select-msisdn")
When `IsPostpaid == 1` and `ExtraSIM` parameter is provided:

**Request Payload**:
```json
{
    "OrderId": 12345,
    "step": "select-msisdn",
    "MSISDN": "566123456",
    "MSISDNCost": 100,
    "IsMNP": 3,                 // PostpaidNewSIM
    "ExtraSIM": "[{\"enabled\": true, \"eSIM\": 0}]"
}
```

**Process Flow**:
```
UpdateOrder (step="select-msisdn", IsPostpaid=1)
├── 1. Validate Port-In + ExtraSIM combination
├── 2. Check existing ExtraSIM records
├── 3. If count mismatch, remove all and recreate
├── 4. For each enabled ExtraSIM:
│   ├── 4.1. Get unique Data MSISDN from BSS
│   ├── 4.2. Operate (reserve) MSISDN
│   └── 4.3. Create ExtraSIM record (OrderType=0, Status=0)
├── 5. Calculate ExtraSIMCost
└── 6. Update order total
```

**Source**: [SalesAPIController.cs:1211-1261](../RedBullSalesPortal/RedBullSalesPortal.Web/Modules/SalesAPI/SalesAPIController.cs#L1211)

---

#### 4.2.2 UpdateOrder (step="authenticate")
ICCID assignment for Extra SIMs:

**Request Payload**:
```json
{
    "OrderId": 12345,
    "step": "authenticate",
    "ICCID": "899661...",
    "ExtraSIM": "[{\"ICCID\": \"899661...\"}]"
}
```

**Process Flow**:
```
UpdateOrder (step="authenticate")
├── ...other validation...
├── Get ExtraSIMs for this order
├── If ExtraSIMs exist and param.ExtraSIM provided:
│   └── For each ExtraSIM:
│       ├── Set ICCID (physical or PickESim for eSIM)
│       └── Save changes
└── Continue with TCC/BSS activation
```

**Source**: [SalesAPIController.cs:1306-1319](../RedBullSalesPortal/RedBullSalesPortal.Web/Modules/SalesAPI/SalesAPIController.cs#L1306)

---

### 4.3 Helper Endpoints

#### 4.3.1 IsExtraSIM
**Endpoint**: `POST /SalesAPI/IsExtraSIM`

**Purpose**: Check if a MSISDN is an Extra SIM

**Request**: `{ "MSISDN": "580123456" }`

**Logic**:
```csharp
// Check if customer's plan is Extra SIM plan
if ((plan.Id != ExtraSIMOfferId) && (plan.Id != PrepaidExtraSIMOfferId) && (plan.Id != ExtraSIMOfferIdExt))
    return false;

// Check if MSISDN exists in ExtraSIMView
ExtraSIMView sIMView = db.ExtraSIMViews.Where(a => a.ExtraSIM == MSISDN).FirstOrDefault();
return (sIMView != null);
```

**Source**: [SalesAPIController.cs:2636-2652](../RedBullSalesPortal/RedBullSalesPortal.Web/Modules/SalesAPI/SalesAPIController.cs#L2636)

---

#### 4.3.2 GetExtraSIMInfo
**Internal Method**

**Purpose**: Get Extra SIM configuration for a plan

**Returns**:
```json
{
    "PlanId": "ESM001",
    "FreeSIMs": 1,
    "PaidSIMs": 4,
    "ExtraSIMCost": 25
}
```

**Source**: [SalesAPIController.cs:2892-2898](../RedBullSalesPortal/RedBullSalesPortal.Web/Modules/SalesAPI/SalesAPIController.cs#L2892)

---

## 5. External System Integration

### 5.1 TCC (Telecom Compliance Center) Integration

#### 5.1.1 AddExtraSIM Method
**File**: [TCCApiHelper.cs:1121-1233](../RedBullSalesPortal/RedBullSalesPortal.Web/Modules/Common/Helpers/TCCApiHelper.cs#L1121)

**Endpoint**: `POST /TCC-Web/api-tcc/individual/v2/verify`

**Request Type**: 2 (Update/Add SIM)

**Request Payload Structure**:
```json
{
    "apiKey": "<from config>",
    "token": "<seller TCC token>",
    "requestType": 2,
    "person": {
        "personId": "1234567890",
        "IdType": 1,
        "nationality": 682,
        "fingerIndex": 1,           // null if IAM token used
        "fingerImage": "base64...", // null if IAM token used
        "exceptionFlag": 0,         // 0=fingerprint, 8=IAM OTP, 11=IAM App Token
        "otp": null,                // only if exceptionFlag=8
        "iamAppToken": null         // only if exceptionFlag=11
    },
    "mobileNumber": {
        "msisdn": "966566123456",
        "msisdnType": "V",          // V=Voice, D=Data
        "simList": [
            {
                "iccid": "899661...",
                "imsi": "42010...",
                "eSim": false
            }
        ],
        "subscriptionType": 1       // 0=Prepaid, 1=Postpaid (Hybrid treated as Postpaid)
    },
    "Operator": {
        "sourceType": 1,
        "sourceId": "<org number>",
        "operatorTCN": "20251230101530123",
        "region": "01",
        "employeeIdType": 1,
        "employeeUsername": "<seller username>",
        "employeeId": "<seller ID>",
        "deviceId": "<seller device>",
        "branchAddress": "24.7136,46.6753"
    }
}
```

**Response**:
```json
{
    "tcn": "uuid-transaction-id",
    "code": 600,
    "message": "success",
    "person": {
        "first": "محمد",
        "father": "أحمد",
        "family": "الأحمد",
        "gender": "M",
        "birthdate": "1990-01-01"
    }
}
```

**Success Code**: 600

---

### 5.2 BSS (Business Support System) Integration

#### 5.2.1 GetDataMSISDN
**Purpose**: Reserve a data MSISDN from the pool

**SOAP Action**: Custom implementation

**Source**: [BssApiHelper.cs](../RedBullSalesPortal/RedBullSalesPortal.Web/Modules/Common/Helpers/BssApiHelper.cs)

---

#### 5.2.2 OperateMSISDN
**Purpose**: Lock/Unlock MSISDN in BSS

**Parameters**:
- MSISDN: The number to operate on
- Seller: Current seller
- OperationType: Default lock, "1030" = release

---

#### 5.2.3 CreateExtraSIMOrder (BSS)
**File**: [BssApiHelper.cs:3262-3297](../RedBullSalesPortal/RedBullSalesPortal.Web/Modules/Common/Helpers/BssApiHelper.cs#L3262)

**SOAP Endpoint**: `/apiaccess/OrderServices/OrderServices`

**SOAP Action**: `CreateSaleOrder`

**Request Structure**:
```xml
<soapenv:Envelope>
    <soapenv:Body>
        <ord:CreateSaleOrderReqMsg>
            <com:ReqHeader>
                <com:Version>1</com:Version>
                <com:TransactionId>20251230101530...</com:TransactionId>
                <com:Channel>SAPP</com:Channel>
                <com:PartnerId>101</com:PartnerId>
                <com:AccessUser>...</com:AccessUser>
                <com:OperatorId>seller_username</com:OperatorId>
            </com:ReqHeader>
            <ord:Order>
                <ord:ExternalOrderId>SAPPEXT0000001234</ord:ExternalOrderId>
                <ord:ExistingCustId>CUST123456</ord:ExistingCustId>
                <ord:ServiceNumber>580123456</ord:ServiceNumber>
            </ord:Order>
            <ord:OrderItem>
                <ord:Subscriber>
                    <ord:Subscriber>
                        <ord:ServiceNum>580123456</ord:ServiceNum>
                        <ord:ICCID>899661...</ord:ICCID>
                        <ord:Language>2063</ord:Language>
                        <ord:Password>encrypted_password</ord:Password>
                        <ord:TCCFlag>1</ord:TCCFlag>
                        <ord:EsimFlag>N</ord:EsimFlag>
                    </ord:Subscriber>
                    <ord:Account>
                        <ord:ExistingAcctId>ACC123456</ord:ExistingAcctId>
                    </ord:Account>
                    <ord:PrimaryOffering>
                        <com:OfferingId>
                            <com:OfferingId>ESM001</com:OfferingId>
                        </com:OfferingId>
                    </ord:PrimaryOffering>
                </ord:Subscriber>
            </ord:OrderItem>
        </ord:CreateSaleOrderReqMsg>
    </soapenv:Body>
</soapenv:Envelope>
```

**Offering ID Selection**:
```csharp
// Postpaid/Hybrid uses ExtraSIMOfferId
// Prepaid uses PrepaidExtraSIMOfferId
((SubType == 1) || (SubType == 2)) ? _config["ApiSettings:ExtraSIMOfferId"] : _config["ApiSettings:PrepaidExtraSIMOfferId"]
```

---

#### 5.2.4 PickESim
**Purpose**: Get eSIM ICCID from BSS

**Used When**: `IsESIM == 1`

---

#### 5.2.5 GetSIMDetails
**Purpose**: Validate physical SIM and get IMSI

**IMSI Calculation for eSIM**:
```csharp
IMSI = "42010" + ICCID.Remove(ICCID.Length - 1).Substring(9)
```

---

## 6. Background Jobs (Hangfire)

### 6.1 ActivateExtraSIMRecords
**Schedule**: Every 10 minutes (7,17,27,37,47,57 * * * *)

**Purpose**: Activate Extra SIMs created via standalone flow (OrderType=1)

**File**: [ScheduledTasksHelper.cs:143-201](../RedBullSalesPortal/RedBullSalesPortal.Web/Modules/Common/Helpers/ScheduledTasksHelper.cs#L143)

**Process Flow**:
```
ActivateExtraSIMRecords
├── 1. Query: ExtraSIMs WHERE Status=0 AND OrderType=1
├── 2. For each pending ExtraSIM:
│   ├── 2.1. Find matching ExtraSIMOrder by ICCID
│   ├── 2.2. Validate OrderId match
│   ├── 2.3. Check expiration (> 1 day old = cancel)
│   ├── 2.4. Get AccountId from BSS
│   ├── 2.5. Get CustomerInfo from BSS
│   └── 2.6. Call BSS CreateExtraSIMOrder
│       ├── Success: Set Status=1 for both records
│       └── Failure: Set Status=-2, release MSISDN
└── 3. Log errors
```

---

### 6.2 ActivateExtraSIMs
**Schedule**: Every 10 minutes (2,12,22,32,42,52 * * * *)

**Purpose**: Activate Extra SIMs created during new activation (OrderType=0)

**File**: [ScheduledTasksHelper.cs:203-246](../RedBullSalesPortal/RedBullSalesPortal.Web/Modules/Common/Helpers/ScheduledTasksHelper.cs#L203)

**Process Flow**:
```
ActivateExtraSIMs
├── 1. Query: ExtraSIMs WHERE Status=0 AND OrderType=0
├── 2. For each pending ExtraSIM:
│   ├── 2.1. Find parent SalesOrder
│   ├── 2.2. Check expiration + no e-contract = cancel
│   ├── 2.3. Wait for e-contract (activation complete)
│   ├── 2.4. Get AccountId from BSS
│   ├── 2.5. Get CustomerInfo from BSS
│   └── 2.6. Call BSS CreateExtraSIMOrder
│       ├── Success: Set Status=1
│       └── Failure: Keep Status=0 (retry on next run)
└── 3. Silent error handling (no logging!)
```

**Important**: This job waits for the main SalesOrder to complete (e-contract generated) before activating Extra SIMs.

---

## 7. Complete Flow Diagrams

### 7.1 Flow A: Extra SIM During New Activation (Postpaid)

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                        FLOW A: Extra SIM During Activation                       │
└─────────────────────────────────────────────────────────────────────────────────┘

┌──────────┐     ┌──────────────┐     ┌──────────────────┐     ┌─────────────────┐
│  Mobile  │     │  Sales API   │     │    Database      │     │   External APIs │
│   App    │     │  Controller  │     │                  │     │   (BSS/TCC)     │
└────┬─────┘     └──────┬───────┘     └────────┬─────────┘     └────────┬────────┘
     │                  │                      │                        │
     │ CreateOrder      │                      │                        │
     │ (IsPostpaid=1)   │                      │                        │
     ├─────────────────►│                      │                        │
     │                  │ INSERT SalesOrder    │                        │
     │                  ├─────────────────────►│                        │
     │                  │                      │                        │
     │ UpdateOrder      │                      │                        │
     │ step=select-msisdn                      │                        │
     │ + ExtraSIM[]     │                      │                        │
     ├─────────────────►│                      │                        │
     │                  │                      │         GetDataMSISDN  │
     │                  │──────────────────────┼───────────────────────►│
     │                  │                      │         580XXXXXXX     │
     │                  │◄─────────────────────┼────────────────────────│
     │                  │                      │         OperateMSISDN  │
     │                  │──────────────────────┼───────────────────────►│
     │                  │                      │                        │
     │                  │ INSERT ExtraSIM      │                        │
     │                  │ (OrderType=0,        │                        │
     │                  │  Status=0)           │                        │
     │                  ├─────────────────────►│                        │
     │                  │                      │                        │
     │ UpdateOrder      │                      │                        │
     │ step=authenticate│                      │                        │
     │ + ExtraSIM ICCID │                      │                        │
     ├─────────────────►│                      │                        │
     │                  │                      │                        │
     │                  │ UPDATE ExtraSIM.ICCID│                        │
     │                  ├─────────────────────►│                        │
     │                  │                      │         TCC AddNumber  │
     │                  │──────────────────────┼───────────────────────►│
     │                  │                      │         (Main Line)    │
     │                  │◄─────────────────────┼────────────────────────│
     │                  │                      │         BSS CreateOrder│
     │                  │──────────────────────┼───────────────────────►│
     │                  │                      │         (Main Line)    │
     │                  │◄─────────────────────┼────────────────────────│
     │    Success       │                      │                        │
     │◄─────────────────│                      │                        │
     │                  │                      │                        │
     │                  │                      │                        │
     ▼                  ▼                      ▼                        ▼

    ┌─────────────────────────────────────────────────────────────────────────┐
    │                 BACKGROUND JOB: ActivateExtraSIMs                        │
    │                 (Runs every 10 minutes)                                  │
    └─────────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
    ┌───────────────────────────────────────────────────────────────────────────┐
    │ 1. Query ExtraSIM WHERE Status=0 AND OrderType=0                          │
    │ 2. Wait for SalesOrder.EContractFileName != null (activation complete)    │
    │ 3. Get AccountId and CustomerId from BSS                                  │
    │ 4. Call BSS CreateExtraSIMOrder                                          │
    │ 5. Update ExtraSIM.Status = 1                                            │
    └───────────────────────────────────────────────────────────────────────────┘
```

---

### 7.2 Flow B: Standalone Extra SIM Order

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                        FLOW B: Standalone Extra SIM Order                        │
└─────────────────────────────────────────────────────────────────────────────────┘

┌──────────┐     ┌──────────────┐     ┌──────────────────┐     ┌─────────────────┐
│  Mobile  │     │  Sales API   │     │    Database      │     │   External APIs │
│   App    │     │  Controller  │     │                  │     │   (BSS/TCC)     │
└────┬─────┘     └──────┬───────┘     └────────┬─────────┘     └────────┬────────┘
     │                  │                      │                        │
     │ CreateExtraSIMs  │                      │                        │
     │ (PrimaryMSISDN,  │                      │                        │
     │  ExtraSIMs[])    │                      │                        │
     ├─────────────────►│                      │                        │
     │                  │                      │         PickESim (opt) │
     │                  │──────────────────────┼───────────────────────►│
     │                  │                      │         GetSIMDetails  │
     │                  │──────────────────────┼───────────────────────►│
     │                  │                      │         GetDataMSISDN  │
     │                  │──────────────────────┼───────────────────────►│
     │                  │                      │         OperateMSISDN  │
     │                  │──────────────────────┼───────────────────────►│
     │                  │                      │                        │
     │                  │ Generate OTP         │                        │
     │                  │ (4 digits)           │                        │
     │                  │                      │                        │
     │                  │ INSERT ExtraSIMOrder │                        │
     │                  │ (Status=0, OTP)      │                        │
     │                  ├─────────────────────►│                        │
     │   {Order, OTP}   │                      │                        │
     │◄─────────────────│                      │                        │
     │                  │                      │                        │
     │ SendOrderOTP     │                      │                        │
     │ (OrderType=1)    │                      │                        │
     ├─────────────────►│                      │                        │
     │                  │                      │       SMS / Ballighny  │
     │                  │──────────────────────┼───────────────────────►│
     │     OK           │                      │                        │
     │◄─────────────────│                      │                        │
     │                  │                      │                        │
     │ [User enters OTP │                      │                        │
     │  on mobile app]  │                      │                        │
     │                  │                      │                        │
     │ ConfirmExtraSIMs │                      │                        │
     │ (OrderId,        │                      │                        │
     │  fingerprint/IAM)│                      │                        │
     ├─────────────────►│                      │                        │
     │                  │                      │   ⚠️ NO OTP VERIFY!   │
     │                  │ UPDATE ExtraSIMOrder │                        │
     │                  │ Status=2             │                        │
     │                  ├─────────────────────►│                        │
     │                  │                      │         TCC AddExtraSIM│
     │                  │──────────────────────┼───────────────────────►│
     │                  │                      │                        │
     │                  │◄─────────────────────┼────────────────────────│
     │                  │                      │                        │
     │                  │ [If TCC Success]     │                        │
     │                  │ INSERT ExtraSIM      │                        │
     │                  │ (OrderType=1,        │                        │
     │                  │  Status=0)           │                        │
     │                  ├─────────────────────►│                        │
     │                  │                      │                        │
     │                  │ Deduct Wallet        │                        │
     │                  ├─────────────────────►│                        │
     │                  │                      │                        │
     │     OK           │                      │                        │
     │◄─────────────────│                      │                        │
     │                  │                      │                        │
     ▼                  ▼                      ▼                        ▼

    ┌─────────────────────────────────────────────────────────────────────────┐
    │                 BACKGROUND JOB: ActivateExtraSIMRecords                  │
    │                 (Runs every 10 minutes)                                  │
    └─────────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
    ┌───────────────────────────────────────────────────────────────────────────┐
    │ 1. Query ExtraSIM WHERE Status=0 AND OrderType=1                          │
    │ 2. Find matching ExtraSIMOrder by ICCID                                  │
    │ 3. Validate OrderId matches                                              │
    │ 4. Check expiration (> 1 day = cancel + release MSISDN)                  │
    │ 5. Get AccountId from BSS: GetAccountId(PrimaryMSISDN)                   │
    │ 6. Get CustomerInfo from BSS                                             │
    │ 7. Call BSS CreateExtraSIMOrder                                          │
    │ 8. Update Status:                                                         │
    │    - Success: ExtraSIM.Status=1, ExtraSIMOrder.Status=1                  │
    │    - Failure: ExtraSIM.Status=-2, ExtraSIMOrder.Status=-2                │
    └───────────────────────────────────────────────────────────────────────────┘
```

---

## 8. State Machine Diagrams

### 8.1 ExtraSIMOrder States

```
                    ┌─────────────────────────────────────────┐
                    │                                         │
                    ▼                                         │
              ┌───────────┐                                   │
    Create    │  Status=0 │ Invalid                          │
   ─────────► │ (Created) │ Coordinates                      │
              └─────┬─────┘ ───────────► ┌──────────────┐    │
                    │                     │  Status=-7   │    │
                    │                     │ (Coord Fail) │    │
                    │                     └──────────────┘    │
                    │ Confirm                                 │
                    ▼                                         │
              ┌───────────┐                                   │
              │  Status=2 │ TCC                              │
              │ (Pending) │ Failure ───────► ┌──────────────┐ │
              └─────┬─────┘                  │  Status=-1   │ │
                    │                         │ (TCC Failed) │ │
                    │ TCC Success             └──────────────┘ │
                    │ + Background Job                         │
                    ▼                                         │
              ┌───────────┐                                   │
              │  Status=1 │                BSS               │
              │ (Success) │ ◄────────── Failure ─────────────┤
              └───────────┘               │                   │
                    ▲                     ▼                   │
                    │              ┌──────────────┐           │
                    │              │  Status=-2   │           │
                    │              │ (BSS Failed) │           │
                    │              └──────────────┘           │
                    │                     │                   │
                    │                     │ OrderId           │
                    │                     │ Mismatch          │
                    │                     │                   │
                    └─────────────────────┴───────────────────┘
```

### 8.2 ExtraSIM States

```
              ┌───────────┐
    Create    │  Status=0 │
   ─────────► │ (Pending) │
              └─────┬─────┘
                    │
        ┌───────────┼───────────┐
        │           │           │
        ▼           ▼           ▼
   Expiration   BSS Success  BSS Failure
   (>1 day)         │           │
        │           │           │
        ▼           ▼           ▼
  ┌───────────┐ ┌───────────┐ ┌───────────┐
  │ Status=-2 │ │ Status=1  │ │ Status=-2 │
  │ (Expired) │ │ (Active)  │ │ (Failed)  │
  └───────────┘ └───────────┘ └───────────┘
        │
        │ Release MSISDN
        ▼
  bss.OperateMSISDN(MSISDN, seller, "1030")
```

---

## 9. Configuration Parameters

### 9.1 AppSettings

| Key | Purpose | Example Value |
|-----|---------|---------------|
| `AppSettings:OTPMessageExtraSIM` | OTP SMS template for Extra SIM | "Your OTP is {0} for Extra SIM {1}. Total: {2} SAR" |
| `AppSettings:RBMOrganizationNo` | Red Bull Mobile TCC Org Number | "1234567890" |
| `AppSettings:SIMReplacementCost` | Cost for SIM operations | "25" |

### 9.2 ApiSettings

| Key | Purpose |
|-----|---------|
| `ApiSettings:ExtraSIMOfferId` | BSS Offer ID for Postpaid Extra SIM |
| `ApiSettings:PrepaidExtraSIMOfferId` | BSS Offer ID for Prepaid Extra SIM |
| `ApiSettings:ExtraSIMOfferIdExt` | Extended Extra SIM Offer ID |
| `ApiSettings:SematiURL` | TCC API Base URL (Production) |
| `ApiSettings:SematiURLTest` | TCC API Base URL (Test) |
| `ApiSettings:SematiApiKey` | TCC API Key (Production) |
| `ApiSettings:BSSChannel` | BSS Channel Code (SAPP) |

---

## 10. Security Considerations

### 10.1 🔴 CRITICAL SECURITY FLAW: OTP Returned in API Response

> ⛔ **LEGACY ANTI-PATTERN - DO NOT REPLICATE IN JAVA IMPLEMENTATION**

**Finding**: The legacy system returns OTP value directly in the API response, expecting the mobile app to perform client-side verification. This is a **severe security vulnerability**.

**Evidence** (CreateExtraSIMs response):
```csharp
// Line 2737 in SalesAPIController.cs
return Ok(new { Order = orders[0] });  // ⚠️ orders[0] contains OTP field!
```

**ExtraSIMOrder entity includes OTP**:
```csharp
public class ExtraSIMOrder {
    // ...
    public string OTP { get; set; }        // ⚠️ Exposed in response!
    public DateTime OTPExpiry { get; set; }
    // ...
}
```

**What happens**:
1. Backend generates OTP and stores it in database
2. Backend returns **entire order object including OTP** to mobile app
3. Backend sends OTP via SMS/Ballighny to customer
4. Mobile app compares user input with OTP from API response (client-side!)
5. Mobile app calls ConfirmExtraSIMs **without sending OTP** to backend
6. Backend confirms order **without any OTP verification**

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│  ⚠️ LEGACY INSECURE FLOW - DO NOT IMPLEMENT                                     │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│   Mobile App                          Backend                    Customer       │
│       │                                  │                          │           │
│       │  CreateExtraSIMs                 │                          │           │
│       ├─────────────────────────────────►│                          │           │
│       │                                  │ Generate OTP="1234"      │           │
│       │                                  │ Save to DB               │           │
│       │                                  │                          │           │
│       │  Response: {Order: {OTP:"1234"}} │                          │           │
│       │◄─────────────────────────────────┤   SMS: "Your OTP: 1234"  │           │
│       │                                  ├─────────────────────────►│           │
│       │                                  │                          │           │
│       │  [App stores OTP="1234" locally] │                          │           │
│       │                                  │                          │           │
│       │  User enters: "1234"             │                          │           │
│       │  App compares locally ✓          │                          │           │
│       │                                  │                          │           │
│       │  ConfirmExtraSIMs (NO OTP sent!) │                          │           │
│       ├─────────────────────────────────►│                          │           │
│       │                                  │ ⚠️ NO VERIFICATION!      │           │
│       │                                  │ Proceeds with TCC...     │           │
│       │                                  │                          │           │
└─────────────────────────────────────────────────────────────────────────────────┘
```

**Why This Is Dangerous**:
1. **OTP Exposure**: Anyone intercepting API response sees the OTP
2. **No Server Validation**: Backend trusts client blindly
3. **Bypass Possible**: Attacker can skip OTP entirely by calling ConfirmExtraSIMs directly
4. **Man-in-the-Middle**: OTP visible in network traffic even with HTTPS (via proxy)

---

### 10.2 ✅ REQUIRED FIX FOR JAVA IMPLEMENTATION

> **MANDATORY**: The Java implementation MUST implement proper server-side OTP verification.

#### 10.2.1 Secure Flow Design

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│  ✅ SECURE FLOW - IMPLEMENT THIS IN JAVA                                        │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│   Mobile App                          Backend                    Customer       │
│       │                                  │                          │           │
│       │  POST /extra-sim/create          │                          │           │
│       ├─────────────────────────────────►│                          │           │
│       │                                  │ Generate OTP="1234"      │           │
│       │                                  │ Hash & Save to DB        │           │
│       │                                  │                          │           │
│       │  Response: {orderId: "ABC123"}   │   SMS: "Your OTP: 1234"  │           │
│       │  ⚠️ NO OTP IN RESPONSE!          ├─────────────────────────►│           │
│       │◄─────────────────────────────────┤                          │           │
│       │                                  │                          │           │
│       │  User enters: "1234"             │                          │           │
│       │                                  │                          │           │
│       │  POST /extra-sim/verify-otp      │                          │           │
│       │  {orderId: "ABC123", otp: "1234"}│                          │           │
│       ├─────────────────────────────────►│                          │           │
│       │                                  │ ✅ Verify OTP server-side│           │
│       │                                  │ ✅ Check expiry          │           │
│       │                                  │ ✅ Increment attempts    │           │
│       │                                  │                          │           │
│       │  Response: {verified: true}      │                          │           │
│       │◄─────────────────────────────────┤                          │           │
│       │                                  │                          │           │
│       │  POST /extra-sim/confirm         │                          │           │
│       │  {orderId: "ABC123", auth: ...}  │                          │           │
│       ├─────────────────────────────────►│                          │           │
│       │                                  │ ✅ Check OTP was verified│           │
│       │                                  │ Proceed with TCC...      │           │
│       │                                  │                          │           │
└─────────────────────────────────────────────────────────────────────────────────┘
```

#### 10.2.2 Java Implementation Requirements

```java
@Entity
@Table(name = "extra_sim_order")
public class ExtraSIMOrder {
    // ... other fields ...

    @Column(name = "otp_hash")
    private String otpHash;              // ✅ Store HASH, not plain text

    @Column(name = "otp_expiry")
    private LocalDateTime otpExpiry;

    @Column(name = "otp_attempts")
    private Integer otpAttempts = 0;     // ✅ Track failed attempts

    @Column(name = "otp_verified")
    private Boolean otpVerified = false; // ✅ Flag for verification status

    @Column(name = "otp_verified_at")
    private LocalDateTime otpVerifiedAt;

    // ⚠️ NO plain OTP field!
}
```

#### 10.2.3 OTP Service Implementation

```java
@Service
public class OTPService {

    private static final int OTP_LENGTH = 6;           // Use 6 digits, not 4
    private static final int OTP_EXPIRY_MINUTES = 5;
    private static final int MAX_ATTEMPTS = 3;

    @Autowired
    private PasswordEncoder passwordEncoder;  // BCrypt

    /**
     * Generate OTP - returns plain OTP for sending via SMS
     * Stores HASH in database
     */
    public String generateOTP(ExtraSIMOrder order) {
        String otp = String.format("%06d", new SecureRandom().nextInt(999999));

        order.setOtpHash(passwordEncoder.encode(otp));
        order.setOtpExpiry(LocalDateTime.now().plusMinutes(OTP_EXPIRY_MINUTES));
        order.setOtpAttempts(0);
        order.setOtpVerified(false);

        return otp;  // Return plain for SMS, but DON'T return to client!
    }

    /**
     * Verify OTP - called by /verify-otp endpoint
     */
    public OTPVerificationResult verifyOTP(ExtraSIMOrder order, String inputOtp) {
        // Check if already verified
        if (order.getOtpVerified()) {
            return OTPVerificationResult.ALREADY_VERIFIED;
        }

        // Check expiry
        if (LocalDateTime.now().isAfter(order.getOtpExpiry())) {
            return OTPVerificationResult.EXPIRED;
        }

        // Check max attempts
        if (order.getOtpAttempts() >= MAX_ATTEMPTS) {
            return OTPVerificationResult.MAX_ATTEMPTS_EXCEEDED;
        }

        // Increment attempts
        order.setOtpAttempts(order.getOtpAttempts() + 1);

        // Verify hash
        if (passwordEncoder.matches(inputOtp, order.getOtpHash())) {
            order.setOtpVerified(true);
            order.setOtpVerifiedAt(LocalDateTime.now());
            return OTPVerificationResult.SUCCESS;
        }

        return OTPVerificationResult.INVALID;
    }

    /**
     * Check if OTP was verified - called before confirm
     */
    public boolean isOTPVerified(ExtraSIMOrder order) {
        if (!order.getOtpVerified()) {
            return false;
        }

        // Optional: Check verification wasn't too long ago (e.g., 10 minutes)
        if (order.getOtpVerifiedAt().plusMinutes(10).isBefore(LocalDateTime.now())) {
            return false;  // Verification expired, need to re-verify
        }

        return true;
    }
}

public enum OTPVerificationResult {
    SUCCESS,
    INVALID,
    EXPIRED,
    MAX_ATTEMPTS_EXCEEDED,
    ALREADY_VERIFIED
}
```

#### 10.2.4 Controller Endpoints

```java
@RestController
@RequestMapping("/api/v1/extra-sim")
public class ExtraSIMController {

    @PostMapping("/create")
    public ResponseEntity<CreateExtraSIMResponse> createExtraSIMs(
            @RequestBody CreateExtraSIMRequest request) {

        ExtraSIMOrder order = extraSIMService.createOrder(request);

        // Generate OTP and send via SMS (but DON'T return it!)
        String otp = otpService.generateOTP(order);
        smsService.sendOTP(order.getPrimaryMsisdn(), otp);

        // ✅ Response does NOT include OTP
        return ResponseEntity.ok(CreateExtraSIMResponse.builder()
                .orderId(order.getOrderId())
                .extraMsisdn(order.getExtraMsisdn())
                .price(order.getPrice())
                .otpExpirySeconds(300)  // Tell client when OTP expires
                .build());
    }

    @PostMapping("/verify-otp")
    public ResponseEntity<VerifyOTPResponse> verifyOTP(
            @RequestBody VerifyOTPRequest request) {

        ExtraSIMOrder order = extraSIMService.findByOrderId(request.getOrderId());

        OTPVerificationResult result = otpService.verifyOTP(order, request.getOtp());

        return switch (result) {
            case SUCCESS -> ResponseEntity.ok(new VerifyOTPResponse(true, "OTP verified"));
            case INVALID -> ResponseEntity.badRequest()
                    .body(new VerifyOTPResponse(false, "Invalid OTP"));
            case EXPIRED -> ResponseEntity.badRequest()
                    .body(new VerifyOTPResponse(false, "OTP expired"));
            case MAX_ATTEMPTS_EXCEEDED -> ResponseEntity.status(429)
                    .body(new VerifyOTPResponse(false, "Too many attempts"));
            case ALREADY_VERIFIED -> ResponseEntity.ok(
                    new VerifyOTPResponse(true, "Already verified"));
        };
    }

    @PostMapping("/confirm")
    public ResponseEntity<?> confirmExtraSIMs(
            @RequestBody ConfirmExtraSIMRequest request) {

        ExtraSIMOrder order = extraSIMService.findByOrderId(request.getOrderId());

        // ✅ CRITICAL: Verify OTP was verified before proceeding
        if (!otpService.isOTPVerified(order)) {
            return ResponseEntity.status(403)
                    .body(new ErrorResponse("OTP not verified"));
        }

        // Now proceed with TCC and activation
        extraSIMService.confirmOrder(order, request);

        return ResponseEntity.ok().build();
    }
}
```

#### 10.2.5 Security Checklist for Java Implementation

| # | Requirement | Priority |
|---|-------------|----------|
| 1 | **NEVER return OTP in API response** | 🔴 Critical |
| 2 | Store OTP as hash (BCrypt), not plain text | 🔴 Critical |
| 3 | Implement server-side OTP verification endpoint | 🔴 Critical |
| 4 | Check OTP verified flag before confirm | 🔴 Critical |
| 5 | Use 6-digit OTP (not 4-digit) | 🟡 High |
| 6 | Implement attempt limiting (max 3) | 🟡 High |
| 7 | OTP expiry: 5 minutes max | 🟡 High |
| 8 | Rate limit OTP generation (1 per minute) | 🟡 High |
| 9 | Log all OTP verification attempts | 🟢 Medium |
| 10 | Verification expiry (10 min after verify) | 🟢 Medium |

### 10.2 Coordinate Validation
- Coordinates are validated for length (>15 chars = invalid)
- Location-based fraud detection exists

### 10.3 Wallet Balance Protection
- Balance is checked before order creation
- Balance is deducted atomically with order success

---

## 11. Error Handling

### 11.1 Exception Patterns

| Exception | Scenario | HTTP Response |
|-----------|----------|---------------|
| "Please enter correct ICCIDs" | Physical SIM without ICCID | 400 Bad Request |
| "Customer has pending Extra SIM request" | Duplicate pending order | 400 Bad Request |
| "Insufficient Wallet Balance" | Low seller balance | 400 Bad Request |
| "T565 : Coordinates is Invalid" | Suspicious coordinates | 400 Bad Request |
| "TCC: {code}: {message}" | TCC API failure | 400 Bad Request |
| "Unable to Assign Data Number" | BSS MSISDN pool empty | 400 Bad Request |
| "Extra SIMs are not allowed with Port-In" | Business rule violation | 400 Bad Request |

### 11.2 Logging

```csharp
// API Request Logging
Log.ForContext("API", "CreateExtraSIM").Information($"{JsonConvert.SerializeObject(param)}");

// Error Logging
Log.ForContext("API", "CreateExtraSIM").Error($"Req: {JsonConvert.SerializeObject(param)}, Exception: {e.Message}");

// TCC Trace Logging
db.TccTraces.Add(new TccTraces {
    EventType = "AddSIM",
    ClientTCN = operatorTCN,
    OrderId = orderId,
    TCCRequest = request,
    TCCResponse = response
});
```

---

## 12. Java Implementation Considerations

### 12.1 Technology Mapping

| .NET Technology | Java Equivalent |
|-----------------|-----------------|
| ASP.NET Core MVC | Spring Boot |
| Entity Framework Core | Spring Data JPA / Hibernate |
| Hangfire | Quartz Scheduler / Spring Scheduling |
| RestSharp | RestTemplate / WebClient / OkHttp |
| Serilog | SLF4J + Logback |
| Newtonsoft.Json | Jackson |

### 12.2 Entity Classes (Java)

```java
@Entity
@Table(name = "ExtraSIM")
public class ExtraSIM {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Integer id;

    @Column(name = "OrderId")
    private Integer orderId;

    @Column(name = "MSISDN", length = 64)
    private String msisdn;

    @Column(name = "ICCID", length = 128)
    private String iccid;

    @Column(name = "IsESIM")
    private Integer isEsim;

    @Column(name = "Status")
    private Integer status;

    @Column(name = "AddedDate")
    private LocalDateTime addedDate;

    @Column(name = "TCCRequest", columnDefinition = "NVARCHAR(MAX)")
    private String tccRequest;

    @Column(name = "TCCResponse", columnDefinition = "NVARCHAR(MAX)")
    private String tccResponse;

    @Column(name = "BSSRequest", columnDefinition = "NVARCHAR(MAX)")
    private String bssRequest;

    @Column(name = "BSSResponse", columnDefinition = "NVARCHAR(MAX)")
    private String bssResponse;

    @Column(name = "OrderType")
    private Integer orderType;

    @Column(name = "OrderLat", length = 50)
    private String orderLat;

    @Column(name = "OrderLng", length = 50)
    private String orderLng;
}
```

### 12.3 Service Layer Structure

```java
@Service
public class ExtraSIMService {

    @Autowired
    private ExtraSIMRepository extraSIMRepository;

    @Autowired
    private ExtraSIMOrderRepository extraSIMOrderRepository;

    @Autowired
    private TCCApiClient tccApiClient;

    @Autowired
    private BSSApiClient bssApiClient;

    public ExtraSIMOrderResponse createExtraSIMs(CreateExtraSIMRequest request) {
        // Implementation
    }

    public void confirmExtraSIMs(ConfirmExtraSIMRequest request) {
        // Implementation
    }
}
```

### 12.4 Scheduled Jobs

```java
@Component
public class ExtraSIMScheduledTasks {

    @Scheduled(cron = "0 7,17,27,37,47,57 * * * *")
    public void activateExtraSIMRecords() {
        // OrderType = 1 (Standalone)
    }

    @Scheduled(cron = "0 2,12,22,32,42,52 * * * *")
    public void activateExtraSIMs() {
        // OrderType = 0 (During Activation)
    }
}
```

---

## 13. Appendix

### 13.1 Enum Definitions

```csharp
public enum OrderType {
    Activation = 0,
    ExtraSIM = 1,
    ChangeSubscriptionType = 2,
    LineTermination = 3,
    TransferOwnership = 4
}

public enum SubscriptionType {
    Prepaid = 0,
    Postpaid = 1,
    Hybrid = 2
}
```

### 13.2 IMSI Calculation

```csharp
// For physical SIM
IMSI = "42010" + ICCID.Substring(9);

// For eSIM
IMSI = "42010" + ICCID.Remove(ICCID.Length - 1).Substring(9);
```

### 13.3 OrderId Generation

```csharp
// For ExtraSIMOrder
string OrderId = MSISDN.Substring(5) + DateTime.Now.ToString("Mdf");
// Example: "12345630Dec" for MSISDN 56612345 on Dec 30
```

---


