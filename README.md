# RailPulse Analytics: Belgian Rail Performance & GTFS Network Dashboards

- **Repository:** `rail-powerbi-dashboarding`
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

![Live Performance Dashboard](real-time%20dataset/screenshots/1.png)

### Connection Setup Used
**Scenario A — Azure Cloud DB:** `Get Data → Azure → Azure SQL Database`, connecting directly to the Azure SQL Server holding the `liveboard_records`, `stations`, and `vehicles` tables. A local SQLite dataset (Scenario B) was used only during early Sprint 1 development and is not part of the final dashboard.

Since the iRail API was used instead of an official SNCB endpoint, [Azure Portal](https://portal.azure.com/#home) was connected to Power BI Service to pull in the Azure SQL data collected from the 5 polled stations.

> ⚠️ **Data Source Note:** Live data is sourced from the **[iRail API](https://docs.irail.be/)**, a community-run public transport API for Belgium, rather than an official SNCB/NMBS endpoint. Because iRail's liveboard data is queried per station, only a limited set of stations (Leuven, Brussel-Centraal, Antwerpen-Centraal, Gent-Sint-Pieters, Liège-Guillemins) were polled for this project, rather than the full national network.

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

### DAX Calculated Table (Date Extraction)
```dax
DateTable =
CALENDAR(
    MIN(liveboard_records[Scheduled Time]),
    MAX(liveboard_records[Scheduled Time])
)
```

**Note:** `DateTable` is a standalone calculated table (not a column on `liveboard_records`) that generates one row per date spanning the earliest to latest `Scheduled Time` in the data, giving a clean `Date` column to drive date-based slicers and visuals.

**Train Class — DAX equivalent (reference only):** The dashboard's actual Train Class column is built in Power Query (`Text.Select` step above). The same logic expressed as a DAX calculated column would look like this:
```dax
Train Class (DAX) =
VAR AfterDot =
    MID(
        vehicles[vehicle_name],
        FIND(".", vehicles[vehicle_name]) + 1,
        LEN(vehicles[vehicle_name])
    )
RETURN
    CONCATENATEX(
        GENERATESERIES(1, LEN(AfterDot)),
        VAR ch = MID(AfterDot, [Value], 1)
        RETURN IF(ch >= "A" && ch <= "Z", ch, ""),
        ""
    )
```
This is not the version actually used in the model — it's included here to show the equivalent logic in DAX for reference.

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

**Methodology note:** A train is considered on time if its delay is under 120 seconds (2 minutes) and it was not canceled. Canceled trains are excluded from both the numerator and denominator of On-Time Rate %, keeping the metric comparable to standard rail industry KPIs. Cancellations are tracked separately via Cancellation Rate %.

### ✅ Answering the Core Operational Questions

The mission brief asks the dashboard to answer four specific operational dilemmas. Here is how each one is addressed, and what the data shows:

**1. Punctuality Scorecard — what is the network's overall On-Time Rate %?**
> Answered by the `On-Time Rate %` KPI card (top banner). Current reading: **96.4%** on-time (delay < 2 minutes), against a **0.0%** Cancellation Rate, an **Average Delay of 0.38 min**, and **1.75K Total Delayed Minutes** across **4.603K Total Trains** recorded.

**2. The Rush Hour Matrix — where exactly do bottlenecks occur across the day?**
> Answered by the Rush Hour Matrix (bottom right): Train Count vs. Avg Delay (min) by Hour. The data shows a **clear volume spike between 6–8 AM**, where both train count and average delay rise together — the morning commute is the network's primary bottleneck window.

**3. Train Class Breakdown — which train category drives the most delayed minutes?**
> Answered by the Train Class Performance chart (bottom left): Total Delayed Minutes by Train Class. **InterCity (IC) trains account for the majority of network delay minutes**, well ahead of S, L, P, EC, ECD, and EUR classes.

**4. Platform Congestion Map — which tracks run the most behind schedule?**
> Answered by the Avg Delay (min) by Station Name map (top right): bubble size reflects each station's average delay. **Brussels shows the largest bubble and highest average delay**, followed by Liège, with Antwerp mid-sized and Ghent showing the smallest bubble — the lowest average delay of the five polled stations.

![Live Performance Dashboard — Avg Delay by Station Map](real-time%20dataset/screenshots/3.png)

### 💡 Top 3 Recommendations for SNCB/NMBS

Based directly on the patterns visible in the dashboard screenshots above:

1. **Prioritize Brussels for Delay Reduction.** The Avg Delay by Station map shows Brussels carrying the highest average delay of the five polled stations, with Liège close behind. Directing schedule review and dispatch resources to these two stations first would target the network's actual worst-performing hubs rather than spreading effort evenly.
2. **Add Extra Buffer Time for IC Trains.** The Train Class Performance chart shows InterCity (IC) services are the single largest contributor to total delayed minutes. Building a small (~2-minute) schedule cushion into IC timetables would absorb minor delays before they cascade into other lines that share track segments.
3. **Focus Crew and Dispatch Support on the 6–8 AM Window.** The Rush Hour Matrix shows train volume and average delay both peaking between 6–8 AM. Concentrating extra dispatch and platform staff during this specific window — rather than spreading resources evenly across the day — would target the network's actual bottleneck period directly.

---

## 🗺️ Part 2 — GTFS Network Dashboard (`Static database/`)

**Workbook:** `RailPulse GTFS Report.pbix`
**Dashboard Link:** [RailPulse GTFS Network Dashboard](https://app.powerbi.com/groups/0574653e-aeab-44e3-8b67-6e20c0a1ded5/reports/c0b0a2a0-dda2-4b09-8c59-23ceb6fde609/ea29ab5eb886913b78d0?experience=power-bi)

![GTFS Network Dashboard](Static%20database/screenshots/1.png)

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

![GTFS Network Dashboard — Route & Station Detail](Static%20database/screenshots/2.png)

### 💡 Top 3 Recommendations for SNCB/NMBS

Based on the national network patterns visible in the GTFS dashboard:

1. **Focus on the 9–15h window, not just rush hour.** Trips stay high (~18–20K) for most of the midday, so that's where extra buffering matters most — not just the morning peak.
2. **Check the Aéroport Cdg TGV dwell time.** At 720 seconds vs. a 54s network average, it's a big outlier — worth confirming it's real and not a data error.
3. **Prioritize IC routes for improvements.** They carry way more trips (~60K) than any other category, so upgrades there help the most riders.

---

## 🛠️ Challenges Encountered

- Power BI Desktop and the On-premises Data Gateway are unavailable on macOS; both dashboards were built entirely in Power BI Service via the browser.
- Creating DAX measures and calculated columns in Power BI Service required explicitly switching from **Viewing** mode to **Editing** mode.
- The live dashboard relies on the **iRail API** rather than an official SNCB data source, which limited the real-time dataset to a small set of manually polled stations rather than the full network.
- Duplicate `route_id` values in the GTFS `routes` table initially blocked a clean 1-to-many relationship to `trips` and required de-duplication in Power Query.
- GTFS time fields needed careful handling in Power Query (native `time` type vs. text parsing) to correctly compute `dwell_seconds`.
- The classic Map visual is being retired in favor of Azure Maps, requiring an upgrade and re-configuration of the location/tooltip fields.

## 🛣️ Roadmap

| Phase | Item | Status |
| :--- | :--- | :--- |
| **Now** | Scheduled cloud data refresh for the live dataset | ⬜ Planned |
| **Now** | Drill-through navigation pages and custom tooltips | ⬜ Planned |
| **Next** | Scale up station coverage — extend live iRail polling to all Belgian stations (or explore an official SNCB/NMBS source) so the real-time dashboard reflects the full national network | ⬜ Planned |
| **Next** | Cross-hub comparison pages using interactive slicers | ⬜ Planned |
| **Later** | Transfer/connection bottleneck analysis using `min_transfer_time` | ⬜ Planned |
| **Later** | Historical trend comparisons and predictive delay forecasting using machine learning | ⬜ Planned |

*"Now" = short-term, achievable with current data setup · "Next" = requires broader data coverage · "Later" = larger scope, exploratory.*

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

---

## ⏱️ Project Timeline

**5 days**

## 👤 Author

**Hussein Abuammar**
🔗 [LinkedIn](https://www.linkedin.com/in/hussein-abuammar/)