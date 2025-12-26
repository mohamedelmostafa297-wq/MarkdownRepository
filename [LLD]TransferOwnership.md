# Transfer Ownership - Table Mapping to Java Microservices

**Document Version:** 1.1 (Verified)
**Created:** December 2025
**Last Verified:** December 2025
**Purpose:** Maps the .NET tables used in Transfer Ownership LLD to their equivalent Java microservices entities
**Verification Status:** All entities verified against actual source code

---

## 1. Overview

This document explains how the database tables used in the **Transfer Ownership** flow (from the legacy .NET `RedBullSalesPortal`) are distributed across the new Java microservices architecture.

### Architecture Transformation Summary

---

## 2. Tables Mapping

### 2.1 TransferOwnership Table

| Property | .NET (Legacy) | Java (New) |
|----------|---------------|------------|
| **Original Table** | `TransferOwnership` | **NOT MIGRATED YET** |
| **Location** | `RedBullSalesPortal` | N/A |
| **Status** | Active in .NET | Pending Migration |

**Analysis:**
- The `TransferOwnership` entity does **NOT exist** in the Java microservices
- This functionality is still handled by the .NET Sales Portal
- The closest equivalent would be in `order-management-service` as a new order type

**Proposed Java Service:** `order-management-service`

**Proposed Entity Structure:**
```java
@Entity
@Table(name = "transfer_ownership")
public class TransferOwnership {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String msisdn;
    private String fromIdNumber;
    private String toIdNumber;
    private String idType;           // Enum: CITIZEN, RESIDENT, VISITOR, etc.
    private Integer nationality;
    private Integer subType;         // Prepaid/Postpaid
    private Integer status;
    private Long sellerId;
    private String iamAppToken;
    private String bssRequest;
    private String bssResponse;
    private String firstName;
    private String lastName;
    private String orderLat;
    private String orderLng;
    private LocalDateTime transDate;
}
```

---

### 2.2 Seller Table

| Property | .NET (Legacy) | Java (New) |
|----------|---------------|------------|
| **Original Table** | `Seller` | **NOT MIGRATED** |
| **Location** | `RedBullSalesPortal` | N/A |
| **Related Java Entity** | - | `User` in `digital-external-omni-auth` |

**Analysis:**
- Seller data in .NET contains sales agent information
- In Java architecture, seller concept is partially covered by:
  - `User` entity in authentication service
  - `BusinessRole` for role-based access

**Java Service:** `digital-external-omni-auth`

**Equivalent Entity Path:**
```
d:\ReposDOTD\digital-external-omni-auth\omni-auth-manager-dataacess\
  src\main\java\com\segmatek\dot\dataaccess\entity\User.java
```

**Actual User Entity Structure (from code):**
```java
@Entity
@Table(name="\"user\"")
public class User extends BaseModel<Long> {
    private String username;
    private String name;
    private String email;
    private String firstName;
    private String lastName;
    private boolean enabled;
    private boolean firstLogin;
    private String title;
    private String academicTitle;
    private String language;
    private String address;
    private String mobilePhoneNumber;
    private String image;
    private String keycloakId;
    private String nationalId;
    private String partyRoleId;
    private String status;
    private LocalDate lastLoginTime;
    private List<String> accountMsisdn;

    @ManyToOne
    private Role role;

    @OneToMany
    List<CustomerEventType> customerEventTypes;
}
```

**Key Differences:**
| .NET Seller Field | Java User Field | Notes |
|-------------------|-----------------|-------|
| `UserId` | `id` (from BaseModel) | Primary key |
| `SellerUserName` | `username` | Username |
| `IdNumber` | `nationalId` | National ID - Direct field |
| `PartnerId` | **NOT MAPPED** | No direct equivalent in User |
| `City` | `address` (string) | Simplified - no relation |
| `TccToken` | **NOT MAPPED** | Moved to external config |
| `DeviceId` | **NOT MAPPED** | Session/Device tracking external |

---

### 2.3 SalesPartner Table

