# Low Level Design (LLD): Transfer Ownership Use Case

##1. Use Case Overview
**Name:** Transfer Ownership

**Description:**
Allows an agent (seller) to transfer the ownership of a mobile line (MSISDN) from one customer to another, updating all related records and triggering required validations, notifications, and integrations.

**Trigger:**
HTTP POST to `/api/Sales/TransferOwnership` (method: `TransferOwnership` in `SalesAPIController`)

**Primary Actor:**
Agent (Seller)

---

##2. Preconditions
- The request must include a valid JWT token (authenticated agent).
- The request body must contain a valid `Id` for an existing `TransferOwnership` record.
- The agent must have permission to perform the operation.

---

##3. Input Parameters
- `Id` (int): TransferOwnership record ID
- `IAMAppToken` (string, optional): Token for identity verification
- (Other fields are loaded from the DB by ID)

---

##4. Main Flow
1. **Retrieve User and Seller:**
 - Get the current user from the JWT token.
 - Retrieve the seller record from the database using the user's ID.
2. **Load TransferOwnership Record:**
 - Find the `TransferOwnership` record by the provided `Id`.
 - Update `IAMAppToken` if provided.
3. **Load Related Data:**
 - Load related `SalesPartner`, `SalesChannel`, `City`, and `Province` for region and channel info.
4. **Validate Location:**
 - Call `VaildateLocation` with order latitude/longitude and seller ID.
 - If invalid, return error.
5. **Call TCC API:**
 - Prepare region and channel info.
 - Call `tcc.TransferOwnership` with user, seller, transferOwnership, region, and channel.
 - Parse TCC response.
6. **Update Names:**
 - If TCC response is successful, update `FirstName` and `LastName` from TCC response.
7. **Call BSS API:**
 - Call `bss.TransferOwnership` with transferOwnership and seller.
 - Update `BSSRequest` and `BSSResponse` fields.
 - If BSS fails, set status and return error.
8. **Change Primary Offer (if needed):**
 - If ID type is not Citizen/Resident, call `bss.ChangePrimaryOffer` with MSISDN and offer ID.
9. **Update Status:**
 - Set `Status =1` (completed) and save changes.
10. **Return Success:**
 - Return the updated transferOwnership object.

---

##5. Alternative/Error Flows
- If the transferOwnership record is not found, return error.
- If location validation fails, return error.
- If TCC API returns error, update status and return error.
- If BSS API returns error, update status and return error.
- All exceptions are caught, logged, and returned as error responses.

---

##6. State Changes
- `TransferOwnership.Status`:0 →1 (on success), or remains unchanged on failure.
- `TransferOwnership.FirstName`, `LastName`: updated from TCC response.
- `TransferOwnership.BSSRequest`, `BSSResponse`: updated from BSS API.
- `TransferOwnership.IAMAppToken`: updated if provided.

---

##7. Notifications/Side Effects
- If primary offer is changed, BSS API is called.
- No SMS or email is sent in this flow (per code).
- All errors are logged using Serilog.

---

##8. Database Tables Involved
- `TransferOwnership`
- `Seller`
- `SalesPartner`
- `SalesChannel`
- `City`
- `Province`

---

##9. External Integrations
- **TCC API:** `tcc.TransferOwnership` (ownership validation and update)
- **BSS API:** `bss.TransferOwnership`, `bss.ChangePrimaryOffer` (update in BSS system)

---

##10. Security
- Requires valid JWT token (checked by `[Authorize]` attribute)
- No direct role/permission check in method, but user is loaded and must exist

---

##11. Logging
- All errors and important steps are logged using Serilog with context

---

##12. Endpoint
- `POST /api/Sales/TransferOwnership`
- Controller: `SalesAPIController`
- Method: `TransferOwnership`

---

##13. Sequence Diagram (Textual)
1. Agent → API: POST TransferOwnership
2. API → DB: Load user, seller, transferOwnership
3. API → TCC: TransferOwnership
4. API → BSS: TransferOwnership
5. API → DB: Update transferOwnership
6. API → Agent: Return result
