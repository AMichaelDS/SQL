# European Soccer Data Preparation and Analysis

## Table of Contents

1. [Introduction](#1-introduction)
2. [Data Description and Preparation](#2-data-description-and-preparation)
3. [Exploratory Data Analysis](#3-exploratory-data-analysis)
4. [Data Analysis and Complex Data Manipulation](#4-data-analysis-and-complex-data-manipulation)
5. [Reporting and Visualisation](#5-reporting-and-visualization)
6. [Conclusion](#6-conclusion)
7. [Appendix](#7-appendix)

---

## 1. Introduction

This portfolio showcases my skills in using SQL queries to perform complex data manipulation and analysis. This project aims to demonstrate my ability to work with SQL databases, clean and prepare data, perform exploratory data analysis, and generate meaningful insights. 
This project analyses European soccer data to gain insight into team performance, player statistics and predict match outcomes. The query is performed using the Microsoft SQL Server Management Studio.

## 2. Data Description and Preparation
### Dataset
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

After getting the dataset in a CSV form, I imported the file into the SSMS, determining the appropriate data type and removing the irrelevant column. Some were done manually, and some were done using dynamic SQL script. I have also selected each table's primary and foreign keys while establishing their relationships.

### Schema
For this project, I created a schema so that it can be easily distinguished from the other dataset. The schema name I picked is soc.

### Data Pre-Processing

Before analysing the dataset, it is important to pre-process the data to ensure that it is clean, consistent, and ready for analysis. 

#### Removing Duplicates

Since there is 2 ID for the player and team, which is the api_id and fifa_id, I checked whether there is only one api_id for each fifa_id and vice versa.
- Team (team & team_attributes)
```SQL
;
```

Then I try to find the column that causes the duplicate

```SQL
;
```
I remove the duplicate team by choosing only the first occurrence. 
```SQL
;
```

- Player (player & player_attributes)
```SQL
;
```
I did not remove any duplicates here since the duplicate only exists on the player_attributes table. So later, by inner-joining the table, it will be eliminated by itself.

#### Eliminating Null Values

In the team_attributes table, the buildUpPlayDribbling column contains null value. This will become an issue later, so I eliminate the null values by changing them to the average for that column.
```SQL
;
```

#### Data Integration and Transformation

After completing all the previous sections, I joined the relevant table to create a unified dataset so that analysis could be easily performed later. I also transformed the date to YYYYMMDD format.
- Country & League
```SQL
;
```

- Team(team & team_attributes)
```SQL
;
```

- Player(player & player_attributes)
```SQL
;
```

- Match(Match & All Tables)
Here I substituted the ID for each country, league, player, and team with the name to make it easier to look at.
```SQL
;
```

#### Data Cleaning

In the PlayerFin table,  the column attacking_work_rate and defensive_work_rate has some irrelevant values, which I converted to null using a case-when statement.

```SQL
;
```

#### Feature Engineering

For the TeamFin table, I noticed that there is no overall rating, which is essential because having an overall rating allows us to evaluate and compare the performance of different soccer teams more comprehensively.

I summed the value for all the metrics and divided it by 9. Initially, I dealt with the null value by simply changing the denominator. But then, I decided to eliminate the null value earlier in the data pre-processing phase.

```SQL
;
```


## 3. Exploratory Data Analysis
After finishing the data preparation, we can now start analysing the dataset. 

### Analysis on Country and League

I created the sum and average for each country and league to analyse which country/league is more dominant. 
```SQL
;
```
Here we can see that Spain Liga BBVA has the highest total goal.

### Analysis on TeamFin

To analyse this table, I picked only the latest assessment date for each team by using this query:
```SQL
;
```

Then I removed the duplicate team/observation by selecting only the first occurrence for each team name.
```SQL
;
```

- Highest Overall Team Rating
I selected ten teams with the highest rating
```SQL
;
```

- Numerical Analysis 
I used a dynamic SQL script to create tables containing team with the highest performance metrics for each numerical attribute.
```SQL
;
```

- Categorical Analysis
Here I performed a categorical analysis to find the distribution of teams across the categorical attribute. It is done to get a relative representation of teams in different categories within each attribute.

```SQL
;
```

### Analysis on PlayerFin

Like what I did for TeamFin, I picked only the latest assessment date for each player.
```SQL
;
```

- Highest Overall Team Rating
I selected ten players with the highest rating
```SQL
;
```

- Highest For Each Rating
I use dynamic SQL script to create tables containing player with the highest rating for each attribute. 
```SQL
;
```

Then I use another script to remove irrelevant table and table that has categorical attribute
```SQL
;
```

- Categorical Analysis
Here I performed a categorical analysis to find the distribution of players across the categorical attribute. It is done to get a relative representation of players in different categories within each attribute.

```SQL
;
```

## 4. Data Analysis and Complex Data Manipulation

### Average Player Ratings

For MatchFin, I created an average rating of players for each team(home & away).  
First, it counted the number of players on the team with null value and summed it up. For teams where all the player rating is null, the average becomes 0. It is done to avoid dividing by 0 later on.
Then I sum the ratings of the player then divide it by the number of players that did not have a null value.

```SQL
;
```

### Total Player Ratings

I used the average player ratings I created to replace the null values for each corresponding team before adding them all up to create total player ratings.
```SQL
;
```

### Winning Metric

Then I proceed to create four winning metrics. 
- First, based on the actual match result

```SQL
;
```
- Secondly, based on the team that has higher ratings

```SQL
;
```
- Thirdly, based on the team that has higher average player ratings

```SQL
;
```
- Lastly, based on the team that has higher total player ratings

```SQL
;
```

### Comparison for Each Metric

Here I compare the accuracy for each metric with the actual match results. The first query shows the overall result
```SQL
;
```

The second query only shows matches that are not tied. It is done because the probability of having the same team, average player, and total ratings between the home and away teams would be tiny.
```SQL
;
```

The third one only shows tied matches, used only to validate the previous result.
```SQL
;
```

### Home & Away Analysis

Lastly, I analysed the team performance based on the home and away match to determine the winning percentage. 

First, I combined the winner for both home and away matches grouped by the team, in which I could find the number of wins per team based on their home and away matches. 

I also combined the home and away matches grouped by the team to get the total of matches.

Then I combined those two queries based on the team so that I could calculate the winning percentage.

```SQL
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






