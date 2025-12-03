# ProductionForecast Service - Business Logic & Caching

**Component**: ProductionForecast
**Layer**: Application/Business Logic
**Assembly**: SmartPulse.Application
**Last Updated**: 2025-11-28

---

## Overview

The business logic layer orchestrates forecast operations, implements two-level caching strategy, and integrates Change Data Capture (CDC) for automatic cache invalidation. This layer coordinates between the HTTP API and database.

### Key Features

- **Two-Level Caching**: Output Cache (L1) + Memory Cache (L2)
- **CDC Integration**: Real-time database change tracking with automatic cache invalidation
- **Stampede Prevention**: SemaphoreSlim per cache key for thread-safe cache loads
- **Authorization**: Role-based and unit-level access control

### Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     Controller                               │
└─────────────────────────────┬───────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                   ForecastService                            │
│                 (Business Logic)                             │
└─────────────────────────────┬───────────────────────────────┘
                              │
          ┌───────────────────┼───────────────────┐
          │                   │                   │
          ▼                   ▼                   ▼
┌─────────────────┐ ┌─────────────────┐ ┌─────────────────┐
│  CacheManager   │ │ ForecastDbService│ │   Validation   │
│  (IMemoryCache) │ │   (EF Core)      │ │   Helpers      │
└─────────────────┘ └─────────────────┘ └─────────────────┘
          │                   │
          │                   ▼
          │         ┌─────────────────┐
          │         │   SQL Server    │
          │         │ Change Tracking │
          │         └─────────────────┘
          │                   │
          │                   ▼
          │         ┌─────────────────┐
          └─────────│  CDC Trackers   │
                    │ (6 active)      │
                    └─────────────────┘
```

**Note**: This service does NOT use Redis or Apache Pulsar. Cache invalidation is local only.

---

## ForecastService

### Purpose

Main orchestrator for forecast operations. Coordinates between HTTP API, business logic, and database.

### Class Definition

```csharp
public class ForecastService : IForecastService
{
    private readonly IForecastDbService _dbService;
    private readonly CacheManager _cache;
    private readonly ILogger<ForecastService> _logger;
    private readonly IHttpContextAccessor _httpContext;
}
```

**Lifetime**: Transient (new instance per request)

---

### Key Methods

#### 1. SaveForecastsAsync

**Signature**:
```csharp
public async Task<ForecastSaveResponseData> SaveForecastsAsync(
    string providerKey,
    string unitType,
    string unitNo,
    ForecastSaveRequestBody request,
    bool shouldReturnSaves = false,
    bool shouldSkipExistingCheck = false,
    CancellationToken cancellationToken = default)
```

**Flow**:

```
1. Authorization check (ValidateSaveRequest)
   └─ User has write access to unit
   └─ Provider key matches user's permissions

2. Validate request data
   └─ No duplicate forecasts
   └─ MWh values > 0.1
   └─ Delivery period valid

3. Check for duplicates (unless shouldSkipExistingCheck=true)
   └─ Query database for existing forecasts
   └─ Separate into "save" and "skip" lists

4. Save new forecasts
   └─ Call IForecastDbService.InsertForecastBatchAsync()
   └─ Bulk insert via EFCore.BulkExtensions
   └─ Insert to t004forecast_latest table

5. Cache invalidation (automatic via CDC)
   └─ CDC tracker detects change in t004forecast_latest
   └─ Output cache evicted by tags
   └─ Memory cache invalidated via CancellationToken

6. Return response
   └─ BatchId (unique identifier)
   └─ SavedCount (new records)
   └─ SkippedCount (duplicates)
```

**Time Complexity**: ~50-500ms depending on batch size

**Response Model**:
```csharp
public class ForecastSaveResponseData
{
    public string BatchId { get; set; }
    public int SavedCount { get; set; }
    public int SkippedCount { get; set; }
    public List<ForecastSaveData> Forecasts { get; set; }  // Optional
    public DateTime SavedAt { get; set; }
}
```

---

#### 2. GetForecastAsync

**Signature**:
```csharp
public async Task<ForecastGetLatestData> GetForecastAsync(
    string providerKey,
    string unitType,
    string unitNo,
    DateTime? from,
    DateTime? to,
    int period,
    CancellationToken cancellationToken = default)
```

**Flow**:

```
1. Authorization check
   └─ User has read access to unit

