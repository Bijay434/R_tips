You can use data.table or tidyverse (or Pandas)!
================
Erika Duan
2023-01-08

- <a href="#introduction" id="toc-introduction">Introduction</a>
- <a href="#creating-a-test-dataset"
  id="toc-creating-a-test-dataset">Creating a test dataset</a>
- <a href="#data-frame-conversions" id="toc-data-frame-conversions">Data
  frame conversions</a>
- <a href="#code-syntax-differences" id="toc-code-syntax-differences">Code
  syntax differences</a>
- <a href="#filter-data" id="toc-filter-data">Filter data</a>
  - <a href="#filter-by-a-single-condition"
    id="toc-filter-by-a-single-condition">Filter by a single condition</a>
  - <a href="#filter-by-multiple-conditions"
    id="toc-filter-by-multiple-conditions">Filter by multiple conditions</a>
  - <a href="#filter-using-regular-expressions"
    id="toc-filter-using-regular-expressions">Filter using regular
    expressions</a>
  - <a href="#filter-across-multiple-columns"
    id="toc-filter-across-multiple-columns">Filter across multiple
    columns</a>
  - <a href="#equivalent-pandas-code"
    id="toc-equivalent-pandas-code">Equivalent Pandas code</a>
  - <a href="#code-benchmarking" id="toc-code-benchmarking">Code
    benchmarking</a>
- <a href="#sort-data" id="toc-sort-data">Sort data</a>
  - <a href="#equivalent-pandas-code-1"
    id="toc-equivalent-pandas-code-1">Equivalent Pandas code</a>
  - <a href="#code-benchmarking-1" id="toc-code-benchmarking-1">Code
    benchmarking</a>
- <a href="#select-columns" id="toc-select-columns">Select columns</a>
  - <a href="#equivalent-pandas-code-2"
    id="toc-equivalent-pandas-code-2">Equivalent Pandas code</a>
  - <a href="#code-benchmarking-2" id="toc-code-benchmarking-2">Code
    benchmarking</a>
- <a href="#transform-columns" id="toc-transform-columns">Transform
  columns</a>
  - <a href="#transform-via-copy-on-modify-versus-modify-in-place"
    id="toc-transform-via-copy-on-modify-versus-modify-in-place">Transform
    via copy on modify versus modify in place</a>
  - <a href="#transform-using-multiple-conditions"
    id="toc-transform-using-multiple-conditions">Transform using multiple
    conditions</a>
  - <a href="#equivalent-pandas-code-3"
    id="toc-equivalent-pandas-code-3">Equivalent Pandas code</a>
  - <a href="#code-benchmarking-3" id="toc-code-benchmarking-3">Code
    benchmarking</a>
- <a href="#group-by-and-summarise-columns"
  id="toc-group-by-and-summarise-columns">Group by and summarise
  columns</a>
  - <a href="#equivalent-pandas-code-4"
    id="toc-equivalent-pandas-code-4">Equivalent Pandas code</a>
  - <a href="#code-benchmarking-4" id="toc-code-benchmarking-4">Code
    benchmarking</a>
- <a href="#chain-code" id="toc-chain-code">Chain code</a>
  - <a href="#equivalent-pandas-code-5"
    id="toc-equivalent-pandas-code-5">Equivalent Pandas code</a>
- <a href="#other-resources" id="toc-other-resources">Other resources</a>

``` r
# Load required R packages -----------------------------------------------------
if (!require("pacman")) install.packages("pacman")
pacman::p_load(here,
               tidyverse,
               data.table,
               compare, # Compare results between two data frames   
               microbenchmark,
               reticulate) # Required for the Python environment interface
```

``` r
# Check Python configurations --------------------------------------------------
# I first created & activated my Python virtual environment using venv [via terminal]
# I then installed pandas inside my virtual environment using pip [via terminal]      

# Force R to use my Python virtual environment
use_virtualenv("C:/Windows/system32/py_396_data_science")

# Use reticulate::py_discover_config() to check Python environment settings 
```

# Introduction

I started data analysis in `R` with the `tidyverse` suite of packages
and `dplyr` still sparks the greatest joy out of all R and Python data
wrangling packages. In R, a useful alternative to `dplyr` is
`data.table` when you need to perform lots of groupings on large
datasets (datasets over 1 million rows).

The pros and cons of using `tidyverse`, `data.table` and the popular
Python data wrangling package `Pandas` are listed below, keeping in mind
that some preferences are subjective.

