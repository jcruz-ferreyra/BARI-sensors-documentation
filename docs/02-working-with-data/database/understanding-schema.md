# Understanding the Schema

This page explains the database schema, including table relationships, column purposes, and key concepts for working with sensor data.

For complete technical specifications including table creation statements and DDL code, see [Complete Schema Reference](../../05-reference/complete-schema.md).

## Overview

The database contains five main tables:

- **nu_sensors** - Metadata about sensor deployments and locations
- **nu_readings** - Environmental sensor readings (temperature, humidity, noise)
- **nu_errors** - Error messages from sensors
- **nu_startup** - Sensor boot/restart logs
- **nu_quality_issues** - Data quality problems and manual corrections

---

## nu_sensors (Metadata)

This table tracks sensor deployments. A "deployment" represents a specific sensor (box_id) installed at a specific location during a specific time period. When a sensor is moved to a new location, it gets a new deployment_id.

### Columns

**deployment_id** (PK, int, not null)
Unique identifier for this deployment. Referenced by nu_readings to link sensor data to physical locations.

**box_id** (int, not null)
Physical sensor identifier (1-55). This number is printed on the sensor hardware and appears in sensor messages.

**location_id** (int, null)
Optional reference to a standardized location. Currently not used.

**coreid** (nvarchar(255), not null)
Particle device ID - unique hardware identifier from Particle.io platform. Used to validate webhook data comes from legitimate sensors.

**location_address** (nvarchar(255), not null)
Human-readable address or landmark description of where the sensor is installed (e.g., "Siever St & Tiffany Moore Tot Park").

**latitude** (decimal(10,8), not null)
Geographic latitude of sensor location in decimal degrees.

**longitude** (decimal(11,8), not null)
Geographic longitude of sensor location in decimal degrees.

**install_datetime** (datetime2(7), not null)
UTC timestamp when the sensor was installed at this location.

**is_active** (bit, null)
Current deployment status: 1 = active, 0 = deactivated. Only one deployment per box_id should have is_active = 1 at any time.

**created_at** (datetime2(7), null)
UTC timestamp when this database record was created (may differ from install_datetime).

**uninstall_datetime** (datetime2(7), null)
UTC timestamp when the sensor was removed from this location. NULL for currently active deployments.

### Example Record

```
deployment_id:      1
box_id:             1
location_id:        NULL
coreid:             e00fce688d4f56458911d5b9
location_address:   Siever st & Tiffany Moore Tot Park
latitude:           42.30900000
longitude:          -71.09100000
install_datetime:   2025-06-20 13:17:00
is_active:          1
created_at:         2025-10-28 20:42:19
uninstall_datetime: NULL
```

This shows sensor Box 1 actively deployed at Siever St since June 20, 2025.

### Indexes

**UQ_active_box_id** (Unique filtered index on box_id where is_active = 1)
Ensures only one active deployment exists per sensor at any time. This index prevents accidentally marking multiple deployments as active for the same box_id, which would cause the Database Writer to fail when trying to determine which deployment_id to use for incoming data.

### Key Concepts

**Deployment vs Sensor:** A sensor (box_id) can have multiple deployments over time. When sensor Box 1 moves from Location A to Location B, it gets two deployment records - the first with uninstall_datetime filled, the second with is_active = 1.

**Active Deployments:** The Database Writer function queries this table to find the current deployment_id for incoming sensor data based on box_id and is_active = 1.

**Location History:** Historical deployments (is_active = 0, uninstall_datetime populated) preserve location history for data analysis.

