# SmartPulse.Services.ProductionForecast - Documentation

**Version**: 2.0
**Last Updated**: 2025-11-28

---

## Project Overview

SmartPulse.Services.ProductionForecast is a **.NET 9.0 REST API** for managing electricity production forecasts. The service provides CRUD operations for forecasts with two-level caching, real-time cache invalidation via SQL Server Change Tracking (CDC), and user-based authorization.

**Technology Stack**:
- **.NET 9.0 / ASP.NET Core 9.0** - Web framework
- **Entity Framework Core 9.0.3** - ORM
- **SQL Server** - Database with Change Tracking
- **IMemoryCache** - In-memory application cache
- **ASP.NET Core Output Cache** - HTTP response caching
- **Electric.Core 7.0.158** - CDC base classes (NuGet package)

**Note**: This service does NOT use Apache Pulsar or Redis. It uses local caching only.

---

## Quick Start

### New to This Service?

**1. Start Here** (15-20 minutes):
- Read [PROJECT_SUMMARY.md](../PROJECT_SUMMARY.md) for executive overview
- Read [System Overview](./architecture/00_system_overview.md) for architecture
- Review [Setup Guide](./guides/setup.md) if you need to run locally

**2. Essential Concepts** (30 minutes):
- [Architectural Patterns](./architecture/architectural_patterns.md) - Design patterns and decisions
- [Data Flow & Communication](./architecture/data_flow_communication.md) - API contracts and flows
- [Caching Patterns](./patterns/caching_patterns.md) - Two-level cache implementation

**3. Deep Dive** (Pick your area):
- **Caching**: [Caching Patterns](./patterns/caching_patterns.md)
- **CDC**: [Change Data Capture](./data/cdc.md)
- **Troubleshooting**: [Troubleshooting Guide](./guides/troubleshooting.md)

---

## Documentation Structure

```
docs/
├── architecture/           # System architecture and design
│   ├── 00_system_overview.md
│   ├── architectural_patterns.md
│   └── data_flow_communication.md
├── components/
│   └── production_forecast/  # Service-specific details
├── data/
│   └── cdc.md               # Change Data Capture
├── patterns/
│   └── caching_patterns.md   # Two-level caching
└── guides/
    ├── setup.md              # Setup instructions
    └── troubleshooting.md    # Problem solving
```

---

## Architecture Documentation

| Document | Description | Read Time |
|----------|-------------|-----------|
| [System Overview](./architecture/00_system_overview.md) | Service architecture, deployment model, components | 15-20 min |
| [Architectural Patterns](./architecture/architectural_patterns.md) | Design principles, caching, CDC patterns | 20-25 min |
| [Data Flow & Communication](./architecture/data_flow_communication.md) | API contracts, request flows, cache invalidation | 20-25 min |

**Key Topics**:
- Two-level caching strategy (Output Cache + Memory Cache)
- CDC-based cache invalidation via SQL Server Change Tracking
- SemaphoreSlim per cache key for stampede prevention
- Tag-based output cache eviction

---

## Data Layer

| Document | Description | Read Time |
|----------|-------------|-----------|
| [Change Data Capture (CDC)](./data/cdc.md) | SQL Server Change Tracking, CDC trackers, invalidation | 15-20 min |

**Key Patterns**:
- SQL Server Change Tracking (built-in, no external dependencies)
- 6 CDC trackers for different data types
- Polling-based change detection (100ms - 10s intervals)
- Local cache invalidation (no distributed broadcast)

---

## Patterns & Best Practices

| Document | Description | Read Time |
|----------|-------------|-----------|
| [Caching Patterns](./patterns/caching_patterns.md) | Two-level cache, stampede prevention, invalidation | 20-25 min |

**Topics Covered**:
- Output Cache with tag-based eviction
- Memory Cache with CancellationToken expiration
- Double-checked locking with SemaphoreSlim
- Cache key conventions

---

## Quick Navigation

### By Role

**Backend Engineer**:
1. [System Overview](./architecture/00_system_overview.md) → [Caching Patterns](./patterns/caching_patterns.md)
2. [CDC](./data/cdc.md) → [Data Flow](./architecture/data_flow_communication.md)

**DevOps/SRE**:
1. [Setup Guide](./guides/setup.md) → [Troubleshooting](./guides/troubleshooting.md)
2. [CDC](./data/cdc.md) (monitoring section)

**Architect**:
1. [System Overview](./architecture/00_system_overview.md) → [Architectural Patterns](./architecture/architectural_patterns.md)
2. [Data Flow & Communication](./architecture/data_flow_communication.md)

### By Concern

**Caching**:
- [Caching Patterns](./patterns/caching_patterns.md)
- [Architectural Patterns](./architecture/architectural_patterns.md) (Caching section)

**Performance**:
- [Caching Patterns](./patterns/caching_patterns.md) (Performance section)
- [CDC](./data/cdc.md) (Performance section)

**Cache Invalidation**:
- [CDC](./data/cdc.md)
- [Data Flow](./architecture/data_flow_communication.md) (Cache Invalidation section)

---

## Key Concepts

### Two-Level Caching

```
L1: Output Cache (ASP.NET Core)
    └─ HTTP response caching
    └─ 60 min TTL, tag-based invalidation

L2: Memory Cache (IMemoryCache)
    └─ Application data caching
    └─ Configurable TTLs (1-1440 min)
    └─ CancellationToken-based invalidation
```

### CDC-Based Cache Invalidation

```
Database Change → SQL Change Tracking → CDC Tracker Polls
    → Detect Changes → Invalidate Cache (Local)
```

- Forecast data: 100ms polling interval
- Config data: 10s polling interval
- No distributed broadcast (single instance deployment)

---

## Performance Characteristics

| Operation | Throughput | Latency |
|-----------|-----------|---------|
| GET (output cache hit) | 10K+ req/sec | <5ms |
| GET (memory cache hit) | 10K+ req/sec | <1ms |
| GET (database query) | 500+ req/sec | 50-200ms |
| POST (save forecast) | 100+ req/sec | 50-500ms |
| Cache invalidation | - | ~100-150ms |

---

## Troubleshooting Quick Links

- **Cache not invalidating**: [CDC Troubleshooting](./data/cdc.md#troubleshooting)
- **Slow API response**: [Caching Patterns - Performance](./patterns/caching_patterns.md#performance-characteristics)
- **CDC not detecting changes**: [CDC Troubleshooting](./data/cdc.md#troubleshooting)

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| **2.0** | 2025-11-28 | Documentation corrected: Removed incorrect Pulsar/Redis references |
| **1.0** | 2025-11-13 | Initial documentation (contained inaccuracies) |

---

**Last Updated**: 2025-11-28
**Documentation Version**: 2.0

**Key Corrections in v2.0:**
- Removed all Apache Pulsar references (not used)
- Removed all Redis references (not used)
- Changed from 4-tier cache to 2-tier cache (accurate)
- Simplified architecture documentation to match actual implementation
