# Data Quality

## Overview

Not all sensor data is equally reliable. The database includes quality tracking to help you make informed decisions about which data to include in your analysis. The `quality_ok` flag in the nu_readings table marks whether individual readings are considered valid or potentially problematic.

Data gets flagged for quality issues when sensors malfunction, experience connectivity problems, or report values that fail validation checks. Understanding these quality markers helps ensure your analysis uses trustworthy data.

## Quality Flag (quality_ok)

Every reading in the nu_readings table includes a `quality_ok` flag:

**quality_ok = 1 (Valid Data)**

The reading passed all validation checks and comes from a sensor with no known issues at that time. This is the default state for most data.

**quality_ok = 0 (Flagged Data)**

The reading has been flagged as potentially unreliable. Reasons include:

- Sensor had an active quality issue during this time period
- Values failed validation (out of expected range, impossible combinations)
- Sensor experienced known hardware or connectivity problems
- Manual flagging after discovering data problems

**How Flags Are Set:**

Automatically: The Database Writer checks the nu_quality_issues table when processing new data. If a sensor has an unresolved quality issue, all new readings get quality_ok = 0.

Retroactively: When quality issues are discovered after data is already written, existing readings can be manually flagged. See [Database Changes - Flagging Data During Quality Issue Periods](../../04-making-changes/database-changes.md#flagging-data-during-quality-issue-periods) for procedures.

## Quality Issues Table

The nu_quality_issues table tracks known problems with specific sensors during specific time periods.

**What it tracks:**

- Which sensor (box_id)
- When the issue started (start_datetime)
- When it was resolved (end_datetime) - NULL if ongoing
- Type of issue (issue_type)
- Detailed description (issue_description)

**Common issue types:**

**clock_drift** - Sensor's internal clock is off, causing incorrect timestamps. Readings may still be valid but are difficult to align with other data sources.

**sensor_malfunction** - Hardware reporting implausible values (stuck readings, extreme outliers, erratic patterns).

**connectivity** - Intermittent communication causing data gaps or delayed transmission.

**deployment_error** - Incorrect metadata (wrong location recorded, misconfigured sensor).

**Check active issues for a sensor:**

```sql
SELECT issue_type, issue_description, start_datetime, end_datetime
FROM nu_quality_issues
WHERE box_id = 19
    AND is_resolved = 0
ORDER BY start_datetime DESC;
```

## Working with Quality Flags

### Including vs. Excluding Flagged Data

The decision to include or exclude flagged data depends on your analysis goals and the specific quality issues involved.

**When to exclude flagged data (quality_ok = 0):**

- Statistical analysis requiring high confidence (averages, trends, correlations)
- Comparing sensor performance where data quality matters
- Publishing results that require defensible data provenance
- Time-sensitive analysis where timestamp accuracy is critical

**When to include flagged data (quality_ok = 0):**

- Investigating what went wrong with a sensor
- Assessing the extent of data quality problems
- Completeness matters more than precision (e.g., checking if sensor was operational)
- Issue type doesn't affect your specific analysis (e.g., clock_drift doesn't matter if you're only looking at value distributions, not time series)

**Example: Exclude flagged data (most common)**

```sql
SELECT r.timestamp, r.temperature, r.humidity, r.noise
FROM nu_readings r
JOIN nu_sensors s ON r.deployment_id = s.deployment_id
WHERE s.location_id = 1
    AND r.timestamp >= '2025-06-01 00:00:00'
    AND r.quality_ok = 1  -- Only valid data
ORDER BY r.timestamp;
```

**Example: Include all data with quality indicator**

```sql
SELECT 
    r.timestamp, 
    r.temperature, 
    r.humidity, 
    r.noise,
    r.quality_ok,
    CASE 
        WHEN r.quality_ok = 1 THEN 'Valid'
        WHEN r.quality_ok = 0 THEN 'Flagged'
    END AS quality_status
FROM nu_readings r
JOIN nu_sensors s ON r.deployment_id = s.deployment_id
WHERE s.location_id = 1
    AND r.timestamp >= '2025-06-01 00:00:00'
ORDER BY r.timestamp;
```

**Example: Compare valid vs. flagged data counts**

```sql
SELECT 
    s.box_id,
    s.location_address,
    COUNT(*) AS total_readings,
    SUM(CASE WHEN r.quality_ok = 1 THEN 1 ELSE 0 END) AS valid_readings,
    SUM(CASE WHEN r.quality_ok = 0 THEN 1 ELSE 0 END) AS flagged_readings,
    CAST(SUM(CASE WHEN r.quality_ok = 1 THEN 1 ELSE 0 END) AS FLOAT) / COUNT(*) * 100 AS valid_percentage
FROM nu_readings r
JOIN nu_sensors s ON r.deployment_id = s.deployment_id
WHERE r.timestamp >= '2025-06-01 00:00:00'
GROUP BY s.box_id, s.location_address
ORDER BY valid_percentage;
```

### Understanding Issue Context

When you encounter flagged data, understanding why it was flagged helps you decide how to handle it.

**Find why specific readings were flagged:**

```sql
SELECT 
    r.timestamp,
    r.temperature,
    r.humidity,
    qi.issue_type,
    qi.issue_description,
    qi.start_datetime,
    qi.end_datetime
FROM nu_readings r
JOIN nu_sensors s ON r.deployment_id = s.deployment_id
JOIN nu_quality_issues qi ON s.box_id = qi.box_id
    AND r.timestamp >= qi.start_datetime
    AND (qi.end_datetime IS NULL OR r.timestamp <= qi.end_datetime)
WHERE r.deployment_id = 19
    AND r.quality_ok = 0
ORDER BY r.timestamp;
```

**Interpreting issue_type:**

**clock_drift** - Timestamps are unreliable, but temperature/humidity/noise values may still be accurate. Consider excluding for time-series analysis but potentially useful for value distribution studies.

**sensor_malfunction** - Sensor readings themselves are questionable. Values may be stuck, erratic, or out of normal range. Exclude from analysis unless investigating the malfunction.

**connectivity** - Data transmission was intermittent. Individual readings that did transmit may be valid, but gaps in data make continuous analysis difficult.

**deployment_error** - Metadata (location, sensor ID) may be incorrect. Reading values might be fine, but spatial analysis is unreliable.

**Check issue_description for specifics** - provides additional context like "Temperature stuck at 32°F" or "Timestamp off by 2 hours" that helps you understand the exact nature of the problem.

## Checking Data Quality

### Before Analysis

Run these queries before starting your analysis to understand the quality of your dataset.

**Check overall data quality for a time period:**

```sql
SELECT 
    COUNT(*) AS total_readings,
    SUM(CASE WHEN quality_ok = 1 THEN 1 ELSE 0 END) AS valid_readings,
    SUM(CASE WHEN quality_ok = 0 THEN 1 ELSE 0 END) AS flagged_readings,
    CAST(SUM(CASE WHEN quality_ok = 1 THEN 1 ELSE 0 END) AS FLOAT) / COUNT(*) * 100 AS valid_percentage
FROM nu_readings
WHERE timestamp >= '2025-06-01 00:00:00'
    AND timestamp <= '2025-06-30 23:59:59';
```

**Identify sensors with quality issues in your date range:**

```sql
SELECT 
    s.box_id,
    s.location_address,
    COUNT(*) AS total_readings,
    SUM(CASE WHEN r.quality_ok = 0 THEN 1 ELSE 0 END) AS flagged_readings,
    CAST(SUM(CASE WHEN r.quality_ok = 0 THEN 1 ELSE 0 END) AS FLOAT) / COUNT(*) * 100 AS flagged_percentage
FROM nu_readings r
JOIN nu_sensors s ON r.deployment_id = s.deployment_id
WHERE r.timestamp >= '2025-06-01 00:00:00'
    AND r.timestamp <= '2025-06-30 23:59:59'
GROUP BY s.box_id, s.location_address
HAVING SUM(CASE WHEN r.quality_ok = 0 THEN 1 ELSE 0 END) > 0
ORDER BY flagged_percentage DESC;
```

**Check for NULL values (missing sensor readings):**

```sql
SELECT 
    s.box_id,
    s.location_address,
    COUNT(*) AS total_readings,
    SUM(CASE WHEN r.temperature IS NULL THEN 1 ELSE 0 END) AS missing_temperature,
    SUM(CASE WHEN r.humidity IS NULL THEN 1 ELSE 0 END) AS missing_humidity,
    SUM(CASE WHEN r.noise IS NULL THEN 1 ELSE 0 END) AS missing_noise
FROM nu_readings r
JOIN nu_sensors s ON r.deployment_id = s.deployment_id
WHERE r.timestamp >= '2025-06-01 00:00:00'
    AND r.timestamp <= '2025-06-30 23:59:59'
GROUP BY s.box_id, s.location_address
HAVING SUM(CASE WHEN r.temperature IS NULL THEN 1 ELSE 0 END) > 0
    OR SUM(CASE WHEN r.humidity IS NULL THEN 1 ELSE 0 END) > 0
    OR SUM(CASE WHEN r.noise IS NULL THEN 1 ELSE 0 END) > 0
ORDER BY s.box_id;
```

**View active quality issues that affect your analysis period:**

```sql
SELECT 
    qi.box_id,
    s.location_address,
    qi.issue_type,
    qi.issue_description,
    qi.start_datetime,
    qi.end_datetime,
    CASE WHEN qi.is_resolved = 1 THEN 'Resolved' ELSE 'Ongoing' END AS status
FROM nu_quality_issues qi
JOIN nu_sensors s ON qi.box_id = s.box_id AND s.is_active = 1
WHERE qi.start_datetime <= '2025-06-30 23:59:59'
    AND (qi.end_datetime IS NULL OR qi.end_datetime >= '2025-06-01 00:00:00')
ORDER BY qi.start_datetime;
```

**Check when data arrived in the pipeline (not when it was written into the database):**

Useful for investigating delays between sensor collection and pipeline processing.

```sql
SELECT 
    timestamp AS sensor_time,
    created_at AS database_time,
    source_blob,
    DATEDIFF(MINUTE, timestamp, created_at) AS processing_delay_minutes
FROM nu_readings
WHERE deployment_id = 19
    AND source_blob >= '19_20251107'
    AND source_blob < '19_20251108'
ORDER BY source_blob;
```

### Common Quality Patterns

Certain patterns in sensor data indicate specific types of problems. Recognizing these helps you identify issues even without explicit quality flags.

**Large data gaps** - No readings for extended periods (hours or days)

*Indicates:* Sensor offline, connectivity issues, power problems

*Check with:*

```sql
SELECT 
    box_id,
    timestamp AS gap_start,
    LEAD(timestamp) OVER (PARTITION BY box_id ORDER BY timestamp) AS gap_end,
    DATEDIFF(MINUTE, timestamp, LEAD(timestamp) OVER (PARTITION BY box_id ORDER BY timestamp)) AS gap_minutes
FROM nu_readings
WHERE deployment_id = 19
    AND timestamp >= '2025-06-01 00:00:00'
    AND DATEDIFF(MINUTE, timestamp, LEAD(timestamp) OVER (PARTITION BY box_id ORDER BY timestamp)) > 60  -- Gaps over 1 hour
ORDER BY gap_minutes DESC;
```

**Stuck values** - Same reading repeated many times consecutively

*Indicates:* Sensor malfunction, frozen sensor, communication error

*Example pattern:* Temperature reads exactly 72.5°F for 6 hours straight

**Extreme outliers** - Values far outside normal range

*Indicates:* Sensor malfunction, calibration error, data corruption

*Example patterns:*

- Temperature: -40°F or 150°F in Boston
- Humidity: 0% or values above 100%
- Noise: Constant 0 dB or extreme spikes

**Impossible value combinations** - Physically implausible data

*Indicates:* Sensor or calculation error

*Example:* Heat index lower than temperature, or heat index at 32°F with 90% humidity

**Sudden value jumps** - Abrupt changes not explained by weather

*Indicates:* Sensor reset, calibration change, hardware replacement

*Example:* Temperature jumps from 75°F to 32°F in one minute

**Regular patterns in noise data** - Repeating noise cycles unrelated to actual environment

*Indicates:* Electrical interference, sensor resonance, faulty microphone

If you notice these patterns, consider reporting them so quality issues can be documented in nu_quality_issues table. Contact {{ contacts.technical_administrator.name }} to flag problematic data.

## Additional Resources

**Managing Quality Issues:**

- [Recording New Quality Issues](../../04-making-changes/database-changes.md#recording-new-quality-issues) - How to document newly discovered data problems
- [Resolving Quality Issues](../../04-making-changes/database-changes.md#resolving-quality-issues) - Marking issues as fixed after sensor repair
- [Flagging Data Retroactively](../../04-making-changes/database-changes.md#flagging-data-during-quality-issue-periods) - Updating quality_ok for historical data

**Query Examples:**

- [Common Queries - Filtering and Quality](common-queries.md#filtering-and-quality) - SQL examples for excluding flagged data
- [Common Queries - Data Export](common-queries.md#data-export) - Exporting data with quality indicators

**Understanding the Schema:**

- [nu_readings Table](understanding-schema.md#nu_readings-environmental-data) - Complete documentation of quality_ok column
- [nu_quality_issues Table](understanding-schema.md#nu_quality_issues-data-quality-tracking) - How quality issues are tracked
