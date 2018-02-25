MLB Statcast data with R
================
Brandon Hoeft
August 19, 2017

-   [baseballr Package](#baseballr-package)
-   [What baseballr Contains](#what-baseballr-contains)
-   [Querying all Batter Events](#querying-all-batter-events)
-   [Save results to disk](#save-results-to-disk)
-   [Load Feather file from disk](#load-feather-file-from-disk)

baseballr Package
-----------------

This is an exploration of the `baseballr` package created by Bill Petti, which contains web scraping functions and access to a variety of baseball statistics and Statcast MLB data from BaseballSevant, which are property of MLB Advanced Media, L.P. All rights reserved.

More information about the package can be found \[here\] (<https://github.com/BillPetti/baseballr>). The package and its dependencies must be installed from Github using the `devtools` package.

What baseballr Contains
-----------------------

Let's take an initial look at the different objects that exist in the `baseballr` package environment. Many of them are functions for scraping data of interest.

``` r
setwd("~/Documents/R Files/MLB Statcast")
library(baseballr)
library(dplyr)
ls("package:baseballr")
```

    ##  [1] "%<>%"                              
    ##  [2] "%>%"                               
    ##  [3] "batter_boxscore"                   
    ##  [4] "code_barrel"                       
    ##  [5] "daily_batter_bref"                 
    ##  [6] "daily_pitcher_bref"                
    ##  [7] "edge_code"                         
    ##  [8] "edge_frequency"                    
    ##  [9] "edge_scrape"                       
    ## [10] "edge_scrape_split"                 
    ## [11] "fg_bat_leaders"                    
    ## [12] "fg_guts"                           
    ## [13] "fg_park"                           
    ## [14] "fg_park_hand"                      
    ## [15] "fip_plus"                          
    ## [16] "master_ncaa_team_lu"               
    ## [17] "ncaa_scrape"                       
    ## [18] "ncaa_season_id_lu"                 
    ## [19] "pitcher_boxscore"                  
    ## [20] "playerid_lookup"                   
    ## [21] "school_id_lu"                      
    ## [22] "scrape_statcast_savant_batter"     
    ## [23] "scrape_statcast_savant_batter_all" 
    ## [24] "scrape_statcast_savant_pitcher"    
    ## [25] "scrape_statcast_savant_pitcher_all"
    ## [26] "standings_on_date_bref"            
    ## [27] "team_consistency"                  
    ## [28] "team_results_bref"                 
    ## [29] "woba_plus"

Querying all Batter Events
--------------------------

The **scrape\_statcast\_savant\_batter\_all()** function in `baseballr` is a web scraping function from &lt;www.baseballsavant.mlb.com&gt; relating to Statcast play-by-play batter data. From the function's description, it does the following: *This function allows you to query Statcast and PITCHf/x data as provided on baseballsavant.mlb.com and have that data returned as a dataframe. Query returns data for all batters over a given time frame.* Some details from the author of the function are available [here](https://github.com/BillPetti/baseballr/blob/master/baseballr_Updates/baseballr_updates_05_08_2017.md).

The query maxes out on my Macbook Air after 30,000 at-bat records are retrieved, so a single query will not retrieve anywhere close to every at-bat event in a full season's worth of games. A single game day can have thousands of at bat records. To get around this, I will iterate the **scrape\_statcast\_savant\_batter\_all()** function over each game day and store the results in a list.

First I created a sequence of the dates to query, dropping any dates where no MLB game was played

``` r
library(lubridate)
# games where no teams played from mlb.com.
no_game_day <- seq(ymd('2017-07-10'),ymd('2017-07-13'), by = 'days')
# identifying game days as TRUE/FALSE
game_day_flag <- !seq(ymd('2017-04-02'),ymd('2017-08-19'), by = 'days') %in% no_game_day
# Date objects will coerce to numeric vectors in the seq argument of a for loop. coerce to character.
game_days <- as.character(seq(ymd('2017-04-02'),ymd('2017-08-19'), by = 'days')[game_day_flag])

glimpse(game_days)
```

    ##  chr [1:136] "2017-04-02" "2017-04-03" "2017-04-04" ...

One problem with each scraped query result's data frame is that about half of the 79 variables in the query results are always typed as `factors`, whose underlying numeric levels can differ depending on the unique observations in that day's baseball results. This will lead to problems when unioning each query's data frame result. The same variable between different query results will have inconsistent `factor` levels due to the variation of unique events recorded that game day on that measure.

To deal with this, a custom function called **coerce\_to\_character** will check if the column is a `factor` and coerce it to a character string instead. This data cleaning will prevent issues when trying to bind each data frame together after all the data has been scraped.

``` r
coerce_factor_to_char <- function(x) {
    if(is.factor(x)) {
        as.character(x)
    } else {
        x
    }
}
```

The following code executed a query to acquire each day's statcast batting data from &lt;baseballsavant.mlb.com&gt; for every player of every game for each query. The resulting data frame from each query was stored as an object in a list.

``` r
daily_statcast_list <- list()

for (date in game_days) {
    query_result <- scrape_statcast_savant_batter_all(date, date)
    query_result <- mutate_all(query_result, coerce_factor_to_char)
    query_result$game_day <- date # keep track of each day.
    daily_statcast_list[[date]] <- query_result # add each day's data frame to list. 
}
```

The 136 queries took about 15 seconds per query on my Macbook Air, for a total runtime of about 34 minutes.

Each data frame stored in the list was then coerced into a single data frame, effectively unioning all 136 data frames. The **rbindlist()** function from `data.table` seemed to work better than `dplyr` **bind\_rows()** or `base` **merge()** or **rbind()** functions for unioning lots of data frames that may have inconsistent properties (e.g. type differences).

``` r
install.packages("data.table")
library(data.table)
statcast_data <- rbindlist(daily_statcast_list)
```

Save results to disk
--------------------

The singular data frame was the exported to a feather file, using the `feather` package. [Feather](https://github.com/wesm/feather/blob/master/README.md) was developed as collaboration between R and Python developers to create a fast, light and language agnostic format for storing data frames.

``` r
install.packages("feather")
library(feather)
path_name = "/Users/bhoeft/Documents/R Files/MLB Statcast/statcast_batter_data.feather"
write_feather(statcast_data, path = path_name)
```

Load Feather file from disk
---------------------------

To read the data back in before an R session.

``` r
library(feather)
statcast_data <- read_feather(path = "/Users/bhoeft/Documents/R Files/MLB Statcast/statcast_batter_data.feather")
```
