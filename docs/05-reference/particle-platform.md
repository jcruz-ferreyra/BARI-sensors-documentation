# Particle Platform Reference

This page documents how the Particle.io platform is used in the sensor data pipeline. It explains Particle's role between the field sensors and Azure, the message formats produced by the sensors, and the platform areas used for monitoring and troubleshooting.

For Azure-side processing, see [Data Flow](../01-understanding-the-system/data-flow.md). For adding or changing sensor configuration, see [Sensor Configuration](../04-making-changes/sensor-configuration.md).

## Overview

Particle.io provides the cloud layer between the physical sensor boxes and the Azure data pipeline.

In this system, sensors publish messages to Particle Cloud through a cellular connection. Particle Cloud then forwards those messages to Azure using webhook integrations. The Azure Webhook Receiver Function accepts the webhook payload, parses the sensor message, and stores the result in Blob Storage for downstream processing.

Particle is responsible for:

* Receiving data from deployed sensor boxes
* Maintaining device connectivity information
* Triggering webhooks to Azure
* Providing event logs for debugging sensor communication
* Supporting device-level monitoring through the Particle Console

Particle is not the long-term storage system. Once data reaches Azure, Blob Storage and SQL Database become the authoritative storage locations.

---

## Sensor Hardware

The field devices are Arduino-based environmental sensor boxes. Each box collects:

* Temperature
* Relative humidity
* Noise readings

Readings are collected every minute, but uploaded to Particle every 15 minutes. This reduces power use because the LTE module does not need to remain continuously active.

Each message contains 15 consecutive one-minute readings from a single sensor box.

The sensors transmit data over a cellular connection. Some sensor boxes may also keep local SD card backups, but SD card recovery is logistics-intensive and rarely used during normal operations.

For firmware behavior, hardware errors, or sensor-side code questions, contact Amy's team.

---

## Account Structure

The sensors are distributed across multiple Particle accounts due to Particle free-tier limits. There may be 6-7 separate accounts involved in the full deployment.

Credentials and account ownership are managed by Amy's team. Do not create new Particle integrations or modify existing ones without confirming the correct account and device group.

For procedures related to adding sensors or changing configuration, see [Sensor Configuration](../04-making-changes/sensor-configuration.md).

---

## Webhook Integrations

Particle webhooks forward sensor events from Particle Cloud to Azure HTTP endpoints.

In this project, webhooks deliver sensor messages to the Azure Webhook Receiver Function. The webhook wraps the raw sensor message in a JSON payload that includes metadata such as the Particle event name, device core ID, and publish timestamp.

A webhook can be configured at different scopes:

| Webhook Scope                     | Meaning                                     | Risk                                        |
| --------------------------------- | ------------------------------------------- | ------------------------------------------- |
| Account-level integration         | Applies broadly across the Particle account | Can affect many devices at once             |
| Product-level integration         | Applies to devices in a Particle product    | Useful for grouped deployments              |
| Device/event-specific integration | Applies more narrowly                       | Lower blast radius, but more setup overhead |

Be careful not to configure duplicate webhook integrations for the same event stream. If the same message is forwarded by multiple integrations, Azure may receive duplicate webhook deliveries.

The Azure Webhook Receiver includes duplicate detection, but duplicate Particle integrations should still be fixed at the source when discovered.

---

## Platform Navigation

Particle Console is used mainly for monitoring and troubleshooting.

Key areas:

| Area         | Purpose                                                                    |
| ------------ | -------------------------------------------------------------------------- |
| Devices      | Check whether a device is online, its last connection, and device metadata |
| Integrations | Review webhook configuration and delivery status                           |
| Events       | Inspect raw published messages from sensors                                |
| Console logs | Debug device communication and publishing behavior                         |

Useful checks include:

* Whether the sensor is publishing events
* Whether the device has connected recently
* Whether webhook integrations are enabled
* Whether the same event appears more than once
* Whether error or startup messages are appearing unexpectedly

Particle-side monitoring helps distinguish between sensor communication problems and Azure processing problems.

---

## Message Types and Data Format

Sensors publish several message types through Particle. The Azure Webhook Receiver parses these messages and classifies them before writing to Blob Storage.

The main message types are:

1. Environmental data messages
2. Error messages
3. Startup messages
4. System information available through Particle metadata

---

## Environmental Data Messages

Environmental data messages contain temperature, relative humidity, and noise readings.

Sensors collect readings every minute, but publish one message every 15 minutes. Each message therefore contains 15 sets of readings, with each set containing:

```text
T, RH, Noise
```

### Generic Format

```text
,BoxID,month,day,hour,minute,T,RH,Noise,T,RH,Noise,...
```

The timestamp fields represent the UTC timestamp of the first reading in the batch. Each subsequent reading is assigned by adding one minute to the previous timestamp.

### Example

```text
,19,6,6,17,34,2483,5480,6389,2482,5483,5181,2477,5481,5222,2484,5486,6121,2485,5487,5649,2483,5481,6591,2483,5482,6923,2488,5484,6904,2484,5482,6860,2480,5482,6731,2487,5488,6829,2487,5491,6762,2485,5486,6515,2479,5488,6711,2488,5489,6806
```

In this example:

* Box ID is `19`
* First timestamp is `2025-06-06 17:34 UTC`
* First raw temperature value is `2483`
* First raw relative humidity value is `5480`
* First raw noise value is `6389`

The second reading is assigned timestamp `2025-06-06 17:35 UTC`, the third reading is assigned `2025-06-06 17:36 UTC`, and so on.

### Postprocessing

Raw values are transmitted as integers to reduce data size and power consumption.

