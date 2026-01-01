# Transfer Ownership - Low Level Design (LLD)
# Change of Line Ownership for Existing Subscribers

## Document Information


> **Note**: This document has been verified against the actual .NET legacy code in `RedbullSalesPortalRestSharp`.
>
> See verification evidence in this document.

---

## 1. Executive Summary

### 1.1 Overview
Transfer Ownership handles the scenario where an **existing subscriber** wants to transfer the ownership of their mobile line to another person. This involves:
- Dual OTP verification (to current owner + to new owner)
- Identity verification for new owner (Nafath/Absher)
- TCC compliance verification (RequestType=17)
- BSS ownership transfer operation
- Optional SIM change during transfer

### 1.2 Key Characteristics (✅ VERIFIED)
- **Trigger**: Existing subscriber requests line ownership transfer
- **Parties Involved**: Current Owner + New Owner (`FromIdNumber` + `ToIdNumber`)
- **Authentication**: Dual OTP + Identity verification (new owner via Nafath/IAMAppToken)
- **TCC RequestType**: 17 (`TCCApiHelper.cs:922`)
- **Cost**: SIM Replacement Cost from config (`SalesAPIController.cs:3659`)
- **Activation**: Synchronous (immediate upon success)
- **Wallet**: ⚠️ **NO wallet check/deduction in legacy code!** (possible bug or feature gap)

### 1.3 Legacy vs Revamped Comparison

| Aspect | Legacy (.NET SalesApp) | Revamped (Java Microservices) |
|--------|------------------------|-------------------------------|
| Table | `TransferOwnership` | `transfer_ownership_order` (NEW) |
| OTP Storage | Plain text (2 OTPs) | BCrypt hash (secure) |
| TCC Integration | Direct API call (RequestType=17) | Via `semati-service` |
| BSS Integration | Direct SOAP call | Via `bss-mediation-service` |
| Cost Config | `SIMReplacementCost` | Environment property |
| Service | Monolithic SalesApp | New `sales-ownership-transfer-service` OR extend `ordermgmtservice` |

---

## 2. Legacy .NET Code References (VERIFIED)

### 2.1 Entity Definition

**Source:** `SalesOrder.cs:314-348`

```csharp
[Table("TransferOwnership")]
public class TransferOwnership
{
    [Key]
    [DatabaseGenerated(DatabaseGeneratedOption.Identity)]
    public int Id { get; set; }
    public string MSISDN { get; set; }              // Line being transferred
    public string FromIdNumber { get; set; }        // Current owner's ID
    public string ToIdNumber { get; set; }          // New owner's ID
    public IdType IdType { get; set; }              // New owner's ID type
    public IdType OldIdType { get; set; }           // Current owner's ID type
    public int Nationality { get; set; }            // New owner's nationality
    public FingerIndex FingerIndex { get; set; }
    public string FingerImage { get; set; }
    public string IAMAppToken { get; set; }         // Nafath token for new owner
    public int SubType { get; set; }                // 0=Prepaid, 1=Postpaid, 2=Hybrid
    public decimal Cost { get; set; }               // Transfer fee (⚠️ SET but NO wallet check/deduction!)
    public decimal WalletBalanceBefore { get; set; } // ⚠️ EXISTS but NEVER SET in code!
    public decimal WalletBalanceAfter { get; set; }  // ⚠️ EXISTS but NEVER SET in code!
    public int Status { get; set; }                 // 0=Pending, 1=Success (NO -1 in code!)
    public int SellerId { get; set; }
    public string OTP { get; set; }                 // OTP to new owner (via SMS)
    public string OwnerOTP { get; set; }            // OTP to current owner (via Ballighny)
    public DateTime? OTPExpiry { get; set; }        // 5 minutes expiry
    public DateTime TransDate { get; set; }
    public string BSSRequest { get; set; }
    public string BSSResponse { get; set; }
    public string CRMCustomerId { get; set; }       // New owner's CRM ID (if exists)
    public string OldCRMCustomerId { get; set; }    // Current owner's CRM ID
    public string FirstName { get; set; }           // New owner's name (from TCC)
    public string LastName { get; set; }
    public string ICCID { get; set; }               // New SIM (optional)
    public string IMSI { get; set; }
    public string OrderLat { get; set; }
    public string OrderLng { get; set; }
}
```

### 2.1.1 ⚠️ Legacy Code Issues Found

During code verification, the following issues were discovered:

| Issue | Description | Impact |
|-------|-------------|--------|
| **Unused Wallet Fields** | `WalletBalanceBefore` and `WalletBalanceAfter` exist in entity but are **NEVER SET** | No wallet tracking for this operation |
| **No Wallet Check** | Unlike SIM Replacement, there's **NO balance check** before TransferOwnership | Seller can do transfer even with 0 balance |
| **No Wallet Deduction** | The `Cost` field is set but **wallet is never deducted** | Free transfers? Possible bug |
| **No Failed Status** | Status never set to -1, stays at 0 on failure | Different from ExtraSIM behavior |

**Recommendation for Java Implementation:** Consider adding proper wallet check and deduction to match SIM Replacement behavior.

### 2.2 API Endpoints

| API | Method | Source | Description |
|-----|--------|--------|-------------|
| `/api/Sales/ValidateTransferId` | POST | SalesAPIController.cs:568 | Validate new owner & create order |
| `/api/Sales/SendOrderOTP` | POST | SalesAPIController.cs:232 | Send OTP to current/new owner |
| `/api/Sales/TransferOwnership` | POST | SalesAPIController.cs:3917 | Execute the transfer |

### 2.3 Flow Details

#### Step 1: ValidateTransferId (SalesAPIController.cs:568-623)

```csharp
[HttpPost, JsonRequest]
public IActionResult ValidateTransferId([FromServices] IUserRetrieveService userRetriever, dynamic req)
{
    string IdNumber = req.IdNumber.Value;           // Current owner
    string NewIdNumber = req.NewIdNumber.Value;     // New owner
    string MSISDN = req.MSISDN.Value;

    // 1. Check outstanding bill
    var outstanding = bss.GetOutstandingAmount(MSISDN);
    if (outstanding > 0)
        return BadRequest($"Customer has an outstanding bill SAR {outstanding}");

    // 2. TCC Eligibility Check for new owner
    var tccRespone = tcc.CheckEligibility(NewIdNumber, nationality, idType, tccIdType, seller);
    if (tccResult.code != 600)
        return BadRequest($"TCC{tccResult.code}: {tccResult.message}");

    // 3. Get CRM Customer Info
    dynamic custoInfo = bss.GetCustomerInformation(NewIdNumber, idType);

    // 4. Create Order
    req.orderType = OrderType.TransferOwnership;
    return CreateOrder(userRetriever, req);  // Creates TransferOwnership record
}
```

#### Step 2: SendOrderOTP (SalesAPIController.cs:240-272)

```csharp
// For Transfer Ownership orders
if ((int)req.OrderType.Value == (int)OrderType.TransferOwnership)
{
    transferOwnership = db.TransferOwnerships.Find(int.Parse(OrderId));
    int OTPReceiver = (int)req.OTPReceiver.Value;  // 1=New Owner, 2=Current Owner

    Random rnd = new Random();
    if (OTPReceiver == 1)
        transferOwnership.OTP = rnd.Next(9999).ToString("0000");       // To new owner
    else
        transferOwnership.OwnerOTP = rnd.Next(9999).ToString("0000");  // To current owner

    transferOwnership.OTPExpiry = DateTime.Now.AddMinutes(5);

    // Send OTP
    if ((oldIdType != Citizen) && (oldIdType != Resident))
    {
        // Non-citizen/resident: send SMS to MSISDN only
        await sms.SendSMS(MSISDN, message);
    }
    else
    {
        if (OTPReceiver == 1)
            await sms.SendSMS(MSISDN, message);     // SMS to new owner
        else
            tcc.SendBallighny(FromIdNumber, message); // Ballighny to current owner
    }

    return Ok(transferOwnership);  // WARNING: Returns OTP in response!
}
```

#### Step 3: TransferOwnership Execution (SalesAPIController.cs:3917-3977)