| Property | .NET (Legacy) | Java (New) |
|----------|---------------|------------|
| **Original Table** | `SalesPartner` | **NOT DIRECTLY MIGRATED** |
| **Location** | `RedBullSalesPortal` | N/A |
| **Related Java Entity** | - | `BusinessDomain` or external config |

**Analysis:**
- Sales Partner data defines partner organizations
- In Java, partner configuration is managed through:
  - `BusinessDomain` entity (very simplified)
  - Configuration service settings
  - External partner management system

**Java Service:** `digital-external-omni-auth`

**Equivalent Entity Path:**
```
d:\ReposDOTD\digital-external-omni-auth\omni-auth-manager-dataacess\
  src\main\java\com\segmatek\dot\dataaccess\entity\BusinessDomain.java
```

**Actual BusinessDomain Entity (from code):**
```java
@Entity
public class BusinessDomain {
    @Id
    @GeneratedValue(strategy = GenerationType.AUTO)
    private int id;

    @Column(unique = true)
    private String name;
}
```

> **Note:** BusinessDomain is a very simple entity with only `id` and `name`.
> Most of the SalesPartner data from .NET is NOT migrated to Java and would need
> to be handled by a separate Partner Management service or configuration.

---

### 2.4 SalesChannel Table

| Property | .NET (Legacy) | Java (New) |
|----------|---------------|------------|
| **Original Table** | `SalesChannel` | `SalesChannel` |
| **Location** | `RedBullSalesPortal` | `digital-product-catalog` |
| **Status** | Migrated | **ACTIVE** |

**Java Entity Path:**
```
d:\ReposDOTD\digitalproductcatalog\
  src\main\java\com\segmatek\digitalproduct\entity\SalesChannel.java
```

**Actual Entity Structure (from code):**
```java
@Entity
@Table(name = "SalesChannel")
public class SalesChannel implements Serializable {
    @Id
    @GeneratedValue(strategy = GenerationType.AUTO)
    @Column(name = "id", nullable = false)
    private Long id;

    @Column(name = "Name", nullable = false, length = 255)
    @Enumerated(EnumType.STRING)
    public ProductSalesChannelName name;

    @ManyToMany(mappedBy = "productOfferingPrice", targetEntity = ProductOffering.class, fetch = FetchType.LAZY)
    @Cascade({CascadeType.SAVE_UPDATE, CascadeType.LOCK})
    private List<ProductOffering> productOffering = new ArrayList<>();
}
```

**ProductSalesChannelName Enum (from code):**
```java
// Location: digitallookup/src/main/java/com/segmatek/DigitalLookup/enums/ProductSalesChannelName.java
public enum ProductSalesChannelName {
    CSO("CSO");  // Currently only one value defined

    private String value;
    // ... constructor and methods
}
```

**Key Changes:**
| .NET Field | Java Field | Change |
|------------|------------|--------|
| `Id` | `id` | Same |
| `Name` (string) | `name` (Enum) | Changed to Enum `ProductSalesChannelName` |
| `TCCSourceType` | **NOT IN ENTITY** | Moved to configuration |
| ProductOfferings | `productOffering` | ManyToMany via `productOfferingPrice` |

> **Note:** The enum currently only has `CSO` value. Additional values may need to be added.

---

### 2.5 City Table

| Property | .NET (Legacy) | Java (New) |
|----------|---------------|------------|
| **Original Table** | `City` | `City` |
| **Location** | `RedBullSalesPortal` | `digital-external-omni-auth` |
| **Status** | Migrated | **ACTIVE** |

**Java Entity Path:**
```
d:\ReposDOTD\digital-external-omni-auth\omni-auth-manager-dataacess\
  src\main\java\com\segmatek\dot\dataaccess\entity\address\City.java
```

**Actual Entity Structure (from code):**
```java
@Entity
public class City extends BaseModel<Long> {
    @Column(nullable = false)
    private String name;

    private String description;

    @ManyToOne
    private Governorate governorate;
}
```

