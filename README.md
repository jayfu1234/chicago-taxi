# Chicago Taxi Trips Analytics 

## Overview

Build a production style data pipeline on Google Cloud Platform to analyze the public Chicago Taxi Trips dataset. We answer three core business questions about top tip earning taxis, taxis that may be overworked, and the impact of US public holidays on demand, along with two bonus insights.

## Architecture

| Data Warehouse | Google BigQuery |
| :---- | :---- |
| Transformation | GCP Dataform |
| Visualization | Looker Studio |

GCP Project: time-dotcom-taxi-500404   
Dataform Repository: chicago-taxi-pipeline   
Output Dataset: chicago\_taxi\_analytics  
Source table: bigquery-public-data.chicago\_taxi\_trips.taxi\_trips
Dashboard: https://datastudio.google.com/reporting/ae6af534-8f74-4a62-90a3-89281217519f

## Pipeline Structure

definitions/  
  sources/  
    taxi\_trips.sqlx              \-- declaration pointing to the public table  
  staging/  
    stg\_taxi\_trips.sqlx           \-- cleaned, filtered trip records  
  marts/  
    top\_tip\_earners.sqlx          \-- Question 1  
    overworked\_taxis.sqlx         \-- Question 2  
    holiday\_impact.sqlx           \-- Question 3  
    hourly\_demand\_pattern.sqlx    \-- Bonus Insight 1  
    company\_performance.sqlx      \-- Bonus Insight 2
    geo_demand_by_area.sqlx       -- Bonus Insight 3

## Staging Layer: Cleaning Rules

stg\_taxi\_trips is built from the raw table with the following filters applied:

- fare \> 0 (removes free or invalid fare entries)  
- trip\_miles \> 0 (removes zero distance trips)  
- trip\_seconds \> 0 (removes zero duration trips)  
- taxi\_id IS NOT NULL (a trip cannot be attributed without a taxi ID)

The current column list covers identifiers, timestamps, duration, distance, all earnings components (fare, tips, tolls, extras, trip\_total), payment\_type, company, and pickup/dropoff coordinates.

## Key Assumptions and Definitions

**1\. "Last 3 months" window.** The dataset's most recent data is December 2023, "Last 3 months" is therefore defined as October 1, 2023 through December 31, 2023\.

**2\. Tip earnings reflect non cash tips only.** Payment type breakdown shows cash trips average just $0.01 in recorded tips, versus $3.70 for credit card trips.

**3\. Trip duration is capped before use in shift modeling.** Trip duration shows a median trip of about 10.5 minutes, with the 90th percentile around 31.5 minutes, but the maximum recorded trip is 86,399 seconds which is just under 24 hours. This is treated as a data quality issue rather than a real trip. Trips longer than 4 hours (14,400 seconds) are treated as outliers and excluded from shift and overwork calculations in Question 2\.

**4\. Shift break threshold is set at 8 hours.** Gap between consecutive trips per taxi shows the bottom 40% of gaps are exactly 15 minutes, timestamp for privacy reasons. The 90th percentile gap is about 345 minutes (5.75 hours). Therefore an 8 hour gap is used as the threshold for "taking a break," consistent with a typical work shift boundary and comfortably above normal short layovers between fares. This is further supported by daily driving hour data for Oct to Dec 2023, where the 90th percentile taxi has around 4.7 hours of driving and the median is about 2.4 hours.

**5\. Inactivity gaps are excluded from shift logic.** Some taxi IDs show gaps of years between trips. These represent long term inactivity and are excluded from shift reconstruction. Gaps longer than 48 hours are treated as the start of a new analysis period rather than counted as one continuous "no break" stretch.

**6\. Overlapping trips are not corrected for shared medallions.** About 3.6% of trips (6,067,964 of 168,255,981) show a start time before the prior trip's recorded end time for the same taxi\_id. This likely reflects multiple drivers sharing one medallion or timestamp rounding. This project does not attempt to split these into separate driver identities; all metrics are reported at the taxi\_id (medallion) level, which may slightly understate true per driver workload in shared medallion cases.

## Exploratory Data Analysis Summary

| Check | Result |
| :---- | :---- |
| Date range | Jan 1, 2013 to Dec 31, 2023 |
| Total trips (post cleaning) | 168,255,981 |
| Unique taxi IDs | 8,228 |
| Avg trips per taxi (lifetime) | 20,449.2 |
| Median tip | $0.00 |
| 75th percentile tip | $2.00 |
| Max recorded tip | $898.88 (likely outlier) |
| Zero tip trips | 100,616,104 (60%) |
| Median trip duration | \~10.5 minutes |
| Max trip duration | 86,399 seconds (excluded as outlier) |
| Overlapping trips | 6,067,964 (3.6%) |
| Most common payment type | Cash (93,369,584 trips, avg tip $0.01) |
| Highest avg tip payment type | Credit Card ($3.70) |
| Negative tips or totals | 0 (none found) |
| Trips with tips over $100 | 605 |
| Trips where tip exceeds 2x the fare | 223,825 |
| Median trips per taxi per day (Q4 2023\) | 6 to 7 |
| 90th percentile trips per taxi per day (Q4 2023\) | 14 |
| Max trips per taxi per day (Q4 2023\) | 46 |
| Median driving hours per taxi per day (Q4 2023\) | \~2.4 hours |
| 90th percentile driving hours per taxi per day (Q4 2023\) | \~4.7 hours |
| Max driving hours per taxi per day (Q4 2023\) | 21.3 hours |
| Highest volume company | Flash Cab (286,457 trips, avg tip $1.57) |
| Highest avg tip company (meaningful volume) | Setare Inc ($4.85, 1,077 trips) |
| Peak demand hour | 5pm (105,103 trips) |
| Lowest demand hour | 2am (7,489 trips) |

