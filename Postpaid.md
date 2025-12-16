📱 Postpaid Call Line Activation - Detailed Flow from Code
1️⃣ Basic Definitions
Activation Types for Postpaid
// In SalesOrder.cs
public enum ActivationType
{
    [Description("Postpaid New SIM")]
    PostpaidNewSIM = 3,        // ← New line
    
    [Description("Postpaid Port-In")]
    PostpaidPortIn = 4,        // ← Number transfer
    
    [Description("Postpaid Data SIM")]
    PostpaidDataSIM = 5        // ← Data only
}
Key Properties
Property	Value	Description
IsMNP	3 (PostpaidNewSIM)	Activation type
SubType	1 (Postpaid)	Subscription type
IsPostpaid	1	Postpaid flag
IsESim	0 or 1	Physical or eSIM
2️⃣ Key Differences: Postpaid vs Prepaid
┌─────────────────────────────────────────────────────────────────────┐
│                    POSTPAID vs PREPAID                              │
├──────────────────────┬──────────────────────┬───────────────────────┤
│       Property       │      Postpaid        │       Prepaid         │
├──────────────────────┼──────────────────────┼───────────────────────┤
│ VAT Tax              │ ❌ None (per invoice) │ ✅ 15% included       │
│ ExtraSIM             │ ✅ Available          │ ❌ Not available      │
│ Credit Limit         │ ✅ Set per plan       │ ❌ N/A                │
│ Bill Cycle           │ 28 days              │ 15 days              │
│ Contract/Commitment  │ ✅ Required           │ ❌ None               │
│ Promissory Note      │ ✅ Nafith integrated  │ ❌ N/A                │
│ OTP Delivery         │ Ballighny (Gov)      │ Standard SMS         │
│ Allowed Sellers      │ Retail only          │ Retail + POS         │
└──────────────────────┴──────────────────────┴───────────────────────┘
3️⃣ Complete Flow Diagram
┌─────────────────────────────────────────────────────────────────────────────┐
│              POSTPAID CALL LINE ACTIVATION FLOW                             │
│                   (Physical SIM & eSIM)                                     │
└─────────────────────────────────────────────────────────────────────────────┘

  📱 Mobile App                    🖥️ Sales API                    🔌 External Systems
       │                               │                                  │
       │  1. CreateOrder               │                                  │
       │──────────────────────────────>│                                  │
       │  (IdNumber, IsMNP=3, IsESim)  │                                  │
       │                               │                                  │
       │                               │  ⚠️ Validation: Is seller POS?   │
       │                               │  if (sellerType==1 && isPostpaid)│
       │                               │     return "POS not allowed"     │
       │                               │                                  │
       │                               │────────────────────────────────> │ TCC
       │                               │  CheckEligibility()              │
       │                               │<─────────────────────────────────│
       │                               │                                  │
       │                               │  → Create SalesOrder             │
       │                               │    IsPostpaid = 1                │
       │                               │    SubType = 1                   │
       │<──────────────────────────────│                                  │
       │                               │                                  │
       │  2. SendOrderOTP              │                                  │
       │──────────────────────────────>│                                  │
       │                               │────────────────────────────────> │ TCC
       │                               │  SendBallighny() ← Official Gov  │
       │<──────────────────────────────│<─────────────────────────────────│
       │                               │                                  │
       │  3. UpdateOrder (basic-info)  │                                  │
       │──────────────────────────────>│                                  │
       │                               │  → Status = InProgress           │
       │<──────────────────────────────│                                  │
       │                               │                                  │
       │  4. UpdateOrder (address-info)│                                  │
       │──────────────────────────────>│                                  │
       │                               │  → Save GPS Location             │
       │<──────────────────────────────│                                  │
       │                               │                                  │
       │  5. UpdateOrder (select-number)                                  │
       │──────────────────────────────>│                                  │
       │  (MSISDN, IsMNP=3)            │────────────────────────────────> │ BSS
       │                               │  OperateMSISDN()                 │
       │<──────────────────────────────│<─────────────────────────────────│
       │                               │                                  │
       │  6. UpdateOrder (select-plan) │                                  │
       │──────────────────────────────>│                                  │
       │  (PlanId, Addons, ExtraSIMs)  │                                  │
       │                               │  📊 Cost Calculation (NO VAT):   │
       │                               │  PlanCost = Price (NO VAT)       │
       │                               │  AddonsCost = Price (NO VAT)     │
       │                               │  ExtraSIMCost = Price (NO VAT)   │
       │                               │                                  │
       │                               │  ⚠️ ExtraSIM + PortIn = ❌        │
       │<──────────────────────────────│                                  │
       │                               │                                  │
       │  7. Promissory Note (Optional)│                                  │
       │──────────────────────────────>│────────────────────────────────> │ Nafith
       │                               │  CreatePromissoryNote()          │
       │                               │<─────────────────────────────────│
       │                               │  SanadStatus = "approved"        │
       │<──────────────────────────────│                                  │
       │                               │                                  │
       │  8. UpdateOrder (authenticate)│                                  │
       │──────────────────────────────>│                                  │
       │                               │                                  │
       │   ┌─────────────────────────────────────────────────────────┐   │
       │   │ Physical SIM (IsESim=0)  │  eSIM (IsESim=1)             │   │
       │   ├─────────────────────────────────────────────────────────┤   │
       │   │ ICCID = param.ICCID      │  ICCID = bss.PickESim()      │   │
       │   │ (from barcode scanner)   │  (system picks)              │   │
       │   │                          │                              │   │
       │   │ SIMInfo = GetSIMDetails()│  IMSI = Derived from ICCID   │   │
       │   └─────────────────────────────────────────────────────────┘   │
       │                               │                                  │
       │                               │────────────────────────────────> │ TCC
       │                               │  AddNumber()                     │
       │                               │<─────────────────────────────────│
       │                               │  code=600 ✅                     │
       │                               │                                  │
       │                               │  📋 Get Commitment Matrix        │
       │                               │  (Duration, AgreementId)         │
       │                               │                                  │
       │                               │────────────────────────────────> │ BSS/CRM
       │                               │  CreateSalesOrder()              │
       │                               │  ├─ PaymentType = 1 (Postpaid)   │
       │                               │  ├─ BillCycleType = 28           │
       │                               │  ├─ CreditLimit = plan.Limit     │
       │                               │  ├─ Contract/Agreement ✅        │
       │                               │  └─ EsimFlag = Y/N               │
       │                               │<─────────────────────────────────│
       │                               │                                  │
       │                               │  💰 Deduct from seller wallet    │
       │                               │  💵 Add commission               │
       │                               │                                  │
       │                               │  📄 Generate eContract            │
       │                               │  "(exclusive of VAT)"            │
       │                               │                                  │
       │                               │  ⏰ After 2 minutes...           │
       │                               │────────────────────────────────> │ SMS/Ballighny
       │                               │  Confirmation SMS                │
       │<──────────────────────────────│<─────────────────────────────────│
       │   ✅ Order Completed          │                                  │
       │                               │                                  │
