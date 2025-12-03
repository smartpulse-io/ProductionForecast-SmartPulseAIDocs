# SmartPulse.Services.ProductionForecast - Documentation Index

**Quick Navigation Guide** | Last Updated: 2025-11-28 | Version: 2.0

---

## Table of Contents

### START HERE
1. **[PROJECT_SUMMARY.md](./PROJECT_SUMMARY.md)** - Executive overview, architecture, tech stack
2. **[docs/README.md](./docs/README.md)** - Documentation structure and quick start

### Architecture & Design
- **[System Overview](./docs/architecture/00_system_overview.md)** - Service architecture, data flow, deployment model
- **[Architectural Patterns](./docs/architecture/architectural_patterns.md)** - Design principles, caching strategy, CDC patterns
- **[Data Flow & Communication](./docs/architecture/data_flow_communication.md)** - Request flows, API contracts, cache invalidation

### Components
- **[Production Forecast Service](./docs/components/production_forecast/README.md)** - Service architecture, domain models, API overview

### Data Layer
- **[Change Data Capture](./docs/data/cdc.md)** - SQL Server Change Tracking, CDC trackers, cache invalidation

### Patterns & Best Practices
- **[Caching Patterns](./docs/patterns/caching_patterns.md)** - Two-level cache, cache-aside, invalidation strategies

### Developer Guides
- **[Setup Guide](./docs/guides/setup.md)** - Getting started, local environment setup
- **[Troubleshooting Guide](./docs/guides/troubleshooting.md)** - Common issues and solutions

---

## Technology Stack

| Technology | Version | Purpose |
|------------|---------|---------|
| .NET | 9.0 | Runtime |
| ASP.NET Core | 9.0 | Web framework |
| Entity Framework Core | 9.0.3 | ORM |
| SQL Server | - | Database with Change Tracking |
| Electric.Core | 7.0.158 | CDC base classes (NuGet) |

**Note**: This service does NOT use Apache Pulsar or Redis. It uses local caching only.

---

## Find Information By...

### By Technology
- **SQL Server** → `docs/data/cdc.md`, `docs/architecture/00_system_overview.md`
- **Entity Framework Core** → `docs/architecture/architectural_patterns.md`
- **ASP.NET Core Output Cache** → `docs/patterns/caching_patterns.md`
- **IMemoryCache** → `docs/patterns/caching_patterns.md`

### By Concern
- **Performance** → `docs/architecture/00_system_overview.md`, `docs/patterns/caching_patterns.md`
- **Caching** → `docs/patterns/caching_patterns.md`, `docs/architecture/architectural_patterns.md`
- **Real-time Updates** → `docs/data/cdc.md` (cache invalidation via CDC)
- **API Design** → `docs/architecture/data_flow_communication.md`

### By Audience
- **New Developer** → START: `PROJECT_SUMMARY.md` → `docs/architecture/00_system_overview.md` → `docs/guides/setup.md`
- **Architect** → `docs/architecture/00_system_overview.md` → `docs/architecture/architectural_patterns.md`
- **DBA** → `docs/data/cdc.md`, `docs/architecture/data_flow_communication.md`
- **DevOps/SRE** → `docs/guides/setup.md`, `docs/guides/troubleshooting.md`

---

## Common Tasks

### I want to...

#### Understand the caching strategy
1. Read: `docs/architecture/architectural_patterns.md` (Caching Patterns section)
2. Read: `docs/patterns/caching_patterns.md` (detailed implementation)
3. Check: `docs/data/cdc.md` (cache invalidation via CDC)

#### Debug cache invalidation issues
1. Check CDC tracker logs
2. Read: `docs/data/cdc.md` (Troubleshooting section)
3. Verify: Change Tracking enabled on tables

#### Add a new API endpoint
1. Review: `docs/architecture/data_flow_communication.md` (API contracts)
2. Add: Output cache policy with appropriate tags
3. Update: CDC tracker if needed for cache invalidation

#### Debug performance issues
1. Check: Cache hit rates (`docs/patterns/caching_patterns.md`)
2. Review: CDC tracker intervals (`docs/data/cdc.md`)
3. Verify: Database queries are optimized

