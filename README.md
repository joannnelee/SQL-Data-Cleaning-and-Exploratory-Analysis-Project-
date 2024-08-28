# 1. Data Cleaning 

## Table of Contents

- [Project Overview](#project-overview)
- [Data Source](data-source)
- [Tools](tools)
- [Data Cleaning](data-cleaning)
- [Exploratory Data Analysis](exploratory-data-analysis)
- [Results/Findings](results/findings)
- [Limitations](limitations)


### Project Overview 

In this project, company layoffs from 2020 to 2023 will be investigated. First, the data will be cleaned and then the data will be explored to find trends, patterns, or any other interesting insights.  

### Data Sources 

World Layoffs: The primary dataset used for this analysis is the "layoffs.csv" file, containing detailed information about company layoffs around the world. 

### Tools

- SQL Server - Data Cleaning and Analysis


### Data Cleaning 

In the initial data preparation phase, we performed the following tasks: 
1. Data loading and inspection.
2. Removal of duplicates.
3. Standardization of data.
4. Investigated null and blank values.
5. Revoed any columns and rows that aren't necessary.

### Exploratory Data Analysis

EDA involved exploring the layoffs data to find key insights, such as:

- The maximum number of layoffs and the highest percentage of layoffs
- Companies with the most layoffs and the size of these companies.
- The time period of the layoffs.
- Rolling total of layoffs per month.
- The top 5 companies with the most layoffs each year.

Interesting code/features worked with

```sql
WITH duplicate_cte AS 
(
SELECT *,
ROW_NUMBER() OVER(
PARTITION BY company,location, industry, total_laid_off, percentage_laid_off, `date`, stage, country, funds_raised_millions) AS row_num
FROM layoffs_staging
)
SELECT *
FROM duplicate_cte 
WHERE row_num >1;	
```

### Results/Findings

The analysis results are summarised as follows:
1. Companies with 100% layoffs tend to be start-ups.


### Limitations 

Some rows had a null value for layoffs. It is uncertain whether these companies had no layoffs or if the data was missing. These rows were removed which may lead to inaccuracies in the analysis. 

### References 

1. [Layoffs data](https://www.kaggle.com/datasets/swaptr/layoffs-2022)


