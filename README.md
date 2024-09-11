# Netflix Movies and TV Shows Data Analysis using SQL

![](https://github.com/najirh/netflix_sql_project/blob/main/logo.png)

## Overview
- This project involves a comprehensive analysis of Netflix's movies and TV shows data using SQL.
- The goal is to extract valuable insights and answer various business questions based on the dataset. 
- The following README provides a detailed account of the project's objectives, business problems, solutions, findings, and conclusions.

## Objectives

- Analyze the distribution of content types (movies vs TV shows).
- Identify the most common ratings for movies and TV shows.
- List and analyze content based on release years, countries, and durations.
- Explore and categorize content based on specific criteria and keywords.

## Dataset

The data for this project is sourced from the Kaggle dataset:

- **Dataset Link:** [Movies Dataset](https://www.kaggle.com/datasets/shivamb/netflix-shows?resource=download)

## Schema

```sql
DROP TABLE IF EXISTS netflix;
CREATE TABLE netflix
(
    show_id      VARCHAR(5),
    type         VARCHAR(10),
    title        VARCHAR(250),
    director     VARCHAR(550),
    cast        VARCHAR(5050),
    country      VARCHAR(550),
    date_added   VARCHAR(55),
    release_year INT,
    rating       VARCHAR(15),
    duration     VARCHAR(15),
    listed_in    VARCHAR(250),
    description  VARCHAR(550)
);
```

## Business Problems and Solutions

### 1. Count the Number of Movies vs TV Shows

```sql
select 
	type,
    count(*)
from
	netflix
group by 
	1;
```

**Objective:** Determine the distribution of content types on Netflix.

### 2. Find the Most Common Rating for Movies and TV Shows

```sql
with cte as (
select 
	type,rating
    ,count(*),
    dense_rank() over(partition by type order by count(*) desc) rnk
from 
	netflix
group by 
	1,2
order by 
	count(*) desc
)
select type,rating from cte
where rnk=1;
```

**Objective:** Identify the most frequently occurring rating for each type of content.

### 3. List All Movies Released in a Specific Year (e.g., 2020)

```sql
select
	*
from 
	netflix
where release_year=2020 and type='Movie';
```

**Objective:** Retrieve all movies released in a specific year.

### 4. Find the Top 5 Countries with the Most Content on Netflix

```sql
WITH RECURSIVE split_strings AS (
    SELECT
        SUBSTRING_INDEX(country, ',', 1) AS country,
        SUBSTRING(country, LENGTH(SUBSTRING_INDEX(country, ',', 1)) + 2) AS remaining
    FROM netflix
    UNION ALL
    SELECT
        SUBSTRING_INDEX(remaining, ',', 1),
        SUBSTRING(remaining, LENGTH(SUBSTRING_INDEX(remaining, ',', 1)) + 2)
    FROM split_strings
    WHERE remaining <> ''
)
SELECT 
	country
	, COUNT(*) 
FROM 
	split_strings
where 
	country<>'NULL'
GROUP BY 
	country
ORDER BY 
	COUNT(*) DESC
limit 5;
```

**Objective:** Identify the top 5 countries with the highest number of content items.

### 5. Identify the Longest Movie

```sql
select 
	* 
from 
	netflix
where 
	type='Movie' and 
    duration =(select MAX(CAST(duration AS UNSIGNED)) FROM netflix);
```

**Objective:** Find the movie with the longest duration.

### 6. Find Content Added in the Last 5 Years

```sql
select 
	* 
from 
	netflix
where 
	DATE_FORMAT(STR_TO_DATE(date_added, '%M %d, %Y'), '%Y-%m-%d')>=DATE_SUB(CURDATE(),INTERVAL 5 YEAR);
```

**Objective:** Retrieve content added to Netflix in the last 5 years.

### 7. Find All Movies/TV Shows by Director 'Rajiv Chilaka'

```sql
select
	*
from
	netflix
where
	director like '%Rajiv Chilaka%';
```

**Objective:** List all content directed by 'Rajiv Chilaka'.

### 8. List All TV Shows with More Than 5 Seasons

```sql
select 
	* 
from 
	netflix
where 
	type='TV Show' and 
	cast(SUBSTRING_INDEX(duration,' ',1) as UNSIGNED) > 5          -- --either use casting or SUBSTRING_INDEX
-- and CAST(duration AS UNSIGNED)>5;
```

**Objective:** Identify TV shows with more than 5 seasons.

### 9. Count the Number of Content Items in Each Genre

```sql
WITH RECURSIVE split_strings AS (
    SELECT
        SUBSTRING_INDEX(listed_in, ',', 1) AS genre,
        SUBSTRING(listed_in, LENGTH(SUBSTRING_INDEX(listed_in, ',', 1)) + 2) AS remaining
    FROM netflix
    UNION ALL
    SELECT
        SUBSTRING_INDEX(remaining, ',', 1),
        SUBSTRING(remaining, LENGTH(SUBSTRING_INDEX(remaining, ',', 1)) + 2)
    FROM split_strings
    WHERE remaining <> ''
)
SELECT 
	genre
	, COUNT(*) total_content
FROM 
	split_strings
where 
	genre<>'NULL'
GROUP BY 
	genre
ORDER BY 
	COUNT(*) DESC;
```

**Objective:** Count the number of content items in each genre.

### 10.Find each year and the average numbers of content release in India on netflix. 
return top 5 year with highest avg content release!

```sql
WITH RECURSIVE split_strings AS (
    SELECT
        SUBSTRING_INDEX(country, ',', 1) AS country,
        SUBSTRING(country, LENGTH(SUBSTRING_INDEX(country, ',', 1)) + 2) AS remaining,
        date_added -- Include the date_added column in the initial selection
    FROM netflix
    UNION ALL
    SELECT
        SUBSTRING_INDEX(remaining, ',', 1),
        SUBSTRING(remaining, LENGTH(SUBSTRING_INDEX(remaining, ',', 1)) + 2),
        date_added -- Pass the same date_added value for each split genre
    FROM split_strings
    WHERE remaining <> ''
)
SELECT 
	country,
    YEAR(STR_TO_DATE(date_added, '%M %d, %Y')),
	round(COUNT(*)/(select count(*) from split_strings where country = 'India')*100,2)  average_content
FROM 
	split_strings
WHERE 
	country = 'India'
GROUP BY 
	1,2 -- Group by both genre and date_added
ORDER BY 
	average_content DESC
limit 5;
```

**Objective:** Calculate and rank years by the average number of content releases by India.

### 11. List All Movies that are Documentaries

```sql
select 
	* 
from 
	netflix
where 
	type='Movie' and 
    listed_in like '%Documentaries%';
```

**Objective:** Retrieve all movies classified as documentaries.

### 12. Find All Content Without a Director

```sql
select 
	* 
from 
	netflix
where
	director ='NULL';
```

**Objective:** List content that does not have a director.

### 13. Find How Many Movies Actor 'Salman Khan' Appeared in the Last 10 Years

```sql
select 
	*
from
	netflix
where
	cast like '%Salman Khan%'
    and release_year>=YEAR(DATE_SUB(CURDATE(),INTERVAL 10 YEAR));
```

**Objective:** Count the number of movies featuring 'Salman Khan' in the last 10 years.

### 14. Find the Top 10 Actors Who Have Appeared in the Highest Number of Movies Produced in India

```sql
WITH RECURSIVE cast_split AS (
    SELECT 
        show_id,
        cast AS full_cast,
        SUBSTRING_INDEX(cast, ',', 1) AS single_cast,
        LENGTH(cast) - LENGTH(REPLACE(cast, ',', '')) + 1 AS total_cast_members,
        1 AS current_cast_member
    FROM netflix
    WHERE country = 'India'

    UNION ALL

    SELECT 
        show_id,
        full_cast,
        SUBSTRING_INDEX(SUBSTRING_INDEX(full_cast, ',', current_cast_member + 1), ',', -1) AS single_cast,
        total_cast_members,
        current_cast_member + 1
    FROM cast_split
    WHERE current_cast_member < total_cast_members
)
SELECT 
    TRIM(single_cast) AS actor, 
    COUNT(*) AS appearances
FROM cast_split
WHERE 
	TRIM(single_cast)<>'NULL'
GROUP BY actor
ORDER BY appearances DESC
LIMIT 10;
```

**Objective:** Identify the top 10 actors with the most appearances in Indian-produced movies.

### 15. Categorize Content Based on the Presence of 'Kill' and 'Violence' Keywords

```sql
select 
	content,
    count(*) 
from (
	select 
		description,
		case when description like '%kill%' or description like '%violence%' then 'Bad'
		else 'Good' end as content
	from 
		netflix) a1
group by 
	content; 
```

**Objective:** Categorize content as 'Bad' if it contains 'kill' or 'violence' and 'Good' otherwise. Count the number of items in each category.

## Findings and Conclusion

- **Content Distribution:** The dataset contains a diverse range of movies and TV shows with varying ratings and genres.
- **Common Ratings:** Insights into the most common ratings provide an understanding of the content's target audience.
- **Geographical Insights:** The top countries and the average content releases by India highlight regional content distribution.
- **Content Categorization:** Categorizing content based on specific keywords helps in understanding the nature of content available on Netflix.

This analysis provides a comprehensive view of Netflix's content and can help inform content strategy and decision-making.
