# Postpaid Call Line Activation - Complete Technical Guide
## Step-by-Step Implementation Reference

---

## Entry Point

### Main File:
```
📁 RedBullSalesPortal/RedBullSalesPortal.Web/Modules/SalesAPI/SalesAPIController.cs
```

### Controller Definition:
```csharp
// SalesAPIController.cs - Lines 70-74
[Authorize(AuthenticationSchemes = JwtBearerDefaults.AuthenticationScheme)]
[Route("api/Sales/[action]")]
[ConnectionKey(typeof(MyRow))]
[IgnoreAntiforgeryToken]
public class SalesAPIController : ServiceEndpoint
```

### Base URL:
```
https://{your-host}/api/Sales/{action}
```

---

## Source Files Map

| File | Path | Purpose |
|------|------|---------|
| **SalesAPIController.cs** | `Modules/SalesAPI/` | Main Controller - All APIs |
| **BssApiHelper.cs** | `Modules/Common/Helpers/` | Huawei CRM/CBS Integration |
| **TCCApiHelper.cs** | `Modules/Common/Helpers/` | CITC TCC Integration |
| **NafithHelper.cs** | `Modules/Common/Helpers/` | Nafith Promissory Notes |
| **SMSHelper.cs** | `Modules/Common/Helpers/` | SMS via SMPP |
| **SalesOrder.cs** | `Models/Tables/` | Order Entity + Enums |
| **SalesDbContext.cs** | `Models/` | Database Context |

---

## Important Constants

```csharp
// 📍 SalesAPIController.cs - Lines 82-83
const decimal VAT = 0.15m;           // 15% Value Added Tax
const decimal ExtraSIMCost = 25m;    // Extra SIM price: 25 SAR
```

```csharp
// 📍 SalesOrder.cs - Lines 15-23
public enum ActivationType
{
    PrepaidNewSIM = 0,      // Prepaid - New SIM
    PrepaidPortIn = 1,      // Prepaid - Port-In
    PrepaidDataSIM = 2,     // Prepaid - Data SIM
    PostpaidNewSIM = 3,     // Postpaid - New SIM ⬅️
    PostpaidPortIn = 4,     // Postpaid - Port-In ⬅️
    PostpaidDataSIM = 5     // Postpaid - Data SIM ⬅️
}
```

---

## Step-by-Step Process

---

## 🔷 Step 1: Customer Validation & Order Creation

### API Endpoint:
```
POST /api/Sales/ValidateIdNew
```

### 📍 Code Location:
```
SalesAPIController.cs - Lines 922-1094
```

### Request:
```json
{
  "IdNumber": "1234567890",
  "IdType": 1,
  "Nationality": 113,
  "ContactNumber": "0512345678",
  "IsMNP": "3",
  "IsESim": "0"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| IdNumber | string | Yes | Saudi ID or Iqama |
| IdType | int | Yes | 1=Citizen, 2=Resident, 3=Passport, etc. |
| Nationality | int | Yes | TCC Nationality code |
| ContactNumber | string | Yes | Contact phone number |
| IsMNP | string | Yes | "3"=PostpaidNewSIM, "4"=PostpaidPortIn, "5"=PostpaidDataSIM |
| IsESim | string | No | "0"=Physical SIM, "1"=eSIM |

### What Happens Internally?

#### 1.1 Blacklist Check
```csharp
// 📍 SalesAPIController.cs - Lines 932-933
if (db.BlackLists.Find(IdNumber) != null)
    return BadRequest("Sorry, this Person is BlackListed");
```

#### 1.2 Fetch Customer Info from CRM
```csharp
// 📍 SalesAPIController.cs - Line 969
var crmResponse = bss.GetCustomerInformation(req.IdNumber.Value, (int)req.IdType.Value);
```

**🔗 External API Call:**
```
// 📍 BssApiHelper.cs - Lines 3676-3730
Endpoint: /apiaccess/CustomerServices/CustomerServices
SOAPAction: QueryCustomerInformation
```

#### 1.3 Block POS from Selling Postpaid
```csharp
// 📍 SalesAPIController.cs - Lines 972-973
if ((sellerType == 1) && (isPostpaid))
    return BadRequest("POS are not allowed to sell Postpaid");
