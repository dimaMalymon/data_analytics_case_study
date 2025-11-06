# Cyclistic — Case Study 1 (SQL / BigQuery)

**Author:**  _Dmytro Malymon_

**Project path (BigQuery table):**  `casestudy1-473810.cyclistic.all_trips_from_09_2024_to08_2025`

# Table of contents

 1. Ask
 2. Prepare
 3. Process
 4. Analyze
 5. Share
 6. Act

# 1) Ask

-   **Business Task:** Cyclistic’s marketing team needs to understand **how annual members and casual riders use Cyclistic bikes differently**. By identifying usage patterns, the company can design targeted strategies to convert casual riders into annual members—maximizing profitability and supporting sustainable growth.
    
-   **Problem Statement:** Casual riders generate short-term revenue through single-ride and day passes, but **annual members are significantly more profitable**. To increase the number of members, the team must uncover key differences in how these two groups use the service.
    
-   **Guiding Questions:** 
	1. How do **annual members** and **casual riders** use Cyclistic bikes differently?
	    
	2. Why might casual riders be motivated to purchase annual memberships?
	    
	3. How can Cyclistic use digital media to influence casual riders to become members?

- **Key Stakeholders:**

	-   **Lily Moreno** — Director of Marketing, responsible for overall strategy.
	    
	-   **Cyclistic Marketing Analytics Team** — Data analysts (including you) providing insights.
	    
	-   **Cyclistic Executive Team** — Decision-makers who will approve or reject the marketing plan.
- **Deliverables:**
	-   A clear statement of the business task.
	    
	-   A description of the data sources used.
	    
	-   Documentation of data cleaning and transformation.
	    
	-   A summary of the analysis.
	    
	-   Visualizations and key findings.
	    
	-   Three actionable recommendations for the marketing strategy.

# 2) Prepare

## Data Source

-   **Cyclistic Historical Trip Data**: Publicly available trip records covering **September 2024 – August 2025**.
    
-   The dataset was published by **Motivate International Inc.**, the real operator of Chicago’s Divvy bike-share program.
    
-   Licensing: The data is released under an open license for public use and excludes personally identifiable information (PII).

## Storage & Access

-   The 12 monthly CSV files were combined into one BigQuery table:  
    `casestudy1-473810.cyclistic.all_trips_from_09_2024_to08_2025`
    
-   This table is stored in **Google BigQuery**, which provides scalability, security, and accessibility for analysis.
    
-   A clean analysis table (`cyclistic_analytics.rides_clean`) will be created later to apply filters and derived fields.
> **Problem:** BigQuery local upload is limited to 100MB, but one CSV was ~130MB.
>     
> **Solution:** Used Google Cloud Storage (GCS) bucket as staging, then imported from GCS to BigQuery.
>     
> **Lesson learned:** For files >100MB use GCS. 
> 
> **How to Use GCS with Your Files**
> 
> 1.  Go to **Google Cloud Console** → **Cloud Storage**.
>     
> 2.  Create a new **bucket** (like a folder in the cloud):
>     
>     -   Example name: `cyclistic-bike-data`
>         
>     -   Choose location (US recommended for free tier).
>         
>     -   Storage class: Standard.
>         
> 3.  Upload your 12 CSVs into this bucket.
>     
>     -   You can just drag-and-drop them in.
>         
>     -   They’ll appear like `gs://cyclistic-bike-data/202409-divvy-tripdata.csv`, etc.
>         
> 4.  In BigQuery → **Create table**:
>     
>     -   Source = `Google Cloud Storage`.
>         
>     -   File path = `gs://cyclistic-bike-data/*.csv` (the `*` means “all CSVs in this folder”).
>         
>     -   File format = CSV.
>         
>     -   Destination = your dataset, e.g. `cyclistic.all_trips_from_09_2024_to08_2025`.
>         
>     -   Schema = Auto-detect (or set manually).
>         
>     -   Load.
## Data Organization

The raw dataset includes one row per ride. Schema of table:

| Field               | Type      | Notes |
|----------------------|-----------|-------|
| ride_id             | STRING    | Unique trip ID |
| rideable_type       | STRING    | Bike type (`classic_bike`, `electric_bike`, `docked_bike`) |
| started_at          | TIMESTAMP | Trip start timestamp |
| ended_at            | TIMESTAMP | Trip end timestamp |
| start_station_name  | STRING    | Start station name |
| start_station_id    | STRING    | Start station ID |
| end_station_name    | STRING    | End station name |
| end_station_id      | STRING    | End station ID |
| start_lat           | FLOAT     | Start latitude |
| start_lng           | FLOAT     | Start longitude |
| end_lat             | FLOAT     | End latitude |
| end_lng             | FLOAT     | End longitude |
| member_casual       | STRING    | Rider type (`member` or `casual`) |

## Credibility & ROCCC check

-   **Reliable**: Data published by Motivate International Inc., a trusted provider.
    
-   **Original**: Directly collected from Cyclistic/Divvy’s bike-share system.
    
-   **Comprehensive**: Covers the full 12-month period without sampling.
    
