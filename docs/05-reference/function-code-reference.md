# Function Code Reference

This page documents the Azure Function Apps used by the sensor data pipeline. It explains what each Function App does, which resources it can access, and how the functions interact with Blob Storage, SQL Database, and the public dashboard.

For deployment procedures, see [Deploying Code Updates](../04-making-changes/deploying-code-updates.md). For Azure resource configuration, see [Function Apps Infrastructure Reference](azure-infrastructure/function-apps.md).

## Overview

The system uses three main Function App resources:

| Function App                     | Main Role                                                                 | Trigger Type | Access Level                                              |
| -------------------------------- | ------------------------------------------------------------------------- | ------------ | --------------------------------------------------------- |
| `sensordata-func-sensor-rec-...` | Receives Particle webhook payloads and writes parsed data to Blob Storage | HTTP         | Public endpoint, Blob Storage write access                |
| `sensordata-func-db-writer-...`  | Processes blobs, writes records to SQL Database, and sends daily reports  | Timer        | Private/internal, Blob Storage read/write, SQL read/write |
| `sensordata-func-api-...`        | Serves dashboard API requests from SQL Database                           | HTTP         | Public endpoint, SQL read access                          |

Each Function App may contain one or more individual Azure Functions. The code is organized so each Function App has a narrow responsibility and only the permissions needed for that responsibility.

---

## `sensordata-func-sensor-rec-...`

The sensor receiver Function App is the public ingestion point for Particle.io webhook messages. It receives raw sensor payloads, validates the request, parses the payload, checks for duplicates and alert conditions, then writes the parsed result to Blob Storage.

### Functions

| Function          | Route          | Trigger   | Purpose                                                               |
| ----------------- | -------------- | --------- | --------------------------------------------------------------------- |
| `webhook_handler` | `/api/webhook` | HTTP POST | Receives Particle webhook payloads and writes parsed blobs to storage |

### Responsibilities

The webhook receiver is responsible for:

* Accepting HTTP POST requests from Particle Cloud
* Validating required Particle webhook fields
* Parsing the raw `data` field into a classified payload
* Detecting duplicate webhook deliveries
* Checking whether an operational alert should be sent
* Writing parsed JSON blobs to Blob Storage
* Returning a JSON response to Particle Cloud

### Input

The function expects a Particle-style JSON payload containing at least:

```json
{
  "event": "<event-name>",
  "data": "<raw-sensor-message>",
  "coreid": "<particle-device-id>",
  "published_at": "<particle-published-timestamp>"
}
```

Optional fields may also be present in the webhook payload, but these four fields are required for processing.

### Output

If processing succeeds, the function writes a JSON blob to the appropriate Blob Storage folder and returns:

```json
{
  "status": "received",
  "box_id": "<box-id-or-unknown>"
}
```

If the message is detected as a duplicate, it returns:

```json
{
  "status": "duplicate"
}
```

Invalid JSON or missing required keys return HTTP 400. Blob connection or upload failures return HTTP 500.

### Blob Storage Interaction

This Function App writes to Blob Storage but does not interact with the SQL Database directly.

The parsed payload is routed by message type:

* `environment/`
* `error/`
* `startup/`
* `unknown/`

For recognized message types, complete and well-formed messages are written to `incoming/`. Recognizable but malformed messages are written to `malformed/`. Unknown messages are written directly to the `unknown/` folder.

### Duplicate Detection

The receiver includes webhook-level duplicate detection using the raw `data` field and the Particle `coreid`.

This mechanism exists because duplicate webhook deliveries can occur if Particle integrations are accidentally configured at multiple levels. When active, duplicate messages are rejected before being written to Blob Storage.

### Alert Handling

The function checks whether the parsed payload should trigger an operational alert. Alerts may be generated for malformed data, unknown formats, high latency, or upload errors.

Alert logs are stored in Blob Storage so repeated alerts from the same sensor can be rate-limited.

### Design Notes

The webhook receiver is intentionally lightweight. It validates and stores incoming data but does not write directly to the database. This design keeps ingestion resilient: if the database is unavailable, incoming data can still be preserved in Blob Storage and processed later by the Database Writer.

---

## `sensordata-func-db-writer-...`

The database writer Function App performs scheduled backend processing. It reads blobs from Blob Storage, validates and transforms them, writes records to SQL Database, moves processed blobs to their final folders, and sends operational reports.