```

#### 1.4 TCC Eligibility Check
```csharp
// 📍 SalesAPIController.cs - Lines 1016-1017
if (_config["AppSettings:BypassTCC"] == "0")
    tccRespone = tcc.CheckEligibility(req.IdNumber.Value, (int)req.Nationality.Value,
                                       (int)req.IdType.Value, Subtype, seller);
```

**🔗 External API Call:**
```
// 📍 TCCApiHelper.cs - Lines 1722-1792
Purpose: Verify customer eligibility with CITC
Returns: TCN (Transaction Control Number)
```

#### 1.5 Create Order Record
```csharp
// 📍 SalesAPIController.cs - Lines 1046-1066
SalesOrder order = new SalesOrder
{
    SellerId = int.Parse(user.Id),
    EligibilityTCN = tccResult.tcn.Value,
    IdNumber = req.IdNumber.Value,
    Nationality = (int)req.Nationality.Value,
    IdType = (IdType)req.IdType.Value,
    SubType = 1,  // Postpaid
    Status = OrderStatus.New,
    OrderDate = DateTime.Now,
    OTP = otp,
    OTPExpiry = DateTime.Now.AddMinutes(5),
    IsMNP = (ActivationType)int.Parse(isMNP),
    IsESim = int.Parse(isESim ?? "0"),
    IsPostpaid = 1
};
db.SalesOrders.Add(order);
db.SaveChanges();
```

### Response:
```json
{
  "custInfo": {
    "idNumber": "1234567890",
    "firstName": "Mohammed",
    "lastName": "Ahmed",
    "customerId": "CUST123456"
  },
  "tccEligibility": {
    "tcn": "e81e370a-c021-49ca-af46-62e4285fe3d7",
    "code": 600,
    "message": "success"
  },
  "order": {
    "Id": 930001,
    "Status": 0,
    "SubType": 1,
    "IsPostpaid": 1
  }
}
```

---

## 🔷 Step 2: Update Basic Information

### API Endpoint:
```
POST /api/Sales/UpdateOrder
```

### 📍 Code Location:
```
SalesAPIController.cs - Lines 1187-1192
```

### Request:
```json
{
  "OrderId": 930001,
  "step": "basic-info",
  "FirstName": "Mohammed",
  "LastName": "Ahmed",
  "Email": "customer@email.com"
}
```

### What Happens?
```csharp
// 📍 SalesAPIController.cs - Lines 1187-1192
case "basic-info":
    ord.Status = OrderStatus.InProgress;
    ord.FirstName = param.FirstName?.Value;
    ord.LastName = param.LastName?.Value;
    ord.Email = param.Email.Value;
    break;
```

---

## 🔷 Step 3: Update Address Information

### API Endpoint:
```
POST /api/Sales/UpdateOrder
```

### 📍 Code Location:
```
SalesAPIController.cs - Lines 1193-1198
```

### Request:
```json
{
  "OrderId": 930001,
  "step": "address-info",
  "Lat": "24.7136",
  "Lng": "46.6753",
  "City": "Riyadh",
  "Address": "King Fahd Road"
}
```

### What Happens?
```csharp
// 📍 SalesAPIController.cs - Lines 1193-1198
case "address-info":
    ord.OrderLat = param.Lat?.Value + "";
    ord.OrderLng = param.Lng?.Value + "";
    ord.OrderCity = param.City?.Value;
    ord.OrderAddress = param.Address?.Value;
    break;