2. Determine query type
   ├─ Latest: Last saved forecast
   ├─ ByDate: For specific delivery date
   └─ ByOffset: For delivery starting N minutes from now

3. Query database (or cache)
   └─ Output cache checked first (via middleware)
   └─ Memory cache checked via CacheManager
   └─ Database query if cache miss

4. Post-process
   └─ ResolutionHelper.Normalize(forecasts, period)
   └─ TimeZoneHelper.Convert(forecasts, userTimeZone)

5. Return response
   └─ List of forecast items
   └─ Metadata (CreatedAt, ValidAfter)
```

**Time Complexity**:
- Cache hit: <5ms
- Cache miss: 50-200ms

---

#### 3. GetForecastMultiAsync

**Signature**:
```csharp
public async Task<Dictionary<string, ForecastGetLatestData>> GetForecastMultiAsync(
    ForecastGetLatestMultiRequest request,
    CancellationToken cancellationToken = default)
```

**Optimization**: Batch query instead of N individual queries

**Performance**: 10-50ms for multiple units (vs N * 50ms without batching)

---

## ForecastDbService

### Purpose

Low-level database operations. Encapsulates EF Core queries and commands.

**Lifetime**: Transient

### Class Definition

```csharp
public class ForecastDbService : IForecastDbService
{
    private readonly ForecastDbContext _dbContext;
    private readonly ILogger<ForecastDbService> _logger;
}
```

---

### Query Methods

#### GetPredictionsAsync

**Query Pattern**:
```csharp
var query = _dbContext.T004ForecastLatest
    .AsNoTracking()  // Read-only optimization
    .Where(f => f.ProviderKey == providerKey
        && f.UnitType == unitType
        && f.UnitNo == unitNo
        && f.DeliveryStart >= from
        && f.DeliveryEnd <= to)
    .OrderByDescending(f => f.DeliveryStart);

return await query.ToListAsync(ct);
```

**Indexes Used**:
- Composite: `(ProviderKey, UnitType, UnitNo, DeliveryStart DESC)`
- Query time: 20-50ms without cache

---

### Command Methods

#### InsertForecastBatchAsync

**Flow**:

```
1. Create batch info (BatchId, RecordCount, CreatedBy)
2. Map forecasts to T004Forecast entities
3. Bulk insert via EFCore.BulkExtensions
   └─ Batch size: 2000 records
4. Upsert to t004forecast_latest
5. Return BatchId
```

**Performance**: ~50-100ms for 1000-5000 records

---

## CacheManager

### Purpose

Thread-safe in-memory caching with automatic invalidation via CDC.

### Two-Level Cache Architecture

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

**Note**: There is NO Redis or distributed cache. This is local caching only.

### Class Definition

```csharp
public class CacheManager
{
    private readonly IMemoryCache _memoryCache;
    private readonly ConcurrentDictionary<string, SemaphoreSlim> _semaphores = new();
    private readonly ConcurrentDictionary<string, CancellationTokenSource> _expirationTokens = new();
}
```

**Lifetime**: Singleton (shared across all requests)

---

### Cache Keys

| Key Pattern | TTL | Purpose |
|-------------|-----|---------|
| `powerplant_all_hierarchies` | 60 min | Unit hierarchy data |
| `all_powerplant_timezones` | 24 hours | Timezone mappings |
| `user_accessible_units_{userId}_{unitType}` | 24 hours | User permissions |
| `company_limitsettings_{companyId}` | 1 min | Forecast limits |

---

### Get Operations with Stampede Prevention

```csharp
public async Task<T?> GetOrCreateAsync<T>(
    string key,
    Func<Task<T>> factory,
    TimeSpan ttl)
{
    // Fast path: Check cache first
    if (_memoryCache.TryGetValue(key, out T cached))
        return cached;

    // Get or create semaphore for this key
    var semaphore = _semaphores.GetOrAdd(key, _ => new SemaphoreSlim(1, 1));
    await semaphore.WaitAsync();

    try
    {
        // Double-check after acquiring lock
        if (_memoryCache.TryGetValue(key, out cached))
            return cached;

        // Load from database
        var value = await factory();

        // Setup expiration token for manual invalidation
        var cts = GetNewOrExistingExpirationTokenSource(key);

        _memoryCache.Set(key, value, new MemoryCacheEntryOptions
        {
            AbsoluteExpirationRelativeToNow = ttl,
            ExpirationTokens = { new CancellationChangeToken(cts.Token) }
        });

        return value;
    }
    finally
    {
        semaphore.Release();
    }
}
```

**Benefits**:
- Prevents cache stampede (multiple simultaneous cache misses)
- Thread-safe via SemaphoreSlim per key
- Manual expiration via CancellationTokenSource

---

### Invalidation Operations

```csharp
public void ExpireCacheByKey(string key)
{
    if (_expirationTokens.TryRemove(key, out var cts))
    {
        cts.Cancel();  // Triggers cache entry removal
        cts.Dispose();
    }
}
```

---

## Change Data Capture (CDC)

### Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                   SQL Server                                 │
│                Change Tracking                               │
└─────────────────────────────┬───────────────────────────────┘
                              │ CHANGETABLE queries
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                   CDC Trackers                               │
│  ┌─────────────────────────────────────────────────────┐    │
│  │ T004ForecastLatestTracker (100ms poll)              │    │
│  │ T000EntityPermissionsTracker (10s poll)             │    │
│  │ T000EntityPropertyTracker (10s poll)                │    │
│  │ T000EntitySystemHierarchyTracker (10s poll)         │    │
│  │ SysUserRolesTracker (10s poll)                      │    │
│  │ PowerPlantTracker (10s poll)                        │    │
│  └─────────────────────────────────────────────────────┘    │
└─────────────────────────────┬───────────────────────────────┘
                              │ OnChangeAction
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                   Cache Invalidation                         │
│  ┌──────────────────┐     ┌──────────────────┐              │
│  │ Output Cache     │     │ Memory Cache     │              │
│  │ EvictByTagAsync()│     │ ExpireCacheByKey │              │
│  └──────────────────┘     └──────────────────┘              │
└─────────────────────────────────────────────────────────────┘
```

