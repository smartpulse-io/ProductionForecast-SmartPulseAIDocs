# ProductionForecast Service

**Version**: 2.0
**Framework**: .NET 9.0
**Last Updated**: 2025-11-28

---

## Overview

The ProductionForecast service is a REST API for managing energy production forecasts for power plants, companies, and groups. It provides endpoints for saving and retrieving time-series forecast data with two-level caching optimized for high read-throughput scenarios.

### Key Capabilities

| Feature | Description |
|---------|-------------|
| **Forecast Management** | Save, retrieve, and manage energy production forecasts (MWh values) |
| **Two-Level Caching** | Output Cache + Memory Cache for optimal performance |
| **Authorization** | Fine-grained access control at unit and user level |
| **CDC Invalidation** | Real-time cache invalidation using SQL Server Change Tracking |
| **API Versioning** | URL-based versioning (v1.0 deprecated, v2.0 active) |

### Performance

| Operation | Throughput | Latency |
|-----------|-----------|---------|
| GET (cache hit) | 10K+ req/sec | <5ms |
| GET (cache miss) | 500+ req/sec | 50-200ms |
| POST (save) | 100+ req/sec | 50-500ms |

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     CLIENT                                   │
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
│  │ (IMemoryCache)│ │(Business Logic)│  │  (6 active)     │   │
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
│  ┌──────────────┐  ┌───────────────┐                        │
│  │   Tables     │  │Change Tracking│                        │
│  └──────────────┘  └───────────────┘                        │
└─────────────────────────────────────────────────────────────┘
```

---

## Two-Level Caching Strategy

```
┌────────────────────────────────────────────┐
│ L1: Output Cache (ASP.NET Core)            │
│ Scope: HTTP responses                       │
│ Latency: <5ms                               │
│ TTL: 60 minutes                             │
│ Invalidation: Tag-based via CDC             │
└────────────────────────────────────────────┘
          ↓ (miss)
┌────────────────────────────────────────────┐
│ L2: Memory Cache (IMemoryCache)            │
│ Scope: Application data                     │
│ Latency: <1ms                               │
│ TTL: 1-1440 minutes (configurable)          │
│ Invalidation: CancellationToken via CDC     │
└────────────────────────────────────────────┘
          ↓ (miss)
┌────────────────────────────────────────────┐
│ Database (SQL Server)                      │
│ Latency: 50-200ms                           │
└────────────────────────────────────────────┘
```

**Note**: This service does NOT use Redis or any distributed cache. It uses local caching only.

---

## API Endpoints

### Base URL

`/api/v{version}/production-forecast`

### v2.0 Endpoints (Current)

| Method | Endpoint | Description | Cached |
|--------|----------|-------------|--------|
| POST | `/{providerKey}/{unitType}/{unitNo}/forecasts` | Save forecasts | No |
| GET | `/{providerKey}/{unitType}/{unitNo}/forecasts/latest` | Get latest | Yes |
| GET | `/{providerKey}/{unitType}/{unitNo}/forecasts/latest-by-date` | Get by date | Yes |
| GET | `/{providerKey}/{unitType}/{unitNo}/forecasts/latest-by-production-time-offset` | Get by offset | Yes |
| POST | `/GetLatestMulti` | Multi-unit get | No |

### Cache Management

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/system/cache-manager/cache-types` | List cache types |
| POST | `/system/cache-manager/all/expire` | Clear all cache |
| POST | `/system/cache-manager/{cacheType}/expire` | Expire specific cache |

---

## CDC Trackers

| Tracker | Table | Interval | Cache Action |
|---------|-------|----------|--------------|
| T004ForecastLatestTracker | t004forecast_latest | 100ms | Output cache eviction |
| T000EntityPermissionsTracker | t000entity_permission | 10s | Memory cache |
| T000EntityPropertyTracker | t000entity_property | 10s | Memory cache |
| T000EntitySystemHierarchyTracker | t000entity_system_hierarchy | 10s | Memory cache |
| SysUserRolesTracker | SysUserRole | 10s | Memory cache |
| PowerPlantTracker | PowerPlant | 10s | Memory cache |

---

## Component Documentation

| Document | Description |
|----------|-------------|
| [Web API Layer](./web_api_layer.md) | Controllers, middleware, API versioning |
| [Business Logic & Caching](./business_logic_caching.md) | Services, CacheManager |
| [Data Layer & Entities](./data_layer_entities.md) | EF Core, repositories, schema |
| [HTTP Client & Models](./http_client_models.md) | DTOs, serialization |

---

## Configuration

### Key Settings

```json
{
  "CacheSettings": {
    "OutputCache": {
      "UseCacheInvalidationChangeTracker": true,
      "Duration": 60
    },
    "MemoryCache": {
      "GipConfigDuration": 60,
      "HierarchyDuration": 60
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

## Related Documentation

- [System Overview](../../architecture/00_system_overview.md)
- [Architectural Patterns](../../architecture/architectural_patterns.md)
- [Caching Patterns](../../patterns/caching_patterns.md)
- [CDC Documentation](../../data/cdc.md)

---

**Last Updated**: 2025-11-28
**Version**: 2.0