```

---

## 🔷 Step 4: Fetch Available Numbers

### API Endpoint:
```
POST /api/Sales/GetMSISDNs
```

### 📍 Code Location:
```
SalesAPIController.cs - Lines 2985-2999
```

### Request:
```json
{
  "vanity": 4,
  "filter": "575",
  "count": 10
}
```

### What Happens?
```csharp
// 📍 SalesAPIController.cs - Lines 2985-2999
public IActionResult GetMSISDNs([FromServices] IUserRetrieveService userRetriever, dynamic param)
{
    // Calls BSS to get available numbers
}
```

**🔗 External API Call:**
```
// 📍 BssApiHelper.cs - Lines 3220-3260
Method: GetAvailableMSISDNs()
Endpoint: /apiaccess/InventoryServices/InventoryServices
SOAPAction: QueryNumber
```

### Vanity Number Pricing Tiers:
```csharp
// 📍 BssApiHelper.cs - Lines 73-87
VanityPrices.Add(1, 15000);  // 15,000 SAR
VanityPrices.Add(2, 5000);   // 5,000 SAR
VanityPrices.Add(3, 2500);   // 2,500 SAR
VanityPrices.Add(4, 0);      // Free (Standard)
VanityPrices.Add(5, 35000);  // 35,000 SAR
VanityPrices.Add(6, 750);    // 750 SAR
VanityPrices.Add(7, 400);    // 400 SAR
```

---

## 🔷 Step 5: Select Number

### API Endpoint:
```
POST /api/Sales/UpdateOrder
```

### 📍 Code Location:
```
SalesAPIController.cs - Lines 1199-1264
```

### Request:
```json
{
  "OrderId": 930001,
  "step": "select-number",
  "MSISDN": "575511234",
  "MSISDNCost": 5000,
  "IsMNP": "3",
  "ExtraSIM": "[{\"enabled\": true, \"eSIM\": 0}]"
}
```

### What Happens?

#### 5.1 Format MSISDN
```csharp
// 📍 SalesAPIController.cs - Line 1202
ord.MSISDN = Regex.Replace(param.MSISDN.Value, @"^(05|\+9665|5|009665|9665)", "5");
```

#### 5.2 Calculate Cost with VAT
```csharp
// 📍 SalesAPIController.cs - Line 1203
ord.MSISDNCost = MSISDNCost * (1 + VAT);  // Price + 15% VAT
```

#### 5.3 Process ExtraSIM (Postpaid Only!)
```csharp
// 📍 SalesAPIController.cs - Lines 1211-1261
if ((ord.IsPostpaid == 1) && (param.ExtraSIM?.Value != null))
{
    List<dynamic> extraSIM = JsonConvert.DeserializeObject<List<dynamic>>(param.ExtraSIM.Value);
    int extSims = extraSIM.Where(a => (bool)a.enabled).Count();

    // ⚠️ IMPORTANT RULE: ExtraSIM NOT allowed with Port-In
    if ((extSims > 0) && (ord.IsMNP == ActivationType.PostpaidPortIn))
        throw new Exception("Extra SIMs are not allowed with Port-In requests");

    foreach (var sim in extraSIM)
    {
        if ((bool)sim.enabled)
        {
            // Get Data MSISDN from BSS
            string MSISDN = bss.GetDataMSISDN(slr);

            ExtraSIM extSIM = new ExtraSIM
            {
                MSISDN = MSISDN,
                OrderId = ord.Id,
                IsESIM = (int)sim.eSIM,
                Status = 0
            };
            db.ExtraSIMs.Add(extSIM);
            extSimCount++;
        }
    }

    // ⚠️ IMPORTANT RULE: First ExtraSIM is FREE
    // 📍 Line 1255
    ord.ExtraSIMCost = (extSimCount > 0) ? (decimal)(extSimCount - 1) * ExtraSIMCost : 0;

    // Postpaid: No VAT on ExtraSIM
    // 📍 Line 1256
    ord.ExtraSIMCost = (ord.IsPostpaid == 0) ? ord.ExtraSIMCost * (1 + VAT) : ord.ExtraSIMCost;
}
```

#### 5.4 Lock Number in System
```csharp
// 📍 SalesAPIController.cs - Lines 1262-1263
if ((ord.IsMNP == ActivationType.PrepaidNewSIM) || (ord.IsMNP == ActivationType.PostpaidNewSIM))
    bss.OperateMSISDN(ord.MSISDN, slr);
