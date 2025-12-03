# SmartPulse.Services.ProductionForecast - Technical Summary

**Last Updated**: 2025-11-28
**Project Type**: Production Forecast Management API
**Complexity**: Moderate-High
**Codebase Size**: ~77 C# files, ~50 key classes

---

## Executive Summary

SmartPulse.Services.ProductionForecast is a **.NET 9.0 REST API** for managing electricity production forecasts. The service provides CRUD operations for forecasts with multi-level caching, real-time cache invalidation via SQL Server Change Data Capture (CDC), and user-based authorization.

### Key Characteristics

| Attribute | Value |
|-----------|-------|
| **Framework** | .NET 9.0 / ASP.NET Core 9.0 |
| **Database** | SQL Server with EF Core 9.0.3 |
| **Caching** | Output Cache + In-Memory Cache |
| **Real-time Sync** | CDC-based local cache invalidation |
| **API Versioning** | v1.0 (deprecated), v2.0 (active) |

---

## Project Structure

```
SmartPulse.Services.ProductionForecast/
├── SmartPulse.Web.Services/        # ASP.NET Core Web API
│   ├── Controllers/                # REST endpoints
│   ├── Extensions/                 # DI configuration
│   ├── Middlewares/                # Request pipeline
│   ├── Policies/                   # Output cache policies
│   └── Services/                   # Background services
│
├── SmartPulse.Application/         # Business Logic Layer
│   ├── CacheManager.cs             # In-memory cache (Singleton)
│   ├── Services/Database/          # Database operations
│   │   ├── ForecastDbService.cs    # Main DB service
│   │   └── CDC/                    # Change trackers (6 trackers)
│   └── Services/Forecast/          # Business logic
│
├── SmartPulse.Repository/          # Data Access Layer
│   └── [Repository classes]        # EF Core repositories
│
├── SmartPulse.Entities/            # Entity Framework Layer
│   └── Sql/                        # DbContext + Entities
│
├── SmartPulse.Models/              # DTOs and Configuration
│   ├── API/                        # Response models
│   ├── Forecast/                   # Forecast DTOs
│   └── Requests/                   # Request validation
│
└── SmartPulse.Base/                # Shared Constants
    └── SystemVariables.cs          # Environment configuration
```

---

## Technology Stack

### Core Dependencies

| Package | Version | Purpose |
|---------|---------|---------|
| .NET | 9.0 | Runtime |
| ASP.NET Core | 9.0 | Web framework |
| Entity Framework Core | 9.0.3 | ORM |
| Electric.Core | 7.0.158 | CDC base classes, helpers |
| SmartPulse.Infrastructure.Core | 7.0.4 | Base DbContext |
| SmartPulse.Infrastructure.Data | 7.0.9 | SQL extensions |
| EFCore.BulkExtensions | latest | Bulk insert |
| Swashbuckle.AspNetCore | 8.0.0 | Swagger/OpenAPI |

### External Systems

| System | Purpose |
|--------|---------|
| SQL Server | Primary database with Change Tracking |

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                         CLIENTS                              │
│            (Web Apps, Mobile, External Systems)              │
└─────────────────────────────┬───────────────────────────────┘
                              │ HTTP/REST
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                   WEB API LAYER                              │
│  ┌──────────────┐  ┌───────────────┐  ┌─────────────────┐   │
│  │ Output Cache │  │  Controllers  │  │   Middleware    │   │
│  │  (60 min)    │  │  (v1.0, v2.0) │  │ (Auth, Logging) │   │
│  └──────────────┘  └───────────────┘  └─────────────────┘   │
└─────────────────────────────┬───────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                  APPLICATION LAYER                           │
│  ┌──────────────┐  ┌───────────────┐  ┌─────────────────┐   │
│  │ CacheManager │  │ForecastService│  │  CDC Trackers   │   │
│  │ (IMemoryCache)│  │(Business Logic)│  │  (6 active)     │   │
│  └──────────────┘  └───────────────┘  └─────────────────┘   │
└─────────────────────────────┬───────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    DATA LAYER                                │
│  ┌──────────────┐  ┌───────────────┐  ┌─────────────────┐   │
│  │ Repositories │  │  DbContext    │  │ Stored Procs    │   │
│  └──────────────┘  └───────────────┘  └─────────────────┘   │
└─────────────────────────────┬───────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                     SQL SERVER                               │
│  ┌──────────────┐  ┌───────────────┐  ┌─────────────────┐   │
│  │   Tables     │  │Change Tracking│  │ Temporal Tables │   │
│  └──────────────┘  └───────────────┘  └─────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

---

## Core Components

### 1. CacheManager

Singleton in-memory cache with stampede prevention:

- **Implementation**: `Microsoft.Extensions.Caching.Memory.IMemoryCache`
- **Concurrency**: `SemaphoreSlim` per cache key
- **Expiration**: `CancellationTokenSource` + absolute TTL
- **Pattern**: Double-checked locking