-   **Current**: Data ends August 2025, giving a full recent year.
    
-   **Cited**: Properly attributed to Motivate International Inc.

## Privacy & Security

-   PII is excluded (no names, addresses, or payment data).
    
-   Only trip-level, anonymized operational data is used.
    
-   Analysis complies with licensing and privacy requirements.

## Data Integrity Verification

-   Confirmed correct number of rows loaded into BigQuery.
    
-   Verified unique `ride_id` values (no duplicates).

**SQL Query:**
```SQL
-- Row count and distinct ride IDs
SELECT COUNT(*) AS rows_total,
       COUNT(DISTINCT ride_id) AS unique_ride_ids
FROM `casestudy1-473810.cyclistic.all_trips_from_09_2024_to08_2025`;
```
**Output:**
	
| rows_total | unique_ride_ids |
|------------|-----------------|
| 5646038  | 5646038       |

> Conclusion: The dataset contains **5646038 rows**, and all `ride_id` values are unique. This confirms that there are no duplicate trip records, and the dataset is consistent at the ride identifier level.

-   Checked time coverage (`MIN(started_at)`, `MAX(started_at)`).

**SQL Query:**
```sql
-- Time coverage of the dataset
SELECT MIN(started_at) AS first_trip,
       MAX(started_at) AS last_trip
FROM `casestudy1-473810.cyclistic.all_trips_from_09_2024_to08_2025`;
```
**Output:**
| first_trip                  | last_trip                   |
|-----------------------------|-----------------------------|
| 2024-08-31 00:00:38.766000 UTC | 2025-08-31 23:55:36.521000 UTC |

> **Problem**: When verifying time coverage, the earliest `started_at` value was **2024-08-31 00:00:38 UTC**.   Since the dataset was expected to span **September 2024 – August 2025**, this indicates that the data provider grouped files by **end date of the trip** rather than strictly by the start date.
> **Additional Check**: To confirm the time coverage, I grouped trips by the month of `started_at`. 
> 
**SQL query**:
 ```sql
-- Rows per month (should show 12 months of data)
SELECT FORMAT_TIMESTAMP('%Y-%m', started_at) AS trip_month,
       COUNT(*) AS trips
FROM `casestudy1-473810.cyclistic.all_trips_from_09_2024_to08_2025`
GROUP BY trip_month
ORDER BY trip_month;
```
> **Result:**
> -   The dataset spans **September 2024 – August 2025**, as expected. 
> -   However, **444 trips** began in **August 2024**, confirming that a small number of trips spill over from the prior month.
> 
> **Additional Check**: To confirm whether the dataset was sorted by **trip end date (`ended_at`)** instead of start date, I checked if any trips finished in **September 2025**.

**SQL query:**
```sql
-- Check if any trips ended in September 2025
SELECT FORMAT_TIMESTAMP('%Y-%m', ended_at) AS end_month,
       COUNT(*) AS trips
FROM `casestudy1-473810.cyclistic.all_trips_from_09_2024_to08_2025`
WHERE FORMAT_TIMESTAMP('%Y-%m', ended_at) = '2025-09'
GROUP BY end_month
ORDER BY end_month;
```

> **Result:**
> 
> -   The query returned **0 rows**, which means there are no trips that ended in **September 2025**.
> -   This indicates that the dataset **stops cleanly at August 2025**.
> -  A previous check found **444 trips starting in August 2024**, confirming that the files are grouped by trip end date, not start date.
> 
> **Decision**
> 
> -   The analysis period will be described as **September 2024 – August 2025**.
> - The **444 trips that started in August 2024** fall outside this timeframe and were therefore **excluded during data cleaning** in the Process stage.
> - This ensures a consistent 12-month dataset, avoiding any temporal overlap from the prior month.

    
-   Inspected categories in `member_casual` and `rideable_type`.

**SQL query:**

```sql
-- Rider type and bike type categories
SELECT ARRAY_AGG(DISTINCT member_casual) AS rider_types,
       ARRAY_AGG(DISTINCT rideable_type) AS bike_types
FROM `casestudy1-473810.cyclistic.all_trips_from_09_2024_to08_2025`;
```
**Output:**

| rider_types | bike_types        |
|-------------|-------------------|
| casual      | classic_bike      |
| member      | electric_bike     |
|             | electric_scooter  |

> **Conclusion:**  The dataset contains two valid rider categories: **casual** and **member**.   Bike types include **classic_bike**, **electric_bike**, and **electric_scooter**.
> **Problem:** The dataset includes _electric_scooter_ rides, which are outside Cyclistic’s core bike-share fleet.
> **Additional check:** To assess their impact, I ran a query to check scooter usage and found **144337 trips**. Since this represents a non-trivial share of activity and I cannot confirm with stakeholders whether to exclude them, I will **keep these trips in the analysis** while documenting this consideration.


    
- Check for missing values in key fields.

**SQL query:**
       
