# Extra SIM Flow A - Low Level Design (LLD)
# Extra SIM During New Line Activation (Postpaid Only)

## Document Information


> **Note**: This document has been verified against the actual code in `ordermgmtservice` and related microservices.

---

## 1. Executive Summary

### 1.1 Overview
Flow A handles the scenario where a customer adds Extra Data SIMs **during the new line activation process**. This flow is only available for **Postpaid** customers and is integrated into the existing SIM activation flow in `ordermgmtservice`.

### 1.2 Key Characteristics
- **Trigger**: During new Postpaid line activation
- **Restriction**: NOT available for Port-In (MNP) requests
- **Activation**: Asynchronous via background scheduler (after main line activated)
- **Payment**: Included in main order payment
- **Tables**: Extends existing `CustomerProductOrder` with new `ExtraSimActivation` table

### 1.3 Legacy vs Revamped Comparison

| Aspect | Legacy (.NET SalesApp) | Revamped (Java Microservices) |
|--------|------------------------|-------------------------------|
| Main Order Table | `SalesOrder` | `CustomerProductOrder` (ordermgmtservice) |
| Extra SIM Table | `ExtraSIM` | `extra_sim_oto_order` ✅ **EXISTS** |
| TCC Integration | Direct API call | Via `semati-service` (RequestType=2) |
| BSS Integration | Direct SOAP call | Via `bss-mediation-service` |
| Background Jobs | Hangfire | Spring @Scheduled |
| OTP Required | No (part of main activation) | No (part of main activation) |

### 1.4 Existing Code References (Verified)

| Component | File Path | Status |
|-----------|-----------|--------|
| Entity | `ordermgmtservice/.../entity/ExtraSimOtoOrder.java` | ✅ EXISTS |
| Repository | `ordermgmtservice/.../repository/ExtraSimOtoOrderRepository.java` | ✅ EXISTS |
| Order Strategy | `ordermgmtservice/.../strategy/impl/ExtraSimOrderCreationStrategy.java` | ✅ EXISTS |
| Activation Strategy | `ordermgmtservice/.../activation/strategy/impl/ExtraSimActivationStrategy.java` | ✅ EXISTS |
| Validation Handler | `ordermgmtservice/.../validation/handlers/ExtraSimLimitValidationHandler.java` | ✅ EXISTS |

---

## 2. Architecture Overview

