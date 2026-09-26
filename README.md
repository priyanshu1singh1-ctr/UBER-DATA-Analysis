# UBER-DATA-Analysis
Ride Trip Analytics — SQL Data Cleaning &amp; Business Insights


An end-to-end SQL project on a real 2016 ride-trip log: from a messy raw export to a cleaned analytical table to 15 business questions answered in SQL, each tied back to a decision a stakeholder would actually care about (reimbursement, driver scheduling, data quality for reporting).



<img width="586" height="497" alt="image" src="https://github.com/user-attachments/assets/fce8fe1a-e03f-4c24-9178-2efd8fa93f98" />






1. DATASET
	
Source	- 999 individual ride records for 2016, one row per trip
Columns - 	START_DATE, END_DATE, CATEGORY, START, STOP, MILES, PURPOSE
Categories	- Business (92.3% of trips), Personal (7.7%)
Raw file	 - data/uber_raw.csv
Cleaned output	-  data/uber_trips_cleaned.csv (998 rows)


2. TOOLS

SQL (portable to PostgreSQL/MySQL) · CTEs · Window functions (RANK, LAG, SUM OVER) · Self-joins · Subqueries · data cleaning with pure SQL string functions.


3. The 15 business questions

Each question was run against the cleaned uber_trips table.

Q1. Trip count and mileage, Business vs Personal
<img width="547" height="87" alt="image" src="https://github.com/user-attachments/assets/8fa828ea-b05c-41a1-b040-df03dc5da570" />




SELECT category, COUNT(*) AS total_trips, ROUND(SUM(miles),1) AS total_miles,
ROUND(AVG(miles),2) AS avg_miles_per_trip
FROM uber_trips GROUP BY category ORDER BY total_trips DESC;


Insight: Business trips are 92% of volume and run ~22% longer on average than Personal trips and mileage reimbursement exposure is concentrated almost entirely in Business use.





Q2. Category's share of trips vs. share of miles
<img width="511" height="87" alt="image" src="https://github.com/user-attachments/assets/7aa1754e-ad0c-4c9e-8b73-f1a610491b41" />

WITH totals AS (
    SELECT SUM(miles) AS grand_miles, COUNT(*) AS grand_trips FROM uber_trips
)
SELECT
    t.category,
    COUNT(*)   AS trips,
    ROUND(100.0 * COUNT(*) / totals.grand_trips, 1)       AS pct_of_trips,
    ROUND(SUM(t.miles), 1)      AS miles,
    ROUND(100.0 * SUM(t.miles) / totals.grand_miles, 1)   AS pct_of_miles
FROM uber_trips t, totals
GROUP BY t.category, totals.grand_trips, totals.grand_miles;

Insight: Business's share of miles (93.6%) slightly exceeds its share of trips (92.3%) and consistent with the longer average distance in Q1, not a separate effect.





Q3. Business mileage by purpose, and the missing-purpose gap
<img width="502" height="256" alt="image" src="https://github.com/user-attachments/assets/f70bb72f-89cd-4106-87a3-670aec40c684" />


SELECT
    COALESCE(purpose, 'Not Specified')       AS purpose,
    COUNT(*)                                 AS trips,
    ROUND(SUM(miles), 1)                     AS total_miles,
    ROUND(100.0 * COUNT(*) /
        (SELECT COUNT(*) FROM uber_trips WHERE category = 'Business'), 1) AS pct_of_business_trips
FROM uber_trips
WHERE category = 'Business'
GROUP BY purpose
ORDER BY total_miles DESC;


Insight: 45% of Business trips and the single largest mileage block — carry no logged purpose. If this feeds an expense system, that gap is a compliance and audit risk before it's an analytics one.






Q4. Top 10 start locations
<img width="496" height="417" alt="image" src="https://github.com/user-attachments/assets/fbbf3301-e481-4e56-90f9-335624a800e0" />

SELECT
    start_location,
    COUNT(*)                                                              AS trips_started,
    SUM(CASE WHEN start_location = 'Unknown Location' THEN 1 ELSE 0 END)  AS is_unknown_flag
FROM uber_trips
GROUP BY start_location
ORDER BY trips_started DESC
LIMIT 10;


Insight: "Unknown Location" would rank as the #2 pickup point if taken at face value — 11% of all trips have no real start location logged. Any location-based report must filter this out explicitly (as every later query here does), or it silently distorts the ranking.







Q5. Locations with above-average trip volume (subquery + filter)

SELECT start_location, trips
FROM (
    SELECT start_location, COUNT(*) AS trips
    FROM uber_trips
    WHERE start_location <> 'Unknown Location'
    GROUP BY start_location
) loc_counts
WHERE trips > (
    SELECT AVG(cnt) FROM (
        SELECT COUNT(*) AS cnt
        FROM uber_trips
        WHERE start_location <> 'Unknown Location'
        GROUP BY start_location
    )
)
ORDER BY trips DESC






Q6. Round trips vs. one-way
<img width="505" height="81" alt="image" src="https://github.com/user-attachments/assets/c1632c60-9b3f-4593-8ebc-8ac8944feb5a" />