```csharp
[HttpPost, JsonRequest]
public async Task<IActionResult> TransferOwnership([FromServices] IUserRetrieveService userRetriever, dynamic req)
{
    TransferOwnership transferOwnership = db.TransferOwnerships.Find((int)req.Id?.Value);
    transferOwnership.IAMAppToken = req.IAMAppToken?.Value;

    // 1. Location Validation
    var result = VaildateLocation(transferOwnership.OrderLat, transferOwnership.OrderLng, sellerId);

    // 2. TCC Transfer Ownership (RequestType=17)
    string tccResp = tcc.TransferOwnership(user, seller, transferOwnership, TccRegion, channel.TCCSourceType);
    dynamic tccResult = JsonConvert.DeserializeObject(tccResp);

    if (tccResult.code.Value == 600)
    {
        // Get name from TCC response
        transferOwnership.FirstName = tccResult.person.trFirst ?? tccResult.person.first;
        transferOwnership.LastName = tccResult.person.trFamily ?? tccResult.person.family;

        // 3. BSS Transfer Ownership
        var bssResp = bss.TransferOwnership(transferOwnership, seller);
        if (long.Parse(bssResp.code) != 0)
            return BadRequest($"CRM: {bssResp.msg}");

        // 4. Change Primary Offer for non-citizen/resident
        if ((IdType != Citizen) && (IdType != Resident))
            bss.ChangePrimaryOffer(MSISDN, offeringId);

        transferOwnership.Status = 1;  // Success
        return Ok(transferOwnership);
    }
    else
    {
        return BadRequest($"TCC: {tccResult.message?.Value}");
    }
}
```

### 2.4 TCC Integration (RequestType=17)

**Source:** `TCCApiHelper.cs:897-998`

```csharp
public string TransferOwnership(UserDefinition user, Seller seller, TransferOwnership ord, int TccRegion, int SourceType)
{
    TccUpdateRequest addNumberRequest = new TccUpdateRequest
    {
        apiKey = _config["ApiSettings:SematiApiKey"],
        token = seller.TccToken,
        requestType = 17,  // TRANSFER_OWNERSHIP
        person = new TccPersonReq
        {
            personId = ord.ToIdNumber,          // New owner's ID
            IdType = (int)ord.IdType,
            nationality = ord.Nationality,
            exceptionFlag = 11,                 // IAM App Token
            iamAppToken = ord.IAMAppToken       // Nafath token
        },
        mobileNumber = new TccMobileNumber
        {
            msisdn = $"966{ord.MSISDN}",
            msisdnType = ord.MSISDN.StartsWith("5") ? "V" : "D",
            simList = simlist.ToArray(),
            oldOwnerId = ord.FromIdNumber,      // Current owner's ID
            subscriptionType = ord.SubType,
            isDefault = false
        },
        Operator = new TccOperator { ... }
    };

    // POST to /TCC-Web/api-tcc/individual/v2/verify
    RestResponse resp = client.Execute(request);
    return resp.Content;
}
```

### 2.5 BSS Integration (TransferOwnershipReqMsg)

**Source:** `BssApiHelper.cs:1445-1532, 2795-2822`

```csharp
public dynamic TransferOwnership(TransferOwnership ord, Seller seller)
{
    // SOAP Request to /apiaccess/SubscriberServices/SubscriberServices
    // SOAPAction: TransferOwnership

    XElement envelope = new XElement(soapenv + "Envelope",
        new XElement(soapenv + "Body",
            new XElement(sub + "TransferOwnershipReqMsg",
                new XElement(com + "ReqHeader", ...),
                new XElement(sub + "AccessInfo",
                    new XElement(com + "ObjectIdType", 4),      // By MSISDN
                    new XElement(com + "ObjectId", ord.MSISDN)
                ),
                new XElement(sub + "TCC_FLAG", 1),              // TCC validated
                new XElement(sub + "CustId", ord.OldCRMCustomerId),
                new XElement(sub + "NewCustInfo",
                    // Either existing customer or new customer info
                    ord.CRMCustomerId != null
                        ? new XElement(com + "ExistCustId", ord.CRMCustomerId)
                        : new XElement(com + "Customer",
                            new XElement(com + "CustLevel", custLevel),
                            new XElement(com + "Name", ...),
                            new XElement(com + "Nationality", ord.Nationality),
                            new XElement(com + "Certificate", ...)
                        )
                ),
                new XElement(sub + "NewAcctInfo",
                    new XElement(com + "Account",
                        new XElement(com + "PaymentType", ord.SubType),
                        new XElement(com + "BillCycleType", billCycle),
                        ...
                    )
                ),
                new XElement(sub + "PasswordInfo", ...)
            )
        )
    );

    return new { code = retCode, msg = retMsg };
}
```

---