```sql
-- Check for missing values in key fields
SELECT
  SUM(CASE WHEN ride_id IS NULL THEN 1 ELSE 0 END) AS null_ride_id,
  SUM(CASE WHEN started_at IS NULL THEN 1 ELSE 0 END) AS null_started_at,
  SUM(CASE WHEN ended_at IS NULL THEN 1 ELSE 0 END) AS null_ended_at,
  SUM(CASE WHEN start_station_name IS NULL THEN 1 ELSE 0 END) AS null_start_station,
  SUM(CASE WHEN end_station_name IS NULL THEN 1 ELSE 0 END) AS null_end_station
FROM `casestudy1-473810.cyclistic.all_trips_from_09_2024_to08_2025`;
```

**Output:**
| null_ride_id | null_started_at | null_ended_at | null_start_station | null_end_station |
|--------------|-----------------|---------------|--------------------|------------------|
| 0            | 0               | 0             | 1161918          | 1204302        |

> **Result:**
> 
> -   `ride_id`, `started_at`, and `ended_at` have **no nulls** (0 values).
> -   `start_station_name` is missing in **1,161,918 rows**.
> -   `end_station_name` is missing in **1,204,302 rows**.
>     
> 
> **Conclusion:**   The core trip identifiers and timestamps are complete, which ensures reliable duration and trip-level analysis. However, a significant portion of rows lack **station names**. These nulls may limit station-level insights (e.g., top stations, common routes). Trips with missing stations will be retained for most analyses, but when station-based metrics are required, I will restrict queries to rows with valid station names.

 - Detect negative or zero-length rides (potential anomalies)

**SQL query:**
```sql
-- Detect negative or zero-length rides (potential anomalies)
SELECT
  COUNTIF(TIMESTAMP_DIFF(ended_at, started_at, SECOND) < 0) AS negative_durations,
  COUNTIF(TIMESTAMP_DIFF(ended_at, started_at, SECOND) BETWEEN 0 AND 59) AS less_than_one_minute,
  COUNTIF(TIMESTAMP_DIFF(ended_at, started_at, SECOND) >= 60) AS valid_over_one_minute
FROM `casestudy1-473810.cyclistic.all_trips_from_09_2024_to08_2025`;
```
**Output:**
| negative_durations | less_than_one_minute | valid_over_one_minute |
|--------------------|----------------------|-----------------------|
| 43                 | 138397               | 5507598               |


> **Result (example interpretation):**
> -   A portion of rides have **negative durations** → broken records.
> -   Another portion are **short rides under 60 seconds** → valid unlock/lock events but not meaningful for analysis.
> -   The majority are **valid trips longer than 1 minute**.
> 
> **Conclusion:**
> -   Rides with **negative durations** will be removed as broken data.
> -   Rides with **<1 minute duration** will also be excluded, since they don’t represent meaningful bike usage and may bias results.





## How this data helps answer the question ?

-   The data provides **rider type, bike type, trip times, stations, and durations**, which directly support the main business question:  
    **“How do annual members and casual riders use Cyclistic bikes differently?”**
    
-   By comparing casual vs. member behaviors across time, duration, station usage, and bike type, we can reveal actionable differences for marketing strategy.

# 3) Process
## Tools Used

-   **Google BigQuery** is the main tool for data cleaning and transformation.   
-   SQL was chosen because it allows efficient handling of large datasets (5.6M+ rows) directly in the cloud without the need to download or split data across multiple local files.   
-   This ensures **data integrity**, **reproducibility**, and **scalability**, as all transformations are fully traceable within SQL queries.

## Data Cleaning and Transformation Steps

### 1. Verified data integrity

Before processing, several validation checks were performed (see _Prepare_ section):

-   Confirmed **unique ride IDs**.
-   Checked for **missing values** in key fields.
-   Identified **invalid or short-duration trips** (under 1 minute).
-   Verified **dataset timeframe** and **rider type categories**.
    

These checks confirmed that the data is suitable for cleaning and analysis.

### 2. Create a Clean Analytical Table

A clean dataset was created from the raw table  
`casestudy1-473810.cyclistic.all_trips_from_09_2024_to08_2025`.  
This transformation standardizes fields, removes anomalies, and derives additional columns needed for analysis.

