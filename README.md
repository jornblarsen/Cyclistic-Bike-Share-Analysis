# Cyclistic-Bike-Share-Analysis
Coursera Google Data Analytics Certificate Capstone

**Introduction**

In this case study, I was told to imagine I worked for a fictional bike-share company, Cyclistic, and I needed to answer some business questions by following the six steps of the data analysis process: Ask, Prepare, Process, Analyze, Share, and Act.

**Scenario / Business Task**

The director of marketing at Cyclistic believes that the key to the company's future growth is maximizing the number of annual members. Thus, I was tasked with leveraging the last 12 months of bike trip data to understand how Cyclistic's casual riders differ from annual members, who pay a subscription. Using the insight gained from the data analysis, me and my team would design a new marketing strategy to best convert our casual riders into annual members.

**Data Sources & Preparation**
```sql
CREATE OR REPLACE TABLE `project-b4912155-c932-494a-b31.cyclistic_data.jun_2025_to_may_2026_trips` AS
SELECT * FROM `project-b4912155-c932-494a-b31.cyclistic_data.jun_2025_trips`
UNION ALL
SELECT * FROM `project-b4912155-c932-494a-b31.cyclistic_data.jul_2025_trips`
UNION ALL
SELECT * FROM `project-b4912155-c932-494a-b31.cyclistic_data.aug_2025_trips`
UNION ALL
SELECT * FROM `project-b4912155-c932-494a-b31.cyclistic_data.sep_2025_trips`
UNION ALL
SELECT * FROM `project-b4912155-c932-494a-b31.cyclistic_data.oct_2025_trips`
UNION ALL
SELECT * FROM `project-b4912155-c932-494a-b31.cyclistic_data.nov_2025_trips`
UNION ALL
SELECT * FROM `project-b4912155-c932-494a-b31.cyclistic_data.dec_2025_trips`
UNION ALL
SELECT * FROM `project-b4912155-c932-494a-b31.cyclistic_data.jan_2026_trips`
UNION ALL
SELECT * FROM `project-b4912155-c932-494a-b31.cyclistic_data.feb_2026_trips`
UNION ALL
SELECT * FROM `project-b4912155-c932-494a-b31.cyclistic_data.mar_2026_trips`
UNION ALL
SELECT * FROM `project-b4912155-c932-494a-b31.cyclistic_data.apr_2026_trips`
UNION ALL
SELECT * FROM `project-b4912155-c932-494a-b31.cyclistic_data.may_2026_trips`
```
**Data Cleaning & Processing**

While standard guidelines suggest removing trips under 60 seconds, a deeper analysis of the Cyclistic dataset revealed a high volume of trips lasting between 1 and 3 minutes where the start and end stations were completely identical. I deduced that these represented mechanical swaps or immediate cancellations rather than real consumer utility. I instituted a conditional cleaning rule to remove trips under 3 minutes only if the origin and destination matched, protecting the integrity of the true trip-duration averages.

```sql
CREATE OR REPLACE TABLE `project-b4912155-c932-494a-b31.cyclistic_data.cleaned_annual_trips` AS

-- Step 1: Calculate the new columns first
WITH calculated_metrics AS (
	SELECT 
		ride_id,
		rideable_type,
		started_at,
		ended_at,
		TIMESTAMP_DIFF(ended_at, started_at, MINUTE) AS ride_length_minutes,
		TIMESTAMP_DIFF(ended_at, started_at, SECOND) AS ride_length_seconds,
		EXTRACT(DAYOFWEEK FROM started_at) AS day_of_week,
		-- Safely replace NULL station values with 'Unknown'
		COALESCE(start_station_name, 'Unknown') AS start_station_name,
		start_station_id,
		COALESCE(end_station_name, 'Unknown') AS end_station_name,
		end_station_id,
		start_lat,
		start_lng,
		end_lat,
		end_lng,
		member_casual
	FROM 
		`project-b4912155-c932-494a-b31.cyclistic_data.jun_2025_to_may_2026_trips`
)

SELECT *
FROM calculated_metrics
WHERE 
	-- CRITERIA A: Absolute minimum ride length rule (removes negative times & < 60s false-starts)
	-- filters out 162,692 rows
	ride_length_seconds >= 60

	-- CRITERIA B: Maximum realistic ride length rule (stolen or unreturned bikes)
	-- filters out 5,843 rows
	AND ride_length_minutes < 1440 

	-- CRITERIA C: Same-Station Mechanical Filter (drops trips < 3 mins starting and ending at same spot)
	-- Filters out 64,389 rows
	AND NOT(
		start_station_name = end_station_name
		AND start_station_name != 'Unknown'  -- <-- Safeguard added here!
		AND ride_length_minutes < 3
	);
```

Out of the initial annual dataset, a total of 195,913 rows (approx. 3.35% of the data) were identified as anomalies and filtered out. The majority of these removals consisted of system test data, lost assets exceeding 24 hours, false starts under 60 seconds, and same-station bike swaps lasting under 3 minutes. Removing these distortions ensures that the subsequent analysis on trip durations and route frequencies reflects true consumer behavior.