```

**🔗 External API Call:**
```
// 📍 BssApiHelper.cs - Lines 2981-3182
Method: OperateMSISDN()
Endpoint: /apiaccess/InventoryServices/InventoryServices
SOAPAction: OperateNumber
Operation Type: "1029" (Lock/Reserve)
```

---

## 🔷 Step 6: Fetch Available Plans

### API Endpoint:
```
POST /api/Sales/GetPrimaryPlans
```

### 📍 Code Location:
```
SalesAPIController.cs - Lines 1946-1988
```

### Request:
```json
{
  "SubType": 1,
  "IsESim": 0
}
```

---

## 🔷 Step 7: Fetch Addons

### API Endpoint:
```
POST /api/Sales/GetAddons
```

### 📍 Code Location:
```
SalesAPIController.cs - Lines 2459-2498
```

### Request:
```json
{
  "PrimaryOffer": "100001"
}
```

---

## 🔷 Step 8: Select Plan

### API Endpoint:
```
POST /api/Sales/UpdateOrder
```

### 📍 Code Location:
```
SalesAPIController.cs - Lines 1265-1285
```

### Request:
```json
{
  "OrderId": 930001,
  "step": "select-plan",
  "PlanId": "100001",
  "PlanName": "Postpaid 200",
  "Addons": "[\"ADDON001\"]"
}
```

### What Happens?

#### 8.1 Calculate Plan Cost (NO VAT for Postpaid!)
```csharp
// 📍 SalesAPIController.cs - Line 1270
// ⚠️ CRITICAL RULE: Postpaid = No VAT, Prepaid = With VAT
ord.PlanCost = (ord.IsPostpaid == 1)
    ? db.PlanArts.Find(ord.PlanId).Price                    // Postpaid: Price as-is
    : db.PlanArts.Find(ord.PlanId).Price * (1 + VAT);       // Prepaid: Price + 15%
```

#### 8.2 Calculate Addon Cost
```csharp
// 📍 SalesAPIController.cs - Lines 1272-1278
decimal addonTotal = 0;
List<string> addons = JsonConvert.DeserializeObject<List<string>>(param.Addons.Value);
foreach (var addon in addons)
{
    // ⚠️ Same rule: Postpaid = No VAT
    addonTotal += (ord.IsPostpaid == 0)
        ? db.Addons.Find(addon).Price * (1 + VAT)   // Prepaid: With VAT
        : db.Addons.Find(addon).Price;              // Postpaid: No VAT
}
ord.AddonsCost = addonTotal;
```

---

## 🔷 Step 9: Create Promissory Note (Vanity Numbers Only)

### ⚠️ This step is CONDITIONAL - Only if MSISDNCost > 0

### API Endpoint:
```
POST /api/Sales/CreateSalesPromissoryNote
```

### 📍 Code Location:
```
SalesAPIController.cs - Lines 3222-3280
```

### Request:
```json
{
  "Id": 930001
}
```

### What Happens?
```csharp
// 📍 SalesAPIController.cs - Lines 3256-3276
if (order.IsMNP == ActivationType.PostpaidNewSIM)
{
    // Lookup commitment duration from CommitmentMatrix
    CommitmentMatrix matrix = db.CommitmentMatrices
        .Where(a => (a.MSISDNPrice == order.MSISDNCost / 1.15m) && (a.PrimaryOffer == order.PlanId))
        .FirstOrDefault();

    note = new PromissoryNote
    {
        IdNumber = order.IdNumber,
        MSISDN = order.MSISDN,
        OrderId = $"SLS{order.Id}",
        TotalAmount = matrix.MSISDNPrice * (1 + VAT),
        Duration = matrix.Duration,
        City = order.OrderCity
    };

    // Create promissory note in Nafith
    PromissoryNote res = nafith.CreateSanad(note);
}
```

**🔗 External API Call:**
```
// 📍 NafithHelper.cs - Lines 208+
Method: CreateSanad()
Purpose: Create promissory note in Nafith/Sanad system
Returns: SanadId, SanadStatus
```

### Check Promissory Note Status (Polling):
```
POST /api/Sales/GetPromissoryNoteStatus
// 📍 SalesAPIController.cs - Lines 3293-3304
```

---

## 🔷 Step 10: Send OTP

### API Endpoint:
```
POST /api/Sales/SendOrderOTP
```

### 📍 Code Location:
```
SalesAPIController.cs - Lines 232-403
```

### Request:
```json
{
  "OrderId": 930001,
  "OrderType": 0,
  "WithPN": "1"
}
```

### What Happens?

#### 10.1 Generate OTP
```csharp
// 📍 SalesAPIController.cs - Lines 344-348
Random rnd = new Random();
ord.OTP = rnd.Next(9999).ToString("0000");
ord.OTPExpiry = DateTime.Now.AddMinutes(2);
```

#### 10.2 Send OTP (Different for Prepaid vs Postpaid!)
```csharp
// 📍 SalesAPIController.cs - Lines 397-400
if (ord.SubType == 0)
    // Prepaid: Regular SMS
    await sms.SendSMS(Regex.Replace(contactNumber, @"^(05|\+9665|5|009665|9665)", "05"), message);