## Analytical Questions: Methodology

**Question 1: Top 100 tip earners (last 3 months)** Aggregate recorded tips by taxi\_id over October to December 2023, rank using RANK() with QUALIFY to handle ties correctly rather than an arbitrary LIMIT 100 cutoff. Contextual metrics (total trips, average tip per trip) are included alongside the rank to give the figure more business meaning.

**Question 2: Top 100 overworkers** Reconstruct shifts per taxi\_id by ordering trips chronologically and treating any gap of 8 or more hours as a shift boundary, also excluding gaps over 48 hours as inactivity. Within each shift, sum total driving time and flag taxis with unusually long shifts or a pattern of working without the expected break.

**Question 3: Holiday impact on trip demand** Compare daily trip counts on major US public holidays against a baseline of the surrounding 14 days (7 before, 7 after, excluding other holidays). Analysis window is 2018 and 2019 to avoid COVID era disruption and to use years with stable, high volume operation. Holidays covered: New Year's Day, MLK Day, Memorial Day, Independence Day, Labor Day, Thanksgiving, Christmas Day.

**Bonus Insight 1: Hourly Demand Pattern** Trip volume, fare levels, and tip levels by hour of day for Oct 1 to Dec 31, 2023\. Hours are grouped into business-meaningful periods (Morning Rush, Midday, Evening Rush, Late Evening, Overnight).

**Bonus Insight 2: Company Performance Comparison** Volume, market share, fleet size, and per-trip economics for the 22 taxi companies operating with at least 1,000 trips in Oct to Dec 2023\.

**Bonus Insight 3: Geographic Demand Analysis** Trip pickups, dropoffs, and net flow aggregated by Chicago community area for Oct to Dec 2023. Census tracts were evaluated but excluded due to 55% suppressed. Lat/lng and community area fields with 97% coverage are used instead. Key findings: O'Hare (area 76) is the highest-volume pickup zone at 22.9% of all pickups with a net source of 247,700 trips, confirming taxis function primarily as airport-exit vehicles. Midway Airport (area 56) shows the same pattern. Near North Side (area 8) is the largest net dropoff destination.

## Known Limitations

- Cash tips are not reliably recorded, so tip based rankings undercount true gratuity for cash paying riders.  
- A small number of trips ( 3.6%) overlap in time for the same taxi\_id, likely due to shared medallions or rounding, and are not split into separate driver identities.  
- Some trip durations are clearly invalid (up to 24 hours) and are excluded from shift based analysis using a 4 hour cap.  
- Pickup and dropoff census tracts are suppressed for some trips per the source dataset's privacy protections, so location based analysis relies on community area and latitude/longitude fields instead.  
- The Question 2 overworked taxis ranking is reported at the taxi\_id (medallion) level, not the driver level, since drivers cannot be uniquely identified in this dataset. Top results may show implausibly long single shifts (the top result shows 322 hours, the second 217 hours) because shared medallions often have driver handoffs in under our 8 hour gap threshold, causing multiple drivers' shifts to be merged into one. The bulk of the top 100 (ranks 10 through 100, ranging from approximately 19 to 50 max shift hours) represents a more realistic "heavy usage" profile. The supporting columns avg\_shift\_hours and total\_shifts help distinguish patterns where a medallion's heavy usage comes from one consistently overworked driver versus multiple drivers sharing the same vehicle.

## Future Work
    
- A trips per taxi metric (fleet utilization) would complement the revenue per taxi and fare per mile metrics already added, distinguishing high-volume fleets from high-efficiency ones more precisely.  
    
- A Looker-side filter on the existing rank column could give viewers interactive control over how many top taxis to display (top 50, top 20, etc.) without changing the underlying SQL.  
    
- **A fully dynamic, Looker-driven date range was considered but not implemented.** The Q1, Q2, and company performance marts aggregate trip-level data down to ranked summary rows with no date column in the output, so a native Looker date filter would have nothing to filter on. Even if the SQL were restructured to expose a date dimension, the RANK() window functions in Q1 and Q2 are computed in BigQuery and cannot be recalculated by Looker Studio at view time. Q2 has an additional risk: its shift reconstruction relies on LAG() looking at each taxi's prior trip, so a dynamically selected window would incorrectly treat the first trip in that window as a new shift start, inflating shift counts at the boundary. With more time, this could be addressed using a BigQuery custom query data source with Looker's @DS\_START\_DATE / @DS\_END\_DATE parameters, which reruns the full aggregation per request, at the cost of higher query latency and the loss of Dataform's managed table outputs.
