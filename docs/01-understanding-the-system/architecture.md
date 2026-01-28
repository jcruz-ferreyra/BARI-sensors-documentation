## Functional Architecture

The diagram below shows the complete data flow from sensors to public dashboard:

<p align="center">
  <img src="images/diagrams/02-functional-architecture.png" alt="Functional Architecture Diagram" width="800">
</p>

### Data Flow Overview

1. **Sensors (55 Particle.io devices)** collect temperature, humidity, and noise data every 15 minutes
2. **Particle Cloud** triggers webhooks when new data arrives
3. **Webhook Function** receives JSON payloads via HTTPS POST and writes to Blob Storage
4. **Blob Storage** archives raw sensor data and triggers the processor function
5. **Blob Processor Function** reads, deduplicates, and writes cleaned data to SQL Database
6. **SQL Database** stores validated sensor readings with composite indexes
7. **API Function** serves data requests from the dashboard via HTTPS GET
8. **Public Dashboard** (Static Web App) displays interactive visualizations for researchers and public

**Legend:**
- Solid arrows (→) = Push/Write operations
- Dashed arrows (⇢) = Pull/Trigger/Read operations
- Rectangles = Processing components
- Cylinders = Storage components