```sql
-- Creates a dataset (schema) if it doesn't already exist.
CREATE SCHEMA IF NOT EXISTS `casestudy1-473810.cyclistic_analytics`;

CREATE OR REPLACE TABLE `casestudy1-473810.cyclistic_analytics.rides_clean` AS
WITH base AS (
  SELECT
    ride_id,
    rideable_type,
    TIMESTAMP(started_at) AS started_at,
     -- TIMESTAMP(): ensures date/time values are stored in a standard timestamp format.
    TIMESTAMP(ended_at)   AS ended_at,
    start_station_name, start_station_id,
    end_station_name, end_station_id,
    SAFE_CAST(start_lat AS FLOAT64) AS start_lat,
    SAFE_CAST(start_lng AS FLOAT64) AS start_lng,
    SAFE_CAST(end_lat   AS FLOAT64) AS end_lat,
    SAFE_CAST(end_lng   AS FLOAT64) AS end_lng,
    -- SAFE_CAST(): safely converts data types; returns NULL instead of error if conversion fails.
    LOWER(member_casual) AS member_casual,
    -- LOWER(): converts text to lowercase for consistent category values.
    TIMESTAMP_DIFF(ended_at, started_at, SECOND) AS ride_length_sec
    -- TIMESTAMP_DIFF(): calculates the time difference between two timestamps (here in seconds).
  FROM `casestudy1-473810.cyclistic.all_trips_from_09_2024_to08_2025`
)
, filtered AS (
  SELECT *,
         ride_length_sec / 60.0 AS ride_length_min
         -- Converts seconds into minutes for easier analysis.
  FROM base
  WHERE
    -- Remove broken data and keep only meaningful trips (≥1 minute)
    ride_length_sec >= 60
    -- Remove trips that started before September 2024 to align with the analysis timeframe
     AND DATE(started_at) >= '2024-09-01'
)
SELECT
  *,
  DATE(started_at) AS ride_date,
  -- DATE(): extracts only the date part (YYYY-MM-DD) from a timestamp.
  FORMAT_DATE('%Y-%m', DATE(started_at)) AS ride_month,
  -- FORMAT_DATE(): formats the date into a specific string pattern (e.g., 2024-09).
  EXTRACT(ISOWEEK FROM DATE(started_at)) AS ride_week,
  -- EXTRACT(): pulls a specific component (ISO week number) from a date.
  EXTRACT(YEAR FROM DATE(started_at)) AS ride_year,
  -- Extracts the year number from a date.
  FORMAT_TIMESTAMP('%A', started_at) AS weekday_name,
  -- FORMAT_TIMESTAMP('%A'): returns the weekday name (e.g., Monday).
  CASE WHEN EXTRACT(DAYOFWEEK FROM started_at) IN (1,7) THEN TRUE ELSE FALSE END AS is_weekend,
  -- DAYOFWEEK(): returns a number 1–7 (Sunday–Saturday); used to flag weekends.
  EXTRACT(HOUR FROM started_at) AS hour_of_day,
  -- Extracts the hour (0–23) from a timestamp.
  CASE
    WHEN EXTRACT(HOUR FROM started_at) BETWEEN 5 AND 10 THEN 'Morning'
    WHEN EXTRACT(HOUR FROM started_at) BETWEEN 11 AND 16 THEN 'Midday'
    WHEN EXTRACT(HOUR FROM started_at) BETWEEN 17 AND 20 THEN 'Evening'
    ELSE 'Night'
  END AS daypart
   -- CASE: conditional logic to group hours into parts of day.
FROM filtered
```
### 3. Data Cleaning Rules Applied

-   **Removed negative durations:** eliminates corrupted records where `ended_at < started_at`.    
-   **Excluded rides shorter than 1 minute:** removes false starts, unlock/lock tests, and accidental unlocks. 
- removing a small number of rides that began in late August 2024. 
-   **Added derived fields:**  
	-   `ride_length_sec` , `ride_length_min`       
    -   `ride_month`, `ride_week`, `ride_year`    
    -   `weekday_name`, `is_weekend`, `hour_of_day`, `daypart`

### 4. Verification of Clean Data

To confirm that cleaning was successful, validation queries were executed:
```sql
-- Check record count and duration statistics
SELECT
  COUNT(*) AS rows_clean,
  MIN(ride_length_min) AS min_duration_min,
  MAX(ride_length_min) AS max_duration_min,
  AVG(ride_length_min) AS avg_duration_min
FROM `casestudy1-473810.cyclistic_analytics.rides_clean`;

--Verify that all expected bike types are present
SELECT DISTINCT rideable_type
FROM `casestudy1-473810.cyclistic_analytics.rides_clean`;

-- Ensure only valid rider categories remain
SELECT DISTINCT member_casual
FROM `casestudy1-473810.cyclistic_analytics.rides_clean`;
```
## **Conclusion**

The raw data was successfully cleaned and transformed into a reliable analytical dataset:

-   All trips have valid start and end timestamps. 
-   Negative and sub-minute durations were removed.
- The final dataset includes trips from September 2024 through August 2025.  
-   New temporal and categorical fields were added for analysis.
    
The resulting table `cyclistic_analytics.rides_clean` is now **ready for the Analyze stage**.


# **4) Analyze**

## **Purpose**

The goal of this stage is to explore and summarize the cleaned Cyclistic data to identify behavioral differences between **annual members** and **casual riders**.  
All analysis is conducted in **Google BigQuery using SQL**, which allows fast aggregation and flexible trend discovery across over 5 million rows of data.

## **Step 1 — Data Preparation for Analysis**

The cleaned dataset `casestudy1-473810.cyclistic_analytics.rides_clean` is used for all queries.  
It includes derived columns such as:

-   `ride_length_min`  
-   `ride_month`
-   `weekday_name` 
-   `is_weekend` 
-   `daypart` 
-   `rideable_type`
-   `member_casual`
    

The data is already standardized and ready for aggregation — no additional formatting required.

## 🔹 1. Descriptive Statistics

### **Overall Summary**