| Package           | `dplyr`                                                                | `data.table`                                                                                               | `Pandas`                                                                                            |
|:------------------|:-----------------------------------------------------------------------|:-----------------------------------------------------------------------------------------------------------|:----------------------------------------------------------------------------------------------------|
| Code readability  | Code is highly readable and integrates well with `\|>` and `%>%` pipes | `data.table` code is more concise, but suffers from decreased readability especially when using long names | `Pandas` allows chaining of methods but its methods syntax can feel less intuitive than `dplyr`     |
| Speed: `group by` | Code fails to execute when doing analysis on groups with \~300K values | Performance remains consistent as the number of groups increase (\>300K groups)                            | Performs equivalently to `dplyr` according to [db-benchmark](https://h2oai.github.io/db-benchmark/) |
| Speed: `sort`     | \-                                                                     | Faster alternative `sort` method to `dplyr` on large datasets (\>1M rows)                                  | \-                                                                                                  |
| Memory usage      | Is copy on modify, which is default R behaviour                        | Allows modify by reference for some operations, which decreases memory allocation needs                    | \-                                                                                                  |

Let’s test this out for ourselves.

# Creating a test dataset

Imagine you have a dataset describing how students are engaging with
online courses:

- Each student has a unique ID.  
- There are 5 different online platforms labelled `A`, `B`, `C`, `D` and
  `E`.  
- Students can take multiple courses at any time.  
- Students have the option of taking different courses within the same
  platform or switching to a different course on a different platform.  
- Platform start dates are recorded when the student first signs up to a
  platform.  
- Platform end dates are recorded when a student completes their last
  course or switches to a new platform.

We can source the R script
[`./dc-dataset_generation_script.R`](./dc-dataset_generation_script.R)
to generate a mock dataset of 1M course enrollments, which is named
`courses_df`.

``` r
# Source R script to generate mock dataset -------------------------------------
# Explicitly state that outputs are generated in the global R environment   
source("dc-dataset_generation_script.R", local = knitr::knit_global())
```

``` r
# Preview mock dataset ---------------------------------------------------------
head(courses_df, 5)
```

**Note:** The script `dc-dataset_generation_script.R` may take several
minutes to load whilst it generates 1M synthetic records.

# Data frame conversions

The [R script above](./dc-dataset_generation_script.R) generates a
`data.table` object named `courses_df`. In R, the default data object is
a `data.frame`. A `data.frame` is just a list of vectors (of the same
length) with each column represented by a separate vector.

The `tibble` data object from `dplyr` is a modified version of the
`data.frame` that prohibits some column name and type conversions. You
can read more about `tibble` design choices in the Advanced R [chapter
on tibbles](https://adv-r.hadley.nz/vectors-chap.html?q=tibble#tibble).

The `data.table` package contains the function `setDT()`, which converts
a `data.frame` or tibble into a `data.table` object by reference
(without creating a second copy of the original object).

**Note:** `courses_df` is loaded as a `data.table` object.

``` r
# Check data object type -------------------------------------------------------
# Check the class of a data.table object  
typeof(courses_df)
#> [1] "list"  

class(courses_df)
#> [1] "data.table" "data.frame"   

# Check the class of a default R data object created using as.data.frame()
data.frame(id = 1:5, letter = letters[1:5]) |>
  class()
#> [1] "data.frame"  

# Check the class of a tibble data object 
data.frame(id = 1:5, letter = letters[1:5]) |>
  as_tibble() %>%  
  class()
#> [1] "tbl_df"     "tbl"        "data.frame"  
```

As this tutorial showcases equivalent Python `Pandas` code, we will also
import the `Pandas` package into our Python environment and create a
copy of `course_df` in our Python environment.

``` python
# Import Python packages for Python data transformations -----------------------
import pandas as pd  
import datetime as dt
import numpy as np

# Load courses_df into the Python environment as a Pandas data frame -----------
# Creates a copy of courses_df from the R environment in the Python environment 
courses_pf = r.courses_df

# Convert platform_start_date and platform_end_date to date type ---------------
# Apply a vectorised datetime type conversion for each column containing "date" 

# Extract a list of column names containing "date"  
date_cols = [col for col in courses_pf.columns if "date" in col]

# Create a for loop which converts a column into the YYYY-mm-dd date format
for col in date_cols:
  courses_pf[col] = pd.to_datetime(courses_pf[col], format = "%Y-%m-%d")
  
courses_pf.dtypes
```

# Code syntax differences

Before we cover specific examples, it is useful to understand the syntax
of `dplyr` and `data.table` functions. The `dplyr` syntax is more
beginner friendly as it comprises a list of data transformation verbs,
which can be chained together to create a new output.

<img src="../../figures/dc-code_syntax.svg" width="80%" />

In contrast, the `data.table` syntax is structured in the form
`DT[i, j, by]` where:

- Data selection (filtering or sorting rows) is performed in the `i`
  placeholder.  
- Data column selection, transformation and deletion are performed in
  the `j` placeholder.  
- Grouping data is performed in the `by` placeholder, with aggregation
  functions specified in the `j` placeholder.

# Filter data

## Filter by a single condition

The code to filter data using `dplyr` or `data.table` is very similar
and `data.table` objects can also be directly transformed using `dplyr`
functions.

``` r
# Filter by platform B and C using dplyr ---------------------------------------
courses_df %>%
  filter(platform %in% c("B", "C"))

# Filter by platform B and C using data.table ----------------------------------
# The data.table helper operator %chin% is equivalent to but faster than %in%
courses_df[platform %chin% c("B", "C")]
```

    ##     student platform           course platform_start_date platform_end_date
    ## 1: 0001a247        C   R_intermediate          2017-12-11        2018-01-11
    ## 2: 00023d66        C machine_learning          2018-11-04        2018-11-20
    ## 3: 000644e7        B   travel_writing          2016-04-08        2016-04-22

## Filter by multiple conditions

When filtering by multiple conditions, we use the symbol `|` to denote
the expression `OR` and the symbol `&` to denote the expression `AND`
(or to be more precise, to fine an union or an intersection between two
conditions). In `dplyr`, the comma is a shorthand for `AND`.

``` r
# Filter by platform B & platform start date >= 2018-09-01 using dplyr ---------
courses_df %>%
  filter(platform == "B",
         platform_start_date >= "2018-09-01")

# Filter by platform B & platform start date >= 2018-09-01 using data.table ----
courses_df[platform == "B" & platform_start_date >= "2018-09-01"]
```

    ##     student platform              course platform_start_date platform_end_date
    ## 1: 01243dd1        B          R_advanced          2018-09-02        2018-09-11
    ## 2: 016df220        B Python_intermediate          2018-11-03        2018-11-14
    ## 3: 017ed65a        B         data_mining          2018-12-03        2019-01-17

## Filter using regular expressions

We can also filter values using regular expressions via the `stringr`
package or the `data.table` helper function `%like%`.

``` r
# Filter by course starting with the letter R using dplyr ----------------------
courses_df %>%
  filter(str_detect(course, "^R"))

# Filter by course starting with the letter R using data.table -----------------
courses_df[course %like% "R"]
```

    ##     student platform         course platform_start_date platform_end_date
    ## 1: 00019f4c        A     R_advanced          2016-01-09        2016-01-18
    ## 2: 00019f4c        A R_intermediate          2016-01-09        2016-01-18
    ## 3: 00019f4c        A     R_advanced          2016-01-09        2016-01-18

## Filter across multiple columns

With the release of [dplyr
1.0.4](https://www.tidyverse.org/blog/2021/02/dplyr-1-0-4-if-any/), we
can also filter by a condition across multiple columns using `if_all()`
and `if_any()`. An equivalent shortcut does not currently exist for
`data.table` or `Pandas`, so columns for filtering still need to be
listed manually.

``` r
# Filter by platform start AND end date between dates using dplyr --------------
courses_df %>%
  filter(if_all(ends_with("_date"), ~between(., "2018-01-01", "2018-03-31"))) %>%
  head(3) # View output
```

    ##     student platform         course platform_start_date platform_end_date
    ## 1: 00074144        A linear_algebra          2018-01-01        2018-03-01
    ## 2: 00074144        A linear_algebra          2018-01-01        2018-03-01
    ## 3: 0008c592        E linear_algebra          2018-02-20        2018-03-01

``` r
# Filter by platform start OR end date between dates using dplyr ---------------
courses_df %>%
  filter(if_any(ends_with("_date"), ~between(., "2018-01-01", "2018-03-31"))) %>%
  head(3) # View output
```

    ##     student platform          course platform_start_date platform_end_date
    ## 1: 000027f0        E         pottery          2018-03-23        2018-05-08
    ## 2: 0001a247        C  R_intermediate          2017-12-11        2018-01-11
    ## 3: 0001bda1        A Python_beginner          2017-11-25        2018-01-01

## Equivalent Pandas code

``` python
# Filter by platform A and B using Pandas in Python ----------------------------
courses_pf[courses_pf.platform.isin(["B", "C"])]  
```

    ##          student platform  ... platform_start_date platform_end_date
    ## 18      0001a247        C  ...          2017-12-11        2018-01-11
    ## 25      00023d66        C  ...          2018-11-04        2018-11-20
    ## 53      000644e7        B  ...          2016-04-08        2016-04-22
    ## 74      00073521        C  ...          2017-08-18        2017-09-28
    ## 99      0007e24b        C  ...          2017-11-22        2017-12-14
    ## ...          ...      ...  ...                 ...               ...
    ## 999817  fff2978e        C  ...          2017-10-14        2017-11-15
    ## 999872  fff63de8        C  ...          2017-08-24        2017-09-24
    ## 999920  fff8ffae        C  ...          2018-03-21        2018-05-18
    ## 999942  fffaf28d        C  ...          2017-10-15        2017-10-27
    ## 999966  fffc8648        C  ...          2016-08-26        2016-10-01
    ## 
    ## [39968 rows x 5 columns]

``` python
# Filter by platform B & platform start date >= 2018-09-01 using Pandas --------
is_platform_B = (courses_pf["platform"] == "B")  
starts_on_or_after_2018_09_01 = (courses_pf["platform_start_date"] >= "2018-09-01")  

courses_pf[is_platform_B & starts_on_or_after_2018_09_01]  
```

    ##          student platform  ... platform_start_date platform_end_date
    ## 4396    01243dd1        B  ...          2018-09-02        2018-09-11
    ## 5442    016df220        B  ...          2018-11-03        2018-11-14
    ## 5717    017ed65a        B  ...          2018-12-03        2019-01-17
    ## 6074    01967a76        B  ...          2018-12-21        2019-01-10
    ## 7789    02054f08        B  ...          2018-12-19        2019-01-07
    ## ...          ...      ...  ...                 ...               ...
    ## 995421  fedbbcc7        B  ...          2018-12-03        2018-12-31
    ## 996874  ff39b58b        B  ...          2018-11-17        2018-12-18
    ## 997195  ff509b2a        B  ...          2018-10-16        2018-10-29
    ## 998737  ffae892d        B  ...          2018-12-30        2019-01-20
    ## 998763  ffb0a677        B  ...          2018-11-09        2018-12-03
    ## 
    ## [1101 rows x 5 columns]

In `Pandas`, we can filter values using regular expressions using the
[string
methods](https://pandas.pydata.org/pandas-docs/stable/user_guide/text.html#testing-for-strings-that-match-or-contain-a-pattern)
`str.contains`, `str.match` or `str.fullmatch`. In this scenario, the
most efficient option is `str.match` to match the first characters of
the string.

``` python
# Filter by course starting with the letter R using Pandas ---------------------
# Alternatively set regex = True when using df.column.str.contains("pattern") 
courses_pf[courses_pf.course.str.match("R_", na = False)]
```

    ##          student platform          course platform_start_date platform_end_date
    ## 12      00019f4c        A      R_advanced          2016-01-09        2016-01-18
    ## 14      00019f4c        A  R_intermediate          2016-01-09        2016-01-18
    ## 15      00019f4c        A      R_advanced          2016-01-09        2016-01-18
    ## 18      0001a247        C  R_intermediate          2017-12-11        2018-01-11
    ## 21      00021de1        A      R_beginner          2017-09-13        2017-09-25
    ## ...          ...      ...             ...                 ...               ...
    ## 999963  fffc7136        A  R_intermediate          2016-03-23        2016-05-01
    ## 999976  fffce55e        E      R_advanced          2018-04-01        2018-04-29
    ## 999979  fffd1b29        A      R_advanced          2018-05-11        2018-06-10
    ## 999990  fffe1f47        A      R_advanced          2018-05-31        2018-06-09
    ## 999995  ffff846d        A      R_beginner          2017-04-27        2017-06-13
    ## 
    ## [176587 rows x 5 columns]

## Code benchmarking

The execution time for `dplyr` and `data.table` is roughly equivalent
for filtering data.

<img src="dc-data_table_vs_dplyr_files/figure-gfm/unnamed-chunk-18-1.png" width="60%" />

# Sort data

Sorting large data frames is computationally expensive, so we should
always minimise the number of times we need to sort a column. In
`dplyr`, `data.table` and `Pandas`, records are consistently ordered in
ascending order by default (likely a design choice from SQL).

In `dplyr`, we sort columns using `arrange()`. To sort columns by
descending order, we append `desc()` inside \`arrange().

The `data.table` package uses an alternative faster sort method
accessible via the `order()` function. To sort columns by descending
order, we append `-` in front of the column name. We can also sort **by
reference** using `setorder()`, which directly modifies the original
data frame.

``` r
# Sort by ascending student id and descending platform name using dplyr --------
courses_df %>%
  arrange(student,
          desc(platform))

# Sort by ascending student id and descending platform name using data.table ---
courses_df[order(student,
                 -platform)]

# setorder() sorts by reference  
# setorder(courses_df, 
#          student,
#          -platform)
```

    ##     student platform         course platform_start_date platform_end_date
    ## 1: 000027f0        E        pottery          2018-03-23        2018-05-08
    ## 2: 000027f0        A     statistics          2016-08-11        2016-09-19
    ## 3: 000027f0        A linear_algebra          2017-02-14        2017-03-13
    ## 4: 000094e9        A travel_writing          2018-11-15        2018-12-20

## Equivalent Pandas code

In `Pandas`, we use the `.sort_values()` method and can assign missing
values to appear in the first or last position via the argument
`na_position = "first"`.

``` python
# Sort by ascending student id and descending platform name using Pandas -------
courses_pf.sort_values(by = ["student", "platform"],
                       ascending = [True, False], 
                       na_position = "first")
```

    ##          student platform  ... platform_start_date platform_end_date
    ## 2       000027f0        E  ...          2018-03-23        2018-05-08
    ## 0       000027f0        A  ...          2016-08-11        2016-09-19
    ## 1       000027f0        A  ...          2017-02-14        2017-03-13
    ## 3       000094e9        A  ...          2018-11-15        2018-12-20
    ## 4       000094e9        A  ...          2018-11-15        2018-12-20
    ## ...          ...      ...  ...                 ...               ...
    ## 999995  ffff846d        A  ...          2017-04-27        2017-06-13
    ## 999996  ffff846d        A  ...          2017-04-27        2017-06-13
    ## 999997  ffff846d        A  ...          2017-04-27        2017-06-13
    ## 999998  ffff846d        A  ...          2017-04-27        2017-06-13
    ## 999999  ffff846d        A  ...          2017-04-27        2017-06-13
    ## 
    ## [1000000 rows x 5 columns]

## Code benchmarking

The `data.table` package uses a much faster sort method compared to
`dplyr`.

<img src="dc-data_table_vs_dplyr_files/figure-gfm/unnamed-chunk-22-1.png" width="60%" />

# Select columns

Using `dplyr`, single or multiple columns can be selected via `select()`
and combined with regular expressions or column type queries wrapped
inside [`where()`](https://tidyselect.r-lib.org/reference/where.html).
As of `data.table` [version
1.13.0](https://github.com/Rdatatable/data.table/blob/master/NEWS.md#datatable-v1130--24-jul-2020),
complex column selections can be performed by inputting a function using
`.SDcols`.

``` r
# Select multiple columns by name using dplyr ----------------------------------
courses_df %>%
  select(student,
         platform)

# Select multiple columns by name using data.table -----------------------------
# .() is a shorthand for list()  
courses_df[,
           .(student,
             platform)]
```

    ##     student platform
    ## 1: 000027f0        A
    ## 2: 000027f0        A
    ## 3: 000027f0        E

``` r
# Select columns ending with date using dplyr ----------------------------------
courses_df %>%
  select(ends_with("date"))

# Select columns ending with date using data.table -----------------------------
courses_df[, 
           grep("date$", colnames(courses_df), value = TRUE),
           with = FALSE]

# Using .SDcols syntax for functions to output columns via .SD
courses_df[, 
           .SD,
           .SDcols = grep("date$", colnames(courses_df), value = TRUE)]
```

    ##    platform_start_date platform_end_date
    ## 1:          2016-08-11        2016-09-19
    ## 2:          2017-02-14        2017-03-13
    ## 3:          2018-03-23        2018-05-08

``` r
# Select columns of date type using dplyr --------------------------------------
# Apply class(col) across all columns in courses_df to identify column type 
vapply(courses_df, class, character(1L))
#>   student       platform      course        platform_start_date    platform_end_date 
#>   "character"   "character"   "character"   "Date"                 "Date"

# Selects columns where the column type inherits the Date class  
courses_df %>%
  select(where(~inherits(., "Date")))

# Select columns of date type using data.table ---------------------------------
# ~inherits(., "Date") is a dplyr shorthand for function(x) inherits(x, 'Date')
courses_df[, 
           .SD,
           .SDcols = function(x) inherits(x, 'Date')]
```

    ##    platform_start_date platform_end_date
    ## 1:          2016-08-11        2016-09-19
    ## 2:          2017-02-14        2017-03-13
    ## 3:          2018-03-23        2018-05-08

For `dplyr`, extracting a single column using `select()` will always
return another data frame, unless we explicitly use `pull()` to output a
vector. For `data.table`, wrapping columns inside a list via `.()`
returns another data frame.

``` r
# Output data frame with select() using dplyr ----------------------------------
courses_df %>%
  select(student) %>%
  class()
#> [1] "data.table" "data.frame"  

# Output vector with select() & pull using dplyr -------------------------------
courses_df %>%
  pull(student) %>%
  class()
#> [1] "character"

# Output data frame following column selection using data.table ----------------
courses_df[, .(student)] %>%
  class()
#> [1] "data.table" "data.frame"  

# Output vector following column selection using data.table --------------------
courses_df[, student] %>%
  class() 
#> [1] "character"
```

## Equivalent Pandas code

In `Pandas`, the `.filter()` method can be used to select columns using
regular expressions and the `.select_dtypes()` method is used to select
columns by column type. `Pandas` data frames can be outputted as a data
frame, series or NumPy array.

``` python
# Select multiple columns by name using Pandas ---------------------------------
courses_pf[["student", "platform"]]
```

    ##          student platform
    ## 0       000027f0        A
    ## 1       000027f0        A
    ## 2       000027f0        E
    ## 3       000094e9        A
    ## 4       000094e9        A
    ## ...          ...      ...
    ## 999995  ffff846d        A
    ## 999996  ffff846d        A
    ## 999997  ffff846d        A
    ## 999998  ffff846d        A
    ## 999999  ffff846d        A
    ## 
    ## [1000000 rows x 2 columns]

``` python
# Select columns ending with date using Pandas ---------------------------------
courses_pf.filter(regex = ("date$"))  
```

    ##        platform_start_date platform_end_date
    ## 0               2016-08-11        2016-09-19
    ## 1               2017-02-14        2017-03-13
    ## 2               2018-03-23        2018-05-08
    ## 3               2018-11-15        2018-12-20
    ## 4               2018-11-15        2018-12-20
    ## ...                    ...               ...
    ## 999995          2017-04-27        2017-06-13
    ## 999996          2017-04-27        2017-06-13
    ## 999997          2017-04-27        2017-06-13
    ## 999998          2017-04-27        2017-06-13
    ## 999999          2017-04-27        2017-06-13
    ## 
    ## [1000000 rows x 2 columns]

``` python
# Select columns of date type using Pandas -------------------------------------
# Access the data frame .dtypes attribute to first identify all column types  
# courses_pf.dtypes
courses_pf.select_dtypes(include=["datetime64"])
```

    ##        platform_start_date platform_end_date
    ## 0               2016-08-11        2016-09-19
    ## 1               2017-02-14        2017-03-13
    ## 2               2018-03-23        2018-05-08
    ## 3               2018-11-15        2018-12-20
    ## 4               2018-11-15        2018-12-20
    ## ...                    ...               ...
    ## 999995          2017-04-27        2017-06-13
    ## 999996          2017-04-27        2017-06-13
    ## 999997          2017-04-27        2017-06-13
    ## 999998          2017-04-27        2017-06-13
    ## 999999          2017-04-27        2017-06-13
    ## 
    ## [1000000 rows x 2 columns]

``` python
# Output data frame following column selection using Pandas --------------------
type(courses_pf[["student"]])
#> <class 'pandas.core.frame.DataFrame'>  

# Output series or vector array following column selection using Pandas --------
type(courses_pf["student"])
#> <class 'pandas.core.series.Series'>  

type(courses_pf["student"].values)
#> <class 'numpy.ndarray'>
```

## Code benchmarking

The execution time for `dplyr` and `data.table` is roughly equivalent
for selecting columns, with `dplyr` exhibiting slightly faster code
execution times.

<img src="dc-data_table_vs_dplyr_files/figure-gfm/unnamed-chunk-34-1.png" width="60%" />

# Transform columns

## Transform via copy on modify versus modify in place

Using `dplyr`, we can create new columns or overwrite existing columns
in our original data frame using `mutate()`, or create or overwrite new
columns and drop all other existing columns using `transmute()`.

``` r
# Create column for platform dwell length using dplyr --------------------------
courses_df %>% 
  mutate(platform_dwell_length = platform_end_date - platform_start_date) %>%
  head(3)  
```

    ##     student platform         course platform_start_date platform_end_date
    ## 1: 000027f0        A     statistics          2016-08-11        2016-09-19
    ## 2: 000027f0        A linear_algebra          2017-02-14        2017-03-13
    ## 3: 000027f0        E        pottery          2018-03-23        2018-05-08
    ##    platform_dwell_length
    ## 1:               39 days
    ## 2:               27 days
    ## 3:               46 days

``` r
courses_df %>% 
  transmute(platform_dwell_length = platform_end_date - platform_start_date) %>%
  head(3)  
```

    ##    platform_dwell_length
    ## 1:               39 days
    ## 2:               27 days
    ## 3:               46 days

In R, a second copy of an object is created by default whenever the
original object is modified and this behaviour is called [copy on
modify](https://adv-r.hadley.nz/names-values.html#copy-on-modify). This
implies means that a shallow data.frame copy is created whenever a
column is modified using `mutate()` with `dplyr`. We can trace this
behaviour using `lobstr::ref()` according to [chapter
2.3.4](https://adv-r.hadley.nz/names-values.html#df-modify) of the
[Advanced R textbook](https://adv-r.hadley.nz/index.html).

``` r
# Check copy on modify behaviour following mutate() ----------------------------
lobstr::ref(courses_df)
#> o [1:0x1a043067b00] <dt[,5]> 
#> +-student = [2:0x7ff450310010] <chr> 
#> +-platform = [3:0x7ff44fb60010] <chr> 
#> +-course = [4:0x7ff44f3b0010] <chr> 
#> +-platform_start_date = [5:0x7ff44ec00010] <date> 
#> \-platform_end_date = [6:0x7ff44e450010] <date> 

# Modify the platform column and check data frame and column references --------
courses_df <- courses_df %>%
  mutate(platform = tolower(platform))  

lobstr::ref(courses_df)
#> o [1:0x1a043924e28] <dt[,5]> 
#> +-student = [2:0x7ff450310010] <chr> 
#> +-platform = [3:0x7ff45c2d0010] <chr> 
#> +-course = [4:0x7ff44f3b0010] <chr> 
#> +-platform_start_date = [5:0x7ff44ec00010] <date> 
#> \-platform_end_date = [6:0x7ff44e450010] <date>    

# A new copy of the courses_df data frame as well as the platform column is 
# created and referenced.  
```

    ##     student platform         course platform_start_date platform_end_date
    ## 1: 000027f0        a     statistics          2016-08-11        2016-09-19
    ## 2: 000027f0        a linear_algebra          2017-02-14        2017-03-13
    ## 3: 000027f0        e        pottery          2018-03-23        2018-05-08

Using `data.table`, we can directly modify or even remove columns in
place using the `:=` operator inside the `j` placeholder of
`DT[i, j, by]`. This is a special property of `data.table` and not
`dplyr` or `Pandas`.

``` r
# Check modify in place behaviour following := assignment ----------------------
lobstr::ref(courses_df)  
#> o [1:0x1a043924e28] <dt[,5]> 
#> +-student = [2:0x7ff450310010] <chr> 
#> +-platform = [3:0x7ff45c2d0010] <chr> 
#> +-course = [4:0x7ff44f3b0010] <chr> 
#> +-platform_start_date = [5:0x7ff44ec00010] <date> 
#> \-platform_end_date = [6:0x7ff44e450010] <date>  

# Modify the platform column and check data frame and column references --------
courses_df[,
           platform := toupper(platform)]

lobstr::ref(courses_df)
#> o [1:0x1a043924e28] <dt[,5]> 
#> +-student = [2:0x7ff450310010] <chr> 
#> +-platform = [3:0x7ff459750010] <chr> 
#> +-course = [4:0x7ff44f3b0010] <chr> 
#> +-platform_start_date = [5:0x7ff44ec00010] <date> 
#> \-platform_end_date = [6:0x7ff44e450010] <date> 

# By modifying columns in place, we do not create a new copy of the courses_df 
# data frame every time we modify a column. 

# Modify multiple columns using data.table -------------------------------------
# To modify multiple columns in place, we move `:=` to the front expression    
courses_df[,
           `:=` (platform_dwell_length = platform_end_date - platform_start_date,
                 platform_start_year = str_extract(platform_start_date, "^.{4}(?!//-)"))]
```

    ##     student platform         course platform_start_date platform_end_date
    ## 1: 000027f0        A     statistics          2016-08-11        2016-09-19
    ## 2: 000027f0        A linear_algebra          2017-02-14        2017-03-13
    ## 3: 000027f0        E        pottery          2018-03-23        2018-05-08
    ##    platform_dwell_length platform_start_year
    ## 1:               39 days                2016
    ## 2:               27 days                2017
    ## 3:               46 days                2018

## Transform using multiple conditions

We can create a column based on multiple conditions using `case_when()`
or `f_case()` for `dplyr` and `data.table` respectively.

``` r
# Create column based on multiple conditions using dplyr -----------------------
courses_df %>%
  mutate(studied_programming = case_when(
    str_detect(course, "^R_") ~ "Studied R",
    str_detect(course, "^Python_") ~ "Studied Python",
    TRUE ~ "No"
  ))

# Create column based on multiple conditions using data.table ------------------
courses_df[,
           studied_programming := fcase(
             str_detect(course, "^R_"), "Studied R",
             str_detect(course, "^Python_"), "Studied Python",
             default = "No")]  
```

    ##     student platform         course platform_start_date platform_end_date
    ## 1: 000027f0        A     statistics          2016-08-11        2016-09-19
    ## 2: 000027f0        A linear_algebra          2017-02-14        2017-03-13
    ## 3: 000027f0        E        pottery          2018-03-23        2018-05-08
    ##    platform_dwell_length platform_start_year studied_programming
    ## 1:               39 days                2016                  No
    ## 2:               27 days                2017                  No
    ## 3:               46 days                2018                  No

``` r
# Remove columns in place using data.table -------------------------------------
# Columns can be removed in place by using := NULL column assignment  
courses_df[,
           c("platform_start_year",
             "studied_programming") := NULL]
```

    ##     student platform         course platform_start_date platform_end_date
    ## 1: 000027f0        A     statistics          2016-08-11        2016-09-19
    ## 2: 000027f0        A linear_algebra          2017-02-14        2017-03-13
    ## 3: 000027f0        E        pottery          2018-03-23        2018-05-08
    ##    platform_dwell_length
    ## 1:               39 days
    ## 2:               27 days
    ## 3:               46 days

Since `dplyr` [version
1.0.0](https://www.tidyverse.org/blog/2020/06/dplyr-1-0-0/), we can use
`mutate()` in combination with `across()` to apply the same
transformation functions across multiple columns. In `data.table`, the
equivalent solution is to specify columns of interest via `.SDcols` and
then loop through each column using `lapply(function)`.

``` r
# Apply the same function across multiple columns using dplyr ------------------
courses_df %>%
  mutate(across(c(student, platform, course), tolower))
```

    ##     student platform         course platform_start_date platform_end_date
    ## 1: 000027f0        a     statistics          2016-08-11        2016-09-19
    ## 2: 000027f0        a linear_algebra          2017-02-14        2017-03-13
    ## 3: 000027f0        e        pottery          2018-03-23        2018-05-08
    ##    platform_dwell_length
    ## 1:               39 days
    ## 2:               27 days
    ## 3:               46 days

``` r
# where() can be used inside across() for conditional column selections  
courses_df %>%
  mutate(across(where(is.character), toupper))
```

    ##     student platform         course platform_start_date platform_end_date
    ## 1: 000027F0        A     STATISTICS          2016-08-11        2016-09-19
    ## 2: 000027F0        A LINEAR_ALGEBRA          2017-02-14        2017-03-13
    ## 3: 000027F0        E        POTTERY          2018-03-23        2018-05-08
    ##    platform_dwell_length
    ## 1:               39 days
    ## 2:               27 days
    ## 3:               46 days

``` r
# Apply the same function across multiple columns using data.table -------------
# data.table modifies all selected columns in place via :=
cols <- c("student", "platform", "course")
courses_df[,
           (cols) := lapply(.SD, tolower),
           .SDcols = cols]  
```

    ##     student platform         course platform_start_date platform_end_date
    ## 1: 000027f0        a     statistics          2016-08-11        2016-09-19
    ## 2: 000027f0        a linear_algebra          2017-02-14        2017-03-13
    ## 3: 000027f0        e        pottery          2018-03-23        2018-05-08
    ##    platform_dwell_length
    ## 1:               39 days
    ## 2:               27 days
    ## 3:               46 days

## Equivalent Pandas code

In `Pandas`, we can transform columns directly by assignment or by using
the `.apply()` method. The benefit of using `.apply()` is that it also
handles bespoke functions. A [Stack Overflow
post](https://stackoverflow.com/questions/19798153/difference-between-map-applymap-and-apply-methods-in-pandas)
discussing the different uses of `.apply()` and `.applymap()`.

``` python
# Create column for platform dwell length using Pandas -------------------------
courses_pf["platform_dwell_length"] = courses_pf["platform_end_date"] - courses_pf["platform_start_date"]
```

    ##          student platform  ... platform_end_date platform_dwell_length
    ## 0       000027f0        A  ...        2016-09-19               39 days
    ## 1       000027f0        A  ...        2017-03-13               27 days
    ## 2       000027f0        E  ...        2018-05-08               46 days
    ## 3       000094e9        A  ...        2018-12-20               35 days
    ## 4       000094e9        A  ...        2018-12-20               35 days
    ## ...          ...      ...  ...               ...                   ...
    ## 999995  ffff846d        A  ...        2017-06-13               47 days
    ## 999996  ffff846d        A  ...        2017-06-13               47 days
    ## 999997  ffff846d        A  ...        2017-06-13               47 days
    ## 999998  ffff846d        A  ...        2017-06-13               47 days
    ## 999999  ffff846d        A  ...        2017-06-13               47 days
    ## 
    ## [1000000 rows x 6 columns]

``` python
# Create column based on multiple conditions using Pandas ----------------------
# Use np.select() rather than .apply() to apply a vectorised transformation
courses_pf["studied_programming"] = np.select(
  [courses_pf["course"].str.match("R_", na = False), 
   courses_pf["course"].str.match("Python_", na = False)],
  ["Studied R",
   "Studied Python"],
  default = "No"
)
```

    ##          student platform  ... platform_dwell_length studied_programming
    ## 0       000027f0        A  ...               39 days                  No
    ## 1       000027f0        A  ...               27 days                  No
    ## 2       000027f0        E  ...               46 days                  No
    ## 3       000094e9        A  ...               35 days                  No
    ## 4       000094e9        A  ...               35 days                  No
    ## ...          ...      ...  ...                   ...                 ...
    ## 999995  ffff846d        A  ...               47 days           Studied R
    ## 999996  ffff846d        A  ...               47 days                  No
    ## 999997  ffff846d        A  ...               47 days      Studied Python
    ## 999998  ffff846d        A  ...               47 days                  No
    ## 999999  ffff846d        A  ...               47 days                  No
    ## 
    ## [1000000 rows x 7 columns]

``` python
# Apply the same function across multiple columns using Pandas -----------------
# Applying a vector operation looped over a list of columns is faster than 
# transforming columns element-wise using .apply().  

# Avoid .apply() unless you need to apply a function element-wise across a column
# For example, the comments code below runs significantly slower    
# courses_pf[object_cols] = courses_pf[object_cols].apply(lambda col: col.str.upper()) 

object_cols = [col for col, dt in courses_pf.dtypes.items() if dt == "object"]

for col in object_cols: 
  courses_pf[col] = courses_pf[col].str.lower()
```

    ##          student platform  ... platform_dwell_length studied_programming
    ## 0       000027f0        a  ...               39 days                  no
    ## 1       000027f0        a  ...               27 days                  no
    ## 2       000027f0        e  ...               46 days                  no
    ## 3       000094e9        a  ...               35 days                  no
    ## 4       000094e9        a  ...               35 days                  no
    ## ...          ...      ...  ...                   ...                 ...
    ## 999995  ffff846d        a  ...               47 days           studied r
    ## 999996  ffff846d        a  ...               47 days                  no
    ## 999997  ffff846d        a  ...               47 days      studied python
    ## 999998  ffff846d        a  ...               47 days                  no
    ## 999999  ffff846d        a  ...               47 days                  no
    ## 
    ## [1000000 rows x 7 columns]

``` python
# Remove columns using Pandas --------------------------------------------------
courses_pf = courses_pf.drop(columns=["studied_programming"])
```

## Code benchmarking

The execution time for `dplyr` and `data.table` is roughly equivalent
for transforming columns, with `data.table` exhibiting slightly faster
code execution times.

<img src="dc-data_table_vs_dplyr_files/figure-gfm/unnamed-chunk-57-1.png" width="60%" />

# Group by and summarise columns

Summarising data across \>100K subgroups is where `data.table` greatly
outperforms `dplyr` in terms of code execution speed. In `dplyr`, a
grouping convention is specified using the `group_by()` function and
summary outputs then created using the `summarise()` function.

``` r
# Summarise by course and platform using dplyr ---------------------------------
courses_df %>%
  group_by(platform, course) %>%
  summarise(mean_length = mean(platform_dwell_length),
            total_courses = n()) 

# dplyr cannot group by student as it has >200K subgroups whereas data.table can 

# Summarise by course and platform using data.table ----------------------------
courses_df[,
           .(mean_length = mean(platform_dwell_length),
             total_courses = .N),
           by = .(platform, course)] 
```

    ##    platform         course   mean_length total_courses
    ## 1:        a     statistics 34.02394 days         41067
    ## 2:        a linear_algebra 33.94655 days         40916
    ## 3:        e        pottery 33.94459 days         12056

Since `dplyr` [version
1.0.0](https://www.tidyverse.org/blog/2020/06/dplyr-1-0-0/), we can use
`summarise()` in combination with `across()` to apply the same
aggregation across multiple columns. A comprehensive \[blog post\] on
`across()` examples can be found
[here](https://willhipson.netlify.app/post/dplyr_across/dplyr_across/).

## Equivalent Pandas code

In `Pandas`, we can summarise data across subgroups using the method
`.groupby()` followed by the method `.agg()`.

``` python
# Summarise by course and platform using Pandas --------------------------------
courses_pf.groupby(["platform", "course"]).agg(mean_length = ("platform_dwell_length", "mean"),
                                               total_course = ("platform_dwell_length", "count"))
```

    ##                                            mean_length  total_course
    ## platform course                                                     
    ## a        bread_baking       34 days 01:34:25.299806576         41360
    ##          carpentry          33 days 22:34:50.910862153         41060
    ##          contemporary_dance 34 days 00:11:18.495177796         40749
    ##          data_mining        33 days 23:38:12.340208439         41163
    ##          linear_algebra     33 days 22:43:01.835956594         40916
    ## ...                                                ...           ...
    ## e        r_intermediate     33 days 20:18:35.596330275         11772
    ##          statistics         34 days 00:59:00.983606557         11956
    ##          travel_writing     33 days 22:36:42.272135981         11531
    ##          ux_design          33 days 22:01:42.149350097         11771
    ##          website_design     34 days 06:07:13.660828705         11705
    ## 
    ## [85 rows x 2 columns]

## Code benchmarking

The execution time for `dplyr` and `data.table` is roughly equivalent
for transforming columns, with `data.table` exhibiting slightly faster
code execution times.

<img src="dc-data_table_vs_dplyr_files/figure-gfm/unnamed-chunk-61-1.png" width="60%" />

# Chain code

In data analysis, we frequently need to perform a sequence of operations
to generate a new named data output. In R, we can chain `dplyr` and
`data.table` operations using `%>%` or `|>` pipes or `data.table` `[]`
chains. I personally prefer using `%>%` for chaining R operations when I
don’t care about minimising package dependencies.

``` r
# Chain dplyr operations using %>% and |> pipes --------------------------------
# Output a data frame containing the earliest platform start date for all 
# courses per platform  
courses_df %>%
  select(-c(student, platform_dwell_length, platform_end_date)) %>%
  arrange(platform, course, platform_start_date) %>%
  group_by(platform, course) %>%
  filter(row_number() == 1L) 

# You can also use the base R pipe |> for R version >= 4.1.0
courses_df |>
  select(-c(student, platform_dwell_length, platform_end_date)) |>
  arrange(platform, course, platform_start_date) |>
  group_by(platform, course) |>
  filter(row_number() == 1L) 

# Chain data.table operations using %>% ----------------------------------------
courses_df[,
           -c("student",
              "platform_dwell_length",
              "platform_end_date",
              "studied_programming")] %>% 
  .[order(platform, course, platform_start_date)] %>%
  .[,
    .SD[1L],
    by = .(platform, course)]

# A base R pipe solution currently does not exist for data.table  

# Chain data.table operations using [] -----------------------------------------
courses_df[,
           -c("student",
              "platform_dwell_length",
              "platform_end_date",
              "studied_programming")
][
  order(platform,
        course,
        platform_start_date)
][
  ,
  .SD[1L],
  by = .(platform, course)]
```

## Equivalent Pandas code

In Python, we can also chain Pandas methods inside `()`. Note that this
practice currently only works on Pandas methods and is therefore a lot
less flexible than using pipes in R.

``` python
# Chain Pandas operations using () ---------------------------------------------
# Multiple Pandas methods can be chained inside a single bracket 
(
  courses_pf
  .sort_values(by = ["platform", "course", "platform_start_date"])
  .groupby(["platform", "course"])
  .first()
  .drop(columns=["student", "platform_dwell_length", "platform_end_date"])
)
```

# Other resources

- The official `data.table`
  [vignnette](https://cran.r-project.org/web/packages/data.table/vignettes/datatable-intro.html).  
- A comprehensive [stack overflow
  discussion](https://stackoverflow.com/questions/21435339/data-table-vs-dplyr-can-one-do-something-well-the-other-cant-or-does-poorly/27840349#27840349)
  about the advantages of using `data.table` versus `dplyr`.  
- A [blog
  post](https://atrebas.github.io/post/2019-03-03-datatable-dplyr/) with
  side by side comparison of data.table versus dplyr operations by
  Atrebas.  
- My [blog
  post](https://erikaduan.github.io/posts/2021-02-16-data-table-part-2/)
  on how to chain multiple `data.table` operations for complex data
  analysis.  
- An
  [explanation](https://tysonbarrett.com//jekyll/update/2019/07/12/datatable/)
  of how `data.table` modifies by reference by Tyson Barrett.  
- A [detailed
  explanation](https://gist.github.com/arunsrinivasan/dacb9d1cac301de8d9ff)
  of binary search based subsets in `data.table` by Arun Srinivasan.  
- The Advanced R [chapter](https://adv-r.hadley.nz/perf-measure.html) on
  measuring code performance by Hadley Wickham.