**BaseModel Fields (inherited):**
```java
// From: digital-external-omni-auth/.../entity/base/BaseModel.java
@MappedSuperclass
public abstract class BaseModel<ID> implements Serializable {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    protected ID id;

    protected LocalDate createdAt;       // Auto-set
    protected LocalDate lastModifiedAt;  // Auto-set
    protected String createdBy;
    protected String lastModifiedBy;
    protected boolean archived = false;
    protected LocalDate archivingDate;
}
```

**Hierarchy Change:**
```
.NET:                          Java:
City → Province                City → Governorate → Country
      ↓                              ↓
      TCCCode                        (No TCCCode - moved to config)
```

**Complete Geographic Hierarchy in Java:**
```
Country (id, name, description)
  └── Governorate (id, name, description, country)
        └── City (id, name, description, governorate)
              └── District (id, name, description, city)
```

---

### 2.6 Province Table

| Property | .NET (Legacy) | Java (New) |
|----------|---------------|------------|
| **Original Table** | `Province` | `Governorate` |
| **Location** | `RedBullSalesPortal` | `digital-external-omni-auth` |
| **Status** | Renamed & Migrated | **ACTIVE** |

**Java Entity Path:**
```
d:\ReposDOTD\digital-external-omni-auth\omni-auth-manager-dataacess\
  src\main\java\com\segmatek\dot\dataaccess\entity\address\Governorate.java
```

**Actual Entity Structure (from code):**
```java
@Entity
public class Governorate extends BaseModel<Long> {
    @Column(nullable = false)
    private String name;

    private String description;

    @ManyToOne
    private Country country;
}
```

**Country Entity (from code):**
```java
@Entity
public class Country extends BaseModel<Long> {
    @Column(unique = true, nullable = false)
    private String name;

    private String description;
}
```

**District Entity (from code):**
```java
@Entity
public class District extends BaseModel<Long> {
    @Column(nullable = false)
    private String name;

    private String description;

    @ManyToOne
    private City city;  // Note: District is under City, NOT Governorate
}
```

**Key Changes:**
| .NET Province Field | Java Governorate Field | Notes |
|---------------------|------------------------|-------|
| `Id` | `id` (from BaseModel) | Same |
| `Name` | `name` | Same |
| `TCCCode` | **REMOVED** | Moved to region configuration service |
| - | `country` | Added parent relationship |
| - | `description` | Added |

**Geographic Hierarchy (Already shown in City section above)**

---

### 2.7 TccTraces Table

| Property | .NET (Legacy) | Java (New) |
|----------|---------------|------------|
| **Original Table** | `TccTraces` | `IntegrationLog` (generic) |
| **Location** | `RedBullSalesPortal` | `rbmdigitalcustomintegration/common-module` |
| **Status** | Redesigned | **ACTIVE (Generic)** |

**Java Entity Path:**
```
d:\ReposDOTD\rbmdigitalcustomintegration\common-module\
  src\main\java\com\segmatek\common\entity\IntegrationLog.java
```

**Entity Structure:**
```java
@Entity
@Table(name = "integration_logs")
public class IntegrationLog {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(name = "client_name")
    private String clientName;        // "TCC", "BSS", "Semati", etc.

    @Column(name = "method")
    private String method;            // HTTP method

    @Column(name = "url", length = 1000)
    private String url;               // API endpoint

    @Column(name = "request_body", columnDefinition = "TEXT")
    private String requestBody;       // Full request JSON/XML

    @Column(name = "response_status")
    private Integer responseStatus;   // HTTP status code

    @Column(name = "duration_ms")
    private Long durationMs;          // Response time

    @Column(name = "created_at")
    private LocalDateTime createdAt;

    @Column(name = "error_message", columnDefinition = "TEXT")
    private String errorMessage;
}
```

**Key Changes:**
| .NET TccTraces Field | Java IntegrationLog Field | Notes |
|----------------------|---------------------------|-------|
| `EventType` | `clientName` | Generic field for any integration |
| `ClientTCN` | Stored in `requestBody` | Part of request payload |
| `OrderId` | Stored in `requestBody` | Part of request payload |
| `SellerUserName` | Stored in `requestBody` | Part of request payload |
| `TCCRequest` | `requestBody` | Generic request field |
| `TCCResponse` | Retrieved via logging | Response content |
| `TCCTCN` | Stored in response | Part of response |
| `TCCCode` | `responseStatus` | HTTP/API status |
| `TCCMessage` | `errorMessage` | Error details |

