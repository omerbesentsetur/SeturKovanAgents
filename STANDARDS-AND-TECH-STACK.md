# Setur Beehive — Development Standards & Tech Stack

> **Purpose**: This document defines the technology stack, coding standards, and established patterns in the Beehive codebase. Use it as context when generating code, reviewing PRs, or decomposing stakeholder requests into implementation tasks.

---

## 1. Tech Stack Summary

| Layer | Technology | Version |
|---|---|---|
| **Framework** | ABP Commercial (Volo.Abp) | 7.3.3 |
| **Runtime** | .NET | 7.0 |
| **Language** | C# | latest (11+) |
| **ORM** | Entity Framework Core | 7.0.x |
| **Database** | SQL Server | — |
| **DI Container** | Autofac | via `Volo.Abp.Autofac` |
| **Object Mapping** | AutoMapper | via `Volo.Abp.AutoMapper` |
| **Background Jobs** | Hangfire | 1.8.2 (SQL Server storage) |
| **Caching** | Redis | StackExchange.Redis 2.2.4 |
| **Blob Storage** | Azure Blob Storage | Azure.Storage.Blobs 12.12.0 |
| **Auth** | OpenIddict (OAuth 2.0 / OIDC) | via ABP 7.3.3 |
| **Logging** | Serilog | Console + SQL Server sinks |
| **API Docs** | Swagger / Swashbuckle | via `Volo.Abp.Swashbuckle` |
| **UI Theme** | Lepton Theme | via ABP Commercial |
| **CMS** | CmsKit Pro | via ABP Commercial |
| **Multi-tenancy** | Volo.Saas | Enabled |
| **CI/CD** | Azure Pipelines | YAML |
| **Testing** | xUnit 2.4.1 + Shouldly 4.0.3 + NSubstitute 4.2.2 | EF Core SQLite in-memory for tests |

---

## 2. Complete Package Inventory

### 2.1 ABP Domain Packages (all v7.3.3)

```
Volo.Abp.AuditLogging.Domain
Volo.Abp.BackgroundJobs.Domain / .Domain.Shared
Volo.Abp.FeatureManagement.Domain
Volo.Abp.OpenIddict.Domain
Volo.Abp.PermissionManagement.Domain.Identity / .Domain.OpenIddict / .Domain.Shared
Volo.Abp.SettingManagement.Domain
Volo.Abp.Identity.Pro.Domain / .Domain.Shared
Volo.Abp.LanguageManagement.Domain
Volo.Abp.LeptonTheme.Management.Domain
Volo.Abp.BlobStoring.Database.Domain / .Domain.Shared
Volo.Abp.TextTemplateManagement.Domain
Volo.CmsKit.Pro.Domain / .Domain.Shared
Volo.Saas.Domain / .Domain.Shared
```

### 2.2 ABP Application Packages (all v7.3.3)

```
Volo.Abp.AuditLogging.Application
Volo.Abp.BackgroundWorkers.Hangfire
Volo.Abp.PermissionManagement.Application
Volo.Abp.FeatureManagement.Application
Volo.Abp.SettingManagement.Application
Volo.Abp.Identity.Pro.Application
Volo.Abp.OpenIddict.Pro.Application
Volo.Abp.Account.Pro.Public.Application / .Admin.Application
Volo.Abp.LanguageManagement.Application
Volo.Abp.TextTemplateManagement.Application
Volo.Abp.LeptonTheme.Management.Application
Volo.CmsKit.Pro.Application
Volo.Saas.Host.Application
```

### 2.3 ABP Infrastructure Packages (all v7.3.3)

```
Volo.Abp.Autofac                          # DI container
Volo.Abp.AutoMapper                       # Object mapping
Volo.Abp.Emailing                         # Email abstraction
Volo.Abp.Hangfire                          # Hangfire integration
Volo.Abp.BlobStoring / .Azure             # Blob storage abstraction + Azure provider
Volo.Abp.AspNetCore.Mvc                    # MVC framework
Volo.Abp.AspNetCore.Mvc.UI.Theme.Lepton   # UI theme
Volo.Abp.AspNetCore.Serilog               # Serilog integration
Volo.Abp.Swashbuckle                       # Swagger/OpenAPI
Volo.Abp.VirtualFileSystem                 # Embedded resources
Volo.Abp.Json                              # JSON handling
Volo.Abp.AspNetCore.Authentication.OpenIdConnect  # External auth
```