**Note**: There is NO Pulsar event publishing. Cache invalidation is local only.

### 6 CDC Trackers

| Tracker | Table | Interval | Cache Action |
|---------|-------|----------|--------------|
| T004ForecastLatestTracker | t004forecast_latest | 100ms | Output cache eviction |
| T000EntityPermissionsTracker | t000entity_permission | 10s | Memory cache |
| T000EntityPropertyTracker | t000entity_property | 10s | Memory cache |
| T000EntitySystemHierarchyTracker | t000entity_system_hierarchy | 10s | Memory cache |
| SysUserRolesTracker | SysUserRole | 10s | Memory cache |
| PowerPlantTracker | PowerPlant | 10s | Memory cache |

---

### T004ForecastLatestTracker

**Purpose**: Track forecast changes and invalidate output cache

```csharp
public class T004ForecastLatestTracker : BaseTracker
{
    private readonly IOutputCacheStore _outputCacheStore;

    protected override string TrackerName => "T004ForecastLatestTracker";
    protected override int IntervalMs => 100; // Fast polling

    public override string TableName => "t004forecast_latest";

    protected override async Task OnChangeAction(List<ChangeItem> changes, Guid traceId)
    {
        foreach (var change in changes)
        {
            // Generate cache tag from change data
            var tag = DataTagHelper.GenerateTag(
                change.PkColumns["UnitType"]?.ToString(),
                change.PkColumns["UnitNo"]?.ToString(),
                change.PkColumns["ProviderKey"]?.ToString(),
                change.PkColumns["Period"]?.ToString(),
                change.PkColumns["DeliveryDate"]?.ToString());

            // Evict cached responses matching this tag
            await _outputCacheStore.EvictByTagAsync(tag, CancellationToken.None);
        }
    }
}
```

---

### CDC Flow Example

```
1. Forecast saved to t004forecast_latest table
2. SQL Server Change Tracking records change
3. T004ForecastLatestTracker polls CHANGETABLE (100ms interval)
4. Change detected → OnChangeAction called
5. Cache invalidation:
   - Output cache: EvictByTagAsync(tag)
   - Tag format: {unitType}.{unitNo}.{providerKey}.{period}
6. Next GET request → cache miss → fresh data from database
```

**End-to-End Latency**: ~100-150ms from database change to cache invalidation

---

## Business Logic Flows

### Flow 1: Save Forecast (Write Path)