### Functions

| Function             | Trigger | Schedule          | Purpose                                                     |
| -------------------- | ------- | ----------------- | ----------------------------------------------------------- |
| `database_populator` | Timer   | Every 30 minutes  | Processes incoming blobs and writes records to SQL Database |
| `daily_report`       | Timer   | Daily at 6:00 UTC | Sends previous-day sensor performance report                |

### `database_populator`

The `database_populator` function is the main processing function for the pipeline.

#### Responsibilities

This function is responsible for:

* Connecting to Blob Storage
* Connecting to SQL Database
* Loading active sensor deployment mappings
* Reading blobs from `incoming/`
* Processing `environment`, `error`, and `startup` data separately
* Detecting duplicate blobs within the current batch
* Validating and transforming blob contents into database records
* Writing records to the appropriate SQL tables
* Moving blobs to `archived/`, `duplicated/`, `failed-processing/`, or `failed-writing/`
* Sending alerts when processing, writing, or blob movement fails

#### Processing Flow

For each data type (`environment`, `error`, `startup`), the function follows this sequence:

1. Read blobs from the corresponding `incoming/` folder.
2. Detect duplicate blobs within the current batch.
3. Move duplicated blobs to `duplicated/`.
4. Process valid blobs into database-ready records.
5. Move blobs that fail validation to `failed-processing/`.
6. Write successfully processed records to SQL Database.
7. Move successfully written blobs to `archived/`.
8. Move blobs with database integrity conflicts to `failed-writing/`.
9. Leave blobs in `incoming/` if a network/database connection error occurs, so they can be retried on the next run.

#### Database Interaction

This Function App has both read and write access to the SQL Database.

It reads from:

* `nu_sensors`
* `nu_quality_issues`
* diagnostic queries used by the daily report

It writes to:

* `nu_readings`
* `nu_errors`
* `nu_startup`

The function uses the active deployment mapping from `nu_sensors` to assign each incoming reading to the correct `deployment_id`.

#### Error Handling

The function distinguishes between recoverable and non-recoverable failures.

Network or connection failures leave blobs in `incoming/` so the next scheduled run can retry them.

Validation or integrity problems move blobs out of `incoming/` to avoid repeated failures:

* `failed-processing/` for parsing, validation, or timestamp overlap problems detected before database insertion
* `failed-writing/` for records that pass processing but fail during database insertion
* `duplicated/` for duplicate blobs within the processing batch

### `daily_report`

The `daily_report` function sends a daily operational summary to the research team.

#### Responsibilities

This function is responsible for:

* Calculating the previous day’s reporting window in Eastern Time
* Querying the database for records written during that period
* Identifying sensors with missing or underperforming data
* Including uninstalled sensors in the summary
* Sending a daily email report

#### Database Interaction

This function only reads from the database. It does not modify sensor data.

Typical report content includes:

* Total records written
* Records written per sensor
* Sensors with no data
* Underperforming sensors
* Uninstalled sensors
* Largest data gaps or sensor health indicators, depending on report configuration

### Design Notes

The Database Writer is separated from the Webhook Receiver so database outages do not cause immediate data loss. Blob Storage acts as a durable queue between ingestion and database insertion.

Processing each data type separately also limits failure scope. A problem with one type of data does not necessarily block processing of the others.

---

## `sensordata-func-api-...`

The API Function App serves the public dashboard. It exposes HTTP endpoints that query the SQL Database and return JSON responses to the frontend.

### Functions

| Function            | Route                    | Trigger  | Purpose                                                                           |
| ------------------- | ------------------------ | -------- | --------------------------------------------------------------------------------- |
| `location_metadata` | `/api/location-metadata` | HTTP GET | Returns metadata for a selected sensor location                                   |
| `readings`          | `/api/readings`          | HTTP GET | Returns sensor readings for a metric, location, time range, and aggregation level |
| `sensors_list`      | `/api/sensors-list`      | HTTP GET | Returns available sensor locations for dashboard selection                        |
| `health_check`      | `/api/health`            | HTTP GET | Returns basic service health status                                               |

### Responsibilities

The API Function App is responsible for:

* Accepting dashboard requests
* Validating query parameters
* Connecting to SQL Database
* Running read-only queries
* Formatting responses as JSON
* Returning appropriate HTTP status codes
* Setting CORS headers for browser access
* Applying short browser cache headers where appropriate

