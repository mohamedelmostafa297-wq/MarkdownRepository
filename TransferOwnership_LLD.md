# [LLD] Transfer Ownership

**Document Version:** 2.0 (Verified from Source Code)
**Last Updated:** December 2025
**Status:** Implemented

---

## 1. Purpose

This document details the **Low-Level Design (LLD)** for the **Transfer Ownership** flow, verified directly from the source code without assumptions.

---

## 2. Endpoint

| Property | Value | Code Reference |
|----------|-------|----------------|
| **Method** | `POST` | `SalesAPIController.cs:3921` |
| **URL** | `/api/Sales/TransferOwnership` | `SalesAPIController.cs:3921` |
| **Controller** | `SalesAPIController` | `Modules/SalesAPI/SalesAPIController.cs` |

```csharp
// SalesAPIController.cs:3921
[HttpPost, JsonRequest]
public async Task<IActionResult> TransferOwnership([FromServices] IUserRetrieveService userRetriever, dynamic req)
```

---

## 3. Input Parameters

| Parameter | Type | Required | Code Reference |
|-----------|------|----------|----------------|
| `Id` | int | Yes | `SalesAPIController.cs:3929` |
| `IAMAppToken` | string | No | `SalesAPIController.cs:3930` |

```csharp
// SalesAPIController.cs:3929-3930
TransferOwnership transferOwnership = db.TransferOwnerships.Find((int)req.Id?.Value);
transferOwnership.IAMAppToken = req.IAMAppToken?.Value;
```

---

## 4. Flow Steps (From Code)

### Step 1: Get Current User from JWT

**Code Reference:** `SalesAPIController.cs:3925`

```csharp
UserDefinition user = (UserDefinition)userRetriever.ByUsername(
    User.FindFirst(ClaimTypes.NameIdentifier)?.Value
);
```

---

### Step 2: Load Data from Database

**Code Reference:** `SalesAPIController.cs:3927-3937`

| Data | Source | Code Line |
|------|--------|-----------|
| `TransferOwnership` | `db.TransferOwnerships.Find((int)req.Id?.Value)` | `:3929` |
| `Seller` | `db.Sellers.Find(user.UserId)` | `:3932` |
| `SalesPartner` | `db.SalesPartners.Find((int)seller.PartnerId)` | `:3933` |
| `SalesChannel` | `db.SalesChannels.Find(partner.SalesChannel)` | `:3934` |
| `City` | `db.Cities.Find(seller.City)` | `:3936` |
| `Province` | `db.Provinces.Find(city.Province)` | `:3937` |
| `TccRegion` | `province.TCCCode` | `:3938` |

```csharp
// SalesAPIController.cs:3932-3938
Seller seller = db.Sellers.Find(user.UserId);
SalesPartner partner = db.SalesPartners.Find((int)seller.PartnerId);
SalesChannel channel = db.SalesChannels.Find(partner.SalesChannel);

City city = db.Cities.Find(seller.City);
Province province = db.Provinces.Find(city.Province);
int TccRegion = province.TCCCode;
```

---

### Step 3: Validate Location

**Code Reference:** `SalesAPIController.cs:3940-3942` + `SalesAPIController.cs:4004-4022`

```csharp
// SalesAPIController.cs:3940-3942
var result = VaildateLocation(transferOwnership.OrderLat, transferOwnership.OrderLng, transferOwnership.SellerId);
if (result != null)
    return result;
```

**Validation Rules (from `VaildateLocation` method):**

| Rule | Error Code | Error Message | Code Line |
|------|------------|---------------|-----------|
| `orderLat == null \|\| orderLng == null` | `T555` | Invalid Order | `:4006-4009` |
| `orderLat.Length > 15 && orderLng.Length > 15` | `T565` | Coordinates is Invalid | `:4011-4014` |
| `orderLat == "14.624054" && orderLng == "56.573993"` | `T566` | Coordinates is Can't Detected | `:4016-4019` |

```csharp
// SalesAPIController.cs:4004-4022
private ObjectResult VaildateLocation(string orderLat, string orderLng, int sellerId)
{
    if (orderLat == null || orderLng == null)
    {
        Log.ForContext<SalesAPIController>().ForContext("SellerId", sellerId).ForContext("Opration", "UpdateOrder").Warning("Order is Invalid for location");
        return BadRequest("T555 : Invalid Order");
    }
    if (orderLat.Length > 15 && orderLng.Length > 15)
    {
        Log.ForContext<SalesAPIController>().ForContext("SellerId", sellerId).ForContext("Opration", "UpdateOrder").Warning("Order is Invalid for Coordinates");
        return BadRequest("T565 : Coordinates is Invalid");
    }
    if (orderLat == "14.624054" && orderLng == "56.573993")
    {
        Log.ForContext<SalesAPIController>().ForContext("SellerId", sellerId).ForContext("Opration", "UpdateOrder").Warning("Order is Invalid for Coordinates");
        return BadRequest("T566 : Coordinates is Can't Detected");
    }
    return null;
}
```