**Endpoint**: `POST /api/v2/production-forecast/{providerKey}/{unitType}/{unitNo}/forecasts`

**Total Time**: 50-500ms (depending on batch size)

#### Step 1: Authorization Check (ValidateSaveRequest)

The system validates that the user has permission to save forecasts for this unit.

| Check | Description | Error Response |
|-------|-------------|----------------|
| User has write access to unit | Verifies user's role includes write permission for the specified `unitNo` | 403 Forbidden |
| Provider key matches user's permissions | Ensures the `providerKey` is authorized for this user | 403 Forbidden |
| Unit is not locked | Checks `T004ForecastLock` table for active locks on the delivery period | 400 Bad Request (Unit Locked) |

**Code Path**: `Controller.SaveForecasts()` → `ValidateSaveRequest()`

#### Step 2: Validate Request Data

Input validation ensures data integrity before database operations.

| Validation | Rule | Error Response |
|------------|------|----------------|
| No duplicate forecasts | Same `DeliveryStart` cannot appear twice in request | 400 Bad Request |
| Value range | `PredictionValue` must be >= 0 | 400 Bad Request |
| Delivery period valid | `DeliveryStart` < `DeliveryEnd` | 400 Bad Request |
| Period alignment | `DeliveryStart` must align with allowed periods (5, 10, 15, 30, 60 min) | 400 Bad Request |
| Future delivery | `DeliveryStart` must be in the future (configurable) | 400 Bad Request |

**Code Path**: `ForecastService.SaveForecastsAsync()` → Model validation attributes

#### Step 3: Check for Duplicates (Optional)

Unless `shouldSkipExistingCheck=true`, the system queries for existing forecasts.

| Action | Description |
|--------|-------------|
| Query existing forecasts | `SELECT` from `t004forecast_latest` where `UnitNo`, `ProviderKey`, `DeliveryStart` match |
| Separate into lists | Forecasts that already exist → "skip" list; New forecasts → "save" list |
| Skip existing | Existing forecasts are not overwritten (unless explicitly requested) |

**Performance**: 10-50ms for database query

#### Step 4: Save New Forecasts

Bulk insert operation using EFCore.BulkExtensions for optimal performance.

| Operation | Description | Table |
|-----------|-------------|-------|
| Create batch info | Generate `BatchId` (GUID), record metadata | `t004forecast_batch_info` |
| Insert to staging | Bulk insert forecasts to staging table | `t004forecast_batch_insert` |
| Trigger processes | Database trigger moves data to main table | `t004forecast` |
| Upsert to latest | Update or insert to latest forecasts table | `t004forecast_latest` |

**Code Path**: `ForecastDbService.InsertForecastBatchAsync()`

**Performance**:
- Batch size: 2000 records per batch
- Insert time: ~50-100ms for 1000-5000 records
- Trigger execution: ~10-50ms

#### Step 5: Cache Invalidation (Automatic via CDC)

SQL Server Change Tracking triggers automatic cache invalidation.

| Step | Description | Latency |
|------|-------------|---------|
| Change recorded | SQL Server Change Tracking captures the INSERT/UPDATE | Immediate |
| CDC Tracker polls | `T004ForecastLatestTracker` queries `CHANGETABLE()` | 100ms interval |
| Change detected | Tracker receives list of changed rows with PK values | - |
| Generate cache tag | Create tag from `UnitType.UnitNo.ProviderKey.Period` | - |
| Evict Output Cache | `IOutputCacheStore.EvictByTagAsync(tag)` | <5ms |
| Memory cache | CancellationToken triggered for related cache keys | <1ms |

**End-to-End Invalidation Latency**: 100-150ms

#### Step 6: Return Response

| Field | Type | Description |
|-------|------|-------------|
| `batchId` | `Guid` | Unique identifier for this batch operation |
| `savedCount` | `int` | Number of new forecasts saved |
| `skippedCount` | `int` | Number of duplicate forecasts skipped |
| `forecasts` | `List<ForecastSaveData>` | (Optional) Saved forecast details if `shouldReturnSaves=true` |

**Response Model**:
```json
{
  "statusCode": 200,
  "isError": false,
  "data": {
    "batchId": "3a2b1c4d-5e6f-7a8b-9c0d-1e2f3a4b5c6d",
    "savedCount": 24,
    "skippedCount": 0,
    "savedAt": "2024-01-15T10:30:00+03:00"
  }
}
```

