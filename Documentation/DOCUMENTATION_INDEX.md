# SmartPulse Documentation Index

**Quick Navigation Guide** | Last Updated: 2026-01-09

---

## 📋 Table of Contents

### 🚀 START HERE
1. **[PROJECT_SUMMARY.md](./PROJECT_SUMMARY.md)** - Complete technical documentation with architecture, flows, and performance metrics
2. **[ACTUAL_DEPENDENCIES.md](./ACTUAL_DEPENDENCIES.md)** - Verified dependency analysis (what is ACTUALLY used)
3. **[docs/README.md](./docs/README.md)** - Documentation structure and quick start guide

### 📐 Architecture & Design
- **[System Overview](./docs/architecture/00_system_overview.md)** - Microservices topology, data flow, deployment model
- **[Architectural Patterns](./docs/architecture/architectural_patterns.md)** - Design principles, communication patterns, resilience strategies
- **[Data Flow & Communication](./docs/architecture/data_flow_communication.md)** - Detailed data flows, API contracts, cache invalidation

### 🔧 Components
- **[Electric.Core Framework](./docs/components/electric_core.md)** - Infrastructure library (CDC ONLY - no Pulsar/Redis in ProductionForecast)

#### Production Forecast Service
- **[Overview](./docs/components/production_forecast/README.md)** - Service architecture, domain models, API overview
- **[Web API Layer](./docs/components/production_forecast/web_api_layer.md)** - REST endpoints, middleware, policies
- **[Business Logic & Caching](./docs/components/production_forecast/business_logic_caching.md)** - IMemoryCache (local), OutputCache, CDC-based invalidation

#### Notification Service
- **[Overview](./docs/components/notification_service/README.md)** - Service architecture, notification channels
- **[Service Architecture](./docs/components/notification_service/service_architecture.md)** - Worker patterns, queue processing
- **[Data Models & Integration](./docs/components/notification_service/data_models_integration.md)** - Domain models, database schema
- **[API Endpoints](./docs/components/notification_service/api_endpoints.md)** - REST API, rate limiting

#### Infrastructure & Core
- **[Infrastructure Components](./docs/components/infrastructure/README.md)** - Shared utilities, logging, monitoring

### 🔌 Integration
- **[CDC (Change Data Capture)](./docs/integration/cdc.md)** - SQL Server Change Tracking implementation
- **[GraphQL Client](./docs/integration/graphql.md)** - Contract service GraphQL client

### 💾 Data Layer
- **[EF Core Strategy](./docs/data/ef_core.md)** - DbContext design, interceptors, L2 cache (EasyCaching)
- **[Change Data Capture](./docs/data/cdc.md)** - CDC implementation details
- **[Database Schema](./docs/data/schema.md)** - Entity relationships, stored procedures

### 🏗️ Patterns & Best Practices
- **[Design Patterns](./docs/patterns/design_patterns.md)** - Repository, Observer, Template Method, Decorator
- **[Caching Patterns](./docs/patterns/caching_patterns.md)** - 3-tier local cache (OutputCache + IMemoryCache + L2)
- **[CDC Patterns](./docs/patterns/cdc_patterns.md)** - Change tracking, cache invalidation strategies

### 📚 Developer Guides
- **[Setup Guide](./docs/guides/setup.md)** - Getting started, local environment, Docker setup
- **[Deployment Guide](./docs/guides/deployment.md)** - Kubernetes deployment, scaling, CI/CD
- **[Performance Guide](./docs/guides/performance.md)** - Optimization techniques, profiling, benchmarks
- **[Troubleshooting Guide](./docs/guides/troubleshooting.md)** - Common issues and solutions

---

## 🔍 Find Information By...

