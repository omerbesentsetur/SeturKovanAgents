# Setur Beehive — Architecture Reference

> **Purpose**: This document describes the architecture of the Beehive platform. Use it as context when decomposing stakeholder requests into tasks, estimating impact, and identifying which layers/modules a change touches.

---

## 1. High-Level Overview

Beehive is an **enterprise internal operations platform** built for Setur (a Koç Group company). It covers HR/onboarding, invoicing, procurement, audits, vehicle fleet, inventory, innovation/ideas, training (TTO), supplier management, cargo logistics, dynamic forms, and more — all integrated with SAP back-office systems.

The application is a **server-rendered monolith** (Razor Pages UI + REST API) built on the **ABP Commercial Framework v7.3.3** with **.NET 7**, following **Domain-Driven Design (DDD)** and **Clean Architecture** principles.

---

## 2. Solution Structure

```
Setur.Beehive.sln  (31 projects)
│
├── src/                              # Application layers
│   ├── Setur.Beehive.Domain.Shared        # Enums, constants, localization resources (netstandard2.0)
│   ├── Setur.Beehive.Domain               # Entities, aggregates, domain services, repository interfaces (net7.0)
│   ├── Setur.Beehive.Application.Contracts# DTOs, app-service interfaces, permissions (netstandard2.0)
│   ├── Setur.Beehive.Application          # App services, background workers, external service wrappers (net7.0)
│   ├── Setur.Beehive.EntityFrameworkCore  # DbContext, EF configurations, migrations, repositories (net7.0)
│   ├── Setur.Beehive.HttpApi              # REST controllers (thin, convention-based) (net7.0)
│   ├── Setur.Beehive.HttpApi.Client       # Client-side proxies & DTOs (netstandard2.0)
│   ├── Setur.Beehive.Web                  # Entry point – Razor Pages, module composition, middleware (net7.0 Web SDK)
│   └── Setur.Beehive.DbMigrator          # Console app for running EF migrations (net7.0)
│
├── Setur.Beehive.ServiceReferences/  # WCF/SOAP service proxies for SAP integration
│
├── modules/
│   └── Volo.Forms/                   # Pluggable dynamic-forms module (full DDD stack)
│
└── test/                             # xUnit test projects (TestBase, Domain, Application, EFCore, Web)
```

---

## 3. Layer Dependency Graph

```
┌──────────────────────────────────────────────────────────────┐
│                   Setur.Beehive.Web                          │
│            (Entry point · Razor Pages · Middleware)           │
│   DependsOn: HttpApi, Application, EntityFrameworkCore,      │
│              23 ABP modules, Volo.Forms.Web                  │
└───────┬──────────────────────┬───────────────────┬───────────┘
        │                      │                   │
        ▼                      ▼                   ▼
  HttpApi (REST)        Application          EntityFrameworkCore
  ┌──────────┐        ┌──────────────┐       ┌──────────────────┐
  │ Thin      │        │ AppServices  │       │ BeehiveDbContext  │
  │ controllers│       │ BG Workers   │       │ 200+ DbSets      │
  │ Convention │       │ SAP Wrappers │       │ SQL Server        │
  │ based      │       │ AutoMapper   │       │ ServiceNowContext │
  └─────┬──────┘       └──────┬───────┘       └────────┬─────────┘
        │                     │                        │
        ▼                     ▼                        ▼
  Application.Contracts     Domain                 Domain
  ┌──────────────────┐  ┌───────────────────┐
  │ DTOs, Interfaces │  │ Aggregates (100+) │
  │ Permissions      │  │ Domain Services   │
  │                  │  │ Repo Interfaces   │
  └───────┬──────────┘  └────────┬──────────┘
          │                      │
          ▼                      ▼
      Domain.Shared          ServiceReferences
  ┌──────────────────┐   ┌────────────────────┐
  │ Enums, Constants │   │ 15 SAP WCF proxies │
  │ Localization     │   └────────────────────┘
  └──────────────────┘
```

**Key rule**: Dependencies flow **downward only**. Upper layers never depend on lower layers' implementations — only on abstractions defined in Domain or Contracts.

---

## 4. Domain Model — Business Modules

The domain contains **100+ aggregate roots**, all inheriting `FullAuditedAggregateRoot<Guid>` (auto audit trail, soft delete).

