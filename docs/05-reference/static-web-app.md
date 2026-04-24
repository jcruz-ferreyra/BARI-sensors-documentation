# Static Web App Reference

This page documents the public dashboard hosted as an Azure Static Web App. The dashboard is accessed from a browser and displays sensor data using API responses from the Azure API Function App.

For API endpoint details, see [API Endpoints Reference](../api-endpoints.md). For the API implementation, see [Function Code Reference](../function-code-reference.md).

## Overview

The Static Web App hosts the public-facing dashboard used to visualize environmental sensor data. It is a browser-based frontend built with static HTML, CSS, and vanilla JavaScript.

The dashboard does not connect directly to Blob Storage or SQL Database. Instead, it calls the API Function App, which queries the database and returns JSON responses.

The dashboard is responsible for:

* Reading the selected location from the URL
* Loading location metadata
* Loading sensor readings from the API
* Rendering time-series charts
* Allowing date range and aggregation selection
* Allowing comparison with another sensor location
* Displaying loading, empty, and error states

---

## Public URL Structure

The dashboard is publicly accessible through the Azure Static Web App base URL.

```text
{{ urls.dashboard_base }}
```

Dashboard pages are accessed using a path and a query parameter.

Example:

```text
{{ urls.dashboard_base }}/noise?deployment_id=5
```

In this example, the dashboard loads noise data for the sensor location identified by `5`.

!!! note "Legacy URL Parameter"
The URL parameter is named `deployment_id` for backward compatibility with previously embedded dashboard links. Internally, this value represents `location_id`, not `deployment_id`.

```
Changing the parameter name would break existing dashboard links embedded elsewhere, including the Common SENSES ArcGIS map.
```

---

## Dashboard Pages

The dashboard has separate pages for heat-related data and noise data.

| Page     | Purpose                                     | Metric Behavior                                               |
| -------- | ------------------------------------------- | ------------------------------------------------------------- |
| `/heat`  | Displays heat-related environmental metrics | User can switch between heat index, temperature, and humidity |
| `/noise` | Displays noise data                         | Metric is fixed to noise                                      |

The two pages share the same general layout and JavaScript behavior. The main difference is that the heat page includes metric tabs, while the noise page does not need metric switching.

---

## Frontend Technology

The dashboard is intentionally built without a frontend framework to keep maintenance simple.

| Technology         | Purpose                          |
| ------------------ | -------------------------------- |
| HTML               | Page structure                   |
| CSS                | Styling and layout               |
| Vanilla JavaScript | Dashboard behavior and API calls |
| Plotly.js          | Interactive time-series charts   |
| Flatpickr          | Date range picker                |

External libraries are loaded from CDNs in the HTML page.

---

## Main Files

The dashboard frontend is organized into static files.

| File              | Purpose                                      |
| ----------------- | -------------------------------------------- |
| `heat.html`       | Heat dashboard page                          |
| `noise.html`      | Noise dashboard page                         |
| `css/styles.css`  | Dashboard styling                            |
| `js/config.js`    | Shared configuration, including API base URL |
| `js/api.js`       | API request helper functions                 |
| `js/charts.js`    | Plotly chart rendering and UI states         |
| `js/dashboard.js` | Main dashboard logic and event handling      |

The HTML page loads configuration, API helpers, chart helpers, and the main dashboard script in that order.

---

## Page Layout

The main dashboard page includes:

* Header with location label and address
* Date range selector
* Aggregation buttons
* Metric tabs on the heat page
* Main chart container
* Optional comparison message
* Comparison sensor dropdown
* Last updated timestamp

The location address is loaded dynamically from the API using the location ID from the URL.

---

## URL Parameter Handling

The dashboard reads the selected location from the browser URL.

```text
?deployment_id=5
```

Although the parameter is called `deployment_id`, the frontend treats the value as a `location_id`.

If the parameter is missing, the dashboard shows an error message and does not attempt to load chart data.

---

## Date Range Behavior

The dashboard uses Flatpickr for date range selection.

Default behavior:

* If no date range is selected, the dashboard loads the last 7 days.
* Users select a start date and end date.
* Once both dates are selected, the dashboard reloads automatically.
* The date picker prevents selecting future dates.
* During manual selection, the selectable range is limited to 30 days from the first selected date.

Dates are sent to the API in `YYYY-MM-DD` format.

---

## Aggregation Behavior

Users can choose the aggregation level using dashboard buttons.

| Button     | API Aggregation Value | Meaning               |
| ---------- | --------------------- | --------------------- |
| `1min avg` | `1min`                | Minute-level readings |
| `1h avg`   | `1hour`               | Hourly averages       |
| `1d avg`   | `1day`                | Daily averages        |

