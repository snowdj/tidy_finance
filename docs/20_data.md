# (PART\*) Financial data {.unnumbered}

# Accessing & managing financial data

In this chapter, we propose a way to organize your financial data. Everybody, who has experience with data, is also familiar with storing data in various formats like CSV, XLS, XLSX, or other delimited value stores. Reading and saving data can become very cumbersome in the case of using different data formats, both across different projects, as well as across different programming languages. Moreover, storing data in delimited files often leads to problems with respect to column type consistency. For instance, date-type columns frequently lead to inconsistencies across different data formats and programming languages. 

This chapter shows how to import different open source data sets. Specifically, our data comes from the application programming interface (API) of Yahoo!Finance, a downloaded standard CSV files, an XLSX file stored in a public Google drive repositories and other macroeconomic time series. We store all the data in a *single* database, which serves as the only source of data in subsequent chapters. We conclude the chapter by providing some tips on managing databases. 

First, we load the global packages that we use throughout this chapter. Later on, we load more packages in the sections where we need them. 


```r
library(tidyverse)
library(lubridate)
library(scales)
```

Moreover, we initially define the date range for which we fetch and store the financial data, making future data updates tractable. In case you need another time frame, you need to adjust these dates. Our data starts with 1960 since most asset pricing studies use data from 1962 on. 


```r
start_date <- ymd("1960-01-01")
end_date <- ymd("2020-12-31")
```

## Fama-French data