### 2.4 Third-Party Libraries

| Package | Version | Purpose |
|---|---|---|
| `Azure.Identity` | 1.5.0 | Azure credential chain |
| `Azure.Storage.Blobs` | 12.12.0 | Blob storage client |
| `ClosedXML` | 0.95.4 | Excel workbook generation |
| `FreeSpire.Barcode` | 6.6.0 | Barcode generation |
| `GraphQL.Client` | 6.1.0 | GraphQL queries (Cupid) |
| `GraphQL.Client.Serializer.Newtonsoft` | 6.1.0 | GraphQL JSON serializer |
| `HtmlAgilityPack` | 1.11.39 | HTML parsing |
| `Microsoft.Graph` | 4.23.0 | Microsoft Graph API |
| `MiniExcel` | 1.26.2 | Lightweight Excel read/write |
| `Newtonsoft.Json` | 13.0.3 | JSON serialization |
| `NSwag.ApiDescription.Client` | 13.15.5 | OpenAPI client generation |
| `Select.HtmlToPdf.NetCore` | 25.2.0 | HTML → PDF conversion |
| `Serilog.AspNetCore` | 4.1.0 | Serilog host integration |
| `Serilog.Sinks.MSSqlServer` | 5.6.1 | SQL Server log sink |
| `Serilog.Sinks.Async` | 1.5.0 | Async sink decorator |
| `Serilog.Sinks.Console` | 4.0.1 | Console log sink |
| `Serilog.Sinks.File` | 5.0.0 | File log sink |
| `StackExchange.Redis` | 2.2.4 | Redis client |
| `System.DirectoryServices.AccountManagement` | 6.0.0 | LDAP / Active Directory |

### 2.5 NuGet Sources

```xml
<!-- NuGet.Config -->
<add key="nuget.org" value="https://api.nuget.org/v3/index.json" />
<add key="BlazoriseMyGet" value="https://www.myget.org/F/blazorise/api/v3/index.json" />
<add key="ABP Commercial NuGet Source" value="https://nuget.abp.io/{KEY}/v3/index.json" />
```

---

## 3. Coding Standards & Patterns

### 3.1 Entity / Aggregate Root Pattern

All domain entities inherit `FullAuditedAggregateRoot<Guid>` which provides:
- `Id` (Guid) — primary key
- `CreationTime`, `CreatorId` — creation audit
- `LastModificationTime`, `LastModifierId` — modification audit
- `IsDeleted`, `DeletionTime`, `DeleterId` — soft delete audit

```csharp
public class Vehicle : FullAuditedAggregateRoot<Guid>
{
    public string Plate { get; set; }
    public Guid VehicleBrandId { get; set; }
    public Guid VehicleModelId { get; set; }
    // ...navigation properties
}
```

**Convention**: One entity class per file, file named after the entity, placed in its own folder under `Domain/`.

### 3.2 DTO Pattern

Each entity has a standardized DTO set:

| DTO | Base Class | Purpose |
|---|---|---|
| `{Entity}Dto` | `FullAuditedEntityDto<Guid>` | Output (read) |
| `{Entity}CreateDto` | — | Create input |
| `{Entity}UpdateDto` | — | Update input |
| `Get{Entities}Input` | `PagedAndSortedResultRequestDto` | List query with filters |
| `{Entity}WithNavigationPropertiesDto` | — | Output with related entities |
| `{Entity}ExcelDto` / `{Entity}ExcelDownloadDto` | — | Export-specific (when applicable) |

**Convention**: DTOs live in `Application.Contracts/` under a folder matching the entity name.

### 3.3 Application Service Pattern

Standard CRUD services use ABP's generic `CrudAppService`:

```csharp
public class TTORewardTypesAppService :
    CrudAppService<
        TTORewardType,           // Entity
        TTORewardTypeDto,        // Output DTO
        int,                     // Primary key type
        GetTTORewardTypesInput,  // List input DTO
        TTORewardTypeCreateDto,  // Create DTO
        TTORewardTypeUpdateDto>  // Update DTO
{
    public TTORewardTypesAppService(IRepository<TTORewardType, int> repository)
        : base(repository) { }
}
```

**For complex logic**: Inject `IRepository<>`, domain services, or external service interfaces directly. All app services inherit from `BeehiveAppService` (→ localized `ApplicationService`).

**Convention**: One app service per aggregate, placed in its own folder under `Application/`.

### 3.4 Repository Pattern

- **Default**: ABP's `IRepository<TEntity, TKey>` handles standard CRUD (no custom implementation needed).
- **Custom**: Extend `EfCoreRepository<BeehiveDbContext, TEntity, TKey>` for complex queries.
- **Registration**: In `BeehiveEntityFrameworkCoreModule`:
  ```csharp
  options.AddRepository<AuditFinding, EfCoreAuditFindingRepository>();
  ```

### 3.5 REST API Pattern

Controllers are **convention-based** (auto-generated from app service interfaces). No manual controller code is needed for standard CRUD.

```csharp
// BeehiveWebModule.cs
options.ConventionalControllers.Create(typeof(BeehiveApplicationModule).Assembly);
```

Custom controllers extend `BeehiveController` (→ `AbpController` with localization):

```csharp
public abstract class BeehiveController : AbpController
{
    protected BeehiveController()
    {
        LocalizationResource = typeof(BeehiveResource);
    }
}
```

**Pagination limit**: `LimitedResultRequestDto.MaxMaxResultCount = 10,000`

### 3.6 Permission Pattern

Permissions defined in `Application.Contracts/Permissions/`:

```csharp
// BeehivePermissions.cs — Constants
public static class Vehicles
{
    public const string Default = GroupName + ".Vehicles";
    public const string Create  = Default + ".Create";
    public const string Edit    = Default + ".Edit";
    public const string Delete  = Default + ".Delete";
}

// BeehivePermissionDefinitionProvider.cs — Registration
var vehiclesPermission = beehiveGroup.AddPermission(BeehivePermissions.Vehicles.Default);
vehiclesPermission.AddChild(BeehivePermissions.Vehicles.Create);
vehiclesPermission.AddChild(BeehivePermissions.Vehicles.Edit);
vehiclesPermission.AddChild(BeehivePermissions.Vehicles.Delete);
```

### 3.7 Localization Pattern

JSON resource files in `Domain.Shared/Localization/Beehive/`:

```json
// en.json
{ "Culture": "en", "Texts": { "Vehicles": "Vehicles", "NewVehicle": "New Vehicle", ... } }
// tr.json
{ "Culture": "tr", "Texts": { "Vehicles": "Araçlar", "NewVehicle": "Yeni Araç", ... } }
```

**Supported languages**: English (`en`), Turkish (`tr`), German (`de`).

### 3.8 Background Worker Pattern

Workers implement ABP's periodic worker base and are registered in `BeehiveApplicationModule`:

```csharp
public override async Task OnApplicationInitializationAsync(ApplicationInitializationContext context)
{
    await context.AddBackgroundWorkerAsync<VehicleReservationNotification>();
    // ... 24+ workers
}
```

### 3.9 External Service Integration Pattern

SAP services follow a consistent wrapper pattern:

```csharp
// Interface in Application layer
public interface IEfaturaToSapService { Task<Result> DocInXmlSoapAsync(...); }

// Implementation wraps WCF client
public class EfaturaToSapService : IEfaturaToSapService { ... }

// Registration in BeehiveApplicationModule
context.Services.AddScoped<IEfaturaToSapService, EfaturaToSapService>();
```

### 3.10 Email Pattern

```csharp
// Use IEmailSender (ABP) + ITemplateRenderer for templated emails
// Queue via IBackgroundJobManager → EmailSendingJob
// Attachments stored in Azure Blob → IBlobContainer<EmailAttachmentContainer>
```