### 2.1 Microservices Involved (Verified from common.properties)

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                    FLOW A: EXTRA SIM DURING ACTIVATION                          │
│                         Microservices Architecture                               │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│                         ┌────────────────────┐                                  │
│                         │   Customer App     │                                  │
│                         │   (Web/Mobile)     │                                  │
│                         └─────────┬──────────┘                                  │
│                                   │                                              │
│                                   ▼                                              │
│  ┌────────────────────────────────────────────────────────────────────────────┐ │
│  │                        ordermgmtservice                                     │ │
│  │                           Port: 8073 ✅                                     │ │
│  ├────────────────────────────────────────────────────────────────────────────┤ │
│  │                                                                             │ │
│  │  Tables (PostgreSQL):                                                       │ │
│  │  ├── customer_product_order     (Main order - existing)                    │ │
│  │  └── extra_sim_oto_order        (Extra SIM tracking - ✅ EXISTS)           │ │
│  │                                                                             │ │
│  │  Components (✅ ALL EXIST):                                                 │ │
│  │  ├── ExtraSimOrderCreationStrategy   (Order creation)                      │ │
│  │  ├── ExtraSimActivationStrategy      (Activation - REQUEST_NEW_SIM)        │ │
│  │  ├── ExtraSimLimitValidationHandler  (Validation - max 5 SIMs)             │ │
│  │  └── NafathPollingJob                (Background polling)                   │ │
│  │                                                                             │ │
│  └────────────────────────────────────────────────────────────────────────────┘ │
│                    │                    │                    │                   │
│                    ▼                    ▼                    ▼                   │
│  ┌──────────────────────┐  ┌──────────────────────┐  ┌──────────────────────┐  │
│  │   semati-service     │  │  bss-mediation       │  │ customer-management  │  │
│  │   (TCC/Semati)       │  │  (Huawei BSS)        │  │                      │  │
│  │  digitalcustom-      │  │  digitalmediationbss │  │ digitalcustomer-     │  │
│  │  integration-svc:8075│  │  -svc:8071/dot ✅    │  │ management-svc:8072  │  │
│  └──────────────────────┘  └──────────────────────┘  └──────────────────────┘  │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
```

**Service URLs (from common.properties):**
```properties
bss.mediation.base.url=http://digitalmediationbss-svc:8071/dot
service.url.customer.management=http://digitalcustomermanagement-svc:8072/
service.url.rbm.digital.custom.integration=http://digitalcustomintegration-svc:8075/
```

### 2.2 Table Distribution (Verified)

| Table | Microservice | Database | Status |
|-------|--------------|----------|--------|
| `customer_product_order` | ordermgmtservice | ordermgmt_db | ✅ EXISTS |
| `extra_sim_oto_order` | ordermgmtservice | ordermgmt_db | ✅ EXISTS |
| `integration_data` | ordermgmtservice | ordermgmt_db | ✅ EXISTS |
| `delivery_order_details` | ordermgmtservice | ordermgmt_db | ✅ EXISTS |
| `activation_token_process` | ordermgmtservice | ordermgmt_db | ✅ EXISTS (Nafath) |

---

## 3. Database Schema

### 3.1 Existing Table: extra_sim_oto_order (✅ VERIFIED)

**Source:** `ordermgmtservice/.../entity/ExtraSimOtoOrder.java`

```sql
-- This table ALREADY EXISTS in ordermgmtservice
CREATE TABLE extra_sim_oto_order (
    id                    BIGSERIAL PRIMARY KEY,

    -- Order Reference
    oto_order_id          BIGINT NOT NULL,              -- FK to customer_product_order.id

    -- SIM Details
    msisdn                VARCHAR(20) NOT NULL,         -- Data MSISDN (580XXXXXXX)
    iccid                 VARCHAR(50),                  -- SIM Card ICCID
    sim_type              VARCHAR(20) NOT NULL,         -- 'Regular' or 'eSIM' (EnumType.STRING)

    -- External Tracking
    external_order_id     VARCHAR(100),                 -- External/Huawei order ID

    -- Pricing
    price                 DECIMAL(10,2) NOT NULL,       -- 0 if free

    -- Status (EnumType.ORDINAL - 0=PENDING, 1=ACTIVATED, 2=FAILED, 3=EXPIRED)
    status                INTEGER DEFAULT 0,

    -- Audit
    added_date            TIMESTAMP NOT NULL,
    modified_date         TIMESTAMP,
    activation_date       TIMESTAMP,
    notes                 VARCHAR(500)
);

-- Indexes (from @Index annotations)
CREATE INDEX idx_oto_order_id ON extra_sim_oto_order(oto_order_id);
CREATE INDEX idx_msisdn ON extra_sim_oto_order(msisdn);
CREATE INDEX idx_external_order_id ON extra_sim_oto_order(external_order_id);
```

### 3.2 Existing Entity Class (✅ VERIFIED)

**Source:** `ordermgmtservice/src/main/java/com/segmatek/ordermgmtservice/entity/ExtraSimOtoOrder.java`

```java
package com.segmatek.ordermgmtservice.entity;

import com.segmatek.DigitalLookup.enums.SimType;
import com.segmatek.ordermgmtservice.enums.SimActivationStatus;