4️⃣ Step-by-Step Code Details
📌 Step 1: Create Order
File: SalesAPIController.cs:970-1001
// Determine if Postpaid
bool isPostpaid = new string[] { "3", "4", "5" }.Contains(isMNP);
// "3" = PostpaidNewSIM, "4" = PostpaidPortIn, "5" = PostpaidDataSIM

// ⚠️ POS sellers are NOT allowed to sell Postpaid
if ((sellerType == 1) && (isPostpaid))
    return BadRequest("POS are not allowed to sell Postpaid");

// Create the order
SalesOrder order = new SalesOrder
{
    IsMNP = (ActivationType)int.Parse(isMNP),      // = 3 (PostpaidNewSIM)
    SubType = 1,                                    // Postpaid
    IsESim = int.Parse(isESim ?? "0"),             // 0=Physical, 1=eSIM
    IsPostpaid = 1,                                 // ← Key flag
    Status = OrderStatus.New,
    // ... other data
};
📌 Step 2: Send OTP (Key Difference)
File: SalesAPIController.cs:397-401
if (ord.SubType == 0)  // Prepaid
    await sms.SendSMS(contactNumber, OTPMessage);
else                   // Postpaid ← Different here
    tcc.SendBallighny(ord.IdNumber, OTPMessage);  // Official government SMS
