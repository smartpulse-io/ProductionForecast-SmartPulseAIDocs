# Architectural Patterns - SmartPulse.Services.ProductionForecast

**Version**: 2.0
**Last Updated**: 2025-11-28
**Status**: Current

---

## Table of Contents

1. [Overview](#overview)
2. [Layered Architecture](#layered-architecture)
3. [Caching Patterns](#caching-patterns)
4. [CDC Pattern](#cdc-pattern)
5. [Repository Pattern](#repository-pattern)
6. [Background Service Patterns](#background-service-patterns)
7. [Error Handling Patterns](#error-handling-patterns)
8. [Design Decisions & Trade-Offs](#design-decisions--trade-offs)

---

## Overview

SmartPulse.Services.ProductionForecast implements a **layered architecture** with emphasis on:

- **Two-level caching** for performance optimization
- **CDC-based cache invalidation** for real-time consistency
- **Clean separation of concerns** across project layers
- **Thread-safe caching** with stampede prevention

### Key Technologies

| Technology | Purpose |
|------------|---------|
| **ASP.NET Core 9.0** | Web API framework |
| **Entity Framework Core 9.0** | ORM and database access |
| **SQL Server Change Tracking** | Change Data Capture |
| **IMemoryCache** | In-memory application cache |
| **Output Cache** | HTTP response caching |

---

## Layered Architecture

### Project Layer Structure

```
┌─────────────────────────────┐
│  Presentation Layer         │  REST endpoints, HTTP validation
│  (SmartPulse.Web.Services)  │
├─────────────────────────────┤
│  Application Layer          │  Business logic, orchestration
│  (SmartPulse.Application)   │  Cache management, CDC trackers
├─────────────────────────────┤
│  Repository/Data Access     │  Database queries, repository pattern
│  (SmartPulse.Repository)    │
├─────────────────────────────┤
│  Entity Layer               │  EF Core DbContext, entities
│  (SmartPulse.Entities)      │
├─────────────────────────────┤
│  Shared Layer               │  DTOs, configuration, constants
│  (SmartPulse.Models/Base)   │
└─────────────────────────────┘
```

### Layer Responsibilities

| Layer | Responsibility | Key Classes |
|-------|---------------|-------------|
| **Web.Services** | HTTP handling, routing, middleware | ProductionForecastController, ExceptionMiddleware |
| **Application** | Business logic, caching, CDC | ForecastService, CacheManager, CDC Trackers |
| **Repository** | Data access abstraction | ForecastRepository, CompanyPowerPlantRepository |
| **Entities** | Database schema | ForecastDbContext, T004Forecast |
| **Models** | DTOs, configuration | ApiResponse, AppSettings |
| **Base** | Constants, variables | SystemVariables |

### Benefits

- Clear responsibilities per layer
- Easy to test each layer independently
- Dependency Injection throughout
- Consistent patterns across the codebase

---

## Caching Patterns

### Two-Level Cache Strategy

```
Level 1: Output Cache (HTTP)
├── Scope: HTTP responses
├── TTL: 60 minutes
├── Invalidation: Tag-based via CDC
└── Implementation: ASP.NET Core OutputCache

Level 2: Memory Cache (Application)
├── Scope: Application data (hierarchies, config)
├── TTL: 1-1440 minutes (configurable per type)
├── Invalidation: CancellationToken via CDC
└── Implementation: IMemoryCache with SemaphoreSlim
```

### Cache-Aside Pattern with Stampede Prevention

**Implementation**: `SmartPulse.Application/CacheManager.cs`

```csharp
public async Task<T?> GetOrCreateAsync<T>(string key, Func<Task<T>> factory, TimeSpan ttl)
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

**Benefits:**
- Prevents cache stampede (multiple simultaneous DB queries)
- Thread-safe via SemaphoreSlim per key
- Manual expiration via CancellationTokenSource
- Double-checked locking for efficiency

### Output Cache Policy

**Implementation**: `SmartPulse.Web.Services/Policies/ForecastPolicy.cs`

```csharp
public class ForecastPolicy : IOutputCachePolicy
{
    public ValueTask CacheRequestAsync(OutputCacheContext context, CancellationToken ct)
    {
        // Generate tags based on route parameters
        var tags = DataTagHelper.GenerateTags(
            unitType, unitNo, providerKey, period, from, to);

        foreach (var tag in tags)
        {
            context.Tags.Add(tag);
        }

        context.EnableOutputCaching = true;
        context.AllowCacheLookup = true;
        context.AllowCacheStorage = true;

        return ValueTask.CompletedTask;
    }
}
```

**Tag Format**: `{unitType}.{unitNo}.{providerKey}.{period}.{date}`

---

## CDC Pattern

### SQL Server Change Tracking

SmartPulse uses SQL Server's built-in Change Tracking feature for detecting database changes.

```sql
-- Enable on database
ALTER DATABASE [ForecastDb]
SET CHANGE_TRACKING = ON
(CHANGE_RETENTION = 2 DAYS, AUTO_CLEANUP = ON);

-- Enable on table
ALTER TABLE [dbo].[t004forecast_latest]
ENABLE CHANGE_TRACKING;
```

### Tracker Implementation

**Base Class**: `SmartPulse.Application/Services/Database/CDC/BaseTracker.cs`

```csharp
public abstract class BaseTracker : TableChangeTrackerBase
{
    protected abstract string TrackerName { get; }
    protected virtual int IntervalMs { get; } = SystemVariables.CDCInterval;

    protected abstract Task OnChangeAction(List<ChangeItem> changes, Guid traceId);

    // Template method pattern
    protected sealed override Task OnChange(List<ChangeItem> changes)
    {
        var traceId = Guid.NewGuid();
        try
        {
            OnChangeAction(changes, traceId);
        }
        catch (Exception ex)
        {
            Logger.LogError(ex, "{TraceId} - {Tracker}: {Message}",
                traceId, TrackerName, ex.Message);
        }
        return Task.CompletedTask;
    }
}
```

### CDC Trackers

| Tracker | Table | Interval | Action |
|---------|-------|----------|--------|
| T004ForecastLatestTracker | t004forecast_latest | 100ms | Output cache eviction |
| T000EntityPermissionsTracker | t000entity_permission | 10s | Memory cache invalidation |
| T000EntityPropertyTracker | t000entity_property | 10s | Config cache invalidation |
| T000EntitySystemHierarchyTracker | t000entity_system_hierarchy | 10s | Hierarchy cache invalidation |
| SysUserRolesTracker | SysUserRole | 10s | User access cache invalidation |
| PowerPlantTracker | PowerPlant | 10s | Timezone cache invalidation |

### Cache Invalidation Flow

```
1. Database change (INSERT/UPDATE/DELETE)
2. SQL Server Change Tracking records change
3. CDC Tracker polls CHANGETABLE (100ms-10s interval)
4. Tracker identifies affected cache keys/tags
5. For memory cache: Cancel CancellationTokenSource
6. For output cache: EvictByTagAsync(tag)
7. Next request fetches fresh data
```

---

## Repository Pattern

### Base Repository

**Implementation**: Inherits from `SmartPulseBaseSqlRepository<T>`

```csharp
public class ForecastRepository : SmartPulseBaseSqlRepository<T004Forecast>
{
    public ForecastRepository(ForecastDbContext dbContext) : base(dbContext)
    {
    }

    public async Task<T004Forecast?> GetByIdAsync(Guid id)
    {
        return await DbContext.Set<T004Forecast>()
            .AsNoTracking()
            .FirstOrDefaultAsync(f => f.BatchId == id);
    }
}
```

### Repository Features

- Generic CRUD operations
- AsNoTracking for read-only queries
- Async methods throughout
- Expression-based filtering

---

## Background Service Patterns

### SystemVariableRefresher

**Purpose**: Refresh environment-based configuration without restart.

```csharp
public class SystemVariableRefresher : BackgroundService
{
    protected override async Task ExecuteAsync(CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            SystemVariables.Refresh();  // Reload from environment
            await Task.Delay(TimeSpan.FromSeconds(10), stoppingToken);
        }
    }
}
```

### CDC Tracker Registration

**Implementation**: `IServiceCollectionExtensions.cs`

```csharp
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

## Error Handling Patterns

### Exception Middleware

**Implementation**: `SmartPulse.Web.Services/Middlewares/ExceptionMiddleware.cs`

```csharp
public async Task InvokeAsync(HttpContext context)
{
    try
    {
        await _next(context);
    }
    catch (ApiException ex)
    {
        context.Response.StatusCode = ex.StatusCode;
        await context.Response.WriteAsJsonAsync(
            ApiResponse<object>.Error(ex.Message, traceId));
    }
    catch (Exception ex)
    {
        _logger.LogError(ex, "{TraceId}: {Message}", traceId, ex.Message);
        context.Response.StatusCode = 500;
        await context.Response.WriteAsJsonAsync(
            ApiResponse<object>.Error("Internal server error", traceId));
    }
}
```

### API Response Pattern

```csharp
public class ApiResponse<T>
{
    public int StatusCode { get; set; }
    public bool IsError { get; set; }
    public string? Message { get; set; }
    public T? Data { get; set; }
    public string TraceId { get; set; }
}
```

---

## Design Decisions & Trade-Offs

### Decision 1: Two-Level Caching (Not Four)

**Why two tiers instead of four (L1-L4):**
- This service doesn't require distributed caching
- Single-instance deployment scenario
- Simpler implementation and maintenance
- **Trade-off:** No cross-instance cache synchronization

### Decision 2: CDC-Based Invalidation

**Why SQL Server Change Tracking:**
- Built-in to SQL Server (no external dependencies)
- Row-level granularity
- Automatic cleanup
- **Trade-off:** Polling-based (100ms-10s latency)

### Decision 3: In-Memory Cache Only

**Why no Redis:**
- Single service instance is sufficient
- Reduces infrastructure complexity
- Lower operational cost
- **Trade-off:** Cache not shared between instances

### Decision 4: Output Cache with Tags

**Why tag-based eviction:**
- Efficient bulk invalidation
- CDC can invalidate related responses
- No manual cache key management
- **Trade-off:** Tag generation complexity

### Decision 5: SemaphoreSlim per Cache Key

**Why per-key locking:**
- Prevents cache stampede
- Fine-grained locking (no global lock)
- Thread-safe without blocking unrelated requests
- **Trade-off:** Memory overhead for semaphores

---

## Performance Characteristics

### Cache Hit Rates

| Cache Level | Expected Hit Rate | Latency |
|-------------|------------------|---------|
| Output Cache | 70-90% | <5ms |
| Memory Cache | 80-95% | <1ms |
| Database | 5-20% | 50-200ms |

### Throughput

| Operation | Throughput | Notes |
|-----------|-----------|-------|
| GET (cache hit) | 10K+ req/sec | Output cache |
| GET (cache miss) | 500+ req/sec | DB query |
| POST (save) | 100+ req/sec | Bulk insert |
| CDC poll | 10+ queries/sec | Per tracker |

---

## Related Documentation

- [System Overview](00_system_overview.md)
- [Data Flow & Communication](data_flow_communication.md)
- [Caching Patterns](../patterns/caching_patterns.md)
- [CDC Documentation](../data/cdc.md)

---

**Document Version**: 2.0
**Last Updated**: 2025-11-28