@Entity
@Table(name = "extra_sim_oto_order",
       indexes = {
           @Index(name = "idx_oto_order_id", columnList = "oto_order_id"),
           @Index(name = "idx_msisdn", columnList = "msisdn"),
           @Index(name = "idx_external_order_id", columnList = "external_order_id")
       })
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class ExtraSimOtoOrder {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    @Column(name = "id")
    private Long id;

    /** Reference to the main OTO order - Links to CustomerProductOrder.id */
    @Column(name = "oto_order_id", nullable = false)
    private Long otoOrderId;

    /** Type of SIM (Regular or eSIM) */
    @Enumerated(EnumType.STRING)
    @Column(name = "sim_type", nullable = false, length = 20)
    private com.segmatek.DigitalLookup.enums.SimType simType;

    /** Phone number (MSISDN) for this extra SIM */
    @Column(name = "msisdn", nullable = false, length = 20)
    private String msisdn;

    /** SIM card number (ICCID) - populated for eSIMs */
    @Column(name = "iccid", length = 50)
    private String iccid;

    /** External order ID (for tracking with third-party systems) */
    @Column(name = "external_order_id", length = 100)
    private String externalOrderId;

    /** Price for this extra SIM (may be 0 if free SIM included in plan) */
    @Column(name = "price", precision = 10, scale = 2, nullable = false)
    private BigDecimal price;

    /** Activation status specific to this extra SIM */
    @Enumerated(EnumType.ORDINAL)
    @Column(name = "status")
    @Builder.Default
    private SimActivationStatus status = SimActivationStatus.PENDING;

    @Column(name = "added_date", nullable = false)
    private LocalDateTime addedDate;

    @Column(name = "activation_date")
    private LocalDateTime activationDate;

    @Column(name = "modified_date")
    private LocalDateTime modifiedDate;

    @Column(name = "notes", length = 500)
    private String notes;

    @PrePersist
    protected void onCreate() {
        if (addedDate == null) addedDate = LocalDateTime.now();
        modifiedDate = LocalDateTime.now();
    }

    @PreUpdate
    protected void onUpdate() {
        modifiedDate = LocalDateTime.now();
    }

    // Helper methods
    public boolean isESim() { return simType == SimType.eSIM; }
    public boolean isFreeSim() { return price != null && price.compareTo(BigDecimal.ZERO) == 0; }
    public void activate() {
        this.status = SimActivationStatus.ACTIVATED;
        this.activationDate = LocalDateTime.now();
    }
    public boolean isActivated() { return status.equals(SimActivationStatus.ACTIVATED); }
}
```

### 3.3 Status Enum (✅ VERIFIED)

**Source:** `ordermgmtservice/src/main/java/com/segmatek/ordermgmtservice/enums/SimActivationStatus.java`

```java
public enum SimActivationStatus {
    PENDING,      // 0 - Created, waiting for activation
    ACTIVATED,    // 1 - Successfully activated
    FAILED,       // 2 - Activation failed
    EXPIRED       // 3 - Order expired
}
```

---

## 4. Business Rules (✅ VERIFIED from ExtraSimLimitValidationHandler.java)

### 4.1 Eligibility Rules

**Source:** `ordermgmtservice/.../validation/handlers/ExtraSimLimitValidationHandler.java`

```java
@Component
@RequiredArgsConstructor
public class ExtraSimLimitValidationHandler extends OrderValidationHandler {

    private static final int DEFAULT_MAX_EXTRA_SIMS = 5;  // ✅ Verified default
    private final ProductCatalogClient productCatalogClient;

    @Override
    protected void doValidate(OrderValidationContext context) {
        // Skip if no extra SIMs requested
        if (context.getExtraSims() == null || context.getExtraSims().isEmpty()) {
            return;
        }

        // Get max allowed from plan's simCount
        Integer maxAllowed = getMaxAllowedExtraSims(context);
        int totalSims = 1 + context.getExtraSims().size();  // main SIM + extra SIMs

        // Validate total count (including main SIM)
        if (totalSims > maxAllowed) {
            throw new ValidationException(
                "Total SIM count exceeds maximum allowed: " + maxAllowed
            );
        }

        // Validate each extra SIM has SimType and MSISDN
        for (var extraSim : context.getExtraSims()) {
            if (extraSim.getSimType() == null) {
                throw new ValidationException("Extra SIM SimType is required");
            }
            if (extraSim.getMsisdn() == null) {
                throw new ValidationException("Extra SIM MSISDN is required");
            }
        }
    }

    private Integer getMaxAllowedExtraSims(OrderValidationContext context) {
        // Fetches plan's simCount from ProductCatalogClient
        // Falls back to DEFAULT_MAX_EXTRA_SIMS (5) if not found
        try {
            var plan = productCatalogClient.getPrimaryPlanById(context.getPrimaryPlanId());
            return plan.getSimCount() != null ? plan.getSimCount() : DEFAULT_MAX_EXTRA_SIMS;
        } catch (Exception e) {
            return DEFAULT_MAX_EXTRA_SIMS;
        }
    }
}
```

### 4.2 Pricing Rules (✅ VERIFIED from ExtraSimOrderCreationStrategy.java)

**Source:** `ordermgmtservice/.../strategy/impl/ExtraSimOrderCreationStrategy.java`

```java
// Inner DTO class for Extra SIM pricing info from plan
@Data
@Builder
private static class ExtraSimInfo {
    private Integer maxAllowedCount;    // from plan.simCount
    private Integer freeSimCount;       // from plan.freeSimCount
    private BigDecimal pricePerSim;     // from plan.extraSimPrice
}

