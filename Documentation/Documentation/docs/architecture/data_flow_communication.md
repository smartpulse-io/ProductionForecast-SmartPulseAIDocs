# Data Flow & Communication - SmartPulse.Services.ProductionForecast

**Version**: 2.0
**Last Updated**: 2025-11-28
**Status**: Current

---

## Table of Contents

1. [Overview](#overview)
2. [Request-Response Flow](#request-response-flow)
3. [API Contract Specifications](#api-contract-specifications)
4. [Cache Invalidation Flow](#cache-invalidation-flow)
5. [Save Forecast Flow](#save-forecast-flow)
6. [CDC-Based Data Synchronization](#cdc-based-data-synchronization)
7. [Performance Characteristics](#performance-characteristics)

---

## Overview

This document details the **data flows**, **API contracts**, and **cache invalidation patterns** for the SmartPulse.Services.ProductionForecast service.

### Key Communication Patterns

| Pattern | Technology | Use Case | Latency |
|---------|------------|----------|---------|
| **Synchronous API** | HTTP/REST | Client-server requests | 1-50ms (with cache) |
| **Output Cache** | ASP.NET Core Output Cache | HTTP response caching | <5ms |
| **Memory Cache** | IMemoryCache | Application data caching | <1ms |
| **CDC Polling** | SQL Server Change Tracking | Cache invalidation triggers | 100ms-10s |

---

## Request-Response Flow

### GET Forecast Request Flow

```
┌─────────────────────────────────────────────────────────────┐
│                      CLIENT REQUEST                          │
│     GET /api/v2/production-forecast/{provider}/{type}/      │
│              {unitNo}/forecasts/latest                       │
└─────────────────────────────┬───────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                     OUTPUT CACHE CHECK                       │
│              ASP.NET Core OutputCache (L1)                   │
│              Tag: {unitType}.{unitNo}.{provider}.{period}    │
├─────────────────────────────────────────────────────────────┤
│  HIT (70-90%): Return cached HTTP response (<5ms)           │
│  MISS: Continue to controller                                │
└─────────────────────────────┬───────────────────────────────┘
                              │ (cache miss)
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                     CONTROLLER LAYER                         │
│                ProductionForecastController                  │
│              Authorization + Parameter Validation            │
└─────────────────────────────┬───────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    APPLICATION LAYER                         │
│                     ForecastService                          │
├─────────────────────────────────────────────────────────────┤
│  1. Check user authorization for unit                        │
│  2. Check Memory Cache (CacheManager)                        │
│  3. If miss: Query repository                                │
└─────────────────────────────┬───────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    MEMORY CACHE CHECK                        │
│              IMemoryCache via CacheManager (L2)              │
│              SemaphoreSlim per key (stampede prevention)     │
├─────────────────────────────────────────────────────────────┤
│  HIT (80-95%): Return cached data (<1ms)                    │
│  MISS: Continue to database                                  │
└─────────────────────────────┬───────────────────────────────┘
                              │ (cache miss)
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                      DATA LAYER                              │
│            ForecastRepository + DbContext                    │
├─────────────────────────────────────────────────────────────┤
│  - Execute stored procedure or EF Core query                 │
│  - AsNoTracking for read-only queries                        │
│  - Return results + populate cache                           │
└─────────────────────────────┬───────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                      SQL SERVER                              │
│                   t004forecast_latest                        │
│              Stored Procedures for complex queries           │
└─────────────────────────────────────────────────────────────┘
```

### Cache Hit Probability

| Cache Level | Expected Hit Rate | Latency |
|-------------|-------------------|---------|
| Output Cache (L1) | 70-90% | <5ms |
| Memory Cache (L2) | 80-95% | <1ms |
| Database | 5-20% | 50-200ms |

---

## API Contract Specifications

### Base URL

`/api/v{version}/production-forecast`

### Endpoint: GET /{providerKey}/{unitType}/{unitNo}/forecasts/latest

**Request:**
```http
GET /api/v2/production-forecast/{providerKey}/{unitType}/{unitNo}/forecasts/latest?period=15&from=2025-01-01&to=2025-01-02 HTTP/1.1
Authorization: Bearer {jwt_token}
```

**Path Parameters:**
| Parameter | Type | Description |
|-----------|------|-------------|
| providerKey | string | Provider identifier |
| unitType | string | Unit type (e.g., "UEVM", "UEVS") |
| unitNo | string | Unit number |

**Query Parameters:**
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| period | int | Yes | Forecast period (5, 10, 15, 30, 60 minutes) |
| from | datetime | Yes | Start date |
| to | datetime | Yes | End date |

**Response 200 OK:**
```json
{
  "statusCode": 200,
  "isError": false,
  "message": null,
  "data": [
    {
      "deliveryStartTime": "2025-01-01T00:00:00Z",
      "deliveryEndTime": "2025-01-01T00:15:00Z",
      "value": 125.5,
      "createdAt": "2024-12-31T23:45:00Z"
    }
  ],
  "traceId": "abc123-def456"
}
```

**Response 401 Unauthorized:**
```json
{
  "statusCode": 401,
  "isError": true,
  "message": "Unauthorized access",
  "data": null,
  "traceId": "abc123-def456"
}
```

**Response 400 Bad Request:**
```json
{
  "statusCode": 400,
  "isError": true,
  "message": "Invalid period. Allowed periods: 5, 10, 15, 30, 60",
  "data": null,
  "traceId": "abc123-def456"
}
```

**Caching Headers:**
```http
Cache-Control: max-age=3600, public
X-Cache-Status: HIT|MISS
```

### Endpoint: POST /{providerKey}/{unitType}/{unitNo}/forecasts

**Request:**
```http
POST /api/v2/production-forecast/{providerKey}/{unitType}/{unitNo}/forecasts HTTP/1.1
Content-Type: application/json
Authorization: Bearer {jwt_token}

{
  "period": 15,
  "forecasts": [
    {
      "deliveryStartTime": "2025-01-01T00:00:00Z",
      "deliveryEndTime": "2025-01-01T00:15:00Z",
      "value": 125.5
    }
  ]
}
```

**Response 200 OK:**
```json
{
  "statusCode": 200,
  "isError": false,
  "message": "Forecasts saved successfully",
  "data": {
    "batchId": "uuid",
    "savedCount": 96
  },
  "traceId": "abc123-def456"
}
```

**Side Effects:**
- Data inserted to `t004forecast` table
- Data upserted to `t004forecast_latest` table
- CDC detects change → Output cache evicted via tags

### Endpoint: GET /system/cache-manager/cache-types

**Request:**
```http
GET /api/v2/system/cache-manager/cache-types HTTP/1.1
Authorization: Bearer {admin_token}
```

**Response 200 OK:**
```json
{
  "statusCode": 200,
  "isError": false,
  "data": [
    "GipConfig",
    "Hierarchy",
    "Region",
    "Timezone",
    "UserAccessibleUnits",
    "CompanyLimitSettings"
  ]
}
```

### Endpoint: POST /system/cache-manager/{cacheType}/expire

**Request:**
```http
POST /api/v2/system/cache-manager/Hierarchy/expire HTTP/1.1
Authorization: Bearer {admin_token}
```

**Response 200 OK:**
```json
{
  "statusCode": 200,
  "isError": false,
  "message": "Cache expired successfully"
}
```

---

## Cache Invalidation Flow

### CDC-Triggered Cache Invalidation

```
┌─────────────────────────────────────────────────────────────┐
│ STEP 1: DATABASE CHANGE                                     │
│                                                             │
│  Application saves forecast → INSERT/UPDATE t004forecast    │
│  Trigger copies to t004forecast_latest                      │
└─────────────────────────────┬───────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│ STEP 2: SQL SERVER CHANGE TRACKING                          │
│                                                             │
│  Change Tracking records change in CHANGETABLE              │
│  Version ID incremented                                     │
└─────────────────────────────┬───────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│ STEP 3: CDC TRACKER POLLS                                   │
│                                                             │
│  T004ForecastLatestTracker polls CHANGETABLE (100ms)        │
│  Detects new changes since last version                     │
└─────────────────────────────┬───────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│ STEP 4: IDENTIFY AFFECTED CACHE TAGS                        │
│                                                             │
│  Extract: unitType, unitNo, providerKey, period, date       │
│  Generate tags: "{unitType}.{unitNo}.{provider}.{period}"   │
└─────────────────────────────┬───────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│ STEP 5: OUTPUT CACHE EVICTION                               │
│                                                             │
│  IOutputCacheStore.EvictByTagAsync(tag)                     │
│  All cached responses with matching tags removed            │
└─────────────────────────────┬───────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│ STEP 6: NEXT REQUEST                                        │
│                                                             │
│  Cache miss → Fresh data fetched from database              │
│  Response cached with updated data                          │
└─────────────────────────────────────────────────────────────┘
```

### Invalidation Latency

| Step | Latency |
|------|---------|
| Database change | <10ms |
| Change Tracking record | <1ms |
| CDC poll interval | 100ms (T004ForecastLatestTracker) |
| Tag extraction | <1ms |
| Cache eviction | <5ms |
| **Total** | **~110-150ms** |

---

## Save Forecast Flow

### Complete Save Forecast Sequence

```
┌─────────────────────────────────────────────────────────────┐
│ STEP 1: API REQUEST                                         │
│                                                             │
│  POST /api/v2/production-forecast/{provider}/{type}/{no}/   │
│  Body: { period, forecasts[] }                              │
└─────────────────────────────┬───────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│ STEP 2: REQUEST VALIDATION                                  │
│                                                             │
│  - Validate period (5, 10, 15, 30, 60)                      │
│  - Validate user authorization for unit                     │
│  - Validate date range and forecast count                   │
│  - Check company limit settings                             │
└─────────────────────────────┬───────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│ STEP 3: LOCK CHECK                                          │
│                                                             │
│  Query T004ForecastLock for unit/date                       │
│  If locked → Return 400 "Forecast period is locked"         │
└─────────────────────────────┬───────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│ STEP 4: PREPARE BATCH                                       │
│                                                             │
│  Create BatchInfo with GUID                                 │
│  Map forecasts to T004Forecast entities                     │
│  Set timestamps, provider, unit info                        │
└─────────────────────────────┬───────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│ STEP 5: BULK INSERT                                         │
│                                                             │
│  EFCore.BulkExtensions.BulkInsertAsync()                    │
│  Batch size: 2000 records                                   │
│  Insert to t004forecast table                               │
└─────────────────────────────┬───────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│ STEP 6: UPSERT TO LATEST TABLE                              │
│                                                             │
│  Database trigger or stored procedure                       │
│  Upsert to t004forecast_latest                              │
│  (Keeps only latest forecast per delivery time)             │
└─────────────────────────────┬───────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│ STEP 7: CDC DETECTS & INVALIDATES                           │
│                                                             │
│  T004ForecastLatestTracker detects change                   │
│  Output cache evicted by tags                               │
│  Next GET request fetches fresh data                        │
└─────────────────────────────────────────────────────────────┘
```

### Batch Processing Details

| Parameter | Value |
|-----------|-------|
| Batch Size | 2000 records |
| Insert Method | BulkInsertAsync |
| Transaction | Per batch |
| Throughput | ~100+ batches/sec |

---

## CDC-Based Data Synchronization

### CDC Trackers in This Service

| Tracker | Table | Interval | Action |
|---------|-------|----------|--------|
| T004ForecastLatestTracker | t004forecast_latest | 100ms | Output cache eviction |
| T000EntityPermissionsTracker | t000entity_permission | 10s | Memory cache invalidation |
| T000EntityPropertyTracker | t000entity_property | 10s | Config cache invalidation |
| T000EntitySystemHierarchyTracker | t000entity_system_hierarchy | 10s | Hierarchy cache invalidation |
| SysUserRolesTracker | SysUserRole | 10s | User access cache invalidation |
| PowerPlantTracker | PowerPlant | 10s | Timezone cache invalidation |

### Memory Cache Invalidation Flow

For non-forecast data (hierarchies, permissions, config):

```
┌─────────────────────────────────────────────────────────────┐
│ 1. Database change (e.g., hierarchy update)                 │
└─────────────────────────────┬───────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│ 2. CDC Tracker polls CHANGETABLE (10s interval)             │
└─────────────────────────────┬───────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│ 3. OnChangeAction called with ChangeItem list               │
└─────────────────────────────┬───────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│ 4. CacheManager.ExpireCacheByKey(cacheKey)                  │
│    - Cancels CancellationTokenSource                        │
│    - Memory cache entry expires immediately                 │
└─────────────────────────────┬───────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│ 5. Next request triggers cache reload from database         │
└─────────────────────────────────────────────────────────────┘
```

---

## Performance Characteristics

### Data Flow Latency Summary

| Flow | Trigger | Latency | Guarantee |
|------|---------|---------|-----------|
| GET (cache hit) | Client request | <5ms | Strong (cached) |
| GET (cache miss) | Client request | 50-200ms | Strong (DB query) |
| POST (save) | Client request | 50-500ms | Durable (DB write) |
| Cache invalidation | Database change | 100-150ms | Eventually consistent |
| Config refresh | CDC tracker | 10-15s | Eventually consistent |

### Throughput

| Operation | Throughput | Notes |
|-----------|-----------|-------|
| GET (cache hit) | 10K+ req/sec | Output cache |
| GET (cache miss) | 500+ req/sec | DB query |
| POST (save) | 100+ req/sec | Bulk insert |
| CDC poll | 10+ queries/sec | Per tracker |

### Key System Characteristics

This service achieves **high performance** through:

1. **Two-level caching** - Output Cache + Memory Cache
2. **Tag-based eviction** - Efficient bulk invalidation
3. **CDC-based invalidation** - Real-time cache freshness
4. **Bulk insert** - High throughput writes
5. **SemaphoreSlim locking** - Stampede prevention

**Architecture Notes:**
- ✅ Single-instance deployment (no distributed cache needed)
- ✅ Local cache invalidation via CDC
- ✅ No message broker dependency
- ✅ Simplified operational model

---

## Related Documentation

- [Architectural Patterns](architectural_patterns.md) - Design decisions and trade-offs
- [System Overview](00_system_overview.md) - High-level system architecture
- [Caching Patterns](../patterns/caching_patterns.md) - Cache implementation details
- [CDC Documentation](../data/cdc.md) - Change tracking configuration

---

**Document Version**: 2.0
**Last Updated**: 2025-11-28