```sql
SELECT
  COUNT(*) AS total_rides,
  ROUND(AVG(ride_length_min), 2) AS avg_duration_min,
  ROUND(APPROX_QUANTILES(ride_length_min, 100)[OFFSET(50)], 2) AS median_duration_min,
  ROUND(MIN(ride_length_min), 2) AS min_duration_min,
  ROUND(MAX(ride_length_min), 2) AS max_duration_min
FROM `casestudy1-473810.cyclistic_analytics.rides_clean`;
```
**Purpose:** Provides a general overview of ride duration distribution across all riders.  
**Key metrics:** total rides, average, median, min, max.
**Output:**
| total_rides | avg_duration_min | median_duration_min | min_duration_min | max_duration_min |
|--------------|------------------|---------------------|------------------|------------------|
| 5,507,155    | 16.45            | 9.65                | 1.0              | 1574.9           |


### **Summary by Rider Type**

```sql
SELECT
  member_casual,
  COUNT(*) AS rides,
  ROUND(AVG(ride_length_min), 2) AS avg_duration_min,
  ROUND(APPROX_QUANTILES(ride_length_min, 100)[OFFSET(50)], 2) AS median_duration_min
FROM `casestudy1-473810.cyclistic_analytics.rides_clean`
GROUP BY member_casual
ORDER BY member_casual;
```
**Purpose:** Compares ride count and duration between **casual** and **member** riders.  
**Expected insight:** Casual riders usually have longer average ride times, while members ride more frequently.
**Output:**
| member_casual | rides    | avg_duration_min | median_duration_min |
|----------------|-----------|------------------|---------------------|
| casual         | 2,001,935 | 23.68            | 11.92               |
| member         | 3,505,220 | 12.31            | 8.65                

## 🔹 2. Temporal Patterns
### **Rides by Month**

```sql
SELECT
  ride_month,
  member_casual,
  COUNT(*) AS rides,
  ROUND(AVG(ride_length_min), 2) AS avg_duration_min
FROM `casestudy1-473810.cyclistic_analytics.rides_clean`
GROUP BY ride_month, member_casual
ORDER BY ride_month, member_casual;
```
**Purpose:** Tracks seasonality and usage differences across months.  
**Expected insight:** Summer months typically show higher ridership, especially for casual users.
**Output:**
| ride_month | member_casual | rides   | avg_duration_min |
|-------------|---------------|---------|------------------|
| 2024-09     | casual        | 334,863 | 22.09            |
| 2024-09     | member        | 465,705 | 12.41            |
| 2024-10     | casual        | 210,743 | 23.88            |
| 2024-10     | member        | 394,751 | 12.13            |
| 2024-11     | casual        | 90,805  | 19.87            |
| 2024-11     | member        | 238,823 | 11.23            |
| 2024-12     | casual        | 37,466  | 18.03            |
| 2024-12     | member        | 137,600 | 10.93            |
| 2025-01     | casual        | 23,453  | 15.01            |
| 2025-01     | member        | 112,352 | 10.25            |
| 2025-02     | casual        | 27,051  | 15.01            |
| 2025-02     | member        | 122,118 | 10.21            |
| 2025-03     | casual        | 83,064  | 21.45            |
| 2025-03     | member        | 208,500 | 11.42            |
| 2025-04     | casual        | 105,528 | 22.41            |
| 2025-04     | member        | 257,962 | 11.50            |
| 2025-05     | casual        | 176,162 | 25.08            |
| 2025-05     | member        | 314,091 | 12.34            |
| 2025-06     | casual        | 279,570 | 26.49            |
| 2025-06     | member        | 379,609 | 13.18            |
| 2025-07     | casual        | 309,293 | 25.48            |
| 2025-07     | member        | 430,535 | 13.63            |
| 2025-08     | casual        | 323,937 | 24.35            |
| 2025-08     | member        | 443,174 | 13.35            |


### **Rides by Day of Week**

```sql
SELECT
  weekday_name,
  member_casual,
  COUNT(*) AS rides,
  ROUND(AVG(ride_length_min), 2) AS avg_duration_min
FROM `casestudy1-473810.cyclistic_analytics.rides_clean`
GROUP BY weekday_name, member_casual
ORDER BY
  CASE weekday_name
    WHEN 'Monday' THEN 1
    WHEN 'Tuesday' THEN 2
    WHEN 'Wednesday' THEN 3
    WHEN 'Thursday' THEN 4
    WHEN 'Friday' THEN 5
    WHEN 'Saturday' THEN 6
    WHEN 'Sunday' THEN 7
  END,
  member_casual;
```
**Purpose:** Shows which days of the week are most popular by rider type.  
**Expected insight:** Members ride more on weekdays (commuting), casual riders on weekends (leisure).
**Output:**
| weekday_name | member_casual | rides   | avg_duration_min |
|---------------|---------------|---------|------------------|
| Monday        | casual        | 233,140 | 23.00            |
| Monday        | member        | 506,615 | 11.78            |
| Tuesday       | casual        | 223,373 | 20.61            |
| Tuesday       | member        | 551,367 | 11.86            |
| Wednesday     | casual        | 227,015 | 19.84            |
| Wednesday     | member        | 543,649 | 11.77            |
| Thursday      | casual        | 254,954 | 20.80            |
| Thursday      | member        | 555,243 | 11.89            |
| Friday        | casual        | 311,288 | 23.36            |
| Friday        | member        | 511,973 | 12.26            |
| Saturday      | casual        | 406,217 | 26.74            |
| Saturday      | member        | 444,038 | 13.52            |
| Sunday        | casual        | 345,948 | 27.46            |
| Sunday        | member        | 392,335 | 13.70            |