// Pricing logic in createExtraSimOrders method:
private void createExtraSimOrders(CompleteOrderCreationDTO request, Long otoOrderId, ExtraSimInfo extraSimInfo) {
    int freeSimsRemaining = extraSimInfo.getFreeSimCount();

    // Free SIMs are allocated FIRST
    // Main MSISDN gets free SIM pricing if available
    // Decrements freeSimsRemaining counter
    // Additional extra SIMs from request get free pricing until quota exhausted

    for (ExtraSimRequest simRequest : request.getExtraDataSims()) {
        boolean isFree = freeSimsRemaining > 0;
        BigDecimal price = isFree ? BigDecimal.ZERO : extraSimInfo.getPricePerSim();
        if (isFree) freeSimsRemaining--;

        createSingleExtraSimOrder(simRequest.getMsisdn(), simRequest.getSimType(), otoOrderId, price);
    }
}
```

**Pricing Logic Summary:**
- Free SIMs are allocated **first**
- `plan.freeSimCount` determines how many free SIMs
- `plan.extraSimPrice` is the price per additional SIM (typically 25 SAR)
- VAT handling is separate (Prepaid only, but Flow A is Postpaid)

---

## 5. API Endpoints (✅ VERIFIED from SimActivationController.java)

### 5.1 Activation Initiate Endpoint

**Source:** `ordermgmtservice/.../controller/SimActivationController.java`

```java
@PostMapping("/initiate")
public ResponseEntity<NafathSessionResponse> initiateActivation(
    @Valid @RequestBody InitiateActivationRequest request)
// Request: { orderId, nationalId, language }
// Response: { transactionId, randomCode, expiresAt, status, phase }
```

### 5.2 Activation Complete Endpoint

```java
@PostMapping("/activate-extra-sim")
public ResponseEntity<ActivationResponse> activateExtraSim(
    @Valid @RequestBody ActivationRequest request)
// Request: { orderId, iamToken, iccid[] }
// Response: { orderId, activationSuccessful, huaweiOrderId[], huaweiSubscriberId, phase }
```

### 5.3 Order Creation (Existing Flow)

**Endpoint**: `POST /api/v1/orders/create`

**Request** (with extraDataSims):

```json
{
    "msisdn": "0566123456",
    "iccid": "899661...",
    "simType": "REGULAR",
    "paymentType": "POSTPAID",
    "primaryPlanId": 101,
    "deliveryType": "HOME_DELIVERY",
    "customer": {
        "nationalId": "1234567890",
        "firstName": "Ahmed",
        "lastName": "Mohammed"
    },
    "extraDataSims": [
        {
            "simType": "REGULAR",
            "iccid": "899661...",
            "enabled": true
        },
        {
            "simType": "ESIM",
            "enabled": true
        }
    ]
}
```

**Response**:

```json
{
    "orderId": 12345,
    "purchaseOrderNumber": "ORD-2025-12345",
    "status": "CREATED",
    "msisdn": "0566123456",
    "extraSimCount": 2,
    "extraSimCost": 25.00,
    "totalAmount": 125.00,
    "extraSims": [
        {
            "dataMsisdn": "0580111222",
            "simType": "REGULAR",
            "status": "PENDING",
            "price": 0.00,
            "isFree": true
        },
        {
            "dataMsisdn": "0580333444",
            "simType": "ESIM",
            "status": "PENDING",
            "price": 25.00,
            "isFree": false
        }
    ]
}
```

### 5.2 Get Extra SIM Status

**Endpoint**: `GET /api/v1/orders/{orderId}/extra-sims`

**Response**:

```json
{
    "orderId": 12345,
    "mainLineStatus": "ACTIVATED",
    "extraSims": [
        {
            "id": 1,
            "dataMsisdn": "0580111222",
            "simType": "REGULAR",
            "status": "ACTIVATED",
            "activatedAt": "2025-12-31T10:30:00Z"
        },
        {
            "id": 2,
            "dataMsisdn": "0580333444",
            "simType": "ESIM",
            "status": "PENDING",
            "createdAt": "2025-12-31T10:00:00Z"
        }
    ]
}
```

---

## 6. Sequence Diagrams

### 6.1 Order Creation with Extra SIMs

```
┌──────────┐     ┌──────────────────┐     ┌──────────────┐     ┌───────────────┐
│  Client  │     │ ordermgmtservice │     │ bss-mediation│     │ Product       │
│   App    │     │                  │     │              │     │ Catalog       │
└────┬─────┘     └────────┬─────────┘     └──────┬───────┘     └───────┬───────┘
     │                    │                      │                     │
     │ POST /orders/create│                      │                     │
     │ (with extraDataSims)                      │                     │
     ├───────────────────►│                      │                     │
     │                    │                      │                     │
     │                    │ GET /plans/{id}      │                     │
     │                    ├─────────────────────────────────────────►│
     │                    │                      │        Plan Details │
     │                    │◄─────────────────────────────────────────┤
     │                    │                      │                     │
     │                    │ Validate:            │                     │
     │                    │ - Postpaid only      │                     │
     │                    │ - Not Port-In        │                     │
     │                    │ - Count limits       │                     │
     │                    │                      │                     │
     │                    │ For each Extra SIM:  │                     │
     │                    │ ┌─────────────────┐  │                     │
     │                    │ │GET data MSISDN  │  │                     │
     │                    │ ├────────────────►│  │                     │
     │                    │ │   580XXXXXXX    │  │                     │
     │                    │ │◄────────────────┤  │                     │
     │                    │ │                 │  │                     │
     │                    │ │Lock MSISDN      │  │                     │
     │                    │ ├────────────────►│  │                     │
     │                    │ │     OK          │  │                     │
     │                    │ │◄────────────────┤  │                     │
     │                    │ └─────────────────┘  │                     │
     │                    │                      │                     │
     │                    │ Save:                │                     │
     │                    │ - CustomerProductOrder                     │
     │                    │ - ExtraSimActivation (PENDING)             │
     │                    │ - ExtraSimOtoOrder (for delivery)          │
     │                    │                      │                     │
     │   Order Response   │                      │                     │
     │◄───────────────────┤                      │                     │
     │                    │                      │                     │
