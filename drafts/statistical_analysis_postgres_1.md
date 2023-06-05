# Data Analysis in PostgreSQL - I

## Introduction

For any real-life dataset, it is useful to have an awareness of the basic features of the data. Properties like maximum, minimum, sum and average values of a column, percentile cutoff ranges, give an initial understanding for the data. It is also helpful to be able to divide the data into different subgroups and get these values for each subgroup.

This article presents an introduction to preliminary data analysis using PostgreSQL and illustrates how to study the aforementioned properties of the data.

## Prerequisites

### Background 

It is assumed you have prior practical exposure to the PostgreSQL database. It is helpful to have a working knowledge of basic data analysis.

### Systems

The commands in this article are tested on PostgreSQL 14.5. SQL queries should work on all recent versions of PostgreSQL.

Note: in the examples below, `$` denotes the operating system prompt as a non-root user. Code snippets without a prompt are SQL statements.

## Example Dataset

PostgreSQL has a number of builtin functions that are useful in analysing data. It is helpful to see these functions in action in the form of SQL queries on an actual dataset. 

This article uses a dataset about cancer statistics. This dataset is based on (U.S) government data and is available publicly (behind a free login-wall) on [data.world](https://data.world/exercises/linear-regression-exercise-1/workspace/file?filename=cancer_reg.csv) (the author has no affiliation with the website). Kaggle also has a number of datasets freely available.

The examples in this article are presented as hypothetical analyses performed on this dataset. It is strongly recommended to try out the examples while reading through the article. 

The next two sections show how to import the dataset (as a CSV file) into PostgreSQL and how to preprocess the data before analysing it. You can skip these steps and start directly with the Data Analysis section by using the preprocessed datafile as an SQL dump file. Alternatively, you can use the Docker Image with the data preloaded.

### Import the Data File into the Database

The dataset used in this article shows the annual average of the number of diagnosed cancer cases and cancer related deaths per county (US). Besides this, it includes basic demographic (age, income, education, employment, unemployment, etc.) data per county. A description of all the columns in the dataset is available on the [data dictionary page](https://data.world/exercises/linear-regression-exercise-1/workspace/data-dictionary). 

Download the CSV file containing the data from the [webpage of the dataset](https://data.world/exercises/linear-regression-exercise-1/workspace/file?filename=cancer_reg.csv) (the page has download buttons). For this particular dataset, the data file is `cancer_reg.csv`. 

It is advisable to create and use a new database to test the examples. As the `postgres` user, open a `psql` prompt on the terminal. You can also use a graphical SQL editor (e.g. Postico, Beekeeper, and many others). The examples below assume the use of a `psql` terminal.

Create a new database:

    CREATE DATABASE cancer_db ;

Connect to the newly created database:

    \c cancer_db

Create a new table `cancer_data`: 

    CREATE TABLE cancer_data (avganncount DECIMAL, avgdeathsperyear INTEGER, target_deathrate DECIMAL,incidencerate DECIMAL, medincome INTEGER, popest2015 INTEGER, povertypercent DECIMAL, studypercap DECIMAL, binnedinc VARCHAR, medianage DECIMAL, medianagemale DECIMAL, medianagefemale DECIMAL, geography VARCHAR, percentmarried DECIMAL, pctnohs18_24 DECIMAL, pcths18_24 DECIMAL, pctsomecol18_24 DECIMAL, pctbachdeg18_24 DECIMAL, pcths25_over DECIMAL, pctbachdeg25_over DECIMAL, pctemployed16_over DECIMAL, pctunemployed16_over DECIMAL, pctprivatecoverage DECIMAL, pctprivatecoveragealone DECIMAL, pctempprivcoverage DECIMAL, pctpubliccoverage DECIMAL, pctpubliccoveragealone DECIMAL, pctwhite DECIMAL, pctblack DECIMAL, pctasian DECIMAL, pctotherrace DECIMAL, pctmarriedhouseholds DECIMAL, birthrate DECIMAL) ;

This table contains columns for each of the fields in the CSV file. The data types of the columns are based on the [data dictionary](https://data.world/exercises/linear-regression-exercise-1/workspace/data-dictionary). It is important to correctly match the column names and data types between the CSV file and the data table. Not doing this leads to errors in importing the data.

Copy the data from the CSV file to the table:

    COPY cancer_data 
    FROM '/path/to/data/file/cancer_reg.csv' DELIMITER ',' CSV HEADER ;
    
In the above command, the file path should be based on the location of the `cancer_reg.csv` file on your system.

Check a couple of rows to get an idea of the data itself:

    SELECT * FROM cancer_data LIMIT 2 ;

### Data Preprocessing

For most analytical exercises, the original data needs to be modified to make it more amenable to analysis. In the `cancer_data` table, the column `geography` contains string values of the form *county, state*. There is no column for individual states. Writing SQL queries based on a portion of a string value is not the right approach. It is easier to have dedicated columns with the county and state values.

Add two new columns, `county` and `state`, to the table `cancer_data`:

    ALTER TABLE cancer_data ADD COLUMN county VARCHAR ;

    ALTER TABLE cancer_data ADD COLUMN state VARCHAR ;

Update these columns based on the value of the county and state in the column `geography`:

    UPDATE cancer_data 
        SET county = SPLIT_PART(geography, ', ', 1), 
        state = SPLIT_PART(geography, ', ', 2) ;

Notice that the separator is a comma followed by a space. Check that the above command did what was expected: 

    SELECT geography, county, state 
    FROM cancer_data LIMIT 5 ;

To enhance readability and usability, create a materialized view based on the original data. In this example, the materialized view is customized as follows: 

1. Only a few selected columns that are used in the analysis
1. Many of the columns are renamed for better readability
1. Additional columns with per capita values of some of the data points, such as the number of `avg_annual_cases` and `avg_annual_deaths`
1. Additional columns with the county and state names (this change was done to the table itself)

The following query creates the new materialized view, `mv_cancer_data`:

    CREATE MATERIALIZED VIEW mv_cancer_data AS 
    SELECT 
    avganncount AS avg_annual_cases, 
    avganncount / popest2015::float AS percapita_annual_cases, 
    avgdeathsperyear AS avg_annual_deaths, 
    avgdeathsperyear / popest2015::float  AS percapita_annual_deaths, 
    medincome AS median_income, 
    popest2015 AS population, 
    povertypercent AS pc_poverty, 
    medianage AS median_age, 
    pctemployed16_over AS pc_employed, 
    pctunemployed16_over AS pc_unemployed, 
    county, 
    state 
    FROM cancer_data ;

This materialized view, `mv_cancer_data` will be used in all further examples. Check the columns and data types in the materialized view:

    \d mv_cancer_data

Below is a description of the columns in `mv_cancer_data`.

* `avg_annual_cases` - Mean number of reported cases of cancer diagnosed annually in the county
* `percapita_annual_cases` - Number of reported cases in the county divided by county population
* `avg_annual_deaths` - Mean number of reported mortalities due to cancer in the county
* `percapita_annual_deaths` - Number of cancer deaths in the county divided by county population
* `median_income` - Median income per county
* `population` - Population of county
* `pc_poverty` - Percent of county populace in poverty
* `median_age` - Median age of county residents 
* `county` - County name
* `state` - State name
* `pc_employed` - Percent of county residents ages 16 and over who are employed
* `pc_unemployed` - Percent of county residents ages 16 and over who are unemployed

Note that percentages of unemployed and employed people do not add up to 100%. The difference is attributed to people who have quit looking for work, are not looking for work, are unable to work, or otherwise are not in the labor force. 

## Data Analysis

This section shows ways to do an exploratory analysis of the data itself - this helps in getting a "feel" of the data before digging deeper. Functions used in this section, such as `sum` or `avg` (average), compute their output based on the value of a number of rows. Such functions are called *aggregate functions*.

Aggregate functions do their computation over the rows that match the conditions in the `WHERE` clause. In the absence of a `WHERE` clause, the function is applied over all the rows.

Note that the queries in this section are based on the materialized view, `mv_cancer_data`, which contains the preprocessed dataset.

### Count the Number of Rows

    SELECT count(*) FROM mv_cancer_data ;

The above query returns the total number of rows in the table `cancer_data`. To get the number of entries for Alabama, count the number rows where the `state` column has the value `Alabama`:

    SELECT count(*) FROM mv_cancer_data 
    WHERE state = 'Alabama' ;
 
### Maximum and Minimum Values of a Column

To get the maximum and minimum values in a column, use the `max` and `min` functions respectively.

    SELECT MIN(avg_annual_deaths) 
    FROM mv_cancer_data ;

The above query returns the minimum number of annual deaths in any county. Get the maximum number of annual (per county) deaths in Alabama counties:

    SELECT max(avg_annual_deaths) 
    FROM mv_cancer_data 
    WHERE state = 'Alabama' ;

### Using Aggregate Functions in the `WHERE` clause

In the previous queries, suppose you also want to get all the other details (columns) of the county which had the maximum number of annual deaths. 

    SELECT * FROM mv_cancer_data 
    WHERE avg_annual_deaths = MAX(avg_annual_deaths) ;

The above query is wrong. It is not allowed to have aggregate functions as part of the `WHERE` clause of a SQL query. The aggregate function, `max`, needs to be used within a sub-query whose output can be then used as part of the main query. To get all the columns of the row with the maximum number of annual deaths:

    SELECT * FROM mv_cancer_data 
    WHERE avg_annual_deaths = 
        (SELECT MAX(avg_annual_deaths) 
        FROM mv_cancer_data) ;

### Sum of the Values of a Column

`sum` adds up the values of a column. To get the total number of annual cancer deaths throughout the country, apply the `sum` function over all the values in the column `avg_annual_deaths`. To get the total number of annual deaths throughout the country (across all counties):

    SELECT sum(avg_annual_deaths) 
    FROM mv_cancer_data ;

To get the total number of annual deaths for a specific state, California:

    SELECT sum(avg_annual_deaths) 
    FROM mv_cancer_data 
    WHERE state = 'California' ;

### Average Value of a Column

The `avg` function averages the (non-null) values of a column. The following query computes the average number of annual cases (per capita) of cancer across all US counties:

    SELECT avg(percapita_annual_cases) 
    FROM mv_cancer_data ;

To compute the average number of annual cases (per capita) of cancer in counties with an average median income higher than $50,000:

    SELECT avg(percapita_annual_cases) 
    FROM mv_cancer_data 
    WHERE median_income > 50000 ;

To compute the average number of annual cases (per capita) of cancer in counties with an average median income lower than $50,000:

    SELECT avg(percapita_annual_cases) 
    FROM mv_cancer_data 
    WHERE median_income < 50000 ;

Observe, based on the outputs of the above two queries, that the average number of diagnosed cases per capita is higher in higher income counties. 

#### Exercise

Modify the above two queries to compute the average number of deaths per capita and observe that there are fewer deaths in counties with higher income. The higher number of diagnoses combined with lower number of deaths can, at least superficially, be attributed to improved testing as well as treatment facilities in higher income regions.

### Percentiles

The `percentile_cont(N)` function computes the value of the *Nth* percentile. The following query gives the 95th percentile of the per capita number of annual deaths across all counties:

    SELECT PERCENTILE_CONT(0.95) 
        WITHIN GROUP 
        (ORDER BY percapita_annual_deaths) 
    FROM mv_cancer_data ;

To get the top 1% of values of an ordered set, you need to find values that are above the 99th percentile. The next query shows the top 1% of counties based on per capita annual deaths:

    SELECT county, state, avg_annual_deaths, percapita_annual_deaths 
    FROM mv_cancer_data 
    WHERE percapita_annual_deaths > 
        (SELECT PERCENTILE_CONT(0.99) 
            WITHIN GROUP 
            (ORDER BY percapita_annual_deaths) 
        FROM mv_cancer_data) ;

The bottom 1% consists of values that are below the 1st percentile:

    SELECT county, state, avg_annual_deaths, percapita_annual_deaths 
    FROM mv_cancer_data 
    WHERE percapita_annual_deaths < 
        (SELECT PERCENTILE_CONT(0.01) 
            WITHIN GROUP 
            (ORDER BY percapita_annual_deaths) 
        FROM mv_cancer_data) ;

## Grouping and Partitioning

### Grouping

One of the goals of analytical exercises is to group the data into different classes to study and compare specific properties of each class. The *grouping operations* in PostgreSQL are often used to achieve this.

To get the number of rows for each state:

    SELECT state, count(*) 
    FROM mv_cancer_data 
    GROUP BY state ;

When a query uses a grouping function together with an aggregation function, the aggregation function is applied separately for each group. 

To find the total number of deaths for each state:

    SELECT state, sum(avg_annual_deaths) AS total_annual_deaths 
    FROM mv_cancer_data 
    GROUP BY state ; 

Add a sorting (ordering) clause in the above query:

    SELECT state, sum(avg_annual_deaths) AS total_annual_deaths 
    FROM mv_cancer_data 
    GROUP BY state 
    ORDER BY total_annual_deaths ;

### Partitioning and Window Functions

Now consider that for each county, you need to show the total number of cancer cases for the state, and the percentage of state-wide cancer cases accounted for by that county. This is not achievable with the grouping functions shown earlier. You need to use *partitioning* to do this:

    SELECT county, 
        state, 
        (avg_annual_cases), 
        sum(avg_annual_cases) OVER (PARTITION BY state) AS total_state_annual_cases, 
        round(
            (avg_annual_cases *100 / 
            sum(avg_annual_cases) OVER (PARTITION BY state))
        ,1) as pc_county_state 
    FROM mv_cancer_data 
    WHERE state = 'Alabama' 
    ORDER BY pc_county_state DESC ;

Grouping functions reduce the number of rows in the input data by applying an aggregation function across a few different rows into a single row. In contrast, window functions do the aggregate computation but without collapsing the number of rows. Applying a `PARTITION BY` clause along with a window function modifies the set of rows over which the aggregation function is applied.

## Conclusion

This article showed how to do preliminary data analysis using PostgreSQL. Many of the builtin functions used for data analysis are listed under **Aggregate Functions** in the [official PostgreSQL documentation](https://www.postgresql.org/docs/current/functions-aggregate.html). The documentation also lists a few more functions than discussed here. The official tutorials on [aggregate functions](https://www.postgresql.org/docs/current/tutorial-agg.html) and [window functions](https://www.postgresql.org/docs/current/tutorial-window.html) are also helpful for a better understanding of these functions.

### Learn More

The next step after preliminary analysis is understanding the statistical properties - mean, standard deviation, correlations, etc. of the data. It is also helpful to use a regression equation to be able to predict the values of one variable based on changes in the other. The second part of this article covers these topics along with relevant examples based on the same dataset used in this article.