Note: Postpaid uses Ballighny - the official government notification system
📌 Steps 3-4: Customer Info & Address
Same as Prepaid flow.
📌 Step 5: Select Number
case "select-number":
    ord.MSISDN = Regex.Replace(param.MSISDN.Value, @"^(05|\+9665|5|009665|9665)", "5");
    ord.MSISDNCost = MSISDNCost * (1 + VAT);  // VAT only on MSISDN cost
    ord.IsMNP = ActivationType.PostpaidNewSIM;  // = 3
    
    // Reserve the number
    if (ord.IsMNP == ActivationType.PostpaidNewSIM)
        bss.OperateMSISDN(ord.MSISDN, seller);
    break;
📌 Step 6: Select Plan ⭐ Critical Difference
File: SalesAPIController.cs:1270-1276
case "select-plan":
    // ════════════════════════════════════════════════════
    // 📊 Plan Cost Calculation - NO VAT for Postpaid!
    // ════════════════════════════════════════════════════
    ord.PlanCost = (ord.IsPostpaid == 1) ? 
        db.PlanArts.Find(ord.PlanId).Price :              // ❌ NO VAT
        db.PlanArts.Find(ord.PlanId).Price * (1 + VAT);   // ✅ WITH VAT
    
    // ════════════════════════════════════════════════════
    // 📊 Addon Cost Calculation - NO VAT for Postpaid!
    // ════════════════════════════════════════════════════
    foreach (var addon in addons)
    {
        addonTotal += (ord.IsPostpaid == 0) ? 
            db.Addons.Find(addon).Price * (1 + VAT) :     // ✅ Prepaid WITH VAT
            db.Addons.Find(addon).Price;                   // ❌ Postpaid NO VAT
    }
    ord.AddonsCost = addonTotal;
    break;
📌 Step 6 (Continued): ExtraSIM - Postpaid Exclusive
File: SalesAPIController.cs:1211-1257
// ════════════════════════════════════════════════════
// 🔹 ExtraSIM is ONLY available for Postpaid
// ════════════════════════════════════════════════════
if ((ord.IsPostpaid == 1) && (param.ExtraSIM?.Value != null))
{
    List<dynamic> extraSIM = JsonConvert.DeserializeObject<List<dynamic>>(param.ExtraSIM.Value);
    int extSims = extraSIM.Where(a => (bool)a.enabled).Count();
    
    // ⚠️ ExtraSIM NOT allowed with Port-In
    if ((extSims > 0) && (ord.IsMNP == ActivationType.PostpaidPortIn))
        throw new Exception("Extra SIMs are not allowed with Port-In requests");
    
    // Create ExtraSIM records
    foreach (var sim in extraSIM)
    {
        if ((bool)sim.enabled)
        {
            ExtraSIM extSIM = new ExtraSIM
            {
                OrderId = ord.Id,
                IsESIM = (int)sim.eSIM,    // 0=Physical, 1=eSIM
                Status = 0
            };
            db.ExtraSIMs.Add(extSIM);
            extSimCount++;
        }
    }
    
    // 💰 ExtraSIM Cost (first one is FREE)
    ord.ExtraSIMCost = (extSimCount > 0) ? (decimal)(extSimCount - 1) * 25m : 0;
    
    // ❌ NO VAT for Postpaid
    ord.ExtraSIMCost = (ord.IsPostpaid == 0) ? 
        ord.ExtraSIMCost * (1 + VAT) :    // Prepaid WITH VAT
        ord.ExtraSIMCost;                  // Postpaid NO VAT
}
ExtraSIM Rules:
┌────────────────────────────────────────────────────┐
│              ExtraSIM Rules                        │
├────────────────────────────────────────────────────┤
│ ✅ Available ONLY for Postpaid                     │
│ ❌ NOT allowed with Port-In                        │
│ 💰 25 SAR per extra SIM                           │
│ 🎁 First SIM is FREE                              │
│ 📱 Supports both Physical and eSIM                │
│ ❌ NO VAT (calculated in monthly invoice)          │
└────────────────────────────────────────────────────┘
📌 Step 7: Promissory Note - Postpaid Exclusive
File: PromissoryNote.cs
[Table("PromissoryNote")]
public class PromissoryNote
{
    public int Id { get; set; }
    public string IdNumber { get; set; }
    public string MSISDN { get; set; }
    public decimal TotalAmount { get; set; }
    public string SanadId { get; set; }           // Nafith Sanad ID
    public string SanadStatus { get; set; }       // "approved" = approved
    public DateTime DueDate { get; set; }
    public int Duration { get; set; }
}
Usage during activation:
// Check for approved promissory note
PromissoryNote note = (ord.IsPostpaid == 1) ? 
    db.PromissoryNotes.Where(a => 
        (a.OrderId == $"SLS{ord.Id}") && 
        (a.SanadStatus == "approved")).FirstOrDefault() 
    : null;

