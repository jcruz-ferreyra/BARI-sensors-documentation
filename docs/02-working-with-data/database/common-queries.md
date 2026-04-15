# Common Queries

This page provides copy-paste SQL queries for common data retrieval tasks. All examples use the nu_readings and nu_sensors tables.

For database connection instructions, see [Connecting to Database](connecting.md). For understanding the complete schema, see [Understanding the Schema](understanding-schema.md).

## Understanding Data Organization

Before querying sensor data, it's important to understand how sensors are identified in the database.

**location_id** - Represents a physical location where sensors have been deployed over time. When a sensor at a location is replaced or moved, the location_id stays the same but a new deployment_id is created. Use location_id for longitudinal analysis that combines data from multiple sensors deployed successively at the same spot.

**deployment_id** - Represents a specific sensor installed at a specific location during a specific time period. Each time a sensor is moved, it gets a new deployment_id. Use deployment_id when you need to distinguish between different sensors at the same location or when analyzing a specific installation period.

**box_id** - The physical sensor identifier (1-55) printed on the hardware. A single box_id can have multiple deployment_id values if the sensor has been moved to different locations. Use box_id to track a specific physical sensor across all its deployments, though this is less common for analysis.

**Which to use?**

- **Most common:** Query by location_id to get all data from a location over time
- **Specific period:** Query by deployment_id to get data from one sensor installation
- **Track hardware:** Query by box_id to follow a specific physical sensor (rare)

The examples below primarily use location_id and deployment_id as these are the most common use cases.

## Basic Data Retrieval

**Get all readings for a specific location:**

```sql
SELECT r.timestamp, r.temperature, r.humidity, r.heat_index, r.noise
FROM nu_readings r
JOIN nu_sensors s ON r.deployment_id = s.deployment_id
WHERE s.location_id = 1
ORDER BY r.timestamp DESC;
```

**Get readings for a specific date range:**

```sql
SELECT r.timestamp, r.temperature, r.humidity, r.heat_index, r.noise, s.location_address
FROM nu_readings r
JOIN nu_sensors s ON r.deployment_id = s.deployment_id
WHERE s.location_id = 1
    AND r.timestamp >= '2025-06-01 00:00:00'
    AND r.timestamp <= '2025-06-30 23:59:59'
ORDER BY r.timestamp;
```