**Cache Keys:**
| Key | TTL | Purpose |
|-----|-----|---------|
| `powerplant_all_hierarchies` | 60 min | Unit hierarchies |
| `all_powerplant_timezones` | 24 hours | Timezone data |
| `user_accessible_units_{userId}_{unitType}` | 24 hours | User permissions |
| `company_limitsettings_{companyId}` | 1 min | Forecast limits |

### 2. CDC Trackers (6 Active)

| Tracker | Table | Interval | Cache Action |
|---------|-------|----------|--------------|
| T004ForecastLatestTracker | t004forecast_latest | 100ms | Output cache eviction |
| T000EntityPermissionsTracker | t000entity_permission | 10s | User access cache |
| T000EntityPropertyTracker | t000entity_property | 10s | Config caches |
| T000EntitySystemHierarchyTracker | t000entity_system_hierarchy | 10s | Hierarchy cache |
| SysUserRolesTracker | SysUserRole | 10s | User access cache |
| PowerPlantTracker | PowerPlant | 10s | Timezone cache |

### 3. ForecastService

Business logic for forecast operations:

- `SaveForecastsAsync()` - Validate and save forecasts with bulk insert
- `GetForecastAsync()` - Retrieve latest forecasts
- `GetForecastMultiAsync()` - Multi-unit retrieval

---

## API Endpoints

### ProductionForecastController

**Base Route**: `/api/v{version}/production-forecast`

| Method | Endpoint | Cached | Description |
|--------|----------|--------|-------------|
| POST | `/{providerKey}/{unitType}/{unitNo}/forecasts` | No | Save forecasts |
| GET | `/{providerKey}/{unitType}/{unitNo}/forecasts/latest` | Yes | Get latest |
| GET | `/{providerKey}/{unitType}/{unitNo}/forecasts/latest-by-date` | Yes | Get by date |
| GET | `/{providerKey}/{unitType}/{unitNo}/forecasts/latest-by-production-time-offset` | Yes | Get by offset |
| POST | `/GetLatestMulti` | No | Multi-unit get |

### CacheManagerController

**Base Route**: `/api/v{version}/system/cache-manager`

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/cache-types` | List cache types |
| POST | `/all/expire` | Clear all cache |
| POST | `/{cacheType}/expire` | Expire specific cache |

---

## Caching Strategy

### Two-Level Cache

```
Level 1: ASP.NET Core Output Cache
├── Duration: 60 minutes
├── Tag-based eviction
└── Policy: ForecastPolicy

Level 2: IMemoryCache (CacheManager)
├── SemaphoreSlim per key
├── CancellationToken expiration
└── Configurable TTLs
```

### Cache Invalidation

CDC Trackers poll SQL Server Change Tracking and invalidate caches:

1. Database change occurs
2. CDC Tracker polls CHANGETABLE (100ms - 10s interval)
3. Tracker identifies affected cache keys/tags
4. Memory cache: `ExpireCacheByKey()` cancels CancellationToken
5. Output cache: `EvictByTagAsync()` removes responses

---

## Configuration

### appsettings.json

```json
{
  "CacheSettings": {
    "OutputCache": {
      "UseCacheInvalidationChangeTracker": true,
      "Duration": 60
    },
    "MemoryCache": {
      "GipConfigDuration": 60,
      "HierarchyDuration": 60,
      "RegionDuration": 60
    }
  }
}
```

### Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `CDC_Interval` | 1000 | CDC poll interval (ms) |
| `CDC_LongInterval` | 10000 | Slow CDC interval (ms) |
| `ALLOWED_PERIODS` | 5,10,15,30,60 | Valid forecast periods |

---

## Database

### Key Entities

| Entity | Purpose |
|--------|---------|
| T004Forecast | Forecast data |
| T004ForecastBatchInfo | Batch metadata |
| T004ForecastLock | Forecast locks |
| PowerPlant | Power plant info (temporal) |
| T000EntitySystemHierarchy | Unit hierarchy |
| T000EntityPermission | User permissions |

### Stored Procedures

| Name | Purpose |
|------|---------|
| sp004get_root_forecast_use_pointoftime | Date-based retrieval |
| sp004get_root_forecast_use_deliverystartdatetimebefore | Offset retrieval |

---

## Quick Start

```bash
# Build
dotnet build

# Run
cd SmartPulse.Web.Services
dotnet run

# API available at
http://localhost:5000/swagger
```

---

## Documentation Structure

```
Documentation/
├── PROJECT_SUMMARY.md              # This file
├── DOCUMENTATION_INDEX.md          # Navigation guide
├── docs/
│   ├── architecture/               # System design
│   ├── components/production_forecast/  # Component details
│   ├── data/                       # Database docs
│   ├── patterns/                   # Design patterns
│   └── guides/                     # Developer guides
└── notes/                          # Analysis notes
```

---

**Document Version**: 2.0
**Last Review**: 2025-11-28