**Managing Deployments:** When adding new deployments or marking sensors as uninstalled, existing readings may need their deployment_id or quality_ok flags updated retroactively. See [Database Changes](../../04-making-changes/database-changes.md#sensor-deployment-management) for procedures on adding deployments, uninstalling sensors, and correcting historical data.

---

## nu_readings (Environmental Data)

This table stores environmental sensor readings with one record per minute per sensor. This is the primary data table for research analysis and powers the public dashboard visualizations.

### Columns

**id** (PK, bigint, not null)
Unique identifier for each reading. Auto-incrementing to handle large data volumes (billions of records over time).

**deployment_id** (FK, int, not null)
Links reading to sensor deployment in nu_sensors table. Determines which physical location the reading came from.

**box_id** (int, not null)
Physical sensor identifier (1-55). Duplicated from deployment for query performance - allows fast filtering without joining to nu_sensors.

**timestamp** (datetime2(7), not null)
UTC timestamp when the reading was collected by the sensor. Each sensor produces one reading per minute.

**temperature** (float, null)
Temperature in Fahrenheit. NULL if sensor malfunction or data quality issue detected.

**humidity** (float, null)
Relative humidity as percentage (0-100). NULL if sensor malfunction or data quality issue detected.

**noise** (float, null)
Noise level in decibels (dB). NULL if sensor malfunction or data quality issue detected.

**heat_index** (float, null)
Calculated heat index in Fahrenheit using temperature and humidity. Computed by Database Writer using the [NWS equation](https://www.wpc.ncep.noaa.gov/html/heatindex_equation.shtml).

**source_blob** (nvarchar(255), not null)
Filename of the blob containing the raw sensor data for this reading. Enables tracing back to original data if issues arise.

**created_at** (datetime2(7), null)
UTC timestamp when this record was written to the database. Used by Daily Reporter to track data pipeline performance.

**quality_ok** (bit, null)
Data quality flag: 1 = valid data, 0 = questionable data. Set to 0 when sensor issues detected or manual corrections needed. See nu_quality_issues table for details.

### Example Record

```
id:             1
deployment_id:  10
box_id:         10
timestamp:      2025-05-03 23:17:00
temperature:    50.5
humidity:       92.1
noise:          40.4
heat_index:     50.04
source_blob:    10_20251028T144801Z
created_at:     2025-11-05 15:58:40
quality_ok:     0
```

This shows a reading from Box 10 collected at 23:17 UTC on May 3rd. Note quality_ok = 0, indicating this reading has been flagged for data quality issues (see nu_quality_issues table for details).

### Indexes

**IX_readings_deployment_id_timestamp**
Optimizes API queries that filter by deployment and time range - the most common query pattern for dashboard requests.

**IX_readings_box_id**
Enables fast lookups by sensor identifier without needing to join to nu_sensors.

**IX_readings_created_box**
Supports Daily Reporter queries that analyze recently written data grouped by sensor.

**UK_readings_box_timestamp** (Unique constraint)
Prevents duplicate readings for the same sensor at the same timestamp. Critical for data integrity during Database Writer processing.

### Key Concepts

**One Reading Per Minute:** Sensors collect data every minute, so a healthy sensor produces 1,440 readings per day (24 hours × 60 minutes).

**Denormalized box_id:** While deployment_id provides the foreign key relationship, box_id is duplicated here for query performance. This avoids joining to nu_sensors for simple box-based queries.

**Quality Flagging:** The quality_ok flag marks problematic data without deleting it. Researchers can choose whether to include quality_ok=0 records in their analysis.

**Data Volume:** With {{ sensors.active_count }} active sensors, this table grows by approximately {{ data.readings_per_day }} records per day ({{ data.readings_per_month }} per month).

**Relationship to Deployments:** When a sensor moves locations, new readings use the new deployment_id. Historical readings retain their original deployment_id, preserving accurate location history.

---

## nu_quality_issues (Data Quality Tracking)

This table tracks known data quality problems for specific sensors during specific time periods. When issues are recorded here, the Database Writer automatically flags affected readings in nu_readings with quality_ok = 0.

### Columns

**id** (PK, bigint, not null)
Unique identifier for each quality issue record.

**box_id** (int, not null)
Sensor experiencing the quality issue.

**start_datetime** (datetime2(7), not null)
UTC timestamp when the quality issue began.

**end_datetime** (datetime2(7), null)
UTC timestamp when the issue was resolved. NULL indicates an ongoing issue.

**issue_type** (nvarchar(50), not null)
Category of quality issue. Common types: 'clock_drift' (timestamp problems), 'sensor_malfunction' (hardware issues), 'connectivity' (communication problems).

**issue_description** (nvarchar(500), null)
Optional detailed notes about the issue, useful for understanding context during data analysis.

**is_resolved** (computed, int, not null)
Automatically calculated: 1 if end_datetime is set (issue resolved), 0 if NULL (ongoing issue).

### Example Record

```
id:                  1
box_id:              1
start_datetime:      2025-06-20 13:17:00
end_datetime:        NULL
issue_type:          clock_drift
issue_description:   Timestamp off by less than 1 hour
is_resolved:         0
```

This shows Box 1 has an ongoing clock drift issue since June 20, 2025.

### Key Concepts

**Automatic Quality Flagging:** When the Database Writer processes sensor data, it queries this table for unresolved issues (is_resolved = 0). Any readings from affected sensors get quality_ok = 0 in nu_readings.

**Resolving Issues:** When a sensor issue is fixed, set end_datetime to mark when the problem ended. The is_resolved column automatically updates to 1.

**Historical Tracking:** Resolved issues (is_resolved = 1) remain in the table to document data quality history. This helps researchers understand why certain time periods have quality_ok = 0.

**Impact on Analysis:** Researchers can choose to exclude quality_ok = 0 readings from analysis, or investigate them separately. The issue_type and issue_description fields provide context for making these decisions.

**Manual Flagging:** Quality issues can be added retroactively. When added, a separate script can update existing nu_readings records to set quality_ok = 0 for the affected time period. See [Database Changes](../../04-making-changes/database-changes.md#flagging-quality-issues) for procedures.

---

## Additional Tables

The database also contains **nu_errors** and **nu_startup** tables for diagnostic and troubleshooting purposes. These tables are rarely queried during normal data analysis workflows and are primarily used for:

- Tracing when sensors began malfunctioning
- Investigating connectivity issues
- Debugging sensor firmware problems
- Tracking sensor restart patterns

For complete specifications of these tables, see [Complete Schema Reference](../../05-reference/complete-schema.md).
