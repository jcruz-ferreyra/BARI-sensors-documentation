# {{ project.name }} Documentation

## Welcome

This documentation describes the technical infrastructure supporting the [Common SENSES Project]({{ urls.project_website }}) sensor data pipeline. Common SENSES is a community-led environmental justice initiative serving neighborhoods along Blue Hill Avenue from Dudley Town Common to Franklin Park in Boston.

## About Common SENSES

Common SENSES is a collaboration between Dudley Street Neighborhood Initiative, Project R.I.G.H.T. Inc., Boston Area Research Initiative (BARI) at Northeastern University, and the City of Boston's Office of Emerging Technology.

**Project Values:**

- **No Surveillance** - Only environmental data, no information about people
- **Environmental Justice** - Supporting community action on environmental issues
- **Data Justice** - All data publicly accessible for community use

Learn more at [commonsensesproject.org]({{ urls.project_website }})

## Project Status

**Current State:** Fully operational production system  
**Deployment Date:** {{ dates.deployment_date }}  
**Last Updated:** {{ dates.last_updated }}  
**Active Sensors:** {{ sensors.active_count }} sensors collecting data every {{ sensors.collection_interval }}  
**Data Volume:** {{ data.readings_per_day }} readings/day, {{ data.readings_per_month }} readings/month  
**Monthly Cost:** <${{ costs.monthly_target }} (Azure infrastructure)

## What This System Does

The sensor data pipeline collects, processes, stores, and visualizes environmental data from Particle.io sensors deployed throughout the Common SENSES project area. The system measures temperature, humidity, and noise levels every {{ sensors.collection_interval }}, supporting both community access and research analysis.

## Quick Navigation

**New to the project?**  
Start with the [Project Overview](00-overview.md) for a comprehensive one-page summary.

**Need to access or analyze data?**  
Go to [Working with Data](02-working-with-data/quick-start.md) for database queries and data export methods.

**Maintaining the system?**  
See [Operating the System](03-operating-the-system/daily-checklist.md) for daily tasks and troubleshooting.

**Taking over development?**  
Read [Understanding the System](01-understanding-the-system/architecture.md), then explore [Reference](05-reference/azure-infrastructure/function-apps.md) for complete technical details.

**Need step-by-step instructions?**  
Check the [Appendices](06-appendices/step-by-step-guides/create-sql-database.md) for Azure configuration, database management, and deployment guides.

## Documentation Sections

### [Overview](00-overview.md)
One-page system summary with architecture, data flow, and key components.

### [Understanding the System](01-understanding-the-system/what-it-does.md)
- [What It Does](01-understanding-the-system/what-it-does.md) - High-level purpose
- [Architecture](01-understanding-the-system/architecture.md) - Component interactions
- [Data Flow](01-understanding-the-system/data-flow.md) - Sensor to dashboard journey
- [Design Decisions](01-understanding-the-system/design-decisions.md) - Why this architecture?

### [Working with Data](02-working-with-data/quick-start.md)
- [Quick Start](02-working-with-data/quick-start.md) - Get data immediately
- [Connecting to Database](02-working-with-data/connecting-to-database.md) - Connection strings and access
- [Common Queries](02-working-with-data/common-queries.md) - Copy-paste SQL examples
- [Blob Storage Access](02-working-with-data/blob-storage-access.md) - Raw JSON files
- [Understanding the Schema](02-working-with-data/understanding-the-schema.md) - Tables and relationships
- [Data Quality](02-working-with-data/data-quality.md) - Flags and validation

### [Operating the System](03-operating-the-system/daily-checklist.md)
- [Daily Checklist](03-operating-the-system/daily-checklist.md) - Routine maintenance tasks
- [Monitoring Health](03-operating-the-system/monitoring-health.md) - System health indicators
- [Cost Tracking](03-operating-the-system/cost-tracking.md) - Budget monitoring
- [Sensor Lifecycle](03-operating-the-system/sensor-lifecycle.md) - Deploy, move, deactivate
- **Troubleshooting:**
  - [Sensor Not Reporting](03-operating-the-system/troubleshooting/sensor-not-reporting.md)
  - [Function Errors](03-operating-the-system/troubleshooting/function-errors.md)
  - [Database Issues](03-operating-the-system/troubleshooting/database-issues.md)
  - [Dashboard Problems](03-operating-the-system/troubleshooting/dashboard-problems.md)

### [Making Changes](04-making-changes/deploying-code-updates.md)
- [Deploying Code Updates](04-making-changes/deploying-code-updates.md) - Function deployment
- [Updating Dashboard](04-making-changes/updating-dashboard.md) - Static Web App changes
- [Database Changes](04-making-changes/database-changes.md) - Schema migrations
- [Sensor Configuration](04-making-changes/sensor-configuration.md) - Particle.io webhooks
- [Infrastructure Changes](04-making-changes/infrastructure-changes.md) - Azure resource modifications

### [Reference](05-reference/azure-infrastructure/function-apps.md)
- **Azure Infrastructure:**
  - [Function Apps](05-reference/azure-infrastructure/function-apps.md)
  - [SQL Database](05-reference/azure-infrastructure/sql-database.md)
  - [Blob Storage](05-reference/azure-infrastructure/blob-storage.md)
  - [Networking](05-reference/azure-infrastructure/networking.md)
  - [Static Web App](05-reference/azure-infrastructure/static-web-app.md)
- [Particle Platform](05-reference/particle-platform.md) - Device specs and webhook config
- [Complete Schema](05-reference/complete-schema.md) - All tables, columns, indexes
- [Function Code Reference](05-reference/function-code-reference.md) - Code organization
- [API Endpoints](05-reference/api-endpoints.md) - Dashboard API documentation

### [Appendices](06-appendices/step-by-step-guides/create-sql-database.md)
- [Step-by-Step Guides](06-appendices/step-by-step-guides/create-sql-database.md) - Azure Portal tasks with screenshots
- [SQL Query Library](06-appendices/sql-query-library.md) - Extended query examples
- [Error Codes](06-appendices/error-codes.md) - Error reference
- [Contacts](06-appendices/contacts.md) - Who to contact for support

## External Resources

- [Common SENSES Project Website]({{ urls.project_website }})
- [Common SENSES Research Map]({{ urls.map_public }})
- [Heat Live Public Dashboard]({{ urls.heat_dashboard_public }})
- [Noise Live Public Dashboard]({{ urls.noise_dashboard_public }})

