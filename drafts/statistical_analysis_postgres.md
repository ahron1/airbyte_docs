Data Analysis in PostgreSQL

## Introduction

Data analysis is of increasing importance in modern applications. Sophisticated analytics requires the use of specialized tools. However, many commonly used, simpler analyses, such as averages and variances, as well as linear regressions and group-wise analyses, can be run within PostgreSQL using builtin functions. Doing the analyses within the database has a few advantages:

1. fewer IT systems to manage and maintain
1. avoiding passing data back and forth between different systems
1. leveraging a mature system for enforcing data integrity

## Prerequisites

### Background 

It is assumed you have prior practical exposure to the PostgreSQL database. It is helpful to have a working knowledge of basic statistical analysis.

### Systems

The commands in this article are tested on PostgreSQL 14.5 running on FreeBSD 13.1. SQL queries should work on all recent versions of PostgreSQL. File system based commands can be OS-specific. 

Note: in the examples below, `$` denotes the operating system prompt as a non-root user. Code snippets with no prompt are SQL statements.

## Example Dataset

PostgreSQL has a number of builtin functions that are useful in analysing data. It is helpful to see the functions in action in the form of SQL queries on an actual dataset. This article uses a dataset about cancer statistics. This dataset is available publicly (behind a free login-wall) on [data.world](https://data.world/exercises/linear-regression-exercise-1/workspace/file?filename=cancer_reg.csv) (the author has no affiliation with the website). Kaggle also has a number of datasets freely available.

Much of this article is organized as hypothetical analyses performed on this dataset. It is strongly recommended to try out the examples while reading through the article. 

### Import the Data File into the Database

This dataset shows the annual average of the number of diagnosed cancer cases and cancer related deaths per county (US). Besides this, it includes basic demographic (age, income, education, employment, unemployment, etc.) data per county. A description of all the columns in the dataset is available on the [data dictionary page](https://data.world/exercises/linear-regression-exercise-1/workspace/data-dictionary). 

Download the CSV file containing the data from the [webpage of the dataset](https://data.world/exercises/linear-regression-exercise-1/workspace/file?filename=cancer_reg.csv) (the page has download buttons). For this particular dataset, the data file is `cancer_reg.csv`. 

It is advisable to create and use a new database to test the examples. As the `postgres` user, open a `psql` prompt on the terminal. You can also use a graphical SQL editor (e.g. Postico, Beekeeper, and many others). The examples below assume the use of a `psql` terminal.

Create a new database:

    create database cancer_db;

Connect to the newly created database:

    \c cancer_db;

Create a new table `cancer_data`: 

    CREATE TABLE cancer_data (avganncount DECIMAL, avgdeathsperyear INTEGER, target_deathrate DECIMAL,incidencerate DECIMAL, medincome INTEGER, popest2015 INTEGER, povertypercent DECIMAL, studypercap DECIMAL, binnedinc VARCHAR, medianage DECIMAL, medianagemale decimal, medianagefemale decimal, geography varchar, percentmarried decimal, pctnohs18_24 decimal, pcths18_24 decimal, pctsomecol18_24 decimal, pctbachdeg18_24 decimal, pcths25_over decimal, pctbachdeg25_over decimal, pctemployed16_over decimal, pctunemployed16_over decimal, pctprivatecoverage decimal, pctprivatecoveragealone decimal, pctempprivcoverage decimal, pctpubliccoverage decimal, pctpubliccoveragealone decimal, pctwhite decimal, pctblack decimal, pctasian decimal, pctotherrace decimal, pctmarriedhouseholds decimal, birthrate decimal);

The new table contains columns for each of the fields in the CSV file. The data types for the columns are based on the [data dictionary](https://data.world/exercises/linear-regression-exercise-1/workspace/data-dictionary). It is important to correctly match the column names and data types between the CSV file and the data table. Not doing this leads to errors in importing the data.

Copy the data from the CSV file to the table:

    COPY cancer_data FROM '/path/to/data/file/cancer_reg.csv' DELIMITER ',' CSV HEADER;
    
In the above command, the file path should be based on the location of the `cancer_reg.csv` file on your system.

Check a couple of rows to get an idea of the data itself:

    SELECT * FROM cancer_data LIMIT 2;

## Data Preprocessing

For most analytical exercises, the original data needs to be modified to make it more amenable for analysis. In this example, the column `geography` contains string values of the form *county, state*. There is no column for individual states. Writing SQL queries based on a portion of a string value is not the right approach. It is easier to have dedicated columns with the county and state values.

Add two new columns, `county` and `state` to the table `cancer_data`:

    ALTER TABLE cancer_data ADD COLUMN county VARCHAR;

    ALTER TABLE cancer_data ADD COLUMN state VARCHAR;

Update these columns based on the value of the county and state in the column `geography`:

    UPDATE cancer_data SET county = SPLIT_PART(geography, ', ', 1), state = SPLIT_PART(geography, ', ', 2);

Notice that the separator is a comma followed by a space. Check that the above command did what was expected: 

    SELECT geography, county, state FROM cancer_data LIMIT 5;

Take a look at the table's columns and data types:

    \d cancer_data

As mentioned earlier, the examples in this article will focus on only a few columns. To make it easier to read through the results of the queries, create a materialized view containing only the selected columns. For many analyses, the per capita values of certain data points, such as the number of `avg_annual_cases` and `avg_annual_deaths` need to be computed. Dividing the raw data point by the population gives the value *per capita*.

    CREATE MATERIALIZED VIEW mv_cancer_data as SELECT avganncount AS avg_annual_cases, round(avganncount / popest2015, 1) as percapita_annual_cases, avgdeathsperyear AS avg_annual_deaths, round(avgdeathsperyear / popest2015, 1) as percapita_annual_deaths, medincome AS median_income, popest2015 AS population, povertypercent AS pc_poverty, medianage AS median_age, pctemployed16_over AS pc_employed, pctunemployed16_over AS pc_unemployed, county, state from test;

This materialized view, `mv_cancer_data` will be used in all further examples. Check the columns and data types in the materialized view:

    \d mv_cancer_data

Below is a description of a few selected columns, which will be used in the examples in this article.

* avg_annual_cases - Mean number of reported cases of cancer diagnosed annually in the county
* avg_annual_deaths - Mean number of reported mortalities due to cancer in the county
* median_income - Median income per county
* population - Population of county
* pc_poverty - Percent of county populace in poverty
* median_age - Median age of county residents 
* county - County name
* state - State name
* pc_employed - Percent of county residents ages 16 and over who are employed
* pc_unemployed - Percent of county residents ages 16 and over who are unemployed

Note that percentages of unemployed and employed people does not add up to 100%. The difference is attributed to people who have quit looking for work, are not looking for work, unable to work, or otherwise are not in the labor force, etc. 

The main differences between the materialized view and the original data table are:

1. Only a few selected columns
1. Many of the columns are renamed for better readability
1. Additional columns with per capita values of some of the data points
1. Additional columns with the county and state names (this change was done to the table itself)

## Preliminary Data Analysis

This section shows ways to do an exploratory analysis of the data itself - this helps in getting a "feel" of the data before digging deeper. Functions used in this section, such as `sum` or `avg` (average), compute their output based on the value of a number of rows. Such functions are called *aggregate functions*.

Aggregate functions do their computation over the rows that match the conditions in the `WHERE` clause. In the absence of a `WHERE` clause, the function is applied over all the rows.

### Count the Number of Rows

    select count(*) from mv_cancer_data;

The above query returns the total number of rows in the table `cancer_data`. To get the number of entries for Alabama, count the number rows where the `state` column has the value `Alabama`:

    select count(*) from mv_cancer_data where state = 'Alabama';

### Maximum and Minimum Values of a Column

To get the maximum and minimum values in a column, use the `max` and `min` functions respectively.

    select min(avg_annual_deaths) from mv_cancer_data ;

The above query returns the minimum number of annual deaths in any county. Similarly, to get the maximum number of annual deaths in any county in Alabama, use the query below:

    select max(avg_annual_deaths) from mv_cancer_data where state = 'Alabama';

### Using Aggregate Functions in the `WHERE` clause

In the previous query, suppose you also want to get all the other details (columns) of the county which had the maximum number of annual deaths. 

    select * from mv_cancer_data where avg_annual_deaths = max (avg_annual_deaths);

The above query is wrong because it is not allowed to have aggregate functions as part of the `WHERE` clause of a SQL query. The aggregate function, `MAX`, needs to be used within a sub-query whose output can be then used as part of the main query. To get all the columns of the row with the maximum number of annual deaths:

    select * from mv_cancer_data where avg_annual_deaths = (select max(avg_annual_deaths) from mv_cancer_data);

### Sum the Values of a Column

To get, for instance, the total number of annual deaths throughout the country, apply the `sum` function over all the values in the column `avg_annual_deaths`. To get the total number of annual deaths throughout the country (across all counties):

    select sum(avg_annual_deaths) from mv_cancer_data;

To get the total number of annual deaths for a specific state, California:

    select sum(avg_annual_deaths) from mv_cancer_data where state = 'California';

### Average Value of a Column

The `AVG` function averages the (non-null) values of a column. The following query computes the number of annual cases (per capita) of cancer across all US counties:

    select avg(percapita_annual_cases) from mv_cancer_data ;

To compute the number of annual cases (per capita) of cancer in counties with an average median income higher than $50,000:

    select avg(percapita_annual_cases) from mv_cancer_data where median_income > 50000;

To compute the number of annual cases (per capita) of cancer in counties with an average median income lower than $50,000:

    select avg(percapita_annual_cases) from mv_cancer_data where median_income < 50000;

Observe, based on the outputs of the above two queries, that the number of diagnosed cases per capita is higher in higher income counties. 

#### Exercise

Modify the above two queries to compute the number of deaths per capita and observe that there are fewer deaths in counties with higher income. The higher number of diagnoses combined with lower number of deaths can, at least superficially, be attributed to improved testing and treatment facilities in higher income regions.

### Percentiles

The `percentile_cont(N)` function computes the value of the *Nth* percentile. The following query gives the 95th percentile of the per capita number of annual deaths across all counties:

    select percentile_cont(0.95) within group (order by percapita_annual_deaths) from mv_cancer_data ;

To get the top 1% of values of an ordered set, you need to find values that are above the 99th percentile. The next query shows the top 1% of counties based on per capita annual deaths:

    select county, state, avg_annual_deaths, percapita_annual_deaths from mv_cancer_data where percapita_annual_deaths > (select percentile_cont(0.99) within group (order by percapita_annual_deaths) from mv_cancer_data) ;

The bottom 1% consists of values that are below the 1st percentile:

    select county, state, avg_annual_deaths, percapita_annual_deaths from mv_cancer_data where percapita_annual_deaths < (select percentile_cont(0.01) within group (order by percapita_annual_deaths) from mv_cancer_data) ;

## Grouping and Partitioning

### Grouping

One of the end goals of analytical exercises is to group the data into different classes to study and compare specific properties of each class. The *grouping operations* in PostgreSQL are often used to achieve this.

To get the number of rows for each state:

    select state, count(*) from mv_cancer_data group by state;

When a query uses a grouping function together with an aggregation function, the aggregation function is applied separately for each group. 

For instance, suppose you want to find the total number of deaths for each state:

    select state, sum(avg_annual_deaths) as total_annual_deaths from mv_cancer_data group by state; 

Add a sorting (ordering) clause in the above query:

    select state, sum(avg_annual_deaths) as total_annual_deaths from mv_cancer_data group by state order by total_annual_deaths;

### Partitioning and Window Functions

Now consider that for each county, you need to show the total number of cancer cases for the state, and the percentage of state-wide cancer cases accounted for by that county. This is not achievable with the grouping functions shown earlier. You need to use partitioning to do this:

    select county, state, (avg_annual_cases), sum(avg_annual_cases) over (partition by state) as total_state_annual_cases, round(avg_annual_cases *100 / sum(avg_annual_cases) over (partition by state),1) as pc_county_state from mv_cancer_data ; where state = 'Alabama' order by pc_county desc limit 2;

Grouping functions reduce the number of rows in the input data by applying an aggregation function across a few different rows into a single row. For example, when a list of counties are grouped by state with respect to the `sum` function on a column X, the output is a list of states and the sum of all the values of column X for all the counties in each state.

In contrast, window functions do the aggregate computation but without collapsing the number of rows. Applying a `PARTITION BY` clause along with a window function modifies the set of rows over which the aggregation function is applied.

## Statistical Properties of a Single Variable

### Mean

The mean of a sample is the average of all the data points in the sample. The `AVG` function discussed earlier calculates the population mean. 

The average of per capita annual deaths across all counties is given by:

    select round(avg(percapita_annual_deaths),2) from mv_cancer_data ;

### Variance and Standard Deviation

The variance of a sample is the average (over the sample size) of the square of the difference between each data point and the mean. Based on this definition, it follows that if the values are closer to the mean, the dataset has a low variance, and vice versa. Variance is said to measure the *dispersion* of the data. Variance is measured in squared units, which are not very intuitive. 

A more commonly used metric of dispersion is standard deviation, which is computed as the square root of the variance. Because it is the square root of variance, the unit of standard deviation is the same as the unit of the data itself.

The function `var_samp` returns the sample variance. The variance of per capita deaths across different counties is given by:

    select round(var_samp(percapita_annual_deaths),2) from mv_cancer_data ;
    
Sample standard deviation is computed using `stddev_samp`. The standard deviation of per capita annual deaths across counties is given by:

    select round(stddev_samp(percapita_annual_deaths),2) from mv_cancer_data ;

Note: the above queries use the sample variance and the sample standard deviation functions. The functions for population variance, `var_pop` and population standard deviation, `stddev_pop`, work similarly. The distinction between sample properties and population properties is beyond the scope of this article.

### Coefficient of Variation

The coefficient of variation (CV) is a commonly used metric - it denotes the dispersion of the data relative to the mean. It is measured as the ratio of the standard deviation to the mean. The value of CV is computed explicitly (there is no builtin function for it): 

    select round(stddev_samp(percapita_annual_deaths)/avg(percapita_annual_deaths),2) from mv_cancer_data ;

### Outliers

Outliers are data points whose values lie far from the rest of the distribution. The presence of outliers in the data can skew aggregate properties like the mean and variance. Excluding the outliers makes the data better behaved. A common method to identify outliers is based on the 3-sigma rule, also known as the [68-95-99.7 Empirical rule](https://www.investopedia.com/terms/e/empirical-rule.asp). Values outside 3 standard deviations of the mean can be considered outliers. 

For instance, to get the number of counties whose unemployment percentage is within 3 standard deviations (3-sigma) of the mean:

    select count(*) from mv_cancer_data where (pc_unemployed > (select avg(pc_unemployed) - 3*stddev_samp(pc_unemployed) from mv_cancer_data) and pc_unemployed < (select avg(pc_unemployed) + 3*stddev_samp(pc_unemployed) from mv_cancer_data));

Note that the distance of 3 sigma is to be applied on both sides of the mean. Similarly, to get the number of counties whose employment percentage is within 3 standard deviations (3-sigma) of the mean:

    select count(*) from mv_cancer_data where (pc_employed > (select avg(pc_employed) - 3*stddev_samp(pc_employed) from mv_cancer_data) and pc_employed < (select avg(pc_employed) + 3*stddev_samp(pc_employed) from mv_cancer_data));

The total number of data points (counties) is:

    select count(*) from mv_cancer_data;

Notice, based on the output of the above three queries, that the percentage of employed people in different counties (`pc_employed`) has more outliers than the percentage of unemployed people (`pc_unemployed`). Recall, as mentioned previously in the section Data Description, that unemployment and employment rates do not add up to 1. 

## Statistical Relations of Two Variables

### Covariance and Correlation 

The covariance and correlation both measure the degree to which two variables jointly deviate from their respective means. For two groups of variables X and Y with n values each, the covariance is measured as:

(1/n)sum((X_i - avg(X))*(Y_i - avg(Y)))

The correlation (coefficient) is the covariance adjusted by the standard deviation. 

Correlation = Covariance / (stddev(X)*stddev(Y))

Note that the correlation function is symmetrical, Correlation(X,Y) = Correlation(Y,X).

A high correlation between two variables indicates a strong statistical (not causal) relationship between the two variables. The direction and magnitude of the change in one variable is strongly affected by the direction and magnitude of the change of the other variable. Thus, one variable (the independent variable) can be used to predict unknown values of the other variable (the dependent variable) - this is shown in the next section (Regression).

To get the correlation coefficient between two variables, use the `corr` function. The following query computes the correlation between the number of annual cases (diagnoses) and number of annual deaths per county:

    select corr(avg_annual_cases, avg_annual_deaths) from mv_cancer_data ;

Observe that this correlation is a high value. Heuristically, correlation values between 0.3 and 0.5 are considered low, values between 0.5 and 0.7 for the correlation coefficient indicate a moderate degree of correlation and moderate predictive power. Values between 0.7 and 0.9 are considered strongly correlated and values above 0.9 are considered very strongly correlated. Negative values of the correlation coefficient indicate that when one variable increases, the other decreases.

The covariance is computed using the `covar_pop` and `covar_samp` functions which have a similar syntax as the `corr` function. Verify that the correlation is indeed computed as the covariance adjusted by the standard deviation:

    select covar_samp(avg_annual_cases, avg_annual_deaths) / (stddev_samp(avg_annual_cases) * stddev_samp(avg_annual_deaths)) from mv_cancer_data ;

#### Exercise 

Compute the correlation between median income and per capita annual deaths. Notice that median income is negatively correlated with per capita annual deaths. This indicates fewer people die of cancer in wealthier counties. This conclusion was also reached from studying the average values in an earlier section.

### Regression

To be able to predict the values of a dependent variable (Y) based on the value of the independent variable (X), you need an expression that encodes the relation between them. This relation is called the regression equation and it is of the form:

Y_predicted = slope * X + intercept 

In simple regression (Ordinary Least Squares - OLS), the slope is the correlation coefficient (of Y and X) scaled by the ratio of the standard deviations of Y and X:

Slope = Correlation (Y, X) * stddev(Y) / stddev(X)

The intercept the is the expected value of Y were X to be 0.

Assume, for sake of illustration, that the dependent variable is number of annual cases of cancer, `avg_annual_deaths`, and the independent variable is the number of cases, `avg_annual_cases`. The goal is to be able to express the number of deaths as a function of the number of cases:

Number_Deaths = Slope * Number_Cases + Intercept + Error

The following query returns the value of the slope:

    select regr_slope(avg_annual_deaths, avg_annual_cases) from mv_cancer_data ;

The Intercept is given by: 

    select regr_intercept(avg_annual_deaths, avg_annual_cases) from mv_cancer_data ;

Thus, the relationship is:

Number_Deaths_Predicted = 0.33 * Number_Cases - 16.7 

### Errors

The regression equation predicts the value of Y for a given value of X. The difference between the predicted value of Y and the actual value of Y is the error. This error can be positive or negative. To find the error across all values of Y, take the average of the sum of the squares of the individual errors. This is called the Root Mean Square (RMS) error. For the earlier values of slope (0.33) and intercept (-16.7), it can be computed with the following query:

    
    -- pseudo code -- select sqrt(avg(((avg_annual_deaths) - (PREDICTED_VALUE))^2)) from mv_cancer_data

    select sqrt(avg(((avg_annual_deaths) - (0.33*avg_annual_cases-16.7))^2)) from mv_cancer_data

One way of reducing the error is to consider a modified dataset that excludes the outliers.

## Conclusion

This article showed how to do preliminary statistical analysis using PostgreSQL. Many of the builtin PostgreSQL functions used for data analysis are listed under **Aggregate Functions** in the [official PostgreSQL documentation](https://www.postgresql.org/docs/current/functions-aggregate.html). The documentation also lists a few more functions that discussed here. The official tutorials on [aggregate functions](https://www.postgresql.org/docs/current/tutorial-agg.html) and [window functions](https://www.postgresql.org/docs/current/tutorial-window.html) are also helpful for a better understanding of these functions.

### Learn More

More advanced metrics such as *p-values* (used for confidence intervals), cannot be directly computed using built in functions. It is possible to write SQL queries to do that. However, such queries tend to be complex, error-prone, and hard to debug. PostgreSQL also has no builtin functions for using the Logit and Probit models of regression. In cases where such functionality is needed, it is advisable to use `PL/Python` - the Python Procedural Language. This allows to run Python commands (and packages) inside a PostgreSQL database. `PL/R` - the R Procedural Language also has many primitives for basic and advanced statistical computations.