---

### Step 4: Call TCC API

**Code Reference:** `SalesAPIController.cs:3944-3947` + `TCCApiHelper.cs:897-998`

```csharp
// SalesAPIController.cs:3944-3947
string tccResp = "{...}"; // Mock response
if (_config["AppSettings:BypassTCC"] == "0")
    tccResp = tcc.TransferOwnership(user, seller, transferOwnership, TccRegion, channel.TCCSourceType);
dynamic tccResult = JsonConvert.DeserializeObject(tccResp);
```

#### TCC Method Signature

**Code Reference:** `TCCApiHelper.cs:897`

```csharp
public string TransferOwnership(UserDefinition user, Seller seller, TransferOwnership ord, int TccRegion, int SourceType, bool reverse = false, bool IsTest = false)
```

#### TCC Request Payload

**Code Reference:** `TCCApiHelper.cs:917-952`

```csharp
TccUpdateRequest addNumberRequest = new TccUpdateRequest
{
    apiKey = (!IsTest) ? _config["ApiSettings:SematiApiKey"] : _config["ApiSettings:SematiApiKeyTest"],
    token = seller.TccToken,
    requestType = 17,  // <-- Transfer Ownership type
    person = new TccPersonReq
    {
        personId = ord.ToIdNumber,
        IdType = (int)((ord.IdType == IdType.Diplomat) ? IdType.Resident : ord.IdType),
        nationality = ord.Nationality,
        exceptionFlag = 11,  // <-- Always 11 for IAMAppToken
        iamAppToken = ord.IAMAppToken
    },
    mobileNumber = new TccMobileNumber
    {
        msisdn = $"966{ord.MSISDN}",
        msisdnType = ord.MSISDN.StartsWith("5") ? "V" : "D",
        simList = simlist.ToArray(),
        oldOwnerId = ord.FromIdNumber,
        subscriptionType = ord.SubType,
        isDefault = false
    },
    Operator = new TccOperator
    {
        sourceType = SourceType,
        sourceId = orgNumber,
        operatorTCN = DateTime.Now.ToString("yyyyMMddHHmmssfff"),
        region = TccRegion.ToString("D2"),
        employeeIdType = 1,
        employeeUsername = user.Username,
        employeeId = seller.IdNumber,
        deviceId = seller.DeviceId,
        branchAddress = $"{TrimLongLat(ord.OrderLat)},{TrimLongLat(ord.OrderLng)}",
    }
};
```

#### TCC Endpoint

**Code Reference:** `TCCApiHelper.cs:903, 956`

| Property | Value |
|----------|-------|
| **Base URL** | `_config["ApiSettings:SematiURL"]` |
| **Action URL** | `/TCC-Web/api-tcc/individual/v2/verify` |

#### TCC Response Tracing

**Code Reference:** `TCCApiHelper.cs:971-986`

```csharp
db.TccTraces.Add(new TccTraces
{
    EventType = "TransferOwnership",
    ClientTCN = addNumberRequest.Operator.operatorTCN,
    OrderId = ord.Id,
    SellerUserName = seller.SellerUserName,
    TCCRequest = JsonConvert.SerializeObject(addNumberRequest),
    TCCResponse = resp.Content,
    TCCTCN = tccResp.tcn?.Value,
    TCCCode = (int)tccResp.code?.Value,
    TCCMessage = tccResp.message?.Value,
});
db.SaveChanges();
```

---

### Step 5: Handle TCC Response

**Code Reference:** `SalesAPIController.cs:3948-3973`

#### On Success (code == 600)

```csharp
// SalesAPIController.cs:3948-3951
if (tccResult.code.Value == 600)
{
    transferOwnership.FirstName = (!string.IsNullOrEmpty(tccResult.person.trFirst?.Value))
        ? tccResult.person.trFirst
        : tccResult.person.first;
    transferOwnership.LastName = (!string.IsNullOrEmpty(tccResult.person.trFamily?.Value))
        ? tccResult.person.trFamily
        : tccResult.person.family;
```

| Field Updated | Priority 1 | Priority 2 | Code Line |
|---------------|------------|------------|-----------|
| `FirstName` | `tccResult.person.trFirst` | `tccResult.person.first` | `:3950` |
| `LastName` | `tccResult.person.trFamily` | `tccResult.person.family` | `:3951` |