### By Technology
- **IMemoryCache** → ProductionForecast uses local in-memory cache (see [PROJECT_SUMMARY.md](./PROJECT_SUMMARY.md))
- **OutputCache** → ASP.NET Core 9.0 middleware (see [PROJECT_SUMMARY.md](./PROJECT_SUMMARY.md))
- **EFCore SecondLevelCache** → EasyCaching InMemory provider (see [docs/data/ef_core.md](./docs/data/ef_core.md))
- **CDC (Change Data Capture)** → SQL Server Change Tracking (see [docs/data/cdc.md](./docs/data/cdc.md))
- **Microsoft SQL Server** → [docs/data/ef_core.md](./docs/data/ef_core.md), [docs/data/schema.md](./docs/data/schema.md)
- **Entity Framework Core** → [docs/data/ef_core.md](./docs/data/ef_core.md)
- **GraphQL** → [docs/integration/graphql.md](./docs/integration/graphql.md) (Client only)
- **ASP.NET Core 9.0** → [docs/components/production_forecast.md](./docs/components/production_forecast.md)

### By Concern
- **Performance** → [PROJECT_SUMMARY.md](./PROJECT_SUMMARY.md) (metrics), [docs/guides/performance.md](./docs/guides/performance.md)
- **Scalability** → [PROJECT_SUMMARY.md](./PROJECT_SUMMARY.md) (deployment), [docs/guides/deployment.md](./docs/guides/deployment.md)
- **Caching (ProductionForecast)** → 3-tier local cache ([PROJECT_SUMMARY.md](./PROJECT_SUMMARY.md))
- **Real-time Updates** → CDC-based invalidation ([docs/data/cdc.md](./docs/data/cdc.md))
- **Security** → [PROJECT_SUMMARY.md](./PROJECT_SUMMARY.md) (authorization flow)

### By Service/Component
- **ProductionForecast Service**
  - Architecture: [PROJECT_SUMMARY.md](./PROJECT_SUMMARY.md) (complete documentation)
  - API: [PROJECT_SUMMARY.md](./PROJECT_SUMMARY.md) (12+ endpoints)
  - Caching: 3-tier local (OutputCache + IMemoryCache + L2)
  - CDC: [docs/data/cdc.md](./docs/data/cdc.md) (6 trackers)

- **NotificationService**
  - Architecture: [docs/components/notification_service.md](./docs/components/notification_service.md)
  - Integration: Separate microservice

- **Electric.Core Library**
  - Overview: [docs/components/electric_core.md](./docs/components/electric_core.md)
  - CDC (Change Data Capture): [docs/data/cdc.md](./docs/data/cdc.md) - **ProductionForecast uses ONLY this**
  - ❌ Pulsar: NOT used by ProductionForecast
  - ❌ Redis: NOT used by ProductionForecast

### By Audience
- **New Developer** → [PROJECT_SUMMARY.md](./PROJECT_SUMMARY.md) → [docs/guides/setup.md](./docs/guides/setup.md)
- **Architect** → [PROJECT_SUMMARY.md](./PROJECT_SUMMARY.md) → [docs/architecture/00_system_overview.md](./docs/architecture/00_system_overview.md)
- **DBA** → [docs/data/schema.md](./docs/data/schema.md), [docs/data/ef_core.md](./docs/data/ef_core.md)
- **DevOps/SRE** → [docs/guides/deployment.md](./docs/guides/deployment.md), [PROJECT_SUMMARY.md](./PROJECT_SUMMARY.md) (deployment section)
- **Performance Engineer** → [docs/guides/performance.md](./docs/guides/performance.md), [PROJECT_SUMMARY.md](./PROJECT_SUMMARY.md) (performance section)

---

## 🎯 Common Tasks

### I want to...

#### Add a new API endpoint
1. **Check existing**: [PROJECT_SUMMARY.md](./PROJECT_SUMMARY.md) (API section)
2. **Add controller**: [docs/components/production_forecast/web_api_layer.md](./docs/components/production_forecast/web_api_layer.md)
3. **Update cache tags**: [PROJECT_SUMMARY.md](./PROJECT_SUMMARY.md) (DataTag format)
4. **Write tests**: [docs/guides/setup.md](./docs/guides/setup.md)

