#  Provide Insights to Chief of Operations in Transportation Domain


## Overview
Goodcabs, a cab service company established two years ago, has gained a strong foothold in the Indian market by focusing on tier-2 cities. Unlike other cab service providers, Goodcabs is committed to supporting local drivers, helping them make a sustainable living in their hometowns while ensuring excellent service to passengers. With operations in ten tier-2 cities across India, Goodcabs has set ambitious performance targets for 2024 to drive growth and improve passenger satisfaction. 

As part of this initiative, the Goodcabs management team aims to assess the company’s performance across key metrics, including trip volume, passenger satisfaction, repeat passenger rate, trip distribution, and the balance between new and repeat passengers. 

However, the Chief of Operations, Bruce Haryali, wanted this immediately but the analytics manager Tony is engaged on another critical project. Tony decided to give this work to Peter Pandey who is the curious data analyst of Goodcabs. Since these insights will be directly reported to the Chief of Operations, Tony also provided some notes to Peter to support his work

## Objectives

Check “ad-hoc-requests.pdf” - this document includes important business questions, requiring SQL-based report generation. 

## Dataset

The data for this project is sourced from the Kaggle dataset:

- **Dataset Link:** [Movies Dataset](https://www.kaggle.com/datasets/shivamb/netflix-shows?resource=download)

## Schema

```sql
CREATE TABLE fact_trips (
    trip_id VARCHAR(25) PRIMARY KEY, 
    trip_date DATE,
    city_id VARCHAR (10), 
    passenger_type VARCHAR(20),
    distance_travelled_km NUMERIC,
    fare_amount NUMERIC,
    passenger_rating NUMERIC,
    driver_rating NUMERIC
);


-- Table: dim_repeat_trip_distribution
CREATE TABLE dim_repeat_trip_distribution (
    month DATE,
    city_id VARCHAR(10) PRIMARY KEY,
    trip_count INT,
    repeat_passenger_count INT
);

-- Table: city_target_passenger_rating
CREATE TABLE city_target_passenger_rating (
    city_id VARCHAR(10) PRIMARY KEY,
    target_avg_passenger_rating NUMERIC
);

-- Table: dim_city
CREATE TABLE dim_city (
    city_id VARCHAR(10) PRIMARY KEY,
    city_name VARCHAR(20)
);

-- Table: dim_date
CREATE TABLE dim_date (
    date DATE PRIMARY KEY,
    start_of_month DATE,
    month_name VARCHAR(20),
    day_type VARCHAR(20)
);

-- Table: fact_passenger_summary
CREATE TABLE fact_passenger_summary (
    month DATE,
    city_id VARCHAR(10) PRIMARY KEY,
    new_passengers INT,
    repeat_passengers INT,
    total_passengers INT
);

-- Table: monthly_target_new_passengers
CREATE TABLE monthly_target_new_passengers (
    month DATE,
    city_id VARCHAR(10) PRIMARY KEY,
    target_new_passengers INT
);

-- Table: monthly_target_trips
CREATE TABLE monthly_target_trips (
    month DATE,
    city_id VARCHAR(10) PRIMARY KEY,
    total_target_trips INT
);


```

## Business Problems and Solutions

**Objective:** Generate a report that displays the total trips, average fare per km, average fare per trip, and the percentage contribution of each city's trips to the overall trips. This report will help in assessing trip volume, pricing efficiency, and each city's contribution to the overall trip count

### 1. City-Level Fare and Trip Summary Report

```sql
SELECT
    dc.city_name AS city_name,
    COUNT(*) AS total_trips,
    ROUND(SUM(fare_amount) / NULLIF(SUM(distance_travelled_km), 0), 2) AS avg_fare_per_km,
    ROUND(AVG(ft.fare_amount), 0) AS avg_fare_per_trip,
    ROUND(COUNT(*)::NUMERIC / (SELECT COUNT(*) FROM fact_trips)::NUMERIC * 100, 2) AS percentage_contribution_to_total_trips
FROM fact_trips AS ft
JOIN dim_city AS dc
     ON ft.city_id = dc.city_id
GROUP BY dc.city_name
ORDER BY total_trips DESC;
```

**Objective:** Generate a report that evaluates the target performance for trips at the monthly and city level. For each city and month, compare the actual total trips with the target trips and categorise the performance as follows:
e If actual trips are greater than target trips, mark it as "Above Target".
 	If actual trips are less than or equal to target trips, mark it as "Below Target".
Additionally, calculate the % difference between actual and target trips to quantify the performance gap.


### Business Request - 2: Monthly City-Level Trips Target Performance Report

```sql
WITH city_month AS (
    SELECT
        DATE_TRUNC('month', trip_date) AS month,
        ft.city_id,
        dc.city_name,
        COUNT(*) AS actual_trips
    FROM fact_trips AS ft
    JOIN dim_city AS dc
         ON ft.city_id = dc.city_id
    GROUP BY 1, 2, 3
)
SELECT 
    cm.city_name,
    TO_CHAR(cm.month, 'Month') AS month_name,
    SUM(cm.actual_trips) AS actual_trips,
    SUM(mt.total_target_trips) AS target_trips,
    CASE
        WHEN SUM(cm.actual_trips) > SUM(mt.total_target_trips) THEN 'Above Target'
        ELSE 'Below Target'
    END AS performance_status,
    ROUND(
        (SUM(cm.actual_trips) - SUM(mt.total_target_trips)) / NULLIF(SUM(mt.total_target_trips), 0) * 100,
        2
    ) AS percentage_difference
FROM city_month AS cm
JOIN monthly_target_trips AS mt 
     ON cm.city_id = mt.city_id
    AND cm.month = mt.month
GROUP BY cm.city_name, cm.month
ORDER BY cm.city_name, cm.month;
```

**Objective:** User Interest and Trends Analysis

### 3. What are the top 10 most popular genres on Netflix based on the count of titles added in each genre?

```sql
SELECT 
    UNNEST(STRING_TO_ARRAY(listed_in,',')) AS genre,
    COUNT(*) AS content,
    ROUND(COUNt(*)::NUMERIC / (SELECT COUNT(*) FROM netflix)::NUMERIC * 100,2) AS Average_content_Per_genre
FROM netflix
GROUP BY 1
ORDER BY 2 DESC
LIMIT 10;

```

**Objective:** Content Release Timing and Trends

### 4. During which month(s) of the year does Netflix release the most content, and how has this trend changed over the years?

```sql
WITH monthly_count AS 
    (SELECT  
    EXTRACT(YEAR FROM TO_DATE(date_added,'Month DD, Year')) AS release_year,
    EXTRACT(MONTH FROM TO_DATE(date_added,'Month DD, Year')) AS release_Months,
    COUNT(*) AS content
FROM netflix
GROUP BY 1,2
--HAVING COUNT(*) > 100
ORDER BY 2,3 DESC) 


 SELECT * 
 FROM       
    (SELECT release_year, release_Months, content, 
            ROW_NUMBER() OVER(PARTITION BY release_year ORDER BY content DESC) AS most_content
    FROM monthly_count)
WHERE most_content = 1 AND release_year IS NOT NULL
ORDER BY content DESC;
```

**Objective:** Top Directors by Genre

### 5. Who are the top 5 most prolific directors in each genre?

```sql
WITH gen_dic AS (SELECT 
    UNNEST(STRING_TO_ARRAY(listed_in,',')) AS genre,
    UNNEST(STRING_TO_ARRAY(director,',')) AS director,
    COUNT(show_id) AS content
FROM netflix
GROUP BY 1,2)

 SELECT *       
FROM        
    (SELECT *, 
        ROW_NUMBER() OVER(PARTITION BY genre ORDER BY content DESC) AS top
    FROM gen_dic
    WHERE director IS NOT NULL AND genre IS NOT NULL)
WHERE top <= 5 
```

**Objective:** Find the movie with the longest duration.

### 6. Find Content Added in the Last 5 Years

```sql
SELECT *
FROM netflix
WHERE TO_DATE(date_added, 'Month DD, YYYY') >= CURRENT_DATE - INTERVAL '5 years';
```

**Objective:** Audience Suitability Trends

### 7. How has the distribution of content ratings (e.g., PG, R, TV-MA) changed over the years?

```sql
WITH trends AS (SELECT
     rating,
     EXTRACT(YEAR FROM TO_DATE(date_added,'Month DD, Year')) AS release_year,
     COUNT(*)
FROM netflix
WHERE rating IN ('PG', 'R', 'TV-MA') AND date_added IS NOT NULL
GROUP BY 1, 2),

overall AS 
(SELECT EXTRACT(YEAR FROM TO_DATE(date_added, 'Month DD,Year')) AS release_year,
    COUNT(*) AS total_content
FROM netflix
WHERE date_added IS NOT NULL
GROUP BY 1)

SELECT 
    t.release_year, 
    t.rating,
    t.count, 
    o.total_content, 
    ROUND(t.count::NUMERIC/o.total_content::NUMERIC * 100,2) AS percentage
FROM trends AS t
JOIN overall AS o ON t.release_year = o.release_year
GROUP BY 1,2,3,4
ORDER BY 2,1;
```

**Objective:** List all content directed by 'Rajiv Chilaka'.

### 8. List All TV Shows with More Than 5 Seasons

```sql
SELECT *
FROM netflix
WHERE type = 'TV Show'
  AND SPLIT_PART(duration, ' ', 1)::INT > 5;
```

**Objective:** International Content Growth

### 9. Which countries have seen the highest growth in Netflix content over the years?

```sql
WITH content AS (
    SELECT 
        EXTRACT(YEAR FROM TO_DATE(date_added, 'Month DD, YYYY')) AS release_year,
        TRIM(UNNEST(STRING_TO_ARRAY(country, ','))) AS countries,
        COUNT(*) AS total_content
    FROM netflix
    WHERE date_added IS NOT NULL
    GROUP BY 1, 2
)

SELECT 
    release_year,
    countries,
    total_content,
    LAG(total_content) OVER (PARTITION BY countries ORDER BY release_year) AS previous_year_content,
    ROUND(
        (total_content::NUMERIC - LAG(total_content) OVER (PARTITION BY countries ORDER BY release_year)) 
        / NULLIF(LAG(total_content) OVER (PARTITION BY countries ORDER BY release_year), 0) * 100, 
        2
    ) AS growth_percentage
FROM 
    content
WHERE 
    release_year > 2018
    AND countries IN ('France', 'Germany', 'India', 'Japan', 'Russia', 'South Korea')
ORDER BY 
    countries, release_year;
```

**Objective:** Count the number of content items in each genre.

### 10.Find each year and the average numbers of content release in India on netflix. 
return top 5 year with highest avg content release!

```sql
SELECT 
    country,
    release_year,
    COUNT(show_id) AS total_release,
    ROUND(
        COUNT(show_id)::numeric /
        (SELECT COUNT(show_id) FROM netflix WHERE country = 'India')::numeric * 100, 2
    ) AS avg_release
FROM netflix
WHERE country = 'India'
GROUP BY country, release_year
ORDER BY avg_release DESC
LIMIT 5;
```

**Objective:** Calculate and rank years by the average number of content releases by India.

### 11. List All Movies that are Documentaries

```sql
SELECT * 
FROM netflix
WHERE listed_in LIKE '%Documentaries';
```

**Objective:** Retrieve all movies classified as documentaries.

### 12. Find All Content Without a Director

```sql
SELECT * 
FROM netflix
WHERE director IS NULL;
```

**Objective:** List content that does not have a director.

### 13. Find How Many Movies Actor 'Salman Khan' Appeared in the Last 10 Years

```sql
SELECT * 
FROM netflix
WHERE casts LIKE '%Salman Khan%'
  AND release_year > EXTRACT(YEAR FROM CURRENT_DATE) - 10;
```

**Objective:** Count the number of movies featuring 'Salman Khan' in the last 10 years.

### 14. Find the Top 10 Actors Who Have Appeared in the Highest Number of Movies Produced in India

```sql
SELECT 
    UNNEST(STRING_TO_ARRAY(casts, ',')) AS actor,
    COUNT(*)
FROM netflix
WHERE country = 'India'
GROUP BY actor
ORDER BY COUNT(*) DESC
LIMIT 10;
```

**Objective:** Identify the top 10 actors with the most appearances in Indian-produced movies.

### 15. Categorize Content Based on the Presence of 'Kill' and 'Violence' Keywords

```sql
SELECT 
    category,
    COUNT(*) AS content_count
FROM (
    SELECT 
        CASE 
            WHEN description ILIKE '%kill%' OR description ILIKE '%violence%' THEN 'Bad'
            ELSE 'Good'
        END AS category
    FROM netflix
) AS categorized_content
GROUP BY category;
```

**Objective:** Categorize content as 'Bad' if it contains 'kill' or 'violence' and 'Good' otherwise. Count the number of items in each category.

## Findings and Conclusion

- **Content Distribution:** The dataset contains a diverse range of movies and TV shows with varying ratings and genres.
- **Common Ratings:** Insights into the most common ratings provide an understanding of the content's target audience.
- **Geographical Insights:** The top countries and the average content releases by India highlight regional content distribution.
- **Content Categorization:** Categorizing content based on specific keywords helps in understanding the nature of content available on Netflix.

This analysis provides a comprehensive view of Netflix's content and can help inform content strategy and decision-making.