#### On Failure (code != 600)

```csharp
// SalesAPIController.cs:3968-3973
else
{
    db.Entry(transferOwnership).State = Microsoft.EntityFrameworkCore.EntityState.Modified;
    db.SaveChanges();
    return BadRequest($"TCC: {tccResult.message?.Value}");
}
```

---

### Step 6: Call BSS API

**Code Reference:** `SalesAPIController.cs:3952-3960` + `BssApiHelper.cs:2795-2822`

```csharp
// SalesAPIController.cs:3952-3954
var bssResp = bss.TransferOwnership(transferOwnership, seller);
transferOwnership.BSSRequest = bssResp.req;
transferOwnership.BSSResponse = bssResp.resp;
```

#### BSS Method

**Code Reference:** `BssApiHelper.cs:2795-2822`

```csharp
public dynamic TransferOwnership(TransferOwnership ord, Seller seller)
{
    // ...
    string actionURL = "/apiaccess/SubscriberServices/SubscriberServices";
    // ...
    request.AddHeader("SOAPAction", "TransferOwnership");
    string xmlBody = GetTransferOwnershipSoap(ord, seller);
    // ...
    return new { code = retCode, msg = retMsg, resp = resp.Content, req = xmlBody };
}
```

#### BSS SOAP Request Structure

**Code Reference:** `BssApiHelper.cs:1445-1532`

| Element | Value | Code Line |
|---------|-------|-----------|
| **SOAP Action** | `TransferOwnership` | `:2809` |
| **Endpoint** | `/apiaccess/SubscriberServices/SubscriberServices` | `:2803` |
| **Message Type** | `TransferOwnershipReqMsg` | `:1475` |
| **Channel** | `_config["ApiSettings:BSSChannel"]` | `:1479` |
| **PartnerId** | `101` | `:1480` |
| **AccessUser** | `_config["ApiSettings:BSSUser"]` | `:1485` |
| **TCC_FLAG** | `1` | `:1493` |
| **CustLevel** | `4` (Postpaid) or `1` (Prepaid) based on SubType | `:1456` |
| **PaymentType** | `SubType` (with 2→1 mapping) | `:1500` |
| **BillCycleType** | `"28"` (Postpaid) or `"15"` (Prepaid) | `:1506` |

#### BSS Error Handling

**Code Reference:** `SalesAPIController.cs:3955-3960`

```csharp
if (long.Parse(bssResp.code) != 0)
{
    db.Entry(transferOwnership).State = Microsoft.EntityFrameworkCore.EntityState.Modified;
    db.SaveChanges();
    return BadRequest($"CRM: {bssResp.msg}");
}
```

---

### Step 7: Change Primary Offer (Conditional)

**Code Reference:** `SalesAPIController.cs:3961-3962`

```csharp
if ((transferOwnership.IdType != IdType.Citizen) && (transferOwnership.IdType != IdType.Resident))
    bss.ChangePrimaryOffer(transferOwnership.MSISDN, (transferOwnership.SubType == 0) ? "1014552422" : "366876333");
```

| Condition | Offer ID | Description |
|-----------|----------|-------------|
| `IdType != Citizen && IdType != Resident && SubType == 0` | `1014552422` | Prepaid Visitor Offer |
| `IdType != Citizen && IdType != Resident && SubType != 0` | `366876333` | Postpaid Visitor Offer |

#### ChangePrimaryOffer Method

**Code Reference:** `BssApiHelper.cs:2949-2978`

```csharp
public object ChangePrimaryOffer(string MSISDN, string primaryOffer)
{
    MSISDN = Regex.Replace(MSISDN, @"^(05|\+9665|5|009665|9665)", "5");
    // ...
    string actionURL = "/apiaccess/OfferingServices/OfferingServices";
    // ...
    request.AddHeader("SOAPAction", "ChangePrimaryOffering");
    string xmlBody = GetChangePrimaryOfferingSoap(MSISDN, primaryOffer);
    // ...
}
```

---

### Step 8: Save & Return

**Code Reference:** `SalesAPIController.cs:3963-3966`

```csharp
transferOwnership.Status = 1;
db.Entry(transferOwnership).State = Microsoft.EntityFrameworkCore.EntityState.Modified;
db.SaveChanges();
return Ok(transferOwnership);
```

---

## 5. Data Model

### TransferOwnership Entity

**Code Reference:** `Models/Tables/SalesOrder.cs:314-348`