| Business Module | Key Aggregates | Notes |
|---|---|---|
| **HR & People** | `UserProfile`, `UserSync`, `OnBoarding`, `OffBoarding`, `EffortOnBoarding/OffBoarding`, `UserArchiveFile` | AD integration, employee lifecycle |
| **Finance & Invoicing** | `Invoice`, `EInvoice`, `SupplierInvoice`, `InvoiceExpenseRequest`, `InvoiceBudget`, `InvoicePurchaseOrder`, `InvoiceComment`, `InvoiceTax`, `InvoiceDocument`, `OldInvoice` | 30+ related entities; deep SAP integration |
| **Procurement & Supply** | `Supplier`, `SupplierDocument`, `SupplierExtract`, `Cargo`, `CargoType/Status/Firm`, `CatalogPurchasing*`, `BidDetail*` | Purchase orders, logistics, bidding |
| **Vehicle Fleet** | `Vehicle`, `VehicleBrand`, `VehicleModel`, `VehicleReservation`, `VehicleReservationStatus/Reason` | Fleet management with reservation workflows |
| **Asset & Inventory** | `Inventory`, `InventoryDetail`, `InventoryOperation`, `InventoryInvoice`, `Accessory`, `LansweeperHardwareData`, `RfidLog` | RFID tracking, Lansweeper sync |
| **Audit & Compliance** | `Audit`, `AuditType`, `AuditFinding`, `AuditAction`, `AuditSignificance`, `AuditStatusHistory` | Findings → Actions → Follow-up |
| **Action Plans** | `ActionPlan`, `ActionPlanCategory`, `ActionPlanQuestion`, `ActionPlanHistory`, `ActionPlanManagerFile` | Experience-driven action plans |
| **Innovation & Ideas** | `Idea`, `IdeaCategory`, `IdeaQuestion/Option`, `IdeaScore`, `IdeaScoreParameter`, `IdeaSubjectOfficer` | Scoring, feedback, officer assignment |
| **TTO (Training)** | `TTOOperation`, `TTOBudget`, `TTOApproval`, `TTORewardType/Amount`, `TTOAchievementCriteria`, `TTOTitleHierarchy` | Suggestion system with reward cycles |
| **Mobile** | `MobileCategory`, `MobileDocument`, `MobileShortcut` | Mobile app content management |
| **Notifications & Email** | `Email`, `NotificationTemplate`, `NotificationSubject`, `FormNotification` | Template-based email with blob attachments |
| **General / Master Data** | `Company`, `CompanyGroup`, `Department`, `Location`, `Country`, `City`, `District`, `Category`, `SubCategory`, `Faq` | Shared reference data |
| **Config & System** | `BeehiveConfig`, `BeehiveParameter`, `BeehiveHttpLog`, `BackgroundWorker`, `BackgroundWorkerTask` | Runtime config, job tracking, HTTP logging |
| **Dynamic Forms** (module) | `Form`, `Question`, `Choice`, `FormResponse`, `Answer` | Volo.Forms — pluggable module |

---

## 5. Application Layer

### 5.1 Application Services

Every business module has at least one `AppService` class. Most extend ABP's `CrudAppService<TEntity, TDto, TKey, TGetListInput, TCreateDto, TUpdateDto>` for automatic CRUD, with custom methods added for domain-specific operations. All inherit from `BeehiveAppService` (→ `ApplicationService` with `BeehiveResource` localization).

**Count**: 40+ application services, organized one-folder-per-feature.

### 5.2 Background Workers (24+ Hangfire Workers)

Registered in `BeehiveApplicationModule.OnApplicationInitialization`:

| Worker | Purpose |
|---|---|
| `ADUserCacheWorker` | Active Directory user cache refresh |
| `AuditFindingNotification` | Audit finding email alerts |
| `BirthdayReminderToManager` | Manager birthday notification |
| `C4CAlternaIntegrationWorker` | Alterna C4C data sync |
| `ComponentsCacheWorker` | Component cache refresh |
| `CupidIntegration` | HR data sync (GraphQL) |
| `FormNotifications` | Dynamic form notification delivery |
| `UserSyncNotification` | User sync status alerts |
| `VehicleReservationNotification` | Reservation reminders |
| `SendUsersToAlternaWorker` | Outbound user sync to Alterna |
| `GetAlternaSurveyDataWorker` | Survey data ingest |
| `InventoryConfirmationReminder` | Inventory confirmation follow-ups |
| `BeyondAlternaIntegration` | Beyond Alterna sync |
| `CheckInCheckOutQRCodeGenerator` | QR code generation (10-min validity) |
| `AlternaTourDataIntegration` | Tour data sync |
| `AlternaHotelDataIntegration` | Hotel data sync |
| `VardiyaSAPDataIntegration` | Shift schedule SAP sync |
| `JiraUserUpdateWorker` | Jira user provisioning |
| `InokuReminderEmail` | Inoku reminder emails |
| `KonuSorumlusuReminderEmail` | Subject officer reminders |
| `BeyondPisanoIntegration` | Pisano integration (Beyond) |
| `DeneyimOdakliAksiyonReminderEmail` | Experience-action reminders |
| `C4CPisanoIntegration` | Pisano integration (C4C) |
| `EInvoiceDocumentsUploadWorker` | E-invoice document upload |
| `DisableOutsourceADAccounts` | Outsource AD account cleanup |
| `LansweeperDataJob` | Lansweeper hardware data sync |

### 5.3 Background Jobs

- `EmailSendingJob` — `AsyncBackgroundJob<SendEmailDto>` for email delivery with attachments.
- Timeout: 10 days (`864,000` seconds).

---

## 6. External Integrations

### 6.1 SAP (WCF/SOAP)

15 WCF Connected Service references in `Setur.Beehive.ServiceReferences`, wrapped by typed service classes in `SapServices/`:

| Service | Purpose |
|---|---|
| `EfaturaToSap` | E-Invoice → SAP |
| `HrSapLegalService` | HR & legal data |
| `KocSistemEInvoice` | Koç Sistem e-invoice |
| `LogoEInvoice` | Logo e-invoicing |
| `NewSapParameters` | SAP parameter sync |
| `NewSapSendInvoice` | Invoice transmission (new format) |
| `NewSupplierExtractPayment` | Supplier payment extraction |
| `NewSuppliers` / `NewSuppliersEkstre` | Supplier data + statements |
| `NewVendorInfo` | Vendor information |
| `SapSendInvoice` | Invoice transmission (legacy) |
| `SupplierExtractPayment` | Payment extraction (legacy) |
| `Suppliers` / `SuppliersEkstre` | Supplier data + statements (legacy) |
| `Vardiya` | Shift/schedule management |

**Registration**: All registered as `AddScoped` in `BeehiveApplicationModule`.

### 6.2 Other External Systems

| System | Protocol | Purpose |
|---|---|---|
| **Cupid** (Koç) | GraphQL (`GraphQL.Client`) | HR/employee data |
| **Microsoft Graph** | REST (`Microsoft.Graph`) | AAD, mailbox access |
| **Active Directory** | LDAP (`System.DirectoryServices`) | User directory sync |
| **Azure Blob Storage** | SDK (`Azure.Storage.Blobs`) | File/document storage |
| **Redis** | TCP (`StackExchange.Redis`) | Distributed caching |
| **Ergosis PDKS** | REST | Personnel attendance system |
| **Lansweeper** | Data sync | Hardware inventory |
| **Alterna / Pisano** | Integration workers | Survey, tour, hotel, C4C sync |
| **Jira** | Worker-based | User provisioning sync |
| **ServiceNow** | EF Core (read-only) | Secondary DB context (`ServiceNowContext`) |
| **Sensormatic** | Integration | Security/access control |
| **ProtekZaman** | Integration | Time/attendance |
| **SeturKvkk** | Integration | GDPR/data privacy compliance |
| **TurizmPrim** | Integration | Tourism/hospitality system |

---

## 7. Data Layer

### 7.1 Database

- **Engine**: SQL Server
- **ORM**: Entity Framework Core 7
- **Primary context**: `BeehiveDbContext` — 200+ `DbSet<>` properties
- **Secondary context**: `ServiceNowContext` — read-only for ServiceNow data
- **Connection**: `"Default"` connection string; `"ServiceNow"` for external DB
- **Primary keys**: `Guid` throughout (distributed-safe)

### 7.2 DbContext Composition