**Additional TCC-Specific Entity:**
```
d:\ReposDOTD\rbmdigitalcustomintegration\semati-service\
  src\main\java\com\segmatek\client\semati\entity\MobileVerification.java
```

```java
@Entity
@Table(name = "mobile_verification")
public class MobileVerification {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Integer id;

    @Column(name = "phone_number")
    private String phoneNumber;

    @Column(name = "national_id")
    private String nationalId;

    @Column(name = "added_date")
    private LocalDateTime addedDate;

    @Column(name = "status_code")
    private Integer statusCode;

    @Column(name = "status_message")
    private String statusMessage;

    @Column(name = "transaction_id")
    private String transactionId;
}
```

---

## 3. Microservices Architecture Mapping

### 3.1 Service Distribution

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        TRANSFER OWNERSHIP FLOW                               │
│                    (Distributed Across Microservices)                        │
└─────────────────────────────────────────────────────────────────────────────┘

┌──────────────────────┐     ┌──────────────────────┐     ┌───────────────────┐
│  digital-external-   │     │  digital-product-    │     │    order-mgmt-    │
│     omni-auth        │     │     catalog          │     │     service       │
│                      │     │                      │     │                   │
│  - User (Seller)     │     │  - SalesChannel      │     │ - ProductOrder    │
│  - City              │     │  - ProductOffering   │     │ - OrderType       │
│  - Governorate       │     │                      │     │ - OrderParams     │
│  - Country           │     │                      │     │                   │
│  - BusinessDomain    │     │                      │     │ (Transfer Owner-  │
│  - BusinessRole      │     │                      │     │  ship to be added)│
└──────────┬───────────┘     └──────────┬───────────┘     └─────────┬─────────┘
           │                            │                           │
           │                            │                           │
           ▼                            ▼                           ▼
┌──────────────────────────────────────────────────────────────────────────────┐
│                        rbmdigitalcustomintegration                            │
│  ┌────────────────┐  ┌────────────────┐  ┌────────────────┐                  │
│  │  tcc-service   │  │ semati-service │  │  sales-app-    │                  │
│  │                │  │                │  │    service     │                  │
│  │ - TCC API      │  │ - Mobile       │  │                │                  │
│  │   Integration  │  │   Verification │  │ - BSS/Huawei   │                  │
│  │                │  │ - Entity:      │  │   Integration  │                  │
│  │                │  │   MobileVerif. │  │                │                  │
│  └────────────────┘  └────────────────┘  └────────────────┘                  │
│                                                                               │
│  ┌────────────────────────────────────────────────────────────────────────┐  │
│  │                        common-module                                    │  │
│  │  - IntegrationLog (replaces TccTraces + other traces)                  │  │
│  └────────────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────────────┘

┌──────────────────────┐     ┌──────────────────────┐
│  bss-mediation-      │     │  digital-customer-   │
│    controller        │     │    management        │
│                      │     │                      │
│  - VendorRequest*    │     │  - Party/Individual  │
│  - VendorResponse*   │     │  - PartyRole         │
│  - BSS Integration   │     │  - Customer          │
│  (* POJOs, not JPA)  │     │  - Identification    │
└──────────────────────┘     └──────────────────────┘
```

---

### 3.2 API Flow in Java Architecture

```
Transfer Ownership Request
         │
         ▼
┌────────────────────────────────────────────────────────────────────────┐
│                    API Gateway (dot-digital-api-gateway)                │
└────────────────────────────────────────────────────────────────────────┘
         │
         ▼
┌────────────────────────────────────────────────────────────────────────┐
│                    Authentication (digital-external-omni-auth)          │
│    - Validate JWT Token                                                 │
│    - Get User (Seller) Info                                            │
│    - Get City → Governorate → Country                                  │
└────────────────────────────────────────────────────────────────────────┘
         │
         ▼
