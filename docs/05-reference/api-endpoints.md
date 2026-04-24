# API Endpoints Reference

This page documents the HTTP endpoints exposed by the API Function App and used by the public dashboard.

For the API Function App implementation, see [Function Code Reference](function-code-reference.md). For dashboard code, see [Static Web App Reference](azure-infrastructure/static-web-app.md).

## Base URL

```text
{{ urls.api_base }}
```

## Endpoint Overview

| Endpoint                 | Method | Purpose                                                  |
| ------------------------ | ------ | -------------------------------------------------------- |
| `/api/health`            | GET    | Checks whether the API Function App is reachable         |
| `/api/sensors-list`      | GET    | Returns available sensor locations                       |
| `/api/location-metadata` | GET    | Returns metadata for one location                        |
| `/api/readings`          | GET    | Returns time-series readings for one location and metric |

---

## `/api/health`

Checks whether the API Function App is running.

### Request

```http
GET /api/health
```

### Response

```json
{
  "status": "healthy",
  "timestamp": "2026-02-03T14:25:10.123456"
}
```

---

## `/api/sensors-list`

Returns the list of available sensor locations used by the dashboard selector.

### Request

```http
GET /api/sensors-list
```

### Response

```json
[
  {
    "location_id": 1,
    "location_address": "Example location",
    "latitude": 42.309,
    "longitude": -71.091
  }
]
```

---

## `/api/location-metadata`

Returns metadata for a specific sensor location.

### Query Parameters

| Parameter     | Required | Description                                       |
| ------------- | -------- | ------------------------------------------------- |
| `location_id` | Yes      | Location identifier from `nu_sensors.location_id` |

### Request

```http
GET /api/location-metadata?location_id=1
```

### Response

```json
{
  "location_id": 1,
  "location_address": "Example location",
  "latitude": 42.309,
  "longitude": -71.091,
  "deployments": [
    {
      "deployment_id": 1,
      "box_id": 1,
      "install_datetime": "2025-06-20T13:17:00",
      "uninstall_datetime": null,
      "is_active": true
    }
  ]
}
```

---

## `/api/readings`

Returns sensor readings for a selected location, metric, time range, and aggregation level.

### Query Parameters

| Parameter     | Required | Description                                                |
| ------------- | -------- | ---------------------------------------------------------- |
| `location_id` | Yes      | Location identifier from `nu_sensors.location_id`          |
| `metric`      | Yes      | One of `temperature`, `humidity`, `heat_index`, or `noise` |
| `start_date`  | Yes      | Start timestamp or date                                    |
| `end_date`    | Yes      | End timestamp or date                                      |
| `aggregation` | Yes      | Aggregation level, such as `1min`, `1hour`, or `1day`      |

### Request

```http
GET /api/readings?location_id=1&metric=temperature&start_date=2025-06-01&end_date=2025-06-02&aggregation=1hour
```

### Response

```json
{
  "location_id": 1,
  "metric": "temperature",
  "aggregation": "1hour",
  "readings": [
    {
      "timestamp": "2025-06-01T13:00:00-04:00",
      "value": 76.4
    },
    {
      "timestamp": "2025-06-01T14:00:00-04:00",
      "value": 77.1
    }
  ]
}
```

---

## Error Responses

Invalid requests return JSON error messages.

```json
{
  "error": "Invalid metric"
}
```

Common status codes:

| Status Code | Meaning                                      |
| ----------- | -------------------------------------------- |
| `200`       | Request succeeded                            |
| `400`       | Invalid or missing query parameter           |
| `404`       | Requested location was not found             |
| `500`       | API failed while querying or formatting data |
| `503`       | API could not connect to the database        |

---

## Caching

Some endpoints include browser cache headers to reduce repeated database queries.

| Endpoint                 | Cache Behavior                                |
| ------------------------ | --------------------------------------------- |
| `/api/readings`          | Short cache, currently 5 minutes              |
| `/api/sensors-list`      | Longer cache, currently 1 hour                |
| `/api/location-metadata` | No explicit cache or implementation-dependent |

---

## Notes

All data is stored in UTC in the database. API responses may convert timestamps for dashboard display depending on endpoint formatting logic.

This page documents the API contract used by the dashboard. Implementation details belong in [Function Code Reference](function-code-reference.md).
