---
layout: article
title: "Merging Comparative Manifesto Project and ParlGov cabinet composition data at the party-level"
author: "Hauke Licht"
date: "2019-03-07"
tags: [data wrangling, tidyverse, parties, policy positions, party data, manifestos, CMP, Parlgov, merging, joining, merge, join, missing data]
excerpt: "In this post, I use data from the Comparative Manifesto Project and ParlGov to find out whether the manifesto used in a given election was published by a party that was in government at this point in time."
mathjax: true
comments: false
---


## Goal of this Post

The data provided by the [Comparative Manifesto Project](https://manifesto-project.wzb.eu/) (CMP) is a rich and valuable source of information for researchers interested in parties policy positions and salience strategies.
The CMP main dataset records policy issue positions at the level of parties nested within countries and elections.
Specifically, issue positions and salience measurements are derived from parties' election manifestos.[^1]
These manifestos are usually produced as parts of parties campaigning efforts and regularly issued six to one month prior to upcoming elections.

[^1]: But see the [indicator `progtype`](https://manifesto-project.wzb.eu/down/data/2018b/codebooks/codebook_MPDataset_MPDS2018b.pdf) in the CMP data, distinguishing different techniques of obtaining policy position measurements.

In this post, we will focus on a particular piece of information that is lacking in the CMP data: an indicator of parties' government status that would allow to distinguish government from opposition parties.
Specifically, our goal is to **find out whether the manifesto used in a given election was published by a party that was in government at this point in time**. 

In order to accomplish this goal, I will add to the CMP data information contained in [ParlGov](www.parlgov.org)'s (PGV) [cabinet view](http://www.parlgov.org/data/table/view_cabinet/).
In the PGV cabinet view, elections map m:1 to countries, elections map 1:m to cabinets, and parties are nested in election-cabinet configurations.
It is a notable feature of this dataset that it keeps track of the different cabinets formed based on the results of a given election as well as of their timing (i.e., cabinet start dates).

## Similar but different: Relating CMP and PGV data

### Two data logics

Adding PGV indicators to the CMP data sounds simpler than it actually is (see also [here](https://manifesto-project.wzb.eu/tutorials/parlgov_merge) and [here](http://dimiter.eu/Data_files/gov_positions/government_positions_from_party_data.html)).
In the CMP data, manifesto-related indicators (e.g., issue position) map 1:1 to country-election-party configurations.
But the election date recorded is that of the post-campaign election.
That is, for a manifesto published at day $X$, the election on day $Y$ is the *upcoming* election in the sense that $X < Y$.

This stands in contrast to the PGV data, where the recorded election is only the point of reference during cabinet formation, so that if a cabinet forms on day $Z$ based on the results of election of day $Y$, we always have $Y < Z$.
As a consequence, a first complication when wanting to determine CMP parties' government status is that *we cannot simply join PGV cabinet data to CMP election data at the party-level using election dates*, but first need to identify the cabinet that was running (i.e., in office) when the election was held at which the CMP manifesto-related datasets are pointing.  

### Different IDs and incomplete lookup/link tables

The second complication is that parties are not only differently named and abbreviated in PGV than in CMP data, but that dataset-specific IDs are generally non-matching.
What is more, to my knowledge there exists no complete look-up table that would allow to match parties between datasets.

## Hands-on

But it would probably not have been so much fun to write this post, if I hadn't had found ways to solve these problems 👍
So let's get started!


### First things first




First, some basic setup: I load all required packages and define some helper functions (and add `roxygen2` docu, just in case):


{% highlight r %}
# setup ----

library(dplyr)
library(lubridate)
library(manifestoR)
library(readr)
library(tidyr)


#' If NA, then ...
#'
#' @description Given a scalar value `x`, function replaces it with a specified value if the input is NA
#'
#' @param x a scalar value to be evaluated in \code{is.na(x)}
#'
#' @param then the value to replace \code{x} with if \code{is.na(x)} evaluates to true \code{TRUE}
#'
#' @return if \code{is.na(x)}, then \code{then}, else \code{x}
if_na <- function(x, then) ifelse( is.na(x), then, x )


#' Is distinct?
#'
#' @description Given a dataframe, matrix, list or vector object, function test if the number of unique rows or elements 
#'     equals the total number of rows or elements, respectively.
#'
#' @param x dataframe, matrix, list or vector object
#'
#' @return logical
is_unique <- function(x) {
    if (inherits(x, "data.frame") | is.matrix(x)){
        nrow(pgv_elcs) == nrow(unique(pgv_elcs))
    } else if (is.vector(x) | is.list(x)) {
        length(pgv_elcs) == nrow(length(pgv_elcs))
    } else {
        stop("`x` must be a data.frame, list or vector object")
    }
}
{% endhighlight %}

Next we'll define a dataframe containing information on the countries we are interested in:


{% highlight r %}
countries <- read_csv(
'"country_name","country_iso2c","country_iso3c"
"Austria","AT","AUT"
"Belgium","BE","BEL"
"Denmark","DK","DNK"
"Finland","FI","FIN"
"France","FR","FRA"
"Germany","DE","DEU"
"Greece","GR","GRC"
"Ireland","IE","IRL"
"Italy","IT","ITA"
"Iceland","IS","ISL"
"Luxembourg","LU","LUX"
"Netherlands","NL","NLD"
"Norway","NO","NOR"
"Portugal","PT","PRT"
"Spain","ES","ESP"
"Sweden","SE","SWE"
"Switzerland","CH","CHE"
"United Kingdom","GB","GBR"'
)
{% endhighlight %}

### Get CMP data

Now we can download the CMP dataset using the manifesto-project's API.
Note that you first have to [register](https://manifesto-project.wzb.eu/signup) with manifesto-project.org and save the API key,[^2] and then load the API key using `mp_setapikey`:


{% highlight r %}
mp_setapikey("your/path/to/secret/manifesto_apikey.txt")
{% endhighlight %}

[^2]: See https://manifesto-project.wzb.eu/information/documents/api for detailed information on the API.


{% highlight r %}
# get available version
cmp_versions <- mp_coreversions()
{% endhighlight %}



{% highlight text %}
## Connecting to Manifesto Project DB API...
{% endhighlight %}



{% highlight r %}
# get newest version
(this_version <- cmp_versions$datasets.id[nrow(cmp_versions)])
{% endhighlight %}



{% highlight text %}
## [1] "MPDS2018b"
{% endhighlight %}



{% highlight r %}
# get data
cmp_raw <- mp_maindataset(version = this_version)
{% endhighlight %}



{% highlight text %}
## Connecting to Manifesto Project DB API... corpus version: 2018-2
{% endhighlight %}



{% highlight r %}
# do all match countries?
cmp_countries <- unique(cmp_raw$countryname)

all(cmp_countries %in% countries$country_name)
{% endhighlight %}



{% highlight text %}
## [1] FALSE
{% endhighlight %}



{% highlight r %}
all(countries$country_name %in% cmp_countries)
{% endhighlight %}



{% highlight text %}
## [1] TRUE
{% endhighlight %}

Checking our list of select countries against the countries contained in the CMP, 
we can verify that none of the countries we are interested in is missing in the CMP data.
Hence, I create the dataframe `cmp` that keeps only rows of select countries and only elections since the 1980s.


{% highlight r %}
cmp <- cmp_raw %>% 
    # keep only select countries
    filter(countryname %in% countries$country_name) %>% 
    mutate(edate = ymd(edate)) %>% 
    # keep only elections since the 80s
    filter(edate > ymd("1979-12-31")) %>% 
    left_join(countries, by = c("countryname" = "country_name")) %>% 
    mutate_if(is.character, trimws)
{% endhighlight %}


{% highlight r %}
head(cmp) %>% 
    select(1:7)
{% endhighlight %}



{% highlight text %}
## # A tibble: 6 x 7
##   country countryname oecdmember eumember edate        date party
##     <dbl> <chr>            <dbl>    <dbl> <date>      <dbl> <dbl>
## 1      11 Sweden              10        0 1982-09-19 198209 11220
## 2      11 Sweden              10        0 1982-09-19 198209 11320
## 3      11 Sweden              10        0 1982-09-19 198209 11420
## 4      11 Sweden              10        0 1982-09-19 198209 11620
## 5      11 Sweden              10        0 1982-09-19 198209 11810
## 6      11 Sweden              10        0 1985-09-15 198509 11220
{% endhighlight %}

Here, we can nicely inspect the structure of the CMP dataset:
Countries map 1:m elections, and elections 1:m to parties.

### Create unique CMP country-election-party combinations 

Now we can get all distinct combinations of countries, elections and parties in the CMP data (dataframe `cmp_elc_ptys`), and validate that no invalid entries are in the resulting dataset.


{% highlight r %}
cmp_elc_ptys <- cmp %>% 
    select(country_iso3c, edate, party, partyname, partyabbrev) %>% 
    rename_at(-1, ~ paste0("cmp_", .)) %>% 
    unique()

# any party more than one abbreviation?
cmp_elc_ptys %>% 
    group_by(country_iso3c, cmp_edate, cmp_party) %>% 
    summarize(n_abrvs = n_distinct(cmp_partyabbrev)) %>% 
    filter(n_abrvs > 1)
{% endhighlight %}



{% highlight text %}
## # A tibble: 0 x 4
## # Groups:   country_iso3c, cmp_edate [0]
## # … with 4 variables: country_iso3c <chr>, cmp_edate <date>,
## #   cmp_party <dbl>, n_abrvs <int>
{% endhighlight %}



{% highlight r %}
# any party more than one ID?
cmp_elc_ptys %>% 
    group_by(country_iso3c, cmp_edate, cmp_partyname) %>% 
    summarize(n_ids = n_distinct(cmp_party)) %>% 
    filter(n_ids > 1)
{% endhighlight %}



{% highlight text %}
## # A tibble: 0 x 4
## # Groups:   country_iso3c, cmp_edate [0]
## # … with 4 variables: country_iso3c <chr>, cmp_edate <date>,
## #   cmp_partyname <chr>, n_ids <int>
{% endhighlight %}

### Get PGV data

Next we download the most actual cabinets and parties views from the PGV website, 
and validate that all countries of interest are contained it.


{% highlight r %}
cabs <- read_csv("http://www.parlgov.org/static/data/development-cp1252/view_cabinet.csv", locale = locale(encoding = 'ISO-8859-1'))
ptys <- read_csv("http://www.parlgov.org/static/data/development-cp1252/view_party.csv", locale = locale(encoding = 'ISO-8859-1'))

pgv_ctrs <- unique(cabs$country_name_short)

# ensure compatibility: all PGV in CMP?
all(unique(cmp$country_iso3c) %in% pgv_ctrs)
{% endhighlight %}



{% highlight text %}
## [1] TRUE
{% endhighlight %}

### Identify 'running' cabinets in PGV data

The next step is crucial! 
As explained above, we are interested in running cabinet parties at the day of an election. 
Hence, we need to leverage both election data and cabinet start date information to identify just these.
This is accomplished by the following very long, but extensively commented expression:


{% highlight r %}
# for PGV country-election-cabinets, ...
pgv_elcs <- cabs %>%
    # keep select countries
    filter(country_name_short %in% unique(cmp$country_iso3c)) %>%
    # go back further to also retain previous cabs
    filter(ymd(election_date) > ymd("1969-12-31")) %>%
    # get distinct country-election configurations
    select(country_name_short, election_date) %>%
    unique() %>% 
    # get date of next election
    group_by(country_name_short) %>% 
    mutate(next_election_date = lead(election_date)) %>% 
    # add cabinet info
    left_join(cabs, by = c("country_name_short", "election_date")) %>% 
    # keep select columns
    select(
        country_name_short, 
        election_date, next_election_date,
        cabinet_id, cabinet_name, start_date
    ) %>%
    unique() %>% 
    # keep only cabinet last formed from a given election 
    # (NOTE: this is the cabinet ruinning when the next election was held)
    group_by(country_name_short, election_date) %>% 
    filter(start_date == max(start_date)) %>% 
    # (NOTE: the previous step makes rows uniquely identified election dates 
    #  within countries, since only one cabinet per country-election is retained)
    ungroup() %>% 
    # again apply date filter
    filter(next_election_date > ymd("1974-12-31")) %>% 
    # rename
    rename(
        country_iso3c = country_name_short
        , cabinet_start_date = start_date
    ) %>%
    rename_at(-1, ~ paste0("pgv_", .))
{% endhighlight %}


{% highlight r %}
# Nr. rows = Nr. distinct rows?
is_unique(pgv_elcs)
{% endhighlight %}



{% highlight text %}
## [1] TRUE
{% endhighlight %}


{% highlight r %}
head(pgv_elcs)
{% endhighlight %}



{% highlight text %}
## # A tibble: 6 x 6
##   country_iso3c pgv_election_da… pgv_next_electi… pgv_cabinet_id
##   <chr>         <date>           <date>                    <int>
## 1 AUT           1971-10-10       1975-10-05                  365
## 2 AUT           1975-10-05       1979-05-06                  350
## 3 AUT           1979-05-06       1983-04-24                  463
## 4 AUT           1983-04-24       1986-11-23                  828
## 5 AUT           1986-11-23       1990-10-07                  873
## 6 AUT           1990-10-07       1994-10-09                  524
## # … with 2 more variables: pgv_cabinet_name <chr>,
## #   pgv_cabinet_start_date <date>
{% endhighlight %}

The logic in `pgv_elcs` is the following: 

- Each row is a unique country-election configuration.
- Column `election_date` records the actual date of the election, while 
  column `next_election_date` records the date of the upcoming election.
- When we compare `start_date` (the selected cabinet's start date) and 
  `next_election_date`, we can confirm that the upcoming election falls 
  into the period of activity of the cabinet we selected.
- Parties in the 'Kreisky IV' cabinet (Austria), which was formed from the 
  parliament elected on 1979-05-06 and took office on 1979-06-05, for instance, 
  are those parties that were in office when the next election was held 
  (on 1983-04-24).
- This 'next' election, in turn, is the one used in the CMP data to index 
  policy positions of those parties who managed to enter a parliament in 
  the given election.
- Note, however, that these parties have likely issued their manifestos *before*
  they were elected into office.
- Hence, if we want to know whether a manifesto was written by a party that
  was holding government office, we need to look at the cabinet that was 
  *running* (i.e., in office) when the election was held (in the above example, 
  parties in the 'Kreisky IV' cabinet, not in the 'Vranitzky I', which took office 
  on 1986-06-16 and was only formed based on the results of the 1983-04-24 election).
<!-- - since manifestos are virtually always issued before an election, and in 
  the CMP the party data recorded for a given country-election configuration 
  is based on the manifestos issued prior to this election, we want to match 
  to a CMP country-election configurations the status of running cabinets
  (e.g., parties in the Kreisky IV cabinet), not of thos cabinets that were
  only formed from the given election (e.g., the Vranitzky I, taking office
  on 1986-06-16, only about two month _after_ the 1983-04-24 election)
--> 


### Create a link table matching CMP to PGV election dates

Once I have leveraged the PGV data to identify running cabinets for elections, I want to allow us to add this information to the CMP data.
For this purpose I create a new link table mapping CMP to PGV elections.
This step, too, requires some great care, since we cannot simply assume that election dates always match exactly.


{% highlight r %}
# join country-elections on (inexact) election dates
ctr_elcs <- pgv_elcs %>% 
    # NOTE: take next election due to differing data logics in CMP and PGV (see explanation above)
    select(country_iso3c, pgv_next_election_date) %>% 
    unique() %>% 
    # get cross-product of elections within countries 
    full_join(
        # join CMP data
        cmp_elc_ptys %>% 
            select(country_iso3c, cmp_edate) %>% 
            unique()
        , by = "country_iso3c"
    ) %>% 
    # compute date difference in days for each CMP-PGV election data pairing 
    # THIS POINT IS KEY!: take next election in PGV data due to differing data logics in CMP and PGV
    mutate(abs_elc_date_diff = abs(pgv_next_election_date - cmp_edate)) %>%
    # at the country CMP-election level (reference data), keep the PGV (next) 
    #   election with the lowest data difference (0 days all but a few instance)
    group_by(country_iso3c, cmp_edate) %>% 
    top_n(1, wt = desc(abs_elc_date_diff)) %>% 
    ungroup()
{% endhighlight %}

As commented in the code, it is crucial to match CMP election dates to the 'next'/'upcoming' election in the PGV data, since this is the one mapping to the cabinet that contains running cabinet parties.


{% highlight r %}
# is distinct?
is_unique(pgv_elcs)
{% endhighlight %}



{% highlight text %}
## [1] TRUE
{% endhighlight %}



{% highlight r %}
# ensure that no CMP election maps to multiple PGV elections
ctr_elcs %>% 
    group_by(country_iso3c, cmp_edate) %>% 
    summarize(n_pgv_dates = n_distinct(pgv_next_election_date)) %>% 
    filter(n_pgv_dates  > 1)
{% endhighlight %}



{% highlight text %}
## # A tibble: 0 x 3
## # Groups:   country_iso3c [0]
## # … with 3 variables: country_iso3c <chr>, cmp_edate <date>,
## #   n_pgv_dates <int>
{% endhighlight %}



{% highlight r %}
# ensure that no PGV election maps to multiple CMP elections
ctr_elcs %>% 
    group_by(country_iso3c, pgv_next_election_date) %>% 
    summarize(n_cmp_dates = n_distinct(cmp_edate)) %>% 
    filter(n_cmp_dates  > 1)
{% endhighlight %}



{% highlight text %}
## # A tibble: 0 x 3
## # Groups:   country_iso3c [0]
## # … with 3 variables: country_iso3c <chr>,
## #   pgv_next_election_date <date>, n_cmp_dates <int>
{% endhighlight %}



{% highlight r %}
# inexact (best) matches ? 
ctr_elcs %>% 
    filter(abs_elc_date_diff > 0)
{% endhighlight %}



{% highlight text %}
## # A tibble: 13 x 4
##    country_iso3c pgv_next_election_date cmp_edate  abs_elc_date_diff
##    <chr>         <date>                 <date>     <time>           
##  1 FRA           1981-06-21             1981-06-14 7 days           
##  2 FRA           1988-06-12             1988-06-05 7 days           
##  3 FRA           1993-03-28             1993-03-21 7 days           
##  4 FRA           1997-06-01             1997-05-25 7 days           
##  5 FRA           2002-06-16             2002-06-09 7 days           
##  6 FRA           2007-06-17             2007-06-10 7 days           
##  7 FRA           2012-06-17             2012-06-10 7 days           
##  8 FRA           2017-06-18             2017-06-11 7 days           
##  9 ITA           1992-04-05             1992-04-06 1 days           
## 10 ITA           1994-03-27             1994-03-28 1 days           
## 11 ITA           2006-04-09             2006-04-10 1 days           
## 12 ITA           2013-02-25             2013-02-24 1 days           
## 13 SWE           1998-09-20             1998-09-21 1 days
{% endhighlight %}

While we can verify that neither any CMP election maps to multiple PGV elections, nor vice versa,
the last query returns some configurations where election dates do not match exactly.
But inspecting date differences allows to conclude that all differences are rather small (i.e., ≤ 7 days), so there is little reason to put the veracity of this data into question.

Having matched CMP and PGV elections, I can construct the link table matching CMP country-elections to PGV country-election-(running-)cabinet configurations:


{% highlight r %}
# take country-election link table ...
running_cabinets <- ctr_elcs %>% 
    # and left-join PGV cabinet info (only info of running cabinets matched)
    left_join(pgv_elcs, by = c("country_iso3c", "pgv_next_election_date")) %>% 
    rename(
        pgv_election_date = pgv_next_election_date
        , pgv_running_cabinet_id = pgv_cabinet_id
        , pgv_running_cabinet_name = pgv_cabinet_name
        , pgv_running_cabinet_start_date = pgv_cabinet_start_date
        , pgv_running_cabinet_election_date = pgv_election_date
    ) %>% 
    select(
        country_iso3c
        , cmp_edate
        , pgv_election_date
        , pgv_running_cabinet_name
        , pgv_running_cabinet_id
        , pgv_running_cabinet_start_date
        , pgv_running_cabinet_election_date
    )
{% endhighlight %}

We can validate that I have matched exactly one running cabinet to each CMP election:


{% highlight r %}
# any duplicates?
running_cabinets %>% 
    group_by(country_iso3c, cmp_edate) %>% 
    filter(n_distinct(pgv_running_cabinet_id) != 1)
{% endhighlight %}



{% highlight text %}
## # A tibble: 0 x 7
## # Groups:   country_iso3c, cmp_edate [0]
## # … with 7 variables: country_iso3c <chr>, cmp_edate <date>,
## #   pgv_election_date <date>, pgv_running_cabinet_name <chr>,
## #   pgv_running_cabinet_id <int>,
## #   pgv_running_cabinet_start_date <date>,
## #   pgv_running_cabinet_election_date <date>
{% endhighlight %}


### Create link table matching CMP and PGV party IDs

Now that I've solved the first problem (matching running cabinets to the CMP data),
I can deal with the other problem: matching PGV to CMP parties.
I'll use the link-table provided by [PartyFacts](https://partyfacts.herokuapp.com).
We first download the file:

{% highlight r %}
# define your data dir
data_dir <- "path/to/your/data/directory"

# download link table
file_name <- "partyfacts-mapping.csv"
if (!file_name %in% list.files(data_dir)) {
    url <- "https://partyfacts.herokuapp.com/download/external-parties-csv/"
    download.file(url, file.path(data_dir, file_name))
}
{% endhighlight %}



Next, we create a link table matching PGV and CMP party IDs:

{% highlight r %}
# read link table
ptf <- read_csv(file.path(data_dir, file_name))
# NOTE: on 2019-02-23, this raised some warnings, which can be ignored

# check if all coiuntries covered in party-facts (PTF) data
all(countries$country_iso3c %in% ptf$country)
{% endhighlight %}



{% highlight text %}
## [1] TRUE
{% endhighlight %}



{% highlight r %}
# create CMP-PGV party link table
pty_links <- ptf %>% 
    # for select countries
    filter(country %in% countries$country_iso3c) %>% 
    # take parties CMP IDs
    filter(dataset_key == "manifesto") %>% 
    select(partyfacts_id, country, dataset_party_id) %>% 
    # inner join drops both CMP parties that have no matching PGV ID,
    # and PGV parties for which no matching CMP code exists
    inner_join(
        ptf %>% 
            # parties PGV IDs (where possible)
            filter(dataset_key == "parlgov") %>% 
            select(partyfacts_id, country, dataset_party_id)
        , by = c("partyfacts_id" = "partyfacts_id")
        , suffix = c("_cmp", "_pgv")
    ) %>% 
    filter(country_cmp == country_pgv) %>% 
    select(-country_pgv) %>% 
    rename(
        country_iso3c = country_cmp
        , cmp_party = dataset_party_id_cmp
        , pgv_party_id = dataset_party_id_pgv
        , ptf_party_id = partyfacts_id
    ) %>% 
    mutate_at(3:4, as.integer)
{% endhighlight %}

We get a dataframe with 310 rows, but there are both instances of 1:m and m:1 PGV to CMP party matchings, as the below queries demonstrate:


{% highlight r %}
nrow(pty_links)
{% endhighlight %}



{% highlight text %}
## [1] 310
{% endhighlight %}



{% highlight r %}
# is distinct?
is_unique(pty_links)
{% endhighlight %}



{% highlight text %}
## [1] TRUE
{% endhighlight %}



{% highlight r %}
# any PGV ID matches to multiple CMP parties?
pty_links %>% 
    group_by(pgv_party_id) %>% 
    filter(n_distinct(cmp_party) > 1)
{% endhighlight %}



{% highlight text %}
## # A tibble: 27 x 4
## # Groups:   pgv_party_id [11]
##    ptf_party_id country_iso3c cmp_party pgv_party_id
##           <int> <chr>             <int>        <int>
##  1          698 BEL               21912          969
##  2          554 BEL               21422          454
##  3          698 BEL               21423          969
##  4          554 BEL               21425          454
##  5           36 BEL               21916          501
##  6           36 BEL               21913          501
##  7         1586 BEL               21221         1113
##  8         1586 BEL               21330         1113
##  9          516 GRC               34020         1441
## 10          516 GRC               34211         1441
## # … with 17 more rows
{% endhighlight %}



{% highlight r %}
# any CMP ID matches to multiple PGV parties?
pty_links %>% 
    group_by(cmp_party) %>% 
    filter(n_distinct(pgv_party_id) > 1)
{% endhighlight %}



{% highlight text %}
## # A tibble: 15 x 4
## # Groups:   cmp_party [6]
##    ptf_party_id country_iso3c cmp_party pgv_party_id
##           <int> <chr>             <int>        <int>
##  1          878 ITA               32220          809
##  2          878 ITA               32220         2666
##  3          851 ITA               32110          910
##  4          851 ITA               32110         1304
##  5         4019 ESP               33099         2607
##  6         4019 ESP               33099         2606
##  7         4019 ESP               33099           62
##  8         4019 ESP               33020         2607
##  9         4019 ESP               33020         2606
## 10         4019 ESP               33020           62
## 11         4019 ESP               33098         2607
## 12         4019 ESP               33098         2606
## 13         4019 ESP               33098           62
## 14         1567 GBR               51620          773
## 15         1567 GBR               51620         1496
{% endhighlight %}

Below we'll see whether this 1:m and m:1 matching PGV:CMP IDs disappears once we add country-election(-cabinet) info.


{% highlight r %}
# how many matched?
cmp_elc_ptys %>% 
    left_join(pty_links) %>% 
    summarise(
        n = n()
        , n_matched = sum(!is.na(pgv_party_id))
    )
{% endhighlight %}



{% highlight text %}
## # A tibble: 1 x 2
##       n n_matched
##   <int>     <int>
## 1  1398      1331
{% endhighlight %}



{% highlight r %}
# any CMP party in data matches to no PGV parties?
cmp_elc_ptys %>% 
    left_join(pty_links) %>% 
    group_by(country_iso3c, cmp_edate, cmp_party) %>% 
    filter(n_distinct(pgv_party_id, na.rm = TRUE) < 1)
{% endhighlight %}



{% highlight text %}
## # A tibble: 67 x 7
## # Groups:   country_iso3c, cmp_edate, cmp_party [67]
##    country_iso3c cmp_edate  cmp_party cmp_partyname cmp_partyabbrev
##    <chr>         <date>         <dbl> <chr>         <chr>          
##  1 FIN           1991-03-17     14223 Left Wing Al… VAS            
##  2 FIN           1995-03-19     14223 Left Wing Al… VAS            
##  3 FIN           1999-03-21     14223 Left Wing Al… VAS            
##  4 FIN           2003-03-16     14223 Left Wing Al… VAS            
##  5 FIN           2007-03-18     14223 Left Wing Al… VAS            
##  6 FIN           2011-04-17     14223 Left Wing Al… VAS            
##  7 BEL           2007-06-10     21917 Flemish Inte… VB             
##  8 BEL           2010-06-13     21917 Flemish Inte… VB             
##  9 NLD           1982-09-08     22710 Centre Party  ""             
## 10 NLD           1994-05-03     22955 Union 55+     Unie 55+       
## # … with 57 more rows, and 2 more variables: ptf_party_id <int>,
## #   pgv_party_id <int>
{% endhighlight %}



{% highlight r %}
# any CMP party in data matches to multiple PGV parties?
cmp_elc_ptys %>% 
    left_join(pty_links) %>% 
    group_by(country_iso3c, cmp_edate, cmp_party) %>% 
    filter(n_distinct(pgv_party_id, na.rm = TRUE) > 1)
{% endhighlight %}



{% highlight text %}
## # A tibble: 52 x 7
## # Groups:   country_iso3c, cmp_edate, cmp_party [24]
##    country_iso3c cmp_edate  cmp_party cmp_partyname cmp_partyabbrev
##    <chr>         <date>         <dbl> <chr>         <chr>          
##  1 ITA           1983-06-26     32220 Italian Comm… PCI            
##  2 ITA           1983-06-26     32220 Italian Comm… PCI            
##  3 ITA           1987-06-14     32110 Green Federa… FdV            
##  4 ITA           1987-06-14     32110 Green Federa… FdV            
##  5 ITA           1987-06-14     32220 Italian Comm… PCI            
##  6 ITA           1987-06-14     32220 Italian Comm… PCI            
##  7 ITA           1992-04-06     32110 Green Federa… FdV            
##  8 ITA           1992-04-06     32110 Green Federa… FdV            
##  9 ITA           1992-04-06     32220 Democratic P… PDS            
## 10 ITA           1992-04-06     32220 Democratic P… PDS            
## # … with 42 more rows, and 2 more variables: ptf_party_id <int>,
## #   pgv_party_id <int>
{% endhighlight %}

We see that there exist no matching PGV party IDs for some configurations in the CMP data. 
Note, however, that this is problematic only insofar as we want to get the government status info from PGV: if we cannot match a PGV party to a CMP party, we cannot say whether the given (unmatched) CMP party was in government or not. 

I'll deal with this problem in the next step.
Specifically, I'll check if further matching efforts need to be undertaken in case we cannot match all gov't parties in a configuration 

### Check if all gov't parties can be matched

First, we enrich the CMP data by PGV party IDs (where possible) as provided in the link table.


{% highlight r %}
cmp_w_pgv_ids <- cmp_elc_ptys %>% 
    left_join(pty_links, by = c("country_iso3c", "cmp_party")) %>% 
    select(-ptf_party_id) %>% 
    group_by(country_iso3c, cmp_edate, cmp_party) %>% 
    mutate(cmp_n_pgv_ids = n_distinct(pgv_party_id, na.rm = TRUE)) %>% 
    ungroup()
{% endhighlight %}

Then we right-join party information from the PGV cabinets view to the dataset containing running cabinet info (`running_cabinets`) created above.


{% highlight r %}
# create running-cabinet party-level dataset
running_parties <- cabs %>% 
    # select party-cabinet info from original PGV cabinets view
    select(
        country_name_short, cabinet_id,
        party_id, party_name_english, party_name_short,
        caretaker, cabinet_party
    ) %>%
    # add party CMP ID (where exists) from original PGV parties view
    left_join(
        ptys %>% select(party_id, cmp)
        , by = "party_id"
    ) %>%
    # compute cabinet size
    group_by(country_name_short, cabinet_id) %>%
    mutate(cabinet_size = sum(cabinet_party)) %>%
    ungroup() %>% 
    # add prefixes to all but the first column
    rename_at(-1, ~ paste0("pgv_", .)) %>%
    # join only running cabinets
    right_join(
        running_cabinets
        , by = c(
            "country_name_short" = "country_iso3c"
            , "pgv_cabinet_id" = "pgv_running_cabinet_id"
        )
    ) %>% 
    # rename
    rename(
        country_iso3c = country_name_short
        , pgv_running_cabinet_id = pgv_cabinet_id
    ) %>% 
    # select columns
    select(
        # all columns as ordered in `running_cabinets` dataframe
        !!names(running_cabinets)
        # other columns ...
        , pgv_cabinet_size
        , pgv_caretaker
        , pgv_party_name_short
        , pgv_party_name_english
        , pgv_party_id
        , pgv_cabinet_party
        , pgv_cmp
    )
{% endhighlight %}

Now we are ready to join the information on running cabinet parties to the CMP dataset.
We use PGV party IDs, keeping in mind that there is a subset of parties in the CMP data for which we could not identify a matching PGV party.

Specifically, we perform a full outer join which gives us a stacked version of the CMP and PGV data, containing:

1. all matching country-election-party pairings
2. all non-matching country-election-parties from the CMP data
3. all non-matching country-election-parties from the PGV data


{% highlight r %}
# join PGV party-level data to PGV-ID-enriched CMP country-election-party data
cmp_full_join_pgv <- cmp_w_pgv_ids %>% 
    # add running parties (full outer join!)
    full_join(running_parties, by = c("country_iso3c", "cmp_edate", "pgv_party_id")) 
{% endhighlight %}

We can better understand the structure of the resulting dataset by inspecting an example configuration:


{% highlight r %}
# inspect an example configuration
cmp_full_join_pgv %>% 
    filter(country_iso3c == "BEL", (pgv_election_date == "2007-06-10" | cmp_edate == "2007-06-10")) %>% 
    select(
        country_iso3c, cmp_edate, 
        pgv_running_cabinet_name, pgv_running_cabinet_start_date, 
        cmp_partyabbrev, pgv_party_name_short
    )
{% endhighlight %}



{% highlight text %}
## # A tibble: 15 x 6
##    country_iso3c cmp_edate  pgv_running_cab… pgv_running_cab…
##    <chr>         <date>     <chr>            <date>          
##  1 BEL           2007-06-10 Verhofstadt II   2003-07-12      
##  2 BEL           2007-06-10 <NA>             NA              
##  3 BEL           2007-06-10 Verhofstadt II   2003-07-12      
##  4 BEL           2007-06-10 <NA>             NA              
##  5 BEL           2007-06-10 Verhofstadt II   2003-07-12      
##  6 BEL           2007-06-10 Verhofstadt II   2003-07-12      
##  7 BEL           2007-06-10 Verhofstadt II   2003-07-12      
##  8 BEL           2007-06-10 Verhofstadt II   2003-07-12      
##  9 BEL           2007-06-10 <NA>             NA              
## 10 BEL           2007-06-10 Verhofstadt II   2003-07-12      
## 11 BEL           2007-06-10 Verhofstadt II   2003-07-12      
## 12 BEL           2007-06-10 Verhofstadt II   2003-07-12      
## 13 BEL           2007-06-10 <NA>             NA              
## 14 BEL           2007-06-10 Verhofstadt II   2003-07-12      
## 15 BEL           2007-06-10 Verhofstadt II   2003-07-12      
## # … with 2 more variables: cmp_partyabbrev <chr>,
## #   pgv_party_name_short <chr>
{% endhighlight %}

In this example, the Belgian 'Verhofstadt II' cabinet (PGV naming) taking office on 2003-07-12, 
parties 'VB' and 'FN' in the PGV data could not be matched to any party within this country-election 
configuration in the CMP dataset when using PGV party IDs (as obtained from the party-facts link table).
Conversely, parties 'groen!', 'sp.a', 'LDD' and 'VB' in the CMP dataset could not be matched to any party 
within this country-election configuration in the PGV data.
Significantly, despite 'VB' exists in both datasets, the CMP 'VB' could not be matched to the PGV 'VB' because the PGV party ID obtained trough the party-facts link table mismatches the one used in the original PGV data.

What's more is that CMP parties for which no PGV party could be matched using PGV party IDs have `NA`s on columns originating from the PGV data frame, and vice versa.
So we want to fill-in missing information.
I will do this by imputing missings from configuration context where possible.[^3]

[^3]: In the example of 'Verhofstadt II' cabinet, we know, for instance, that we can write 'Verhofstadt II' in column `pgv_running_cabinet_name` where there are currently `NA`s because we matched PGV to CMP data using our  country-election date look-up table, and we know that we have selected only one cabinet configuration per election from the PGV data (the 'running' cabinet). Hence there are no ambiguities at the election-cabinet level.

That's what I do next:


{% highlight r %}
# fill-in missing information by inferring from configuration's contexts
cmp_full_join_pgv_filled <- cmp_full_join_pgv %>% 
    # a) take PGV country-election groupings ...
    group_by(country_iso3c, pgv_election_date) %>% 
    #    ... fill-in missing CMP election date (can be inferred from grouping)
    fill(cmp_edate, .direction = "up") %>%
    fill(cmp_edate, .direction = "down") %>%
    #    ... and remove PGV configurations that are completly missing in CMP data (if any)
    filter(!is.na(cmp_edate)) %>% # should be all
    # b) take CMP country-election groupings ...
    group_by(country_iso3c, cmp_edate) %>% 
    #    ... and fill-in missing but inferrable PGV info
    fill(
        pgv_election_date
        , pgv_running_cabinet_start_date
        , pgv_running_cabinet_name
        , pgv_running_cabinet_id 
        , pgv_running_cabinet_election_date
        , pgv_cabinet_size
        , pgv_caretaker
        , .direction = "down"
    ) %>%
    fill(
        pgv_election_date
        , pgv_running_cabinet_start_date
        , pgv_running_cabinet_name
        , pgv_running_cabinet_id 
        , pgv_running_cabinet_election_date
        , pgv_cabinet_size
        , pgv_caretaker
        , .direction = "up"
    ) %>% 
    ungroup()
{% endhighlight %}

The result of filling-in missing but inferable information from the context gives complete configuration for which we know which CMP party is non-matching in PGV data and vice versa:


{% highlight r %}
cmp_full_join_pgv_filled %>% 
    filter(country_iso3c == "BEL",  cmp_edate == "2007-06-10") %>% 
    select(
        country_iso3c, cmp_edate, 
        pgv_running_cabinet_name, pgv_running_cabinet_start_date, 
        cmp_partyabbrev, pgv_party_name_short,
        pgv_cabinet_size, pgv_cabinet_party
    )
{% endhighlight %}



{% highlight text %}
## # A tibble: 15 x 8
##    country_iso3c cmp_edate  pgv_running_cab… pgv_running_cab…
##    <chr>         <date>     <chr>            <date>          
##  1 BEL           2007-06-10 Verhofstadt II   2003-07-12      
##  2 BEL           2007-06-10 Verhofstadt II   2003-07-12      
##  3 BEL           2007-06-10 Verhofstadt II   2003-07-12      
##  4 BEL           2007-06-10 Verhofstadt II   2003-07-12      
##  5 BEL           2007-06-10 Verhofstadt II   2003-07-12      
##  6 BEL           2007-06-10 Verhofstadt II   2003-07-12      
##  7 BEL           2007-06-10 Verhofstadt II   2003-07-12      
##  8 BEL           2007-06-10 Verhofstadt II   2003-07-12      
##  9 BEL           2007-06-10 Verhofstadt II   2003-07-12      
## 10 BEL           2007-06-10 Verhofstadt II   2003-07-12      
## 11 BEL           2007-06-10 Verhofstadt II   2003-07-12      
## 12 BEL           2007-06-10 Verhofstadt II   2003-07-12      
## 13 BEL           2007-06-10 Verhofstadt II   2003-07-12      
## 14 BEL           2007-06-10 Verhofstadt II   2003-07-12      
## 15 BEL           2007-06-10 Verhofstadt II   2003-07-12      
## # … with 4 more variables: cmp_partyabbrev <chr>,
## #   pgv_party_name_short <chr>, pgv_cabinet_size <int>,
## #   pgv_cabinet_party <int>
{% endhighlight %}

So now we are ready to again deal with the core problem: the fact that we cannot match PGV party info to some CMP parties.
This fact is problematic for our purpose (identifying parties' government  status) *if and only if* not all PGV parties that are recorded as cabinet party  for a given configuration can be matched.
This is because if we can match all  cabinet parties, we can infer the government status of non-matched parties: they are all not cabinet parties.
 
A straight-forward way to validate this condition is to try to replicate the `cabinet_size` measure.
The logic is simple: 
For configurations in which aggregating party cabinet membership information at the country-election(-cabinet) level does not allow us to replicate this measure, we know that we failed to match at least one government party.
In such a case, we could only infer the missing government status information if only one party in this configuration was not matched.
Otherwise, we would not have enough information to infer non-matching parties government status.

So I check this in the dataset `cmp_full_join_pgv_filled`:



{% highlight r %}
cmp_full_join_pgv_filled %>% 
    group_by(country_iso3c, cmp_edate, pgv_running_cabinet_start_date) %>%
    mutate(
        postmatch_cabinet_size = n_distinct(ifelse(pgv_cabinet_party == 1, pgv_party_name_short, NA), na.rm = TRUE)
        , flag = pgv_cabinet_size == postmatch_cabinet_size
    ) %>%
    filter(!flag)
{% endhighlight %}



{% highlight text %}
## # A tibble: 0 x 20
## # Groups:   country_iso3c, cmp_edate, pgv_running_cabinet_start_date
## #   [0]
## # … with 20 variables: country_iso3c <chr>, cmp_edate <date>,
## #   cmp_party <dbl>, cmp_partyname <chr>, cmp_partyabbrev <chr>,
## #   pgv_party_id <int>, cmp_n_pgv_ids <int>,
## #   pgv_election_date <date>, pgv_running_cabinet_name <chr>,
## #   pgv_running_cabinet_id <int>,
## #   pgv_running_cabinet_start_date <date>,
## #   pgv_running_cabinet_election_date <date>, pgv_cabinet_size <int>,
## #   pgv_caretaker <int>, pgv_party_name_short <chr>,
## #   pgv_party_name_english <chr>, pgv_cabinet_party <int>,
## #   pgv_cmp <int>, postmatch_cabinet_size <int>, flag <lgl>
{% endhighlight %}

Wow! We are lucky: I was able to replicate the `cabinet_size` measure for all configurations in the dataset, and hence can infer that parties that could not be matched are invariable opposition parties.[^4]

[^4]: In case you wonder about the clumsy definition of column `postmatch_cabinet_size`: Since some PGV parties match to multiple CMP parties, I need to count distinct PGV parties to exactly replicate the  `cabinet_size` measure. If I would instead have used `postmatch_cabinet_size = sum(pgv_cabinet_party, na.rm = NA)`, the indicator would have double counted m:1 matched CMP parties, and in these cases `postmatch_cabinet_size > pgv_cabinet_size`.

### Create the complete dataset 

Now I have almost reached my goal. 
The final step is to add the inferred government status information (i.e., replace `NA` values with `0`), drop PGV parties that are non-matching in the CMP data, and gather all datasets at the level of CMP country-election-party configurations.


{% highlight r %}
# take the filled-in fully-joined dataset
cmp_w_pty_govt_status <- cmp_full_join_pgv_filled %>% 
    # get rid of PGV parties that are non-matching in CMP data
    filter(!is.na(cmp_party)) %>% 
    # infer missing government status by ...
    # ... a) looking within CMP country-election-party configurations, and
    group_by(country_iso3c, cmp_edate, cmp_party) %>% 
    fill(pgv_cabinet_party) %>% 
    # ... b) replace with 0 where still NA
    mutate(pgv_cabinet_party = ifelse(is.na(pgv_cabinet_party), 0, pgv_cabinet_party)) %>% 
    # aggregate at party-level within CMP data
    group_by(
        country_iso3c
        , cmp_edate
        , pgv_running_cabinet_name
        , pgv_running_cabinet_id
        , pgv_running_cabinet_start_date
        , pgv_running_cabinet_election_date
        , pgv_cabinet_size
        , pgv_caretaker
        , cmp_party
        , cmp_partyname
        , cmp_partyabbrev
        , pgv_cabinet_party
    ) %>% 
    # add informative comments
    summarize(
        comment = case_when(
            cmp_n_pgv_ids == 0 ~ "no matching party found within matching ParlGov cabinet configuration"
            , cmp_n_pgv_ids == 1 ~ sprintf(
                "CMP party matches to party %s (%s) within ParlGov cabinet configuration"
                , if_na(pgv_party_name_short, "?")
                , if_na(pgv_party_id, "?")
            )
            , cmp_n_pgv_ids > 1 ~ sprintf(
                "CMP party matches to multiple parties within ParlGov cabinet configuration: %s"
                , paste0(
                    sprintf(
                        "%s (%s)"
                        , if_na(pgv_party_name_short, "?")
                        , if_na(pgv_party_id, "?")
                    )
                    , collapse = ", "
                )
            )
            , TRUE ~ NA_character_
        # in order to aggregate, keep distinct comments (always 1 within grouping)
        ) %>% unique()
    ) %>% 
    ungroup()
{% endhighlight %}

We can verify that we have added valid government status information to every row in the original CMP dataset:

{% highlight r %}
nrow(cmp_w_pty_govt_status) == nrow(cmp)
{% endhighlight %}



{% highlight text %}
## [1] TRUE
{% endhighlight %}

The resulting dataset contains just everything we need to join it back to the original CMP dataset `cmp_raw` ...

{% highlight r %}
head(cmp_w_pty_govt_status)
{% endhighlight %}



{% highlight text %}
## # A tibble: 6 x 13
##   country_iso3c cmp_edate  pgv_running_cab… pgv_running_cab…
##   <chr>         <date>     <chr>                       <int>
## 1 AUT           1983-04-24 Kreisky IV                    463
## 2 AUT           1983-04-24 Kreisky IV                    463
## 3 AUT           1983-04-24 Kreisky IV                    463
## 4 AUT           1986-11-23 Vranitzky I                   828
## 5 AUT           1986-11-23 Vranitzky I                   828
## 6 AUT           1986-11-23 Vranitzky I                   828
## # … with 9 more variables: pgv_running_cabinet_start_date <date>,
## #   pgv_running_cabinet_election_date <date>, pgv_cabinet_size <int>,
## #   pgv_caretaker <int>, cmp_party <dbl>, cmp_partyname <chr>,
## #   cmp_partyabbrev <chr>, pgv_cabinet_party <dbl>, comment <chr>
{% endhighlight %}



... but I leave this exercise to you 😊

