# Complete Schema Reference

This page provides the complete technical specification of the database schema, including all tables, columns, data types, constraints, and indexes. This is the authoritative reference for database structure.

For a practical guide to working with the schema, see [Understanding the Schema](../02-working-with-data/database/understanding-schema.md).

## Table Overview

The database contains five main tables:

| Table | Purpose | Primary Key |
|-------|---------|-------------|
| nu_sensors | Sensor deployment metadata and locations | deployment_id |
| nu_readings | Environmental sensor readings | id |
| nu_errors | Error messages from sensors | id |
| nu_startup | Sensor boot/restart logs | id |
| nu_quality_issues | Data quality problems and corrections | id |

All timestamp columns use `DATETIME2(7)` and store values in UTC.

---

## nu_sensors

Tracks sensor deployments. Each deployment represents a specific sensor (box_id) installed at a specific location during a specific time period.

### Table Creation

```sql
CREATE TABLE nu_sensors (
    deployment_id INT IDENTITY(1,1) PRIMARY KEY,
    box_id INT NOT NULL,
    location_id INT NULL,
    coreid NVARCHAR(255) NOT NULL,
    location_address NVARCHAR(255) NOT NULL,
    latitude DECIMAL(10, 8) NOT NULL,
    longitude DECIMAL(11, 8) NOT NULL,
    install_datetime DATETIME2 NOT NULL,
    is_active BIT DEFAULT 1,
    created_at DATETIME2 DEFAULT GETUTCDATE(),
    uninstall_datetime DATETIME2 NULL
);
```

### Indexes

```sql
-- Ensures only one active deployment per sensor
CREATE UNIQUE INDEX UQ_active_box_id 
ON nu_sensors (box_id) 
WHERE is_active = 1;
```

### Column Specifications

| Column | Type | Nullable | Description |
|--------|------|----------|-------------|
| deployment_id | INT | NOT NULL | Primary key, auto-incrementing unique identifier |
| box_id | INT | NOT NULL | Physical sensor identifier (1-55) |
| location_id | INT | NULL | Optional standardized location reference (unused) |
| coreid | NVARCHAR(255) | NOT NULL | Particle device ID for webhook validation |
| location_address | NVARCHAR(255) | NOT NULL | Human-readable location description |
| latitude | DECIMAL(10,8) | NOT NULL | Geographic latitude in decimal degrees |
| longitude | DECIMAL(11,8) | NOT NULL | Geographic longitude in decimal degrees |
| install_datetime | DATETIME2 | NOT NULL | UTC timestamp of sensor installation |
| is_active | BIT | NULL | Deployment status: 1=active, 0=inactive (default: 1) |
| created_at | DATETIME2 | NULL | UTC timestamp when record was created (default: GETUTCDATE()) |
| uninstall_datetime | DATETIME2 | NULL | UTC timestamp when sensor was removed |

### Constraints and Business Rules

- Only one deployment per box_id can have `is_active = 1` (enforced by UQ_active_box_id index)
- When a sensor is moved, the old deployment gets `is_active = 0` and `uninstall_datetime` populated
- Database Writer queries this table to map box_id to deployment_id for incoming data

---

---

## nu_readings

Stores environmental sensor readings with one record per minute per sensor. This is the primary data table queried by the API and dashboard.

### Table Creation

```sql
CREATE TABLE nu_readings (
    id BIGINT IDENTITY(1,1) PRIMARY KEY,
    deployment_id INT NOT NULL,
    box_id INT NOT NULL,
    timestamp DATETIME2 NOT NULL,
    temperature FLOAT NULL,
    humidity FLOAT NULL,
    noise FLOAT NULL,
    heat_index FLOAT NULL,
    source_blob NVARCHAR(255) NOT NULL,
    created_at DATETIME2 DEFAULT GETUTCDATE(),
    quality_ok BIT DEFAULT 1,
    CONSTRAINT UK_readings_box_timestamp 
        UNIQUE (box_id, timestamp),
    FOREIGN KEY (deployment_id) REFERENCES nu_sensors(deployment_id)
);
```

