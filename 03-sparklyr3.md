### Using SQL
It’s also possible to execute SQL queries directly against tables within a Spark cluster. The ``spark_connection`` object implements a DBI interface for Spark, so you can use ``dbGetQuery`` to execute SQL and return the result as an R data frame:
<pre><code>
library(DBI)
iris_preview <- dbGetQuery(sc, "SELECT * FROM iris LIMIT 10")
iris_preview
##    Sepal_Length Sepal_Width Petal_Length Petal_Width Species
## 1           5.1         3.5          1.4         0.2  setosa
## 2           4.9         3.0          1.4         0.2  setosa
## 3           4.7         3.2          1.3         0.2  setosa
## 4           4.6         3.1          1.5         0.2  setosa
## 5           5.0         3.6          1.4         0.2  setosa
## 6           5.4         3.9          1.7         0.4  setosa
## 7           4.6         3.4          1.4         0.3  setosa
## 8           5.0         3.4          1.5         0.2  setosa
## 9           4.4         2.9          1.4         0.2  setosa
## 10          4.9         3.1          1.5         0.1  setosa
</code></pre>
--------------------------------------------------------------------------------------
### Machine Learning
You can orchestrate machine learning algorithms in a Spark cluster via the machine learning functions within sparklyr. These functions connect to a set of high-level APIs built on top of DataFrames that help you create and tune machine learning workflows.

Here’s an example where we use ml_linear_regression to fit a linear regression model. We’ll use the built-in mtcars dataset, and see if we can predict a car’s fuel consumption (mpg) based on its weight (wt), and the number of cylinders the engine contains (cyl). We’ll assume in each case that the relationship between mpg and each of our features is linear.
<pre><code>
# copy mtcars into spark
mtcars_tbl <- copy_to(sc, mtcars)

# transform our data set, and then partition into 'training', 'test'
partitions <- mtcars_tbl %>%
  filter(hp >= 100) %>%
  mutate(cyl8 = cyl == 8) %>%
  sdf_partition(training = 0.5, test = 0.5, seed = 1099)
</code></pre>
<pre><code>
# fit a linear model to the training dataset
fit <- partitions$training %>%
  ml_linear_regression(response = "mpg", features = c("wt", "cyl"))
## * No rows dropped by 'na.omit' call
fit
## Call: ml_linear_regression(., response = "mpg", features = c("wt", "cyl"))
## 
## Coefficients:
## (Intercept)          wt         cyl 
##   33.499452   -2.818463   -0.923187
</code></pre>
For linear regression models produced by Spark, we can use summary() to learn a bit more about the quality of our fit, and the statistical significance of each of our predictors.

summary(fit)
## Call: ml_linear_regression(., response = "mpg", features = c("wt", "cyl"))
## 
## Deviance Residuals::
##    Min     1Q Median     3Q    Max 
## -1.752 -1.134 -0.499  1.296  2.282 
## 
## Coefficients:
##             Estimate Std. Error t value  Pr(>|t|)    
## (Intercept) 33.49945    3.62256  9.2475 0.0002485 ***
## wt          -2.81846    0.96619 -2.9171 0.0331257 *  
## cyl         -0.92319    0.54639 -1.6896 0.1518998    
## ---
## Signif. codes:  0 '***' 0.001 '**' 0.01 '*' 0.05 '.' 0.1 ' ' 1
## 
## R-Squared: 0.8274
## Root Mean Squared Error: 1.422
</code></pre>
Spark machine learning supports a wide array of algorithms and feature transformations and as illustrated above it’s easy to chain these functions together with dplyr pipelines. To learn more see the machine learning section.

### Reading and Writing Data
You can read and write data in CSV, JSON, and Parquet formats. Data can be stored in HDFS, S3, or on the local filesystem of cluster nodes.

<pre><code>
temp_csv <- tempfile(fileext = ".csv")
temp_parquet <- tempfile(fileext = ".parquet")
temp_json <- tempfile(fileext = ".json")

spark_write_csv(iris_tbl, temp_csv)
iris_csv_tbl <- spark_read_csv(sc, "iris_csv", temp_csv)

spark_write_parquet(iris_tbl, temp_parquet)
iris_parquet_tbl <- spark_read_parquet(sc, "iris_parquet", temp_parquet)

spark_write_json(iris_tbl, temp_json)
iris_json_tbl <- spark_read_json(sc, "iris_json", temp_json)

src_tbls(sc)
## [1] "batting"      "flights"      "iris"         "iris_csv"    
## [5] "iris_json"    "iris_parquet" "mtcars"
</code></pre>

### Distributed R
You can execute arbitrary r code across your cluster using spark_apply. 
For example, we can apply ``rgamma`` over iris as follows:
<pre><code>
spark_apply(iris_tbl, function(data) {
  data[1:4] + rgamma(1,2)
})
## # Source:   table<sparklyr_tmp_118fd5007f7aa> [?? x 4]
## # Database: spark_connection
##    Sepal_Length Sepal_Width Petal_Length Petal_Width
##           <dbl>       <dbl>        <dbl>       <dbl>
##  1      7.06982     5.46982      3.36982     2.16982
##  2      6.86982     4.96982      3.36982     2.16982
##  3      6.66982     5.16982      3.26982     2.16982
##  4      6.56982     5.06982      3.46982     2.16982
##  5      6.96982     5.56982      3.36982     2.16982
##  6      7.36982     5.86982      3.66982     2.36982
##  7      6.56982     5.36982      3.36982     2.26982
##  8      6.96982     5.36982      3.46982     2.16982
##  9      6.36982     4.86982      3.36982     2.16982
## 10      6.86982     5.06982      3.46982     2.06982
## # ... with more rows
</code></pre>
You can also group by columns to perform an operation over each group of rows and make use of any package within the closure:
<pre><code>
spark_apply(
  iris_tbl,
  function(e) broom::tidy(lm(Petal_Width ~ Petal_Length, e)),
  names = c("term", "estimate", "std.error", "statistic", "p.value"),
  group_by = "Species"
)
## # Source:   table<sparklyr_tmp_118fd583f9c2b> [?? x 6]
## # Database: spark_connection
##      Species         term    estimate  std.error  statistic      p.value
##        <chr>        <chr>       <dbl>      <dbl>      <dbl>        <dbl>
## 1 versicolor  (Intercept) -0.08428835 0.16070140 -0.5245029 6.023428e-01
## 2 versicolor Petal_Length  0.33105360 0.03750041  8.8279995 1.271916e-11
## 3  virginica  (Intercept)  1.13603130 0.37936622  2.9945505 4.336312e-03
## 4  virginica Petal_Length  0.16029696 0.06800119  2.3572668 2.253577e-02
## 5     setosa  (Intercept) -0.04822033 0.12164115 -0.3964146 6.935561e-01
## 6     setosa Petal_Length  0.20124509 0.08263253  2.4354220 1.863892e-02
</code></pre>