```csharp
[ReplaceDbContext(typeof(IIdentityProDbContext))]  // Replaces ABP Identity context
[ReplaceDbContext(typeof(IFormsDbContext))]         // Replaces Volo.Forms context
[ConnectionStringName("Default")]
public class BeehiveDbContext : AbpDbContext<BeehiveDbContext>,
    IIdentityProDbContext, IFormsDbContext
```

Single unified context replaces multiple ABP module contexts for simpler schema management.

### 7.3 Migrations

- **Strategy**: EF Core Code-First
- **Runner**: `Setur.Beehive.DbMigrator` console app + auto-migration on Web startup (`BeehiveDbMigrationService.MigrateAsync()`)
- **Design-time factory**: `BeehiveDbContextFactory` for EF tooling

### 7.4 Repositories

- Default generic `IRepository<TEntity, TKey>` (ABP) for all entities
- 30+ custom `EfCore*Repository` implementations for complex queries
- Registration: `options.AddRepository<TEntity, TConcreteRepository>()`

---

## 8. Authentication & Authorization

| Concern | Implementation |
|---|---|
| **Auth protocol** | OpenIddict (OAuth 2.0 / OIDC) |
| **Token lifetime** | 360 minutes (identity + access) |
| **External IdP** | Google, Microsoft Account, Twitter; Duende SSO (`duendesso-staging.setur.software`) |
| **Permissions** | Hierarchical: Group → Module → CRUD actions (50+ permission groups) |
| **Multi-tenancy** | Enabled via `Volo.Saas` — tenant DB isolation supported |

### Permission Model

```
Beehive (root group)
├── Dashboard.Host / Dashboard.Tenant
├── SendEmail, Profile, Faq, Hangfire
└── Per-module permissions (50+ modules)
    └── Each: Default (view), Create, Edit, Delete
    └── Custom: SyncUsers, ITAdmin, DisasterHoldingAdmin, etc.
```

---

## 9. Cross-Cutting Concerns

| Concern | Technology | Config |
|---|---|---|
| **Logging** | Serilog → Console (Error) + SQL Server (`LogBeehive` table, auto-create) | `appsettings.json` Serilog section |
| **Caching** | Redis (StackExchange.Redis) | `Redis:Configuration` |
| **Blob storage** | Azure Blob Storage (`Volo.Abp.BlobStoring.Azure`) | `Azure:ConnectionString`, containers auto-created |
| **Background processing** | Hangfire (SQL Server storage) | `Volo.Abp.BackgroundWorkers.Hangfire` |
| **Email** | `IEmailSender` (ABP) + `ITemplateRenderer` (Liquid/Scriban) | Templates: `EmailLayout`, `EmailLayoutWithLink`, etc. |
| **Audit trail** | `FullAuditedAggregateRoot<Guid>` on all entities (created/modified/deleted by + timestamps) | Automatic via ABP |
| **Localization** | JSON resource files (en, tr, de) in `Domain.Shared` | `BeehiveResource` |
| **DI container** | Autofac | Configured in `Program.cs` |
| **Object mapping** | AutoMapper (`BeehiveApplicationAutoMapperProfile`, 80+ mappings) | Convention-based |
| **API docs** | Swagger/Swashbuckle (`AbpSwashbuckleModule`) | Auto-generated |
| **Health checks** | Built but currently **disabled** (commented out) | `HealthChecksBuilderExtensions.cs` |

---

## 10. Volo.Forms Module (Pluggable)

A self-contained **dynamic forms** module following the same DDD layered architecture:

```
modules/Volo.Forms/
├── src/
│   ├── Volo.Forms.Domain.Shared
│   ├── Volo.Forms.Domain            # Form, Question, Choice, Response, Answer, Settings
│   ├── Volo.Forms.Application.Contracts
│   ├── Volo.Forms.Application
│   ├── Volo.Forms.EntityFrameworkCore
│   ├── Volo.Forms.HttpApi           # FormController, QuestionController, ResponseController
│   ├── Volo.Forms.HttpApi.Client
│   ├── Volo.Forms.Web
│   └── Volo.Forms.Installer
└── test/
```

Integrated via project references at every layer. The main `BeehiveDbContext` replaces `IFormsDbContext` to keep all data in one database.

---

## 11. CI/CD Pipeline

**File**: `azure-pipelines.yml`

