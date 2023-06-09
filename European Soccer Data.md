# European Soccer Data Preparation and Analysis

## Table of Contents

1. [Introduction](#1-introduction)
2. [Data Description and Preparation](#2-data-description-and-preparation)
3. [Exploratory Data Analysis](#3-exploratory-data-analysis)
4. [Data Analysis and Complex Data Manipulation](#4-data-analysis-and-complex-data-manipulation)
5. [Reporting and Visualization](#5-reporting-and-visualization)
6. [Conclusion](#6-conclusion)
7. [Appendix](#7-appendix)

---

## 1. Introduction

This portfolio showcases my skills in using SQL queries to perform complex data manipulation and analysis. The objective of this project is to demonstrate my ability to work with SQL databases, clean and prepare data, perform exploratory data analysis, and generate meaningful insights. 
This project focuses on analyzing European soccer data to gain insight into team performance, player statistics, and predict match outcomes. The query is performed using the Microsoft SQL Server Management Studio

## 2. Data Description and Preparation
### Dataset
The dataset that I am using is the european soccer data set obtained from [Kaggle](https://www.kaggle.com/datasets/hugomathien/soccer). It contains 7 tables with the following description:

| Table Name         | Description                                                                | Number Of Attributes |
|--------------------|----------------------------------------------------------------------------|----------------------|
| Country            | Contains country ID and country name                                       | 2                    |
| Leagues            | Contains league ID and league name for each country                        | 3                    |
| Matches            | Contains details about each match, including teams, players, and scores    | 115                  |
| Players            | Includes data on individual players, such as birthday, height, and weight  | 7                    |
| Player_Attributes  | Includes various player assessment metrics, such as the overall rating     | 42                   |
| Teams              | Provides information about the teams                                       | 5                    |
| Team_Attributes    | Provides various team assessment metrics                                   | 25                   |

After getting the dataset in a CSV form, I imported the file into the SSMS, determining the appropriate data type and removing the irrelevant column. Some are done manually and some were done using dynamic SQL script. I have also selected the primary and foreign keys for each table while establishing the relationships between them.

### Schema
For this project I created a schema so that it can be easily distinguished with the other dataset. The schema name I picked is soc.

### Data Pre-Processing

Before doing any analysis on the dataset, it is important to preprocess the data to ensure that the data is clean, consistent, and ready for analysis. 

**Removing Duplicates**

Since there is 2 ID for the player and team which is the api_id and fifa_id, I checked whether there is only 1 api_id for each fifa_id and vice versa.
- Team (team & team_attributes)
```sql
;
```
Resulting in:

	Then I try to find the column that causes the duplicate
```sql
;
```
In which I remove the duplicate team by choosing the only the first occurence. 
```sql
;
```

- Player (player & player_attributes)
```sql
;
```
In here I did not remove any duplicate since there is only duplicate on the player_attributes table.  So later by inner-joining the table it will be eliminated by itself.

**Eliminating Null Values**

In the team_attributes table, the buildUpPlayDribbling column contains null value. This will become an issue later on so I eliminate the null values by changing it to the average for that column.
```sql
;
```

**Data Integration and Transformation**

After completing all the previous section, I joined the relevant table to create a unified dataset so that analysis can be easily performed later. I also transformed the date to YYYYMMDD format.
- Country & League
```sql
;
```

- Team(team & team_attributes)
```sql
;
```

- Player(player & player_attributes)
```sql
;
```

- Match(Match & All Tables)
Here I substituted the ID for each country, league, player, and team with the name so that it will be easier to look at.
```sql
;
```

**Data Cleaning**

In the PlayerFin table,  the column attacking_work_rate and defensive_work_rate has some values that is irrelevant in which I converted it to null by using case when statement.

```sql
;
```

**Feature Engineering**
For the TeamFin table, I noticed that there is no overall_rating in it which I think is important so we can perform analysis on it because having an overall rating allows us to evaluate and compare the performance of different soccer teams in a more comprehensive way.

Here I summed the value for all the metrics and dividing it by 9. In the beginning I choose to deal with the null value by simply changing the denominator. But then I decided to eliminate the null value earlier in the data preprocessing phase.

```sql
;
```


## 3. Exploratory Data Analysis
After finishing the data preparation now we can start doing analysis on the dataset. 

**Analysis on Country and League**
Here I created the sum and average for each country and league to analyse which country/league is more dominant. 
```sql
;
```
In here we can see that Spain Liga BBVA has the highest number of total goal

**Analysis on TeamFin**
In order to do analysis on this table, I picked only the latest assessment date for each team by using this query:
```sql
;
```

Then I removed the duplicate team/observation by selecting only the first occurence for each team name.
```sql
;
```

- Highest Overall Team Rating
I selected 10 teams with the highest rating
```sql
;
```

- Numerical Analysis 
I used dynamic SQL script to create table containing team with the highest performance metrics for each numerical attributes.
```sql
;
```

- Categorical Analysis
Here I performed a categorical analysis to find the distribution of team across the categorical attribute. This is done to get a relative representation of teams in different categories within each attribute.

```sql
;
```

**Analysis on PlayerFin**
Like what I did for TeamFin, I picked only the latest assessment date for each player.
```sql
;
```

- Highest Overall Team Rating
I selected 10 players with the highest rating
```sql
;
```

- Highest For Each Rating
I use dynamic SQL script to create table containing player with the highest rating for each attribute. 
```sql
;
```

	Then I use another script to remove irrelevant table and table that has categorical attribute
```sql
;
```

- Categorical Analysis
Here I performed a categorical analysis to find the distribution of player across the categorical attribute. This is done to get a relative representation of players in different categories within each attribute.

```sql
;
```

## 4. Data Analysis and Complex Data Manipulation
**Average Player Ratings**
For MatchFin, I created an average rating of player for each team(home & away).  
First it counted the number of players on the team that has null value and sum it up. For team where all players rating is null the average become null. This is done to avoid dividing by 0 later.
Then I sum the ratings of the player then divide it by the number of players that does not have null value.

```sql
;
```

**Total Player Ratings**
I used the average player ratings that i have created before to replace the null values for each corresponding team before adding them all up to create total player ratings.
```sql
;
```

**Winning Metric**
Then I proceed to create 4 winning metrics. 
- First based on actual match result

```sql
;
```
- Secondly based on the team that has higher ratings

```sql
;
```
- Thirdly based on the team that has higher average player ratings

```sql
;
```
- Lastly based on the team that has higher total player ratings

```sql
;
```

**Comparison for Each Metrics**
Here I compare the accuracy for each metrics with the actual match results. The first query shows the overall result
```sql
;
```

The second query only show matches that is not tied, this is done because the probability of team having the same team, average player, and total ratings would be really small.
```sql
;
```

The third one only show matches that is tied, used only as a validation for the previous result.
```sql
;
```

**Home & Away Analysis**
Lastly, I analysed the performance of team performance based on the home and away matches to determine the winning percentage. 

First I combined the winner for both home and away matches grouped by the team, in which I am able to find the number of win per team based on their home and away matches. 

I also combined the home and away matches grouped by team to get the total of matches.

Then I combined those 2 queries based on the team so that I can calculate the winning percentage.

```sql
;
```

## 5. Reporting and Visualization
- Generate comprehensive reports and create interactive dashboards using SQL queries and visualization tools.
- Present the key findings and insights from the data analysis.

## 6. Conclusion
- Summarize the project and its accomplishments.
- Reflect on the insights gained and their implications.
- Discuss the strengths and limitations of your analysis.
- Highlight the lessons learned during the process and suggest future enhancements or areas for further exploration.


## 7. Appendix

The appendix includes SQL code snippets, a database schema diagram, and references used in this project.