#### Debug performance issues
1. **Check metrics**: [PROJECT_SUMMARY.md](./PROJECT_SUMMARY.md) (performance characteristics)
2. **Query optimization**: [docs/data/ef_core.md](./docs/data/ef_core.md)
3. **Cache diagnostics**: [PROJECT_SUMMARY.md](./PROJECT_SUMMARY.md) (cache architecture)
4. **CDC lag**: [docs/data/cdc.md](./docs/data/cdc.md)
5. **Tools**: [docs/guides/performance.md](./docs/guides/performance.md)

#### Add local cache entry (ProductionForecast)
1. **Update CacheManager**: Add new cache key using IMemoryCache
2. **Add CDC tracker**: [docs/data/cdc.md](./docs/data/cdc.md)
3. **Test invalidation**: Verify CDC triggers cache expiration
4. **Note**: ProductionForecast uses ONLY local cache (no Redis)

#### Scale the system
1. **Read**: [PROJECT_SUMMARY.md](./PROJECT_SUMMARY.md) (deployment section)
2. **Understand**: Each instance has own cache, CDC invalidates all
3. **Deploy**: [docs/guides/deployment.md](./docs/guides/deployment.md) (Kubernetes)

#### Monitor production
1. **Metrics**: [PROJECT_SUMMARY.md](./PROJECT_SUMMARY.md) (throughput, latency tables)
2. **Logs**: [docs/guides/troubleshooting.md](./docs/guides/troubleshooting.md)
3. **Health checks**: Add /health endpoint (see Known Limitations)

#### Write unit tests
1. **Guidelines**: [docs/guides/setup.md](./docs/guides/setup.md)
2. **Note**: Currently no tests (Priority 0 improvement)

#### Migrate database schema
1. **EF Core migrations**: [docs/data/ef_core.md](./docs/data/ef_core.md)
2. **Apply order**: [docs/guides/setup.md](./docs/guides/setup.md)

---

## 📊 Documentation Statistics

| Category | Files | Purpose |
|----------|-------|---------|
| **Main Documentation** | [PROJECT_SUMMARY.md](./PROJECT_SUMMARY.md) | Complete technical documentation (1240 lines, 23 diagrams) |
| **Dependency Verification** | [ACTUAL_DEPENDENCIES.md](./ACTUAL_DEPENDENCIES.md) | Verified actual dependencies vs. potential dependencies |
| **Architecture** | 3 files | System design, patterns, data flows |
| **Components** | 10+ files | Service-specific details |
| **Integration** | 2 files | CDC, GraphQL |
| **Data Layer** | 3 files | EF Core, CDC, schema |
| **Patterns** | 3 files | Design patterns, caching, CDC |
| **Guides** | 4 files | Setup, deployment, performance, troubleshooting |

---

## 🔗 Key Diagrams (in [PROJECT_SUMMARY.md](./PROJECT_SUMMARY.md))

1. **Complete Infrastructure View** - All microservices ecosystem (ProductionForecast + NotificationService + Electric.Core)
2. **ProductionForecast Service Architecture** - Actual implementation (local cache only)
3. **Layered Architecture** - Presentation → Application → Domain → Infrastructure → Data
4. **Request/Response Flows** - GET (cache hit), GET (cache miss), POST (save)
5. **Three-Tier Cache Architecture** - OutputCache + IMemoryCache + L2Cache (all local)
6. **CDC Architecture** - SQL Server Change Tracking integration
7. **T004ForecastLatestTracker Flow** - Output cache invalidation
8. **CacheManager Thread-Safety** - SemaphoreSlim pattern
9. **Authorization Flow** - Role-based + unit-level permissions
10. **Database Schema** - Entity relationships
11. **Deployment** - Single instance architecture

---

## ⚙️ Configuration & Setup

