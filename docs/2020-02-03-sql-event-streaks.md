---
layout: page
title: "SQL Story of unbroken chains of events (daily/weekly streaks)"
permalink: /sql/
---

Recently I decided to write my own habit tracker. 
I'm planning to have 2 kinds of habits: everyday habits and every-week habits. Plus in order to accomplish some of them you must check them several times per period. To track it and prevent me from quitting my new habits we'll calculate unbroken chains of events in 3 parts:
* [Everyday habit streak](#daily)
* [Every week habit streak](#weekly)
* [With postgreSQL queries](#postgreSQL)

### 1. Everyday habit streak <a name="daily"></a>
For a start let's calculate consecutive series of events that should happen every day.
For the demonstration purpose let's use data dumps on [Stack Exchange Data Explorer](https://data.stackexchange.com/). We will use a post table and try to find streaks of post creation. You can find the schema description [here](https://meta.stackexchange.com/questions/2677/database-schema-documentation-for-the-public-data-dump-and-sede). It uses SQL Server, but next, we'll rewrite it for PostgresSQL.
First, let's check what do we have for user 1.
```sql
SELECT 
  ROW_NUMBER() OVER (ORDER BY CreationDate) rowNumber,
  CAST(CreationDate AS DATE) date
FROM Posts
WHERE OwnerUserId = ##UserId##
ORDER BY CreationDate
```
([you can run this snippet here](https://data.stackexchange.com/stackoverflow/query/1190826/show-me-your-data?UserId=1) and check all 144 rows of data)
We can see lots of posts created during the first days. And we can find 5 day streak here:
```
rowNumber   date
-----------------------
1   2008-07-31 00:00:00
2   2008-07-31 00:00:00
3   2008-07-31 00:00:00
4   2008-08-04 00:00:00
5   2008-08-04 00:00:00
6   2008-08-04 00:00:00

7   2008-08-10 00:00:00 ----
8   2008-08-11 00:00:00     |
9   2008-08-12 00:00:00     |
10  2008-08-12 00:00:00     |
11  2008-08-12 00:00:00     |
12  2008-08-12 00:00:00     |5 day
13  2008-08-12 00:00:00     |streak
14  2008-08-13 00:00:00     |found  
15  2008-08-13 00:00:00     | 
16  2008-08-13 00:00:00     |
17  2008-08-14 00:00:00     |
18  2008-08-14 00:00:00     |
19  2008-08-14 00:00:00     |
20  2008-08-14 00:00:00 ----

21  2008-08-17 00:00:00
...
```
What is `ROW_NUMBER()` function in the snippet above? It's called [window function](https://www.postgresql.org/docs/8.4/functions-window.html), it allows to make calculations across all rows of the current query.

Before using it in our calculations let's resolve another problem. 
In our 5 days streak we can see a lot of repeating days, which gives us nothing new, let's group it and calculate the number of posts for these specific days - column `amount`.
```sql
SELECT 
  ROW_NUMBER() OVER (ORDER BY MIN(CreationDate)) rowNumber,
  COUNT(*) amount,
  CAST(MIN(CreationDate)AS DATE) date
FROM Posts
WHERE OwnerUserId = ##UserId##
  AND CreationDate BETWEEN '2008-08-10' and '2008-08-17'
GROUP BY CAST(CreationDate AS DATE)
ORDER BY MIN(CreationDate)
```    
([run it!](https://data.stackexchange.com/stackoverflow/query/1190898/amount-of-posts-per-day?UserId=1))
```
r amount  date
---------------------------
1   1   2008-08-10 00:00:00 ----
2   1   2008-08-11 00:00:00     |our
3   5   2008-08-12 00:00:00     |5 days
4   3   2008-08-13 00:00:00     |streak
5   4   2008-08-14 00:00:00 ----
```
Now we can see a pattern: day 5 of our streak here is also the 5th row. 
Let's use this sweet `ROW_NUMBER()` function and subtract row number from our date.
```sql
SELECT 
  ROW_NUMBER() OVER (ORDER BY MIN(CreationDate)) row,
  COUNT(*) amount,
  CAST(MIN(CreationDate)AS DATE) date,
  CAST(MIN(CreationDate) - ROW_NUMBER() OVER (ORDER BY MIN(CreationDate)) as DATE) dateMinusRow
FROM Posts
WHERE OwnerUserId = ##UserId##
  AND CreationDate BETWEEN '2008-08-10' and '2008-08-17'
GROUP BY CAST(CreationDate AS DATE)
ORDER BY MIN(CreationDate)
```
([run it here](https://data.stackexchange.com/stackoverflow/query/1192171/dates-minus-rownumbers-for-specific-dates?UserId=1))

```
r   amount          date                dateMinusRow
-------------------------------------------------------
1   1       2008-08-10 00:00:00     2008-08-09 00:00:00 --
2   1       2008-08-11 00:00:00     2008-08-09 00:00:00    |same value
3   5       2008-08-12 00:00:00     2008-08-09 00:00:00    |because of
4   3       2008-08-13 00:00:00     2008-08-09 00:00:00    |(date - row_number)
5   4       2008-08-14 00:00:00     2008-08-09 00:00:00 --
```
And apply this logic to the whole set:
```sql
SELECT 
  ROW_NUMBER() OVER (ORDER BY MIN(CreationDate)) as DATE) row,
  COUNT(*) amount,
  CAST(MIN(CreationDate)AS DATE) date,
  CAST(MIN(CreationDate) - ROW_NUMBER() OVER (ORDER BY MIN(CreationDate)) as DATE) dateMinusRow
FROM Posts
WHERE OwnerUserId = ##UserId##
GROUP BY CAST(CreationDate AS DATE)
ORDER BY MIN(CreationDate)
```
([run me](https://data.stackexchange.com/stackoverflow/query/1190901/dates-minus-rownumbers?UserId=1))

```
r   amount      date             dateMinusRow
------------------------------------------------------
1    3       2008-07-31 00:00:00   2008-07-30 00:00:00
2    3       2008-08-04 00:00:00   2008-08-02 00:00:00
            
3    1       2008-08-10 00:00:00   2008-08-07 00:00:00-- 
4    1       2008-08-11 00:00:00   2008-08-07 00:00:00  |same value
5    5       2008-08-12 00:00:00   2008-08-07 00:00:00  |because of
6    3       2008-08-13 00:00:00   2008-08-07 00:00:00  |(date - row_number)
7    4       2008-08-14 00:00:00   2008-08-07 00:00:00--
            
8    2       2008-08-17 00:00:00   2008-08-09 00:00:00
```
As we can see it works perfectly, column `dateMinusRow` has the same value for all streak days. 
All we need to do now is group the values and enjoy the result.
```sql
SELECT
  COUNT(*) streak,
  SUM(amount) streakAmount,
  MIN(date) startDate,
  MAX(date) endDate,
  dateMinusRow dateMinusRow
FROM (
  SELECT 
    COUNT(*) amount,
    CAST(MIN(CreationDate)AS DATE) date,
    CAST(MIN(CreationDate) - ROW_NUMBER() OVER (ORDER BY MIN(CreationDate)) as DATE) dateMinusRow
  FROM Posts
  WHERE OwnerUserId = ##UserId##
  GROUP BY CAST(CreationDate AS DATE)
) groupedDays
GROUP BY dateMinusRow
```
And result was as we expected ([run it here and check all records](https://data.stackexchange.com/stackoverflow/query/1190903/streak-for-days?UserId=1))

```
streak          start                  endDate
---------------------------------------------------
1       2008-07-31 00:00:00     2008-07-31 00:00:00 
1       2008-08-04 00:00:00     2008-08-04 00:00:00 
5       2008-08-10 00:00:00     2008-08-14 00:00:00 - our 5 days streak
1       2008-08-17 00:00:00     2008-08-17 00:00:00 
```
We can work with this query further and find out what is our user's max daily streak ([wasn't it 5?](https://data.stackexchange.com/stackoverflow/query/1190909/max-day-streak?UserId=1)). But now let's talk about weeks and use this `ROW_NUMBER()` method there.

### 2. Every week habit streak <a name="weekly"></a>
To use the same calculation method we need to understand what can represent all events during the same week.

First, we can use `DATEPART()` to get a week number and year from the date. 
In this case, we'll have to get our head around calculations when streak goes from one year to another.

Another way to solve the week streak case is to use the difference in weeks between our CreationDate and some start date. 
We'll use `0` as a start date. Sql Server interprets it like `1900-01-01 00:00:00.000`. 
Also by default Sql Server uses Sunday as the first day of the week: 
```sql
SELECT 
  COUNT(*) anount,
  MIN(p.CreationDate) minDate,
  MAX(p.CreationDate) maxDate,
  DATEDIFF(wk, 0, CreationDate) weeks,
  DATEDIFF(wk, 0, CreationDate) - ROW_NUMBER() OVER (ORDER BY MIN(CreationDate)) weekMinusRow
FROM Posts p
WHERE OwnerUserId = ##UserId##
GROUP BY DATEDIFF(wk, 0, CreationDate)
``` 
In my case, I prefer to use Monday as the first day. Even though `DATEDIFF` ignores `SET DATEFIRST` there is a hint from [here](https://social.msdn.microsoft.com/Forums/sqlserver/en-US/8cc3493a-7ae5-4759-ab2a-e7683165320b/problem-with-datediff-and-datefirst?forum=transactsql).

([run version with Monday as a first day](https://data.stackexchange.com/stackoverflow/query/1190918/base-week-streak-query?UserId=1))

So our final query:
```sql
SET DATEFIRST 1
SELECT
  COUNT(*) streak,
  MIN(minDate) startDate,
  MAX(maxDate) endDate
FROM (
  SELECT 
    COUNT(*) counts,
    MIN(p.CreationDate) minDate,
    MAX(p.CreationDate) maxDate,
    DATEDIFF(wk, DATEADD(dd,-@@datefirst,0), DATEADD(dd,-@@datefirst,CreationDate)) weeks,
    DATEDIFF(wk, DATEADD(dd,-@@datefirst,0), DATEADD(dd,-@@datefirst,CreationDate)) - ROW_NUMBER() OVER (ORDER BY MIN(CreationDate)) weekMinusRow
  FROM Posts p
  WHERE OwnerUserId = ##UserId##
  GROUP BY DATEDIFF(wk, DATEADD(dd,-@@datefirst,0), DATEADD(dd,-@@datefirst,CreationDate))
  HAVING COUNT(*) >= ##numberOfRepetitions##
) groupedRows
GROUP BY weekMinusRow
ORDER BY endDate DESC
```
([run it](https://data.stackexchange.com/stackoverflow/query/1190747/week-streaks?numberOfRepetitions=4&UserId=1))

```
streak      startDate                  endDate
------------------------------------------------------
1        2009-03-30 08:00:18       2009-04-03 10:35:03
1        2009-03-10 01:05:12       2009-03-14 17:45:04
1        2008-12-01 04:56:38       2008-12-05 00:34:40
3        2008-10-07 08:01:40       2008-10-24 15:45:15
1        2008-08-25 00:11:43       2008-08-28 13:26:27
2        2008-08-04 02:45:07       2008-08-17 03:00:34
```

By the way, I also added `HAVING COUNT(*) >= ##numberOfRepetitions##` to filter by the number of actions that were performed per week. 
From now on for streak week we should post harder, at least 4 times a week ;-)   

### 3. With PostgreSQL queries <a name="postgreSQL"></a>
Data dumps on [Stack Exchange Data Explorer](https://data.stackexchange.com/) are amazing and a lot of fun to play with. But in my current project, I use PostgreSQL. Let's rewrite our queries. For better readability, we will use `with` clause and extract subquery.

| Sql Server | PostgreSQL | 
|---|---|
|CAST(date AS DATE)|date_trunc('day', date)|
|CAST(MIN(CreationDate)AS DATE) - ROW_NUMBER() OVER (ORDER BY MIN(CreationDate))|date_trunc('day', CreationDate) - INTERVAL '1' DAY * ROW_NUMBER() OVER (ORDER BY MIN(CreationDate))
|DATEDIFF(wk, start, end)|(DATE_PART('day', end - start)/7)::int|

####The final query for day streaks:
```sql
WITH
  groups(date, dateMinusRow) AS (
    SELECT 
      date_trunc('day', CreationDate) date,
      date_trunc('day', CreationDate) - INTERVAL '1' DAY * DENSE_RANK() OVER (ORDER BY date_trunc('day', CreationDate)) dateMinusRow
    FROM Posts
    GROUP BY date_trunc('day', CreationDate)
    HAVING COUNT(*) >= 1
  )
SELECT
  COUNT(*) AS streak,
  MIN(date) AS startDate,
  MAX(date) AS endDate
FROM groups
GROUP BY dateMinusRow
ORDER BY endDate DESC
```
[run it here](http://sqlfiddle.com/#!17/85a699/10/0)

####The final query for week streaks:
```sql
WITH
  groups(minDate, maxDate, weekMinusRow) AS (
    SELECT
      MIN(CreationDate) minDate,
      MAX(CreationDate) maxDate,
      (DATE_PART('day', CreationDate - to_timestamp(0))/7)::int - ROW_NUMBER() OVER (ORDER BY MIN(CreationDate)) weekMinusRow
    FROM Posts
    GROUP BY (DATE_PART('day', CreationDate - to_timestamp(0))/7)::int
    HAVING COUNT(*) >= 2
  )
SELECT
  COUNT(*) streak,
  MIN(minDate) startDate,
  MAX(maxDate) endDate
FROM groups
GROUP BY weekMinusRow
ORDER BY endDate DESC
```
[run it here](http://sqlfiddle.com/#!17/aaa67/21/0)
