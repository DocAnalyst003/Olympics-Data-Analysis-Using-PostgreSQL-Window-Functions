# 📊 Olympics Data Analysis Using PostgreSQL Summary Stats And Window Functions
This project explores advanced SQL analytics using PostgreSQL window functions on a real-world Olympics dataset. It focuses on answering time-based and comparative questions that go beyond standard GROUP BY queries, such as rankings, running totals, and performance trends.

![Ru2f3](https://github.com/user-attachments/assets/e859dcd7-2ee4-4f8c-ab31-569a560ed816)


## Introduction

As someone building a career in data analytics and data engineering, I kept running into a wall. I can easily write standard SQL queries, GROUP BY data, and pull basic summaries, but I couldn't answer the more interesting questions for an Olympic Dataset: 

1. Who is the reigning champion right now?
2. What's the running total of medals won so far?
3. How does this year's performance compare to the last three years?

These questions require you to look across rows in a meaningful, structured way, and that's exactly what window functions do.

Querying the Summer Olympics dataset (each row representing an awarded medal with columns for year, city, sport, discipline, event, athlete, country, gender, and medal type), I worked through the window functions, covering everything from row numbering and rankings to moving averages and advanced pivoting techniques.

## Questions I Explored

1. How do window functions differ from GROUP BY, and when should each be used?
2. How can I fetch values from other rows, past or future, without complex self-joins?
3. How do I rank rows in a dataset, including handling ties correctly?
4. How do I split data into equal pages or performance tiers (top/middle/bottom thirds)?
5. How do aggregate functions like SUM and AVG behave as window functions, and what are frames?
6. How do moving averages and moving totals work and what do they reveal that standard aggregations miss?
7. How can I pivot, roll up, and compress query results for cleaner reporting?

## SQL Skills Used

The following PostgreSQL skills were applied throughout:

- 🪟 Window Functions with `OVER()`
- 📋 `ROW_NUMBER()`, `RANK()`, `DENSE_RANK()`
- ⬆️⬇️ `LAG()`, `LEAD()`, `FIRST_VALUE()`, `LAST_VALUE()`
- 📦 `NTILE()` for paging / binning
- ➕ Aggregate window functions: `SUM()`, `AVG()`, `MAX()`, `MIN()`
- 🖼️ Frame clauses: `ROWS BETWEEN`
- 🔄 `CROSSTAB()` pivoting via `tablefunc` extension
- 🎲 `ROLLUP` and `CUBE` for group-level subtotals
- 🧹 `COALESCE()` for null handling
- 🔗 `STRING_AGG()` for compressing multi-row results into one
- 🧮 CTEs (Common Table Expressions)

---

## The Analysis

### Introduction to Window Functions

Window functions perform operations across a set of rows related to the current row without collapsing results into a single row, the way GROUP BY does. The anatomy of a window function is:

```sql
function_name()[e.g MAX,MIN,SUM,RANK] OVER (PARTITION BY col ORDER BY col ROWS BETWEEN ...)
```

The `OVER` clause is what makes it a window function. Without `ORDER BY` or `PARTITION BY` inside it, the function operates over the entire table.

**1. ROW_NUMBER and ORDER BY inside OVER**

The most basic window function is `ROW_NUMBER()`. I used it to number the Olympic Games in ascending and then descending order, and to number athletes by how many medals they earned:

A. Numbering Olympic Games in Ascending Order(Oldest to Most Recent)
```sql
-- Numbering Olympic games in ascending order
SELECT
  Year,
-- Assign numbers to each year
  ROW_NUMBER() OVER (ORDER BY Year ASC) AS Row_N
FROM (
  SELECT DISTINCT Year FROM Summer_Medals
) AS Years
ORDER BY Year ASC;
```
Below is the SQL output that I got from the query;
<img width="937" height="595" alt="games_num" src="https://github.com/user-attachments/assets/4f16fbab-9ff7-4137-ab85-f1f795c698cb" />

B. Numbering Olympic Games starting from Most Recent Year to Oldest 
```sql
-- Numbering Olympic games in descending order
SELECT
  Year,
-- Assigning the LOwest Numbers to the Most recent Years
  ROW_NUMBER() OVER (ORDER BY Year DESC) AS Row_N
FROM (
  SELECT DISTINCT Year FROM Summer_Medals
) AS Years
ORDER BY Year;
```
Below is the SQL output that I got from the query above; 
<img width="937" height="605" alt="games_num_desc" src="https://github.com/user-attachments/assets/320067c2-32bf-43a7-9661-887398eb0fb5" />

C. Numbering Each Athlete by how many medals they earned. 
```sql
WITH Athlete_Medals AS (
  SELECT
    -- Count the number of medals each athlete has earned
    Athlete,
    COUNT(*) AS Medals
  FROM Summer_Medals
  GROUP BY Athlete)

SELECT
  -- Number each athlete by how many medals they've earned
  athlete,
  Medals,
  ROW_NUMBER() OVER (ORDER BY Medals DESC) AS Row_N
FROM Athlete_Medals
ORDER BY Medals DESC;
```
Below is the Output I got from the above query;
<img width="939" height="593" alt="athlete_med_num" src="https://github.com/user-attachments/assets/50bbc77b-ab24-425a-8cd7-75eca20e1690" />

  Conclusion: Phelps Micheal had the most medals among athletes by 2012(22 Medals). 

**2. Use of ORDER BY and LAG to Identify Reigning Champions**

Using `LAG()` with `ORDER BY` inside `OVER`, I identified reigning Weightlifting champions — those who won both the previous and current Olympics:

```sql
WITH Weightlifting_Gold AS (
  SELECT    -- Return each year's champions' country
    Year,
    Country AS Champion
  FROM Summer_Medals
  WHERE
    Discipline = 'Weightlifting'
    AND Event = '69KG'
    AND Gender = 'Men'
    AND Medal = 'Gold'
)

SELECT
  Year,
  Champion,    -- fetch previous year's champion
  LAG(Champion) OVER (ORDER BY Year ASC) AS Last_Champion
FROM Weightlifting_Gold
ORDER BY Year ASC;
```
<img width="941" height="447" alt="reign_champs" src="https://github.com/user-attachments/assets/4ce9048c-93f2-4fae-8dd1-095424ba825e" />


**PARTITION BY — Separating Groups**

`PARTITION BY` splits the table into partitions so that window functions operate independently **per group**. Without it, `LAG` would incorrectly pull the last row of a previous event into the first row of the next event.

To visualize the reigning Javelin Throw champions over the Olympic Years per Gender, I added the partition portion as shown in the SQL code below; 
```sql
-- Reigning champions by gender — partitioned correctly
WITH Athletics_Gold AS (
  SELECT
    Gender,
    Year,
    Country AS Champion
  FROM Summer_Medals
  WHERE
    Year IN (2000, 2004, 2008, 2012)
    AND Discipline = 'Athletics'
    AND Event = '200M'
    AND Medal = 'Gold'
)

SELECT
  Gender,
  Year,
  Champion,
  LAG(Champion) OVER (PARTITION BY Gender ORDER BY Year ASC) AS Last_Champion
FROM Athletics_Gold
ORDER BY Gender ASC, Year ASC;
```
Below is the output obtained from this; 
<img width="945" height="587" alt="reign_champs_gend" src="https://github.com/user-attachments/assets/89f35066-f84d-4953-99e2-5c2b728bf011" />

I decided to take this further to see the reigning champions per gender in 2 particular events, 100M and 10000M. To do this, I came up with the query shown below; 

```sql
WITH Athletics_Gold AS (
  SELECT DISTINCT
    Gender, Year, Event, Country
  FROM Summer_Medals
  WHERE
    Year >= 2000 AND
    Discipline = 'Athletics' AND
    Event IN ('100M', '10000M') AND
    Medal = 'Gold')

SELECT
  Gender, Year, Event,
  Country AS Champion,
  -- Fetch the previous year's champion by gender and event
  LAG(Country) OVER (PARTITION BY gender, event
            ORDER BY Year ASC) AS Last_Champion
FROM Athletics_Gold
ORDER BY Event ASC, Gender ASC, Year ASC;
```
The Output I got is shown below;
<img width="939" height="607" alt="reign_champs_gend_eve" src="https://github.com/user-attachments/assets/a7dfba42-b103-40df-ac26-1ed2ce4bad49" />

---

### Chapter 2: Fetching, Ranking, and Paging



**Fetching with LAG, LEAD, FIRST_VALUE, LAST_VALUE**

| Function | Type | What It Returns |
|---|---|---|
| `LAG(col, n)` | Relative | Value n rows **before** current row | i.e Past values
| `LEAD(col, n)` | Relative | Value n rows **after** current row | i.e Future values
| `FIRST_VALUE(col)` | Absolute | First value in table/partition |
| `LAST_VALUE(col)` | Absolute | Last value in table/partition (needs `RANGE BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING`) |

To practice the use of LEAD to get future values, I set myself a challenge and used the SQL code shown below to identify Future Medalists at least 3 years from the current Medalists per row; 
```sql
WITH Discus_Medalists AS (
  SELECT DISTINCT
    Year,
    Athlete
  FROM Summer_Medals
  WHERE Medal = 'Gold'
    AND Event = 'Discus Throw'
    AND Gender = 'Women'
    AND Year >= 2000)

SELECT
  -- For each year, fetch the current and future medalists
  year,
  Athlete,
  LEAD(Athlete, 
  3) OVER (ORDER BY year ASC) AS Future_Champion
FROM Discus_Medalists
ORDER BY Year ASC;
```
This is the output I got from the above code; 
<img width="937" height="553" alt="future_champs" src="https://github.com/user-attachments/assets/72b3159e-c7e9-4031-8728-b920e4371dc1" />
    Conclusion; As expected, only 2000 year had a value in the future champion mas the rest had no champion 3 years into their future.

I wanted to fetch all male medalists and also fetch the name of the first male medalist. To do this, I used the SQL code shown below; 

```sql
WITH All_Male_Medalists AS (
  SELECT DISTINCT
    Athlete
  FROM Summer_Medals
  WHERE Medal = 'Gold'
    AND Gender = 'Men')

SELECT
  -- Fetch all athletes and the first athlete alphabetically
  Athlete,
  FIRST_VALUE(Athlete) OVER (
    ORDER BY Athlete ASC
  ) AS First_Athlete
FROM All_Male_Medalists;
```
This is the output I got from the above code; 
<img width="941" height="585" alt="all_name_vs_first" src="https://github.com/user-attachments/assets/faaf9270-c565-4cba-89f7-283ca62114a9" />



**Ranking with ROW_NUMBER, RANK, DENSE_RANK**

The three ranking functions handle ties differently:

| Function | Handles Ties | Skips Ranks After Ties? |
|---|---|---|
| `ROW_NUMBER()` | No — assigns unique numbers | N/A |
| `RANK()` | Yes — same rank for ties | Yes — skips next rank(s) |
| `DENSE_RANK()` | Yes — same rank for ties | No — never skips ranks |

I wanted to rank all Athletes and rank them based on the number of medals each has. To do this, I used the code shown below; 

```sql
WITH Athlete_Medals AS (
  SELECT
    Athlete,
    COUNT(*) AS Medals
  FROM Summer_Medals
  GROUP BY Athlete)

SELECT
  Athlete,
  Medals,
  -- Rank athletes by the medals they've won
  RANK() OVER (ORDER BY Medals DESC) AS Rank_N
FROM Athlete_Medals
ORDER BY Medals DESC;
```
This is the output I got from the above code; 
<img width="943" height="595" alt="athlete_medals_rank" src="https://github.com/user-attachments/assets/d475bff2-38ce-4a4f-a3cc-be955791ecde" />
    Conclusion; Phelps Micheal has the highest number of medals (22) and got the rank 1. However, I noted that this form of ranking assigned people with the same number of medals the same rank i.e Ano Takashi, Mangiarotti Edoardo, and Shakhlin Boris. 

To further challenge myself, I wanted to rank athletes in Korea and Japan who had atleast one medal. Below is the code I used for that; 

```sql
WITH Athlete_Medals AS (
  SELECT
    Country, Athlete, COUNT(*) AS Medals
  FROM Summer_Medals
  WHERE
    Country IN ('JPN', 'KOR')
    AND Year >= 2000
  GROUP BY Country, Athlete
  HAVING COUNT(*) > 1)

SELECT
  Country,
  -- Rank athletes in each country by the medals they've won
  Athlete,
  DENSE_RANK() OVER (PARTITION BY Country
                ORDER BY Medals DESC) AS Rank_N
FROM Athlete_Medals
ORDER BY Country ASC, RANK_N ASC;
```
Below is the output I got from the above SQL code; 
<img width="939" height="539" alt="athlete_medals_countr_rank" src="https://github.com/user-attachments/assets/87d22304-232c-4f71-a263-d1e8e23cec35" />
Conclusion: Japanese KItajima Kosuke has the highest number of medals in Japan and Korea at 7 medals. 

**Paging with NTILE**

`NTILE(n)` splits data into n approximately equal pages — useful for API pagination and for labeling data as top/middle/bottom performers:

```sql
-WITH Events AS (
  SELECT DISTINCT Event
  FROM Summer_Medals)
  
SELECT
  --- Split up the distinct events into 111 unique groups
  Event,
  NTILE(111) OVER (ORDER BY Event ASC) AS Page
FROM Events
ORDER BY Event ASC;
```
Below is the output I got from the above SQL code; 
<img width="943" height="603" alt="paging_events" src="https://github.com/user-attachments/assets/d56358dd-132d-4353-ae5d-2cb15dcd2f76" />

To further challenge myself, I wanted to separate and get particular averages for thirds in my dataset that has 666 records. I used the query below to do this; 

```sql
WITH Athlete_Medals AS (
  SELECT Athlete, COUNT(*) AS Medals
  FROM Summer_Medals
  GROUP BY Athlete
  HAVING COUNT(*) > 1),
  
  Thirds AS (
  SELECT
    Athlete,
    Medals,
    NTILE(3) OVER (ORDER BY Medals DESC) AS Third
  FROM Athlete_Medals)
  
SELECT
  -- Get the average medals earned in each third
  Third,
  AVG(Medals) AS Avg_Medals
FROM Thirds
GROUP BY Third
ORDER BY Third ASC;
```
Below is the output I got from the above query; 
<img width="943" height="431" alt="thirds_avg" src="https://github.com/user-attachments/assets/61a287de-c4f3-4f1a-ba0d-17fc140599c0" />

---

### Chapter 3: Aggregate Window Functions and Frames

Aggregate functions like `SUM`, `AVG`, `MAX`, `MIN`, and `COUNT` can be used as window functions — unlocking running totals, moving averages, and cumulative maximums that simply aren't possible with `GROUP BY` alone.

**Aggregate Window Functions**

I wanted to calculate running total USA Gold medals in competitions held after 2000. I used the code below to do this;

```sql
WITH Athlete_Medals AS (
  SELECT
    Athlete, COUNT(*) AS Medals
  FROM Summer_Medals
  WHERE
    Country = 'USA' AND Medal = 'Gold'
    AND Year >= 2000
  GROUP BY Athlete)

SELECT
  -- Calculate the running total of athlete medals
  Athlete,
  Medals,
  SUM(Medals) OVER (ORDER BY Athlete ASC) AS Max_Medals
FROM Athlete_Medals
ORDER BY Athlete ASC;
```
Below is the output I got from the above query; 
<img width="943" height="605" alt="usa-runn_tot_gold" src="https://github.com/user-attachments/assets/1b09f15d-aa09-455a-881a-2f27ed0a6d89" />

To further challenge myself, I decided to explore and know the max medals China, Korea, and Japan have individually got over the Olympic years. To do this, I used the code below; 

```sql
WITH Country_Medals AS (
  SELECT
    Year, Country, COUNT(*) AS Medals
  FROM Summer_Medals
  WHERE
    Country IN ('CHN', 'KOR', 'JPN')
    AND Medal = 'Gold' AND Year >= 2000
  GROUP BY Year, Country)

SELECT
  -- Return the max medals earned so far per country
  Year,
  Country,
  Medals,
  MAX(Medals) OVER (PARTITION BY Country
                ORDER BY Year ASC) AS Max_Medals
FROM Country_Medals
ORDER BY Country ASC, Year ASC;
```
This is the output I got from the query above;
<img width="941" height="595" alt="max_medals_countr" src="https://github.com/user-attachments/assets/9fd93138-720c-4afe-a229-178b87310af1" />


**Frames — ROWS BETWEEN**

A frame defines which rows the window function looks at relative to the current row. Syntax:

```
ROWS BETWEEN [start] AND [end]
```

Where start/end can be: `n PRECEDING`, `CURRENT ROW`, or `n FOLLOWING`.
I wanted to get the max number of current and next year's Goldmedals among Scandinavian Countries. I used the below query to do this; 

```sql
WITH Scandinavian_Medals AS (
  SELECT
    Year, COUNT(*) AS Medals
  FROM Summer_Medals
  WHERE
    Country IN ('DEN', 'NOR', 'FIN', 'SWE', 'ISL')
    AND Medal = 'Gold'
  GROUP BY Year)

SELECT
  -- Select each year's medals
  year,
  Medals,
  -- Get the max of the current and next years'  medals
  MAX(Medals) OVER (ORDER BY year ASC
             ROWS BETWEEN CURRENT ROW
             AND 1 FOLLOWING) AS Max_Medals
FROM Scandinavian_Medals
ORDER BY Year ASC;
```
Below is the output I got; 
<img width="937" height="597" alt="max_medals_scandinav" src="https://github.com/user-attachments/assets/eaf1d838-06cc-4b19-961f-c6bdaee31db5" />

**Moving Averages and Moving Totals**

A **moving average** smooths out fluctuations and reveals trends. A **moving total** tracks cumulative recent performance. Both use `ROWS BETWEEN`:

I wanted to identify the moving Average of Russian Gold Medals in years after 1986 Olympic year. This is the query I wrote to visualize this; 

```sql
-- 3-game moving average of US gold medals (last 2 + current game)
WITH Russian_Medals AS (
  SELECT
    Year, COUNT(*) AS Medals
  FROM Summer_Medals
  WHERE
    Country = 'RUS'
    AND Medal = 'Gold'
    AND Year >= 1980
  GROUP BY Year)

SELECT
  Year, Medals,
  --- Calculate the 3-year moving average of medals earned
  AVG(Medals) OVER
    (ORDER BY Year ASC
     ROWS BETWEEN
     2 PRECEDING AND CURRENT ROW) AS Medals_MA
FROM Russian_Medals
ORDER BY Year ASC;
```
Below is the output I got from the above query; 
<img width="939" height="553" alt="russ_mov_gold_avg" src="https://github.com/user-attachments/assets/e6839499-4b6f-4c1b-b620-a47ad7725be3" />

**ROWS vs RANGE:** `ROWS BETWEEN` treats each row individually. `RANGE BETWEEN` treats rows with the same ORDER BY value as a single entity. In practice, `ROWS BETWEEN` is almost always the right choice.

I wanted to check every countries 3-Olympic game mocing total medals. To do this, I used the query below; 

```sql
WITH Country_Medals AS (
  SELECT
    Year, Country, COUNT(*) AS Medals
  FROM Summer_Medals
  GROUP BY Year, Country)

SELECT
  Year, Country, Medals,
  -- Calculate each country's 3-game moving total
  SUM(Medals) OVER
    (PARTITION BY Country
     ORDER BY Year ASC
     ROWS BETWEEN
     2 PRECEDING AND CURRENT ROW) AS Medals_MA
FROM Country_Medals
ORDER BY Country ASC, Year ASC;
```
This is the output I got from the above query; 
<img width="935" height="597" alt="country_mov_total" src="https://github.com/user-attachments/assets/33a782b6-2ed7-4910-97a3-82481e4fb045" />

---

### Chapter 4: Beyond Window Functions

These are Techniques that supercharge window functions — pivoting tables with CROSSTAB, generating group-level subtotals with ROLLUP and CUBE, and cleaning up null results with COALESCE and STRING_AGG.

**Pivoting with CROSSTAB**

Pivoting transforms a table by making a column's unique values into new columns — making results easier to scan, especially for reporting and dashboards.

I wanted to visualize male and female gold medalists in the Pole Vault Event in the 2008 and 2012 Olympic years. However, I wanted it to be displayed in an easy to see and understand output. To do this, I used pivoting as shown in the query below; 

```sql
-- Create the correct extension to enable CROSSTAB
CREATE EXTENSION IF NOT EXISTS tablefunc;

SELECT * FROM CROSSTAB($$
  SELECT
    Gender, Year, Country
  FROM Summer_Medals
  WHERE
    Year IN (2008, 2012)
    AND Medal = 'Gold'
    AND Event = 'Pole Vault'
  ORDER By Gender ASC, Year ASC;
-- Fill in the correct column names for the pivoted table
$$) AS ct (Gender VARCHAR,
           "2008" VARCHAR,
           "2012" VARCHAR)

ORDER BY Gender ASC;
```
Below is the output I got; 
<img width="937" height="255" alt="basic_pivot" src="https://github.com/user-attachments/assets/1cdc104f-fe75-4f7d-9c91-0f0dce5aee39" />

To further challenge myself, I wanted to visualize the number of Gold awards won by France and Great Britain in 2004, 2008, and 2012. To do this, I used the query below; 

```sql
CREATE EXTENSION IF NOT EXISTS tablefunc;

SELECT * FROM CROSSTAB($$
  WITH Country_Awards AS (
    SELECT
      Country,
      Year,
      COUNT(*) AS Awards
    FROM Summer_Medals
    WHERE
      Country IN ('FRA', 'GBR', 'GER')
      AND Year IN (2004, 2008, 2012)
      AND Medal = 'Gold'
    GROUP BY Country, Year)

  SELECT
    Country,
    Year,
    RANK() OVER
      (PARTITION BY Year
       ORDER BY Awards DESC) :: INTEGER AS rank
  FROM Country_Awards
  ORDER BY Country ASC, Year ASC;
-- Fill in the correct column names for the pivoted table
$$) AS ct (Country VARCHAR,
           "2004" INTEGER,
           "2008" INTEGER,
           "2012" INTEGER)

Order by Country ASC;
```
Below is the output I got from the above query; 
<img width="941" height="327" alt="country_award_perf" src="https://github.com/user-attachments/assets/82674a35-46f6-45bc-9a10-7981c038304f" />


**ROLLUP and CUBE — Group-Level Subtotals**

| Subclause | What It Generates |
|---|---|
| `ROLLUP` | Hierarchical subtotals (e.g., Country-level totals, then grand total) |
| `CUBE` | All possible group-level subtotals (every combination) |

I wanted to see the total Gold Awards won by different geders in Denmark, Norway, and Sweden while also having a country subtotal of both genders. To do this, I used the SQL query shown below;

```sql
-- Count the gold medals per country and gender
SELECT
  Country,
  Gender,
  COUNT(*) AS Gold_Awards
FROM Summer_Medals
WHERE
  Year = 2004
  AND Medal = 'Gold'
  AND Country IN ('DEN', 'NOR', 'SWE')
-- Generate Country-level subtotals
GROUP BY Country, ROLLUP(Gender)
ORDER BY Country ASC, Gender ASC;
```
This is the output shown below; 
<img width="937" height="607" alt="gender_perf_country" src="https://github.com/user-attachments/assets/f4f2b7ff-8180-4eb4-bcfa-232f97a4e563" />

From this, I noted that the row which contained a null value in the gender column was the row showing total medals won by a country. 


I wanted to know how each gender womn the bronze, silver and gold awards while also knowing the total awards won. To do this, I used the query shown below; 

```sql
-- Count the medals per gender and medal type
SELECT
  gender,
  medal,
  Count(*) AS Awards
FROM Summer_Medals
WHERE
  Year = 2012
  AND Country = 'RUS'
-- Get all possible group-level subtotals
GROUP BY CUBE(Gender,Medal)
ORDER BY Gender ASC, Medal ASC;
```
Below is the output I got from the above query; 
<img width="921" height="605" alt="award_perf_gender" src="https://github.com/user-attachments/assets/e88f138d-eedf-4095-be7c-fcdb6e725c67" />

**COALESCE — Replacing NULLs**

ROLLUP and CUBE produce nulls for subtotal rows. `COALESCE` replaces them with meaningful labels:

I wanted to know how men and women performed in Denmark, Norway, and Sweden and how many gold medals they won. To do this, I used the query shown below; 

```sql
SELECT
  -- Replace the nulls in the columns with meaningful text
  COALESCE(Country, 'All countries') AS Country,
  COALESCE(Gender, 'All genders') AS Gender,
  COUNT(*) AS Awards
FROM Summer_Medals
WHERE
  Year = 2004
  AND Medal = 'Gold'
  AND Country IN ('DEN', 'NOR', 'SWE')
GROUP BY ROLLUP(Country, Gender)
ORDER BY Country ASC, Gender ASC;
```
This is the output I got from the query above; 
<img width="943" height="605" alt="award_perf_gender_nor" src="https://github.com/user-attachments/assets/32641346-b398-4b2c-992b-6b1e0b779707" />

This method replaces the null values with words that describe the actual result. 

**STRING_AGG — Compressing Rows into One**

When rankings make the rank column redundant, `STRING_AGG` collapses results into a single readable row:

```sql
WITH Country_Medals AS (
  SELECT
    Country,
    COUNT(*) AS Medals
  FROM Summer_Medals
  WHERE Year = 2000
    AND Medal = 'Gold'
  GROUP BY Country),

  Country_Ranks AS (
  SELECT
    Country,
    RANK() OVER (ORDER BY Medals DESC) AS Rank
  FROM Country_Medals
  ORDER BY Rank ASC)

-- Compress the countries column
SELECT STRING_AGG(Country, ', ')
FROM Country_Ranks
-- Select only the top three ranks
WHERE Rank<=3;
```
Below is the output I got from the above query; 
<img width="939" height="181" alt="string_agg" src="https://github.com/user-attachments/assets/7c457643-d4c8-40ec-85e3-cf0232bdd8e6" />

---

## Key Takeaways

Window functions are genuinely one of the most important tools in SQL. Here's what they unlock that standard aggregations simply cannot:

**For analysts:** Running totals, moving averages, and period-over-period comparisons — without writing complex subqueries or self-joins. Things like "what's this month's sales vs last month's?" are one window function away.

**For data engineers:** Efficient ranking and deduplication. Instead of `GROUP BY` plus joins, a single `ROW_NUMBER() OVER (PARTITION BY ... ORDER BY ...)` gives you the latest record per group.

**For reporting:** CROSSTAB pivoting, ROLLUP/CUBE subtotals, and COALESCE together transform raw query results into presentation-ready tables — the kind you'd hand to a business stakeholder.

**For society and decision-making:** Time-series analysis using moving averages can track trends in public health data, economic indicators, and social metrics — giving policymakers the "signal" behind the "noise" of raw numbers.

---

