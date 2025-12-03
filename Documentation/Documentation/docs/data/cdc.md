# Change Data Capture (CDC) - SmartPulse.Services.ProductionForecast

**Version**: 2.0
**Last Updated**: 2025-11-28
**Technology**: SQL Server Change Tracking
**Purpose**: Real-time cache invalidation via database change detection.

---

## Table of Contents

1. [Overview](#overview)
2. [Architecture](#architecture)
3. [SQL Server Change Tracking Setup](#sql-server-change-tracking-setup)
4. [CDC Tracker Implementation](#cdc-tracker-implementation)
5. [Trackers in This Service](#trackers-in-this-service)
6. [Cache Invalidation Actions](#cache-invalidation-actions)
7. [Performance Characteristics](#performance-characteristics)
8. [Best Practices](#best-practices)
9. [Troubleshooting](#troubleshooting)

---

## Overview

SmartPulse.Services.ProductionForecast uses **SQL Server Change Tracking** to detect database changes and trigger **local cache invalidation**. This is a simpler approach than full CDC with message brokers - changes are detected via polling and caches are invalidated within the same service instance.

### CDC Flow

```
┌─────────────────────────────────────────────────────────────┐
│ 1. DATABASE CHANGE                                          │
│    Application INSERT/UPDATE/DELETE on tracked table        │
└─────────────────────────────┬───────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│ 2. SQL SERVER CHANGE TRACKING                               │
│    Records change in CHANGETABLE with version ID            │
└─────────────────────────────┬───────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│ 3. CDC TRACKER POLLS                                        │
│    Background service queries CHANGETABLE periodically      │
│    Interval: 100ms (forecast) or 10s (config)               │
└─────────────────────────────┬───────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│ 4. CACHE INVALIDATION                                       │
│    - Output Cache: EvictByTagAsync()                        │
│    - Memory Cache: Cancel CancellationTokenSource           │
└─────────────────────────────────────────────────────────────┘
```

### Key Characteristics

| Aspect | Implementation |
|--------|---------------|
| **Change Detection** | SQL Server Change Tracking (built-in) |
| **Polling** | Background services per table |
| **Invalidation Scope** | Local instance only |
| **Message Broker** | None (not needed for single instance) |
| **Latency** | 100ms - 10s depending on tracker |

---

## Architecture

### Component Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                   SQL SERVER DATABASE                        │
│  ┌─────────────────┐  ┌─────────────────┐                   │
│  │ t004forecast_   │  │ t000entity_     │                   │
│  │ latest          │  │ permission      │                   │
│  │ [Change Track]  │  │ [Change Track]  │                   │
│  └─────────────────┘  └─────────────────┘                   │
└─────────────────────────────┬───────────────────────────────┘
                              │ CHANGETABLE queries
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                   CDC TRACKERS                               │
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
│                   CACHE INVALIDATION                         │
│  ┌──────────────────┐     ┌──────────────────┐              │
│  │ Output Cache     │     │ Memory Cache     │              │
│  │ (IOutputCache    │     │ (CacheManager)   │              │
│  │  Store)          │     │                  │              │
│  │ EvictByTagAsync()│     │ ExpireCacheByKey │              │
│  └──────────────────┘     └──────────────────┘              │
└─────────────────────────────────────────────────────────────┘
```

### Base Classes (from Electric.Core NuGet)

The CDC infrastructure uses base classes from the `Electric.Core` NuGet package:

| Class | Purpose |
|-------|---------|
| `TableChangeTrackerBase` | Base class for table-specific trackers |
| `ChangeItem` | Represents a single change (version, operation, PK) |
| `ChangeTracker` | Core polling engine for CHANGETABLE |

---

## SQL Server Change Tracking Setup

### Enable on Database

```sql
-- Enable change tracking on the database
ALTER DATABASE [ForecastDb]
SET CHANGE_TRACKING = ON
(CHANGE_RETENTION = 2 DAYS, AUTO_CLEANUP = ON);
```

### Enable on Tables

```sql
-- Forecast data table
ALTER TABLE [dbo].[t004forecast_latest]
ENABLE CHANGE_TRACKING;

-- Permission table
ALTER TABLE [dbo].[t000entity_permission]
ENABLE CHANGE_TRACKING;

-- Property table
ALTER TABLE [dbo].[t000entity_property]
ENABLE CHANGE_TRACKING;

-- Hierarchy table
ALTER TABLE [dbo].[t000entity_system_hierarchy]
ENABLE CHANGE_TRACKING;

-- User roles table
ALTER TABLE [dbo].[SysUserRole]
ENABLE CHANGE_TRACKING;

-- Power plant table
ALTER TABLE [dbo].[PowerPlant]
ENABLE CHANGE_TRACKING;
```

### Verify Change Tracking Status

```sql
-- Check database-level change tracking
SELECT name, is_cdc_enabled, is_change_tracking_enabled
FROM sys.databases
WHERE name = 'ForecastDb';

-- Check table-level change tracking
SELECT t.name AS TableName, ct.is_track_columns_updated_on
FROM sys.change_tracking_tables ct
JOIN sys.tables t ON ct.object_id = t.object_id;
```

### Query Changes (CHANGETABLE)

```sql
-- Get changes since version @last_version
SELECT CT.SYS_CHANGE_VERSION,
       CT.SYS_CHANGE_OPERATION,
       CT.UnitType,
       CT.UnitNo,
       CT.ProviderKey
FROM CHANGETABLE(CHANGES t004forecast_latest, @last_version) AS CT;
```

---

## CDC Tracker Implementation

### Base Tracker Class

**File**: `SmartPulse.Application/Services/Database/CDC/BaseTracker.cs`

```csharp
public abstract class BaseTracker : TableChangeTrackerBase
{
    protected readonly ILogger Logger;
    protected readonly IServiceProvider ServiceProvider;

    protected abstract string TrackerName { get; }
    protected virtual int IntervalMs { get; } = SystemVariables.CDCInterval; // 1000ms default

    // Override in derived classes for custom intervals
    protected virtual int LongIntervalMs { get; } = SystemVariables.CDCLongInterval; // 10000ms

    protected abstract Task OnChangeAction(List<ChangeItem> changes, Guid traceId);

    // Template method - called by base class when changes detected
    protected sealed override Task OnChange(List<ChangeItem> changes)
    {
        var traceId = Guid.NewGuid();
        try
        {
            return OnChangeAction(changes, traceId);
        }
        catch (Exception ex)
        {
            Logger.LogError(ex, "{TraceId} - {Tracker}: Error processing changes - {Message}",
                traceId, TrackerName, ex.Message);
            return Task.CompletedTask;
        }
    }
}
```

### Tracker Registration

**File**: `SmartPulse.Web.Services/Extensions/IServiceCollectionExtensions.cs`

```csharp
public static IServiceCollection AddCDCTrackers(
    this IServiceCollection services,
    AppSettings appSettings)
{
    // Always register these trackers
    services.AddSmartpulseTableChangeTracker<T000EntityPermissionsTracker>();
    services.AddSmartpulseTableChangeTracker<T000EntityPropertyTracker>();
    services.AddSmartpulseTableChangeTracker<T000EntitySystemHierarchyTracker>();
    services.AddSmartpulseTableChangeTracker<SysUserRolesTracker>();
    services.AddSmartpulseTableChangeTracker<PowerPlantTracker>();

    // Conditionally register forecast tracker based on config
    if (appSettings.CacheSettings.OutputCache.UseCacheInvalidationChangeTracker)
    {
        services.AddSmartpulseTableChangeTracker<T004ForecastLatestTracker>();
    }

    return services;
}
```

---

## Trackers in This Service

### T004ForecastLatestTracker

**Purpose**: Invalidate output cache when forecast data changes.

```csharp
public class T004ForecastLatestTracker : BaseTracker
{
    private readonly IOutputCacheStore _outputCacheStore;

    protected override string TrackerName => "T004ForecastLatestTracker";
    protected override int IntervalMs => 100; // Fast polling for forecast data

    public override string TableName => "t004forecast_latest";
    public override string SelectColumns => "CT.UnitType, CT.UnitNo, CT.ProviderKey, CT.Period, CT.DeliveryDate";

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

            Logger.LogDebug("{TraceId} - Evicted output cache tag: {Tag}", traceId, tag);
        }
    }
}
```

### T000EntitySystemHierarchyTracker

**Purpose**: Invalidate hierarchy cache when unit structure changes.

```csharp
public class T000EntitySystemHierarchyTracker : BaseTracker
{
    private readonly CacheManager _cacheManager;

    protected override string TrackerName => "T000EntitySystemHierarchyTracker";
    protected override int IntervalMs => 10000; // 10s polling

    public override string TableName => "t000entity_system_hierarchy";

    protected override Task OnChangeAction(List<ChangeItem> changes, Guid traceId)
    {
        // Invalidate all hierarchy-related caches
        _cacheManager.ExpireCacheByKey("powerplant_all_hierarchies");

        Logger.LogInformation("{TraceId} - Hierarchy cache invalidated ({Count} changes)",
            traceId, changes.Count);

        return Task.CompletedTask;
    }
}
```

### T000EntityPermissionsTracker

**Purpose**: Invalidate user permission caches when access rules change.

```csharp
public class T000EntityPermissionsTracker : BaseTracker
{
    protected override string TrackerName => "T000EntityPermissionsTracker";
    protected override int IntervalMs => 10000;

    public override string TableName => "t000entity_permission";

    protected override Task OnChangeAction(List<ChangeItem> changes, Guid traceId)
    {
        // Invalidate all user access caches
        _cacheManager.ExpireAllCachesByPrefix("user_accessible_units_");

        Logger.LogInformation("{TraceId} - User permissions cache invalidated", traceId);
        return Task.CompletedTask;
    }
}
```

### Tracker Summary Table

| Tracker | Table | Interval | Cache Action |
|---------|-------|----------|--------------|
| T004ForecastLatestTracker | t004forecast_latest | 100ms | Output cache tag eviction |
| T000EntityPermissionsTracker | t000entity_permission | 10s | User access memory cache |
| T000EntityPropertyTracker | t000entity_property | 10s | Config memory cache |
| T000EntitySystemHierarchyTracker | t000entity_system_hierarchy | 10s | Hierarchy memory cache |
| SysUserRolesTracker | SysUserRole | 10s | User access memory cache |
| PowerPlantTracker | PowerPlant | 10s | Timezone memory cache |

---

## Cache Invalidation Actions

### Output Cache Invalidation (Tag-Based)

For forecast data, changes trigger tag-based eviction:

```csharp
// Tag format: {unitType}.{unitNo}.{providerKey}.{period}.{date}
var tag = "UEVM.001.PROVIDER1.15.2025-01-01";

// Evict all cached responses with this tag
await _outputCacheStore.EvictByTagAsync(tag, CancellationToken.None);
```

### Memory Cache Invalidation (CancellationToken)

For reference data, changes cancel the cache entry's token:

```csharp
// In CacheManager
public void ExpireCacheByKey(string key)
{
    if (_expirationTokens.TryRemove(key, out var cts))
    {
        cts.Cancel();  // Triggers cache entry removal
        cts.Dispose();
    }
}

// In tracker
_cacheManager.ExpireCacheByKey("powerplant_all_hierarchies");
```

### Invalidation Flow Diagram

```
┌─────────────────────────────────────────────────────────────┐
│ Forecast Data Change                                        │
│ (t004forecast_latest)                                       │
└─────────────────────────────┬───────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│ T004ForecastLatestTracker                                   │
│ - Extract: unitType, unitNo, provider, period, date         │
│ - Generate tag                                              │
└─────────────────────────────┬───────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│ Output Cache                                                │
│ EvictByTagAsync(tag)                                        │
│ → All matching cached HTTP responses removed                │
└─────────────────────────────────────────────────────────────┘


┌─────────────────────────────────────────────────────────────┐
│ Reference Data Change                                       │
│ (t000entity_system_hierarchy)                               │
└─────────────────────────────┬───────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│ T000EntitySystemHierarchyTracker                            │
│ - Identify affected cache keys                              │
└─────────────────────────────┬───────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│ Memory Cache (CacheManager)                                 │
│ ExpireCacheByKey("powerplant_all_hierarchies")              │
│ → CancellationToken cancelled → Entry removed               │
└─────────────────────────────────────────────────────────────┘
```

---

## Performance Characteristics

### Polling Overhead

| Tracker Type | Interval | Queries/Minute | CPU Impact |
|--------------|----------|----------------|------------|
| Forecast (100ms) | Fast | 600 | ~2-5% |
| Config (10s) | Slow | 6 | <1% |

### Latency

| Stage | Latency |
|-------|---------|
| Database change | <10ms |
| Change tracking record | <1ms |
| Poll detection | 100ms - 10s (depends on interval) |
| Cache eviction | <5ms |
| **Total (fast tracker)** | **~110-150ms** |
| **Total (slow tracker)** | **~10-15s** |

### Resource Usage

| Resource | Usage |
|----------|-------|
| DB connections | 1 per tracker (pooled) |
| Memory | ~10KB per tracker |
| Network | ~1KB per poll query |

---

## Best Practices

### 1. Choose Appropriate Poll Intervals

```csharp
// Fast polling for user-facing data
protected override int IntervalMs => 100;  // Forecast data

// Slow polling for admin-managed data
protected override int IntervalMs => 10000;  // Config, hierarchies
```

### 2. Use Specific Select Columns

```csharp
// Only select columns needed for cache key generation
public override string SelectColumns =>
    "CT.UnitType, CT.UnitNo, CT.ProviderKey, CT.Period";

// Avoid SELECT * - reduces data transfer
```

### 3. Handle Tracker Errors Gracefully

```csharp
protected override Task OnChangeAction(List<ChangeItem> changes, Guid traceId)
{
    try
    {
        // Process changes...
    }
    catch (Exception ex)
    {
        // Log but don't throw - let tracker continue
        Logger.LogError(ex, "{TraceId} - Error processing changes", traceId);
    }

    return Task.CompletedTask;
}
```

### 4. Log Invalidation Events

```csharp
Logger.LogInformation(
    "{TraceId} - Cache invalidated: {CacheType}, Changes: {Count}",
    traceId, "Hierarchy", changes.Count);
```

### 5. Configure Change Retention Appropriately

```sql
-- 2 days is usually sufficient
ALTER DATABASE [ForecastDb]
SET CHANGE_TRACKING (CHANGE_RETENTION = 2 DAYS);
```

---

## Troubleshooting

### CDC Not Detecting Changes

**Symptoms**: Changes made but cache not invalidating.

**Checklist**:

1. Verify change tracking enabled on database:
```sql
SELECT is_change_tracking_enabled
FROM sys.databases
WHERE name = 'ForecastDb';
```

2. Verify change tracking enabled on table:
```sql
SELECT t.name
FROM sys.change_tracking_tables ct
JOIN sys.tables t ON ct.object_id = t.object_id;
```

3. Check tracker is registered:
```csharp
// In Program.cs or startup
services.AddSmartpulseTableChangeTracker<T004ForecastLatestTracker>();
```

4. Verify tracker is running (check logs):
```
[INF] T004ForecastLatestTracker started polling
```

### High CPU from Trackers

**Symptoms**: Excessive CPU usage from CDC polling.

**Solutions**:

1. Increase poll interval:
```csharp
protected override int IntervalMs => 1000;  // Increase from 100ms
```

2. Reduce number of tracked tables.

3. Add filters to reduce change volume:
```csharp
public override string ExtraSqlFilter =>
    "CT.SYS_CHANGE_OPERATION <> 'D'";  // Ignore deletes
```

### Version ID Issues

**Symptoms**: Duplicate or missed changes.

**Solutions**:

1. The base class handles version tracking automatically.

2. If needed, reset by restarting the service (starts from current version).

3. Check for version cleanup:
```sql
-- View minimum valid version
SELECT CHANGE_TRACKING_MIN_VALID_VERSION(OBJECT_ID('t004forecast_latest'));
```

### Cache Not Being Invalidated

**Symptoms**: CDC detects changes but cache still stale.

**Checklist**:

1. Verify correct cache key/tag generation.

2. Check CacheManager is injected correctly.

3. Verify output cache store is configured:
```csharp
builder.Services.AddOutputCache();
```

4. Check logs for eviction messages:
```
[DBG] Evicted output cache tag: UEVM.001.PROV1.15.2025-01-01
```

---

## Related Documentation

- [Architectural Patterns](../architecture/architectural_patterns.md) - Overall design
- [Caching Patterns](../patterns/caching_patterns.md) - Cache implementation
- [Data Flow & Communication](../architecture/data_flow_communication.md) - Request flows

---

**Document Version**: 2.0
**Last Updated**: 2025-11-28