### 3.11 AutoMapper Configuration

Single profile in `BeehiveApplicationAutoMapperProfile.cs` with 80+ mappings using convention-based mapping:

```csharp
CreateMap<Vehicle, VehicleDto>();
CreateMap<VehicleCreateDto, Vehicle>();
```

Registered in module: `options.AddMaps<BeehiveApplicationModule>()`

### 3.12 EF Core Configuration Pattern

Fluent configuration in `BeehiveDbContextModelCreatingExtensions.cs`:

```csharp
builder.Entity<Vehicle>(b =>
{
    b.ToTable(BeehiveConsts.DbTablePrefix + "Vehicles", BeehiveConsts.DbSchema);
    b.ConfigureByConvention();
    // indexes, relationships, constraints
});
```

**DbSet registration** in `BeehiveDbContext.cs`:
```csharp
public DbSet<Vehicle> Vehicles { get; set; }
```

---

## 4. Module System

ABP module composition via `[DependsOn]` attribute. The full dependency chain:

```
BeehiveWebModule (entry point)
├── BeehiveHttpApiModule
│   └── BeehiveApplicationContractsModule
│       └── BeehiveDomainSharedModule
├── BeehiveApplicationModule
│   ├── BeehiveDomainModule
│   │   ├── BeehiveDomainSharedModule
│   │   └── 16 ABP domain modules
│   ├── BeehiveApplicationContractsModule
│   ├── AbpBackgroundWorkersHangfireModule
│   └── 14 ABP application modules
├── BeehiveEntityFrameworkCoreModule
│   ├── BeehiveDomainModule
│   └── 11 ABP EF Core modules
├── FormsWebModule (Volo.Forms at each layer)
└── 18 ABP Web/UI/Auth modules
```

### Adding a New Module Dependency

1. Add NuGet/Project reference in `.csproj`
2. Add `typeof(NewModule)` to the `[DependsOn]` attribute in the corresponding `*Module.cs`

---

## 5. Build Configuration

### 5.1 Shared Properties (`common.props`)

```xml
<LangVersion>latest</LangVersion>      <!-- C# 11+ features -->
<Version>1.0.0</Version>
<NoWarn>$(NoWarn);CS1591</NoWarn>        <!-- No XML doc warnings -->
<AbpProjectType>app</AbpProjectType>    <!-- ABP project marker -->
```

### 5.2 Target Frameworks

| Project Type | Framework |
|---|---|
| Domain.Shared, Application.Contracts, HttpApi.Client | `netstandard2.0` (max compatibility) |
| Domain, Application, EFCore, HttpApi, DbMigrator | `net7.0` |
| Web (entry point) | `net7.0` (Web SDK) |

### 5.3 CI/CD Pipeline (`azure-pipelines.yml`)

```
Stages:
  1. StartVM        → Power on AZWEUIBUILDER build agent
  2. RunPipeline    → dotnet restore → build Release → publish Web → upload artifact
  3. StopVM         → Deallocate build agent

Pool: SeturAzUiLinuxAgentPool
SDK: 7.0.x
Output: 'drop' artifact container
```

---

## 6. Testing Standards

### 6.1 Test Stack

| Component | Package | Version |
|---|---|---|
| Framework | xUnit | 2.4.1 |
| Assertions | Shouldly | 4.0.3 |
| Mocking | NSubstitute | 4.2.2 |
| Base | Volo.Abp.TestBase | 7.3.3 |
| DB (tests) | Volo.Abp.EntityFrameworkCore.Sqlite | 7.3.3 (in-memory) |

### 6.2 Test Project Hierarchy

```
BeehiveTestBase (shared setup, data seeding, disables BG jobs)
  └── BeehiveDomainTestModule
  └── BeehiveEntityFrameworkCoreTestModule (SQLite in-memory)
      └── BeehiveApplicationTestModule
          └── BeehiveWebTestModule
```

### 6.3 Test Pattern

