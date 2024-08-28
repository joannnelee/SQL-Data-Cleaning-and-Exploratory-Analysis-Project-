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
5. Remove any columns and rows that aren't necessary.

Code:
1. Removing Duplicates
```sql
SELECT *
FROM layoffs;

-- Creating staging table by copying data

CREATE TABLE layoffs_staging 
LIKE layoffs;

INSERT layoffs_staging
SELECT *
FROM layoffs;

-- 1. Remove duplicates

#checking for duplicates
SELECT *,
ROW_NUMBER() OVER(
PARTITION BY company, industry, total_laid_off, percentage_laid_off, `date`) AS row_num
FROM layoffs_staging;

-- making CTE 

WITH duplicate_cte AS 
(
SELECT *,
ROW_NUMBER() OVER(
PARTITION BY company, industry, total_laid_off, percentage_laid_off, `date`) AS row_num
FROM layoffs_staging
)
SELECT *
FROM duplicate_cte 
WHERE row_num >1;		

#to confirm they are duplicates check company 'oda' 

SELECT *
FROM layoffs_staging
WHERE company = 'Oda';		

-- No duplicates so we partition using all columns 

SELECT *,
ROW_NUMBER() OVER(
PARTITION BY company, location, industry, total_laid_off, percentage_laid_off, `date`,stage, country, funds_raised_millions) AS row_num
FROM layoffs_staging;

-- making updated CTE 

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

#check company 'Casper' now 

SELECT *
FROM layoffs_staging
WHERE company = 'Casper'; 	#there is a duplicate so we want to remove 1 row 


-- CTE put into a stage 2 database where duplicates can be deleted 
-- making table with extra rows and deleting it where the row is = 2 

CREATE TABLE `layoffs_staging2` (
  `company` text,
  `location` text,
  `industry` text,
  `total_laid_off` text,
  `percentage_laid_off` text,
  `date` text,
  `stage` text,
  `country` text,
  `funds_raised_millions` text,
  `row_num` INT
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci;

SELECT *
FROM layoffs_staging2;

-- inserting data into empty table but with new column 'row_num' 

INSERT INTO layoffs_staging2
SELECT *,
ROW_NUMBER() OVER(
PARTITION BY company,location, industry, total_laid_off, percentage_laid_off, `date`, stage, country, funds_raised_millions) AS row_num
FROM layoffs_staging;

SELECT *
FROM layoffs_staging2
WHERE row_num >1;

-- deleting duplicates where row number > 1 

DELETE 
FROM layoffs_staging2
WHERE row_num > 1;

#run this again to check duplicates removed 
SELECT *
FROM layoffs_staging2
WHERE row_num >1;
```

2. Standardising Data
```sql
#some rows have spaces in the beginning of company names so we trim these 

SELECT company, TRIM(company)
FROM layoffs_staging2;

UPDATE layoffs_staging2
SET company = TRIM(company);

-- now looking at industry 
SELECT DISTINCT industry
FROM layoffs_staging2
ORDER BY 1;

#multiple crypto industries, we want to group these so they are all under the same industry 

SELECT *
FROM layoffs_staging2
WHERE industry LIKE 'Crypto%';

#want to update all to 'crypto'

UPDATE layoffs_staging2
SET industry = 'Crypto'
WHERE industry LIKE 'Crypto%';

#now they are all the same industry, if we run SELECT DISTINCT industry, we can see it's updated

-- now looking at location 

SELECT DISTINCT location
FROM layoffs_staging2
ORDER BY 1;

#location looks okay

-- now looking at country
SELECT DISTINCT country
FROM layoffs_staging2
ORDER BY 1;

#there is a United States with a '.' at the end of it, we want to remove this 

SELECT DISTINCT *
FROM layoffs_staging2
WHERE country LIKE 'United States%'
ORDER BY 1;

SELECT DISTINCT country, TRIM(TRAILING '.' FROM country) 	
FROM layoffs_staging2
ORDER BY 1;

UPDATE layoffs_staging2
SET country = TRIM(TRAILING '.' FROM country)
WHERE country LIKE 'United States%';

SELECT * 
FROM layoffs_staging2;

-- time series, our 'date' column is a text which is not good for time series 
-- We want to change this to a date column rather than text 

UPDATE layoffs_staging2
SET `date` = NULL
WHERE `date` = 'NULL';

SELECT `date`,
STR_TO_DATE(`date`, '%m/%d/%Y')	
FROM layoffs_staging2;

#changes the date to a standardised format 

UPDATE layoffs_staging2
SET `date` = STR_TO_DATE(`date`, '%m/%d/%Y');

SELECT `date`
FROM layoffs_staging2;

#changed to the date format but it's still a text so change to date format 

ALTER TABLE layoffs_staging2
MODIFY COLUMN `date` DATE;
```

3. Null and Blank Values

```sql
SELECT *
FROM layoffs_staging2
WHERE total_laid_off IS NULL
AND percentage_laid_off IS NULL;

SELECT *
FROM layoffs_staging2
WHERE industry IS NULL
OR industry = '';

#airbnb has a blank in 'industry' column, let's look at this

SELECT *
FROM layoffs_staging2 
WHERE company = 'Airbnb';

#we know airbnb is in travel when we run this so we can populate this blank with travel 


#we join the non-blank to the blank to update this information 

SELECT t1.industry, t2.industry
FROM layoffs_staging2 t1
JOIN layoffs_staging2 t2
	ON t1.company = t2.company 	
WHERE (t1.industry IS NULL OR t1.industry = '')
AND t2.industry IS NOT NULL;

#now we write update statement 

UPDATE layoffs_staging2 t1
JOIN layoffs_staging2 t2
	ON t1.company = t2.company
SET t1.industry = t2.industry 
WHERE t1.industry IS NULL
AND t2.industry IS NOT NULL;

#update blanks to NULL before updating data 

UPDATE layoffs_staging2
SET industry = NULL
WHERE industry = '';

UPDATE layoffs_staging2
SET industry = NULL
WHERE industry = 'NULL';

#Company 'Bally%' still has a blank but when we look into this there is not another populated row 
SELECT * 
FROM layoffs_staging2
WHERE company LIKE 'Bally%';

-- that's all we can do for nulls and blanks 
```

