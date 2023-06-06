# Data Analysis in PostgreSQL - II

## Introduction

This is the second part of the article on data analysis using PostgreSQL. The first part of this article discusses how to study the preliminary properties of the data. It demonstrated SQL queries to get basic features like maximum and minimum values, sums and averages, percentile values, etc. for entire columns as well as for subsets (subgroups) of columns.

The next logical step in analyzing the data is studying its statistical properties, such as variances, standard deviations, and correlations, as well as linear regressions. Basic statistical analysis does not require the use of specialized tools. It can be run directly within PostgreSQL using built-in functions. Doing the analysis within the database has a few advantages:

1. fewer IT systems to manage and maintain
1. avoiding passing data back and forth between different systems
1. leveraging a mature RDBMS for enforcing data integrity and consistency

## Prerequisites

### Background 

In addition to hands-on experience with the PostgreSQL database, it is helpful to have a working knowledge of basic statistical analysis. It is also assumed you are comfortable with the material discussed in the first part of this article.

### Systems

The commands in this article are tested on PostgreSQL 14.5 and should work on all recent versions of PostgreSQL.

### Example Dataset

To demonstrate basic statistical analysis, this article uses examples based on a dataset on cancer statistics. It is strongly recommended to try out the examples while reading through the article. 

The first part of the article discusses in detail how to get the data into PostgreSQL and preprocess it. It starts from a CSV file and imports it into a table in a new database. Preprocessing is done on this table and the dataset for analysis is presented as a materialized view. This materialized view, `mv_cancer_data`, is used in all the examples in this article.  It is necessary to follow the steps in the Example Dataset section of the previous article and prepare `mv_cancer_data`. 

Alternatively, you can download the SQL dump file containing all the data. You can also use the Docker image and get started directly.

## Statistical Properties of a Single Variable

### Mean

The mean of a sample is the average of all the data points in the sample. The `avg` function discussed earlier calculates the mean value of the population. 

The average of per capita annual deaths across all counties is given by:

    SELECT avg(percapita_annual_deaths)
    FROM mv_cancer_data ;

### Variance and Standard Deviation

The variance of a sample is the average (over the sample size) of the square of the difference between each data point and the mean. Based on this definition, it follows that if the values are closer to the mean, the dataset has a low variance, and vice versa. Variance is said to measure the *dispersion* of the data. Variance is measured in squared units, which are not very intuitive. 

A more commonly used metric of dispersion is standard deviation, which is computed as the square root of the variance. Because it is the square root of the variance, the unit of standard deviation is the same as the unit of the data itself.

The function `var_samp` returns the sample variance. The variance of per capita deaths across different counties is given by:

    SELECT var_samp(percapita_annual_deaths)
    FROM mv_cancer_data ;
    
Sample standard deviation is computed using `stddev_samp`. The standard deviation of per capita annual deaths across counties is given by:

    SELECT stddev_samp(percapita_annual_deaths)
    FROM mv_cancer_data ;

Note: the above queries use the sample variance and the sample standard deviation functions. The functions for population variance (`var_pop`) and population standard deviation (`stddev_pop`), work similarly. The distinction and relation between sample properties and population properties is beyond the scope of this article.

### Coefficient of Variation

The coefficient of variation (CV) is a commonly used metric - it denotes the dispersion of the data relative to the mean. It is measured as the ratio of the standard deviation to the mean. The value of CV is computed explicitly (there is no builtin function for it): 

    SELECT 
        stddev_samp(percapita_annual_deaths) /
        avg(percapita_annual_deaths)
        AS coefficient_of_variation
    FROM mv_cancer_data ;

### Outliers