else
    // ⚠️ Postpaid: Via TCC Ballighny service
    tcc.SendBallighny(ord.IdNumber, message);
```

**🔗 External API Calls:**
```
// For Prepaid:
// 📍 SMSHelper.cs
Method: SendSMS()
Protocol: SMPP

// For Postpaid:
// 📍 TCCApiHelper.cs - Lines 413-444
Method: SendBallighny()
Purpose: Send OTP via Ballighny (CITC service)
```

---

## 🔷 Step 11: Authentication & Activation (Final Step)

### API Endpoint:
```
POST /api/Sales/UpdateOrder
```

### 📍 Code Location:
```
SalesAPIController.cs - Lines 1286-1534
```

### Request (Physical SIM):
```json
{
  "OrderId": 930001,
  "step": "authenticate",
  "ICCID": "8996610123456789012",
  "FingerIndex": 1,
  "FingerImage": "base64_fingerprint",
  "AuthMethod": "1",
  "OnBehalfOf": "John Doe"
}
```

### Request (eSIM):
```json
{
  "OrderId": 930001,
  "step": "authenticate",
  "Email": "customer@email.com",
  "AuthMethod": "3",
  "IAMToken": "nafath_token",
  "UseIAMToken": true
}
```

### What Happens? (Order is CRITICAL!)

#### 11.1 Get ICCID
```csharp
// 📍 SalesAPIController.cs - Line 1297
// Physical SIM: From request, eSIM: Auto-assigned from BSS
ord.ICCID = (ord.IsESim == 0) ? param.ICCID.Value : bss.PickESim(seller, ord.Id);
```

**🔗 External API Call (eSIM only):**
```
// 📍 BssApiHelper.cs - Lines 2464-2727
Method: PickESim()
Endpoint: /apiaccess/InventoryServices/InventoryServices
SOAPAction: PickNumber
```

#### 11.2 Validate ICCID Format
```csharp
// 📍 SalesAPIController.cs - Lines 1298-1299
if ((!Regex.Match(ord.ICCID, "^899661\\d{13}$").Success) &&
    (!Regex.Match(ord.ICCID, "^899661\\d{14}$").Success))
    return BadRequest("ICCID Is Invalid");
```

#### 11.3 Get SIM Details
```csharp
// 📍 SalesAPIController.cs - Line 1301
dynamic SIMInfo = (ord.IsESim == 0)
    ? bss.GetSIMDetails(ord.ICCID, seller, ord.Id)
    : new { IMSI = "42010" + ord.ICCID.Remove(ord.ICCID.Length - 1).Substring(9) };
ord.IMSI = SIMInfo.IMSI;
```

**🔗 External API Call:**
```
// 📍 BssApiHelper.cs - Lines 2729-2823
Method: GetSIMDetails()
Endpoint: /apiaccess/InventoryServices/InventoryServices
SOAPAction: QueryResource
```

#### 11.4 Update Email (eSIM only)
```csharp
// 📍 SalesAPIController.cs - Lines 1369-1373
if ((ord.CRMCustomerId != null) && (ord.IsESim == 1))
{
    bss.UpdateCustomerEmail(ord.CRMCustomerId, ord.Email);
    System.Threading.Thread.Sleep(2000);  // Wait 2 seconds
}
```

#### 11.5 Register with TCC
```csharp
// 📍 SalesAPIController.cs - Lines 1346-1357
if ((ord.IsMNP == ActivationType.PrepaidPortIn) || (ord.IsMNP == ActivationType.PostpaidPortIn))
{
    // Port-In
    tccResponse = tcc.NumberMNP(user, seller, ord, TccRegion, channel.TCCSourceType, ordSIMs);
}
else
{
    // New Number
    tccResponse = tcc.AddNumber(user, seller, ord, TccRegion, channel.TCCSourceType, ordSIMs);
}
```

**🔗 External API Calls:**
```
// For New Number:
// 📍 TCCApiHelper.cs - Lines 1001-1119
Method: AddNumber()

