# Database Changes

This page covers procedures for modifying existing database records, including sensor deployments, quality issues, and retroactive data corrections.

**Prerequisites:** This page assumes you have:

- Successfully connected to the database (see [Connecting to Database](../02-working-with-data/database/connecting.md))
- Write permissions on the database (contact IT if uncertain)
- Configured network access if needed (see [SQL Database Reference](../05-reference/azure-infrastructure/sql-database.md#network-access))
- A database management tool or programmatic access (see [Connection Tools](../02-working-with-data/database/connecting.md#connection-tools))

All queries shown can be executed either through a database manager (Azure Data Studio, SSMS) or programmatically using Python, PowerShell, or other database clients.

---

## Sensor Deployment Management

### Adding New Sensor Deployments

!!! warning "Sensor Tracking Spreadsheet"
    Physical sensor relocations require tracking in both the database and the master sensor tracking spreadsheet maintained by the research team. Before creating a new deployment:

    1. Contact the research team for access to the sensor tracking spreadsheet
    2. Update the spreadsheet with the new deployment information (usually done by the research team)
    3. Create the new deployment record in the database following the procedure above
    
    This ensures consistency between operational tracking and database records.

When a sensor is moved to a new location, you need to close the previous deployment and create a new one. This is a two-step process that must be done in order.

**Prerequisites:**

- Know the box_id of the sensor being moved
- Have the new location details (address, coordinates, coreid)
- Have the exact install_datetime for the new location
- Have the uninstall_datetime for the previous location

**Constraints:**

- Only one deployment per box_id can have `is_active = 1` at a time (enforced by unique index)
- The `deployment_id` is auto-incrementing and cannot be manually assigned
- You cannot add a new active deployment if the previous one is still marked `is_active = 1`

**Procedure:**

**Step 1: Close the previous deployment**

Set the old deployment to inactive and record when it was uninstalled:

    ```sql
    UPDATE nu_sensors
    SET 
        is_active = 0,
        uninstall_datetime = '2025-11-07 00:00:00'  -- UTC timestamp when sensor was removed
    WHERE box_id = 9 AND is_active = 1;
    ```

**Step 2: Add the new deployment**

Insert the new deployment record with the new location details:

    ```sql
    INSERT INTO nu_sensors (box_id, location_id, coreid, location_address, latitude, longitude, install_datetime)
    VALUES (
        9,                                    -- Same box_id
        NULL,                                 -- location_id (optional, verify with the research team its value)
        'e00fce68cd4bb76b1b85dbe7',           -- Particle device coreid
        'Savin & Tupelo',                     -- New location address
        42.31670,                             -- New latitude
        -71.08095,                            -- New longitude
        '2025-10-31 11:59:00'                 -- UTC timestamp when sensor installed at new location
    );
    ```

**Notes:**

- The new record automatically gets the next available `deployment_id` (you cannot specify it)
- The new record defaults to `is_active = 1` and `created_at = GETUTCDATE()`
- If Step 1 fails or is skipped, Step 2 will fail with a unique constraint violation

**Verification:**

Check that the deployment was added correctly:

    ```sql
    SELECT deployment_id, box_id, location_address, install_datetime, is_active
    FROM nu_sensors
    WHERE box_id = 9
    ORDER BY deployment_id DESC;
    ```

You should see:

- Old deployment with `is_active = 0` and `uninstall_datetime` populated
- New deployment with `is_active = 1` and `uninstall_datetime = NULL`

**After Adding Deployment:**

If data was already recorded under the old deployment_id after the sensor was moved, you'll need to update those readings. See [Updating deployment_id for Readings](#updating-deployment_id-for-readings) below.

---

### Correcting Deployment Information

Use these queries to fix typos or incorrect information in existing deployment records. These updates are for **correcting data entry errors only** - not for recording sensor movements.

**Important:** If a sensor has physically moved to a new location, you must create a new deployment record (see [Adding New Sensor Deployments](#adding-new-sensor-deployments) above). Do not update the existing deployment's location fields.

**Correcting Location Id:**

    ```sql
    UPDATE nu_sensors
    SET location_id = 'Corrected Location Id'
    WHERE deployment_id = 10;
    ```

**Correcting Location Address:**

    ```sql
    UPDATE nu_sensors
    SET location_address = 'Corrected Address'
    WHERE deployment_id = 10;
    ```

**Correcting Coordinates:**

    ```sql
    UPDATE nu_sensors
    SET 
        latitude = 42.31234,
        longitude = -71.09876
    WHERE deployment_id = 10;
    ```

**Correcting Install Datetime:**

    ```sql
    UPDATE nu_sensors
    SET install_datetime = '2025-10-31 12:00:00'
    WHERE deployment_id = 10;
    ```

**Correcting Particle Core ID:**

    ```sql
    UPDATE nu_sensors
    SET coreid = 'e00fce68cd4bb76b1b85dbe7'
    WHERE deployment_id = 10;
    ```

---

## Quality Issue Management

### Recording New Quality Issues

When you identify a data quality problem with a sensor, record it in the nu_quality_issues table. This automatically flags future readings from that sensor with `quality_ok = 0` during Database Writer processing.

**Common Issue Types:**

- `clock_drift` - Sensor timestamp significantly off from actual time
- `sensor_malfunction` - Hardware reporting implausible or erratic values
- `connectivity` - Intermittent communication problems causing data gaps
- `deployment_error` - Incorrect metadata or location information

**Adding a Quality Issue:**

    ```sql
    INSERT INTO nu_quality_issues (box_id, start_datetime, issue_type, issue_description)
    VALUES (
        19,                                    -- Sensor with the issue
        '2025-11-15 08:00:00',                 -- UTC timestamp when issue began
        'sensor_malfunction',                  -- Issue category
        'Temperature readings stuck at 32°F'   -- Detailed description
    );
    ```

**Notes:**

- Leave `end_datetime` as NULL - it will be set when the issue is resolved
- The `is_resolved` column automatically computes to 0 (unresolved)
- Future data from this sensor will be flagged with `quality_ok = 0`
- Existing data is NOT automatically flagged - see [Flagging Data During Quality Issue Periods](#flagging-data-during-quality-issue-periods) for retroactive flagging

**Writing Clear Descriptions:**

Good issue descriptions help researchers understand data quality problems:

    ```sql
    -- Good: Specific and actionable
    'Temperature readings consistently 10°F higher than nearby sensors'

    -- Good: Clear timeframe context
    'No data received after firmware update at 14:30 UTC'

    -- Poor: Too vague
    'Something wrong with sensor'
    ```

---

### Resolving Quality Issues

When a sensor issue is fixed (hardware replaced, firmware updated, etc.), mark the issue as resolved by setting the `end_datetime`.

**Marking an Issue as Resolved:**

    ```sql
    UPDATE nu_quality_issues
    SET end_datetime = '2025-11-18 12:00:00'  -- UTC timestamp when issue was fixed
    WHERE id = 5;
    ```

**Verification:**

The `is_resolved` computed column automatically updates to 1:

    ```sql
    SELECT id, box_id, issue_type, start_datetime, end_datetime, is_resolved
    FROM nu_quality_issues
    WHERE id = 5;
    ```

You should see `is_resolved = 1` and `end_datetime` populated.

**After Resolving:**

- Future readings from this sensor will no longer be automatically flagged
- Readings between `start_datetime` and `end_datetime` remain flagged with `quality_ok = 0`
- If readings after `end_datetime` were already written with `quality_ok = 0`, you may need to update them - see [Unflagging Data After Issue Resolution](#unflagging-data-after-issue-resolution)

---

## Retroactive Data Corrections

These procedures handle scenarios where database records need updating after data has already been written. Common situations include delayed deployment updates, retroactive quality issue recording, or discovering data problems after the fact.

!!! warning "Test First"
    Retroactive corrections can affect large numbers of records. Always:

    1. Run a SELECT query first to see how many records will be affected
    2. Verify the date ranges and box_id filters are correct
    3. Consider backing up affected data before major updates
    4. Test on a small subset if possible

### Updating deployment_id for Readings

**Scenario:** You moved a sensor to a new location but didn't add the new deployment record until days later. Readings were written using the old deployment_id, but they actually came from the new location.

**Solution:** Update readings to use the correct deployment_id based on the install_datetime.

**Step 1: Identify affected readings**

Find how many readings need updating:

    ```sql
    SELECT COUNT(*) as affected_readings
    FROM nu_readings r
    JOIN nu_sensors s_old ON r.deployment_id = s_old.deployment_id
    JOIN nu_sensors s_new ON r.box_id = s_new.box_id
    WHERE r.box_id = 9
    AND s_new.deployment_id = 56  -- New deployment
    AND r.timestamp >= s_new.install_datetime  -- Readings after new install
    AND r.deployment_id = s_old.deployment_id  -- Still using old deployment
    AND s_old.deployment_id != s_new.deployment_id;
    ```

**Step 2: Update the readings**

    ```sql
    UPDATE nu_readings
    SET deployment_id = 56  -- New deployment_id
    WHERE box_id = 9
    AND timestamp >= '2025-10-31 11:59:00'  -- install_datetime of new deployment
    AND deployment_id = 9;  -- Old deployment_id
    ```

**Example:**
Box 9 moved to new location on Oct 31 at 11:59 UTC (new deployment_id = 45), but the deployment record wasn't added until Nov 5. All readings from Oct 31 - Nov 5 were written with the old deployment_id = 44. This query reassigns those ~7,200 readings to the correct deployment.

---

### Flagging Data After Sensor Uninstall

**Scenario:** A sensor was uninstalled, but continued transmitting data temporarily (cached readings or delayed shutoff). This "ghost data" should be flagged as unreliable.

**Solution:** Flag all readings after uninstall_datetime as quality_ok = 0 for the specific deployment.

**Step 1: Identify the deployment and affected readings**

First, find the deployment_id for the uninstalled sensor:

    ```sql
    SELECT deployment_id, box_id, location_address, install_datetime, uninstall_datetime
    FROM nu_sensors
    WHERE box_id = 9
    AND uninstall_datetime IS NOT NULL
    ORDER BY deployment_id DESC;
    ```

Then check how many readings are affected:

    ```sql
    SELECT COUNT(*) as readings_after_uninstall
    FROM nu_readings r
    JOIN nu_sensors s ON r.deployment_id = s.deployment_id
    WHERE r.deployment_id = 9  -- Specific deployment that was uninstalled
    AND s.uninstall_datetime IS NOT NULL
    AND r.timestamp > s.uninstall_datetime
    AND r.quality_ok = 1;
    ```

**Step 2: Flag the readings**

    ```sql
    UPDATE nu_readings
    SET quality_ok = 0
    WHERE deployment_id = 9  -- Use specific deployment_id, not box_id
    AND timestamp > '2025-11-07 00:00:00'  -- uninstall_datetime from nu_sensors
    AND quality_ok = 1;
    ```

**Why use deployment_id:** A box_id can have multiple deployments over time. Using deployment_id ensures you only flag readings from the specific uninstalled deployment, not from earlier or later deployments of the same sensor.

---

### Flagging Data During Quality Issue Periods

**Scenario:** You discovered a sensor was malfunctioning last week and added a quality issue record today. Readings from last week are already in the database with quality_ok = 1, but should be flagged.

**Solution:** Update readings within the quality issue time range to quality_ok = 0.

**Step 1: Identify affected readings**

    ```sql
    SELECT COUNT(*) as affected_readings
    FROM nu_readings r
    JOIN nu_quality_issues qi ON r.box_id = qi.box_id
    WHERE qi.id = 5  -- Quality issue record ID
    AND r.timestamp >= qi.start_datetime
    AND (qi.end_datetime IS NULL OR r.timestamp <= qi.end_datetime)
    AND r.quality_ok = 1;
    ```

**Step 2: Flag the readings**

    ```sql
    UPDATE nu_readings
    SET quality_ok = 0
    WHERE box_id = 19
    AND timestamp >= '2025-11-15 08:00:00'  -- start_datetime from quality issue
    AND timestamp <= '2025-11-18 12:00:00'  -- end_datetime from quality issue (or remove this line if ongoing)
    AND quality_ok = 1;
    ```

---

### Unflagging Data After Issue Resolution

**Scenario:** A quality issue was resolved days ago (end_datetime set), but you just ran the Database Writer which flagged new readings because it checked for unresolved issues before you updated end_datetime. Or readings after the resolution were already in the database when you added the quality issue.

**Solution:** Set quality_ok = 1 for readings after the issue was resolved.

**Step 1: Identify affected readings**

    ```sql
    SELECT COUNT(*) as incorrectly_flagged
    FROM nu_readings r
    JOIN nu_quality_issues qi ON r.box_id = qi.box_id
    WHERE qi.id = 5  -- Quality issue that was resolved
    AND qi.end_datetime IS NOT NULL  -- Issue is resolved
    AND r.timestamp > qi.end_datetime  -- Readings after resolution
    AND r.quality_ok = 0;
    ```

**Step 2: Unflag the readings**

    ```sql
    UPDATE nu_readings
    SET quality_ok = 1
    WHERE box_id = 19
    AND timestamp > '2025-11-18 12:00:00'  -- end_datetime from quality issue
    AND quality_ok = 0;
    ```

**Warning:** This assumes no OTHER quality issues affect this sensor. Check for other active issues:

    ```sql
    SELECT id, issue_type, start_datetime, end_datetime, is_resolved
    FROM nu_quality_issues
    WHERE box_id = 19
    AND id != 5  -- Exclude the issue we just resolved
    AND is_resolved = 0;  -- Check for other unresolved issues
    ```

If other unresolved issues exist for this sensor, unflagging all quality_ok = 0 readings may be incorrect.

---

<!-- ### Schema Modifications

**Adding New Columns**

- Procedures for schema migrations
- Testing changes in development environment
- Deployment to production

**Creating New Indexes**

- When to add indexes
- Testing index performance impact
- Monitoring index usage

### Backup and Rollback

**Before Making Changes**

- Creating manual backups for critical updates
- Testing queries in read-only mode first
- Documenting changes made

**Rolling Back Changes**

- Identifying incorrect updates
- Restoring from backup if needed
- Audit trail considerations -->