## 3. Complete Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│                         TRANSFER OWNERSHIP FLOW                                          │
├─────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                          │
│  ┌──────────────┐                                                                       │
│  │  Seller App  │                                                                       │
│  └──────┬───────┘                                                                       │
│         │                                                                                │
│         ▼                                                                                │
│  ┌──────────────────────────────────────────────────────────────────────────────────┐  │
│  │  STEP 1: ValidateTransferId                                                       │  │
│  │  ─────────────────────────────                                                    │  │
│  │  Input: CurrentOwnerID, NewOwnerID, MSISDN, Nationality, ICCID                   │  │
│  │                                                                                   │  │
│  │  1. Check Outstanding Bill (BSS) ────────────────► Must be zero                  │  │
│  │  2. TCC CheckEligibility (NewOwner) ─────────────► Code 600 = OK                 │  │
│  │  3. Get Customer Info (BSS) ─────────────────────► Get CRM IDs                   │  │
│  │  4. Create TransferOwnership Order ──────────────► Status = 0 (Pending)          │  │
│  │                                                                                   │  │
│  │  Output: TransferOwnership { Id, MSISDN, FromId, ToId, Cost, Status=0 }          │  │
│  └──────────────────────────────────────────────────────────────────────────────────┘  │
│         │                                                                                │
│         ▼                                                                                │
│  ┌──────────────────────────────────────────────────────────────────────────────────┐  │
│  │  STEP 2: SendOrderOTP (Dual OTP)                                                  │  │
│  │  ───────────────────────────────                                                  │  │
│  │                                                                                   │  │
│  │  OTPReceiver=1 (New Owner):                                                       │  │
│  │  ├─► Generate 4-digit OTP                                                        │  │
│  │  └─► Send SMS to MSISDN                                                          │  │
│  │                                                                                   │  │
│  │  OTPReceiver=2 (Current Owner):                                                   │  │
│  │  ├─► Generate 4-digit OwnerOTP                                                   │  │
│  │  └─► Send via TCC Ballighny (registered mobile)                                  │  │
│  │                                                                                   │  │
│  │  Expiry: 5 minutes                                                                │  │
│  └──────────────────────────────────────────────────────────────────────────────────┘  │
│         │                                                                                │
│         ▼                                                                                │
│  ┌──────────────────────────────────────────────────────────────────────────────────┐  │
│  │  STEP 3: Identity Verification (New Owner)                                        │  │
│  │  ─────────────────────────────────────────                                        │  │
│  │                                                                                   │  │
│  │  ├─► Initiate Nafath Authentication                                              │  │
│  │  ├─► User approves on Nafath App                                                 │  │
│  │  └─► Receive IAMAppToken                                                         │  │
│  │                                                                                   │  │
│  └──────────────────────────────────────────────────────────────────────────────────┘  │
│         │                                                                                │
│         ▼                                                                                │
│  ┌──────────────────────────────────────────────────────────────────────────────────┐  │
│  │  STEP 4: TransferOwnership Execution                                              │  │
│  │  ───────────────────────────────────                                              │  │
│  │  Input: OrderId, IAMAppToken, Both OTPs verified                                  │  │
│  │                                                                                   │  │
│  │  1. Validate Location                                                             │  │
│  │  2. TCC TransferOwnership (RequestType=17) ──────► Get Name from response        │  │
│  │  3. BSS TransferOwnership ───────────────────────► Update CRM                    │  │
│  │  4. Change Primary Offer (if non-citizen) ───────► Update plan                   │  │
│  │  5. Update Status = 1 (Success)                                                   │  │
│  │                                                                                   │  │
│  │  Output: TransferOwnership { Status=1 }                                           │  │
│  └──────────────────────────────────────────────────────────────────────────────────┘  │
│                                                                                          │
└─────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 4. Java Microservices Architecture