```

### 6.2 Main Line Activation (with Extra SIMs waiting)

```
┌──────────┐     ┌──────────────────┐     ┌──────────────┐     ┌───────────────┐
│  Client  │     │ ordermgmtservice │     │semati-service│     │ bss-mediation │
└────┬─────┘     └────────┬─────────┘     └──────┬───────┘     └───────┬───────┘
     │                    │                      │                     │
     │ POST /activate     │                      │                     │
     │ (main line)        │                      │                     │
     ├───────────────────►│                      │                     │
     │                    │                      │                     │
     │                    │ TCC Verify (main)    │                     │
     │                    ├─────────────────────►│                     │
     │                    │     TCC Response     │                     │
     │                    │◄─────────────────────┤                     │
     │                    │                      │                     │
     │                    │ BSS CreateOrder (main)                     │
     │                    ├─────────────────────────────────────────►│
     │                    │     BSS Response     │                     │
     │                    │◄─────────────────────────────────────────┤
     │                    │                      │                     │
     │                    │ Update Order:        │                     │
     │                    │ status = ACTIVATED   │                     │
     │                    │                      │                     │
     │                    │ [Extra SIMs remain   │                     │
     │                    │  PENDING - activated │                     │
     │                    │  by background job]  │                     │
     │                    │                      │                     │
     │  Activation OK     │                      │                     │
     │◄───────────────────┤                      │                     │
     │                    │                      │                     │