```csharp
public class TransferOwnership
{
    [Key]
    [DatabaseGenerated(DatabaseGeneratedOption.Identity)]
    public int Id { get; set; }
    public string MSISDN { get; set; }
    public string FromIdNumber { get; set; }
    public string ToIdNumber { get; set; }
    public IdType IdType { get; set; }
    public IdType OldIdType { get; set; }
    public int Nationality { get; set; }
    public FingerIndex FingerIndex { get; set; }
    public string FingerImage { get; set; }
    public string IAMAppToken { get; set; }
    public int SubType { get; set; }
    public decimal Cost { get; set; }
    public decimal WalletBalanceBefore { get; set; }
    public decimal WalletBalanceAfter { get; set; }
    public int Status { get; set; }
    public int SellerId { get; set; }
    public string OTP { get; set; }
    public string OwnerOTP { get; set; }
    public DateTime? OTPExpiry { get; set; }
    public DateTime TransDate { get; set; }
    public string BSSRequest { get; set; }
    public string BSSResponse { get; set; }
    public string CRMCustomerId { get; set; }
    public string OldCRMCustomerId { get; set; }
    public string FirstName { get; set; }
    public string LastName { get; set; }
    public string ICCID { get; set; }
    public string IMSI { get; set; }
    public string OrderLat { get; set; }
    public string OrderLng { get; set; }
}
```

### IdType Enum

**Code Reference:** `Models/Tables/SalesOrder.cs:36-50`

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

## 6. Database Tables Used

| Table | Usage | Code Reference |
|-------|-------|----------------|
| `TransferOwnership` | Main record | `:3929` |
| `Seller` | Agent info | `:3932` |
| `SalesPartner` | Partner info | `:3933` |
| `SalesChannel` | Channel config | `:3934` |
| `City` | Seller city | `:3936` |
| `Province` | TCC region | `:3937` |
| `TccTraces` | API logging | `TCCApiHelper.cs:973` |

---

## 7. External API Summary

| API | Method | Endpoint | SOAP Action | Code Reference |
|-----|--------|----------|-------------|----------------|
| **TCC** | `TransferOwnership` | `/TCC-Web/api-tcc/individual/v2/verify` | N/A (REST) | `TCCApiHelper.cs:956` |
| **BSS** | `TransferOwnership` | `/apiaccess/SubscriberServices/SubscriberServices` | `TransferOwnership` | `BssApiHelper.cs:2803,2809` |
| **BSS** | `ChangePrimaryOffer` | `/apiaccess/OfferingServices/OfferingServices` | `ChangePrimaryOffering` | `BssApiHelper.cs:2958,2964` |

---

## 8. Logging

| Location | Logger | Context | Code Reference |
|----------|--------|---------|----------------|
| TCC Request | Serilog | `API=TCCTransferOwnership, OrderId` | `TCCApiHelper.cs:954` |
| TCC Response | Serilog | `API=TCCTransferOwnership, OrderId` | `TCCApiHelper.cs:969` |
| TCC Error | Serilog | `API=TCCTransferOwnership, OrderId` | `TCCApiHelper.cs:993` |
| BSS Request | Serilog | `Type=TransferOwnership, OrderId` | `BssApiHelper.cs:2812` |
| BSS Response | Serilog | `Type=TransferOwnership, OrderId` | `BssApiHelper.cs:2814` |
| Location Validation | Serilog | `SellerId, Opration=UpdateOrder` | `SalesAPIController.cs:4008,4013,4018` |

---

## 9. Error Responses

| Stage | Error Format | Example | Code Reference |
|-------|--------------|---------|----------------|
| Location null | `T555 : Invalid Order` | `BadRequest("T555 : Invalid Order")` | `:4009` |
| Coordinates long | `T565 : Coordinates is Invalid` | `BadRequest("T565 : Coordinates is Invalid")` | `:4014` |
| Coordinates default | `T566 : Coordinates is Can't Detected` | `BadRequest("T566 : Coordinates is Can't Detected")` | `:4019` |
| TCC Failure | `TCC: {message}` | `BadRequest($"TCC: {tccResult.message?.Value}")` | `:3972` |
| BSS Failure | `CRM: {message}` | `BadRequest($"CRM: {bssResp.msg}")` | `:3959` |
| Exception | `{exception.Message}` | `BadRequest(e.Message)` | `:3978` |

---

## 10. State Changes

| Field | Before | After | Condition | Code Reference |
|-------|--------|-------|-----------|----------------|
| `Status` | `0` | `1` | On full success | `:3963` |
| `FirstName` | - | TCC response | TCC success | `:3950` |
| `LastName` | - | TCC response | TCC success | `:3951` |
| `BSSRequest` | - | BSS SOAP request | Always after TCC success | `:3953` |
| `BSSResponse` | - | BSS SOAP response | Always after TCC success | `:3954` |
| `IAMAppToken` | - | Request value | If provided | `:3930` |