### Database Access

This Function App should have read-only access to the SQL Database.

It queries:

* `nu_sensors`
* `nu_readings`

It does not write to the database or interact with Blob Storage.

### Main Endpoints

#### `/api/location-metadata`

Returns metadata for a specific location.

Expected query parameter:

```text
location_id
```

Used by the dashboard to display location-specific information.

#### `/api/readings`

Returns time-series sensor readings.

Expected query parameters:

```text
location_id
metric
start_date
end_date
aggregation
```

Supported metrics include:

* `temperature`
* `humidity`
* `heat_index`
* `noise`

Supported aggregation levels depend on implementation, but typically include raw/minute-level, hourly, and daily views.

#### `/api/sensors-list`

Returns the list of available sensor locations. This endpoint is used to populate dashboard selectors.

#### `/api/health`

Returns a basic health check response:

```json
{
  "status": "healthy",
  "timestamp": "<utc-timestamp>"
}
```

### CORS and Caching

The API includes CORS headers so the public dashboard can call the endpoints from the browser.

Some endpoints also include cache headers:

* Readings responses are cached briefly to reduce repeated database queries.
* Sensor list responses can be cached longer because location metadata changes less frequently.

### Design Notes

The API Function App is separated from the ingestion and processing functions to keep public dashboard traffic isolated from backend processing. It has only the permissions required to serve read-only data.

---

## Permission Model

The three Function Apps follow a least-privilege design.

| Function App    | Blob Storage                                                 | SQL Database | Public Access |
| --------------- | ------------------------------------------------------------ | ------------ | ------------- |
| Sensor Receiver | Write/read for ingestion, duplicate tracking, and alert logs | None         | Yes           |
| Database Writer | Read/write                                                   | Read/write   | No            |
| API             | None                                                         | Read-only    | Yes           |

This separation limits the impact of failures or exposed endpoints. Public-facing functions do not have unnecessary write permissions to the database.

---

## Code Organization

The Function App entry points are defined in `function_app.py`. Shared logic is organized into helper modules by responsibility.

Common module groups include:

| Module Area                                     | Purpose                                                    |
| ----------------------------------------------- | ---------------------------------------------------------- |
| `blob_storage/`                                 | Blob connection, upload, read, and move operations         |
| `database/`                                     | SQL connection, read queries, and write operations         |
| `shared/parse.py`                               | Payload and query parameter parsing                        |
| `shared/process.py`                             | Transformation of parsed blobs into database-ready records |
| `shared/alert.py`                               | Operational alert generation                               |
| `shared/report.py`                              | Daily report formatting and email delivery                 |
| `shared/duplicated.py` or `shared/duplicate.py` | Duplicate detection logic                                  |
| `shared/format.py`                              | API response formatting                                    |
| `shared/utils.py`                               | Shared date/time and helper utilities                      |

The actual function files contain inline comments for implementation-level details. This reference page is intended to explain the role and behavior of each Function App, not replace the source code.

---

## Operational Behavior Summary

| Scenario                                      | Expected Behavior                                                       |
| --------------------------------------------- | ----------------------------------------------------------------------- |
| Particle sends valid sensor data              | Webhook Receiver writes parsed blob to `incoming/`                      |
| Duplicate webhook payload received            | Webhook Receiver returns duplicate status and does not write a new blob |
| Blob is valid and database write succeeds     | Database Writer writes records and moves blob to `archived/`            |
| Blob fails validation                         | Database Writer moves blob to `failed-processing/`                      |
| Blob conflicts with existing database records | Database Writer moves blob to `failed-writing/`                         |
| Database is temporarily unavailable           | Database Writer leaves blobs in `incoming/` for retry                   |
| Dashboard requests data                       | API validates request, queries SQL, and returns JSON                    |
| API cannot connect to database                | API returns service unavailable response                                |

---

## Related Documentation

* [Data Flow](../01-understanding-the-system/data-flow.md)
* [Blob Storage Reference](azure-infrastructure/blob-storage.md)
* [Complete Schema Reference](complete-schema.md)
* [API Endpoints Reference](api-endpoints.md)
* [Deploying Code Updates](../04-making-changes/deploying-code-updates.md)
* [Networking Reference](azure-infrastructure/networking.md)