// If note is approved, MSISDN cost is covered by the system
if ((note != null) && (ord.IsMNP == ActivationType.PostpaidNewSIM))
{
    ord.OrderTotal -= ord.MSISDNCost ?? 0;
    ord.MSISDNCost = 0;  // Nafith covers the cost
}
📌 Step 8: Authentication & Activation ⭐
8.1 Physical SIM vs eSIM Handling
File: SalesAPIController.cs:1297-1301
// ════════════════════════════════════════════════════
// 📱 Difference between Physical SIM and eSIM
// ════════════════════════════════════════════════════

// Get ICCID
ord.ICCID = (ord.IsESim == 0) ? 
    param.ICCID.Value :          // Physical: from barcode scanner
    bss.PickESim(seller, ord.Id); // eSIM: system picks automatically

// Get SIM information
dynamic SIMInfo = (ord.IsESim == 0) ? 
    bss.GetSIMDetails(ord.ICCID, seller, ord.Id) :   // Physical: from BSS
    new { IMSI = "42010" + ord.ICCID.Remove(ord.ICCID.Length - 1).Substring(9) }; // eSIM: calculated

ord.IMSI = SIMInfo.IMSI;
┌─────────────────────────────────────────────────────────────────┐
│           Physical SIM vs eSIM                                  │
├───────────────────────────┬─────────────────────────────────────┤
│      Physical (IsESim=0)  │         eSIM (IsESim=1)             │
├───────────────────────────┼─────────────────────────────────────┤
│ ICCID = from scanner      │ ICCID = bss.PickESim()              │
│                           │ (system picks from inventory)       │
├───────────────────────────┼─────────────────────────────────────┤
│ SIMInfo = GetSIMDetails() │ IMSI = calculated from ICCID        │
│ (query from BSS)          │ "42010" + ICCID[9:-1]               │
├───────────────────────────┼─────────────────────────────────────┤
│ Validation: ^899661\d{13,14}$ │ Same validation                 │
└───────────────────────────┴─────────────────────────────────────┘
8.2 Get Commitment Matrix (Postpaid Exclusive)
CommitmentMatrix matrix = (ord.IsPostpaid == 1) ? 
    db.CommitmentMatrices.Where(a => 
        (a.PrimaryOffer == ord.PlanId) && 
        (a.MSISDNPrice == (MSDNCost / (1 + VAT)))).FirstOrDefault() 
    : null;
Commitment Matrix Model:
public class CommitmentMatrix
{
    public string PrimaryOffer { get; set; }    // Plan ID
    public int Duration { get; set; }           // Commitment period (months)
    public string AgreementId { get; set; }     // BSS contract ID
    public decimal MSISDNPrice { get; set; }    // MSISDN price tier
}
8.3 Create Order in BSS/CRM
File: BssApiHelper.cs
<!-- SOAP Request for Postpaid -->
<ord:Account>
    <ord:NewAccount>
        <!-- ════════════════════════════════════ -->
        <!-- PaymentType = 1 for Postpaid         -->
        <!-- ════════════════════════════════════ -->
        <ord:PaymentType>1</ord:PaymentType>
        
        <!-- Bill cycle 28 days for Postpaid -->
        <ord:BillCycleType>28</ord:BillCycleType>
        
        <!-- ════════════════════════════════════ -->
        <!-- Credit Limit - Postpaid Exclusive    -->
        <!-- ════════════════════════════════════ -->
        <ord:CreditLimit>{plan.CreditLimit * 10000}</ord:CreditLimit>
        
        <!-- Bill Medium: SMS + Email -->
        <ord:BillMedium>
            <com:BillMediumCode>2</com:BillMediumCode>  <!-- SMS -->
        </ord:BillMedium>
        <ord:BillMedium>
            <com:BillMediumCode>3</com:BillMediumCode>  <!-- Email -->
        </ord:BillMedium>
    </ord:NewAccount>
</ord:Account>

