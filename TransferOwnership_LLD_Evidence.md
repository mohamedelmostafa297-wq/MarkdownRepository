# Transfer Ownership LLD — Code Evidence

This document provides direct code evidence for each point in the LLD for the Transfer Ownership use case, based strictly on the current codebase.

---

##1. Use Case Overview
- **Controller:** `SalesAPIController`
- **Method:** `public async Task<IActionResult> TransferOwnership([FromServices] IUserRetrieveService userRetriever, dynamic req)`
- **File:** `RedBullSalesPortal.Web/Modules/SalesAPI/SalesAPIController.cs`

---

##2. Preconditions
- **JWT Token:**
 - `[Authorize(AuthenticationSchemes = JwtBearerDefaults.AuthenticationScheme)]` on controller
- **ID in Request:**
 - `TransferOwnership transferOwnership = db.TransferOwnerships.Find((int)req.Id?.Value);`
- **User/Seller Existence:**
 - `UserDefinition user = (UserDefinition)userRetriever.ByUsername(User.FindFirst(ClaimTypes.NameIdentifier)?.Value);`
 - `Seller seller = db.Sellers.Find(user.UserId);`

---

##3. Input Parameters
- `Id` from `req.Id?.Value`
- `IAMAppToken` from `req.IAMAppToken?.Value`

---

##4. Main Flow
- **Retrieve User/Seller:**
 - See above
- **Load TransferOwnership:**
 - `TransferOwnership transferOwnership = db.TransferOwnerships.Find((int)req.Id?.Value);`
 - `transferOwnership.IAMAppToken = req.IAMAppToken?.Value;`
- **Load Related Data:**
 - `SalesPartner partner = db.SalesPartners.Find((int)seller.PartnerId);`
 - `SalesChannel channel = db.SalesChannels.Find(partner.SalesChannel);`
 - `City city = db.Cities.Find(seller.City);`
 - `Province province = db.Provinces.Find(city.Province);`
- **Validate Location:**
 - `var result = VaildateLocation(transferOwnership.OrderLat, transferOwnership.OrderLng, transferOwnership.SellerId);`
- **Call TCC API:**
 - `tcc.TransferOwnership(user, seller, transferOwnership, TccRegion, channel.TCCSourceType);`
- **Update Names:**
 - `transferOwnership.FirstName = (!string.IsNullOrEmpty(tccResult.person.trFirst?.Value)) ? tccResult.person.trFirst : tccResult.person.first;`
 - `transferOwnership.LastName = (!string.IsNullOrEmpty(tccResult.person.trFamily?.Value)) ? tccResult.person.trFamily : tccResult.person.family;`
- **Call BSS API:**
 - `var bssResp = bss.TransferOwnership(transferOwnership, seller);`
 - `transferOwnership.BSSRequest = bssResp.req;`
 - `transferOwnership.BSSResponse = bssResp.resp;`
- **Change Primary Offer:**
 - `if ((transferOwnership.IdType != IdType.Citizen) && (transferOwnership.IdType != IdType.Resident)) bss.ChangePrimaryOffer(transferOwnership.MSISDN, ...);`
- **Update Status:**
 - `transferOwnership.Status =1;`
 - `db.Entry(transferOwnership).State = Microsoft.EntityFrameworkCore.EntityState.Modified;`
 - `db.SaveChanges();`
- **Return Success:**
 - `return Ok(transferOwnership);`

---

##5. Alternative/Error Flows
- **Not found:**
 - `if (transferOwnership == null) return BadRequest(...);`
- **Location validation:**
 - `if (result != null) return result;`
- **TCC/BSS errors:**
 - `if (tccResult.code.Value !=600) ... return BadRequest(...);`
 - `if (long.Parse(bssResp.code) !=0) ... return BadRequest(...);`
- **Exception handling:**
 - `catch (Exception e) { ... return BadRequest(e.Message); }`

---

##6. State Changes
- `transferOwnership.Status` set to1 on success
- `transferOwnership.FirstName`, `LastName` updated from TCC
- `transferOwnership.BSSRequest`, `BSSResponse` updated from BSS
- `transferOwnership.IAMAppToken` updated if provided

---

##7. Notifications/Side Effects
- **ChangePrimaryOffer:**
 - Only called if ID type is not Citizen/Resident
- **No SMS/Email:**
 - No code for notification in this method
- **Logging:**
 - `Log.ForContext("API", "TransferOwnership").Error(...)`

---

##8. Database Tables Involved
- `TransferOwnership` (via db.TransferOwnerships)
- `Seller` (db.Sellers)
- `SalesPartner` (db.SalesPartners)
- `SalesChannel` (db.SalesChannels)
- `City` (db.Cities)
- `Province` (db.Provinces)

---

##9. External Integrations
- **TCC API:**
 - `tcc.TransferOwnership(user, seller, transferOwnership, TccRegion, channel.TCCSourceType);`
- **BSS API:**
 - `bss.TransferOwnership(transferOwnership, seller);`
 - `bss.ChangePrimaryOffer(...)`

---

##10. Security
- `[Authorize]` attribute on controller
- User loaded from JWT

---

##11. Logging
- `Log.ForContext("API", "TransferOwnership").Error(...)`

---

##12. Endpoint
- `POST /api/Sales/TransferOwnership`
- `SalesAPIController.TransferOwnership`

---

##13. Sequence Diagram (Textual)
- All steps are directly implemented as described in the LLD, see method body for exact order.