### **Rides by Daypart**

```sql
SELECT
  daypart,
  member_casual,
  COUNT(*) AS rides
FROM `casestudy1-473810.cyclistic_analytics.rides_clean`
GROUP BY daypart, member_casual
ORDER BY daypart, member_casual;
```


**Purpose:** Reveals when users ride during the day.  
**Expected insight:** Members tend to ride in morning/evening (commute hours), while casual riders prefer midday.
**Output:**
| daypart | member_casual | rides   |
|----------|---------------|---------|
| Evening  | casual        | 560,225 |
| Evening  | member        | 1,002,189 |
| Midday   | casual        | 844,281 |
| Midday   | member        | 1,294,237 |
| Morning  | casual        | 324,148 |
| Morning  | member        | 894,450 |
| Night    | casual        | 273,281 |
| Night    | member        | 314,344 |


## 🔹 3. Equipment & Station Insights

### **Bike Type Usage**

```sql
SELECT
  rideable_type,
  member_casual,
  COUNT(*) AS rides
FROM `casestudy1-473810.cyclistic_analytics.rides_clean`
GROUP BY rideable_type, member_casual
ORDER BY member_casual, rideable_type;
```
**Purpose:** Compares preferences for classic, electric, and scooter rides by user type.  
**Expected insight:** Casual riders likely use more electric bikes and scooters for convenience.
**Output:**
| rideable_type   | member_casual | rides   |
|-----------------|---------------|---------|
| classic_bike    | casual        | 761,764 |
| electric_bike   | casual        | 1,158,784 |
| electric_scooter| casual        | 81,387  |
| classic_bike    | member        | 1,390,272 |
| electric_bike   | member        | 2,058,833 |
| electric_scooter| member        | 56,115  |

### **Top Start Stations**

```sql
SELECT
  member_casual,
  start_station_name,
  COUNT(*) AS start_count
FROM `casestudy1-473810.cyclistic_analytics.rides_clean`
WHERE start_station_name IS NOT NULL
GROUP BY member_casual, start_station_name
QUALIFY ROW_NUMBER() OVER (PARTITION BY member_casual ORDER BY COUNT(*) DESC) <= 10;
```


**Purpose:** Identifies the most popular starting locations for each rider type.  
**Expected insight:** Casual riders favor tourist areas; members use stations near work/residential zones.
**Output:**
| member_casual | start_station_name                    | start_count |
|----------------|---------------------------------------|-------------|
| casual         | Streeter Dr & Grand Ave               | 36,964      |
| casual         | DuSable Lake Shore Dr & Monroe St     | 31,863      |
| casual         | Michigan Ave & Oak St                 | 22,821      |
| casual         | Millennium Park                       | 20,936      |
| casual         | DuSable Lake Shore Dr & North Blvd    | 19,630      |
| casual         | Shedd Aquarium                        | 18,328      |
| casual         | Dusable Harbor                         | 16,481      |
| casual         | Theater on the Lake                    | 15,769      |
| casual         | Navy Pier                              | 13,022      |
| casual         | Michigan Ave & 8th St                  | 12,396      |
| member         | Kingsbury St & Kinzie St              | 31,824      |
| member         | Clinton St & Washington Blvd          | 25,874      |
| member         | Clinton St & Madison St               | 23,209      |
| member         | Clark St & Elm St                     | 23,109      |
| member         | Canal St & Madison St                 | 21,976      |
| member         | Clinton St & Jackson Blvd             | 19,431      |
| member         | Wells St & Elm St                     | 19,414      |
| member         | State St & Chicago Ave                | 18,985      |
| member         | Wells St & Concord Ln                 | 18,722      |
| member         | University Ave & 57th St              | 17,609      |

## 🔹 4. Additional Analysis (Optional Deep Dives)

### **Weekend vs Weekday Comparison**

```sql
SELECT
  member_casual,
  is_weekend,
  COUNT(*) AS rides,
  ROUND(AVG(ride_length_min), 2) AS avg_duration_min
FROM `casestudy1-473810.cyclistic_analytics.rides_clean`
GROUP BY member_casual, is_weekend
ORDER BY member_casual, is_weekend;
```

**Purpose:** Measures how weekday vs weekend behavior differs.  
**Expected insight:** Casual riders dominate weekends; members ride consistently all week.
**Output:**
| member_casual | is_weekend | rides   | avg_duration_min |
|----------------|------------|---------|------------------|
| casual         | false      | 1,249,770 | 21.64           |
| casual         | true       | 752,165   | 27.07           |
| member         | false      | 2,668,847 | 11.91           |
| member         | true       | 836,373   | 13.60           |


### **Seasonal Patterns (Quarterly Summary)**

