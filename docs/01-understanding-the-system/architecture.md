# System Architecture

This page provides a high-level view of the BARI Environmental Sensor Network infrastructure. It shows the major components, how they connect, and how data flows from sensors in the field to the public dashboard. For detailed technical specifications of individual components, see the [Reference](../05-reference/azure-infrastructure/function-apps.md) section. For step-by-step data processing, see [Data Flow](data-flow.md).

## Architecture Overview

![Functional Architecture](../images/diagrams/functional-architecture.png)


## Core Components

![Particle Sensors](../images/diagrams/icon-sensor.png){ width="30" style="vertical-align: middle; opacity: 0.7" } **Particle Sensors**

{{ sensors.active_count }} environmental sensors deployed throughout the Common SENSES project area. Each sensor collects temperature, humidity, and noise data every {{ sensors.collection_interval }} and transmits readings to Particle Cloud via cellular connection.

For technical specifications see [Particle Platform Reference](../05-reference/particle-platform.md#sensors).


![Particle Cloud](../images/diagrams/icon-particle.png){ width="20" style="vertical-align: middle; opacity: 0.7" } **Particle Cloud**

Particle.io's cloud platform receives data from sensors and triggers webhooks to deliver readings to Azure infrastructure.

For webhook configuration details see [Particle Platform Reference](../05-reference/particle-platform.md#webhooks).


![Webhook Receiver](../images/diagrams/icon-func.png){ width="20" style="vertical-align: middle; opacity: 0.7" } **Webhook Receiver**

Azure Function that receives JSON payloads from Particle webhooks, validates and parses sensor data, then writes to Blob Storage as individual files. Each webhook delivery contains multiple sensor readings grouped together.

For function code and configuration see [Function Apps Reference](../05-reference/azure-infrastructure/function-apps.md#webhook-receiver).


![Blob Storage](../images/diagrams/icon-blob.png){ width="20" style="vertical-align: middle; opacity: 0.7" } **Blob Storage**

Azure Blob Storage container organized into folders (incoming/, archived/, failed-writing/) that stores raw sensor data as JSON files for downstream processing.

For storage organization and access see [Blob Storage Reference](../05-reference/azure-infrastructure/blob-storage.md).


![Database Writer](../images/diagrams/icon-func.png){ width="20" style="vertical-align: middle; opacity: 0.7" } **Database Writer**

Azure Function that runs every {{ processing.writer_schedule }} to process files in the incoming/ folder. Validates, deduplicates, and writes sensor readings to the SQL database, then moves processed files to archived/ or failed-{reason}/ folders.

For function code and configuration see [Function Apps Reference](../05-reference/azure-infrastructure/function-apps.md#database-writer).


![SQL Database](../images/diagrams/icon-sql.png){ width="20" style="vertical-align: middle; opacity: 0.7" } **SQL Database**

Azure SQL Database that stores validated sensor readings, error logs, startup events, and sensor deployment records. Primary data source for research analysis and dashboard queries.

For complete schema details see [Complete Schema Reference](../05-reference/complete-schema.md) and [SQL Database Reference](../05-reference/azure-infrastructure/sql-database.md).


![API](../images/diagrams/icon-func.png){ width="20" style="vertical-align: middle; opacity: 0.7" } **API**

Azure Function that provides HTTP endpoints for querying sensor data. Serves the public dashboard with filtered, aggregated data based on time ranges and sensor selections.

For endpoint documentation see [API Endpoints Reference](../05-reference/api-endpoints.md) and [Function Apps Reference](../05-reference/azure-infrastructure/function-apps.md#api).


![Public Facing Dashboard](../images/diagrams/icon-dash.png){ width="20" style="vertical-align: middle; opacity: 0.7" } **Public Facing Dashboard**

Azure Static Web App hosting the public dashboard at [link]({{ urls.heat_dashboard_public }}) for heat data and [link]({{ urls.noise_dashboard_public }}) for noise data. Built with vanilla JavaScript and Plotly.js for data visualization.

For dashboard code and deployment see [Static Web App Reference](../05-reference/azure-infrastructure/static-web-app.md).


![Daily Reporter](../images/diagrams/icon-func.png){ width="20" style="vertical-align: middle; opacity: 0.7" } **Daily Reporter**

Azure Function that runs on a daily schedule to send operational summaries via email. Reports include sensor health status, data collection metrics, and any errors from the previous 24 hours.

For function code and configuration see [Function Apps Reference](../05-reference/azure-infrastructure/function-apps.md#daily-reporter).


## Component Interactions

![Particle Sensors](../images/diagrams/icon-sensor.png){ width="30" style="vertical-align: middle; opacity: 0.7" } **Particle Sensors**
→
![Particle Cloud](../images/diagrams/icon-particle.png){ width="20" style="vertical-align: middle; opacity: 0.7" } **Particle Cloud**

Sensors transmit readings via cellular connection every {{ sensors.collection_interval }}.


![Particle Cloud](../images/diagrams/icon-particle.png){ width="20" style="vertical-align: middle; opacity: 0.7" } **Particle Cloud**
→
![Webhook Receiver](../images/diagrams/icon-func.png){ width="20" style="vertical-align: middle; opacity: 0.7" } **Webhook Receiver**

Particle Cloud pushes sensor data to Webhook Receiver via HTTP webhook.


![Webhook Receiver](../images/diagrams/icon-func.png){ width="20" style="vertical-align: middle; opacity: 0.7" } **Webhook Receiver**
→
![Blob Storage](../images/diagrams/icon-blob.png){ width="20" style="vertical-align: middle; opacity: 0.7" } **Blob Storage**

Webhook Receiver writes validated JSON files to incoming/ folder in Blob Storage.

![Blob Storage](../images/diagrams/icon-blob.png){ width="20" style="vertical-align: middle; opacity: 0.7" } **Blob Storage**
⇢
![Database Writer](../images/diagrams/icon-func.png){ width="20" style="vertical-align: middle; opacity: 0.7" } **Database Writer**

Database Writer polls incoming/ folder on schedule to retrieve unprocessed files.


![Database Writer](../images/diagrams/icon-func.png){ width="20" style="vertical-align: middle; opacity: 0.7" } **Database Writer**
→
![SQL Database](../images/diagrams/icon-sql.png){ width="20" style="vertical-align: middle; opacity: 0.7" } **SQL Database**

Database Writer validates, deduplicates, and writes sensor readings to database tables.


![Public Facing Dashboard](../images/diagrams/icon-dash.png){ width="20" style="vertical-align: middle; opacity: 0.7" } **Public Facing Dashboard**
→
![API](../images/diagrams/icon-func.png){ width="20" style="vertical-align: middle; opacity: 0.7" } **API**

Dashboard sends requests to API endpoints based on user-selected time ranges and sensors.


![SQL Database](../images/diagrams/icon-sql.png){ width="20" style="vertical-align: middle; opacity: 0.7" } **SQL Database**
⇢
![API](../images/diagrams/icon-func.png){ width="20" style="vertical-align: middle; opacity: 0.7" } **API**

API Function queries database and returns filtered, aggregated sensor readings.


![SQL Database](../images/diagrams/icon-sql.png){ width="20" style="vertical-align: middle; opacity: 0.7" } **SQL Database**
⇢
![Daily Reporter](../images/diagrams/icon-func.png){ width="20" style="vertical-align: middle; opacity: 0.7" } **Daily Reporter**

Daily Reporter queries operational metrics and sends summary email reports on schedule.