We start by downloading some famous Fama-French factors [e.g., @Fama1993] and portfolio returns commonly used in empirical asset pricing. Fortunately, there is a neat package by [Nelson Areal](https://github.com/nareal/frenchdata/) that allows us to easily access the data: the `frenchdata` package provides functions to download and read data sets from [Prof. Kenneth French finance data library](https://mba.tuck.dartmouth.edu/pages/faculty/ken.french/data_library.html).


```r
library(frenchdata)
```

We can use the main function of the package to download monthly Fama-French factors. The set *3 Factors* includes the return time series of the market, size, and value factors alongside the risk-free rates. Note that we have to do some manual work to correctly parse all the columns and scale them appropriately as the raw Fama-French data comes in very unpractical data format. For precise descriptions of the variables, we suggest consulting [Prof. Kenneth French finance data library](https://mba.tuck.dartmouth.edu/pages/faculty/ken.french/data_library.html) directly. If you are on the site, check the raw data files to appreciate the time saved by `frenchdata`.


```r
factors_ff_monthly_raw <- download_french_data("Fama/French 3 Factors")
factors_ff_monthly <- factors_ff_monthly_raw$subsets$data[[1]] |>
  transmute(
    month = floor_date(ymd(str_c(date, "01")), "month"),
    rf = as.numeric(RF) / 100,
    mkt_excess = as.numeric(`Mkt-RF`) / 100,
    smb = as.numeric(SMB) / 100,
    hml = as.numeric(HML) / 100
  ) |>
  filter(month >= start_date & month <= end_date)
```

It is straightforward to download the corresponding *daily* Fama-French factors with the same function. 


```r
factors_ff_daily_raw <- download_french_data("Fama/French 3 Factors [Daily]")
factors_ff_daily <- factors_ff_daily_raw$subsets$data[[1]] |>
  transmute(
    date = ymd(date),
    rf = as.numeric(RF) / 100,
    mkt_excess = as.numeric(`Mkt-RF`) / 100,
    smb = as.numeric(SMB) / 100,
    hml = as.numeric(HML) / 100
  ) |>
  filter(date >= start_date & date <= end_date)
```

In a subsequent chapter, we also use the 10 monthly industry portfolios, so let us fetch that data, too. 


```r
industries_ff_monthly_raw <- download_french_data("10 Industry Portfolios")
industries_ff_monthly <- industries_ff_monthly_raw$subsets$data[[1]] |>
  mutate(month = floor_date(ymd(str_c(date, "01")), "month")) |>
  mutate(across(where(is.numeric), ~ . / 100)) |>
  select(month, everything(), -date) |>
  filter(month >= start_date & month <= end_date)
```

It is worth taking a look at all available portfolio return time series from Kenneth French's homepage. You should check out the other sets by calling `get_french_data_list()`.

## q-factors

In recent years, the academic discourse experienced the rise of alternative factor models, e.g., in the form of the @Hou2015 *q*-factor model. We refer to the [extended background](http://global-q.org/background.html) information provided by the original authors for further information. The *q* factors can be downloaded directly from the authors' homepage from within `read_csv()`.

We also need to adjust this data. First, we discard information we will not use here. Then, we rename the columns with the "R_"-prescript using regular expressions and write all column names in lower case. You can try sticking to a consistent style for naming objects, which we try to illustrate here - the emphasis is on *try*. You can check out style guides available online, e.g., [Hadley Wickham's `tidyverse` style guide](https://style.tidyverse.org/index.html).


```r
factors_q_monthly_link <- 
  "http://global-q.org/uploads/1/2/2/6/122679606/q5_factors_monthly_2020.csv"
factors_q_monthly <- read_csv(factors_q_monthly_link) |>
  mutate(month = ymd(str_c(year, month, "01", sep = "-"))) |>
  select(-R_F, -R_MKT, -year) |>
  rename_with(~ str_remove(., "R_")) |>
  rename_with(~ str_to_lower(.)) |>
  mutate(across(-month, ~ . / 100)) |>
  filter(month >= start_date & month <= end_date)
```

## Macroeconomic predictors

Our next data source is a set of macroeconomic variables often used as predictors for the equity premium. @Goyal2008 comprehensively reexamine the performance of variables suggested by the academic literature to be good predictors of the equity premium. The authors host the data updated to 2020 on [Amit Goyal's website](https://sites.google.com/view/agoyal145). Since the data is a .xlsx-file stored on a public Google drive location, we need additional packages to access the data directly from our R session. Therefore, we load `readxl` to read the .xlsx-file and `googledrive` for the Google drive connection.


```r
library(readxl)
library(googledrive)
```

Usually, you need to authenticate if you interact with Google drive directly in R. Since the data is stored via a public link, we can proceed without any authentication. 


```r
drive_deauth()
```

The `drive_download()` function from the `googledrive` package allows us to download the data and store it locally.


```r
macro_predictors_link <- 
  "https://drive.google.com/file/d/1ACbhdnIy0VbCWgsnXkjcddiV8HF4feWv/view"
drive_download(
  macro_predictors_link, 
  path = "data/macro_predictors.xlsx"
  )
```

Next, we read in the new data and transform the columns to the variables that we later use. You can consult the material on [Amit Goyal's website](https://sites.google.com/view/agoyal145) for the definitions of the variables and the transformations.


```r
macro_predictors <- read_xlsx(
  "data/macro_predictors.xlsx", 
  sheet = "Monthly"
) |>
  mutate(month = ym(yyyymm)) |>
  filter(month >= start_date & month <= end_date) |>
  mutate(across(where(is.character), as.numeric)) |>
  mutate(
    IndexDiv = Index + D12,
    logret = log(IndexDiv) - log(lag(IndexDiv)),
    Rfree = log(Rfree + 1),
    rp_div = lead(logret - Rfree, 1), # Future excess market return
    dp = log(D12) - log(Index), # Dividend Price ratio
    dy = log(D12) - log(lag(Index)), # Dividend yield
    ep = log(E12) - log(Index), # Earnings price ratio
    de = log(D12) - log(E12), # Dividend payout ratio
    tms = lty - tbl, # Term spread
    dfy = BAA - AAA # Default yield spread
  ) |>
  select(month, rp_div, dp, dy, ep, de, svar,
         bm = `b/m`, ntis, tbl, lty, ltr,
         tms, dfy, infl
  ) |>
  drop_na()
```

Finally, after reading in the macro predictors to our memory, we remove the raw data file from our temporary storage. 


```r
file.remove("data/macro_predictors.xlsx")
```

```
[1] TRUE
```

## Other macroeconomic data

The Federal Reserve bank of St. Louis provides the Federal Reserve Economic Data (FRED), an extensive database for macroeconomic data. In total, there are 817,000 US and international time series from 108 different sources. As an illustration, we use the already familiar `tidyquant` package to fetch consumer price index (CPI) data that can be found under the [CPIAUCNS](https://fred.stlouisfed.org/series/CPIAUCNS) key. 


```r
library(tidyquant)

cpi_monthly <- tq_get("CPIAUCNS",
  get = "economic.data",
  from = start_date, to = end_date
) |>
  transmute(
    month = floor_date(date, "month"),
    cpi = price / price[month == max(month)]
  )
```

To download other time series, we just have to it up on the FRED website and extract the corresponding key from the address. For instance, the produce price index for gold ores can be found under the [PCU2122212122210](https://fred.stlouisfed.org/series/) key. The `tidyquant` package provides access to around 10,000 time series of the FRED database. If your desired time series is not included, we recommend working with the `fredr` package [@fredr]. Note that you need to get an API key to use its functionality, but refer to the package documentation for details. 

## Setting up a database

Now that we have downloaded some data from the web into the memory of our R session, let us set up a database to store that information for future use. We will use the data stored in this database throughout the following chapters, but you could alternatively implement a different strategy and replace the respective code. 

There are many ways to set up and organize a database, depending on the use case. For our purpose, the most efficient way is to use an [SQLite](https://www.sqlite.org/index.html) database, which is the C-language library that implements a small, fast, self-contained, high-reliability, full-featured, SQL database engine. Note that [SQL](https://en.wikipedia.org/wiki/SQL) (Structured Query Language) is a standard language for accessing and manipulating databases, and it heavily inspired the `dplyr` functions. We refer to [this tutorial](https://www.w3schools.com/sql/sql_intro.asp) for more information on SQL. 

There are two packages that make working with SQLite in R very simple: `RSQLite` embeds the SQLite database engine in R and `dbplyr` is the database back-end for `dplyr`. These packages allow to set up a database to remotely store tables and use these remote database tables as if they are in-memory data frames by automatically converting `dplyr` into SQL. Check out the [RSQLite](https://cran.r-project.org/web/packages/RSQLite/vignettes/RSQLite.html) and [dbplyr vignettes](https://db.rstudio.com/databases/sqlite/) for more information.


```r
library(RSQLite)
library(dbplyr)
```

A SQLite database is easily created - the code below is really all there is. Note that we use the `extended_types=TRUE` option to enable date types when storing and fetching data, otherwise date columns are stored as integer values. 


```r
tidy_finance <- dbConnect(
  SQLite(), 
  "data/tidy_finance.sqlite", 
  extended_types = TRUE
)
```

Next, we create a remote table with the monthly Fama-French factor data. Notice that we use the base R pipe placeholder `_` and a named argument to pipe  `factors_ff_monthly` to the argument `value`.


```r
factors_ff_monthly |>
  dbWriteTable(
    tidy_finance, 
    "factors_ff_monthly", 
    value = _, 
    overwrite = TRUE
  )
```

We can use the remote table as an in-memory data frame by building a connection via `tbl()`.


```r
factors_ff_monthly_db <- tbl(tidy_finance, "factors_ff_monthly")
```

All `dplyr` calls are evaluated lazily, i.e., the data is not in the memory of our R session, and actually, the database does most of the work. You can see that by noticing that the output below does not show the number of rows. In fact, the following code chunk only fetches the top 10 rows from the database for printing.  


```r
factors_ff_monthly_db |>
  select(month, rf)
```

```
# Source:   SQL [?? x 2]
# Database: sqlite 3.38.5 [C:\Users\christoph.scheuch\Documents\GitHub\tidy_finance\data\tidy_finance.sqlite]
  month          rf
  <date>      <dbl>
1 1960-01-01 0.0033
2 1960-02-01 0.0029
3 1960-03-01 0.0035
4 1960-04-01 0.0019
5 1960-05-01 0.0027
# … with more rows
```

If we want to have the whole table in memory, we need to `collect()` it. You will see that we regularly load the data into the memory in the next chapters.


```r
factors_ff_monthly_db |>
  select(month, rf) |>
  collect()
```

```
# A tibble: 732 × 2
  month          rf
  <date>      <dbl>
1 1960-01-01 0.0033
2 1960-02-01 0.0029
3 1960-03-01 0.0035
4 1960-04-01 0.0019
5 1960-05-01 0.0027
# … with 727 more rows
```

The last couple of code chunks are really all there is to organize a simple database! You can also share the SQLite database across devices and programming languages. 

Before we move on to the next data source, let us also store the other four tables in our new SQLite database. 


```r
factors_ff_daily |>
  dbWriteTable(
    tidy_finance, 
    "factors_ff_daily", 
    value = _, 
    overwrite = TRUE
  )

industries_ff_monthly |>
  dbWriteTable(
    tidy_finance, 
    "industries_ff_monthly", 
    value = _, 
    overwrite = TRUE
  )

factors_q_monthly |>
  dbWriteTable(
    tidy_finance, 
    "factors_q_monthly", 
    value = _, 
    overwrite = TRUE
  )

macro_predictors |>
  dbWriteTable(
    tidy_finance, 
    "macro_predictors", 
    value = _, 
    overwrite = TRUE
  )

cpi_monthly |>
  dbWriteTable(
    tidy_finance, 
    "cpi_monthly", 
    value = _, 
    overwrite = TRUE
  )
```

From now on, all you need to do to access data that is stored in the database is to follow three steps: (i) Establish the connection to the SQLite database, (ii) call the table you want to extract, and (iii) collect the data. For your convenience, the following steps show all you need in a compact fashion.


```r
library(tidyverse)
library(RSQLite)
tidy_finance <- dbConnect(
  SQLite(), 
  "data/tidy_finance.sqlite", 
  extended_types = TRUE
)
factors_q_monthly <- tbl(tidy_finance, "factors_q_monthly")
factors_q_monthly <- factors_q_monthly |> collect()
```

## Managing SQLite databases

Finally, at the end of our data chapter, we revisit the SQLite database itself. When you drop database objects such as tables or delete data from tables, the database file size remains unchanged because SQLite just marks the deleted objects as free and reserves their space for the future uses. As a result, the database file always grows in size.

To optimize the database file, you can run the `VACUUM` command in the database, which rebuilds the database and frees up unused space. You can execute the command in the database using the `dbSendQuery()` function. 


```r
dbSendQuery(tidy_finance, "VACUUM")
```

```
<SQLiteResult>
  SQL  VACUUM
  ROWS Fetched: 0 [complete]
       Changed: 0
```

The `VACUUM` command actually performs a couple of additional cleaning steps, which you can read up in [this tutorial](https://www.sqlitetutorial.net/sqlite-vacuum/). 

Apart from cleaning up, you might be interested in listing all the tables that are currently in your database. You can do this via the `dbListTables()` function. 


```r
dbListTables(tidy_finance)
```

```
Warning: Closing open result set, pending rows
```

```
 [1] "beta"                  "compustat"             "cpi_monthly"          
 [4] "crsp_daily"            "crsp_monthly"          "factors_ff_daily"     
 [7] "factors_ff_monthly"    "factors_q_monthly"     "industries_ff_monthly"
[10] "macro_predictors"     
```

This function comes in handy if you are unsure about the correct naming of the tables in your database. 

## Exercises

1. Download the monthly Fama-French factors manually from [Ken French's data library](https://mba.tuck.dartmouth.edu/pages/faculty/ken.french/data_library.html) and read them in via `read_csv()`. Validate that you get the same data as via the `frenchdata` package. 
1. Download the Fama-French 5 factors using the package. Then, compare the estimates of the three factors that are common to the Three-Factor model. Explain your findings.
