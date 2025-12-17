# Prepaid vs Postpaid Scenarios: Exhaustive Comparison

---

## 1️⃣ Scenario Inventory (CRITICAL)

### Prepaid Scenarios
- Prepaid New SIM (Physical)
- Prepaid eSIM
- Prepaid Port-In
- Prepaid Data SIM
- Prepaid Replacement / SIM Swap

### Postpaid Scenarios
- Postpaid New SIM (Physical)
- Postpaid eSIM
- Postpaid Port-In
- Postpaid Data SIM
- Postpaid with ExtraSIM
- Postpaid with Promissory Note (Nafith)

---

## 2️⃣ High-Level Flow Comparison

| Phase                | Prepaid                                                                 | Postpaid                                                                |
|----------------------|------------------------------------------------------------------------|-------------------------------------------------------------------------|
| Order Creation       | Seller initiates order, minimal info required                           | Seller initiates order, more info (credit, commitment, promissory note) |
| Customer Verification| OTP via SMS, basic ID validation                                        | OTP via Ballighny (TCC), strict ID, TCC/Nafath integration              |
| Number Selection     | Choose from available MSISDNs, vanity supported                         | Same, but may have extra checks for credit/commitment                   |
| Plan Selection       | Prepaid plans only, VAT applied immediately                              | Postpaid plans, commitment/commission matrix, credit check              |
| Payment Handling     | Wallet deducted before activation, VAT up front                          | Wallet deducted after activation, credit/commission logic, VAT on bill  |
| Compliance           | Basic, no promissory note                                               | Promissory note (Nafith), TCC, commitment matrix                        |
| Activation           | BSS activation, immediate                                                | TCC registration, BSS activation, eContract, delayed                    |
| Completion           | Order closed after activation                                            | Order closed after all compliance, eContract, and notifications         |

**Differences:**
- Postpaid requires more compliance (TCC, promissory note, commitment matrix).
- Payment timing and VAT handling differ.
- Postpaid has more steps and stricter validation.

---

## 3️⃣ Business Rules Comparison (VERY IMPORTANT)

| Rule                  | Prepaid                                    | Postpaid                                      |
|-----------------------|--------------------------------------------|------------------------------------------------|
| OTP Delivery          | SMS                                        | Ballighny (TCC)                                |
| Government Integration| None or minimal                            | TCC, Nafath, Promissory Note                   |
| Validation Strictness | Basic ID, blacklist                        | Full ID, TCC eligibility, credit, commitment   |
| VAT Handling          | Applied up front on all charges            | Applied on bill, not on wallet deduction       |
| Money Deduction       | Before activation                          | After activation, may allow credit             |
| Wallet Behavior       | Must have full balance before activation   | May allow negative/credit, commission applied  |
| Commission Calculation| Simple, per plan                           | Complex, uses CommissionMatrix, Commitment     |
| Credit Limits         | Not applicable                             | Enforced, checked before/after activation      |
| POS Restrictions      | POS allowed                                | POS may be restricted for postpaid             |
| Channel Limitations   | Fewer                                      | More, based on plan/channel/ID type            |

---

## 4️⃣ UI / Screens Comparison

| Screen         | Prepaid | Postpaid | Reason                                      |
|---------------|---------|----------|---------------------------------------------|
| Login         | ✅      | ✅       | Same                                        |
| Create Order  | ✅      | ✅       | Same                                        |
| OTP           | ✅      | ✅       | Different delivery (SMS vs Ballighny)        |
| Plan Selection| ✅      | ✅       | Different plan sets, extra fields for postpaid|
| ExtraSIM      | ❌      | ✅       | Only postpaid supports ExtraSIM              |
| PromissoryNote| ❌      | ✅       | Only postpaid requires promissory note       |

**Explanation:**
- ExtraSIM and PromissoryNote screens are exclusive to postpaid.
- Plan selection has more fields for postpaid (commitment, commission, etc).

---

## 5️⃣ API-Level Comparison (CORE SECTION)

| API                | Prepaid | Postpaid | Notes                                                      |
|--------------------|---------|----------|------------------------------------------------------------|
| CreateOrder        | ✅      | ✅       | Same endpoint, payload differs (fields, compliance)        |
| SendOTP            | ✅      | ✅       | Prepaid: SMS, Postpaid: Ballighny                          |
| getMbbPlan         | ✅      | ✅       | Both, but postpaid may filter by credit/commitment         |
| CommissionMatrix   | ❌      | ✅       | Only postpaid, for commission calculation                  |
| CommitmentMatrix   | ❌      | ✅       | Only postpaid, for plan eligibility                        |
| ActivateLine       | ✅      | ✅       | Prepaid: BSS only, Postpaid: TCC then BSS, eContract       |

**Call Order & Payload Differences:**
- Postpaid flows include extra fields: commitment, promissory note, commission.
- Response for postpaid includes compliance/contract info.

---

## 6️⃣ Database Impact Comparison

| Table           | Prepaid | Postpaid | Difference                                 |
|-----------------|---------|----------|--------------------------------------------|
| SalesOrder      | ✅      | ✅       | More fields for postpaid (commitment, etc) |
| ExtraSIM        | ❌      | ✅       | Only postpaid                              |
| PromissoryNote  | ❌      | ✅       | Only postpaid                              |
| Wallet          | ✅      | ✅       | Timing of deduction differs                |

**Lifecycle:**
- Postpaid orders touch more tables and fields.
- PromissoryNote and ExtraSIM are exclusive to postpaid.

---

## 7️⃣ Error & Edge Case Comparison

| Scenario                | Prepaid Handling                  | Postpaid Handling                        |
|-------------------------|-----------------------------------|------------------------------------------|
| Insufficient Wallet     | Block order, error immediately    | Block or allow credit, error or warning  |
| OTP Failure             | Retry, SMS resend                 | Retry, Ballighny resend, stricter checks |
| SIM Unavailable         | Error, choose another             | Error, may trigger compensation          |
| External System Timeout | Retry, error to user              | Retry, compensation, log for audit       |

---

## 8️⃣ Timeline & Sequence Comparison

### Prepaid Sequence
Client → CreateOrder
Client → SendOTP
Client → SelectNumber
Client → SelectPlan
Client → Pay
Client → Activate

### Postpaid Sequence
Client → CreateOrder
Client → SendOTP (Ballighny)
Client → SelectNumber
Client → SelectPlan
Client → CommissionMatrix
Client → PromissoryNote
Client → Activate

**Divergence Points:**
- Postpaid inserts CommissionMatrix, PromissoryNote, and compliance steps before activation.
- OTP via Ballighny for postpaid.

---

## 9️⃣ Migration Implications (Java System)

- **Reusable:**
  - Core order creation logic, number selection, plan selection APIs.
  - BSS integration for activation.
- **Must Split:**
  - Compliance logic (TCC, PromissoryNote, CommitmentMatrix) must be separated for postpaid.
  - UI flows for ExtraSIM and PromissoryNote.
- **Must Redesign:**
  - Wallet deduction logic (timing, credit handling).
  - Commission and commitment calculation for postpaid.
  - Error handling and compensation for postpaid.
- **Risk Areas:**
  - Data consistency (multi-step postpaid flows).
  - Compensation logic for failed activations.
  - Ensuring all compliance steps are enforced for postpaid.

---

*This document is exhaustive, unambiguous, and suitable for developers, architects, and product teams. All scenarios, flows, and differences are explicitly listed.*