#### Save Forecast Sequence Diagram

```mermaid
sequenceDiagram
    autonumber
    participant Client
    participant Controller
    participant Service as ForecastService
    participant DB as Database
    participant CDC as CDC Tracker
    participant Cache as Output Cache

    Client->>Controller: POST /forecasts
    Controller->>Controller: ValidateSaveRequest()

    Note over Controller: Step 1: Authorization Check
    Controller->>Service: SaveForecastsAsync()

    Note over Service: Step 2: Validate Request Data
    Service->>Service: Validate predictions

    Note over Service,DB: Step 3: Check for Duplicates
    Service->>DB: Query existing forecasts
    DB-->>Service: Return existing list
    Service->>Service: Separate save/skip lists

    Note over Service,DB: Step 4: Save New Forecasts
    Service->>DB: InsertForecastBatchAsync()
    DB->>DB: INSERT t004forecast_batch_insert
    DB->>DB: Trigger → INSERT t004forecast
    DB->>DB: UPSERT t004forecast_latest
    DB->>DB: Change Tracking records change
    DB-->>Service: Return BatchId

    Note over Service,Client: Step 6: Return Response
    Service-->>Controller: Return BatchForecast
    Controller-->>Client: 200 OK {batchId, savedCount, skippedCount}

    Note over CDC,Cache: Step 5: Cache Invalidation (Async, within 100ms)
    CDC->>DB: Poll CHANGETABLE (100ms interval)
    DB-->>CDC: Return changes
    CDC->>CDC: OnChangeAction()
    CDC->>Cache: EvictByTagAsync(tag)
    Cache-->>CDC: Cache evicted
```

---

### Flow 2: Get Forecast (Read Path)

**Endpoint**: `GET /api/v2/production-forecast/{providerKey}/{unitType}/{unitNo}/forecasts/latest`

**Query Parameters**:
- `from`: Start date (ISO 8601)
- `to`: End date (ISO 8601)
- `period`: Minutes (5, 10, 15, 30, 60)
- `no-details`: Boolean (exclude metadata)

#### Step 1: Output Cache Check (L1)

ASP.NET Core Output Cache middleware checks for cached response.

| Scenario | Action | Latency |
|----------|--------|---------|
| Cache HIT | Return cached HTTP response directly | <5ms |
| Cache MISS | Forward request to controller | - |

**Cache Key**: Generated from URL path + query parameters + user context
**Cache Policy**: `ForecastPolicy` with tag-based invalidation

#### Step 2: Authorization Check

| Check | Description | Error Response |
|-------|-------------|----------------|
| User has read access | Verifies user's role includes read permission for `unitNo` | 403 Forbidden |
| Unit exists | Validates `unitNo` exists in `PowerPlant` table | 404 Not Found |

**Code Path**: `Controller.GetLatestForecasts()` → Authorization middleware

#### Step 3: Memory Cache Check (L2)

`CacheManager.GetOrCreateAsync()` implements thread-safe caching with stampede prevention.

| Scenario | Action | Latency |
|----------|--------|---------|
| Cache HIT | Return cached data from `IMemoryCache` | <1ms |
| Cache MISS | Acquire `SemaphoreSlim` lock, query database | - |

**Stampede Prevention**: Only one thread queries database; others wait for result

#### Step 4: Database Query

| Operation | Description | Performance |
|-----------|-------------|-------------|
| Query TVF | Call `tb004get_munit_forecasts_latest()` | 50-200ms |
| AsNoTracking | Read-only query optimization | 15-30% faster |
| Index usage | Uses composite index on `(ProviderKey, UnitType, UnitNo, DeliveryStart)` | - |

**Code Path**: `ForecastDbService.GetPredictionsAsync()`

#### Step 5: Store in Cache

| Cache Level | Storage | TTL |
|-------------|---------|-----|
| L2 (Memory) | `IMemoryCache.Set()` with `CancellationChangeToken` | 60 min (configurable) |
| L1 (Output) | Response cached by middleware with tags | 60 min |

**Cache Tags**: `{unitType}.{unitNo}.{providerKey}.{period}` for targeted invalidation

#### Step 6: Return Response