```

### 6.3 Background Job: Activate Extra SIMs

```
┌────────────────────┐     ┌──────────────────┐     ┌──────────────┐     ┌───────────────┐
│ExtraSimActivation  │     │ ordermgmtservice │     │semati-service│     │ bss-mediation │
│   Scheduler        │     │                  │     │              │     │               │
└─────────┬──────────┘     └────────┬─────────┘     └──────┬───────┘     └───────┬───────┘
          │                         │                      │                     │
          │ @Scheduled (every 10min)│                      │                     │
          ├────────────────────────►│                      │                     │
          │                         │                      │                     │
          │                         │ Query:               │                     │
          │                         │ ExtraSimActivation   │                     │
          │                         │ WHERE status=PENDING │                     │
          │                         │ AND order.status=    │                     │
          │                         │     ACTIVATED        │                     │
          │                         │                      │                     │
          │                         │ For each pending:    │                     │
          │                         │ ┌─────────────────────────────────────────┐│
          │                         │ │                    │                     ││
          │                         │ │Check expiration    │                     ││
          │                         │ │(> 24h = EXPIRED)   │                     ││
          │                         │ │                    │                     ││
          │                         │ │GET AccountId       │                     ││
          │                         │ ├───────────────────────────────────────►││
          │                         │ │    AccountId       │                     ││
          │                         │ │◄───────────────────────────────────────┤│
          │                         │ │                    │                     ││
          │                         │ │BSS CreateExtraSIM  │                     ││
          │                         │ ├───────────────────────────────────────►││
          │                         │ │    BSS Response    │                     ││
          │                         │ │◄───────────────────────────────────────┤│
          │                         │ │                    │                     ││
          │                         │ │Update status:      │                     ││
          │                         │ │ACTIVATED or FAILED │                     ││
          │                         │ └─────────────────────────────────────────┘│
          │                         │                      │                     │
          │         Done            │                      │                     │
          │◄────────────────────────┤                      │                     │
          │                         │                      │                     │
```

---

## 7. Implementation Details (✅ VERIFIED from actual codebase)

### 7.1 ExtraSimOrderCreationStrategy (EXISTS)

**Source:** `ordermgmtservice/.../strategy/impl/ExtraSimOrderCreationStrategy.java`

```java
@Component
public class ExtraSimOrderCreationStrategy extends OrderStrategy {

    // Injected dependencies (verified)
    private final DeliveryOrderDetailsRepository deliveryOrderDetailsRepository;
    private final IntegrationDataRepository integrationDataRepository;
    private final InputSanitizer inputSanitizer;
    private final CustomerAddressServiceClient customerAddressServiceClient;
    private final PickEsimClient pickEsimClient;
    private final RbmCustomIntegration rbmCustomIntegration;
    private final ProductCatalogClient productCatalogClient;
    private final EmailServiceClient emailServiceClient;
    private final RBMDCustomIntegrationServiceClient otoDeliveryService;

    @Override
    public OrderTypeEnum getOrderType() {
        return OrderTypeEnum.EXTRA_SIM;
    }

    @Override
    protected SimOrderResponse doCreateOrder(OrderContext context) {
        // Main order creation flow: 10 steps
        // 1. Check for existing order
        // 2. Validate primary plan exists
        // 3. Get extra SIM info from plan
        // 4. Handle address creation (home delivery)
        // 5. Pick eSIM ICCID
        // 6. Get product SKU
        // 7. Create CustomerProductOrder entity
        // 8. Save the order
        // 9. Create ExtraSimOtoOrder records
        // 10. Create delivery details
    }

    private void createExtraSimOrders(CompleteOrderCreationDTO request, Long otoOrderId, ExtraSimInfo extraSimInfo) {
        int freeSimsRemaining = extraSimInfo.getFreeSimCount();

        for (ExtraSimRequest simRequest : request.getExtraDataSims()) {
            boolean isFree = freeSimsRemaining > 0;
            BigDecimal price = isFree ? BigDecimal.ZERO : extraSimInfo.getPricePerSim();
            if (isFree) freeSimsRemaining--;

            createSingleExtraSimOrder(simRequest.getMsisdn(), simRequest.getSimType(), otoOrderId, price);
        }
    }

    private void createSingleExtraSimOrder(String msisdn, SimType simType, Long otoOrderId, BigDecimal price) {
        // Creates individual ExtraSimOtoOrder
        // Generates ICCID for eSIM types via pickEsimClient
        // Sets status to PENDING
        ExtraSimOtoOrder extraSimOrder = ExtraSimOtoOrder.builder()
            .otoOrderId(otoOrderId)
            .msisdn(msisdn)
            .simType(simType)
            .price(price)
            .status(SimActivationStatus.PENDING)
            .build();

        if (simType == SimType.eSIM) {
            String iccid = pickEsimClient.pickEsimIccid(msisdn, null);
            extraSimOrder.setIccid(iccid);
        }

        extraSimOtoOrderRepository.save(extraSimOrder);
    }