┌────────────────────────────────────────────────────────────────────────┐
│                    Order Management (order-management-service)          │
│    - Create/Update Transfer Ownership Order                            │
│    - Validate Location                                                  │
│    - Manage Order State                                                 │
└────────────────────────────────────────────────────────────────────────┘
         │
    ┌────┴────┐
    ▼         ▼
┌─────────┐ ┌──────────────────────────────────────────────────────────┐
│  TCC    │ │              rbmdigitalcustomintegration                  │
│  API    │ │  ┌─────────────────┐    ┌────────────────────────────┐   │
│         │ │  │  tcc-service    │    │  semati-service            │   │
│         │ │  │  - POST /verify │    │  - MobileVerification      │   │
└─────────┘ │  └─────────────────┘    └────────────────────────────┘   │
            │              │                                            │
            │              ▼                                            │
            │  ┌─────────────────────────────────────────────────────┐ │
            │  │  common-module (IntegrationLog)                      │ │
            │  │  - Log TCC Request/Response                          │ │
            │  └─────────────────────────────────────────────────────┘ │
            └──────────────────────────────────────────────────────────┘
                       │
                       ▼
┌────────────────────────────────────────────────────────────────────────┐
│                    BSS Mediation (bss-mediation-controller)             │
│    - Transform Request to BSS Format                                   │
│    - Call Huawei BSS API                                               │
│    - Handle Response                                                   │
└────────────────────────────────────────────────────────────────────────┘
         │
         ▼
