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

For this project I created a schema so that it can be easily distinguished with the other dataset. The schema name I picked is soc.

Data preprocessing steps such as handling missing values, removing duplicates, and performing data validation and quality checks were applied.

## 3. Exploratory Data Analysis
- Explore the dataset using SQL queries to gain insights and understand the data better.
- Calculate descriptive statistics, aggregate and summarize data.
- Visualize the results using appropriate charts and graphs.

## 4. Data Analysis and Complex Data Manipulation
- Present the results of your SQL queries.
- Provide visualizations or summaries to enhance understanding.
- Interpret and discuss the insights and patterns you discovered.
- Demonstrate advanced SQL techniques for complex data manipulation.
- Work with subqueries, joins, and table relationships to extract meaningful information from the data.
- Showcase more complex SQL queries developed.
- Explore additional analysis or insights beyond the basic queries.

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