```sql
SELECT
  CASE
    WHEN EXTRACT(MONTH FROM ride_date) IN (12,1,2) THEN 'Winter'
    WHEN EXTRACT(MONTH FROM ride_date) IN (3,4,5) THEN 'Spring'
    WHEN EXTRACT(MONTH FROM ride_date) IN (6,7,8) THEN 'Summer'
    ELSE 'Autumn'
  END AS season,
  member_casual,
  COUNT(*) AS rides,
  ROUND(AVG(ride_length_min), 2) AS avg_duration_min
FROM `casestudy1-473810.cyclistic_analytics.rides_clean`
GROUP BY season, member_casual
ORDER BY season, member_casual;
```


**Purpose:** Simplifies trends by season, showing how weather affects usage.
**Output:**

| season | member_casual | rides   | avg_duration_min |
|---------|---------------|---------|------------------|
| Autumn | casual        | 636,411 | 22.37            |
| Autumn | member        | 1,099,279 | 12.05           |
| Spring | casual        | 364,754 | 23.48            |
| Spring | member        | 780,553 | 11.82            |
| Summer | casual        | 912,800 | 25.39            |
| Summer | member        | 1,253,318 | 13.39           |
| Winter | casual        | 87,970  | 16.30            |
| Winter | member        | 372,070 | 10.49            |

## 🔹 5. Summary of Key Trends & Relationships


### **1. Ride frequency and duration**
-   **Members:** Take a much higher number of rides overall .   
-   **Casual riders:** Ride less frequently but have **longer average trip durations**.  
    → _Interpretation:_ Members use bikes as part of regular commuting or short trips; casual riders use them for leisure or exploration.
    
### **2. Seasonality**

-   Both groups peak in the **summer months** (June–August).
-   Casual rider activity increases sharply during warm months, while members maintain steadier usage year-round.  
    → _Interpretation:_ Casual users are influenced by weather and tourism, members ride for consistent daily needs.
    
### 3. Weekday vs. Weekend Patterns
-   **Both groups ride more on weekdays**, but **members are more weekday-oriented**.
-   **Casual riders** have a stronger weekend share and **ride longer on weekends**.  
    → _Interpretation:_ Members mainly commute; casual riders balance weekday and weekend leisure trips.

### **4. Daypart patterns**

-   Both members and casual riders peak at **Midday**. Members are more balanced with substantial **Morning** and **Evening** rides (commute signature), while casual riders are **more Midday-heavy** and have a **higher Night** share.


### **5. Bike type preferences**

-   Electric dominates for both members and casuals (~58%), classic ~40%, scooters minimal; scooter share is higher among casuals.

### **6. Station usage**

-   **Casual riders:** Top starts cluster at **lakefront & tourist hubs** — Streeter Dr & Grand (Navy Pier area), DuSable Lake Shore Dr & Monroe (Millennium Park), Michigan & Oak (beachfront), Shedd Aquarium, DuSable Harbor, Theater on the Lake, etc.
    
-   **Members:** Concentrated near **downtown transit and job centers** — Clinton/Canal/Madison/Jackson (Ogilvie & Union Station area), Kingsbury & Kinzie (River North), Clark/Wells & Elm/Concord (Near North/Old Town), State & Chicago (Red Line), plus **University Ave & 57th** (UChicago) indicating academic/commuter usage.
    → _Interpretation:_ Casual = leisure/tourism corridors; Members = commute/transit/residential nodes.
    

## **Conclusion**

All SQL analysis confirms that **casual riders and members use Cyclistic bikes in distinct ways**:

-   Members use bikes regularly as part of commuting routines.    
-   Casual riders use bikes more seasonally, for leisure or tourism.
    
These patterns directly support marketing strategies aimed at **converting casual users** by promoting membership benefits such as cost savings for frequent rides or convenience for recurring commuters.

# **5) Share**

## Purpose
The goal of this stage was to present the analysis in a **clear, executive-ready format** that answers the core question:

> **How do annual members and casual riders use Cyclistic bikes differently, and how can these insights support membership growth?**

The final deliverable is a **PowerPoint presentation** built from data prepared in **BigQuery**, with visualizations created in **Excel** and **Tableau**.

## Tools for Communication

-   **BigQuery (SQL):** Used to clean, transform, and aggregate the full dataset (`rides_clean`).   
-   **Excel:** Used to quickly convert exported summary tables into clean charts for selected views.    
-   **Tableau:** Used to build more polished and interactive visualizations (station maps), then exported for use in the slide deck.    
-   **PowerPoint:** Used to structure the narrative and present final insights to stakeholders.
    

All visuals are based on **BigQuery summary outputs**, ensuring consistency between code, charts, and conclusions.

## What I Shared

### 1. Framing & methodology

-   Timeframe: **September 2024 – August 2025** (trips starting before 2024-09-01 excluded).   
-   Explained that all analysis was done in SQL on a cleaned dataset, then exported to Excel/Tableau for visualization.    
-   Key cleaning rules:    
    -   Removed trips < 60 seconds and negative durations.        
    -   Retained legitimate long-duration rides.       
    -   Added key derived fields (ride_length_min, weekday/weekend, daypart, month, bike type, etc.).        