| Field             | Raw Conversion                              | Example                                  |
| ----------------- | ------------------------------------------- | ---------------------------------------- |
| Temperature       | `T_raw / 100` gives degrees Celsius         | `2483 / 100 = 24.83 °C`                  |
| Relative humidity | `RH_raw / 100` gives percent RH             | `5480 / 100 = 54.80%`                    |
| Noise             | `(Noise_raw / 32768) * 6.144 * 50` gives dB | `(6389 / 32768) * 6.144 * 50 = 59.90 dB` |

Temperature is later converted to Fahrenheit before being stored in the database. Heat index is calculated downstream using temperature and relative humidity.

Precision can be reduced for display or reporting. The sensor temperature accuracy is approximately 0.1 °C, and relative humidity accuracy is approximately 1%.

For Azure-side transformation and database insertion, see [Data Flow](../01-understanding-the-system/data-flow.md).

---

## Error Messages

Error messages are reported when the sensor firmware detects hardware or communication problems. These messages can indicate that physical maintenance may be needed.

### Generic Format

```text
BoxIDplusDate Hour:Minute ErrorCode
```

Where `BoxIDplusDate` is:

```text
2-digit BoxID + 2-digit year + 2-digit month + 2-digit day
```

### Example

```text
19250610 14:42 E10
```

In this example:

* Box ID is `19`
* Timestamp is `2025-06-10 14:42 UTC`
* Error code is `E10`

### Error Codes

| Error Code | Description                                                            |
| ---------- | ---------------------------------------------------------------------- |
| `E1`       | Serial Port not open                                                   |
| `E2`       | Temperature and relative humidity sensor not found                     |
| `E3`       | RTC not found                                                          |
| `E4`       | LTE module connection to cloud timeout; only logged to SD card         |
| `E5`       | LTE module receiving data from Arduino timeout; only logged to SD card |
| `E6`       | SD module failure                                                      |
| `E7`       | Buffer overflow                                                        |
| `E8`       | SD open file failure                                                   |
| `E9`       | SD error logging failure                                               |
| `E10`      | ADC not found                                                          |

Error messages are stored in Blob Storage and later written to the `nu_errors` table by the Database Writer.

---

## Startup Messages

Startup messages are sent when the sensor system boots or restarts. They confirm that the cellular module connected to the internet and that the Arduino microcontroller communicated successfully with Particle Cloud.

These messages are useful in two situations:

1. During installation, they provide a quick confirmation that the sensor box is functioning.
2. During normal operation, unexpected startup messages may indicate power loss, battery issues, or microcontroller resets.

### Generic Format

```text
BoxIDplusDate Hour:Minute LTE Setup Done
```

Where `BoxIDplusDate` is:

```text
2-digit BoxID + 2-digit year + 2-digit month + 2-digit day
```

The text `LTE Setup Done` is static.

### Example

```text
19250610 16:47 LTE Setup Done
```

In this example:

* Box ID is `19`
* Timestamp is `2025-06-10 16:47 UTC`
* The sensor is reporting startup and successful LTE setup

Startup messages are stored in Blob Storage and later written to the `nu_startup` table by the Database Writer.

---

## System Information

Particle also provides device-level system information that can help with diagnostics. This includes information such as:

* Last connection time
* Online/offline status
* Signal strength
* Device metadata
* Event publishing history

This information is not currently part of the main SQL data model. It may be useful for future sensor health monitoring, such as pulling device status once per day and comparing Particle-side connectivity with Azure-side data arrival.

---

## Monitoring and Troubleshooting

Particle Console is useful for identifying whether a problem starts before or after Azure receives the data.

### Check if a Sensor Is Publishing

Use the Events view in Particle Console to confirm whether the sensor is publishing messages.

If messages appear in Particle but not in Azure, investigate webhook delivery, Azure Function availability, or Blob Storage writes.

If messages do not appear in Particle, the issue is likely sensor-side, connectivity-related, power-related, or account/device configuration-related.

### Check Webhook Delivery

Use the Integrations section to confirm that the webhook is enabled and pointing to the expected Azure Webhook Receiver endpoint.

Possible webhook issues include:

* Disabled integration
* Incorrect Azure endpoint URL
* Duplicate integrations sending the same event more than once
* Webhook authentication or function key problems
* Azure Function temporarily unavailable

### Check for Error Messages

Error messages indicate that the sensor firmware detected a problem. These should be reviewed when a sensor stops reporting normal environmental data or shows unusual behavior.

Common examples:

* `E2`: Temperature and humidity sensor not found
* `E6`: SD module failure
* `E10`: ADC not found

### Check for Startup Messages

Startup messages are expected during installation or planned restarts. Frequent unexpected startup messages may indicate unstable power, battery problems, or repeated microcontroller resets.

### Distinguishing Particle vs Azure Issues

| Observation                                          | Likely Source                                                         |
| ---------------------------------------------------- | --------------------------------------------------------------------- |
| No messages in Particle Events                       | Sensor, power, cellular connection, or Particle account/device issue  |
| Messages visible in Particle but not in Blob Storage | Webhook or Azure Webhook Receiver issue                               |
| Blobs visible in `incoming/` but not in SQL          | Database Writer or SQL Database issue                                 |
| Data visible in SQL but not on dashboard             | API Function or dashboard issue                                       |
| Duplicate messages in Particle or Azure              | Duplicate Particle webhook integrations or repeated sensor publishing |

---

## Related Documentation

* [Data Flow](../01-understanding-the-system/data-flow.md)
* [Blob Storage Reference](azure-infrastructure/blob-storage.md)
* [Complete Schema Reference](complete-schema.md)
* [Function Code Reference](function-code-reference.md)
* [Sensor Configuration](../04-making-changes/sensor-configuration.md)
* [Troubleshooting: Sensor Not Reporting](../03-operating-the-system/troubleshooting/sensor-not-reporting.md)