Changing the aggregation level reloads the dashboard data.

---

## Metric Behavior

The heat dashboard supports metric switching through tabs.

Supported heat page metrics:

* `heat_index`
* `temperature`
* `humidity`

The selected metric is stored in `window.METRIC` and passed to the readings API.

The noise dashboard is fixed to the `noise` metric and does not need metric tabs.

---

## API Calls

The dashboard calls three API endpoints during normal use.

| API Endpoint             | Used For                                                                                  |
| ------------------------ | ----------------------------------------------------------------------------------------- |
| `/api/location-metadata` | Loads the address and metadata for the selected location                                  |
| `/api/readings`          | Loads time-series readings for the selected location, metric, date range, and aggregation |
| `/api/sensors-list`      | Loads available sensor locations for comparison dropdown                                  |

The health endpoint is not normally used by the dashboard UI, but can be used to check whether the API Function App is reachable.

---

## Readings API Response

The readings API returns data already formatted for frontend use.

Example response:

```json
{
  "location_id": 5,
  "metric": "temperature",
  "readings": [
    {
      "timestamp": "2025-06-01T13:00:00-04:00",
      "temperature": 76.4
    },
    {
      "timestamp": "2025-06-01T14:00:00-04:00",
      "temperature": 77.1
    }
  ]
}
```

Timestamps are converted by the API from UTC to Eastern Time before being returned to the frontend.

The metric value key matches the requested metric. For example:

* `temperature` requests return objects with a `temperature` field
* `humidity` requests return objects with a `humidity` field
* `heat_index` requests return objects with a `heat_index` field
* `noise` requests return objects with a `noise` field

---

## Comparison Feature

The dashboard supports comparing the selected location with another sensor location.

On page load, the dashboard calls `/api/sensors-list` and populates the comparison dropdown. The current location is excluded from the dropdown.

When a comparison sensor is selected:

1. The dashboard keeps the selected comparison `location_id` in frontend state.
2. The dashboard reloads the main chart data.
3. A second `/api/readings` request is made for the comparison location using the same metric, date range, and aggregation.
4. The chart renders both the primary location and comparison location if data is available.

If comparison data is unavailable, the dashboard displays a message instead of failing the entire chart.

---

## Chart Rendering

Charts are rendered with Plotly.js.

The chart module handles:

* Loading state
* Empty data state
* Error state
* Time-series rendering
* Optional comparison series
* Comparison messages

The main dashboard script is responsible for requesting data and passing the resulting readings to the chart module.

---

## Last Updated Display

After chart data loads, the dashboard updates the footer with the local browser time:

```text
Last updated: <local date and time>
```

This timestamp indicates when the browser last refreshed the dashboard data. It does not represent the latest sensor reading timestamp.

---

## Deployment Role

The Static Web App only serves frontend files. It does not process sensor data and does not store data.

Data flow for dashboard display:

```text
Browser → Static Web App → API Function App → SQL Database
```

The browser downloads the static files from the Static Web App, then the JavaScript code calls the API Function App for data.

---

## Security and Public Access

The dashboard is public by design. No authentication is required to view the dashboard.

Because the dashboard is public, it should only expose data intended for public viewing. Sensitive credentials, database connection strings, storage account keys, or private API keys must never be included in frontend files.

Configuration files such as `config.js` may include public API URLs, but must not include secrets.

---

## Troubleshooting

| Symptom                                      | Likely Cause                                         | Where to Check                            |
| -------------------------------------------- | ---------------------------------------------------- | ----------------------------------------- |
| Page loads but chart shows no data           | No readings for selected location/date/metric        | API response, SQL data availability       |
| Page shows missing `deployment_id` error     | URL does not include required query parameter        | Browser URL                               |
| Location address unavailable                 | Metadata API failed or location ID not found         | `/api/location-metadata`                  |
| Comparison dropdown does not load            | Sensors list API failed                              | `/api/sensors-list`                       |
| Chart fails to render                        | API error, invalid response, or frontend chart error | Browser console, API logs                 |
| Static page does not load                    | Static Web App hosting issue                         | Azure Static Web App resource             |
| Data visible in SQL but missing on dashboard | API query or frontend parameter issue                | API Function logs and browser network tab |

---

## Related Documentation

* [API Endpoints Reference](../api-endpoints.md)
* [Function Code Reference](../function-code-reference.md)
* [Complete Schema Reference](../complete-schema.md)
* [Data Flow](../../01-understanding-the-system/data-flow.md)
* [Networking Reference](networking.md)