SELECT
    CASE WHEN start_location = stop_location THEN 'Round Trip' ELSE 'One-Way' END AS trip_type,
    COUNT(*)               AS trips,
    ROUND(AVG(miles), 2)   AS avg_miles
FROM uber_trips
GROUP BY trip_type

Insight: One in five trips is a same-location round trip, and — as expected — they run ~25% shorter than one-way trips, a useful sanity check that the distance data behaves logically






Q7. Distance profile by category
<img width="587" height="85" alt="image" src="https://github.com/user-attachments/assets/2faf44ca-3efd-4adc-ac9b-c52824eebe02" />

SELECT
    category,
    ROUND(AVG(miles), 2) AS avg_miles,
    MIN(miles)           AS min_miles,
    MAX(miles)           AS max_miles
FROM uber_trips
GROUP BY category






Q8. The single longest trip
<img width="960" height="112" alt="image" src="https://github.com/user-attachments/assets/bc9634b2-e808-4f1b-9f5f-999df519885e" />

SELECT trip_id, start_datetime, start_location, stop_location, miles, purpose
FROM uber_trips
ORDER BY miles DESC
LIMIT 1;







Q9. Day-of-week pattern
SELECT
    CASE strftime('%w', start_datetime)
        WHEN '0' THEN 'Sunday' WHEN '1' THEN 'Monday' WHEN '2' THEN 'Tuesday'
        WHEN '3' THEN 'Wednesday' WHEN '4' THEN 'Thursday' WHEN '5' THEN 'Friday'
        ELSE 'Saturday' END                                                    AS day_of_week,
    CASE WHEN strftime('%w', start_datetime) IN ('0','6') THEN 'Weekend' ELSE 'Weekday' END AS day_type,
    COUNT(*)              AS trips,
    ROUND(SUM(miles), 1)  AS total_miles,
    ROUND(AVG(miles), 2)  AS avg_miles
FROM uber_trips
GROUP BY day_of_week, day_type
ORDER BY trips DESC;

Friday is the busiest day (167 trips), narrowly ahead of Sunday (165) — weekdays account for 70% of all trips (703 of 998) vs. 30% on weekends, but average trip distance is similar across both (≈11.3 vs ≈11.0 miles), so the weekday/weekend split shows up in volume, not trip length.








Q10. Peak time-of-day for Business trips
<img width="356" height="127" alt="image" src="https://github.com/user-attachments/assets/fb890ca2-95e8-49a7-9180-926de159a0b2" />

SELECT
    CASE
        WHEN CAST(strftime('%H', start_datetime) AS INTEGER) BETWEEN 6 AND 11  THEN '06:00-11:59 Morning'
        WHEN CAST(strftime('%H', start_datetime) AS INTEGER) BETWEEN 12 AND 16 THEN '12:00-16:59 Afternoon'
        WHEN CAST(strftime('%H', start_datetime) AS INTEGER) BETWEEN 17 AND 20 THEN '17:00-20:59 Evening'
        ELSE '21:00-05:59 Night'
    END                    AS time_band,
    COUNT(*)               AS business_trips
FROM uber_trips
WHERE category = 'Business'
GROUP BY time_band
ORDER BY business_trips DESC;


Insight: Afternoon is the peak Business window (39% of all Business trips) and the practical takeaway for a scheduling stakeholder is that availability matters most between 12:00 and 17:00, not at commute times.







Q11. Top 3 locations by mileage, per category (RANK() OVER PARTITION BY)
<img width="625" height="175" alt="image" src="https://github.com/user-attachments/assets/8c53d1ad-7bdc-4b7c-a345-cbbdda029b70" />

WITH loc_miles AS (
    SELECT category, start_location, SUM(miles) AS total_miles
    FROM uber_trips
    WHERE start_location <> 'Unknown Location'
    GROUP BY category, start_location
),
ranked AS (
    SELECT *, RANK() OVER (PARTITION BY category ORDER BY total_miles DESC) AS mileage_rank
    FROM loc_miles
)
SELECT category, start_location, ROUND(total_miles, 1) AS total_miles, mileage_rank
FROM ranked
WHERE mileage_rank <= 3
ORDER BY category, mileage_rank;







Q12. Running (cumulative) Business mileage — reimbursement tracking

SELECT
    trip_id,
    start_datetime,
    miles,
    ROUND(SUM(miles) OVER (
        ORDER BY start_datetime ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ), 1) AS running_business_miles
FROM uber_trips
WHERE category = 'Business'
ORDER BY start_datetime;




<img width="637" height="582" alt="image" src="https://github.com/user-attachments/assets/5939ebfb-5cf8-4d81-9d6d-95417451f4a5" />






# What this project demonstrates

* Diagnosing real data-quality issues by profiling data, not assuming it's clean.
* Writing dialect-portable SQL to solve a problem a built-in date-cast function can't handle (inconsistent formatting), rather than reaching for a external cleaning tool
* CTEs, subqueries, HAVING, self-joins, and window functions (RANK, LAG, running SUM) used for a reason, not for their own sake
* Translating each query into a one-line answer a non-technical stakeholder could act on







<img width="987" height="570" alt="image" src="https://github.com/user-attachments/assets/72162b6b-cf5f-4596-8191-7133b6bae3c3" />
