### Indexes

```sql
-- Query performance for API requests (deployment + time range)
CREATE INDEX IX_readings_deployment_id_timestamp 
ON nu_readings (deployment_id, timestamp);

-- Lookup by box_id
CREATE INDEX IX_readings_box_id 
ON nu_readings (box_id);

-- Daily report queries (recent data by sensor)
CREATE INDEX IX_readings_created_box
ON nu_readings (created_at, box_id);
```

### Column Specifications

| Column | Type | Nullable | Description |
|--------|------|----------|-------------|
| id | BIGINT | NOT NULL | Primary key, auto-incrementing unique identifier |
| deployment_id | INT | NOT NULL | Foreign key to nu_sensors, links reading to location |
| box_id | INT | NOT NULL | Physical sensor identifier for quick lookups |
| timestamp | DATETIME2(7) | NOT NULL | UTC timestamp when reading was collected |
| temperature | FLOAT | NULL | Temperature in Fahrenheit |
| humidity | FLOAT | NULL | Relative humidity as percentage (0-100) |
| noise | FLOAT | NULL | Noise level in decibels (dB) |
| heat_index | FLOAT | NULL | Calculated heat index in Fahrenheit |
| source_blob | NVARCHAR(255) | NOT NULL | Blob filename containing raw data for traceability |
| created_at | DATETIME2 | NULL | UTC timestamp when record was written to database |
| quality_ok | BIT | NULL | Data quality flag: 1=valid, 0=questionable (default: 1) |

### Constraints and Business Rules

- **UK_readings_box_timestamp:** Prevents duplicate readings for same sensor at same timestamp
- **Foreign Key:** deployment_id must exist in nu_sensors table
- Sensor values can be NULL if sensor malfunction or data corruption detected
- quality_ok flag set to 0 when data quality issues identified (see nu_quality_issues table)
- Box_id duplicates deployment_id information for query performance (denormalized)

### Typical Record Volume

With {{ sensors.active_count }} sensors collecting data every minute:

- ~{{ data.readings_per_day }} readings per day
- ~{{ data.readings_per_month }} readings per month

---

## nu_quality_issues

Tracks known data quality problems for specific sensors during specific time periods. Used to automatically flag questionable data in nu_readings table.

### Table Creation

```sql
CREATE TABLE nu_quality_issues (
    id BIGINT IDENTITY(1,1) PRIMARY KEY,
    box_id INT NOT NULL,
    start_datetime DATETIME2 NOT NULL,
    end_datetime DATETIME2 NULL,
    issue_type NVARCHAR(50) NOT NULL,
    issue_description NVARCHAR(500) NULL,
    is_resolved AS CASE WHEN end_datetime IS NOT NULL THEN 1 ELSE 0 END PERSISTED
);
```

### Indexes

```sql
-- Query unresolved issues by sensor
CREATE INDEX IX_quality_issues_box_resolved 
ON nu_quality_issues (box_id, is_resolved);

-- Filter by issue type
CREATE INDEX IX_quality_issues_type 
ON nu_quality_issues (issue_type);

-- Find issues overlapping with specific time ranges
CREATE INDEX IX_quality_issues_datetime 
ON nu_quality_issues (start_datetime, end_datetime);
```

### Column Specifications

| Column | Type | Nullable | Description |
|--------|------|----------|-------------|
| id | BIGINT | NOT NULL | Primary key, auto-incrementing unique identifier |
| box_id | INT | NOT NULL | Sensor with the quality issue |
| start_datetime | DATETIME2 | NOT NULL | UTC timestamp when issue began |
| end_datetime | DATETIME2 | NULL | UTC timestamp when issue resolved (NULL if ongoing) |
| issue_type | NVARCHAR(50) | NOT NULL | Issue category (e.g., 'clock_drift', 'sensor_malfunction', 'connectivity') |
| issue_description | NVARCHAR(500) | NULL | Optional detailed notes about the issue |
| is_resolved | INT (computed) | NOT NULL | Computed: 1 if end_datetime is set, 0 if ongoing |

