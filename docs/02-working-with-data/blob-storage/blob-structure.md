# Blob Storage

This page documents the organization and purpose of the Azure Blob Storage container used to stage sensor data between the Webhook Receiver and the Database Writer. For how data moves through these folders, see [Data Flow](../../01-understanding-the-system/data-flow.md). For the Webhook Receiver and Database Writer logic, see [Function Apps Reference](function-apps.md).

## Container Overview

All sensor data passes through a single Blob Storage container: `nu-sensor-data`. The container acts as a buffer between the Webhook Receiver (which writes blobs) and the Database Writer (which reads and processes them). This decoupling allows each function to operate independently — if the database is temporarily unavailable, incoming data accumulates safely in blob storage until the next writer trigger.

The container is organized into top-level folders by data type, with a consistent set of subfolders that track each blob's processing status. Two special folders — `tracking/` and `alerts/` — support deduplication and operational alerting.

## Folder Structure

```
nu-sensor-data/
├── environment/
│   ├── incoming/
│   ├── malformed/
│   ├── archived/
│   ├── duplicated/
│   ├── failed-processing/
│   └── failed-writing/
├── error/
│   ├── incoming/
│   ├── malformed/
│   ├── archived/
│   ├── duplicated/
│   ├── failed-processing/
│   └── failed-writing/
├── startup/
│   ├── incoming/
│   ├── malformed/
│   ├── archived/
│   ├── duplicated/
│   ├── failed-processing/
│   └── failed-writing/
├── unknown/
├── alerts/
└── tracking/
```

Note that empty folders are not visible in Azure Storage Explorer or the portal. If a folder is missing, it simply means no blobs currently reside there — the folder will reappear when new blobs are written to it.

## Data Type Classification

When the Webhook Receiver receives a payload, it examines the `data` field (the raw sensor message) and uses regular expressions to classify it into one of four data types:

**environment** — Standard sensor readings containing temperature, humidity, and noise data. This is the primary data type and makes up the vast majority of blobs.

**error** — Error messages reported by the sensor firmware, indicating hardware or connectivity problems.

**startup** — Boot/restart messages logged when a sensor powers on or resets.

**unknown** — Messages that don't match any recognized format. These are stored directly in the `unknown/` folder (no subfolders) with the naming pattern `unknown/{coreid}_{parsed_at_utc_iso}.json`, using the Particle `coreid` since the box_id cannot be extracted from an unrecognized format.

For environment, error, and startup data types, the Webhook Receiver determines whether the message is complete and well-formed or merely recognizable but malformed, and routes the blob to the appropriate subfolder.

## Webhook Receiver Subfolders

When the Webhook Receiver classifies a blob as environment, error, or startup, it writes it to one of two subfolders:

**incoming/** — The message follows the expected format and is complete. All values are present, parseable, and internally consistent. These blobs are ready for the Database Writer to process.

**malformed/** — The format is recognizable (the regex matched), but the data has problems that prevent correct parsing. Examples include an extra or missing value in the readings list (making it non-divisible by 3 for the T/RH/Noise triplets), or values with extra digits that fall outside valid ranges. Malformed blobs are stored for investigation but are not processed by the Database Writer.

Blob naming in both folders uses the pattern `{datatype}/{subfolder}/{box_id}_{parsed_at_utc_iso}.json`. The box_id is used (rather than coreid) because if the format is recognized, the box_id can be extracted from the message itself.

## Database Writer Subfolders

The Database Writer function triggers on a schedule and processes all blobs in the `incoming/` folders. After processing, each blob is moved (not copied) to one of four destination folders. The blob filename is preserved exactly as it was in `incoming/`.

**archived/** — The blob was successfully validated and its records were written to the database. This is the expected outcome for healthy sensor data. Archived blobs are retained for traceability. In the future, old archived blobs may be deleted to manage storage costs, keeping only the most recent months.

**duplicated/** — The blob is an exact duplicate of another blob within the same processing batch. This situation occurs only when the Webhook Receiver's deduplication is deactivated (for example, during maintenance). Under normal operation with deduplication active, duplicates are caught at the webhook level and never reach `incoming/`.

**failed-processing/** — The blob failed validation during processing. The most common cause is timestamp overlap between blobs in the same batch for the same sensor. Two blobs from the same sensor may each contain 15 minutes of data, but their timestamp windows partially overlap (for example, one starts at 11:10 and the other at 11:16). This signals a sensor clock malfunction. When an overlap is detected, both overlapping blobs are moved to `failed-processing/` since neither can be trusted.

**failed-writing/** — The blob passed validation but failed when the Database Writer attempted to INSERT records into the database, due to integrity constraint violations. Typically this means the new blob's timestamps overlap with records already written to the database from a previous batch. Unlike within-batch overlaps (where both blobs are discarded), here only the new blob is moved to `failed-writing/` since the earlier data is already in the database. Note that connectivity errors do not move blobs here — if the database is unreachable, blobs remain in `incoming/` and are retried on the next trigger.

It is worth noting that overlaps caught at write time are relatively rare. When a sensor has clock problems, the timestamp issues tend to be continuous rather than intermittent, so overlapping blobs are more likely to appear together in the same batch (caught during processing) than split across consecutive batches (caught during writing).

## Deduplication Tracking

The `tracking/` folder supports the Webhook Receiver's duplicate detection mechanism. When deduplication is active, each sensor has a tracking file at `tracking/{coreid}.json`. This file contains a list of strings representing the last N raw `data` field values received from that sensor.

When a new webhook arrives, the Webhook Receiver reads the tracking file for that sensor's coreid. If the raw data string already appears in the list, the blob is flagged as a duplicate and processing ends. If the data is new, it is appended to the tracking list and the blob proceeds through classification and storage.

This mechanism catches duplicates at the webhook level, which was implemented to address a specific issue where the Particle webhook was configured at both the account and product levels, causing every message to be delivered twice.

## Operational Alerts

The `alerts/` folder manages rate-limiting for operational alerts sent to the development team. When the Webhook Receiver encounters a condition worth reporting — malformed data, unknown format, high latency, or other anomalies — it checks the file at `alerts/{coreid}.json` before sending a notification.

The alert file is a JSON dictionary where keys are alert categories and values are ISO timestamp strings recording when that alert type was last sent for that sensor. The Webhook Receiver compares the current time against the stored timestamp and only sends a new alert if the configured delay has elapsed. This prevents alert fatigue from sensors that are persistently malfunctioning, since a broken sensor would otherwise trigger the same alert every 15 minutes.

## Blob Content Format

All blobs are stored as JSON files. The Webhook Receiver constructs a dictionary that includes the parsed data, the original raw string, metadata from the webhook payload, and processing information. For the specific JSON structure of environment blobs, see the [Data Flow](../../01-understanding-the-system/data-flow.md#webhook-receiver-to-blob-storage) documentation.

## Access and Connection

The storage account uses private networking by default. Azure Function Apps within the VNet connect automatically via private endpoints. For manual access from a local machine (for example, to inspect or move blobs), see [Desktop Blob Storage Tasks](../../04-making-changes/blob-storage-tasks.md) and [Networking Reference](networking.md).