When sorting ride lengths in descending order, I discovered a significant cluster of trips capping out right at 1,439 minutes (24 hours), indicating system timeouts or unreturned assets. To prevent these outliers from artificially inflating the average ride duration, I analyzed the trip distribution and determined that over 98% of trips naturally conclude within 3 hours. I applied a filter to exclude any trips exceeding 24 hours (or 3 hours, depending on what you chose) to ensure accurate reporting on standard user behavior.

Ran a grouping query to find the "cliff":

```sql
SELECT 
	CEIL(ride_length_minutes / 60) AS ride_hours,
	COUNT(*) AS trip_count
FROM
	`project-b4912155-c932-494a-b31.cyclistic_data.cleaned_annual_trips`
GROUP BY
	ride_hours
ORDER BY
	ride_hours ASC;
```
<img width="157" height="587" alt="image" src="https://github.com/user-attachments/assets/b7ee0b9e-9d4f-42aa-b1db-2e682454e17b" />

While the dataset technically includes a very small number of trips lasting between 4 and 24 hours, an exploratory distribution analysis revealed that of all user rides naturally conclude within 3 hours. Therefore, the remaining analysis focuses on this 0-to-3 hour window to accurately reflect standard everyday commuting and leisure habits.

Re-ran my main cleaning query to filter out rides > 180 minutes (3 hours)
Filtered down from 5652790 rows to 5641726	(99.8%)
In total, 5,848,703 to 5641726	

**Key Insights & Visualizations**

Calculated the mean of ride_length:

```sql
SELECT
	AVG(ride_length_seconds)/60 as avg_ride_length_mins
FROM
	`project-b4912155-c932-494a-b31.cyclistic_data.cleaned_annual_trips`
```
Result: 14.080016458320307 mins

Calculate max ride_length
```sql
SELECT
	MAX(ride_length_seconds)/60 as max_ride_length_mins
FROM
	`project-b4912155-c932-494a-b31.cyclistic_data.cleaned_annual_trips`
```
Result: 180.98333333333332 mins

Calculated the mode of day_of_week:

```sql
SELECT
	day_of_week,
	COUNT(ride_id) AS total_trips
FROM
	`project-b4912155-c932-494a-b31.cyclistic_data.cleaned_annual_trips`
GROUP BY 
	day_of_week
ORDER BY 
	total_trips DESC
```
<img width="246" height="211" alt="image" src="https://github.com/user-attachments/assets/7a2d27bc-978a-4452-ab9d-dd4d8e259086" />

Calculate average ride_length for members
```sql
SELECT
	AVG(ride_length_seconds)/60 AS avg_member_ride_length_min
FROM
	`project-b4912155-c932-494a-b31.cyclistic_data.cleaned_annual_trips`
WHERE
	member_casual = 'member'
```
Result: 11.815538006743516 (mins)

Average ride_length for casuals
```sql
SELECT
	AVG(ride_length_seconds)/60 AS avg_casual_ride_length_min
FROM
	`project-b4912155-c932-494a-b31.cyclistic_data.cleaned_annual_trips`
WHERE
	member_casual = 'casual'
```
Result: 18.241123202802097 (mins)

Average ride_length for members for each day of the week
```sql
SELECT
	day_of_week,
	ROUND((AVG(ride_length_seconds)/60), 2) AS avg_member_duration_minutes,
	COUNT(ride_id) AS total_member_trips
FROM
	`project-b4912155-c932-494a-b31.cyclistic_data.cleaned_annual_trips`
WHERE
	member_casual = 'member'
GROUP BY
	day_of_week
ORDER BY 
	day_of_week ASC;
```
<img width="377" height="211" alt="image" src="https://github.com/user-attachments/assets/4211f659-04dd-45e1-821a-c3e770405fba" />

Average ride_length for casuals for each day of the week
```sql
SELECT
	day_of_week,
	ROUND((AVG(ride_length_seconds)/60), 2) AS avg_casual_duration_minutes,
	COUNT(ride_id) AS total_casual_trips
FROM
	`project-b4912155-c932-494a-b31.cyclistic_data.cleaned_annual_trips`
WHERE
	member_casual = 'casual'
GROUP BY
	day_of_week
ORDER BY 
	day_of_week ASC;
```
<img width="422" height="211" alt="image" src="https://github.com/user-attachments/assets/55353933-4be9-45e0-b41b-847e0b3d030f" />

Total number of member trips per day
```sql
SELECT
	day_of_week,
	COUNT(ride_id) AS total_member_trips
FROM
	`project-b4912155-c932-494a-b31.cyclistic_data.cleaned_annual_trips`
WHERE
	member_casual = 'member'
GROUP BY
	day_of_week
ORDER BY 
	total_member_trips DESC;
```
<img width="247" height="211" alt="image" src="https://github.com/user-attachments/assets/f4530dca-708a-4a54-bd5b-e772d97241e1" />

Total number of casual trips per day
```sql
SELECT
	day_of_week,
	COUNT(ride_id) AS total_casual_trips
FROM
	`project-b4912155-c932-494a-b31.cyclistic_data.cleaned_annual_trips`
WHERE
	member_casual = 'casual'
GROUP BY
	day_of_week
ORDER BY 
	total_casual_trips DESC;
```
<img width="249" height="211" alt="image" src="https://github.com/user-attachments/assets/b96563c9-b842-4c30-b98e-0fb571a885e1" />


**Data-Driven Recommendations**