### 4.1 Option A: New Dedicated Service

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                  sales-ownership-transfer-service (NEW)                          │
│                           Port: 8097                                             │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  Tables (PostgreSQL - sales_ownership_transfer_db):                             │
│  └── transfer_ownership_order    (Main transfer order)                          │
│                                                                                  │
│  Controllers:                                                                    │
│  └── TransferOwnershipController   /api/v1/transfer-ownership/*                 │
│                                                                                  │
│  Services:                                                                       │
│  ├── TransferOwnershipService                                                   │
│  ├── OTPService              (Dual OTP handling - secure)                       │
│  └── IdentityVerificationService (Nafath integration)                           │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
          │              │              │              │
          ▼              ▼              ▼              ▼
  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐
  │  semati-    │ │    bss-     │ │    sms-     │ │  customer-  │
  │  service    │ │  mediation  │ │   service   │ │ management  │
  │  (TCC)      │ │  (Huawei)   │ │             │ │             │
  │  Port:7002  │ │  Port:8088  │ │  Port:8089  │ │  Port:8072  │
  └─────────────┘ └─────────────┘ └─────────────┘ └─────────────┘
```

### 4.2 Option B: Extend ordermgmtservice

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                         ordermgmtservice (EXTENDED)                              │
│                              Port: 8070                                          │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  NEW Tables:                                                                     │
│  └── transfer_ownership_order                                                   │
│                                                                                  │
│  NEW Controller:                                                                 │
│  └── TransferOwnershipController   /api/v1/transfer-ownership/*                 │
│                                                                                  │
│  NEW Strategy:                                                                   │
│  └── TransferOwnershipActivationStrategy                                        │
│      └── getSematiRequestType() → TRANSFER_OWNERSHIP (17)                       │
│                                                                                  │
│  UPDATED Enums:                                                                  │
│  └── SematiRequestType                                                          │
│      └── TRANSFER_OWNERSHIP(17, "TransferOwnership")  // NEW                    │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## 5. Required Java Components

### 5.1 SematiRequestType Enum Update

**Current:** `SematiRequestType.java` has 3 types (1, 2, 18)

**Required Update:**
```java
public enum SematiRequestType {
    QUERY_NEW_MOBILE(1, "QueryNewMobileNumber"),
    REQUEST_NEW_SIM(2, "RequestNewSIM"),
    QUERY_TRANSFER_OPERATOR(18, "QueryTransferOperator"),

    // NEW: Transfer Ownership
    TRANSFER_OWNERSHIP(17, "TransferOwnership");  // ← ADD THIS

    // ...
}
```

### 5.2 TransferOwnershipOrder Entity

```java
@Entity
@Table(name = "transfer_ownership_order")
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class TransferOwnershipOrder {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String msisdn;

    @Column(name = "from_id_number", nullable = false)
    private String fromIdNumber;

    @Column(name = "to_id_number", nullable = false)
    private String toIdNumber;

    @Enumerated(EnumType.STRING)
    @Column(name = "id_type")
    private IdType idType;           // New owner's ID type

    @Enumerated(EnumType.STRING)
    @Column(name = "old_id_type")
    private IdType oldIdType;        // Current owner's ID type

    private Integer nationality;

    @Column(name = "iam_app_token")
    private String iamAppToken;

    @Column(name = "sub_type")
    private Integer subType;         // 0=Prepaid, 1=Postpaid, 2=Hybrid

    private BigDecimal cost;

    @Enumerated(EnumType.STRING)
    private TransferStatus status;   // PENDING, SUCCESS, FAILED

    @Column(name = "seller_id")
    private Integer sellerId;

    // OTP fields - stored as BCrypt hash (SECURE)
    @Column(name = "otp_hash")
    private String otpHash;          // OTP to new owner

    @Column(name = "owner_otp_hash")
    private String ownerOtpHash;     // OTP to current owner

    @Column(name = "otp_expiry")
    private LocalDateTime otpExpiry;

    @Column(name = "otp_attempts")
    private Integer otpAttempts = 0;

    @Column(name = "owner_otp_attempts")
    private Integer ownerOtpAttempts = 0;

    @Column(name = "trans_date")
    private LocalDateTime transDate;

    // CRM Integration
    @Column(name = "crm_customer_id")
    private String crmCustomerId;    // New owner's CRM ID

    @Column(name = "old_crm_customer_id")
    private String oldCrmCustomerId; // Current owner's CRM ID

    // New SIM (optional)
    private String iccid;
    private String imsi;

    // Geolocation
    @Column(name = "order_lat")
    private String orderLat;

    @Column(name = "order_lng")
    private String orderLng;

    // Audit
    @Column(name = "first_name")
    private String firstName;

    @Column(name = "last_name")
    private String lastName;

    @Column(name = "bss_request")
    private String bssRequest;

    @Column(name = "bss_response")
    private String bssResponse;
}
```

### 5.3 Status Enum

```java
public enum TransferStatus {
    PENDING(0),   // Initial state - also stays here on failure (allows retry)
    SUCCESS(1);   // TCC & BSS both succeeded

    // NOTE: NO FAILED status in legacy code!
    // When operation fails, status stays PENDING and error is returned.

    private final int code;
}
```

### 5.4 API Endpoints

```java
@RestController
@RequestMapping("/api/v1/transfer-ownership")
@RequiredArgsConstructor
public class TransferOwnershipController {

    private final TransferOwnershipService service;

    /**
     * Step 1: Validate new owner and create transfer order
     */
    @PostMapping("/validate")
    public ResponseEntity<TransferOrderResponse> validateTransfer(
            @Valid @RequestBody ValidateTransferRequest request) {
        return ResponseEntity.ok(service.validateAndCreateOrder(request));
    }

    /**
     * Step 2: Send OTP to owner
     * @param receiver 1=New Owner (SMS), 2=Current Owner (Ballighny)
     */
    @PostMapping("/{orderId}/send-otp")
    public ResponseEntity<Void> sendOtp(
            @PathVariable Long orderId,
            @RequestParam("receiver") int receiver) {
        service.sendOtp(orderId, receiver);
        return ResponseEntity.ok().build();
    }

    /**
     * Step 3: Verify OTP
     */
    @PostMapping("/{orderId}/verify-otp")
    public ResponseEntity<OtpVerificationResponse> verifyOtp(
            @PathVariable Long orderId,
            @Valid @RequestBody VerifyOtpRequest request) {
        return ResponseEntity.ok(service.verifyOtp(orderId, request));
    }

    /**
     * Step 4: Execute transfer
     */
    @PostMapping("/{orderId}/execute")
    public ResponseEntity<TransferResult> executeTransfer(
            @PathVariable Long orderId,
            @Valid @RequestBody ExecuteTransferRequest request) {
        return ResponseEntity.ok(service.executeTransfer(orderId, request));
    }
}
```

---

## 6. Security Improvements (Java vs .NET)

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                     SECURITY IMPROVEMENTS IN JAVA VERSION                        │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  LEGACY (.NET) - INSECURE:              REVAMPED (Java) - SECURE:              │
│  ════════════════════════               ═══════════════════════════             │
│                                                                                  │
│  ❌ OTP returned in API response         ✅ OTP NEVER in response                │
│  ❌ OTP stored as plain text             ✅ OTP stored as BCrypt hash            │
│  ❌ No attempt limiting                  ✅ Max 3 attempts per OTP               │
│  ❌ 4-digit OTP only                     ✅ 6-digit OTP                          │
│  ❌ 5-minute expiry                      ✅ 5-minute expiry (configurable)       │
│  ❌ No rate limiting                     ✅ Rate limiting on OTP send            │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## 7. Integration Points

### 7.1 TCC (semati-service)

| Action | Request Type | Endpoint |
|--------|-------------|----------|
| Check Eligibility | N/A | `/api/v1/semati/check-eligibility` |
| Transfer Ownership | 17 | `/api/v1/semati/verify` |
| Send Ballighny | N/A | `/api/v1/semati/ballighny` |

### 7.2 BSS (bss-mediation-service)

| Action | SOAP Action | Endpoint |
|--------|------------|----------|
| Get Outstanding Amount | QueryBalance | `/api/v1/bss/balance` |
| Get Customer Info | QueryCustomerInfo | `/api/v1/bss/customer` |
| Transfer Ownership | TransferOwnership | `/api/v1/bss/transfer-ownership` |
| Change Primary Offer | ChangePrimaryOffer | `/api/v1/bss/change-offer` |

### 7.3 SMS Service

| Action | Method |
|--------|--------|
| Send OTP to MSISDN | `smsService.sendOtp(msisdn, otp)` |

---

## 8. Configuration

```yaml
# application.yml
transfer-ownership:
  cost: 25.00                    # SAR
  otp:
    length: 6
    expiry-minutes: 5
    max-attempts: 3
    resend-cooldown-seconds: 60
  validation:
    require-zero-balance: true
    require-dual-otp: true
    require-nafath: true
```

---

## 9. Status Code Mapping (✅ VERIFIED)

| Status | Meaning | Source |
|--------|---------|--------|
| 0 | Pending | `SalesAPIController.cs:3657` - Initial state, also stays 0 on failure |
| 1 | Success | `SalesAPIController.cs:3959` - TCC & BSS both succeeded |

> **⚠️ IMPORTANT**: Unlike ExtraSIM, there is **NO Status = -1** in TransferOwnership!
> When TCC or BSS fails, the order stays at Status=0 and returns BadRequest.
> This allows the seller to retry the operation.

---