<!-- ════════════════════════════════════════════ -->
<!-- Contract/Commitment - Postpaid Exclusive     -->
<!-- ════════════════════════════════════════════ -->
<com:Contract>
    <com:AgreementId>{matrix.AgreementId}</com:AgreementId>
    <com:DurationUnit>M</com:DurationUnit>           <!-- Month -->
    <com:DurationValue>{matrix.Duration}</com:DurationValue>
</com:Contract>

<!-- eSIM Flag -->
<ord:EsimFlag>{(order.IsESim == 1) ? "Y" : "N"}</ord:EsimFlag>
5️⃣ Cost Calculation for Postpaid
┌────────────────────────────────────────────────────────────────────┐
│                    COST CALCULATION - POSTPAID                     │
├────────────────────────────────────────────────────────────────────┤
│                                                                    │
│  MSISDN Cost              × 1.15 (VAT)  = MSISDNCost ✅            │
│  + Plan Cost              NO VAT         = PlanCost ❌              │
│  + Addons Cost            NO VAT         = AddonsCost ❌            │
│  + ExtraSIM Cost          NO VAT         = ExtraSIMCost ❌          │
│  ────────────────────────────────────────────────────────────────  │
│  = Total (OrderTotal)                                              │
│                                                                    │
│  ⚠️ Note: 15% VAT is calculated on monthly invoice, not at sale    │
└────────────────────────────────────────────────────────────────────┘
Code:
const decimal VAT = 0.15m;

// MSISDN - WITH VAT
ord.MSISDNCost = MSISDNCost * (1 + VAT);

// Plan - NO VAT for Postpaid
ord.PlanCost = (ord.IsPostpaid == 1) ? 
    plan.Price :                    // NO VAT
    plan.Price * (1 + VAT);         // WITH VAT

// Total
ord.OrderTotal = ord.MSISDNCost + ord.PlanCost + ord.AddonsCost + ord.ExtraSIMCost;
6️⃣ eContract Generation
File: SalesAPIController.cs:1683-1684
// VAT text in contract
sb.Replace("{VatPlaceholder}", 
    order.IsPostpaid == 1 ? "(exclusive of VAT)" : "Including VAT");

sb.Replace("{VatStatement}", 
    order.IsPostpaid == 1 ? 
        "Prices are not inclusive of VAT. VAT will be calculated in each invoice cycle." :
        "Prices are inclusive of VAT.");
7️⃣ Confirmation SMS
File: SalesAPIController.cs:1483-1490
// Send confirmation SMS after 2 minutes delay
Task.Delay(120000).ContinueWith((task) =>
{
    if (ord.IsMNP == ActivationType.PostpaidDataSIM)
        tcc.SendBallighny(ord.IdNumber, confirmationSMS);  // Ballighny
    else
        sms.SendSMS("0" + ord.MSISDN, confirmationSMS);    // Standard SMS
});
8️⃣ Complete Comparison Table
Aspect	Postpaid	Prepaid
IsMNP	3, 4, 5	0, 1, 2
SubType	1	0
IsPostpaid	1	0
Allowed Sellers	Retail only	Retail + POS
VAT on Plan	❌ (per invoice)	✅ 15%
VAT on Addons	❌ (per invoice)	✅ 15%
ExtraSIM	✅ Available	❌ Not available
Credit Limit	✅ Set per plan	❌ N/A
Bill Cycle	28 days	15 days
Contract	✅ Required	❌ None
Promissory Note	✅ Nafith	❌ N/A
OTP Delivery	Ballighny	Standard SMS
Bill Medium	SMS + Email	SMS only
9️⃣ ExtraSIM - Physical vs eSIM
// For each ExtraSIM in the order
foreach (var sim in extraSIM)
{
    // Get ICCID
    sim.ICCID = (sim.IsESIM == 0) ? 
        ordExtraSIM[i].ICCID :           // Physical: from user input
        bss.PickESim(seller, ord.Id);    // eSIM: system picks
}
🔟 External System Integrations
System	File	Purpose
BSS/CRM	BssApiHelper.cs	Order creation, SIM management, Credit Limit
TCC	TCCApiHelper.cs	Identity registration, Ballighny SMS
Nafith	NafithHelper.cs	Promissory notes (Sanad)
SMS Gateway	SMSHelper.cs	Standard SMS notifications
T