4. Removing columns and rows

```sql
-- Looking at companies with a NULL in layoffs this may mean that they had no layoffs so these rows are removed 

DELETE
FROM layoffs_staging2
WHERE total_laid_off IS NULL
AND percentage_laid_off IS NULL;

#we can now remove the row_num column bc we dont need this 

ALTER TABLE layoffs_staging2 
DROP COLUMN row_num;

-- now we have our finalised clean data that we can analyse 
```

### Exploratory Data Analysis

EDA involved exploring the layoffs data to find key insights, such as:

- The maximum number of layoffs and the highest percentage of layoffs
- Companies with the most layoffs and the size of these companies.
- The time period of the layoffs.
- Rolling total of layoffs per month.
- The top 5 companies with the most layoffs each year.

Code:

```sql
SELECT *
FROM layoffs_staging2;

ALTER TABLE layoffs_staging2
MODIFY COLUMN total_laid_off INT;

ALTER TABLE layoffs_staging2
MODIFY COLUMN funds_raised_millions INT;

-- Looking at the maximum number of layoffs and the highest percentage of layoffs 
SELECT MAX(total_laid_off), MAX(percentage_laid_off)
FROM layoffs_staging2;

-- Which companies had 1, meaning 100% of the company was laid off
SELECT *
FROM layoffs_staging2
WHERE percentage_laid_off = 1;
-- mostly start-up companies that went out of business during this time 

-- ordering by funds_raised_millions to see how big these companies were 
SELECT *
FROM layoffs_staging2
WHERE percentage_laid_off = 1
ORDER BY funds_raised_millions DESC;
-- big companies like BritishVolt and Quibi with 100% layoffs 


-- Companies with the biggest single layoff

SELECT company, total_laid_off
FROM layoffs_staging2
ORDER BY 2 DESC
LIMIT 5;
-- these layoffs were on a single day 

-- companies with the most total layoffs
SELECT company, SUM(total_laid_off)
FROM layoffs_staging2
GROUP BY company
ORDER BY 2 DESC
LIMIT 10;		

-- looking at the period of the earliest and latest layoffs 
SELECT MIN(`date`), MAX(`date`)
FROM layoffs_staging2;
-- started in 2020 to early 2023 

-- by location
SELECT location, SUM(total_laid_off)
FROM world_layoffs.layoffs_staging2
GROUP BY location
ORDER BY 2 DESC
LIMIT 10;

-- looking at layoffs by country and the sum of total layoffs by year
SELECT country, SUM(total_laid_off)
FROM layoffs_staging2
GROUP BY country
ORDER BY 2 DESC;

SELECT YEAR(`date`), SUM(total_laid_off)
FROM layoffs_staging2
GROUP BY YEAR(`date`)
ORDER BY 1 DESC;

-- the stage the companies were at during layoffs 
SELECT stage, SUM(total_laid_off)
FROM layoffs_staging2
GROUP BY stage
ORDER BY 2 DESC;

-- percentage laid off not very relevant because we don't know the total amount of employees 
SELECT company, SUM(percentage_laid_off)
FROM layoffs_staging2
GROUP BY company
ORDER BY 2 DESC;


-- rolling total of layoffs per month

SELECT SUBSTRING(`date`,1,7) AS `MONTH`, SUM(total_laid_off)
FROM layoffs_staging2
WHERE SUBSTRING(`date`,1,7) IS NOT NULL
GROUP BY `MONTH`
ORDER BY 1 ASC
;

WITH Rolling_Total AS
(
SELECT SUBSTRING(`date`,1,7) AS `MONTH`, SUM(total_laid_off) AS total_off
FROM layoffs_staging2
WHERE SUBSTRING(`date`,1,7) IS NOT NULL
GROUP BY `MONTH`
ORDER BY 1 ASC
)
SELECT `MONTH`, total_off
,SUM(total_off) OVER(ORDER BY `MONTH`) AS rolling_total
FROM Rolling_Total;

-- look at companies with the most layoffs per year 

SELECT company, SUM(total_laid_off)
FROM layoffs_staging2
GROUP BY company
ORDER BY 2 DESC;

-- want to rank  which years they laid off the most employees
SELECT company,YEAR(`date`), SUM(total_laid_off)
FROM layoffs_staging2
GROUP BY company, YEAR(`date`)
ORDER BY 3 DESC;


WITH Company_Year (company, years, total_laid_off) AS
(
SELECT company,YEAR(`date`), SUM(total_laid_off)
FROM layoffs_staging2
GROUP BY company, YEAR(`date`)
), Company_Year_Rank AS
(
SELECT *, 
DENSE_RANK() OVER (PARTITION BY years ORDER BY total_laid_off DESC) AS ranking
FROM Company_Year
WHERE years IS NOT NULL
)
SELECT *
FROM Company_Year_Rank
WHERE ranking <= 5
;

```

### Results/Findings

The analysis results are summarised as follows:
1. Companies with 100% layoffs tend to be start-ups.


### Limitations 

Some rows had a null value for layoffs. It is uncertain whether these companies had no layoffs or if the data was missing. These rows were removed which may lead to inaccuracies in the analysis. 

### References 

1. [Layoffs data](https://www.kaggle.com/datasets/swaptr/layoffs-2022)