```csharp
public class TTORewardTypesAppServiceTests : BeehiveApplicationTestBase
{
    private readonly ITTORewardTypesAppService _service;
    private readonly IRepository<TTORewardType, int> _repository;

    public TTORewardTypesAppServiceTests()
    {
        _service = GetRequiredService<ITTORewardTypesAppService>();
        _repository = GetRequiredService<IRepository<TTORewardType, int>>();
    }

    [Fact]
    public async Task GetListAsync()
    {
        // Arrange - data seeded via IDataSeedContributor
        // Act
        var result = await _service.GetListAsync(new GetTTORewardTypesInput());
        // Assert (Shouldly)
        result.TotalCount.ShouldBeGreaterThanOrEqualTo(0);
    }
}
```

### 6.4 Data Seeding for Tests

```csharp
public class TTORewardTypesDataSeedContributor : IDataSeedContributor, ITransientDependency
{
    public async Task SeedAsync(DataSeedContext context)
    {
        // Insert test data via repositories
    }
}
```

---

## 7. Configuration Management

### 7.1 Key appsettings.json Sections

| Section | Purpose |
|---|---|
| `ConnectionStrings:Default` | Primary SQL Server (Beehive DB) |
| `ConnectionStrings:ServiceNow` | Read-only ServiceNow DB |
| `Azure:ConnectionString` / `Azure:StorageUrl` | Blob storage |
| `Redis:Configuration` | Redis cache endpoint |
| `AuthServer:Authority` | OpenIddict authority URL |
| `Cupid:BaseUrl` | GraphQL HR service |
| `OutUrls:SSOAuthorityUrl` | Duende SSO endpoint |
| `OutUrls:VardiyaSapUrl` | SAP shift management SOAP endpoint |
| `ErgosisPDKS` | Ergosis attendance API credentials |
| `Settings:JobTimerPeriod` | Background worker timer (ms) |
| `Serilog` | Logging configuration (sinks, levels) |
| `QRCodeSettings:ValidityTime` | QR code expiry (minutes) |
| `StringEncryption:DefaultPassPhrase` | Data encryption key |

### 7.2 Lepton Theme Settings

```json
"Volo.Abp.LeptonTheme.Style": "Style1",
"Volo.Abp.LeptonTheme.Layout.MenuPlacement": "Top",
"Volo.Abp.LeptonTheme.Layout.MenuStatus": "AlwaysOpened",
"Volo.Abp.LeptonTheme.Layout.Boxed": "False"
```

---

## 8. Checklist for New Feature Implementation

When implementing a new business feature, follow this sequence:

1. **Domain.Shared** — Add enum(s) and constants if needed
2. **Domain** — Create entity class(es) inheriting `FullAuditedAggregateRoot<Guid>`
3. **Application.Contracts** — Create DTOs (`Dto`, `CreateDto`, `UpdateDto`, `GetListInput`), app service interface, permissions
4. **Application** — Implement app service (extend `CrudAppService` or custom), add AutoMapper mappings
5. **EntityFrameworkCore** — Add `DbSet` to `BeehiveDbContext`, add fluent config in `ModelCreatingExtensions`, run `dotnet ef migrations add`
6. **Localization** — Add keys to `en.json`, `tr.json`, `de.json`
7. **Permissions** — Register in `BeehivePermissionDefinitionProvider`
8. **Tests** — Add data seed contributor + app service tests
9. **Web** (if UI needed) — Add Razor pages, JS, navigation menu entry

---

## 9. Key Conventions Quick Reference

| Convention | Standard |
|---|---|
| Primary key type | `Guid` (all entities) |
| Entity base class | `FullAuditedAggregateRoot<Guid>` |
| Soft delete | Automatic (via base class) |
| Audit trail | Automatic (created/modified/deleted by + timestamps) |
| DI registration | Scoped for services, Transient for jobs/seeders |
| API routing | Convention-based (auto from app service interface) |
| Max page size | 10,000 items |
| Background job timeout | 864,000 seconds (10 days) |
| Token lifetime | 360 minutes |
| Localization | JSON files, 3 languages (en, tr, de) |
| DB table prefix | `BeehiveConsts.DbTablePrefix` |
| Test DB | SQLite in-memory |
| Assertion lib | Shouldly |
| Mock lib | NSubstitute |
