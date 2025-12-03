# Caching Patterns - SmartPulse.Services.ProductionForecast

**Version**: 2.0
**Last Updated**: 2025-11-28
**Purpose**: Two-level caching strategy for the ProductionForecast service.

---

## Table of Contents

1. [Overview](#overview)
2. [Two-Level Cache Architecture](#two-level-cache-architecture)
3. [Output Cache (Level 1)](#output-cache-level-1)
4. [Memory Cache (Level 2)](#memory-cache-level-2)
5. [Cache-Aside Pattern with Stampede Prevention](#cache-aside-pattern-with-stampede-prevention)
6. [Cache Invalidation Strategies](#cache-invalidation-strategies)
7. [Cache Key Conventions](#cache-key-conventions)
8. [Performance Characteristics](#performance-characteristics)
9. [Best Practices](#best-practices)

---

## Overview

SmartPulse.Services.ProductionForecast implements a **two-level caching strategy** optimized for single-instance deployment:

### Cache Hierarchy

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

### Why Two Levels (Not Three or Four)?

| Decision | Rationale |
|----------|-----------|
| No Redis | Single-instance deployment; no need for distributed cache |
| No EF Core L2 | Direct DB queries via stored procedures; EF tracking disabled |
| Simple architecture | Reduced operational complexity |
| Sufficient performance | Two levels provide >95% cache hit rate |

---

## Two-Level Cache Architecture

### Cache Flow Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                      API REQUEST                             │
│  GET /api/v2/production-forecast/{provider}/{type}/{unit}/  │
│                    forecasts/latest                          │
└─────────────────────────────┬───────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                   OUTPUT CACHE CHECK                         │
│              (ForecastPolicy.CacheRequestAsync)              │
│                                                              │
│  Key: Generated from route + query parameters                │
│  Tags: {unitType}.{unitNo}.{provider}.{period}.{date}       │
├─────────────────────────────────────────────────────────────┤
│  HIT: Return cached HTTP response immediately                │
│  MISS: Continue to controller →                              │
└─────────────────────────────┬───────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                   MEMORY CACHE CHECK                         │
│              (CacheManager.GetOrCreateAsync)                 │
│                                                              │
│  - SemaphoreSlim per key (stampede prevention)               │
│  - Double-checked locking pattern                            │
│  - CancellationToken for manual expiration                   │
├─────────────────────────────────────────────────────────────┤
│  HIT: Return cached data                                     │
│  MISS: Execute factory → Query database →                    │
└─────────────────────────────┬───────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                   DATABASE QUERY                             │
│              (Repository + Stored Procedures)                │
│                                                              │
│  - AsNoTracking for read-only queries                        │
│  - Stored procedures for complex queries                     │
│  - Result cached in both levels                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Output Cache (Level 1)

### Configuration

**File**: `SmartPulse.Web.Services/Program.cs`

```csharp
builder.Services.AddOutputCache(options =>
{
    options.AddPolicy("ForecastPolicy", builder =>
    {
        builder.AddPolicy<ForecastPolicy>();
        builder.Expire(TimeSpan.FromMinutes(60));
    });
});
```

### ForecastPolicy Implementation

**File**: `SmartPulse.Web.Services/Policies/ForecastPolicy.cs`

```csharp
public class ForecastPolicy : IOutputCachePolicy
{
    public ValueTask CacheRequestAsync(
        OutputCacheContext context,
        CancellationToken cancellationToken)
    {
        // Extract route parameters
        var unitType = context.HttpContext.Request.RouteValues["unitType"]?.ToString();
        var unitNo = context.HttpContext.Request.RouteValues["unitNo"]?.ToString();
        var providerKey = context.HttpContext.Request.RouteValues["providerKey"]?.ToString();
        var period = context.HttpContext.Request.Query["period"].ToString();
        var from = context.HttpContext.Request.Query["from"].ToString();
        var to = context.HttpContext.Request.Query["to"].ToString();

        // Generate tags for invalidation
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

    public ValueTask ServeFromCacheAsync(
        OutputCacheContext context,
        CancellationToken cancellationToken)
    {
        return ValueTask.CompletedTask;
    }

    public ValueTask ServeResponseAsync(
        OutputCacheContext context,
        CancellationToken cancellationToken)
    {
        return ValueTask.CompletedTask;
    }
}
```

### Tag Format

Tags enable efficient bulk invalidation:

```
Format: {unitType}.{unitNo}.{providerKey}.{period}.{date}

Examples:
- UEVM.001.PROVIDER1.15.2025-01-01
- UEVS.002.PROVIDER2.60.2025-01-15
```

### Controller Usage

```csharp
[HttpGet("{providerKey}/{unitType}/{unitNo}/forecasts/latest")]
[OutputCache(PolicyName = "ForecastPolicy")]
public async Task<IActionResult> GetLatestForecasts(
    string providerKey,
    string unitType,
    string unitNo,
    [FromQuery] int period,
    [FromQuery] DateTime from,
    [FromQuery] DateTime to)
{
    // Response automatically cached with generated tags
    var result = await _forecastService.GetLatestAsync(...);
    return Ok(result);
}
```

---

## Memory Cache (Level 2)

### CacheManager Implementation

**File**: `SmartPulse.Application/CacheManager.cs`

```csharp
public class CacheManager
{
    private readonly IMemoryCache _memoryCache;
    private readonly ConcurrentDictionary<string, SemaphoreSlim> _semaphores = new();
    private readonly ConcurrentDictionary<string, CancellationTokenSource> _expirationTokens = new();

    public async Task<T?> GetOrCreateAsync<T>(
        string key,
        Func<Task<T>> factory,
        TimeSpan ttl)
    {
        // Fast path: Check cache first
        if (_memoryCache.TryGetValue(key, out T cached))
            return cached;

        // Get or create semaphore for this key (stampede prevention)
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

    public void ExpireCacheByKey(string key)
    {
        if (_expirationTokens.TryRemove(key, out var cts))
        {
            cts.Cancel();
            cts.Dispose();
        }
    }

    private CancellationTokenSource GetNewOrExistingExpirationTokenSource(string key)
    {
        return _expirationTokens.GetOrAdd(key, _ => new CancellationTokenSource());
    }
}
```

### Cache Key Types and TTLs

| Cache Type | Key Pattern | TTL | Purpose |
|------------|-------------|-----|---------|
| Hierarchy | `powerplant_all_hierarchies` | 60 min | Unit hierarchy data |
| Timezone | `all_powerplant_timezones` | 24 hours | Timezone mappings |
| User Access | `user_accessible_units_{userId}_{unitType}` | 24 hours | User permissions |
| Company Limits | `company_limitsettings_{companyId}` | 1 min | Forecast limits |
| GIP Config | `gip_config_{key}` | 60 min | System configuration |
| Region | `region_{regionId}` | 60 min | Region data |

---

## Cache-Aside Pattern with Stampede Prevention

### The Problem: Cache Stampede

When a popular cache entry expires, multiple concurrent requests may all miss the cache simultaneously, causing a "stampede" of identical database queries.

```
Without Protection:
Request 1 → Cache miss → DB query →
Request 2 → Cache miss → DB query →
Request 3 → Cache miss → DB query →
Request 4 → Cache miss → DB query →
(100 concurrent requests = 100 DB queries)
```

### The Solution: SemaphoreSlim Per Key

```csharp
// Per-key semaphore ensures only one request queries the database
var semaphore = _semaphores.GetOrAdd(key, _ => new SemaphoreSlim(1, 1));
await semaphore.WaitAsync();

try
{
    // Double-check pattern
    if (_memoryCache.TryGetValue(key, out cached))
        return cached;  // Another request already populated cache

    // Only one request reaches here
    var value = await factory();
    _memoryCache.Set(key, value, options);
    return value;
}
finally
{
    semaphore.Release();
}
```

```
With Protection:
Request 1 → Cache miss → Acquire lock → DB query → Cache set → Release lock
Request 2 → Cache miss → Wait for lock → Acquire lock → Cache hit! →
Request 3 → Cache miss → Wait for lock → Acquire lock → Cache hit! →
Request 4 → Cache miss → Wait for lock → Acquire lock → Cache hit! →
(100 concurrent requests = 1 DB query)
```

### Benefits

- **Per-key granularity**: Different keys can be loaded concurrently
- **No global lock**: Requests for different data don't block each other
- **Double-checked locking**: Prevents redundant queries after lock acquisition
- **Memory efficient**: Semaphores created only when needed

---

## Cache Invalidation Strategies

### Strategy 1: TTL-Based Expiration

Automatic expiration after configured duration:

```csharp
_memoryCache.Set(key, value, new MemoryCacheEntryOptions
{
    AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(60)
});
```

**Use cases:**
- Reference data that changes infrequently
- Data where slight staleness is acceptable

### Strategy 2: CDC-Triggered Invalidation

Real-time invalidation via SQL Server Change Tracking:

```csharp
// In CDC Tracker (e.g., T000EntitySystemHierarchyTracker)
protected override Task OnChangeAction(List<ChangeItem> changes, Guid traceId)
{
    // Invalidate memory cache
    _cacheManager.ExpireCacheByKey("powerplant_all_hierarchies");

    _logger.LogInformation("{TraceId} - Hierarchy cache invalidated", traceId);
    return Task.CompletedTask;
}
```

### Strategy 3: Tag-Based Output Cache Eviction

For forecast data, CDC triggers tag-based eviction:

```csharp
// In T004ForecastLatestTracker
protected override async Task OnChangeAction(List<ChangeItem> changes, Guid traceId)
{
    foreach (var change in changes)
    {
        var tag = DataTagHelper.GenerateTag(
            change.UnitType,
            change.UnitNo,
            change.ProviderKey,
            change.Period,
            change.Date);

        await _outputCacheStore.EvictByTagAsync(tag, CancellationToken.None);
    }
}
```

### Strategy 4: Manual API Invalidation

Admin endpoint for manual cache clearing:

```csharp
[HttpPost("{cacheType}/expire")]
public IActionResult ExpireCache(string cacheType)
{
    _cacheManager.ExpireCacheByType(cacheType);
    return Ok(new { message = "Cache expired successfully" });
}
```

---

## Cache Key Conventions

### Naming Convention

```
Format: {domain}_{entity}_{identifier}

Examples:
- powerplant_all_hierarchies
- all_powerplant_timezones
- user_accessible_units_{userId}_{unitType}
- company_limitsettings_{companyId}
- gip_config_{configKey}
```

### Output Cache Tag Convention

```
Format: {unitType}.{unitNo}.{providerKey}.{period}.{date}

Examples:
- UEVM.001.PROV1.15.2025-01-01
- UEVS.*.PROV2.60.*  (wildcards for bulk invalidation)
```

---

## Performance Characteristics

### Cache Hit Rates

| Cache Level | Expected Hit Rate | Latency |
|-------------|-------------------|---------|
| Output Cache (L1) | 70-90% | <5ms |
| Memory Cache (L2) | 80-95% | <1ms |
| Database | 5-20% | 50-200ms |

**Combined Effective Hit Rate**: ~95-99%

### Latency Breakdown

```
Request with Output Cache hit:  <5ms    (70-90% of requests)
Request with Memory Cache hit:  <10ms   (5-20% of requests)
Request with DB query:          50-200ms (1-5% of requests)

Average latency: ~5-10ms (95th percentile)
```

### Memory Usage

| Cache Type | Typical Size | Max Entries |
|------------|--------------|-------------|
| Output Cache | 100MB-500MB | 10K responses |
| Memory Cache | 50MB-200MB | 1K entries |

### Throughput Impact

| Scenario | Throughput |
|----------|-----------|
| All cache hits | 10K+ req/sec |
| 50% cache hits | 2K-5K req/sec |
| All cache misses | 500 req/sec |

---

## Best Practices

### 1. Cache Sizing

```csharp
// Configure memory cache limits
services.AddMemoryCache(options =>
{
    options.SizeLimit = 1024;  // Track entry sizes
    options.CompactionPercentage = 0.25;  // Compact when 25% over limit
});

// Set size on entries
_memoryCache.Set(key, value, new MemoryCacheEntryOptions
{
    Size = 1,  // Relative size unit
    AbsoluteExpirationRelativeToNow = ttl
});
```

### 2. TTL Selection Guidelines

| Data Type | Suggested TTL | Rationale |
|-----------|---------------|-----------|
| Static reference data | 24 hours | Rarely changes |
| User permissions | 24 hours | Changes infrequently |
| Configuration | 1 hour | Balance freshness vs. load |
| Forecast limits | 1 minute | May change during operations |
| Hierarchies | 1 hour | Admin-managed data |

### 3. Error Handling

```csharp
public async Task<T?> GetOrCreateAsync<T>(string key, Func<Task<T>> factory, TimeSpan ttl)
{
    try
    {
        if (_memoryCache.TryGetValue(key, out T cached))
            return cached;

        var value = await factory();

        if (value != null)
        {
            _memoryCache.Set(key, value, ttl);
        }

        return value;
    }
    catch (Exception ex)
    {
        _logger.LogError(ex, "Cache operation failed for key: {Key}", key);
        // Fallback: Return from factory without caching
        return await factory();
    }
}
```

### 4. Logging Cache Operations

```csharp
// Log cache misses for monitoring
if (!_memoryCache.TryGetValue(key, out T cached))
{
    _logger.LogDebug("Cache miss for key: {Key}", key);
}

// Log invalidations
public void ExpireCacheByKey(string key)
{
    _logger.LogInformation("Cache invalidated: {Key}", key);
    // ... invalidation logic
}
```

### 5. Avoid Over-Caching

**Don't cache:**
- User-specific transient data
- Data that changes every request
- Large datasets that consume too much memory
- Security-sensitive data that must always be fresh

**Do cache:**
- Reference data (hierarchies, timezones, config)
- Computed/aggregated data
- Data shared across users
- Expensive database queries

---

## Related Documentation

- [Architectural Patterns](../architecture/architectural_patterns.md) - Design decisions
- [Data Flow & Communication](../architecture/data_flow_communication.md) - Request flows
- [CDC Documentation](../data/cdc.md) - Change tracking setup

---

**Document Version**: 2.0
**Last Updated**: 2025-11-28
