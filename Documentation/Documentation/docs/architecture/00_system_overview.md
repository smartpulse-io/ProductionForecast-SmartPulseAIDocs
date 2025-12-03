# System Overview - SmartPulse.Services.ProductionForecast

**Version**: 2.0
**Last Updated**: 2025-11-28
**Status**: Current

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [System Purpose](#2-system-purpose)
3. [Technology Stack](#3-technology-stack)
4. [Architecture Overview](#4-architecture-overview)
5. [Project Layers](#5-project-layers)
6. [Core Components](#6-core-components)
7. [Data Flow](#7-data-flow)
8. [Caching Architecture](#8-caching-architecture)
9. [CDC Implementation](#9-cdc-implementation)
10. [API Overview](#10-api-overview)

---

## 1. Introduction

SmartPulse.Services.ProductionForecast is a .NET 9.0 REST API for managing electricity production forecasts. This document provides a comprehensive overview of the system architecture and its components.

---

## 2. System Purpose

The service provides:

- **Forecast Management**: CRUD operations for production forecasts
- **Multi-Resolution Support**: 5, 10, 15, 30, 60-minute periods
- **Hierarchical Units**: Power Plants (PP), Companies (CMP), Groups (GRP)
- **Real-time Cache Invalidation**: CDC-based automatic cache refresh
- **Authorization**: User-based access control per unit

### Core Capabilities

| Capability | Description |
|------------|-------------|
| Forecast CRUD | Save, retrieve, query forecasts by various criteria |
| Multi-period support | 5, 10, 15, 30, 60-minute resolution |
| Batch operations | Bulk insert with 2000 record batches |
| Cache invalidation | Real-time via SQL Server Change Tracking |
| Authorization | User-unit access control |

---

## 3. Technology Stack

### Runtime & Framework

| Technology | Version | Purpose |
|------------|---------|---------|
| .NET | 9.0 | Runtime platform |
| ASP.NET Core | 9.0 | Web API framework |
| Entity Framework Core | 9.0.3 | ORM |

### Database

| Technology | Purpose |
|------------|---------|
| SQL Server | Primary database |
| Change Tracking | CDC for cache invalidation |
| Temporal Tables | Audit trail (PowerPlant, CompanyPowerplant) |

### Caching

| Technology | Purpose |
|------------|---------|
| ASP.NET Core Output Cache | HTTP response caching (60 min TTL) |
| IMemoryCache | Application-level caching |

### Key NuGet Packages

```xml
<PackageReference Include="Electric.Core" Version="7.0.158" />
<PackageReference Include="SmartPulse.Infrastructure.Core" Version="7.0.4" />
<PackageReference Include="SmartPulse.Infrastructure.Data" Version="7.0.9" />
<PackageReference Include="Microsoft.EntityFrameworkCore" Version="9.0.3" />
<PackageReference Include="Microsoft.EntityFrameworkCore.SqlServer" Version="9.0.3" />
<PackageReference Include="EFCore.BulkExtensions" />
<PackageReference Include="Microsoft.AspNetCore.Mvc.Versioning" Version="5.1.0" />
<PackageReference Include="Swashbuckle.AspNetCore" Version="8.0.0" />
```

---

## 4. Architecture Overview

### High-Level View

```
┌─────────────────────────────────────────────────────────────────┐
│                           CLIENTS                                │
│              (Web Applications, Mobile, External Systems)        │
└─────────────────────────────────┬───────────────────────────────┘
                                  │ HTTP/REST
                                  ▼
┌─────────────────────────────────────────────────────────────────┐
│                    SMARTPULSE.WEB.SERVICES                       │
│                                                                  │
│  ┌────────────────┐  ┌─────────────────┐  ┌─────────────────┐   │
│  │  Output Cache  │  │   Controllers   │  │   Middleware    │   │
│  │   (60 min)     │  │   v1.0 / v2.0   │  │  Pipeline       │   │
│  └────────────────┘  └─────────────────┘  └─────────────────┘   │
│                                                                  │
└─────────────────────────────────┬───────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────┐
│                    SMARTPULSE.APPLICATION                        │
│                                                                  │
│  ┌────────────────┐  ┌─────────────────┐  ┌─────────────────┐   │
│  │  CacheManager  │  │ ForecastService │  │  CDC Trackers   │   │
│  │ (IMemoryCache) │  │ (Business Logic)│  │   (6 active)    │   │
│  └────────────────┘  └─────────────────┘  └─────────────────┘   │
│                                                                  │
└─────────────────────────────────┬───────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────┐
│              SMARTPULSE.REPOSITORY + ENTITIES                    │
│                                                                  │
│  ┌────────────────┐  ┌─────────────────┐  ┌─────────────────┐   │
│  │  Repositories  │  │ ForecastDbContext│  │ Stored Procs   │   │
│  └────────────────┘  └─────────────────┘  └─────────────────┘   │
│                                                                  │
└─────────────────────────────────┬───────────────────────────────┘
                                  │
                                  ▼
┌─────────────────────────────────────────────────────────────────┐
│                         SQL SERVER                               │
│                                                                  │
│  ┌────────────────┐  ┌─────────────────┐  ┌─────────────────┐   │
│  │    Tables      │  │ Change Tracking │  │ Temporal Tables │   │
│  └────────────────┘  └─────────────────┘  └─────────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Component Roles

| Component | Role | Technology |
|-----------|------|------------|
| **SmartPulse.Web.Services** | HTTP entry point, API controllers | ASP.NET Core 9.0 |
| **SmartPulse.Application** | Business logic, caching, CDC | .NET 9.0 |
| **SmartPulse.Repository** | Data access abstraction | EF Core repositories |
| **SmartPulse.Entities** | Database schema, DbContext | EF Core 9.0.3 |
| **SmartPulse.Models** | DTOs, configuration | .NET classes |
| **SmartPulse.Base** | Shared constants | SystemVariables |
| **SQL Server** | Primary data store | SQL Server 2019+ |

---

## 5. Project Layers

### SmartPulse.Web.Services

**Purpose**: HTTP entry point and request handling

| Component | Responsibility |
|-----------|---------------|
| Controllers | API endpoints (REST v1.0, v2.0) |
| Middleware | Request/response pipeline |
| Policies | Output cache configuration |
| Extensions | Dependency injection setup |
| Services | Background services |

**Key Files:**
- `Controllers/ProductionForecast/ProductionForecastController.cs`
- `Controllers/System/CacheManagerController.cs`
- `Extensions/IServiceCollectionExtensions.cs`
- `Policies/ForecastPolicy.cs`

### SmartPulse.Application

**Purpose**: Business logic and orchestration

| Component | Responsibility |
|-----------|---------------|
| CacheManager | In-memory cache management |
| ForecastService | Forecast business operations |
| ForecastDbService | Database operations |
| CDC Trackers | Change detection and cache invalidation |
| Helpers | Utility functions |

**Key Files:**
- `CacheManager.cs`
- `Services/Forecast/ForecastService.cs`
- `Services/Database/ForecastDbService.cs`
- `Services/Database/CDC/BaseTracker.cs`

### SmartPulse.Repository

**Purpose**: Data access abstraction

| Repository | Responsibility |
|------------|---------------|
| ForecastRepository | T004Forecast CRUD |
| CompanyPowerPlantRepository | Company-plant mappings |
| T000EntityPropertyRepository | Entity properties |

### SmartPulse.Entities

**Purpose**: Database schema and EF Core configuration

| Component | Responsibility |
|-----------|---------------|
| ForecastDbContext | EF Core DbContext |
| Entity classes | Database table mappings |
| Stored procedures | Complex query definitions |

### SmartPulse.Models

**Purpose**: DTOs and configuration models

| Component | Responsibility |
|-----------|---------------|
| API models | Request/response DTOs |
| Forecast models | Forecast data structures |
| AppSettings | Configuration classes |

### SmartPulse.Base

**Purpose**: Shared constants and configuration

| Component | Responsibility |
|-----------|---------------|
| SystemVariables | Environment-based configuration |

---

## 6. Core Components

### 6.1 CacheManager

Singleton service managing in-memory cache with thread-safe access.

**Features:**
- `IMemoryCache` with configurable TTLs
- `SemaphoreSlim` per key for stampede prevention
- `CancellationTokenSource` for manual expiration
- Double-checked locking pattern

**Implementation Pattern:**
```csharp
public class CacheManager
{
    private readonly IMemoryCache _memoryCache;
    private readonly ConcurrentDictionary<string, SemaphoreSlim> _semaphores;
    private readonly ConcurrentDictionary<string, CancellationTokenSource> _expirationTokenSources;

    public async Task<T?> GetOrCreateAsync<T>(string key, Func<Task<T>> factory, TimeSpan ttl)
    {
        if (_memoryCache.TryGetValue(key, out T cached))
            return cached;

        var semaphore = _semaphores.GetOrAdd(key, _ => new SemaphoreSlim(1, 1));
        await semaphore.WaitAsync();
        try
        {
            // Double-check after acquiring lock
            if (_memoryCache.TryGetValue(key, out cached))
                return cached;

            var value = await factory();
            var cts = GetNewOrExistingExpirationTokenSource(key);
            _memoryCache.Set(key, value, new MemoryCacheEntryOptions
            {
                AbsoluteExpirationRelativeToNow = ttl,
                ExpirationTokens = { new CancellationChangeToken(cts.Token) }
            });
            return value;
        }
        finally { semaphore.Release(); }
    }
}
```

**Cache Keys:**

| Key | TTL | Purpose |
|-----|-----|---------|
| `company_all_powerplantgipconfig` | 60 min | GIP configurations |
| `powerplant_all_hierarchies` | 60 min | Unit hierarchies |
| `all_powerplant_timezones` | 24 hours | Timezone data |
| `user_accessible_units_{userId}_{unitType}` | 24 hours | User permissions |
| `company_limitsettings_{companyId}` | 1 min | Forecast limits |
| `group_region_{groupId}` | 60 min | Group region |
| `user_role_{userId}_{role}` | 60 min | User roles |

### 6.2 ForecastService

Business logic orchestrator for forecast operations.

**Methods:**

| Method | Description |
|--------|-------------|
| `SaveForecastsAsync()` | Validate and save forecasts with bulk insert |
| `GetForecastAsync()` | Retrieve latest forecasts |
| `GetForecastMultiAsync()` | Multi-unit retrieval |

**Save Flow:**
1. Validate empty predictions
2. Check for duplicates (PredictionComparer)
3. Validate forecast limits
4. Insert batch via BulkInsertAsync
5. Evict output cache tags
6. Notify position service (optional)

### 6.3 ForecastDbService

Database operations layer.

**Query Methods:**
- `GetPredictionsAsync()` - Forecast retrieval via stored procedures/TVFs
- `GetAllPowerPlantHierarchiesAsync()` - Hierarchy data
- `GetUserAccessibleUnits()` - User permissions
- `GetBatchInfo()` - Batch metadata

**Command Methods:**
- `InsertForecastBatchAsync()` - Bulk insert with EFCore.BulkExtensions

**Bulk Insert Configuration:**
```csharp
new BulkConfig
{
    BatchSize = 2000,
    BulkCopyTimeout = 3600,
    SetOutputIdentity = false,
    TrackingEntities = false,
    WithHoldlock = false
}
```

### 6.4 CDC Trackers

Six change trackers monitoring database tables:

| Tracker | Table | Interval | Cache Action |
|---------|-------|----------|--------------|
| T004ForecastLatestTracker | t004forecast_latest | 100ms | Output cache eviction |
| T000EntityPermissionsTracker | t000entity_permission | 10s | User access cache |
| T000EntityPropertyTracker | t000entity_property | 10s | Config caches |
| T000EntitySystemHierarchyTracker | t000entity_system_hierarchy | 10s | Hierarchy cache |
| SysUserRolesTracker | SysUserRole | 10s | User access cache |
| PowerPlantTracker | PowerPlant | 10s | Timezone cache |

---

## 7. Data Flow

### 7.1 Write Flow (Save Forecast)

```
1. POST /api/v2/production-forecast/{provider}/{unitType}/{unitNo}/forecasts
2. ProductionForecastController.Save()
3. ForecastService.SaveForecastsAsync()
   ├── Validate empty predictions
   ├── Check for duplicates (PredictionComparer)
   ├── Validate forecast limits
   └── ForecastDbService.InsertForecastBatchAsync()
       └── BulkInsertAsync (batch size: 2000)
4. Evict output cache tags
5. Return ApiResponse<IEnumerable<NormalizedBatchForecast>>

[Async CDC Flow]
6. CDC Tracker polls CHANGETABLE (100ms)
7. Detect changes in t004forecast_latest
8. Evict affected output cache tags
```

### 7.2 Read Flow (Get Forecast)

```
1. GET /api/v2/production-forecast/{provider}/{unitType}/{unitNo}/forecasts/latest
2. Output Cache check
   ├── HIT: Return cached response (<5ms)
   └── MISS: Continue to controller
3. ProductionForecastController.GetLatest()
4. ForecastService.GetForecastAsync()
5. CacheManager: Get cached hierarchies (memory)
6. ForecastDbService.GetPredictionsAsync()
   └── Execute stored procedure or TVF
7. Return ApiResponse<IEnumerable<NormalizedUnitForecast>>
8. Store in output cache with tags (TTL: 60min)
```

---

## 8. Caching Architecture

### Two-Level Cache

```
┌─────────────────────────────────────────────────────────────┐
│               LEVEL 1: OUTPUT CACHE                          │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  ASP.NET Core Output Cache Middleware                  │  │
│  │  - Policy: "Forecast"                                  │  │
│  │  - Duration: 60 minutes                                │  │
│  │  - Tag-based eviction                                  │  │
│  │  - Tag format: {unitType}.{unitNo}.{provider}.{period} │  │
│  └───────────────────────────────────────────────────────┘  │
│                           ↓ Miss                             │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│               LEVEL 2: MEMORY CACHE                          │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  CacheManager (IMemoryCache)                           │  │
│  │  - Singleton service                                   │  │
│  │  - SemaphoreSlim per key                               │  │
│  │  - CancellationTokenSource expiration                  │  │
│  │  - Configurable TTLs per cache type                    │  │
│  └───────────────────────────────────────────────────────┘  │
│                           ↓ Miss                             │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│                     SQL SERVER                               │
│  ┌───────────────────────────────────────────────────────┐  │
│  │  Source of Truth                                       │  │
│  │  - Tables with Change Tracking                         │  │
│  │  - Stored Procedures and TVFs                          │  │
│  └───────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

### Cache Invalidation Flow

```
1. Database change (INSERT/UPDATE/DELETE)
2. SQL Server Change Tracking records change
3. CDC Tracker polls CHANGETABLE (100ms - 10s)
4. Tracker processes changes:
   ├── Memory cache: ExpireCacheByKey() - cancels CancellationToken
   └── Output cache: EvictByTagAsync() - removes responses
5. Next request fetches fresh data
```

---

## 9. CDC Implementation

### SQL Server Change Tracking

```sql
-- Enable on database
ALTER DATABASE [ForecastDb]
SET CHANGE_TRACKING = ON
(CHANGE_RETENTION = 2 DAYS, AUTO_CLEANUP = ON);

-- Enable on table
ALTER TABLE [dbo].[t004forecast_latest]
ENABLE CHANGE_TRACKING;
```

### BaseTracker Implementation

All CDC trackers inherit from `BaseTracker`:

```csharp
public abstract class BaseTracker : TableChangeTrackerBase
{
    protected abstract string TrackerName { get; }
    protected virtual int IntervalMs { get; } = SystemVariables.CDCInterval;

    protected abstract Task OnChangeAction(List<ChangeItem> changes, Guid traceId);

    protected sealed override Task OnChange(List<ChangeItem> changes)
    {
        var traceId = Guid.NewGuid();
        try { OnChangeAction(changes, traceId); }
        catch (Exception ex) { Logger.LogError(ex, ...); }
        return Task.CompletedTask;
    }
}
```

### Tracker Registration

```csharp
// In IServiceCollectionExtensions.cs
services.AddSmartpulseTableChangeTracker<T000EntityPermissionsTracker>();
services.AddSmartpulseTableChangeTracker<T000EntityPropertyTracker>();
services.AddSmartpulseTableChangeTracker<T000EntitySystemHierarchyTracker>();
services.AddSmartpulseTableChangeTracker<SysUserRolesTracker>();
services.AddSmartpulseTableChangeTracker<PowerPlantTracker>();

if (appSettings.CacheSettings.OutputCache.UseCacheInvalidationChangeTracker)
{
    services.AddSmartpulseTableChangeTracker<T004ForecastLatestTracker>();
}
```

---

## 10. API Overview

### ProductionForecastController

**Base Route**: `/api/v{version}/production-forecast`

| Method | Route | Version | Cached | Description |
|--------|-------|---------|--------|-------------|
| POST | `/{providerKey}/{unitType}/{unitNo}/forecasts` | v2.0 | No | Save forecasts |
| GET | `/{providerKey}/{unitType}/{unitNo}/forecasts/latest` | v1.0, v2.0 | Yes | Get latest |
| GET | `/{providerKey}/{unitType}/{unitNo}/forecasts/latest-by-date` | v2.0 | Yes | Get by date |
| GET | `/{providerKey}/{unitType}/{unitNo}/forecasts/latest-by-production-time-offset` | v2.0 | Yes | Get by offset |
| POST | `/GetLatestMulti` | v2.0 | No | Multi-unit get |

### CacheManagerController

**Base Route**: `/api/v{version}/system/cache-manager`

| Method | Route | Description |
|--------|-------|-------------|
| GET | `/cache-types` | List cache types |
| POST | `/all/expire` | Clear all cache |
| POST | `/{cacheType}/expire` | Expire specific cache |

### Route Parameters

| Parameter | Type | Values |
|-----------|------|--------|
| providerKey | string | FinalForecast, UserForecast, ForecastImport |
| unitType | enum | PP (Power Plant), CMP (Company), GRP (Group) |
| unitNo | int | Unit identifier |

---

## Related Documentation

- [Architectural Patterns](architectural_patterns.md)
- [Data Flow & Communication](data_flow_communication.md)
- [Caching Patterns](../patterns/caching_patterns.md)
- [CDC Documentation](../data/cdc.md)

---

**Document Version**: 2.0
**Last Updated**: 2025-11-28