```
Stage 1: StartVM       → Start AZWEUIBUILDER build VM
Stage 2: RunPipeline   → dotnet restore → build (Release) → publish (Web) → PublishBuildArtifacts
Stage 3: StopVM        → Deallocate build VM
```

- **Agent pool**: `SeturAzUiLinuxAgentPool` (custom)
- **.NET SDK**: 7.0.x
- **NuGet sources**: nuget.org + ABP Commercial private feed + Blazorise MyGet
- **Output**: `drop` artifact (Web project publish)

---

## 12. Deployment Topology

```
┌─────────────────────────────────────────────────────────┐
│                     Azure (Production)                  │
│                                                         │
│  ┌─────────────┐  ┌──────────────┐  ┌───────────────┐ │
│  │ Web App      │  │ SQL Server   │  │ Azure Blob    │ │
│  │ (Beehive.Web)│  │ (Beehive DB) │  │ Storage       │ │
│  └──────┬───────┘  └──────────────┘  └───────────────┘ │
│         │                                               │
│  ┌──────┴───────┐  ┌──────────────┐                    │
│  │ Redis Cache  │  │ Hangfire     │                    │
│  │              │  │ (SQL backed) │                    │
│  └──────────────┘  └──────────────┘                    │
└─────────────────────────────────────────────────────────┘
         │                    │
         ▼                    ▼
┌─────────────────┐  ┌─────────────────────────┐
│ SAP Systems     │  │ External Services        │
│ (WCF/SOAP)      │  │ (Cupid, Graph, AD,      │
│ 15 endpoints    │  │  Alterna, Ergosis, etc.) │
└─────────────────┘  └─────────────────────────┘
```

---

## 13. Key Configuration Files Reference

| File | Purpose |
|---|---|
| `aspnet-core/common.props` | Shared build properties (LangVersion, Version, NoWarn) |
| `aspnet-core/NuGet.Config` | Package source URLs (nuget.org, ABP Commercial, Blazorise) |
| `src/Setur.Beehive.Web/appsettings.json` | Connection strings, Redis, Azure, SAP URLs, Serilog, settings |
| `src/Setur.Beehive.Web/Program.cs` | Host builder, Serilog init, auto-migration, global config |
| `src/Setur.Beehive.Web/Startup.cs` | DI registration, ABP initialization |
| `src/Setur.Beehive.Web/BeehiveWebModule.cs` | Module composition (23 DependsOn), middleware pipeline |
| `src/Setur.Beehive.Application/BeehiveApplicationModule.cs` | Service registration, background workers, SAP clients |
| `src/Setur.Beehive.EntityFrameworkCore/EntityFrameworkCore/BeehiveDbContext.cs` | 200+ DbSets, context replacement |
| `src/Setur.Beehive.EntityFrameworkCore/EntityFrameworkCore/BeehiveDbContextModelCreatingExtensions.cs` | Fluent EF Core configurations |

---

## 14. Impact Analysis Guide for Task Decomposition

When a stakeholder request arrives, use this guide to identify touched layers:

| Change Type | Layers Affected | Files to Modify |
|---|---|---|
| **New entity/feature** | Domain.Shared → Domain → Contracts → Application → EFCore → HttpApi (auto) → Web (optional UI) | Enum, Entity, DTO, AppService, DbContext DbSet, ModelCreating, Migration, Permission |
| **New API endpoint** | Contracts → Application → HttpApi (if custom controller) | Interface, DTO, AppService, Controller |
| **New background worker** | Application | Worker class + register in `BeehiveApplicationModule` |
| **SAP integration change** | ServiceReferences → Application | WCF proxy + Service wrapper + AppService |
| **Permission change** | Contracts (`Permissions/`) | `BeehivePermissions.cs` + `BeehivePermissionDefinitionProvider.cs` |
| **Localization** | Domain.Shared (`Localization/`) | `en.json`, `tr.json`, `de.json` |
| **UI change** | Web (`Pages/`) | Razor page + JS + CSS |
| **Database schema change** | EFCore → DbMigrator | Entity, DbContext, ModelCreating, `dotnet ef migrations add` |
| **Configuration change** | Web (`appsettings.json`) + relevant module | Config section + module `Configure<>` call |
| **External integration** | Application (+ ServiceReferences if SOAP) | Service interface, implementation, registration |
