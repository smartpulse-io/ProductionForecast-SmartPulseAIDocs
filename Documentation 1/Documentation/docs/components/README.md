# Components Documentation

**Last Updated**: 2025-11-28

---

## Overview

This directory contains documentation for the ProductionForecast service components.

## Available Components

### ProductionForecast Service

The main service documented in this repository.

| Document | Description |
|----------|-------------|
| [Overview](./production_forecast/README.md) | Service architecture, API overview |
| [Web API Layer](./production_forecast/web_api_layer.md) | Controllers, middleware |
| [Business Logic & Caching](./production_forecast/business_logic_caching.md) | Services, caching |
| [Data Layer & Entities](./production_forecast/data_layer_entities.md) | EF Core, entities |
| [HTTP Client & Models](./production_forecast/http_client_models.md) | DTOs, models |

## Technology Stack

| Technology | Version | Purpose |
|------------|---------|---------|
| .NET | 9.0 | Runtime |
| ASP.NET Core | 9.0 | Web framework |
| Entity Framework Core | 9.0.3 | ORM |
| SQL Server | - | Database |
| Electric.Core | 7.0.158 | CDC base classes (NuGet) |

**Note**: This service uses local caching only (Output Cache + IMemoryCache). No Redis or Pulsar.

---

## Quick Navigation

- **Architecture**: [System Overview](../architecture/00_system_overview.md)
- **Caching**: [Caching Patterns](../patterns/caching_patterns.md)
- **CDC**: [Change Data Capture](../data/cdc.md)

---

**Last Updated**: 2025-11-28