Note: Timestamps are stored in UTC. Adjust dates accordingly or see [Data Export](#data-export) for timezone conversion.

**Get latest 1000 readings from a sensor:**

```sql
SELECT TOP 1000 r.timestamp, r.temperature, r.humidity, r.heat_index, r.noise
FROM nu_readings r
WHERE r.deployment_id = 19
ORDER BY r.timestamp DESC;
```

**Get readings from multiple locations:**

```sql
SELECT r.timestamp, s.location_id, s.location_address, r.temperature, r.humidity
FROM nu_readings r
JOIN nu_sensors s ON r.deployment_id = s.deployment_id
WHERE s.location_id IN (1, 5, 10, 15)
    AND r.timestamp >= '2025-06-01 00:00:00'
ORDER BY s.location_id, r.timestamp;
```

**List all available sensor locations:**

```sql
SELECT location_id, MIN(location_address) AS location_address
FROM nu_sensors
WHERE location_id IS NOT NULL
GROUP BY location_id
ORDER BY MIN(location_address);
```

## Filtering and Quality

**Exclude readings with quality issues:**

```sql
SELECT r.timestamp, r.temperature, r.humidity, r.heat_index, r.noise
FROM nu_readings r
JOIN nu_sensors s ON r.deployment_id = s.deployment_id
WHERE s.location_id = 1
    AND r.quality_ok = 1  -- Only valid data
    AND r.timestamp >= '2025-06-01 00:00:00'
ORDER BY r.timestamp;
```

**Find readings within specific ranges:**

```sql
SELECT r.timestamp, r.temperature, r.humidity, r.noise
FROM nu_readings r
WHERE r.deployment_id = 19
    AND r.temperature BETWEEN 70 AND 90  -- Fahrenheit
    AND r.humidity BETWEEN 40 AND 60     -- Percentage
    AND r.noise < 70                     -- Decibels
    AND r.quality_ok = 1
ORDER BY r.timestamp;
```

**Get only readings with non-null values:**

```sql
SELECT r.timestamp, r.temperature, r.humidity, r.heat_index, r.noise
FROM nu_readings r
WHERE r.deployment_id = 19
    AND r.temperature IS NOT NULL
    AND r.humidity IS NOT NULL
    AND r.noise IS NOT NULL
    AND r.quality_ok = 1
ORDER BY r.timestamp;
```

**Find all readings flagged with quality issues:**

```sql
SELECT r.timestamp, r.temperature, r.humidity, r.noise, r.source_blob
FROM nu_readings r
JOIN nu_sensors s ON r.deployment_id = s.deployment_id
WHERE s.location_id = 1
    AND r.quality_ok = 0  -- Flagged readings only
ORDER BY r.timestamp;
```

**Check which sensors have active quality issues:**

```sql
SELECT qi.box_id, s.location_address, qi.issue_type, qi.issue_description, qi.start_datetime
FROM nu_quality_issues qi
JOIN nu_sensors s ON qi.box_id = s.box_id AND s.is_active = 1
WHERE qi.is_resolved = 0
ORDER BY qi.start_datetime DESC;
```

For more information on interpreting quality flags, see [Data Quality](data-quality.md).

## Aggregations

**Calculate hourly averages for a location:**

```sql
SELECT
    DATEADD(HOUR, DATEDIFF(HOUR, 0, r.timestamp), 0) AS timestamp,
    AVG(r.temperature) AS avg_temperature,
    AVG(r.humidity) AS avg_humidity,
    AVG(r.heat_index) AS avg_heat_index,
    AVG(r.noise) AS avg_noise
FROM nu_readings r
JOIN nu_sensors s ON r.deployment_id = s.deployment_id
WHERE s.location_id = 1
    AND r.timestamp >= '2025-06-01 00:00:00'
    AND r.timestamp <= '2025-06-30 23:59:59'
    AND r.quality_ok = 1
GROUP BY DATEADD(HOUR, DATEDIFF(HOUR, 0, r.timestamp), 0)
ORDER BY timestamp;
```

**Calculate daily averages for a location:**

```sql
SELECT
    DATEADD(DAY, DATEDIFF(DAY, 0, r.timestamp), 0) AS timestamp,
    AVG(r.temperature) AS avg_temperature,
    AVG(r.humidity) AS avg_humidity,
    AVG(r.heat_index) AS avg_heat_index,
    AVG(r.noise) AS avg_noise
FROM nu_readings r
JOIN nu_sensors s ON r.deployment_id = s.deployment_id
WHERE s.location_id = 1
    AND r.timestamp >= '2025-06-01 00:00:00'
    AND r.timestamp <= '2025-06-30 23:59:59'
    AND r.quality_ok = 1
GROUP BY DATEADD(DAY, DATEDIFF(DAY, 0, r.timestamp), 0)
ORDER BY timestamp;
```

**Find minimum and maximum values over a time period:**

```sql
SELECT
    MIN(r.temperature) AS min_temp,
    MAX(r.temperature) AS max_temp,
    MIN(r.humidity) AS min_humidity,
    MAX(r.humidity) AS max_humidity,
    MIN(r.noise) AS min_noise,
    MAX(r.noise) AS max_noise
FROM nu_readings r
JOIN nu_sensors s ON r.deployment_id = s.deployment_id
WHERE s.location_id = 1
    AND r.timestamp >= '2025-06-01 00:00:00'
    AND r.timestamp <= '2025-06-30 23:59:59'
    AND r.quality_ok = 1;
```

**Count readings per sensor per day:**

```sql
SELECT
    DATEADD(DAY, DATEDIFF(DAY, 0, r.timestamp), 0) AS date,
    s.box_id,
    s.location_address,
    COUNT(*) AS total_readings,
    SUM(CASE WHEN r.quality_ok = 1 THEN 1 ELSE 0 END) AS valid_readings,
    SUM(CASE WHEN r.quality_ok = 0 THEN 1 ELSE 0 END) AS flagged_readings
FROM nu_readings r
JOIN nu_sensors s ON r.deployment_id = s.deployment_id
WHERE r.timestamp >= '2025-06-01 00:00:00'
    AND r.timestamp <= '2025-06-30 23:59:59'
GROUP BY DATEADD(DAY, DATEDIFF(DAY, 0, r.timestamp), 0), s.box_id, s.location_address
ORDER BY date, s.box_id;
```

This shows total readings alongside valid vs. flagged counts for quality assessment.

**Compare multiple locations side-by-side:**

```sql
SELECT
    DATEADD(DAY, DATEDIFF(DAY, 0, r.timestamp), 0) AS date,
    s.location_id,
    s.location_address,
    AVG(r.temperature) AS avg_temperature,
    AVG(r.humidity) AS avg_humidity
FROM nu_readings r
JOIN nu_sensors s ON r.deployment_id = s.deployment_id
WHERE s.location_id IN (1, 5, 10)
    AND r.timestamp >= '2025-06-01 00:00:00'
    AND r.timestamp <= '2025-06-30 23:59:59'
    AND r.quality_ok = 1
GROUP BY DATEADD(DAY, DATEDIFF(DAY, 0, r.timestamp), 0), s.location_id, s.location_address
ORDER BY date, s.location_id;
```

## Sensor Metadata

**Get all active sensor deployments:**

```sql
SELECT deployment_id, box_id, location_id, location_address, latitude, longitude, install_datetime
FROM nu_sensors
WHERE is_active = 1
ORDER BY location_address;
```

**Find sensor location by box_id:**

```sql
SELECT box_id, location_address, latitude, longitude, install_datetime, is_active
FROM nu_sensors
WHERE box_id = 19
ORDER BY install_datetime DESC;
```

**Find current deployment for a specific box_id:**

```sql
SELECT deployment_id, location_address, latitude, longitude, install_datetime
FROM nu_sensors
WHERE box_id = 19
    AND is_active = 1;
```

**Get deployment history for a location:**

```sql
SELECT deployment_id, box_id, install_datetime, uninstall_datetime, is_active
FROM nu_sensors
WHERE location_id = 1
ORDER BY install_datetime;
```

**Find location details by deployment_id:**

```sql
SELECT deployment_id, box_id, location_address, latitude, longitude, install_datetime, uninstall_datetime
FROM nu_sensors
WHERE deployment_id = 19;
```

**Count total active sensors:**

```sql
SELECT COUNT(*) AS active_sensor_count
FROM nu_sensors
WHERE is_active = 1;
```

**Find sensors deployed within a date range:**

```sql
SELECT deployment_id, box_id, location_address, install_datetime
FROM nu_sensors
WHERE install_datetime >= '2025-06-01 00:00:00'
    AND install_datetime <= '2025-06-30 23:59:59'
ORDER BY install_datetime;
```

## Advanced: Querying by Pipeline Arrival Time

Sometimes you need to find data based on when it arrived in the system, not when it was collected or written to the database. This is useful for:

- Investigating pipeline delays
- Finding data that arrived during a specific incident
- Debugging timestamp issues

The `source_blob` field contains the arrival timestamp in its filename format: `{box_id}_{YYYYMMDDTHH}`

**Get readings that arrived in the pipeline during a specific date range:**

```sql
SELECT *
FROM nu_readings
WHERE box_id = 23  -- adding the box_id improves query performance (index)
  AND source_blob >= '23_20251107T05'
  AND source_blob < '23_20251109T05';
```

This retrieves all readings from box 23 that arrived between November 7, 2025 05:00 and November 9, 2025 05:00 UTC.

**Find unique blob files from a time range:**

```sql
SELECT DISTINCT source_blob
FROM nu_readings
WHERE box_id = 23  -- adding the box_id improves query performance (index)
  AND source_blob >= '23_20251107T05'
  AND source_blob < '23_20251109T05';
```

**Format breakdown:**

- `23` - box_id
- `20251107` - YYYYMMDD (November 7, 2025)
- `T` - separator
- `05` - Hour in UTC (05:00)

Note: This queries by arrival time at the Webhook Receiver, not sensor collection time (timestamp) or database write time (created_at).

## Data Export

**Export readings with timezone conversion (UTC to Eastern Time):**

```sql
SELECT 
    r.deployment_id,
    s.box_id,
    s.location_address,
    r.timestamp AT TIME ZONE 'UTC' AT TIME ZONE 'Eastern Standard Time' AS timestamp_eastern,
    r.temperature,
    r.humidity,
    r.heat_index,
    r.noise,
    r.quality_ok
FROM nu_readings r
JOIN nu_sensors s ON r.deployment_id = s.deployment_id
WHERE s.location_id = 1
    AND r.timestamp >= '2025-06-01 00:00:00'
    AND r.timestamp <= '2025-06-30 23:59:59'
ORDER BY r.timestamp;
```

Note: `AT TIME ZONE 'Eastern Standard Time'` automatically handles daylight saving time transitions, converting to Eastern Daylight Time (EDT) when applicable.

**Export all data for a location (full dataset):**

```sql
SELECT 
    r.timestamp,
    s.box_id,
    s.location_address,
    r.temperature,
    r.humidity,
    r.heat_index,
    r.noise,
    r.quality_ok,
    r.source_blob
FROM nu_readings r
JOIN nu_sensors s ON r.deployment_id = s.deployment_id
WHERE s.location_id = 1
ORDER BY r.timestamp;
```

**Export daily aggregates ready for Excel:**

```sql
SELECT 
    CONVERT(DATE, r.timestamp) AS date,
    s.location_address,
    AVG(r.temperature) AS avg_temperature,
    MIN(r.temperature) AS min_temperature,
    MAX(r.temperature) AS max_temperature,
    AVG(r.humidity) AS avg_humidity,
    AVG(r.heat_index) AS avg_heat_index,
    AVG(r.noise) AS avg_noise,
    COUNT(*) AS reading_count
FROM nu_readings r
JOIN nu_sensors s ON r.deployment_id = s.deployment_id
WHERE s.location_id = 1
    AND r.timestamp >= '2025-06-01 00:00:00'
    AND r.quality_ok = 1
GROUP BY CONVERT(DATE, r.timestamp), s.location_address
ORDER BY date;
```

**Export with all metadata for complete traceability:**

```sql
SELECT 
    r.timestamp,
    s.deployment_id,
    s.box_id,
    s.location_id,
    s.location_address,
    s.latitude,
    s.longitude,
    r.temperature,
    r.humidity,
    r.heat_index,
    r.noise,
    r.quality_ok,
    r.source_blob,
    r.created_at
FROM nu_readings r
JOIN nu_sensors s ON r.deployment_id = s.deployment_id
WHERE r.timestamp >= '2025-06-01 00:00:00'
    AND r.timestamp <= '2025-06-30 23:59:59'
ORDER BY s.location_id, r.timestamp;
```

**Exporting results:**

After running any query in SSMS or Azure Data Studio:

1. Right-click on the results grid
2. Select "Save Results As..." or "Save as CSV"
3. Choose your file location and format

For VS Code with SQL extension:

1. Right-click on results table
2. Select "Save as CSV" or "Save as JSON"