┌────────────────────────────────────────────────────────────────────────┐
│                    Customer Management (digitalcustomermanagement)      │
│    - Update Customer/Party Records                                     │
│    - Update Identification                                             │
└────────────────────────────────────────────────────────────────────────┘
```

---

## 4. IdType Enum Mapping

### .NET Definition:
```csharp
public enum IdType {
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

### Java Equivalent (To Be Created):
**Location:** `digital-lookup` shared module

```java
public enum IdType {
    CITIZEN(1),
    RESIDENT(2),
    VISITOR(3),
    GCC_PASSPORT(4),
    GCC_NATIONAL_ID(5),
    PILGRIM_PASSPORT(6),
    PILGRIM_BORDER(7),
    UMRAH_PASSPORT(8),
    VISITOR_VISA(9),
    UMRAH_VISA(10),
    HAJ_VISA(11),
    DIPLOMAT(99);

    private final int value;

    IdType(int value) {
        this.value = value;
    }

    public int getValue() {
        return value;
    }
}
```

---

## 5. Configuration Mapping

### .NET Configuration Keys → Java Properties

| .NET Config Key | Java Property | Service |
|-----------------|---------------|---------|
| `AppSettings:BypassTCC` | `tcc.bypass.enabled` | `tcc-service` |
| `AppSettings:RBMOrganizationNo` | `tcc.organization.number` | `tcc-service` |
| `ApiSettings:SematiURL` | `semati.api.base-url` | `semati-service` |
| `ApiSettings:SematiApiKey` | `semati.api.key` | `semati-service` |
| `ApiSettings:BSSChannel` | `bss.channel.id` | `bss-mediation-controller` |
| `ApiSettings:BSSUser` | `bss.access.user` | `bss-mediation-controller` |
| `ApiSettings:BSSPassword` | `bss.access.password` | `bss-mediation-controller` |

---

## 6. Implementation Checklist for Transfer Ownership

To fully migrate Transfer Ownership to Java microservices:

### Phase 1: Entity Creation
- [ ] Create `TransferOwnership` entity in `order-management-service`
- [ ] Create `TransferOwnershipOrderType` configuration
- [ ] Add `IdType` enum to `digital-lookup` module

### Phase 2: Service Implementation
- [ ] Implement `TransferOwnershipService` in order-management
- [ ] Integrate with `tcc-service` for TCC API calls
- [ ] Integrate with `bss-mediation-controller` for BSS calls
- [ ] Add location validation logic

### Phase 3: API Endpoints
- [ ] Create REST endpoint `/api/v1/transfer-ownership`
- [ ] Add request/response DTOs
- [ ] Implement validation

### Phase 4: Integration
- [ ] Wire up `digital-external-omni-auth` for user/seller context
- [ ] Connect `digitalcustomermanagement` for customer updates
- [ ] Set up `IntegrationLog` for TCC/BSS tracing

---

## 7. Database Schema Comparison

### .NET Database (SQL Server)
```
SalesPortal_DB
├── TransferOwnership
├── Seller
├── SalesPartner
├── SalesChannel
├── City
├── Province
└── TccTraces
```

### Java Databases (PostgreSQL - Multiple)

```
auth_db (digital-external-omni-auth)
├── user
├── city
├── governorate
├── country
├── business_domain
└── business_role

product_catalog_db (digital-product-catalog)
├── SalesChannel
├── ProductOffering
└── ProductOfferingPrice

order_db (order-management-service)
├── product_order
├── customer_product_order
├── order_type
└── [transfer_ownership] ← TO BE ADDED

integration_db (rbmdigitalcustomintegration)
├── integration_logs
└── mobile_verification

customer_db (digitalcustomermanagement)
├── Party
├── Individual
├── PartyRole
├── Customer
└── PartyIdentification
```

---

## 8. Summary Table

| .NET Table | Java Service | Java Entity | Status |
|------------|--------------|-------------|--------|
| `TransferOwnership` | `order-management-service` | To be created | **PENDING** |
| `Seller` | `digital-external-omni-auth` | `User` + `BusinessRole` | **PARTIAL** |
| `SalesPartner` | `digital-external-omni-auth` | `BusinessDomain` | **PARTIAL** |
| `SalesChannel` | `digital-product-catalog` | `SalesChannel` | **MIGRATED** |
| `City` | `digital-external-omni-auth` | `City` | **MIGRATED** |
| `Province` | `digital-external-omni-auth` | `Governorate` | **MIGRATED** (renamed) |
| `TccTraces` | `rbmdigitalcustomintegration` | `IntegrationLog` | **REDESIGNED** |

---

## 9. Verification Summary

This document was verified against the actual source code. The following files were checked:

| Entity | File Path | Verified |
|--------|-----------|----------|
| SalesChannel | `digitalproductcatalog/.../entity/SalesChannel.java` | YES |
| ProductSalesChannelName | `digitallookup/.../enums/ProductSalesChannelName.java` | YES |
| City | `digital-external-omni-auth/.../entity/address/City.java` | YES |
| Governorate | `digital-external-omni-auth/.../entity/address/Governorate.java` | YES |
| Country | `digital-external-omni-auth/.../entity/address/Country.java` | YES |
| District | `digital-external-omni-auth/.../entity/address/District.java` | YES |
| User | `digital-external-omni-auth/.../entity/User.java` | YES |
| BusinessDomain | `digital-external-omni-auth/.../entity/BusinessDomain.java` | YES |
| BaseModel | `digital-external-omni-auth/.../entity/base/BaseModel.java` | YES |
| IntegrationLog | `rbmdigitalcustomintegration/common-module/.../entity/IntegrationLog.java` | YES |
| MobileVerification | `rbmdigitalcustomintegration/semati-service/.../entity/MobileVerification.java` | YES |
| CustomerProductOrder | `ordermgmtservice/.../entity/CustomerProductOrder.java` | YES |
| ProductOrder | `ordermgmtservice/.../entity/ProductOrder.java` | YES |
| OrderType | `ordermgmtservice/.../entity/OrderType.java` | YES |

### Key Findings:
1. **TransferOwnership** - Does NOT exist in Java microservices (still in .NET)
2. **Seller** - Partially mapped to `User` entity, missing many fields
3. **SalesPartner** - `BusinessDomain` is very simplified (only id, name)
4. **ProductSalesChannelName** - Currently only has `CSO` value defined
5. **Geographic Hierarchy** - District is under City (not Governorate)
6. **VendorRequest/VendorResponse** - Are POJOs, not JPA entities

---

*Document generated and verified from source code analysis - December 2025*