**Response Model**:
```json
{
  "statusCode": 200,
  "isError": false,
  "data": {
    "unitNo": 123,
    "period": 60,
    "predictions": [
      {
        "deliveryStart": "2024-01-15T00:00:00+03:00",
        "value": 450.5,
        "measureUnit": 1
      },
      {
        "deliveryStart": "2024-01-15T01:00:00+03:00",
        "value": 455.2,
        "measureUnit": 1
      }
    ]
  }
}
```

**Performance Summary**:

| Path | Latency |
|------|---------|
| Output Cache hit (L1) | <5ms |
| Memory Cache hit (L2) | <10ms |
| Database query (cache miss) | 50-200ms |

#### Get Forecast Sequence Diagram

```mermaid
sequenceDiagram
    autonumber
    participant Client
    participant OutputCache as Output Cache (L1)
    participant Controller
    participant Service as ForecastService
    participant Cache as CacheManager (L2)
    participant DB as Database

    Client->>OutputCache: GET /forecasts/latest

    Note over OutputCache: Step 1: Output Cache Check
    OutputCache->>OutputCache: Check cache by key + tags

    alt Output Cache HIT
        OutputCache-->>Client: 200 OK (cached) [<5ms]
    else Output Cache MISS
        OutputCache->>Controller: Forward request

        Note over Controller: Step 2: Authorization Check
        Controller->>Controller: Validate user access
        Controller->>Service: GetForecastAsync()

        Note over Service,Cache: Step 3: Memory Cache Check
        Service->>Cache: GetOrCreateAsync(key, factory)
        Cache->>Cache: TryGetValue(key)

        alt Memory Cache HIT
            Cache-->>Service: Return cached data [<1ms]
        else Memory Cache MISS
            Cache->>Cache: Acquire SemaphoreSlim lock
            Cache->>Cache: Double-check after lock

            Note over Cache,DB: Step 4: Database Query
            Cache->>DB: tb004get_munit_forecasts_latest()
            DB-->>Cache: Return forecasts [50-200ms]

            Note over Cache: Step 5: Store in Cache
            Cache->>Cache: Set with CancellationChangeToken
            Cache->>Cache: Release lock
            Cache-->>Service: Return data
        end

        Service-->>Controller: Return UnitForecast

        Note over Controller,OutputCache: Step 5: Store in Output Cache
        Controller-->>OutputCache: 200 OK + cache headers
        OutputCache->>OutputCache: Store with tags

        Note over OutputCache,Client: Step 6: Return Response
        OutputCache-->>Client: 200 OK
    end
```

---

## State Diagrams

### Cache Entry State Diagram

The following state diagram shows the lifecycle of a cache entry in the two-level cache system:

```mermaid
stateDiagram-v2
    [*] --> EMPTY

    EMPTY --> LOADING: Request (cache miss)
    LOADING --> CACHED: Data loaded from database

    CACHED --> SERVING: Request (cache hit)
    SERVING --> IDLE: Response sent
    IDLE --> CACHED: Next request

    CACHED --> INVALIDATED: CDC detects change
    IDLE --> INVALIDATED: CDC detects change
    IDLE --> EXPIRED: TTL expires

    INVALIDATED --> EMPTY: Entry removed
    EXPIRED --> EMPTY: Entry removed

    note right of EMPTY: No cache entry exists
    note right of LOADING: Acquiring lock, querying DB\n50-200ms
    note right of CACHED: Entry stored in L1 and/or L2
    note right of SERVING: Returning cached response\n<5ms
    note right of IDLE: Waiting for TTL or next request
    note right of INVALIDATED: CDC triggered removal
    note right of EXPIRED: TTL elapsed
```

**State Descriptions**:

| State | Description | Duration |
|-------|-------------|----------|
| EMPTY | No cache entry exists | - |
| LOADING | Acquiring lock, querying database | 50-200ms |
| CACHED | Entry stored in L1 and/or L2 cache | Until TTL or invalidation |
| SERVING | Returning cached response | <5ms |
| IDLE | Cached but not being accessed | Until TTL or invalidation |
| INVALIDATED | CDC detected change, entry removed | Immediate |
| EXPIRED | TTL elapsed, entry removed | Immediate |

---

### Request Processing State Diagram

