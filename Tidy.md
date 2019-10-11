Tidy data
========================================================
author: Clay Wright
date: 10
autosize: true

What is Tidy data?
========================================================
Hadley Wickam:
>It is often said that 80% of data analysis is spent on the process of cleaning and preparing
the data (Dasu and Johnson 2003). Data preparation is not just a first step, but must be
repeated many over the course of analysis as new problems come to light or new data is
collected.

[Author of R for data science](<https://r4ds.had.co.nz/tidy-data.html>) 

Tidy datasets are **structured to facilitate analysis** now and in the future. 


*The Principles of Tidy Data* provide a standard organization that facilitates initial exploration and analysis as well as simplifying the use of data analysis tools and existing pipelines. 

Using R you can create a pipeline that will allow you to easily repeat an analysis.

The Principles of Tidy Data
=========================================================
Data is a collection of values (numbers or letters/strings). 
Datasets generally have rows and columns. 
Values (cells in a spreadsheet) are associated with variables (what is measured or recorded) and an observation (an associated set of values). 

In a Tidy dataset:  
1. Each variable must have its own column.  
2. Each observation must have its own row.  
3. Each value must have its own cell.  

![](tidy-1.png)

Why?
=========================================================
This consistent structure will allow you to use many tools to facilitate restructuring and creating derived variables. 
>Many of these tools are contained in a collection of R packages called the tidyverse

```r
#install and load the tidyverse packages
if(!("tidyverse" %in% rownames(installed.packages())))
  install.packages("tidyverse")
library("tidyverse")
```

This structure will also provide clues as to how the data can/should be plotted based on the types and numbers of variables. 

A few key ideas
========================================================
- long data *vs* wide data  
Wide datasets often have columns which represent values of a variable and cells representing multiple observations *e.g.*

```r
table4a
```

```
# A tibble: 3 x 3
  country     `1999` `2000`
* <chr>        <int>  <int>
1 Afghanistan    745   2666
2 Brazil       37737  80488
3 China       212258 213766
```
Here 1999 and 2000 are year variables and the observations for each year in each country are the rows. 

long data *vs* wide data cont.
========================================================
How can we make this dataset tidy?  
We need to **gather** the 1999 and 2000 columns into a single observation column and create a year variable column. To do this we now use the `pivot_longer` function


```r
table4a %>% 
  pivot_longer(-country, names_to = "year", values_to = "cases")
```

```
# A tibble: 6 x 3
  country     year   cases
  <chr>       <chr>  <int>
1 Afghanistan 1999     745
2 Afghanistan 2000    2666
3 Brazil      1999   37737
4 Brazil      2000   80488
5 China       1999  212258
6 China       2000  213766
```

long data *vs* wide data cont.
========================================================
-   
Sometimes observations will be spread across rows *e.g.*

```r
head(table2)
```

```
# A tibble: 6 x 4
  country      year type           count
  <chr>       <int> <chr>          <int>
1 Afghanistan  1999 cases            745
2 Afghanistan  1999 population  19987071
3 Afghanistan  2000 cases           2666
4 Afghanistan  2000 population  20595360
5 Brazil       1999 cases          37737
6 Brazil       1999 population 172006362
```

long data *vs* wide data cont.
========================================================
To tidy this up we need to spread the "type" and "count" columns into two columns. 
The function for this used to be called `spread`, but is now called `pivot_wider`.

```r
table2 %>% 
  pivot_wider(names_from = "type", values_from = "count")
```

```
# A tibble: 6 x 4
  country      year  cases population
  <chr>       <int>  <int>      <int>
1 Afghanistan  1999    745   19987071
2 Afghanistan  2000   2666   20595360
3 Brazil       1999  37737  172006362
4 Brazil       2000  80488  174504898
5 China        1999 212258 1272915272
6 China        2000 213766 1280428583
```


The problem - crazy spectrophotometer datasets
========================================================
xlsx dataset 
- metadata  
- procedural details  
- layout  
    - some categorical variables  
- measurements at different wavelengths overtime   
    - continuous response variable  
- results (summaries)  

Getting the data into R
============================================================
We will use the readxl package, which is not in the core tidyverse packages.


```r
#install.packages("readxl")
library(readxl)
```


```r
path <- "11_96wellplate_50%sonicfresh_pH6.5vs7.5_0.1mgmlchl_30seclight3mindarkconstshake_redlight_0.5mMDCPIPfresh.xlsx"
data <- read_excel(path = path)

which(data[,1] == "Layout")
```

```
[1] 92
```

```r
range <- paste0("B", which(data[,1] == "Layout") + 4, ":O", which(data[,1] == "Layout") + 4 + 3*8)
layout <- read_excel(path, range = range)
```

Getting the data into R
============================================================
Look at all of these missing values!
Now we need to fill in column 1 

```r
#rename weird columns
layout %>% rename(row = ...1) -> layout 
layout %>% rename(variable = ...14) -> layout
#fill in rows
layout %>% 
  fill(row) -> layout
#pivot longer
layout %>%
  pivot_longer(-c(row,variable), 
         names_to = "column", values_to = "values") -> layout
```


Tidy layout data
============================================================
We can go ahead and remove any rows that don't have data in values. 

```r
layout %>% drop_na() -> layout
```
And split the variable column into 2.

```r
layout %>% 
  pivot_wider(names_from = variable, values_from = values) -> layout
```

Finally let's make a `well` variable combining `row` and `column`

```r
layout %>% unite(row, column, col = "well", sep = "") -> layout
#layout$well <- paste0(layout$row, layout$column)
head(layout)
```

```
# A tibble: 6 x 4
  well  `Well ID` Name         `Conc/Dil`
  <chr> <chr>     <chr>        <chr>     
1 B2    SPL1      6.5_CP       <NA>      
2 B3    SPL1      6.5_CP       <NA>      
3 B4    SPL1      6.5_CP       <NA>      
4 B6    SPL2      6.5_CP+DCPIP <NA>      
5 B7    SPL2      6.5_CP+DCPIP <NA>      
6 B8    SPL2      6.5_CP+DCPIP <NA>      
```


Getting the real data 
=============================================================


```r
begin_595 <- which(data[,1] == "595")
begin_750 <- which(data[,1] == "750")
range <- paste0("R", begin_595+4, "C2:R", begin_750, "C", length(unique(layout$well)) + 3)
data_595 <- read_excel(path, range = range)
```


```r
end_750 <- which(data[,1] == "Results")
range <- paste0("R", begin_750+4, "C2:R", end_750, "C", length(unique(layout$well)) + 3)
data_750 <- read_excel(path, range = range)
```

Assembling the complete, TIDY dataset
=============================================================
Strategy
- Make the measurements long
- Make a timepoint column to join the measurements on
    - since times differ between 595 and 750 measurements
- join the measurements
- join the layout data

Assembling the complete, TIDY dataset
=============================================================

```r
data_595 %>% pivot_longer(-(1:2), names_to = "well", values_to = "A_595") -> data_595
data_750 %>% pivot_longer(-(1:2), names_to = "well", values_to = "A_750") -> data_750

data_595$timepoint <- as.numeric(as.factor(data_595$Time))
data_750$timepoint <- as.numeric(as.factor(data_750$Time))
  
data <- inner_join(data_595, data_750, by = c("timepoint", "well"))
data <- left_join(data, layout)
head(data)
```

```
# A tibble: 6 x 11
  Time.x              `T° 595` well  A_595 timepoint Time.y             
  <dttm>                 <dbl> <chr> <dbl>     <dbl> <dttm>             
1 1899-12-31 00:00:05     23.3 B2     1.18         1 1899-12-31 00:00:46
2 1899-12-31 00:00:05     23.3 B3     1.24         1 1899-12-31 00:00:46
3 1899-12-31 00:00:05     23.3 B4     1.40         1 1899-12-31 00:00:46
4 1899-12-31 00:00:05     23.3 B6     1.20         1 1899-12-31 00:00:46
5 1899-12-31 00:00:05     23.3 B7     1.25         1 1899-12-31 00:00:46
6 1899-12-31 00:00:05     23.3 B8     1.16         1 1899-12-31 00:00:46
# … with 5 more variables: `T° 750` <dbl>, A_750 <dbl>, `Well ID` <chr>,
#   Name <chr>, `Conc/Dil` <chr>
```

Looking forward
=============================================================
Now this looks like a Tidy dataset!

We want to produce a graph which hopefully will show that the 595/750 ratio for trials without DCPIP added do not change at all, while the trials with DCPIP added have a slightly negative slope, indicating that abs 595 is decreasing (reduction of DCPIP) relative to abs 750 (reference). We also want to compare pH 6.5 to 7.5.

To do this we will need to:
- Create an average time column (lubridate and dplyr)
- Create a 595/750 ratio column (dplyr)
- Split the name column into pH and DCPIP variables (dplyr)
- Make a separate dataset for the standard curve




```
Error in parse(text = x, srcfile = src) : <text>:10:1: unexpected '['
9: strtoi(as.difftime(data$Time.y, format = "%Y-%m-%d %H:%M:%S %Z", units = "mins"))
10: [
    ^
```