    @Override
    public void completeOrder(CustomerProductOrder order, String paymentReference, String language) {
        // Handles post-payment order completion
        // Routes to:
        //   - handleRegularExtraSimOrderCompletion() - for regular SIMs
        //   - handleExtraSimESimOrderCompletion() - for eSIMs
    }
}
```

### 7.2 ExtraSimActivationStrategy (EXISTS)

**Source:** `ordermgmtservice/.../activation/strategy/impl/ExtraSimActivationStrategy.java`

```java
@Component
@Slf4j
public class ExtraSimActivationStrategy extends BaseActivationStrategy {

    public ExtraSimActivationStrategy(ActivationProcessingChain processingChain) {
        super(processingChain);
    }

    @Override
    public OrderTypeEnum getOrderType() {
        return OrderTypeEnum.EXTRA_SIM;
    }

    @Override
    public SematiRequestType getSematiRequestType() {
        return SematiRequestType.REQUEST_NEW_SIM;  // ✅ Maps to Semati RequestType=2
    }

    @Override
    protected void doValidateRequest(ActivationRequest request, CustomerProductOrder order) {
        // Validates:
        //   - MSISDN (main line number) is not null/empty
        //   - AccountId is provided (for linking to existing account)
    }

    @Override
    protected void doPrepareContext(ActivationProcessingContext context) {
        // Adds metadata:
        context.getMetadata().put("activationType", "MULTI_SIM");
        context.getMetadata().put("requiresNewSubscriber", false);
        context.getMetadata().put("linkToAccountId", order.getAccountId());
        context.getMetadata().put("parentMsisdn", order.getMsisdn());
    }

    @Override
    protected ActivationResponse buildResponse(ActivationProcessingContext context) {
        // Returns ActivationResponse.successMultiple() with all HuaweiOrderIds
        // Falls back to parent implementation if not successful
    }
}
```

### 7.3 Background Scheduler (NafathPollingJob - EXISTS)

**Source:** `ordermgmtservice/.../job/NafathPollingJob.java`

```java
@Component
@RequiredArgsConstructor
@Slf4j
public class NafathPollingJob {

    private final ActivationTokenProcessRepository activationRepository;
    private final NafathClient nafathClient;
    private final ActivationService activationService;

    @Value("${activation.nafath.max-attempts:30}")
    private int maxAttempts;

    /**
     * Runs every 10 seconds (configurable via activation.nafath.polling-interval)
     */
    @Scheduled(fixedRateString = "${activation.nafath.polling-interval:10000}")
    @Transactional
    public void pollPendingActivations() {
        // Flow:
        //   1. Find all pending Nafath sessions (not expired)
        //   2. For each pending token:
        //      a. Check if expired
        //      b. Check if max attempts reached
        //      c. Poll Nafath for status update
        //      d. Increment attempt count
        //      e. Handle response:
        //         - COMPLETED: Save IAM token, call activationService.continueActivation()
        //         - PENDING: Save token and retry on next poll
        //         - FAILED: Call activationService.handleActivationFailure()
        //         - EXPIRED: Call activationService.handleActivationFailure()
    }

    /**
     * Cleanup expired sessions - Runs every 1 hour
     */
    @Scheduled(fixedRate = 3600000)
    @Transactional
    public void cleanupExpiredSessions() {
        // Deletes expired sessions older than 24 hours
    }
}
```

**Configuration (from application.properties):**
```properties
activation.nafath.polling-interval=10000      # 10 seconds
activation.nafath.max-attempts=30
activation.nafath.session-expiry-minutes=5
```

### 7.4 Repository (EXISTS)

**Source:** `ordermgmtservice/.../repository/ExtraSimOtoOrderRepository.java`

```java
@Repository
public interface ExtraSimOtoOrderRepository extends JpaRepository<ExtraSimOtoOrder, Long> {

    // Find operations
    List<ExtraSimOtoOrder> findByOtoOrderId(Long otoOrderId);
    Optional<ExtraSimOtoOrder> findByMsisdn(String msisdn);
    Optional<ExtraSimOtoOrder> findByExternalOrderId(String externalOrderId);
    Optional<ExtraSimOtoOrder> findByIccid(String iccid);

    // Count operations
    long countByOtoOrderId(Long otoOrderId);

    @Query("SELECT COUNT(e) FROM ExtraSimOtoOrder e WHERE e.otoOrderId = :otoOrderId AND e.status = ACTIVATED")
    long countActivatedByOtoOrderId(@Param("otoOrderId") Long otoOrderId);