```mermaid
stateDiagram-v2
    [*] --> RECEIVED: Request arrives

    RECEIVED --> AUTHORIZING: Process request

    AUTHORIZING --> CHECKING_L1: Authorized
    AUTHORIZING --> REJECTED: Unauthorized

    REJECTED --> [*]: 401/403

    CHECKING_L1 --> RESPONDING_L1: L1 HIT
    CHECKING_L1 --> CHECKING_L2: L1 MISS

    RESPONDING_L1 --> COMPLETED: <5ms

    CHECKING_L2 --> RESPONDING_L2: L2 HIT
    CHECKING_L2 --> QUERYING_DB: L2 MISS

    RESPONDING_L2 --> COMPLETED: <1ms

    QUERYING_DB --> CACHING: Data returned (50-200ms)
    CACHING --> COMPLETED: Store in L1 + L2

    COMPLETED --> [*]: 200 OK

    note right of CHECKING_L1: Output Cache
    note right of CHECKING_L2: Memory Cache
    note right of QUERYING_DB: SQL Server
```

---

## User Flow Diagram

### End-to-End User Flow: Forecast Operations

```mermaid
flowchart TD
    subgraph Authentication
        A[USER / Client] --> B{Login / Get Token}
        B -->|Invalid Token| C[Error 401]
        B -->|Valid Token| D{Choose Operation}
    end

    subgraph Operations
        D --> E[GET Forecast<br/>Read]
        D --> F[POST Save<br/>Forecasts]
        D --> G[ADMIN<br/>Cache Operations]
    end

    subgraph GET Flow
        E --> H[Validate Parameters<br/>provider, unitType,<br/>unitNo, period, from/to]
        H -->|Invalid| I[Error 400<br/>Bad Request]
        H -->|Valid| J{Check Unit Access}
        J -->|No Access| K[Error 403<br/>Forbidden]
        J -->|Has Access| L[Cache Lookup<br/>L1 → L2 → DB]
        L --> M[200 OK<br/>Forecast Data]
    end

    subgraph POST Flow
        F --> N[Validate Request<br/>period, values, dates]
        N -->|Invalid| O[Error 400<br/>Bad Request]
        N -->|Valid| P{Check Unit Access<br/>& Lock Status}
        P -->|No Access/Locked| Q[Error 400<br/>Unit Locked]
        P -->|Has Access| R[Prepare Batch Insert]
        R --> S[Bulk Insert to Database]
        S --> T[CDC Detects Change]
        T --> U[Cache Invalidated<br/>by tag]
        U --> V[200 OK<br/>BatchId, SaveCount]
    end

    subgraph Admin Flow
        G --> W{Validate Admin Role}
        W -->|Not Admin| X[Error 403<br/>Forbidden]
        W -->|Is Admin| Y[Select Cache Type<br/>to Expire]
        Y --> Z[Expire Cache Entry]
        Z --> U
    end

    style A fill:#e1f5fe
    style M fill:#c8e6c9
    style V fill:#c8e6c9
    style C fill:#ffcdd2
    style I fill:#ffcdd2
    style K fill:#ffcdd2
    style O fill:#ffcdd2
    style Q fill:#ffcdd2
    style X fill:#ffcdd2
```

**User Flow Descriptions**:

| Operation | Steps | Expected Time |
|-----------|-------|---------------|
| GET Forecast (cache hit) | Auth → Validate → Cache Lookup → Response | <10ms |
| GET Forecast (cache miss) | Auth → Validate → DB Query → Cache Store → Response | 50-200ms |
| POST Save Forecast | Auth → Validate → Lock Check → Bulk Insert → CDC → Response | 50-500ms |
| Admin Cache Expire | Auth → Admin Check → Expire Cache → Response | <10ms |

---

## Performance Metrics

| Operation | Throughput | Latency P50 | Latency P99 |
|-----------|-----------|------------|------------|
| Save forecast (1000 items) | 100/sec | 100ms | 300ms |
| Get latest (cache hit) | 10K+/sec | <5ms | 10ms |
| Get latest (cache miss) | 500/sec | 50ms | 200ms |
| CDC detection | - | 100ms | 150ms |
| Cache invalidation | - | <5ms | 10ms |

---

## Related Documentation

- [System Overview](../../architecture/00_system_overview.md)
- [Caching Patterns](../../patterns/caching_patterns.md)
- [CDC Documentation](../../data/cdc.md)

---

**Last Updated**: 2025-11-28
**Version**: 2.0