### Constraints and Business Rules

- **is_resolved** is a computed column - automatically updates based on end_datetime
- Multiple overlapping issues can exist for the same sensor
- Database Writer checks this table before inserting readings to set quality_ok flag

### Common Issue Types

- `clock_drift` - Sensor timestamp off by significant amount
- `sensor_malfunction` - Hardware reporting implausible values
- `connectivity` - Intermittent communication problems
- `deployment_error` - Incorrect metadata or location information

---

## nu_errors

Stores error messages reported by sensors. Used primarily for diagnostics and troubleshooting sensor malfunctions.

### Table Creation

```sql
CREATE TABLE nu_errors (
    id BIGINT IDENTITY(1,1) PRIMARY KEY,
    box_id INT NOT NULL,
    timestamp DATETIME2 NOT NULL,
    error_code NVARCHAR(50) NULL,
    source_blob NVARCHAR(255) NOT NULL,
    created_at DATETIME2 DEFAULT GETUTCDATE(),
    CONSTRAINT UK_error_readings_box_timestamp 
        UNIQUE (box_id, timestamp)
);
```

### Indexes

```sql
-- Query errors by sensor and time range
CREATE INDEX IX_error_box_timestamp 
ON nu_errors (box_id, timestamp);
```

### Column Specifications

| Column | Type | Nullable | Description |
|--------|------|----------|-------------|
| id | BIGINT | NOT NULL | Primary key, auto-incrementing unique identifier |
| box_id | INT | NOT NULL | Sensor that reported the error |
| timestamp | DATETIME2 | NOT NULL | UTC timestamp when error occurred |
| error_code | NVARCHAR(50) | NULL | Error code or type from sensor firmware |
| source_blob | NVARCHAR(255) | NOT NULL | Blob filename containing original error message |
| created_at | DATETIME2 | NULL | UTC timestamp when record was written to database |

### Constraints and Business Rules

- **UK_error_readings_box_timestamp:** Prevents duplicate error records for same sensor at same timestamp
- Errors are written by Database Writer when processing error message blobs from Particle webhooks
- Used primarily for troubleshooting - helps identify when sensors began malfunctioning

---

## nu_startup

Stores sensor startup/boot events. Used to track when sensors restart, which can indicate power issues, firmware updates, or connectivity problems.

### Table Creation

```sql
CREATE TABLE nu_startup (
    id BIGINT IDENTITY(1,1) PRIMARY KEY,
    box_id INT NOT NULL,
    timestamp DATETIME2 NOT NULL,
    source_blob NVARCHAR(255) NOT NULL,
    created_at DATETIME2 DEFAULT GETUTCDATE(),
    CONSTRAINT UK_startup_readings_box_timestamp 
        UNIQUE (box_id, timestamp)
);
```

### Indexes

```sql
-- Query startup events by sensor and time range
CREATE INDEX IX_startup_box_timestamp 
ON nu_startup (box_id, timestamp);
```

### Column Specifications

| Column | Type | Nullable | Description |
|--------|------|----------|-------------|
| id | BIGINT | NOT NULL | Primary key, auto-incrementing unique identifier |
| box_id | INT | NOT NULL | Sensor that restarted |
| timestamp | DATETIME2 | NOT NULL | UTC timestamp when sensor booted |
| source_blob | NVARCHAR(255) | NOT NULL | Blob filename containing original startup message |
| created_at | DATETIME2 | NULL | UTC timestamp when record was written to database |

### Constraints and Business Rules

- **UK_startup_readings_box_timestamp:** Prevents duplicate startup records for same sensor at same timestamp
- Startup events written by Database Writer when processing startup message blobs from Particle webhooks
- Frequent restarts may indicate power supply issues, connectivity problems, or environmental conditions affecting sensor hardware
- Useful for correlating data gaps in nu_readings with sensor restart patterns

---