---

## 11. Sequence Diagram

```
Agent                    SalesAPIController              Database                 TCCApiHelper                BssApiHelper
  │                             │                           │                          │                          │
  │ POST /TransferOwnership     │                           │                          │                          │
  │ {Id, IAMAppToken}           │                           │                          │                          │
  │────────────────────────────▶│                           │                          │                          │
  │                             │                           │                          │                          │
  │                             │ Find TransferOwnership    │                          │                          │
  │                             │ Find Seller               │                          │                          │
  │                             │ Find SalesPartner         │                          │                          │
  │                             │ Find SalesChannel         │                          │                          │
  │                             │ Find City                 │                          │                          │
  │                             │ Find Province             │                          │                          │
  │                             │──────────────────────────▶│                          │                          │
  │                             │◀──────────────────────────│                          │                          │
  │                             │                           │                          │                          │
  │                             │ VaildateLocation()        │                          │                          │
  │                             │──────┐                    │                          │                          │
  │                             │◀─────┘ (return null=OK)   │                          │                          │
  │                             │                           │                          │                          │
  │                             │ TransferOwnership()       │                          │                          │
  │                             │─────────────────────────────────────────────────────▶│                          │
  │                             │                           │                          │                          │
  │                             │                           │     POST /TCC-Web/api-tcc/individual/v2/verify      │
  │                             │                           │                          │─────────▶ TCC Server     │
  │                             │                           │                          │◀─────────                │
  │                             │                           │                          │                          │
  │                             │                           │ Add TccTraces            │                          │
  │                             │                           │◀─────────────────────────│                          │
  │                             │◀─────────────────────────────────────────────────────│                          │
  │                             │  {code:600, person:{...}} │                          │                          │
  │                             │                           │                          │                          │
  │                             │ TransferOwnership()       │                          │                          │
  │                             │────────────────────────────────────────────────────────────────────────────────▶│
  │                             │                           │                          │                          │
  │                             │                           │     SOAP TransferOwnership                          │
  │                             │                           │                          │          │───────▶ BSS   │
  │                             │                           │                          │          │◀───────       │
  │                             │◀────────────────────────────────────────────────────────────────────────────────│
  │                             │  {code:0, msg:...}        │                          │                          │
  │                             │                           │                          │                          │
  │                             │ [If Visitor]              │                          │                          │
  │                             │ ChangePrimaryOffer()      │                          │                          │
  │                             │────────────────────────────────────────────────────────────────────────────────▶│
  │                             │◀────────────────────────────────────────────────────────────────────────────────│
  │                             │                           │                          │                          │
  │                             │ Update Status=1           │                          │                          │
  │                             │ SaveChanges()             │                          │                          │
  │                             │──────────────────────────▶│                          │                          │
  │                             │◀──────────────────────────│                          │                          │
  │                             │                           │                          │                          │
  │ 200 OK                      │                           │                          │                          │
  │ {TransferOwnership object}  │                           │                          │                          │
  │◀────────────────────────────│                           │                          │                          │
```

---

## 12. Configuration Dependencies

| Config Key | Usage | Code Reference |
|------------|-------|----------------|
| `AppSettings:BypassTCC` | Skip TCC call if = "1" | `SalesAPIController.cs:3945` |
| `AppSettings:RBMOrganizationNo` | Org number for SourceType=1 | `TCCApiHelper.cs:899` |
| `ApiSettings:SematiURL` | TCC base URL | `TCCApiHelper.cs:903` |
| `ApiSettings:SematiApiKey` | TCC API key | `TCCApiHelper.cs:920` |
| `ApiSettings:BSSChannel` | BSS channel ID | `BssApiHelper.cs:1479` |
| `ApiSettings:BSSUser` | BSS access user | `BssApiHelper.cs:1485` |
| `ApiSettings:BSSPassword` | BSS password | `BssApiHelper.cs:1525` |

---

## 13. Code File References

| File | Path | Purpose |
|------|------|---------|
| `SalesAPIController.cs` | `Modules/SalesAPI/SalesAPIController.cs` | Main API endpoint |
| `TCCApiHelper.cs` | `Modules/Common/Helpers/TCCApiHelper.cs` | TCC integration |
| `BssApiHelper.cs` | `Modules/Common/Helpers/BssApiHelper.cs` | BSS integration |
| `SalesOrder.cs` | `Models/Tables/SalesOrder.cs` | Entity definitions |

---

*Document generated from source code analysis - December 2025*
