# European Soccer Data Preparation and Analysis

## Table of Contents
1. [Introduction](#1-introduction)
2. [Data Description and Preparation](#2-data-description-and-preparation)
 - [Dataset](#a-dataset)
 - [Database Diagrams](#b-database-diagrams)
 - [Schema](#c-schema)
3. [Data Pre-Processing](#3-data-pre-processing)
 - [Removing Duplicates](#a-removing-duplicates)
 - [Eliminating Null Values](#b-eliminating-null-values)
 - [Data Integration and Transformation](#c-data-integration-and-transformation)
 - [Data Cleaning](#d-data-cleaning)
 - [Feature Engineering](#e-feature-engineering)
4. [Exploratory Data Analysis](#4-exploratory-data-analysis)
 - [Analysis on Country and League](#a-analysis-on-country-and-league)
 - [Analysis on Team](#b-analysis-on-team)
 - [Analysis on Player](#c-analysis-on-player)
5. [Data Analysis and Complex Data Manipulation](#5-data-analysis-and-complex-data-manipulation)
 - [Average Player Ratings](#a-average-player-ratings)
 - [Total Player Ratings](#b-total-player-ratings)
 - [Winning Metric](#c-winning-metric)
 - [Comparison for Each Metric](#d-comparison-for-each-metric)
 - [Home & Away Analysis](#e-home--away-analysis)
6. [Conclusion](#6-conclusion)
7. [Appendix](#7-appendix)

---

## 1. Introduction
This portfolio showcases my skills in using SQL queries to perform complex data manipulation and analysis. This project aims to demonstrate my ability to work with SQL databases, clean and prepare data, perform exploratory data analysis, and generate meaningful insights. 
This project analyses European soccer data to gain insight into team performance, player statistics and predict match outcomes. The query is performed using the Microsoft SQL Server Management Studio and the visualisation using Microsoft Power BI.

## 2. Data Description and Preparation
### A. Dataset
The dataset that I am using is the European soccer data set obtained from [Kaggle](https://www.kaggle.com/datasets/hugomathien/soccer). It contains seven tables with the following description:

| Table Name         | Description                                                                | Number Of Attributes |
|--------------------|----------------------------------------------------------------------------|----------------------|
| Country            | Contains country ID and country name                                       | 2                    |
| Leagues            | Contains league ID and league name for each country                        | 3                    |
| Matches            | Contains details about each match, including teams, players, and scores    | 115                  |
| Players            | Includes data on individual players, such as birthday, height, and weight  | 7                    |
| Player_Attributes  | Includes various player assessment metrics, such as the overall rating     | 42                   |
| Teams              | Provides information about the teams                                       | 5                    |
| Team_Attributes    | Provides various team assessment metrics                                   | 25                   |

After getting the dataset in a CSV form, I imported the file into the SSMS, determining the appropriate data type and removing the irrelevant column. Some were done manually, and some were done using dynamic SQL script. 

```SQL
/* Dynamic SQL script to remove irrelevant column */
DECLARE @column_name1 VARCHAR(100);
DECLARE @sql_statement NVARCHAR(1000);
DECLARE column_cursor CURSOR FOR
SELECT COLUMN_NAME
FROM INFORMATION_SCHEMA.COLUMNS
WHERE TABLE_NAME = 'Match' AND (COLUMN_NAME LIKE 'home_player_X[0-9]%' OR COLUMN_NAME LIKE 'home_player_y[0-9]%' OR COLUMN_NAME LIKE 'away_player_X[0-9]%' OR COLUMN_NAME LIKE 'away_player_y[0-9]%');
OPEN column_cursor;
FETCH NEXT FROM column_cursor INTO @column_name1;
WHILE @@FETCH_STATUS = 0
BEGIN
    SET @sql_statement = 'ALTER TABLE soc.Match DROP COLUMN IF EXISTS' + QUOTENAME(@column_name1);
    EXEC sp_executesql @sql_statement;
    FETCH NEXT FROM column_cursor INTO @column_name1;
END
CLOSE column_cursor;
DEALLOCATE column_cursor;
```

Please check [Appendix](#7-appendix) to see column information
```SQL
--Query to get column information
SELECT TABLE_NAME AS [Table Name], COLUMN_NAME AS [Column], DATA_TYPE AS [Data Type], IS_NULLABLE AS [Is Nullable]
FROM
    INFORMATION_SCHEMA.COLUMNS
WHERE
   TABLE_NAME = 'Country'           OR 
   TABLE_NAME = 'League'            OR 
   TABLE_NAME = 'Match'             OR 
   TABLE_NAME = 'Player'            OR 
   TABLE_NAME = 'Player_Attributes' OR 
   TABLE_NAME = 'Team'              OR 
   TABLE_NAME = 'Team_Attributes';
```

### B. Database Diagrams
I selected each table's primary and foreign keys while establishing their relationships.

![image](https://github.com/AMichaelDS/SQL/assets/132055953/675abefc-5e44-477f-8d15-1e0e119098cc)

### C. Schema
For this project, I created a schema so that it can be easily distinguished from the other dataset. The schema name I picked is soc.

## 3. Data Pre-Processing
Before analysing the dataset, it is important to pre-process the data to ensure that it is clean, consistent, and ready for analysis. 

### A. Removing Duplicates
Since there is 2 ID for the player and team, which is the api_id and fifa_id, I checked whether there is only one api_id for each fifa_id and vice versa.

#### Team (team & team_attributes)
```SQL
SELECT DISTINCT t1.team_api_id, t1.team_fifa_api_id
FROM soc.Team /*Team_Attributes*/t1
JOIN (
    SELECT team_fifa_api_id
    FROM soc.Team
    GROUP BY team_fifa_api_id
    HAVING COUNT(DISTINCT team_api_id) > 1
) t2 ON t1.team_fifa_api_id = t2.team_fifa_api_id
ORDER BY team_fifa_api_id, team_api_id;
```

![image](https://github.com/AMichaelDS/SQL/assets/132055953/b67fed88-c08f-4eb3-b070-bc52a96416f1)

Then I try to find the column that causes the duplicate

```SQL
SELECT *
FROM soc.Team
WHERE team_long_name IN (
  SELECT team_long_name
  FROM soc.Team
  GROUP BY team_long_name
  HAVING COUNT(*) > 1
)
ORDER BY team_fifa_api_id;
```

![image](https://github.com/AMichaelDS/SQL/assets/132055953/3334ad57-d2fe-4e33-ab87-cd3eef869147)

I remove the duplicate team by choosing only the first occurrence. 

```SQL
DROP TABLE IF EXISTS soc.Team2;

SELECT T.*
INTO soc.Team2
FROM soc.Team AS T
INNER JOIN (
  SELECT team_long_name, MIN(team_api_id) AS first_team_api_id
  FROM soc.Team
  GROUP BY team_long_name
) AS First ON T.team_long_name = First.team_long_name AND T.team_api_id = First.first_team_api_id;
```

#### Player (player & player_attributes)
```SQL
/*More than 1 fifa id per 1 api id*/
SELECT DISTINCT t1.player_api_id, t1.player_fifa_api_id
FROM soc.Player_Attributes t1
JOIN (
    SELECT player_api_id
    FROM soc.Player_Attributes
    GROUP BY player_api_id
    HAVING COUNT(DISTINCT player_fifa_api_id) > 1
) t2 ON t1.player_api_id = t2.player_api_id
ORDER BY player_api_id, player_fifa_api_id;

/*More than 1 api id per 1 fifa id*/
SELECT DISTINCT t1.player_api_id, t1.player_fifa_api_id
FROM soc.Player_Attributes t1
JOIN (
    SELECT player_fifa_api_id
    FROM soc.Player_Attributes
    GROUP BY player_fifa_api_id
    HAVING COUNT(DISTINCT player_api_id) > 1
) t2 ON t1.player_fifa_api_id = t2.player_fifa_api_id
ORDER BY player_fifa_api_id, player_api_id;
```

![image](https://github.com/AMichaelDS/SQL/assets/132055953/ee834a8b-2405-4a34-befd-ff2428c5e677)

Since the duplicate only exists on the player_attributes table, I did not remove any duplicates her because later by inner-joining the attribute table with the player table, it will be eliminated by itself.

### B. Eliminating Null Values
In the team_attributes table, the buildUpPlayDribbling column contains null value. 

![image](https://github.com/AMichaelDS/SQL/assets/132055953/81e5a92e-773c-48e7-b2bf-0b8b4ae0653f)

This will become an issue later, so I eliminate the null values by changing them to the average for that column.

```SQL
UPDATE soc.Team_Attributes
SET buildUpPlayDribbling = (
    SELECT AVG(buildUpPlayDribbling)
    FROM soc.Team_Attributes
    WHERE buildUpPlayDribbling IS NOT NULL
)
WHERE buildUpPlayDribbling IS NULL;
```

![image](https://github.com/AMichaelDS/SQL/assets/132055953/94533fc8-5a7e-4cf8-83f2-14c6904024b4)

### C. Data Integration and Transformation
After completing all the previous sections, I joined the relevant table to create a unified dataset so that analysis could be easily performed later. I also transformed the date to YYYYMMDD format.

#### Country & League
```SQL
DROP TABLE IF EXISTS soc.Country_League;

SELECT c.country_id, c.name AS country_name, l.league_id, l.name AS league_name
INTO soc.Country_League
FROM soc.League AS L 
JOIN soc.Country AS C 
ON C.country_id = L.country_id;
```

![image](https://github.com/AMichaelDS/SQL/assets/132055953/1253600e-f6a4-4bb3-8956-219a1a145b40)

#### Team(team & team_attributes)
```SQL
/* Joining*/
DROP TABLE IF EXISTS soc.TeamFin;

SELECT t.team_long_name, t.team_short_name, assessment_date = CONVERT(DATE, ta.date), t.team_api_id AS team_api_id_team, ta.*
INTO soc.TeamFin
FROM soc.Team AS t
INNER JOIN soc.Team_Attributes AS ta
ON t.team_api_id = ta.team_api_id AND t.team_fifa_api_id = ta.team_fifa_api_id;

/* Dropping the date and api id column since we have the updated one called team_api_id_team */
ALTER TABLE soc.TeamFin
DROP COLUMN IF EXISTS date, team_api_id;

/* Renaming the team_api_id_team column back to team_api_id */
EXEC sp_rename 'soc.TeamFin.team_api_id_team', 'team_api_id', 'COLUMN';
```

![image](https://github.com/AMichaelDS/SQL/assets/132055953/39d9ad51-3787-48eb-b47f-cfeaf20227e6)

#### Player(player & player_attributes)
```SQL
/* Joining */
DROP TABLE IF EXISTS soc.PlayerFin;

SELECT p.player_name, birthday = CONVERT(DATE, p.birthday) , p.height, p.weight, assessment_date = CONVERT(DATE, pa.date), pa.*
INTO soc.PlayerFin
FROM soc.Player AS p
INNER JOIN  soc.Player_Attributes AS pa
ON p.player_api_id = pa.player_api_id AND p.player_fifa_api_id = pa.player_fifa_api_id;

/* Removing unchanged date */
ALTER TABLE soc.PlayerFin
DROP COLUMN IF EXISTS date;
```

![image](https://github.com/AMichaelDS/SQL/assets/132055953/c097f9b6-f444-4baf-8e4f-10975343cf7a)

#### Match(Match & All Tables)
Here I substituted the ID for each country, league, player, and team with the name to make it easier to look at. 

```SQL
/* Please take note that this query is executed much later on after cleaning the other table,
and creating analysis table for team and player. 
It is only located here to match the other join query  */

DROP TABLE IF EXISTS soc.MatchFin;

SELECT DISTINCT M.match_api_id, CL.country_name, CL.league_name, season, stage, CONVERT(DATE, M.date) AS date, 
    HT.team_long_name AS home_team, HT_AN.overall_rating AS home_team_rating,
    AT.team_long_name AS away_team, AT_AN.overall_rating AS away_team_rating,
    home_team_goal, 
    away_team_goal,					  
          HTP1.player_name AS home_player_1,
    HTP1_AN.overall_rating AS home_player_1_rating,
          HTP2.player_name AS home_player_2,
    HTP2_AN.overall_rating AS home_player_2_rating,
          HTP3.player_name AS home_player_3,
    HTP3_AN.overall_rating AS home_player_3_rating,
          HTP4.player_name AS home_player_4,
    HTP4_AN.overall_rating AS home_player_4_rating,
          HTP5.player_name AS home_player_5,
    HTP5_AN.overall_rating AS home_player_5_rating,
          HTP6.player_name AS home_player_6,
    HTP6_AN.overall_rating AS home_player_6_rating,
          HTP7.player_name AS home_player_7,
    HTP7_AN.overall_rating AS home_player_7_rating,
          HTP8.player_name AS home_player_8,
    HTP8_AN.overall_rating AS home_player_8_rating,
          HTP9.player_name AS home_player_9,
    HTP9_AN.overall_rating AS home_player_9_rating,
         HTP10.player_name AS home_player_10,
   HTP10_AN.overall_rating AS home_player_10_rating,
         HTP11.player_name AS home_player_11,
   HTP11_AN.overall_rating AS home_player_11_rating,
          ATP1.player_name AS away_player_1,
    ATP1_AN.overall_rating AS away_player_1_rating,
          ATP2.player_name AS away_player_2,
    ATP2_AN.overall_rating AS away_player_2_rating,
          ATP3.player_name AS away_player_3,
    ATP3_AN.overall_rating AS away_player_3_rating,
          ATP4.player_name AS away_player_4,
    ATP4_AN.overall_rating AS away_player_4_rating,
          ATP5.player_name AS away_player_5,
    ATP5_AN.overall_rating AS away_player_5_rating,
          ATP6.player_name AS away_player_6,
    ATP6_AN.overall_rating AS away_player_6_rating,
          ATP7.player_name AS away_player_7,
    ATP7_AN.overall_rating AS away_player_7_rating,
          ATP8.player_name AS away_player_8,
    ATP8_AN.overall_rating AS away_player_8_rating,
          ATP9.player_name AS away_player_9,
    ATP9_AN.overall_rating AS away_player_9_rating,
         ATP10.player_name AS away_player_10,
   ATP10_AN.overall_rating AS away_player_10_rating,
         ATP11.player_name AS away_player_11,
   ATP11_AN.overall_rating AS away_player_11_rating
INTO soc.MatchFin
FROM soc.Match AS M
JOIN soc.Country_League AS CL ON CL.country_id = M.country_id
LEFT JOIN soc.Team            AS HT       ON HT.team_api_id         = M.home_team_api_id
LEFT JOIN soc.Team            AS AT       ON AT.team_api_id         = M.away_team_api_id
LEFT JOIN soc.analysis_team   AS AT_AN    ON AT_AN.team_api_id      = AT.team_api_id
LEFT JOIN soc.analysis_team   AS HT_AN    ON HT_AN.team_api_id      = HT.team_api_id
LEFT JOIN soc.Player          AS HTP1     ON HTP1.player_api_id     = M.home_player_1
LEFT JOIN soc.Player          AS HTP2     ON HTP2.player_api_id     = M.home_player_2
LEFT JOIN soc.Player          AS HTP3     ON HTP3.player_api_id     = M.home_player_3
LEFT JOIN soc.Player          AS HTP4     ON HTP4.player_api_id     = M.home_player_4
LEFT JOIN soc.Player          AS HTP5     ON HTP5.player_api_id     = M.home_player_5
LEFT JOIN soc.Player          AS HTP6     ON HTP6.player_api_id     = M.home_player_6
LEFT JOIN soc.Player          AS HTP7     ON HTP7.player_api_id     = M.home_player_7
LEFT JOIN soc.Player          AS HTP8     ON HTP8.player_api_id     = M.home_player_8
LEFT JOIN soc.Player          AS HTP9     ON HTP9.player_api_id     = M.home_player_9
LEFT JOIN soc.Player          AS HTP10    ON HTP10.player_api_id    = M.home_player_10
LEFT JOIN soc.Player          AS HTP11    ON HTP11.player_api_id    = M.home_player_11
LEFT JOIN soc.Player          AS ATP1     ON ATP1.player_api_id     = M.away_player_1 
LEFT JOIN soc.Player          AS ATP2     ON ATP2.player_api_id     = M.away_player_2 
LEFT JOIN soc.Player          AS ATP3     ON ATP3.player_api_id     = M.away_player_3 
LEFT JOIN soc.Player          AS ATP4     ON ATP4.player_api_id     = M.away_player_4 
LEFT JOIN soc.Player          AS ATP5     ON ATP5.player_api_id     = M.away_player_5 
LEFT JOIN soc.Player          AS ATP6     ON ATP6.player_api_id     = M.away_player_6 
LEFT JOIN soc.Player          AS ATP7     ON ATP7.player_api_id     = M.away_player_7 
LEFT JOIN soc.Player          AS ATP8     ON ATP8.player_api_id     = M.away_player_8 
LEFT JOIN soc.Player          AS ATP9     ON ATP9.player_api_id     = M.away_player_9 
LEFT JOIN soc.Player          AS ATP10    ON ATP10.player_api_id    = M.away_player_10
LEFT JOIN soc.Player          AS ATP11    ON ATP11.player_api_id    = M.away_player_11
LEFT JOIN soc.analysis_player AS HTP1_AN  ON HTP1_AN.player_api_id  = HTP1.player_api_id
LEFT JOIN soc.analysis_player AS HTP2_AN  ON HTP2_AN.player_api_id  = HTP2.player_api_id
LEFT JOIN soc.analysis_player AS HTP3_AN  ON HTP3_AN.player_api_id  = HTP3.player_api_id
LEFT JOIN soc.analysis_player AS HTP4_AN  ON HTP4_AN.player_api_id  = HTP4.player_api_id
LEFT JOIN soc.analysis_player AS HTP5_AN  ON HTP5_AN.player_api_id  = HTP5.player_api_id
LEFT JOIN soc.analysis_player AS HTP6_AN  ON HTP6_AN.player_api_id  = HTP6.player_api_id
LEFT JOIN soc.analysis_player AS HTP7_AN  ON HTP7_AN.player_api_id  = HTP7.player_api_id
LEFT JOIN soc.analysis_player AS HTP8_AN  ON HTP8_AN.player_api_id  = HTP8.player_api_id
LEFT JOIN soc.analysis_player AS HTP9_AN  ON HTP9_AN.player_api_id  = HTP9.player_api_id
LEFT JOIN soc.analysis_player AS HTP10_AN ON HTP10_AN.player_api_id = HTP10.player_api_id
LEFT JOIN soc.analysis_player AS HTP11_AN ON HTP11_AN.player_api_id = HTP11.player_api_id
LEFT JOIN soc.analysis_player AS ATP1_AN  ON ATP1_AN.player_api_id  = ATP1.player_api_id
LEFT JOIN soc.analysis_player AS ATP2_AN  ON ATP2_AN.player_api_id  = ATP2.player_api_id
LEFT JOIN soc.analysis_player AS ATP3_AN  ON ATP3_AN.player_api_id  = ATP3.player_api_id
LEFT JOIN soc.analysis_player AS ATP4_AN  ON ATP4_AN.player_api_id  = ATP4.player_api_id
LEFT JOIN soc.analysis_player AS ATP5_AN  ON ATP5_AN.player_api_id  = ATP5.player_api_id
LEFT JOIN soc.analysis_player AS ATP6_AN  ON ATP6_AN.player_api_id  = ATP6.player_api_id
LEFT JOIN soc.analysis_player AS ATP7_AN  ON ATP7_AN.player_api_id  = ATP7.player_api_id
LEFT JOIN soc.analysis_player AS ATP8_AN  ON ATP8_AN.player_api_id  = ATP8.player_api_id
LEFT JOIN soc.analysis_player AS ATP9_AN  ON ATP9_AN.player_api_id  = ATP9.player_api_id
LEFT JOIN soc.analysis_player AS ATP10_AN ON ATP10_AN.player_api_id = ATP10.player_api_id
LEFT JOIN soc.analysis_player AS ATP11_AN ON ATP11_AN.player_api_id = ATP11.player_api_id
ORDER BY home_team, away_team;
```

![image](https://github.com/AMichaelDS/SQL/assets/132055953/1f5c3055-a432-41f3-83e3-7c4d92c95971)

### D. Data Cleaning
In the PlayerFin table,  the column attacking_work_rate and defensive_work_rate has some irrelevant values, which I converted to null using a case-when statement.

```SQL
UPDATE soc.playerFin
SET defensive_work_rate =
    CASE 
        WHEN defensive_work_rate IN ('_0', 'ean', '0', 'o', 'ormal', 'tocky', 'es') THEN NULL
        WHEN defensive_work_rate = 'low' THEN 'Low'
        WHEN defensive_work_rate = 'medium' THEN 'Medium'
        WHEN defensive_work_rate = 'high' THEN 'High'
        WHEN defensive_work_rate BETWEEN '1' AND '3' THEN 'Low'
        WHEN defensive_work_rate BETWEEN '4' AND '6' THEN 'Medium'
        WHEN defensive_work_rate BETWEEN '7' AND '9' THEN 'High'
        ELSE defensive_work_rate -- To handle any other values
    END,
    attacking_work_rate =
    CASE 
        WHEN attacking_work_rate IN ('le', 'None', 'norm', 'stoc', 'y') THEN NULL
        WHEN attacking_work_rate = 'low' THEN 'Low'
        WHEN attacking_work_rate = 'medium' THEN 'Medium'
        WHEN attacking_work_rate = 'high' THEN 'High'
        ELSE attacking_work_rate -- To handle any other values
    END;
```

### E. Feature Engineering
For the TeamFin table, I noticed that there is no overall rating, which is essential because having an overall rating allows us to evaluate and compare the performance of different soccer teams more comprehensively.
I summed the value for all the metrics and divided it by 9. Initially, I dealt with the null value by simply changing the denominator. But then, I decided to eliminate the null value earlier in the data pre-processing phase.

```SQL
/* Creating Overall_Rating Column */
ALTER TABLE soc.TeamFin
ADD Overall_Rating INT;

UPDATE soc.TeamFin
SET Overall_Rating =
  CASE
    WHEN buildUpPlayDribbling IS NOT NULL THEN
      CAST(
        ROUND(
          (
            COALESCE(buildUpPlaySpeed, 0) + COALESCE(buildUpPlayDribbling, 0) + COALESCE(buildUpPlayPassing, 0)
            + COALESCE(chanceCreationPassing, 0) + COALESCE(chanceCreationCrossing, 0) + COALESCE(chanceCreationShooting, 0)
            + COALESCE(defencePressure, 0) + COALESCE(defenceAggression, 0) + COALESCE(defenceTeamWidth, 0)
          ) / 9.0
        , 0)
        AS INT
      )
    ELSE /* In the beginning the buildUpPlayDribbling contains null value and it was fixed above*/ 
      CAST(
        ROUND(
          (
            COALESCE(buildUpPlaySpeed, 0) + COALESCE(buildUpPlayPassing, 0)
            + COALESCE(chanceCreationPassing, 0) + COALESCE(chanceCreationCrossing, 0) + COALESCE(chanceCreationShooting, 0)
            + COALESCE(defencePressure, 0) + COALESCE(defenceAggression, 0) + COALESCE(defenceTeamWidth, 0)
          ) / 8.0
        , 0)
        AS INT
      )
  END;;
```

![image](https://github.com/AMichaelDS/SQL/assets/132055953/1166152d-2ecd-4478-9b63-8ffdeeb02539)

## 4. Exploratory Data Analysis
After finishing the data preparation, we can now start analysing the dataset. 

### A. Analysis on Country and League
I created the sum and average for each country and league to analyse which country/league is more dominant. 

```SQL
DROP TABLE IF EXISTS soc.Analysis_Country_League;

SELECT country_name, league_name, num_team, total_home_goal, total_away_goal, total_goal, avg_home_goal_per_team, avg_away_goal_per_team, avg_home_goal_per_team + avg_away_goal_per_team AS total_avg_goal_per_team
INTO soc.Analysis_Country_League
FROM(
	SELECT country_name, league_name, num_team, total_home_goal, total_away_goal, total_home_goal + total_away_goal AS total_goal, total_home_goal/num_team AS avg_home_goal_per_team, total_away_goal/num_team AS avg_away_goal_per_team
	FROM (
		SELECT country_name, league_name, COUNT(DISTINCT home_team) AS num_team, SUM(home_team_goal) AS total_home_goal, SUM(away_team_goal) AS total_away_goal
		FROM soc.MatchFin
		GROUP BY country_name, league_name
	) AS subquery
) AS subquery
ORDER BY num_team;
```

![image](https://github.com/AMichaelDS/SQL/assets/132055953/16bdad11-1ea8-42d6-8ce5-3874b5f907b9)

![image](https://github.com/AMichaelDS/SQL/assets/132055953/cc49ec87-3fe4-4139-8675-23da41c61eab)

Here we can see that Spain Liga BBVA has the highest total goal which could imply that it is the most dominant league out of the other league.

### B. Analysis on Team
To analyse this table, I picked only the latest assessment date for each team by using this query:

```SQL
DROP TABLE IF EXISTS soc.Temp;

SELECT TF1.*
INTO soc.Temp
FROM soc.TeamFin AS TF1
INNER JOIN (
  SELECT team_long_name, MAX(assessment_date) AS max_date
  FROM soc.TeamFin
  GROUP BY team_long_name
) AS TF2 ON TF1.team_long_name = TF2.team_long_name AND TF1.assessment_date = TF2.max_date;
```

![image](https://github.com/AMichaelDS/SQL/assets/132055953/5e94b1a6-f06a-4d55-8496-fc24d91d2cbe)

Then I removed the duplicate team/observation by selecting only the first occurrence for each team name.

```SQL
DROP TABLE IF EXISTS soc.Analysis_Team;

SELECT *
INTO soc.Analysis_Team
FROM (
    SELECT *,
           ROW_NUMBER() OVER (PARTITION BY team_long_name ORDER BY team_api_id) AS row_num
    FROM soc.Temp
) AS subquery
WHERE row_num = 1;

/* Dropping row_num */
ALTER TABLE soc.Analysis_Team
DROP COLUMN IF EXISTS row_num;
```

![image](https://github.com/AMichaelDS/SQL/assets/132055953/7ce72515-a5b7-40b1-aa24-dc67cb40e614)

#### Highest Overall Team Rating
I selected ten teams with the highest rating

```SQL
SELECT TOP 10 team_long_name, Overall_Rating
FROM soc.Analysis_Team
ORDER BY overall_rating DESC;
```

![image](https://github.com/AMichaelDS/SQL/assets/132055953/edab345c-650d-4e8d-b4cd-1d0d9f35044b)

![image](https://github.com/AMichaelDS/SQL/assets/132055953/8a5e1cf1-8940-478d-a8b6-47152ba0d7cd)

Lazio has the highest overall rating out of all the team.

#### Numerical Analysis 
I used a dynamic SQL script to create tables containing team with the highest performance metrics for each numerical attribute.

```SQL
DECLARE @sql NVARCHAR(MAX);
SET @sql = N'';

SELECT @sql = @sql + N'
WITH MaxValue AS (
    SELECT MAX(' + QUOTENAME(Column_Name) + ') AS MaxValue FROM soc.Analysis_Team
)
SELECT a.*
INTO soc.' + QUOTENAME('Team_Stat_' + Column_Name) + '
FROM soc.Analysis_Team a
CROSS JOIN MaxValue
WHERE ' + QUOTENAME(Column_Name) + ' = MaxValue.MaxValue;' + CHAR(13)
FROM (
    VALUES
        ('buildUpPlaySpeed'),
        ('buildUpPlayDribbling'),
        ('buildUpPlayPassing'),
        ('chanceCreationPassing'),
        ('chanceCreationCrossing'),
        ('chanceCreationShooting'),
        ('defencePressure'),
        ('defenceAggression'),
        ('defenceTeamWidth')
) AS T(Column_Name);

EXEC sp_executesql @sql;
```
Here is all the table that is created using the query above

![image](https://github.com/AMichaelDS/SQL/assets/132055953/20461c3e-d655-478a-a3b8-c63dd5f9e79e)

<details>
<summary>Build Up Play Dribbling</summary>
	
![image](https://github.com/AMichaelDS/SQL/assets/132055953/9d0a2446-3817-4d62-bf43-5c99b1080d5b)
</details>

<details>
<summary>Build-Up Play Passing</summary>

![image](https://github.com/AMichaelDS/SQL/assets/132055953/42d40609-4982-43a1-bf68-80d61fefa7a4)	
</details>

<details>
<summary>Build-Up Play Speed</summary>
	
![image](https://github.com/AMichaelDS/SQL/assets/132055953/269ad582-128d-4758-b2e5-651b95e1326d)
</details>

<details>
<summary>Chance Creation Crossing</summary>
	
![image](https://github.com/AMichaelDS/SQL/assets/132055953/69563bca-1921-4786-9e1d-e0b1f00f7ced)
</details>

<details>
<summary>Chance Creation Passing</summary>

![image](https://github.com/AMichaelDS/SQL/assets/132055953/77756a8e-8659-4ce5-ba2a-569c7f8b921f)
</details>

<details>
<summary>Chance Creation Shooting</summary>

![image](https://github.com/AMichaelDS/SQL/assets/132055953/e1a82838-f48d-4ed3-91fc-a9a631b35716)
</details>

<details>
<summary>Defence Aggression</summary>

![image](https://github.com/AMichaelDS/SQL/assets/132055953/7548f732-6bcf-48ee-aa58-11ce2f96d8bd)
</details>

<details>
<summary>Defence Pressure</summary>

![image](https://github.com/AMichaelDS/SQL/assets/132055953/e623f3c0-8250-4de5-82c4-600d723a24d8)
</details>

<details>
<summary>Defence Team Width</summary>

![image](https://github.com/AMichaelDS/SQL/assets/132055953/94e4fe5e-6442-46af-a107-35c8f11144e9)
</details>

#### Categorical Analysis
Here I performed a categorical analysis to find the distribution of teams across the categorical attribute. It is done to get a relative representation of teams in different categories within each attribute.

<details>
<summary>Build-Up Play Dribbling Class</summary>
	
```SQL
SELECT 
    buildUpPlayDribblingClass, 
    COUNT(*) AS buildUpPlayDribblingCount, 
    CAST(ROUND(COUNT(*) * 100.0 / (SELECT COUNT(*) FROM soc.analysis_team), 2) AS decimal(10, 2)) AS buildUpPlayDribblingPercentage
FROM soc.analysis_team
GROUP BY buildUpPlayDribblingClass;
```

![image](https://github.com/AMichaelDS/SQL/assets/132055953/bfe0c535-771d-439f-8e14-e43b93559160)
![image](https://github.com/AMichaelDS/SQL/assets/132055953/7345a932-4585-4c2a-9244-1c95126ddaa1)
</details>

<details>
<summary>Build-Up Play Passing Class</summary>
	
```SQL
SELECT 
    buildUpPlayPassingClass, 
    COUNT(*) AS buildUpPlayPassingCount, 
    CAST(ROUND(COUNT(*) * 100.0 / (SELECT COUNT(*) FROM soc.analysis_team), 2) AS decimal(10, 2)) AS buildUpPlayPassingPercentage
FROM soc.analysis_team
GROUP BY buildUpPlayPassingClass;
```

![image](https://github.com/AMichaelDS/SQL/assets/132055953/0ae7d854-591e-4578-94f6-4fde9ef414be)
![image](https://github.com/AMichaelDS/SQL/assets/132055953/1f3836b5-746b-4b0a-bd5d-0cff07208dde)
</details>

<details>
<summary>Build-Up Play Positioning Class</summary>

```SQL
SELECT 
    buildUpPlayPositioningClass, 
    COUNT(*) AS buildUpPlayPositioningCount, 
    CAST(ROUND(COUNT(*) * 100.0 / (SELECT COUNT(*) FROM soc.analysis_team), 2) AS decimal(10, 2)) AS buildUpPlayPositioningPercentage
FROM soc.analysis_team
GROUP BY buildUpPlayPositioningClass;
```

![image](https://github.com/AMichaelDS/SQL/assets/132055953/4a3a14b0-951e-4755-8f0a-4aae850dcf20)
![image](https://github.com/AMichaelDS/SQL/assets/132055953/39e7b0a0-121a-4f2c-8e6a-9c78c6a52ab7)
</details>

<details>
<summary>Build-Up Play Speed Class</summary>

```SQL
SELECT 
    buildUpPlaySpeedClass, 
    COUNT(*) AS buildUpPlaySpeedCount, 
    CAST(ROUND(COUNT(*) * 100.0 / (SELECT COUNT(*) FROM soc.analysis_team), 2) AS decimal(10, 2)) AS buildUpPlaySpeedPercentage
FROM soc.analysis_team
GROUP BY buildUpPlaySpeedClass;
```	
![image](https://github.com/AMichaelDS/SQL/assets/132055953/62500a6e-b354-465a-a319-5f6a9812a8cb)
![image](https://github.com/AMichaelDS/SQL/assets/132055953/f60551c9-7813-4122-a432-403d769d72ca)
</details>

<details>
<summary>Chance Creation Crossing Class</summary>

```SQL
SELECT 
    chanceCreationCrossingClass, 
    COUNT(*) AS chanceCreationCrossingCount, 
    CAST(ROUND(COUNT(*) * 100.0 / (SELECT COUNT(*) FROM soc.analysis_team), 2) AS decimal(10, 2)) AS chanceCreationCrossingPercentage
FROM soc.analysis_team
GROUP BY chanceCreationCrossingClass;
```

![image](https://github.com/AMichaelDS/SQL/assets/132055953/e8d60518-7736-4c81-9185-ecbb800242bb)
![image](https://github.com/AMichaelDS/SQL/assets/132055953/e0ac2d31-16b1-4dca-bc8a-9b4f4fc9a9bc)
</details>

<details>
<summary>Chance Creation Passing Class</summary>

```SQL
SELECT 
    chanceCreationPassingClass, 
    COUNT(*) AS chanceCreationPassingCount, 
    CAST(ROUND(COUNT(*) * 100.0 / (SELECT COUNT(*) FROM soc.analysis_team), 2) AS decimal(10, 2)) AS chanceCreationPassingPercentage
FROM soc.analysis_team
GROUP BY chanceCreationPassingClass;
```

![image](https://github.com/AMichaelDS/SQL/assets/132055953/ae9854cb-c033-470e-af2a-a9e5b5d39002)
![image](https://github.com/AMichaelDS/SQL/assets/132055953/ec1c6197-29fd-4bbc-8603-db0607af9c2d)
</details>

<details>
<summary>Chance Creation Positioning Class</summary>

```SQL
SELECT 
    chanceCreationPositioningClass, 
    COUNT(*) AS chanceCreationPositioningCount, 
    CAST(ROUND(COUNT(*) * 100.0 / (SELECT COUNT(*) FROM soc.analysis_team), 2) AS decimal(10, 2)) AS chanceCreationPositioningPercentage
FROM soc.analysis_team
GROUP BY chanceCreationPositioningClass;
```

![image](https://github.com/AMichaelDS/SQL/assets/132055953/69df8936-d33a-4e92-97d0-4a7b5ebdbb43)
![image](https://github.com/AMichaelDS/SQL/assets/132055953/75408fcb-ce1d-42fc-b649-bd2078536505)
</details>

<details>
<summary>Chance Creation Shooting Class</summary>

```SQL
SELECT 
    chanceCreationShootingClass, 
    COUNT(*) AS chanceCreationShootingCount, 
    CAST(ROUND(COUNT(*) * 100.0 / (SELECT COUNT(*) FROM soc.analysis_team), 2) AS decimal(10, 2)) AS chanceCreationShootingPercentage
FROM soc.analysis_team
GROUP BY chanceCreationShootingClass;
```

![image](https://github.com/AMichaelDS/SQL/assets/132055953/05e76efe-ba39-4390-b520-9fbbe9f1d816)
![image](https://github.com/AMichaelDS/SQL/assets/132055953/0174bd15-76cb-452b-9ef6-f97fc2a6ff0b)
</details>

<details>
<summary>Defence Aggression Class</summary>

```SQL
SELECT 
    defenceAggressionClass, 
    COUNT(*) AS defenceAggressionCount, 
    CAST(ROUND(COUNT(*) * 100.0 / (SELECT COUNT(*) FROM soc.analysis_team), 2) AS decimal(10, 2)) AS defenceAggressionPercentage
FROM soc.analysis_team
GROUP BY defenceAggressionClass;
```

![image](https://github.com/AMichaelDS/SQL/assets/132055953/f009a32c-c8f6-429d-b940-fb7c4f1f531e)
![image](https://github.com/AMichaelDS/SQL/assets/132055953/a71b9f3b-43e1-4e7d-aebb-06ece7aa945f)
</details>

<details>
<summary>Defence Defender Line Class</summary>

```SQL
SELECT 
    defenceDefenderLineClass, 
    COUNT(*) AS defenceDefenderLineCount, 
    CAST(ROUND(COUNT(*) * 100.0 / (SELECT COUNT(*) FROM soc.analysis_team), 2) AS decimal(10, 2)) AS defenceDefenderLinePercentage
FROM soc.analysis_team
GROUP BY defenceDefenderLineClass;
```

![image](https://github.com/AMichaelDS/SQL/assets/132055953/38af477b-c6d1-4740-a37e-790ce31b11dd)
![image](https://github.com/AMichaelDS/SQL/assets/132055953/cc675d7b-6386-4c4f-a27a-3ebcbba32fb8)
</details>

<details>
<summary>Defence Pressure Class</summary>

```SQL
SELECT 
    defencePressureClass, 
    COUNT(*) AS defencePressureCount, 
    CAST(ROUND(COUNT(*) * 100.0 / (SELECT COUNT(*) FROM soc.analysis_team), 2) AS decimal(10, 2)) AS defencePressurePercentage
FROM soc.analysis_team
GROUP BY defencePressureClass;
```

![image](https://github.com/AMichaelDS/SQL/assets/132055953/42a28dd2-0b68-4946-9405-974f6b6c6fb2)
	
![image](https://github.com/AMichaelDS/SQL/assets/132055953/a7530f0a-8130-4b92-8d34-4b3509b36dbd)
</details>

<details>
<summary>Defence Team Width Class</summary>

```SQL
SELECT 
    defenceTeamWidthClass, 
    COUNT(*) AS defenceTeamWidthCount, 
    CAST(ROUND(COUNT(*) * 100.0 / (SELECT COUNT(*) FROM soc.analysis_team), 2) AS decimal(10, 2)) AS defenceTeamWidthPercentage
FROM soc.analysis_team
GROUP BY defenceTeamWidthClass;
```

![image](https://github.com/AMichaelDS/SQL/assets/132055953/19baf585-e1fb-4699-8eac-945f9b55e03f)
![image](https://github.com/AMichaelDS/SQL/assets/132055953/8adef15b-f6e5-4bb9-903c-2a1db65b55b5)
</details>

### C. Analysis on Player
Like what I did for TeamFin, I picked only the latest assessment date for each player.

```SQL
DROP TABLE IF EXISTS soc.Analysis_Player;

SELECT PF1.*
INTO soc.Analysis_Player
FROM soc.PlayerFin AS PF1
INNER JOIN (
  SELECT player_name, birthday, MAX(assessment_date) AS max_date
  FROM soc.PlayerFin
  GROUP BY player_name, birthday
) AS PF2 ON PF1.player_name = PF2.player_name AND PF1.birthday = PF2.birthday AND PF1.assessment_date = PF2.max_date
WHERE overall_rating IS NOT NULL /*to remove the duplicate record*/
ORDER BY player_name;
```

![image](https://github.com/AMichaelDS/SQL/assets/132055953/97435005-d29f-409c-ba2a-c228db5e2692)

#### Highest Overall Player Rating
I selected ten players with the highest rating

```SQL
SELECT TOP 10 *
FROM soc.Analysis_Player
ORDER BY overall_rating DESC;
```

![image](https://github.com/AMichaelDS/SQL/assets/132055953/362621b6-cd54-4934-8b27-732925a8552f)
![image](https://github.com/AMichaelDS/SQL/assets/132055953/6168f3df-1642-4775-ae93-6e3d39a3b412)


Lionel Messi has the highest player rating out of all the other player.

#### Highest For Each Rating
I use dynamic SQL script to create tables containing player with the highest rating for each attribute. 

```SQL
DECLARE @sql NVARCHAR(MAX);
SET @sql = N'';

SELECT @sql = @sql + N'SELECT TOP 1 * INTO soc.' + QUOTENAME('Player_Stat_' + COLUMN_NAME) + N' FROM soc.Analysis_Player ORDER BY ' + QUOTENAME(COLUMN_NAME) + N' DESC;' + CHAR(13)
FROM INFORMATION_SCHEMA.COLUMNS
WHERE TABLE_NAME = 'Analysis_Player';

EXEC sp_executesql @sql;
```

Then I use another script to remove irrelevant table and table that has categorical attribute

```SQL
- Then drop the irrelevant tabel
DECLARE @sql NVARCHAR(MAX);
SET @sql = N'';

SELECT @sql = @sql + N'DROP TABLE IF EXISTS ' + QUOTENAME(TABLE_SCHEMA) + '.' + QUOTENAME(TABLE_NAME) + ';' + CHAR(13)
FROM INFORMATION_SCHEMA.TABLES
WHERE TABLE_SCHEMA = 'soc'
  AND (
    TABLE_NAME = 'Player_Stat_player_name'
    OR TABLE_NAME = 'Player_Stat_assessment_date'
    OR TABLE_NAME = 'Player_Stat_player_attributes_id'
    OR TABLE_NAME = 'Player_Stat_player_fifa_api_id'
    OR TABLE_NAME = 'Player_Stat_player_api_id'
    OR TABLE_NAME = 'Player_Stat_preferred_foot'
    OR TABLE_NAME = 'Player_Stat_attacking_work_rate'
    OR TABLE_NAME = 'Player_Stat_defensive_work_rate'
  );

EXEC sp_executesql @sql;
```
Here is all the table that is created using the query above.

![image](https://github.com/AMichaelDS/SQL/assets/132055953/562b8b72-b51a-4400-899d-54266dcb554d)

#### Categorical Analysis
Here I performed a categorical analysis to find the distribution of players across the categorical attribute. It is done to get a relative representation of players in different categories within each attribute.

<details>
<summary>Count and percentage of players by preferred foot</summary>
	
```SQL
SELECT 
    preferred_foot,
    COUNT(*) AS playerCount,
    CAST(ROUND(COUNT(*) * 100.0 / (SELECT COUNT(*) FROM soc.Analysis_Player), 2) AS decimal(10,2)) AS playerPercentage
FROM soc.Analysis_Player
GROUP BY preferred_foot;
```

![image](https://github.com/AMichaelDS/SQL/assets/132055953/3d89bc65-c3d0-425a-af77-85cd1ebf4f0d)
	
![image](https://github.com/AMichaelDS/SQL/assets/132055953/cf216695-db87-4706-88ed-2277dcc5e670)
</details>

<details>
<summary>Count and percentage of players by attacking work rate</summary>
	
```SQL
SELECT 
    attacking_work_rate,
    COUNT(*) AS playerCount,
    CAST(ROUND(COUNT(*) * 100.0 / (SELECT COUNT(*) FROM soc.Analysis_Player), 2) AS decimal(10,2)) AS playerPercentage
FROM soc.Analysis_Player
GROUP BY attacking_work_rate;
```

![image](https://github.com/AMichaelDS/SQL/assets/132055953/c869d14a-c4c7-47be-bc14-2e84db81fc04)
	
![image](https://github.com/AMichaelDS/SQL/assets/132055953/56c61596-7177-4567-9d2a-a8ac7bd13080)
</details>

<details>
<summary>Count and percentage of players by defensive work rate</summary>
	
```SQL
SELECT 
    defensive_work_rate,
    COUNT(*) AS playerCount,
    CAST(ROUND(COUNT(*) * 100.0 / (SELECT COUNT(*) FROM soc.Analysis_Player), 2) AS decimal(10,2)) AS playerPercentage
FROM soc.Analysis_Player
GROUP BY defensive_work_rate;
```

![image](https://github.com/AMichaelDS/SQL/assets/132055953/7af7bcaa-d8c7-4ae0-a13e-841f123ca775)
	
![image](https://github.com/AMichaelDS/SQL/assets/132055953/92ea703a-74fb-462f-833e-cc226756c485)
</details>

## 5. Data Analysis and Complex Data Manipulation
### A. Average Player Ratings
For MatchFin, I created an average rating of players for each team(home & away).  
First, it counted the number of players on the team with null value and summed it up. For teams where all the player rating is null, the average becomes 0. It is done to avoid dividing by 0 later on.
Then I sum the ratings of the player then divide it by the number of players that did not have a null value.

```SQL
ALTER TABLE soc.MatchFin
ADD home_avg_player_rating INT, away_avg_player_rating INT;

UPDATE soc.MatchFin
SET home_avg_player_rating =
	CASE WHEN (
		CASE WHEN home_player_1_rating  IS NULL THEN 1 ELSE 0 END +
        CASE WHEN home_player_2_rating  IS NULL THEN 1 ELSE 0 END +
        CASE WHEN home_player_3_rating  IS NULL THEN 1 ELSE 0 END +
        CASE WHEN home_player_4_rating  IS NULL THEN 1 ELSE 0 END +
        CASE WHEN home_player_5_rating  IS NULL THEN 1 ELSE 0 END +
        CASE WHEN home_player_6_rating  IS NULL THEN 1 ELSE 0 END +
        CASE WHEN home_player_7_rating  IS NULL THEN 1 ELSE 0 END +
        CASE WHEN home_player_8_rating  IS NULL THEN 1 ELSE 0 END +
        CASE WHEN home_player_9_rating  IS NULL THEN 1 ELSE 0 END +
        CASE WHEN home_player_10_rating IS NULL THEN 1 ELSE 0 END +
        CASE WHEN home_player_11_rating IS NULL THEN 1 ELSE 0 END
    ) = 11 THEN 0  -- All ratings are null, set average as 0
    ELSE
		CAST(
			ROUND(
			(
			COALESCE(CAST(home_player_1_rating  AS DECIMAL), 0) +
            COALESCE(CAST(home_player_2_rating  AS DECIMAL), 0) +
            COALESCE(CAST(home_player_3_rating  AS DECIMAL), 0) +
            COALESCE(CAST(home_player_4_rating  AS DECIMAL), 0) +
            COALESCE(CAST(home_player_5_rating  AS DECIMAL), 0) +
            COALESCE(CAST(home_player_6_rating  AS DECIMAL), 0) +
            COALESCE(CAST(home_player_7_rating  AS DECIMAL), 0) +
            COALESCE(CAST(home_player_8_rating  AS DECIMAL), 0) +
            COALESCE(CAST(home_player_9_rating  AS DECIMAL), 0) +
            COALESCE(CAST(home_player_10_rating AS DECIMAL), 0) +
            COALESCE(CAST(home_player_11_rating AS DECIMAL), 0)
            ) / 
				(11 - (
				CASE WHEN home_player_1_rating  IS NULL THEN 1 ELSE 0 END +
                CASE WHEN home_player_2_rating  IS NULL THEN 1 ELSE 0 END +
                CASE WHEN home_player_3_rating  IS NULL THEN 1 ELSE 0 END +
                CASE WHEN home_player_4_rating  IS NULL THEN 1 ELSE 0 END +
                CASE WHEN home_player_5_rating  IS NULL THEN 1 ELSE 0 END +
                CASE WHEN home_player_6_rating  IS NULL THEN 1 ELSE 0 END +
                CASE WHEN home_player_7_rating  IS NULL THEN 1 ELSE 0 END +
                CASE WHEN home_player_8_rating  IS NULL THEN 1 ELSE 0 END +
                CASE WHEN home_player_9_rating  IS NULL THEN 1 ELSE 0 END +
                CASE WHEN home_player_10_rating IS NULL THEN 1 ELSE 0 END +
                CASE WHEN home_player_11_rating IS NULL THEN 1 ELSE 0 END
				))
			, 0)
		AS INT)
END,
	away_avg_player_rating =
	CASE WHEN (
		CASE WHEN away_player_1_rating  IS NULL THEN 1 ELSE 0 END +
		CASE WHEN away_player_2_rating  IS NULL THEN 1 ELSE 0 END +
		CASE WHEN away_player_3_rating  IS NULL THEN 1 ELSE 0 END +
		CASE WHEN away_player_4_rating  IS NULL THEN 1 ELSE 0 END +
		CASE WHEN away_player_5_rating  IS NULL THEN 1 ELSE 0 END +
		CASE WHEN away_player_6_rating  IS NULL THEN 1 ELSE 0 END +
		CASE WHEN away_player_7_rating  IS NULL THEN 1 ELSE 0 END +
		CASE WHEN away_player_8_rating  IS NULL THEN 1 ELSE 0 END +
		CASE WHEN away_player_9_rating  IS NULL THEN 1 ELSE 0 END +
		CASE WHEN away_player_10_rating IS NULL THEN 1 ELSE 0 END +
		CASE WHEN away_player_11_rating IS NULL THEN 1 ELSE 0 END
	) = 11 THEN 0  -- All ratings are null, set average as 0
    ELSE
		CAST(
			ROUND(
			(
            COALESCE(CAST(away_player_1_rating  AS DECIMAL), 0) +
            COALESCE(CAST(away_player_2_rating  AS DECIMAL), 0) +
            COALESCE(CAST(away_player_3_rating  AS DECIMAL), 0) +
            COALESCE(CAST(away_player_4_rating  AS DECIMAL), 0) +
            COALESCE(CAST(away_player_5_rating  AS DECIMAL), 0) +
            COALESCE(CAST(away_player_6_rating  AS DECIMAL), 0) +
            COALESCE(CAST(away_player_7_rating  AS DECIMAL), 0) +
            COALESCE(CAST(away_player_8_rating  AS DECIMAL), 0) +
            COALESCE(CAST(away_player_9_rating  AS DECIMAL), 0) +
            COALESCE(CAST(away_player_10_rating AS DECIMAL), 0) +
            COALESCE(CAST(away_player_11_rating AS DECIMAL), 0)
			) /
				(11 - (
				CASE WHEN away_player_1_rating  IS NULL THEN 1 ELSE 0 END +
				CASE WHEN away_player_2_rating  IS NULL THEN 1 ELSE 0 END +
				CASE WHEN away_player_3_rating  IS NULL THEN 1 ELSE 0 END +
				CASE WHEN away_player_4_rating  IS NULL THEN 1 ELSE 0 END +
				CASE WHEN away_player_5_rating  IS NULL THEN 1 ELSE 0 END +
				CASE WHEN away_player_6_rating  IS NULL THEN 1 ELSE 0 END +
				CASE WHEN away_player_7_rating  IS NULL THEN 1 ELSE 0 END +
				CASE WHEN away_player_8_rating  IS NULL THEN 1 ELSE 0 END +
				CASE WHEN away_player_9_rating  IS NULL THEN 1 ELSE 0 END +
				CASE WHEN away_player_10_rating IS NULL THEN 1 ELSE 0 END +
				CASE WHEN away_player_11_rating IS NULL THEN 1 ELSE 0 END
				))
			, 0)
		AS INT)
END;
```

![image](https://github.com/AMichaelDS/SQL/assets/132055953/01c187b5-aa13-44f3-88d4-10ec68f6dbbe)

### B. Total Player Ratings
I used the average player ratings I created to replace the null values for each corresponding team before adding them all up to create total player ratings.

```SQL
ALTER TABLE soc.MatchFin
ADD home_player_total_ratings INT,
    away_player_total_ratings INT;

UPDATE soc.MatchFin
SET home_player_total_ratings =
  COALESCE(home_player_1_rating, home_avg_player_rating) +
  COALESCE(home_player_2_rating, home_avg_player_rating) +
  COALESCE(home_player_3_rating, home_avg_player_rating) +
  COALESCE(home_player_4_rating, home_avg_player_rating) +
  COALESCE(home_player_5_rating, home_avg_player_rating) +
  COALESCE(home_player_6_rating, home_avg_player_rating) +
  COALESCE(home_player_7_rating, home_avg_player_rating) +
  COALESCE(home_player_8_rating, home_avg_player_rating) +
  COALESCE(home_player_9_rating, home_avg_player_rating) +
  COALESCE(home_player_10_rating, home_avg_player_rating) +
  COALESCE(home_player_11_rating, home_avg_player_rating),
  away_player_total_ratings =
  COALESCE(away_player_1_rating, away_avg_player_rating) +
  COALESCE(away_player_2_rating, away_avg_player_rating) +
  COALESCE(away_player_3_rating, away_avg_player_rating) +
  COALESCE(away_player_4_rating, away_avg_player_rating) +
  COALESCE(away_player_5_rating, away_avg_player_rating) +
  COALESCE(away_player_6_rating, away_avg_player_rating) +
  COALESCE(away_player_7_rating, away_avg_player_rating) +
  COALESCE(away_player_8_rating, away_avg_player_rating) +
  COALESCE(away_player_9_rating, away_avg_player_rating) +
  COALESCE(away_player_10_rating, away_avg_player_rating) +
  COALESCE(away_player_11_rating, away_avg_player_rating);
```

![image](https://github.com/AMichaelDS/SQL/assets/132055953/7a587beb-f9c3-4adb-a8b8-ca41e1884c71)

### C. Winning Metric
Then I proceed to create four winning metrics. 

#### First, based on the actual match result
```SQL
ALTER TABLE soc.MatchFin
ADD Winner_Match VARCHAR(10);

UPDATE soc.MatchFin
SET Winner_Match = CASE WHEN home_team_goal > away_team_goal THEN 'Home'
				   WHEN home_team_goal < away_team_goal THEN 'Away'
				   ELSE 'Tied'
				   END;
```

![image](https://github.com/AMichaelDS/SQL/assets/132055953/203f8c3b-6ffa-406a-bd3a-8c86e2f26e00)

#### Secondly, based on the team that has higher ratings
```SQL
ALTER TABLE soc.MatchFin
ADD Winner_Team VARCHAR(10);

UPDATE soc.MatchFin
SET Winner_Team = CASE WHEN home_team_rating > away_team_rating THEN 'Home'
				  WHEN home_team_rating < away_team_rating THEN 'Away'
				  ELSE 'Tied'
				  END;
```

![image](https://github.com/AMichaelDS/SQL/assets/132055953/b24f5c47-af85-4555-b5ee-17200d6b5d06)

#### Thirdly, based on the team that has higher average player ratings
```SQL
ALTER TABLE soc.MatchFin
ADD Winner_Player_Avg VARCHAR(10);

UPDATE soc.MatchFin
SET Winner_Player_Avg = CASE WHEN home_avg_player_rating > away_avg_player_rating THEN 'Home'
						WHEN home_avg_player_rating < away_avg_player_rating THEN 'Away'
						ELSE 'Tied'
						END;
```

![image](https://github.com/AMichaelDS/SQL/assets/132055953/0f066f8f-6e6a-40a8-9f1c-2fe628bcc807)

#### Lastly, based on the team that has higher total player ratings
```SQL
ALTER TABLE soc.MatchFin
ADD Winner_Player_Tot VARCHAR(10);

UPDATE soc.MatchFin
SET Winner_Player_Tot = CASE WHEN home_player_total_ratings > away_player_total_ratings THEN 'Home'
						WHEN home_player_total_ratings < away_player_total_ratings THEN 'Away'
						ELSE 'Tied'
						END;
```

![image](https://github.com/AMichaelDS/SQL/assets/132055953/84419f4e-e0fe-4a31-b571-712fd5c992a3)

### D. Comparison for Each Metric
#### Here I compare the accuracy for each metric with the actual match results. The first query shows the overall result
```SQL
SELECT TeamOccurrences, PlayerAvgOccurrences, PlayerTotOccurrences, TotalOccurrences,
    CAST((CAST(TeamOccurrences AS decimal) / TotalOccurrences) * 100 AS decimal(10,2)) AS TeamPercentage,
    CAST((CAST(PlayerAvgOccurrences AS decimal) / TotalOccurrences) * 100 AS decimal(10,2)) AS PlayerAvgPercentage,
    CAST((CAST(PlayerTotOccurrences AS decimal) / TotalOccurrences) * 100 AS decimal(10,2)) AS PlayerTotPercentage
FROM    (SELECT
        (SELECT COUNT(*) FROM soc.MatchFin WHERE Winner_Team = Winner_Match) AS TeamOccurrences,
        (SELECT COUNT(*) FROM soc.MatchFin WHERE Winner_Player_Avg = Winner_Match) AS PlayerAvgOccurrences,
        (SELECT COUNT(*) FROM soc.MatchFin WHERE Winner_Player_Tot = Winner_Match) AS PlayerTotOccurrences,
        (SELECT COUNT(*) FROM soc.MatchFin) AS TotalOccurrences) AS Counts;
```

![image](https://github.com/AMichaelDS/SQL/assets/132055953/2578d91d-ee9d-474a-a9c1-18c8d4da4dd5)

We can see that the total player ratings has 50.08 % to correctly predict the outcome of the match. 

#### The second query only shows matches that are not tied. It is done because the probability of having the same team, average player, and total ratings between the home and away teams would be tiny.
```SQL
SELECT TeamOccurrences, PlayerAvgOccurrences, PlayerTotOccurrences, TotalOccurrences,
    CAST((CAST(TeamOccurrences AS decimal) / TotalOccurrences) * 100 AS decimal(10,2)) AS TeamPercentage,
    CAST((CAST(PlayerAvgOccurrences AS decimal) / TotalOccurrences) * 100 AS decimal(10,2)) AS PlayerAvgPercentage,
    CAST((CAST(PlayerTotOccurrences AS decimal) / TotalOccurrences) * 100 AS decimal(10,2)) AS PlayerTotPercentage
FROM    (SELECT
        (SELECT COUNT(*) FROM soc.MatchFin WHERE Winner_Team = Winner_Match AND Winner_Team <> 'Tied' AND Winner_Match <> 'Tied') AS TeamOccurrences,
        (SELECT COUNT(*) FROM soc.MatchFin WHERE Winner_Player_Avg = Winner_Match AND Winner_Player_Avg <> 'Tied' AND Winner_Match <> 'Tied') AS PlayerAvgOccurrences,
        (SELECT COUNT(*) FROM soc.MatchFin WHERE Winner_Player_Tot = Winner_Match AND Winner_Player_Tot <> 'Tied' AND Winner_Match <> 'Tied') AS PlayerTotOccurrences,
        (SELECT COUNT(*) FROM soc.MatchFin WHERE Winner_Match <> 'Tied') AS TotalOccurrences) AS Counts;
```

![image](https://github.com/AMichaelDS/SQL/assets/132055953/d3398703-670d-45a8-ae30-9fd035a73561)

We can see that the total player ratings has 65.87 % to correctly predict the outcome of the match. 

#### The third one only shows tied matches, used only to validate the previous result.
```SQL
SELECT TeamOccurrences, PlayerAvgOccurrences, PlayerTotOccurrences, TotalOccurrences,
    CAST((CAST(TeamOccurrences AS decimal) / TotalOccurrences) * 100 AS decimal(10,2)) AS TeamPercentage,
    CAST((CAST(PlayerAvgOccurrences AS decimal) / TotalOccurrences) * 100 AS decimal(10,2)) AS PlayerAvgPercentage,
    CAST((CAST(PlayerTotOccurrences AS decimal) / TotalOccurrences) * 100 AS decimal(10,2)) AS PlayerTotPercentage
FROM    (SELECT
        (SELECT COUNT(*) FROM soc.MatchFin WHERE Winner_Team = Winner_Match AND Winner_Match = 'Tied') AS TeamOccurrences,
        (SELECT COUNT(*) FROM soc.MatchFin WHERE Winner_Player_Avg = Winner_Match AND Winner_Match = 'Tied') AS PlayerAvgOccurrences,
        (SELECT COUNT(*) FROM soc.MatchFin WHERE Winner_Player_Tot = Winner_Match AND Winner_Match = 'Tied') AS PlayerTotOccurrences,
        (SELECT COUNT(*) FROM soc.MatchFin WHERE Winner_Match = 'Tied') AS TotalOccurrences) AS Counts;
```

![image](https://github.com/AMichaelDS/SQL/assets/132055953/994c3219-c37f-40af-9e4f-a39a9050367f)

We can see that player average rating can predict the outcome of tied match with 12.39% accuracy.

### E. Home & Away Analysis
Lastly, I analysed the team performance based on the home and away match to determine the winning percentage. 
First, I combined the winner for both home and away matches grouped by the team, in which I could find the number of wins per team based on their home and away matches. 
I also combined the home and away matches grouped by the team to get the total of matches.
Then I combined those two queries based on the team so that I could calculate the winning percentage.

```SQL
DROP TABLE IF EXISTS soc.Analysis_Home_Away;

SELECT	
  t.Team,
  t.TotalHomeWins,
  t.TotalAwayWins,
  t.TotalWins,
  m.TotalHomeMatches,
  m.TotalAwayMatches,
  m.TotalMatches AS TotalMatches,
  CAST((T.TotalHomeWins / CAST(M.TotalHomeMatches AS decimal)) * 100 AS DECIMAL(10, 1)) AS TotalHomeWinPercentage,
  CAST((T.TotalAwayWins / CAST(M.TotalAwayMatches AS decimal)) * 100 AS DECIMAL(10, 1)) AS TotalAwayWinPercentage,
  CAST((T.TotalWins / CAST(M.TotalMatches AS decimal)) * 100 AS DECIMAL(10, 1)) AS TotalWinPercentage
INTO soc.Analysis_Home_Away
FROM
(
    SELECT
      Team,
      COUNT(Winner_Match) AS TotalWins,
      SUM(CASE WHEN Winner_Match = 'Home' THEN 1 ELSE 0 END) AS TotalHomeWins,
      SUM(CASE WHEN Winner_Match = 'Away' THEN 1 ELSE 0 END) AS TotalAwayWins
    FROM
    (
        SELECT home_team AS Team, Winner_Match
        FROM soc.MatchFin
        WHERE Winner_Match = 'Home'

        UNION ALL

        SELECT away_team AS Team, Winner_Match
        FROM soc.MatchFin
        WHERE Winner_Match = 'Away'
    ) AS CombinedTeams
    GROUP BY Team
) AS T
INNER JOIN
(
    SELECT
      Team,
      SUM(CASE WHEN IsHome = 1 THEN MatchCount ELSE 0 END) AS TotalHomeMatches,
      SUM(CASE WHEN IsHome = 0 THEN MatchCount ELSE 0 END) AS TotalAwayMatches,
      SUM(MatchCount) AS TotalMatches
    FROM
    (
        SELECT home_team AS Team, COUNT(*) AS MatchCount, 1 AS IsHome
        FROM soc.MatchFin
        GROUP BY home_team

        UNION ALL

        SELECT away_team AS Team, COUNT(*) AS MatchCount, 0 AS IsHome
        FROM soc.MatchFin
        GROUP BY away_team
    ) AS MatchCounts
    GROUP BY Team
) AS M ON T.Team = M.Team
ORDER BY TotalWinPercentage DESC;
```

![image](https://github.com/AMichaelDS/SQL/assets/132055953/02ddb571-f595-4df0-81bd-a15843985e19)
![image](https://github.com/AMichaelDS/SQL/assets/132055953/9757782d-bab5-40ab-8ae2-63a2ef361ccf)


## 6. Conclusion
In conclusion, this project demonstrates my skills and expertise in utilizing SQL queries for data analysis and manipulation. The project aimed to showcase my proficiency in working with SQL databases, data cleaning and preparation, exploratory data analysis, and generating meaningful insights.

Throughout the project, I effectively described and prepared the data, ensuring its quality and reliability for analysis. I performed data pre-processing tasks such as removing duplicates, eliminating null values, integrating and transforming the data, cleaning it, and conducting feature engineering to enhance its usefulness.

The exploratory data analysis section provided valuable insights into various aspects of the dataset. I analyzed different dimensions such as country and league, team, and player, uncovering patterns and trends that contribute to a better understanding of the data. This analysis included metrics like average player ratings, total player ratings, winning metrics, and comparisons for each metric, offering comprehensive insights into team performance and player statistics. Additionally, the analysis of home and away matches provided further understanding of performance variations in different settings.

By leveraging SQL queries, I effectively manipulated and analyzed the complex data, enabling me to draw meaningful conclusions. The project demonstrates my ability to derive insights from data and utilize them for strategic decision-making, predictive modeling, and performance evaluation.

Overall, this project showcases my competence in SQL, data analysis, and problem-solving. It highlights my capability to work with real-world datasets, clean and preprocess data, perform exploratory analysis, and generate valuable insights. The skills demonstrated in this project are applicable across various industries and can contribute to data-driven decision-making and actionable recommendations.


## 7. Appendix

The appendix includes SQL code snippets, a database schema diagram, and references used in this project.

![image](https://github.com/AMichaelDS/SQL/assets/132055953/5f4cfeab-8f1a-4f64-b6d8-ab5bcbb1dafb)

![image](https://github.com/AMichaelDS/SQL/assets/132055953/20dd32b3-9fcc-472c-aed9-9a0eb9aa5ae9)

![image](https://github.com/AMichaelDS/SQL/assets/132055953/79d1704e-da81-482f-af2b-8b969d6a0b8b)

![image](https://github.com/AMichaelDS/SQL/assets/132055953/cdf82add-61c7-41a9-8d10-bada4268ac62)

![image](https://github.com/AMichaelDS/SQL/assets/132055953/409d1ce8-ddc3-46f7-b0d2-dd746d0c2113)