Outliers are data points whose values lie far from the rest of the distribution. The presence of outliers in the data can skew aggregate properties like the mean and variance. Excluding the outliers makes the data better behaved. A common method to identify outliers is based on the 3-sigma rule, also known as the [68-95-99.7 Empirical rule](https://www.investopedia.com/terms/e/empirical-rule.asp). Values outside 3 standard deviations of the mean can be considered outliers. 

To get the number of counties whose **unemployment percentage** is within 3 standard deviations (3-sigma) of the mean:

    SELECT count(*) 
    FROM mv_cancer_data 
    WHERE (
        pc_unemployed > (
            SELECT avg(pc_unemployed) - 
                3*stddev_samp(pc_unemployed) 
            FROM mv_cancer_data) 
        AND 
        pc_unemployed < (
            SELECT avg(pc_unemployed) + 
                3*stddev_samp(pc_unemployed) 
            FROM mv_cancer_data) 
    ) ;

Note that the distance of 3 sigmas is to be applied on both sides of the mean. Similarly, to get the number of counties whose **employment** percentage is within 3 standard deviations (3-sigma) of the mean:

    SELECT count(*) 
    FROM mv_cancer_data 
    WHERE (
        pc_employed > (
            SELECT avg(pc_employed) - 
                3*stddev_samp(pc_employed) 
            FROM mv_cancer_data) 
        and 
        pc_employed < (
            SELECT avg(pc_employed) + 
                3*stddev_samp(pc_employed) 
            FROM mv_cancer_data) 
    );

For comparison, the total number of data points (counties) is:

    SELECT count(*) FROM mv_cancer_data ;

Notice, based on the output of the above three queries, that the percentage of employed people in different counties (`pc_employed`) has more outliers than the percentage of unemployed people (`pc_unemployed`). 

The presence of outliers skews statistical properties (like averages and correlations) in their direction. Whether to eliminate outliers from a dataset is a subjective decision. It is advisable to study the outlier values before deciding whether to remove them.

## Statistical Relations of Two Variables

### Covariance and Correlation 

Covariance and correlation both measure the degree to which two variables jointly deviate from their respective means. For two groups of variables X and Y with n values each, the covariance is measured as:

$\frac{1}{n}\sum((X_i - avg(X))(Y_i - avg(Y)))$

The correlation (coefficient) is the covariance adjusted by the standard deviations. 

$Correlation = \frac{Covariance}{stddev(X)*stddev(Y)}$

Note that the correlation function is symmetrical, Correlation(X, Y) = Correlation(Y, X).

A high correlation between two variables indicates a strong statistical (not causal) relationship between the two variables. The direction and magnitude of the change in one variable strongly affect the direction and magnitude of the change in the other variable. Thus, one variable (the independent variable) can be used to predict unknown values of the other variable (the dependent variable) - this is shown in the next section (Regression).

To get the correlation coefficient between two variables, use the `corr` function. The following query computes the correlation between the number of annual cases (diagnoses) and the number of annual deaths per county:

    SELECT CORR(avg_annual_cases, avg_annual_deaths) FROM mv_cancer_data ;

Observe that this correlation is a high value. Heuristically, correlation values between 0.3 and 0.5 are considered low, values between 0.5 and 0.7 for the correlation coefficient indicate a moderate degree of correlation and moderate predictive power. Values between 0.7 and 0.9 are considered strongly correlated and values above 0.9 are considered very strongly correlated. Negative values of the correlation coefficient indicate that when one variable increases, the other decreases.

Note that a low (between 0.3 and 0.5) value of the correlation does not mean one variable *cannot* be used to predict the values of the other variable. It only means the prediction errors will be high. Indeed, in many social sciences disciplines, it is common to work with correlations in this range.

The covariance is computed using the `covar_pop` and `covar_samp` functions which have similar syntax as the `corr` function. Verify that the correlation (as obtained in the previous query) is indeed computed as the covariance adjusted by the standard deviation:

    SELECT 
        covar_samp(avg_annual_cases, avg_annual_deaths) / 
        (stddev_samp(avg_annual_cases) * stddev_samp(avg_annual_deaths)) 
    FROM mv_cancer_data ;

#### Exercise 

Compute the correlation between median income and per capita annual deaths. Notice that median income is negatively correlated with per capita annual deaths. This indicates fewer people die of cancer in wealthier counties. This conclusion was also reached by studying the average values in an earlier section. 

Try also to study the correlations between other pairs of variables you think might be related.

### Regression

To be able to predict the values of a dependent variable (Y) based on the value of the independent variable (X), you need an expression that encodes the relation between them. This relation is called the regression equation and it is of the form:

$Y_{predicted} = slope * X + intercept$

$Y_{actual} = slope * X + intercept + error$

In simple regression (Ordinary Least Squares - OLS), the slope is the correlation coefficient (of Y and X) scaled by the ratio of the standard deviations of Y and X:

$Slope = Correlation(Y,X)*\frac{stddev(Y)}{stddev(X)}$

The intercept is the expected value of Y if X is 0.

Assume, for the sake of illustration, that the dependent variable is the number of annual cases of cancer, `avg_annual_deaths`, and the independent variable is the number of cases, `avg_annual_cases`. The goal is to be able to express the number of deaths as a function of the number of cases:

$Number_{Deaths}^{actual} = Slope * Number_{Cases} + Intercept + Error$

The following query returns the value of the slope:

    SELECT regr_slope(avg_annual_deaths, avg_annual_cases) 
    FROM mv_cancer_data ;

The Intercept is given by: 

    SELECT regr_intercept(avg_annual_deaths, avg_annual_cases) 
    FROM mv_cancer_data ;

Thus, based on the output of the above two queries, the relationship is:

$Number_{Deaths}^{predicted} = 0.33 * Number_{Cases} - 16.8 $

Note that any pair of variables with a significant correlation coefficient can be used to construct a regression equation. Deciding which variable is the dependent variable and which is the independent variable is subjective and dependent on the use case.

#### Exercise

Obtain the regression equation for two other pairs of variables which have a correlation coefficient > 0.3.

### Errors

The regression equation predicts the value of Y for a given value of X. The difference between the predicted value of Y and the actual value of Y is the error. This error can be positive or negative. To find the error across all values of Y, take the average of the sum of the squares of the individual errors. This is called the Root Mean Square (RMS) error. For the earlier values of slope (0.33) and intercept (-16.7), it can be computed with the following query:

    
    -- pseudo code -- SELECT sqrt(avg(((avg_annual_deaths) - (PREDICTED_VALUE))^2)) FROM mv_cancer_data

    SELECT sqrt(
        avg(
            (avg_annual_deaths - (0.33*avg_annual_cases-16.8))^2
        )
    ) 
    FROM mv_cancer_data ;

One way of reducing the error is to consider a modified dataset that excludes the outliers. Typically, outliers are eliminated from the columns that go into the regression equation.

Note that the higher the correlation coefficient between X and Y, the lower the error when predicting Y from X (or X from Y).

#### Exercises

1. Compute the error based on the regression of two other variables. Verify that the regression equation based on highly correlated variables leads to lower errors than the regression of variables with low-moderate correlation.

1. Construct a new dataset without outliers (make a new materialized view based on `mv_cancer_data`). Try to estimate the correlation, regression equations, and errors based on this dataset. Notice that the dataset without outliers has lower errors.

## Conclusion

Knowledge of SQL and basic statistics is a valuable tool for the data analyst. It eliminates the need to rely on dedicated software packages to perform preliminary statistical analysis using PostgreSQL.

The built-in functions used for data analysis are listed under **Aggregate Functions** in the [official PostgreSQL documentation](https://www.postgresql.org/docs/current/functions-aggregate.html). 

### Learn More

More advanced metrics such as *p-values* (used for confidence intervals), cannot be directly computed using built-in functions. It is possible to write SQL queries to do that. However, such queries tend to be complex, error-prone, and hard to debug. PostgreSQL also has no built-in functions for using the Logit and Probit models of regression. In cases where such functionality is needed, it is advisable to use `PL/Python` - the Python Procedural Language. This allows to run Python commands (and packages) inside a PostgreSQL database. `PL/R` - the R Procedural Language in PostgreSQL also has many primitives for basic and advanced statistical computations.