-   Focus: comparison of **members vs casuals**.

### 2. Core visualizations (Excel/Tableau → PPT)

-   **Rides by month & rider type** (line chart)  
    → Shows seasonality and member dominance in overall volume.
    
-   **Weekday vs weekend usage** (clustered bars)  
    → Shows both segments are weekday-heavy, with weekends relatively more important and longer for casual riders.
    
-   **Daypart distribution** (100% stacked bars)  
    → Midday leads for both; members show stronger morning/evening presence; casuals skew more Midday & Night.
    
-   **Bike type mix** (100% stacked bars)  
    → Electric bikes dominate for both; scooters are minor but more used by casual riders.
    
-   **Top start stations by rider type** (map)  
    → Casuals: tourist & lakefront hubs; Members: transit, business, and residential nodes.


## Key Messages Communicated

Across the slides, I emphasized that:

1.  **Members ride more frequently and predictably** (strong weekday and commute patterns).   
2.  **Casual riders ride less often**, more influenced by season, location, and time of day.    
3.  **Specific stations and time windows** reveal high-potential opportunities to convert casual riders into members.

Each visual included:

-   A **title that states the insight** (not just “Chart”).    
-   1–3 short bullets explaining what to see.    
-   Consistent color coding for **member vs casual** across all tools.


# 6) Act

## Final Conclusion

The analysis of Cyclistic trips from **September 2024 to August 2025** shows clear, consistent differences between **annual members** and **casual riders**:

-   **Members** ride far more frequently, mainly on **weekdays**, with strong **morning and evening** activity around key transit and job hubs.
    
-   **Casual riders** ride less frequently but take **longer trips**, are more sensitive to **season and weekends**, favor **tourist and lakefront stations**, and show greater usage of scooters and electric bikes.


## Top 3 Recommendations based on my analysis

### 1) Target “commuter-like” casual riders at key stations
**What:**  
Deploy focused membership campaigns at stations where **casual rides are high**, especially those with:

-   High **weekday** usage    
-   High **morning/evening** share (commute hours)

**How to act:**

-   In-app prompts and QR codes at docks: _“Riding here often? Save with a monthly or annual pass.”_
-   Digital signage and geo-targeted ads around those stations.    
-   Limited-time “Commute Starter” membership discounts for riders with repeated casual usage on those routes.

**Why:**  
These riders already use Cyclistic like members; a clear value message + low-friction signup can convert them efficiently.

### 2) Sell membership on **value + convenience** for electric and frequent riders

**What:**  
Both members and casual riders use **electric bikes heavily**. Casuals who ride electric or take longer trips are paying more per ride.

**How to act:**

-   In-app pricing comparison after casual rides:  
    _“You spent $X on trips this month. A membership would cost $Y.”_    
-   Specific **electric-inclusive membership messaging**:   
    -   “Predictable cost for your electric rides”
    -   “Unlimited short trips for commuters & regular riders”
-   Retarget riders with **3+ casual rides per month** via email/app notifications showing savings.

**Why:**  
Your data shows strong electric usage and repeat behavior; cost transparency directly addresses the main barrier to upgrading.


### 3) Use seasonal & location-based campaigns to convert high-intent casuals

**What:**  
Casual demand spikes in **summer**, **weekends**, and around **tourist hotspots** (e.g., Streeter Dr & Grand Ave, Millennium Park, Navy Pier area).

**How to act:**

-   Run **“End of Summer / Stay Riding”** campaigns:    
    -   Offer discounts on annual passes to casual riders with multiple summer trips.        
-   At high-casual tourist stations: 
    -   Promote **short-term membership or monthly passes** instead of repeated single rides.        
-   Bundle:   
    -   Weekend/holiday passes with a clear path to upgrade: _“Turn this pass into a membership and we’ll deduct what you’ve already spent.”_

**Why:**  
You meet casual riders where they already are and turn peak-season enthusiasm into longer-term commitment.


## Next Steps (for Cyclistic team)

If this were a real engagement, recommended next actions:

1.  **Run targeted experiments**    
    -   A/B test dock signage + in-app membership offers at top 5–10 conversion pocket stations.        
    -   Track uplift in **membership signups**, **repeat rides**, and **electric usage**.        
2.  **Integrate behavioral triggers**  
    -   Automatically trigger membership prompts for users hitting specific thresholds (e.g., 3+ casual rides/month, predominantly commute hours).        
3.  **Refine pricing & packages**   
    -   Explore **monthly** and **seasonal** memberships for casual-heavy segments (e.g., summer riders, campus riders near University Ave & 57th St).

## Additional Data to Consider

To deepen or validate the strategy, Cyclistic could incorporate:

-   **Demographic or survey data** (where ethically available) to understand who is converting.    
-   **App/web engagement data** to see which nudges or screens lead to membership signup.   
-   **Weather and events data** to refine seasonal and weekend campaigns.    
-   **Price sensitivity tests** (different discount levels per station/segment).
    
These would not change the core insights but would help **optimize** targeting and messaging.