---

## Document Structure

```
Documentation/
├── PROJECT_SUMMARY.md              # Executive overview (start here)
├── DOCUMENTATION_INDEX.md          # This file
├── docs/
│   ├── README.md                   # Docs folder overview
│   ├── architecture/
│   │   ├── 00_system_overview.md   # System architecture
│   │   ├── architectural_patterns.md # Design patterns
│   │   └── data_flow_communication.md # Data flows & API
│   ├── components/
│   │   └── production_forecast/    # Service-specific docs
│   ├── data/
│   │   └── cdc.md                  # Change Data Capture
│   ├── patterns/
│   │   └── caching_patterns.md     # Caching implementation
│   └── guides/
│       ├── setup.md                # Setup instructions
│       └── troubleshooting.md      # Problem solving
└── notes/                          # Internal analysis notes
```

---

## Key Diagrams

All diagrams are in ASCII/text format within documentation files:

1. **System Architecture** → `docs/architecture/00_system_overview.md`
2. **Cache Flow** → `docs/patterns/caching_patterns.md`
3. **CDC Flow** → `docs/data/cdc.md`
4. **Request Flow** → `docs/architecture/data_flow_communication.md`
5. **Save Forecast Flow** → `docs/architecture/data_flow_communication.md`

---

## Quick Reference

### Cache Types

| Cache | Technology | TTL | Invalidation |
|-------|------------|-----|--------------|
| Output Cache (L1) | ASP.NET Core Output Cache | 60 min | Tag-based via CDC |
| Memory Cache (L2) | IMemoryCache | 1-1440 min | CancellationToken via CDC |

### CDC Trackers

| Tracker | Table | Interval | Action |
|---------|-------|----------|--------|
| T004ForecastLatestTracker | t004forecast_latest | 100ms | Output cache eviction |
| T000EntityPermissionsTracker | t000entity_permission | 10s | Memory cache |
| T000EntityPropertyTracker | t000entity_property | 10s | Memory cache |
| T000EntitySystemHierarchyTracker | t000entity_system_hierarchy | 10s | Memory cache |
| SysUserRolesTracker | SysUserRole | 10s | Memory cache |
| PowerPlantTracker | PowerPlant | 10s | Memory cache |

### API Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/{provider}/{type}/{unit}/forecasts/latest` | Get latest forecasts |
| GET | `/{provider}/{type}/{unit}/forecasts/latest-by-date` | Get by date |
| POST | `/{provider}/{type}/{unit}/forecasts` | Save forecasts |
| GET | `/system/cache-manager/cache-types` | List cache types |
| POST | `/system/cache-manager/{type}/expire` | Expire cache |

---

## Learning Path

### For Beginners (New to project)
1. Read: `PROJECT_SUMMARY.md` (10 min)
2. Read: `docs/architecture/00_system_overview.md` (15 min)
3. Setup: `docs/guides/setup.md` (30 min)
4. Understand caching: `docs/patterns/caching_patterns.md` (15 min)

### For Architects (System design review)
1. Read: `PROJECT_SUMMARY.md` (10 min)
2. Read: `docs/architecture/00_system_overview.md` (15 min)
3. Read: `docs/architecture/architectural_patterns.md` (20 min)
4. Review: `docs/data/cdc.md` (15 min)

---

## Troubleshooting Quick Links

- **Cache not invalidating** → `docs/data/cdc.md` (Troubleshooting section)
- **Slow API response** → `docs/patterns/caching_patterns.md` (Performance section)
- **CDC not detecting changes** → `docs/data/cdc.md` (Troubleshooting section)
- **High CPU from trackers** → `docs/data/cdc.md` (Best Practices section)

---

## Document Maintenance

- **Last Updated**: 2025-11-28
- **Version**: 2.0
- **Review Schedule**: Quarterly or after major changes

---

**Key Points:**
- This is a **single-service** documentation (ProductionForecast only)
- Service uses **two-level caching** (Output Cache + Memory Cache)
- **No distributed cache** (no Redis)
- **No message broker** (no Pulsar)
- Cache invalidation via **SQL Server Change Tracking + CDC trackers**
