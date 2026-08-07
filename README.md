# RailPulse Analytics: Belgian Rail Performance & GTFS Network Dashboards

- **Repository:** `rail-powerbi-dashboarding`
- **Type of Challenge:** `Learning`
- **Team Challenge:** `Solo`
- **Author:** Hussein Abuammar
- **Institution:** BeCode
- **Year:** 2026
- **Deployment Strategy:** Power BI Service (Azure SQL Integration)

---

## 📌 Project Overview

This repository contains **two complementary Power BI dashboards** built on Belgian railway (SNCB/NMBS) data:

| Sub-Project | Folder | Data Source | Focus |
| :--- | :--- | :--- | :--- |
| **Live Performance Dashboard** | `real-time dataset/` | Azure SQL Database (iRail API feed) | Real-time punctuality, delays, cancellations, platform congestion |
| **GTFS Network Dashboard** | `Static database/` | Static GTFS feed (schedule data) | Network structure, trip scheduling, accessibility, station dwell time |

Both dashboards were developed entirely in **Power BI Service** (browser-based) on macOS, since Power BI Desktop and the On-premises Data Gateway are Windows-only tools.

---

## 🚆 Part 1 — Live Performance Dashboard (`real-time dataset/`)

**Workbook:** `RailPulse Dataset Report.pbix`
**Dashboard Link:** [RailPulse Performance Dashboard](https://app.powerbi.com/groups/0574653e-aeab-44e3-8b67-6e20c0a1ded5/reports/d09652e9-47c6-4ff5-93d9-992d6ad67b31/20507b7183086048142e?experience=power-bi)

### Problem Statement
Rail operators generate large volumes of live operational data every day. Raw database tables are difficult for managers and operational staff to interpret quickly. This dashboard transforms live railway operational data into an intuitive view of punctuality, delays, platform congestion, and train traffic throughout the day.

### Data Architecture
```
iRail API (Live Liveboard Data) → Azure SQL Database → Power BI Service → Interactive Dashboard
```
A local SQLite dataset was used during early development but was not used in the final dashboard — Azure SQL Database is the single source of truth.

> ⚠️ **Data Source Note:** This dashboard pulls live data from the **[iRail API](https://docs.irail.be/)**, a community-run public transport API for Belgium, rather than an official SNCB/NMBS data endpoint. Because iRail's liveboard data is queried per station, only a limited set of stations (Leuven, Brussel-Centraal, Antwerpen-Centraal, Gent-Sint-Pieters, Liège-Guillemins) were polled for this project, rather than the full national network.

### Data Model (Star Schema)
```
stations (1) ─┐
              ├─► liveboard_records (*)
vehicles (1) ─┘
```

**Fact table — `liveboard_records`:** `record_id`, `station_id`, `vehicle_id`, `scheduled_time`, `delay_seconds`, `platform`, `canceled`, `fetched_at`

**Dimension tables:**
- `stations`: `station_id`, `station_uri`, `station_name` — *Leuven, Brussel-Centraal, Antwerpen-Centraal, Gent-Sint-Pieters, Liège-Guillemins*
- `vehicles`: `vehicle_id`, `vehicle_name`, `Train Class` (calculated)

### Data Preparation (Power Query)
A custom column extracts the train class from the vehicle identifier:
```m
Text.Select(
    Text.AfterDelimiter([vehicle_name], ".", 1),
    {"A".."Z"}
)
```
Detected train classes: `IC`, `S`, `L`, `P`, `EC`, `ECD`, `EUR`

### DAX Measures
```dax
On-Time Rate % =
DIVIDE(
    CALCULATE(COUNTROWS(liveboard_records), liveboard_records[delay_seconds] < 120, liveboard_records[canceled] = FALSE),
    CALCULATE(COUNTROWS(liveboard_records), liveboard_records[canceled] = FALSE)
)

Cancellation Rate % =
DIVIDE(
    CALCULATE(COUNTROWS(liveboard_records), liveboard_records[canceled] = TRUE()),
    COUNTROWS(liveboard_records),
    0
)

Avg Delay (min) = AVERAGE(liveboard_records[delay_seconds]) / 60

Total Delayed Minutes = SUM(liveboard_records[delay_seconds]) / 60
```

**Methodology note:** A train is considered on time if its delay is under 120 seconds and it was not canceled. Canceled trains are excluded from both the numerator and denominator of On-Time Rate %, keeping the metric comparable to standard rail industry KPIs. Cancellations are tracked separately via Cancellation Rate %.

### Dashboard Layout
1. **Punctuality & Operational KPIs (Top Banner):** On-Time Rate % (96.4%), Cancellation Rate % (0.0%), Average Delay (0.38 min), Total Delayed Minutes (1.75K), Total Trains (4.603K).
2. **Train Class Performance (Bottom Left):** Total Delayed Minutes by Train Class — InterCity (IC) trains drive the majority of network delay minutes.
3. **Platform Congestion Map (Bottom Middle):** Volume by platform — Platforms 4, 3, and 5 handle disproportionately high volume.
4. **Rush Hour Matrix (Bottom Right):** Train Count vs. Avg Delay (min) by Hour — clear volume spikes during morning rush (6–8 AM).

### Top 3 Tactical Recommendations for SNCB/NMBS
1. **Spread Out Platform Usage** — Platforms 4, 3, and 5 take the heaviest load; move some trains to less busy tracks during rush hour.
2. **Add Extra Buffer Time for IC Trains** — A 2-minute schedule cushion prevents small delays from cascading across other lines.
3. **Focus Crew Support on Morning Rush Hours** — Extra dispatch/support staff between 6–8 AM to resolve delays instantly.

---

## 🗺️ Part 2 — GTFS Network Dashboard (`Static database/`)

**Workbook:** `RailPulse GTFS Report.pbix`
**Dashboard Link:** [RailPulse GTFS Network Dashboard](https://app.powerbi.com/groups/0574653e-aeab-44e3-8b67-6e20c0a1ded5/reports/c0b0a2a0-dda2-4b09-8c59-23ceb6fde609/ea29ab5eb886913b78d0?experience=power-bi)

### Problem Statement
Static GTFS feeds describe the full scheduled network — routes, trips, stops, and timetables — but raw GTFS tables are hard to explore directly. This dashboard turns the GTFS feed into an interactive view of network coverage, service frequency, accessibility, and station-level dwell time.

Because this dashboard is built on the **static GTFS schedule feed** rather than the iRail live API, it covers the **full national network** of stops and routes — unlike the real-time dashboard in Part 1, which is limited to the five stations polled live.

### Data Schema (GTFS Standard)
| Table | Entity Type | Description |
| :--- | :--- | :--- |
| `agency` | Dimension | Transit authority details, timezone, contact info |
| `calendar` | Dimension | Service availability by weekday |
| `calendar_dates` | Dimension | Calendar exceptions (additions/cancellations) |
| `routes` | Dimension | Line identifiers (`route_short_name`, `route_type`, agency link) |
| `stops` | Dimension | Station locations (`stop_lat`, `stop_lon`, `stop_name`, `wheelchair_boarding`) |
| `trips` | Fact/Dim | Scheduled journeys linking routes, service IDs, accessibility flags |
| `stop_times` | Fact | Arrival/departure times, stop sequence, dwell intervals |
| `transfers` | Fact | Stop-to-stop and trip-to-trip transfer connections |
| `feed_info`, `translations` | Standalone | Metadata / localization — not joined to the model |

### Data Model Relationships
- `agency[agency_id]` → `routes[agency_id]` (1:N)
- `routes[route_id]` → `trips[route_id]` (1:N — required de-duplicating `route_id` in Power Query first)
- `calendar[service_id]` → `trips[service_id]` (1:N)
- `calendar[service_id]` → `calendar_dates[service_id]` (1:N)
- `trips[trip_id]` → `stop_times[trip_id]` (1:N)
- `stops[stop_id]` → `stop_times[stop_id]` (1:N)
- `stops[stop_id]` → `transfers[from_stop_id]` (1:N, active) / `transfers[to_stop_id]` (1:N, inactive)
- `trips[trip_id]` → `transfers[from_trip_id]` / `transfers[to_trip_id]` (inactive)
- `feed_info` and `translations` remain standalone reference tables

### Data Transformations (Power Query / M)
**Departure hour extraction** — converts time values into discrete integer hours (0–23):
```m
Time.Hour([departure_time])
```
**Station dwell time (`dwell_seconds`)** — idling duration per scheduled stop:
```m
Duration.TotalSeconds([departure_time] - [arrival_time])
```

### DAX Measures
```dax
Total Trips = COUNTROWS('trips')

Active Stops = DISTINCTCOUNT('stops'[stop_id])

Wheelchair Accessible % =
DIVIDE(
    CALCULATE(COUNTROWS('trips'), 'trips'[wheelchair_accessible] = 1),
    COUNTROWS('trips'),
    0
)

Bikes Allowed % =
DIVIDE(
    CALCULATE(COUNTROWS('trips'), 'trips'[bikes_allowed] = 1),
    COUNTROWS('trips'),
    0
)

Avg Dwell Time Sec = AVERAGE(stop_times[dwell_seconds])
```

### Dashboard Layout
1. **Rail Network Geographic Coverage** (Azure Maps) — `stops[stop_lat]`/`stop_lon` (don't summarize), tooltip `stop_name`.
2. **Service Frequency by Departure Hour** (Clustered Column Chart) — X-axis `departure_hour` (filtered 0–23), Y-axis `[Total Trips]`.
3. **Filter by Route** (Slicer) — `routes[route_short_name]`.
4. **KPI Scorecard Ribbon** — Total Network Scheduled Trips, Bike Accessible Trips %.
5. **Trip Volume by Route Category** (Bar Chart) — `route_short_name` vs `[Total Trips]`.
6. **Highest Traffic Transit Hubs** (Top 10 Table) — `stop_name` vs `[Total Trips]`.
7. **Average Station Dwell Time** (Table) — `stop_name` vs `[Avg Dwell Time Sec]`, sorted descending.

---

## 🛠️ Challenges Encountered

- Power BI Desktop and the On-premises Data Gateway are unavailable on macOS; both dashboards were built entirely in Power BI Service via the browser.
- Creating DAX measures and calculated columns in Power BI Service required explicitly switching from **Viewing** mode to **Editing** mode.
- The live dashboard relies on the **iRail API** rather than an official SNCB data source, which limited the real-time dataset to a small set of manually polled stations rather than the full network.
- Duplicate `route_id` values in the GTFS `routes` table initially blocked a clean 1-to-many relationship to `trips` and required de-duplication in Power Query.
- GTFS time fields needed careful handling in Power Query (native `time` type vs. text parsing) to correctly compute `dwell_seconds`.
- The classic Map visual is being retired in favor of Azure Maps, requiring an upgrade and re-configuration of the location/tooltip fields.

## 🔭 Future Enhancements

- **Scale up station coverage** — extend the live iRail polling to cover all Belgian stations (or explore an official SNCB/NMBS data source) instead of the current limited set, to make the real-time dashboard fully representative of the national network.
- Scheduled cloud data refresh for the live dataset.
- Drill-through navigation pages and custom tooltips.
- Cross-hub comparison pages using interactive slicers.
- Transfer/connection bottleneck analysis using `min_transfer_time`.
- Historical trend comparisons and predictive delay forecasting using machine learning.

---

## 🧰 Technologies Used

Microsoft Power BI Service · Microsoft Azure SQL Database · Azure Portal · Azure Maps · Power Query (M) · DAX · SQL · GTFS Standard · iRail API · GitHub

---

## 📁 Repository Structure

```
rail-powerbi-dashboarding/
│── README.md
│── real-time dataset/
│   │── RailPulse Dataset Report.pbix
│   └── screenshots/
│── Static database/
│   │── RailPulse GTFS Report.pbix
│   └── screenshots/
```

---

## 📜 License & Acknowledgments

- **Data Sources:** Live liveboard data via the [iRail API](https://docs.irail.be/) and the General Transit Feed Specification (GTFS) public data release for the Belgian Railway Network (NMBS/SNCB).
- **Platform:** Built with Microsoft Power BI Service, Azure SQL Database, and Azure Maps visual integration.