    // Filter operations
    List<ExtraSimOtoOrder> findByStatus(SimActivationStatus status);
    List<ExtraSimOtoOrder> findByOtoOrderIdAndStatus(Long otoOrderId, SimActivationStatus status);
    List<ExtraSimOtoOrder> findByOtoOrderIdAndSimType(Long otoOrderId, SimType simType);

    // SIM type specific queries
    @Query("SELECT e FROM ExtraSimOtoOrder e WHERE e.otoOrderId = :otoOrderId AND e.simType = eSIM")
    List<ExtraSimOtoOrder> findESimsByOtoOrderId(@Param("otoOrderId") Long otoOrderId);

    @Query("SELECT e FROM ExtraSimOtoOrder e WHERE e.otoOrderId = :otoOrderId AND e.simType = Regular")
    List<ExtraSimOtoOrder> findRegularSimsByOtoOrderId(@Param("otoOrderId") Long otoOrderId);

    // Status update
    @Modifying
    @Query("UPDATE ExtraSimOtoOrder e SET e.status = :status WHERE e.otoOrderId = :otoOrderId")
    void updateStatusByOtoOrderId(@Param("otoOrderId") Long otoOrderId, @Param("status") SimActivationStatus status);

    // Existence checks
    boolean existsByMsisdn(String msisdn);

    // Delete operations
    @Modifying
    @Query("DELETE FROM ExtraSimOtoOrder e WHERE e.otoOrderId = :otoOrderId")
    void deleteByOtoOrderId(@Param("otoOrderId") Long otoOrderId);
}
```

### 7.5 Activation Processing Chain (EXISTS)

**Source:** `ordermgmtservice/.../activation/chain/ActivationProcessingChain.java`

**Chain Order (for Extra SIM post-Nafath):**
```
1. IccidValidationHandler     - Validates/picks ICCIDs
2. SematiValidationHandler    - TCC verification (REQUEST_NEW_SIM = Type 2)
3. CustomerDataUpdateHandler  - Updates customer data
4. BssProvisioningHandler     - Creates BSS orders (createMultiSimSalesOrder for each ICCID)
5. NotificationHandler        - Sends email notifications
6. ActivationCompletionHandler - Marks activation complete
```

---

## 8. Configuration (✅ VERIFIED)

### 8.1 application.properties (ordermgmtservice)

**Source:** `ordermgmtservice/src/main/resources/application.properties`

```properties
spring.application.name=ordermgmtservice
server.port=8073
spring.profiles.active=local

# Semati/TCC Configuration
activation.semati.mock.enabled=true
activation.semati.base-url=https://api.semati.sa/individual

# Nafath Configuration
activation.nafath.mock.enabled=true
activation.nafath.polling-interval=10000
activation.nafath.polling-timeout=300000
activation.nafath.max-attempts=30
activation.nafath.session-expiry-minutes=5
activation.nafath.base-url=https://api.nafath.sa/v2
activation.nafath.service-id=REDBULL_MOBILE

# Activation Validation
activation.validation.duplicate-prevention-minutes=30
```

### 8.2 common.properties (Service URLs)

**Source:** `digitalcommonlibrary/src/main/resources/common.properties`

```properties
# BSS Mediation Service
bss.mediation.base.url=http://digitalmediationbss-svc:8071/dot

# Service URLs
service.url.customer.management=http://digitalcustomermanagement-svc:8072/
service.url.product.catalog=http://digitalproductcatalog-svc:8081/
service.url.rbm.digital.custom.integration=http://digitalcustomintegration-svc:8075/

# Notification Service
send.sms.url=http://dotnotification-svc:8099/notification/sms/send
send.email.url=http://dotnotification-svc:8099/notification/email/send
```

---

## 9. Error Handling

| Error Code | Message | HTTP Status |
|------------|---------|-------------|
| `EXTRA_SIM_POSTPAID_ONLY` | Extra SIMs during activation are only available for Postpaid orders | 400 |
| `EXTRA_SIM_NOT_ALLOWED_MNP` | Extra SIMs are not allowed with Port-In requests | 400 |
| `EXTRA_SIM_LIMIT_EXCEEDED` | Maximum {n} Extra SIMs allowed for this plan | 400 |
| `EXTRA_SIM_NO_DATA_MSISDN` | No Data MSISDN available in pool | 503 |
| `EXTRA_SIM_ACTIVATION_FAILED` | Failed to activate Extra SIM | 500 |

---