// For Port-In:
// 📍 TCCApiHelper.cs - Lines 688-802
Method: NumberMNP()
```

#### 11.6 Create Order in CRM
```csharp
// 📍 SalesAPIController.cs - Lines 1405-1435
if ((ord.IsMNP == ActivationType.PrepaidPortIn) || (ord.IsMNP == ActivationType.PostpaidPortIn))
{
    // Port-In
    dynamic crmResult = bss.CreateSalesMNPOrder(ord, seller, plan, matrix, initialBalance);
}
else
{
    // New Number
    dynamic crmResult = bss.CreateSalesOrder(ord, seller, plan, matrix, initialBalance);
}

// On failure: Cancel TCC registration (ROLLBACK)
if (crmResult.code != 0)
{
    tcc.CancelNumberGeneric(ord.MSISDN, ord.IdNumber, (int)ord.IdType, ord.Nationality, ord.Id);
    return BadRequest($"CRM: {crmResult.code}: {crmResult.message}");
}
```

**🔗 External API Calls:**
```
// For New Number:
// 📍 BssApiHelper.cs - Lines 3381-3419
Method: CreateSalesOrder()
Endpoint: /apiaccess/OrderServices/OrderServices
SOAPAction: CreateSaleOrder

// For Port-In:
// 📍 BssApiHelper.cs - Lines 3421-3459
Method: CreateSalesMNPOrder()
```

#### 11.7 Deduct from Wallet
```csharp
// 📍 SalesAPIController.cs - Lines 1437-1439
ord.WalletBalanceBefore = (decimal)seller.WalletBalance;
seller.WalletBalance -= (double)(ord.OrderTotal);
ord.WalletBalanceAfter = (decimal)seller.WalletBalance;
```

#### 11.8 Calculate Commission (POS Only)
```csharp
// 📍 SalesAPIController.cs - Lines 1441-1455
if ((partner.SalesChannel == 1) && (ord.OrderTotal > 0))
{
    decimal commission = ord.Commission;
    SellerCommission sellerCommission = new SellerCommission
    {
        Amount = commission,
        OrderId = ord.Id,
        SellerId = ord.SellerId,
        WalletBalanceBefore = (decimal)seller.WalletBalance
    };
    seller.WalletBalance += (double)commission;
    sellerCommission.WalletBalanceAfter = (decimal)seller.WalletBalance;
    db.SellerCommissions.Add(sellerCommission);
}
```

#### 11.9 Generate eContract
```csharp
// 📍 SalesAPIController.cs - Line 1481
ord.EContractFileName = GenerateEContract(user, ord);
```

#### 11.10 Update Status
```csharp
// 📍 SalesAPIController.cs - Line 1529
ord.Status = OrderStatus.Completed;
```

---

## 🔷 Post-Activation (Async Operations)

### Send Confirmation SMS:
```csharp
// 📍 SalesAPIController.cs - Lines 1483-1489
Task.Delay(120000).ContinueWith((task) =>  // After 2 minutes
{
    if (ord.IsMNP == ActivationType.PostpaidDataSIM)
        tcc.SendBallighny(ord.IdNumber, confirmationSMS);
    else
        sms.SendSMS("0" + SMSReceiver, confirmationSMS);
});
```

### Generate eInvoice (ZATCA):
```csharp
// 📍 SalesAPIController.cs - Lines 1490-1521
if (zatca_channels.Contains(seller.SellerChannel))
{
    eInvoiceRecord = GenerateEInvoiceLocal(eInvoiceRecord);
}
```

---

## External Integrations Summary

| System | File | Purpose |
|--------|------|---------|
| **BSS/CRM (Huawei)** | BssApiHelper.cs | Customer, Numbers, Orders management |
| **TCC (CITC)** | TCCApiHelper.cs | Eligibility, SIM registration, Ballighny |
| **Nafith** | NafithHelper.cs | Promissory notes for vanity numbers |
| **SMS Gateway** | SMSHelper.cs | OTP for Prepaid |
| **ZATCA** | EInvoiceHelper.cs | Electronic invoices |

---

## Key Differences: Prepaid vs Postpaid

| Aspect | Prepaid | Postpaid |
|--------|---------|----------|
| **VAT on Plan** | 15% VAT | NO VAT |
| **VAT on Addons** | 15% VAT | NO VAT |
| **OTP Delivery** | Regular SMS | Ballighny (TCC) |
| **ExtraSIM** | Not Available | Available |
| **Promissory Note** | Not Available | For Vanity Numbers |
| **SubType** | 0 | 1 |

---

## Quick Reference Table

| Step | API | Lines in SalesAPIController.cs |
|------|-----|-------------------------------|
| 1. Validate | ValidateIdNew | 922-1094 |
| 2. Basic Info | UpdateOrder(basic-info) | 1187-1192 |
| 3. Address | UpdateOrder(address) | 1193-1198 |
| 4. Get Numbers | GetMSISDNs | 2985-2999 |
| 5. Select Number | UpdateOrder(select-number) | 1199-1264 |
| 6. Get Plans | GetPrimaryPlans | 1946-1988 |
| 7. Get Addons | GetAddons | 2459-2498 |
| 8. Select Plan | UpdateOrder(select-plan) | 1265-1285 |
| 9. Promissory Note | CreateSalesPromissoryNote | 3222-3280 |
| 10. Send OTP | SendOrderOTP | 232-403 |
| 11. Authenticate | UpdateOrder(authenticate) | 1286-1534 |

---

## Flow Diagram

```
┌──────────────────────────────────────────────────────────────────────────┐
│                        POSTPAID ACTIVATION FLOW                          │
└──────────────────────────────────────────────────────────────────────────┘

     ┌─────────────────┐
     │ ValidateIdNew   │ ──► BSS.GetCustomerInformation()
     │ Lines 922-1094  │ ──► TCC.CheckEligibility()
     └────────┬────────┘
              │
              ▼
     ┌─────────────────┐
     │ UpdateOrder     │
     │ (basic-info)    │
     │ Lines 1187-1192 │
     └────────┬────────┘
              │
              ▼
     ┌─────────────────┐
     │ UpdateOrder     │
     │ (address-info)  │
     │ Lines 1193-1198 │
     └────────┬────────┘
              │
              ▼
     ┌─────────────────┐
     │ GetMSISDNs      │ ──► BSS.GetAvailableMSISDNs()
     │ Lines 2985-2999 │
     └────────┬────────┘
              │
              ▼
     ┌─────────────────┐
     │ UpdateOrder     │ ──► BSS.OperateMSISDN() [Lock]
     │ (select-number) │ ──► BSS.GetDataMSISDN() [ExtraSIM]
     │ Lines 1199-1264 │
     └────────┬────────┘
              │
              ▼
     ┌─────────────────┐
     │ GetPrimaryPlans │
     │ Lines 1946-1988 │
     └────────┬────────┘
              │
              ▼
     ┌─────────────────┐
     │ GetAddons       │
     │ Lines 2459-2498 │
     └────────┬────────┘
              │
              ▼
     ┌─────────────────┐
     │ UpdateOrder     │
     │ (select-plan)   │
     │ Lines 1265-1285 │
     └────────┬────────┘
              │
              ▼
     ┌─────────────────┐
     │ CreateSales     │ ──► Nafith.CreateSanad()
     │ PromissoryNote  │     (ONLY if MSISDNCost > 0)
     │ Lines 3222-3280 │
     └────────┬────────┘
              │ (conditional)
              ▼
     ┌─────────────────┐
     │ SendOrderOTP    │ ──► TCC.SendBallighny() [Postpaid]
     │ Lines 232-403   │
     └────────┬────────┘
              │
              ▼
     ┌─────────────────┐
     │ UpdateOrder     │ ──► BSS.PickESim() or GetSIMDetails()
     │ (authenticate)  │ ──► TCC.AddNumber() or NumberMNP()
     │ Lines 1286-1534 │ ──► BSS.CreateSalesOrder()
     └────────┬────────┘     ──► Wallet Deduction
              │              ──► Generate eContract
              ▼
     ┌─────────────────┐
     │    COMPLETED    │
     │   Status = 3    │
     └─────────────────┘
```

---

**This guide was created from direct source code analysis**