- **Local Development**: [docs/guides/setup.md](./docs/guides/setup.md)
- **Environment Variables**: [PROJECT_SUMMARY.md](./PROJECT_SUMMARY.md) (configuration section)
- **Database Migrations**: [docs/data/ef_core.md](./docs/data/ef_core.md)

---

## 🆘 Troubleshooting

Quick links:
- **Cache not invalidating**: [docs/data/cdc.md](./docs/data/cdc.md) (CDC-based invalidation)
- **Slow API response**: [docs/guides/performance.md](./docs/guides/performance.md) + [docs/data/ef_core.md](./docs/data/ef_core.md)
- **Database migration failed**: [docs/guides/troubleshooting.md](./docs/guides/troubleshooting.md)
- **Note**: ProductionForecast does NOT use Pulsar or Redis

---

## 📚 Learning Path

### For Beginners (New to project)
1. Read: [PROJECT_SUMMARY.md](./PROJECT_SUMMARY.md) - Executive Summary (5 min)
2. Read: [PROJECT_SUMMARY.md](./PROJECT_SUMMARY.md) - Architecture Overview (10 min)
3. Setup: [docs/guides/setup.md](./docs/guides/setup.md) (30 min)
4. Explore: [PROJECT_SUMMARY.md](./PROJECT_SUMMARY.md) - API Endpoints (10 min)

### For Architects (System design review)
1. Read: [PROJECT_SUMMARY.md](./PROJECT_SUMMARY.md) (20 min)
2. Review: [docs/architecture/00_system_overview.md](./docs/architecture/00_system_overview.md) (15 min)
3. Assess: [PROJECT_SUMMARY.md](./PROJECT_SUMMARY.md) - Known Limitations (5 min)

### For Performance Engineers
1. Read: [PROJECT_SUMMARY.md](./PROJECT_SUMMARY.md) - Performance Characteristics (10 min)
2. Read: [docs/guides/performance.md](./docs/guides/performance.md) (20 min)
3. Review: [docs/data/ef_core.md](./docs/data/ef_core.md) (15 min)

---

## 📞 Support

- **Documentation Issues**: Check [PROJECT_SUMMARY.md](./PROJECT_SUMMARY.md) first
- **Architecture Questions**: See [PROJECT_SUMMARY.md](./PROJECT_SUMMARY.md) - Architecture Overview
- **Component Specifics**: See [PROJECT_SUMMARY.md](./PROJECT_SUMMARY.md) or respective [docs/components/](./docs/components/) files

---

## 📄 Document Maintenance

- **Last Updated**: 2026-01-09
- **Version**: 2.0 (Corrected with actual dependencies)
- **Main Document**: [PROJECT_SUMMARY.md](./PROJECT_SUMMARY.md) (1240 lines, 23 Mermaid diagrams)
- **Review Schedule**: Quarterly
- **Status**: ✅ Verified against actual codebase

---

## ⚠️ IMPORTANT NOTES

### What ProductionForecast Actually Uses:
- ✅ **IMemoryCache** (local in-memory cache)
- ✅ **OutputCache** (ASP.NET Core 9.0 middleware)
- ✅ **EFCore SecondLevelCache** (EasyCaching InMemory)
- ✅ **Electric.Core CDC** (Change Data Capture ONLY)
- ✅ **Electric.Core Electricity helpers** (Forecast calculation)
- ✅ **GraphQL Client** (Contract service)

### What ProductionForecast Does NOT Use:
- ❌ **Redis** (not used in ProductionForecast)
- ❌ **Apache Pulsar** (not used in ProductionForecast)
- ❌ **MongoDB** (not used in ProductionForecast)
- ❌ **Electric.Core Pulsar features**
- ❌ **Electric.Core Redis features**

See **[ACTUAL_DEPENDENCIES.md](./ACTUAL_DEPENDENCIES.md)** for detailed verification.

---

**Navigation**: Start with [PROJECT_SUMMARY.md](./PROJECT_SUMMARY.md) for complete technical documentation.
