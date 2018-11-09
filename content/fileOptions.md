---
author: Laura DeCicco
date: 2018-11-07
slug: formats
draft: True
title: Pretty big data… now what?
type: post
categories: Data Science
image: static/comparison.jpg
author_twitter: DeCiccoDonk
author_github: ldecicco-usgs
author_gs: jXd0feEAAAAJ
 
author_staff: laura-decicco
author_email: <ldecicco@usgs.gov>

tags: 
  - R
 
 
description: Exploring file format options in R.
keywords:
  - R
 
 
  - files
  - io
---
In the group that I work with <https://owi.usgs.gov/datascience/>, the
vast majority of the projects use flat files for data storage.
Sometimes, the files get a bit large, so we create a set of files…but
basically we’ve been fine without wading into the world of databases.
Recently however, the data involved in our projects are creeping up to
be bigger and bigger. We’re still not anywhere in the “BIG DATA (TM)”
realm, but big enough to warrant exploring options. This blog explores
the options: csv (both from `readr` and `data.table`), RDS, fst, sqlite,
feather, monetDB. One of the takeaways I’ve learned was that there is
not a single right answer. This post will attempt to lay out the options
and summarize the pros and cons.

In a blog post that laid out similar work:
[sqlite-feather-and-fst](https://kbroman.org/blog/2017/04/30/sqlite-feather-and-fst/)
and continued
[here](https://kbroman.org/blog/2017/05/11/reading/writing-biggish-data-revisited/),
Karl Broman discusses his journey from flat files to “big-ish data”.
I’ve taken some of his workflow, added more robust for `fst` and
`monetDB`, and used my own data.

First question: should we set up a shared database?

Shared Database
===============

A database is probably many data scientist’s go-to tool for data storage
and access. There are many database options, and discussing the pros and
cons of each could (and does!) fill a semester-long college course. This
post will not cover those topics.

Our initial question was: when should we even *consider* going through
the process of setting up a shared database? There’s overhead involved,
and our group would either need a spend a fair amount of time getting
over the initial learning-curve or spend a fair amount of our limited
resources on access to skilled data base administrators. None of these
hurdles are insurmountable, but we want to make sure our project and
data needs are worth those investments.

If a single file can be easily passed around to coworkers, and loaded
entirely in memory directly in R, there doesn’t seem to be any reason to
consider a shared database. Maybe the data can be logically chunked into
several files (or 100’s….or 1,000’s) that make collaborating on the same
data easier. What conditions warrant our “R-flat-file-happy” group to
consider a database?

Not being an expert, I asked and got great advice from members of the
[rOpenSci](https://ropensci.org/) community. This is what I learned:

-   “Identify how much CRUD (create, read, uprequested\_date, delete)
    you need to do over time and how complicated your conceptual model
    is. If you need people to be interacting and changing data a shared
    database can help add change tracking and important constraints to
    data inputs. If you have multiple things that interact like sites,
    species, field people, measurement classes, complicated
    requested\_date concepts etc then the db can help.” [Steph
    Locke](https://twitter.com/TheStephLocke)

-   “One thing to consider is whether the data are updated and how, and
    by single or multiple processes.” [Elin
    Waring](https://twitter.com/ElinWaring)

-   “I encourage people towards databases when/if they need to make use
    of all the validation logic you can put into databases. If they just
    need to query, a pattern I like is to keep the data in a text-based
    format like CSV/TSV and load the data into sqlite for querying.”
    [Bryce Mecum](https://twitter.com/brycem)

-   “I suppose another criterion is whether multiple people need to
    access the same data or just a solo user. Concurrency afforded by
    DBs is nice in that regard.” [James
    Balamuta](https://twitter.com/axiomsofxyz)

All great points! In the majority of our data science projects, the
focus is not on creating and maintaining complex data systems…it’s using
large amounts of data. Most if not all of that data already come from
other databases (usually through web services). So…the big benefits for
setting up a shared database for our projects at the moment seems
unnecessary.

Now what?
=========

OK, so we don’t need to buy an Oracle license. We still want to make a
smart choice in the way we save and access the data. We usually have
1-to-many file(s) that we share between a few people. So, we’ll want to
minimize the file size to reduce that transfer time (we have used Google
drive and S3 buckets to store files to share historically). We’d also
like to minimize the time to read and write the files. Maintaining
attributes (like column types) is also ideal.

I will be using a large data frame to test `data.table`,`readr`, `fst`,
`feather`, `sqlite`, and `MonetDBLite`. Late in the game, I tried to
incorporate `sparklyr` into this analysis. `sparklyr` looks and sounds
like an appealing option especially for “really big data”. I was not
able to get my standard examples presented here to work however. Also,
the dependency on a specific version of Java made me nervous (at least,
at the time of writing this blog post). So, while it might be an
attractive solution, there was a bit too much of a learning curve for
the needs of our group.

What is in the data frame is not important to this analysis. Keep in
mind your own personal “biggish” data frame and your hardware might have
different results.

Let’s start by loading the whole file into memory. The columns are a mix
of factors, characters, numerics, requested\_dates, and logicals.

``` r
biggish <- readRDS("test.rds")

nrow(biggish)
```

    ## [1] 3731514

``` r
ncol(biggish)
```

    ## [1] 38

Read, write, and files size
---------------------------

Using the “biggish” data frame, I’m going to write and read the files
completely in memory to start. Because we are often shuffling files
around (one person pushes up to an S3 bucket and another pulls them down
for example), I also want to compare compressed files vs not compressed
when possible.

If you can read in all your data at once, read/write time and file size
should be enough to help you choose your file format. There are many
instances in our “biggish” data projects that we don’t always need nor
want ALL the data ALL the time.

I will also compare how long it takes to pull a subset of the data by
pulling out a requested\_date column, numeric, and string, and only rows
that have specific strings and greater than a threshold. First, I’ll
show individually how to do each of these operations. The end of this
post will contain a table summarizing all the information. It will be
generated using the `microbenchmark` package.

RDS No Compression
------------------

We’ll start with the basic R binary file, the “RDS” file. `saveRDS` has
an argument “compress” that defaults to `TRUE`. Not compressing the
files results in a bigger file size, but quicker read and write times.

``` r
library(dplyr)

file_name <- "test.rds"
# Write:
saveRDS(biggish, file = file_name, compress = FALSE)
# Read:
rds_df <- readRDS(file_name)
```

RDS files must be read entirely in memory so partial read times are not
applicable. However, I will use 2 examples throughout to test the
timings. The examples are deliberately set up to test many `dplyr` basic
verbs and various data types, and other tricky situations like
timezones.

``` r
min_bytes <- 100000
param_cd <- "00060"
group_col <- "statecd"
service <- "dv"

# Partial Read:
partial_data <- readRDS(file_name) %>%
  filter(bytes > !!min_bytes,
         grepl(!!param_cd, parametercds)) %>%
  select(bytes, requested_date, parametercds)

# Group and summarize:
group_summary <- readRDS(file_name) %>%
  filter(service == !!service, 
         !is.na(!!sym(group_col)),
         requested_date > as.POSIXct("2016-10-02 00:00:00", 
                                     tz = "America/New_York")) %>%
  mutate(requested_date = as.Date(requested_date)) %>%
  group_by(.dots = c(group_col, "requested_date")) %>%
  summarize(MB = sum(as.numeric(bytes), na.rm = TRUE)/10^6)
```

Timing in seconds:

| format |  read|  write|  partial|  grouping|
|:-------|-----:|------:|--------:|---------:|
| rds    |  29.7|   28.5|     31.5|      31.1|

RDS Compression
---------------

``` r
file_name <- "test_compressed.rds"
# Write:
saveRDS(biggish, file = file_name, compress = TRUE)
# Read:
rds_compressed_df <- readRDS(file_name)
```

The partial data files will be the same process as “RDS No Compression”.
Timing in seconds:

| format          |  read|  write|  partial|  grouping|
|:----------------|-----:|------:|--------:|---------:|
| rds compression |    27|   51.4|     30.4|      28.5|

readr No Compression
--------------------

``` r
library(readr)
file_name <- "test.csv"
# Write:
write_csv(biggish, path = file_name)
# Read:
readr_df <- read_csv(file_name)
attr(readr_df$requested_date, "tzone") <- "America/New_York"
```

`readr` also must be read entirely in memory so partial read times are
not applicable. If there’s a known, continuous set of rows, you can use
the arguments “skip” and “n\_max” to pull just what you need. However,
that is not flexible enough for most of our needs, so I am not including
that in this evaluation.

``` r
min_bytes <- 100000
param_cd <- "00060"
group_col <- "statecd"
service <- "dv"

# Partial Read:
partial_data_rdr <- read_csv(file_name) %>%
  filter(bytes > !!min_bytes,
         grepl(!!param_cd, parametercds)) %>%
  select(bytes, requested_date, parametercds)

attr(partial_data_rdr$requested_date, "tzone") <- "America/New_York"

# Group and summarize:
group_summary_rdr <- read_csv(file_name) %>%
  filter(service == !!service, 
         !is.na(!!sym(group_col)),
         requested_date > as.POSIXct("2016-10-02 00:00:00")) %>%
  mutate(requested_date = as.Date(requested_date)) %>%
  group_by(.dots = c(group_col, "requested_date")) %>%
  summarize(MB = sum(as.numeric(bytes), na.rm = TRUE)/10^6)
```

Timing in seconds:

| format |  read|  write|  partial|  grouping|
|:-------|-----:|------:|--------:|---------:|
| readr  |  19.2|   66.4|     21.6|      19.7|

readr Compression
-----------------

``` r
library(readr)
file_name <- "test_readr.csv.gz"
# Write:
write_csv(biggish,  path = file_name)
# Read:
readr_compressed_df <- read_csv(file_name)
```

The partial data files will be the same process as “readr No
Compression”.

| format            |  read|  write|  partial|  grouping|
|:------------------|-----:|------:|--------:|---------:|
| readr compression |  27.1|   81.6|     28.6|      28.4|

fread No Compression
--------------------

``` r
library(data.table)
library(fasttime)
file_name <- "test.csv"
# Write:
fwrite(biggish, file = file_name)
# Read:
fread_df <- fread(file_name, 
                  data.table = FALSE, 
                  na.strings = "",) %>%
  mutate(requested_date = fastPOSIXct(requested_date, tz = "America/New_York"))
```

`fread` includes arguments “select”/“drop” to only load specific columns
into memory. This improves the load time if there are many columns that
aren’t needed. If there’s a known, continuous set of rows, you can use
the arguments “skip” and “nrows” to pull just what you need. However,
that is not flexible enough for most of our needs, so I am not including
that in this evaluation.

Also, I am keeping this analysis as a “data.frame” (rather than
“data.table”) because it is the system our group has decided to stick
with.

``` r
min_bytes <- 100000
param_cd <- "00060"
group_col <- "statecd"
service <- "dv"

# Partial Read:
partial_data_fread <- fread(file_name, na.strings = "",
                            data.table = FALSE,
                            select = c("bytes","requested_date","parametercds")) %>%
  filter(bytes > !!min_bytes,
         grepl(!!param_cd, parametercds)) %>%
  mutate(requested_date = fastPOSIXct(requested_date, tz = "America/New_York"))

# Group and summarize:
group_summary_fread <- fread(file_name,na.strings = "",
                             data.table = FALSE,
                             select = c("bytes","requested_date","service",group_col)) %>%
  mutate(requested_date = fastPOSIXct(requested_date, 
                                      tz = "America/New_York")) %>%
  filter(service == !!service, 
         !is.na(!!sym(group_col)),
         requested_date > as.POSIXct("2016-10-02 00:00:00")) %>%
  mutate(requested_date = as.Date(requested_date)) %>%
  group_by(.dots = c(group_col, "requested_date")) %>%
  summarize(MB = sum(as.numeric(bytes), na.rm = TRUE)/10^6)
```

    ## Warning in evalq(sum(as.numeric(bytes), na.rm = TRUE)/10^6, <environment>):
    ## NAs introduced by coercion

    ## Warning in evalq(sum(as.numeric(bytes), na.rm = TRUE)/10^6, <environment>):
    ## NAs introduced by coercion

    ## Warning in evalq(sum(as.numeric(bytes), na.rm = TRUE)/10^6, <environment>):
    ## NAs introduced by coercion

    ## Warning in evalq(sum(as.numeric(bytes), na.rm = TRUE)/10^6, <environment>):
    ## NAs introduced by coercion

| format |  read|  write|  partial|  grouping|
|:-------|-----:|------:|--------:|---------:|
| fread  |  11.2|    2.9|      3.8|       3.4|

Note! I didn’t explore adjusting the `nThread` argument in
`fread`/`fwrite`. I also didn’t include a compressed version of
`fread`/`fwrite`. Our crew is a hog-pog of Windows, Mac, and Linux, and
we try to make our code work on any OS. Many of the solutions for
combining compression with `data.table` functions looked fragile on the
different OS.

The `data.table` package has an open GitHub issues to support
compression in the future. It may be worth updating this script once
that is added.

feather Compression
-------------------

``` r
library(feather)
file_name <- "test.feather"
# Write:
write_feather(biggish, path = file_name)
# Read:
feather_df <- read_feather(file_name)
```

`read_feather` includes an argument “columns” to only load specific
columns into memory. This improves the load time if there are many
columns that aren’t needed.

``` r
min_bytes <- 100000
param_cd <- "00060"
group_col <- "statecd"
service <- "dv"

# Partial Read:
partial_data_feather <- read_feather(file_name, 
        columns = c("bytes","requested_date","parametercds")) %>%
  filter(bytes > !!min_bytes,
         grepl(!!param_cd, parametercds))

# Group and summarize:
group_summary_feather <- read_feather(file_name, 
                      columns = c("bytes","requested_date","service",group_col)) %>%
  filter(service == !!service, 
         !is.na(!!sym(group_col)),
         requested_date > as.POSIXct("2016-10-02 00:00:00", 
       tz = "America/New_York")) %>%
  mutate(requested_date = as.Date(requested_date)) %>%
  group_by(.dots = c(group_col, "requested_date")) %>%
  summarize(MB = sum(as.numeric(bytes), na.rm = TRUE)/10^6)
```

Timing in seconds:

| format  |  read|  write|  partial|  grouping|
|:--------|-----:|------:|--------:|---------:|
| feather |   5.6|      4|      1.7|       0.4|

For the same reason as `fread`, I didn’t try compressing the `feather`
format. Both `data.table` and `feather` have open GitHub issues to
support compression in the future. It may be worth updating this script
once those features are added.

fst No Compression
------------------

``` r
library(fst)
file_name <- "test.fst"

# Write:
write_fst(biggish, path = file_name, compress = 0)
# Read:
fst_df <- read_fst(file_name)
```

``` r
min_bytes <- 100000
param_cd <- "00060"
group_col <- "statecd"
service <- "dv"

# Partial Read:
partial_data_fst <- read_fst(file_name, 
        columns = c("bytes","requested_date","parametercds")) %>%
  filter(bytes > !!min_bytes,
         grepl(!!param_cd, parametercds)) 

# Group and summarize:
group_summary_fst <- read_fst(file_name, 
                      columns = c("bytes","requested_date","service",group_col)) %>%
  filter(service == !!service, 
         !is.na(!!sym(group_col)),
         requested_date > as.POSIXct("2016-10-02 00:00:00", 
       tz = "America/New_York")) %>%
  mutate(requested_date = as.Date(requested_date)) %>%
  group_by(.dots = c(group_col, "requested_date")) %>%
  summarize(MB = sum(as.numeric(bytes), na.rm = TRUE)/10^6)
```

Timing in seconds:

| format |  read|  write|  partial|  grouping|
|:-------|-----:|------:|--------:|---------:|
| fst    |   4.2|    6.5|      1.7|       0.3|

fst Compression
---------------

``` r
library(fst)
file_name <- "test_compressed.fst"

# Write:
write_fst(biggish, path = file_name, compress = 100)
# Read:
fst_df <- read_fst(file_name)
```

The partial data files will be the same process as “fst No Compression”.
Timing in seconds:

| format          |  read|  write|  partial|  grouping|
|:----------------|-----:|------:|--------:|---------:|
| fst compression |     6|   12.4|      1.8|       0.3|

SQLite
------

SQLite does not have a storage class set aside for storing dates and/or
times.

``` r
library(RSQLite)

file_name <- "test.sqlite"

sqldb <- dbConnect(SQLite(), dbname=file_name)

# Write:
dbWriteTable(sqldb,name =  "test", biggish,
             row.names=FALSE, overwrite=TRUE,
             append=FALSE, field.types=NULL)
# Read:
sqlite_df <- tbl(sqldb,"test") %>% 
  collect() %>%
  mutate(requested_date = as.POSIXct(requested_date, 
                                     tz = "America/New_York",
                                     origin = "1970-01-01"))
```

Things to notice here, you can’t just use `grep`.

``` r
min_bytes <- 100000
param_cd <- "00060"
group_col <- "statecd"
service <- "dv"

# Partial Read:
partial_data_sqlite <- tbl(sqldb,"test") %>% 
  select(bytes, requested_date , parametercds) %>%
  filter(bytes > !!min_bytes,
         parametercds %like% '%00060%') %>%
  collect() %>%
  mutate(requested_date = as.POSIXct(requested_date, 
                                     tz = "America/New_York",
                                     origin = "1970-01-01"))

# Group and summarize:
filter_time <- as.numeric(as.POSIXct("2016-10-02 00:00:00", tz = "America/New_York"))

group_summary_sqlite <- tbl(sqldb,"test") %>%
  select(bytes, requested_date, service, !!group_col) %>%
  filter(service == !!service, 
         !is.na(!!sym(group_col)),
         requested_date > !! filter_time) %>%
  mutate(requested_date = strftime('%Y-%m-%d', datetime(requested_date, 'unixepoch'))) %>%
  group_by(!!sym(group_col), requested_date) %>%
  summarize(MB = sum(bytes, na.rm = TRUE)/10^6) %>%
  collect()

dbDisconnect(sqldb)
```

Timing in seconds:

| format |  read|  write|  partial|  grouping|
|:-------|-----:|------:|--------:|---------:|
| sqlite |  27.5|   34.8|      1.5|       1.6|

It is important to note that this is the first “partial” and “grouping”
solution that is completely done outside of R. So when you are getting
data that pushes the limits (or passes the limits) of what you can load
directly into R, this is the first basic solution.

MonetDB
-------

``` r
library(MonetDBLite)
library(DBI)

file_name <- "test.monet"

con <- dbConnect(MonetDBLite(), dbname = file_name)

# Write:
dbWriteTable(con, name =  "test", biggish,
             row.names=FALSE, overwrite=TRUE,
             append=FALSE, field.types=NULL)
# Read:
monet_df <- dbReadTable(con, "test")
attr(monet_df$requested_date, "tzone") <- "America/New_York"
```

``` r
min_bytes <- 100000
param_cd <- "00060"
group_col <- "statecd"
service <- "dv"

# Partial Read:
partial_data_monet <- tbl(con,"test") %>% 
  select(bytes, requested_date , parametercds) %>%
  filter(bytes > !!min_bytes,
         parametercds %like% '%00060%') %>%
  collect() 

attr(partial_data_monet$requested_date, "tzone") <- "America/New_York"

# Group and summarize:
# MonetDB needs the time in UTC, formatted exactly as:
# 'YYYY-mm-dd HH:MM:SS', hence the last "format" commend:
filter_time <- as.POSIXct("2016-10-02 00:00:00", tz = "America/New_York")
attr(filter_time, "tzone") <- "UTC"
filter_time <- format(filter_time)

group_summary_monet <- tbl(con,"test") %>%
  select(bytes, requested_date, service, !!group_col) %>%
  filter(service == !!service, 
         !is.na(!!sym(group_col)), 
         requested_date > !! filter_time) %>% 
  mutate(requested_date = str_to_date(timestamp_to_str(requested_date, '%Y-%m-%d'),'%Y-%m-%d')) %>%
  group_by(!!sym(group_col), requested_date) %>%
  summarize(MB = sum(bytes, na.rm = TRUE)/10^6) %>%
  collect()

dbDisconnect(con, shutdown=TRUE)
```

Timing in seconds:

| format  |  read|  write|  partial|  grouping|
|:--------|-----:|------:|--------:|---------:|
| MonetDB |     3|   30.2|      1.6|       1.3|

Again, it is important to note that the “partial” and “grouping”
solutions are completely done outside of R. So when you are getting data
that pushes the limits (or passes the limits) of what you can load
directly into R, this is another good solution. There also appears to be
a lot more flexibility in using date/times directly in MonetDB compared
to SQLite.

Comparison
==========

<table class="table table-hover" style="margin-left: auto; margin-right: auto;">
<thead>
<tr>
<th style="text-align:center;">
File Format
</th>
<th style="text-align:center;">
Read
</th>
<th style="text-align:center;">
Write
</th>
<th style="text-align:center;">
Partial
</th>
<th style="text-align:center;">
Grouping
</th>
<th style="text-align:center;">
File Size (MB)
</th>
</tr>
</thead>
<tbody>
<tr>
<td style="text-align:center;">
rds
</td>
<td style="text-align:center;">
29.7
</td>
<td style="text-align:center;">
28.5
</td>
<td style="text-align:center;">
<span style="     color: red;">31.5</span>
</td>
<td style="text-align:center;">
<span style="     color: red;">31.1</span>
</td>
<td style="text-align:center;">
1280.6
</td>
</tr>
<tr>
<td style="text-align:center;">
rds compression
</td>
<td style="text-align:center;">
27.0
</td>
<td style="text-align:center;">
51.4
</td>
<td style="text-align:center;">
<span style="     color: red;">30.4</span>
</td>
<td style="text-align:center;">
<span style="     color: red;">28.5</span>
</td>
<td style="text-align:center;">
55.4
</td>
</tr>
<tr>
<td style="text-align:center;">
readr
</td>
<td style="text-align:center;">
19.2
</td>
<td style="text-align:center;">
66.4
</td>
<td style="text-align:center;">
<span style="     color: red;">21.6</span>
</td>
<td style="text-align:center;">
<span style="     color: red;">19.7</span>
</td>
<td style="text-align:center;">
703.9
</td>
</tr>
<tr>
<td style="text-align:center;">
readr compression
</td>
<td style="text-align:center;">
27.1
</td>
<td style="text-align:center;">
81.6
</td>
<td style="text-align:center;">
<span style="     color: red;">28.6</span>
</td>
<td style="text-align:center;">
<span style="     color: red;">28.4</span>
</td>
<td style="text-align:center;">
65.7
</td>
</tr>
<tr>
<td style="text-align:center;">
fread
</td>
<td style="text-align:center;">
11.2
</td>
<td style="text-align:center;">
2.9
</td>
<td style="text-align:center;">
<span style=" font-weight: bold;    color: blue;">3.8</span>
</td>
<td style="text-align:center;">
<span style=" font-weight: bold;    color: blue;">3.4</span>
</td>
<td style="text-align:center;">
503.7
</td>
</tr>
<tr>
<td style="text-align:center;">
feather
</td>
<td style="text-align:center;">
5.6
</td>
<td style="text-align:center;">
4.0
</td>
<td style="text-align:center;">
<span style=" font-weight: bold;    color: blue;">1.7</span>
</td>
<td style="text-align:center;">
<span style=" font-weight: bold;    color: blue;">0.4</span>
</td>
<td style="text-align:center;">
818.4
</td>
</tr>
<tr>
<td style="text-align:center;">
fst
</td>
<td style="text-align:center;">
4.2
</td>
<td style="text-align:center;">
6.5
</td>
<td style="text-align:center;">
<span style=" font-weight: bold;    color: blue;">1.7</span>
</td>
<td style="text-align:center;">
<span style=" font-weight: bold;    color: blue;">0.3</span>
</td>
<td style="text-align:center;">
988.6
</td>
</tr>
<tr>
<td style="text-align:center;">
fst compression
</td>
<td style="text-align:center;">
6.0
</td>
<td style="text-align:center;">
12.4
</td>
<td style="text-align:center;">
<span style=" font-weight: bold;    color: blue;">1.8</span>
</td>
<td style="text-align:center;">
<span style=" font-weight: bold;    color: blue;">0.3</span>
</td>
<td style="text-align:center;">
121.8
</td>
</tr>
<tr>
<td style="text-align:center;">
sqlite
</td>
<td style="text-align:center;">
27.5
</td>
<td style="text-align:center;">
34.8
</td>
<td style="text-align:center;">
<span style=" font-weight: bold;    color: green;">1.5</span>
</td>
<td style="text-align:center;">
<span style=" font-weight: bold;    color: green;">1.6</span>
</td>
<td style="text-align:center;">
464.2
</td>
</tr>
<tr>
<td style="text-align:center;">
MonetDB
</td>
<td style="text-align:center;">
3.0
</td>
<td style="text-align:center;">
30.2
</td>
<td style="text-align:center;">
<span style=" font-weight: bold;    color: green;">1.6</span>
</td>
<td style="text-align:center;">
<span style=" font-weight: bold;    color: green;">1.3</span>
</td>
<td style="text-align:center;">
719.5
</td>
</tr>
</tbody>
</table>
Not that `sqlite` and `MonetDB` are the only formats here that allow
careful filtering and calculate summaries without loading the whole data
set. So if our “pretty big data” gets “really big”, those will formats
will rise to the top.

If you can read in all the rows without crashing R, `fread`, `feather`,
and `fst` are fast!

Another consideration, who are your collaborators? If everyone’s using R
exclusively, this table on it’s own is a fine way to judge what format
to pick. If your collaborators are half R, half Python…you might favor
`feather` since that format works well in both systems.

Collecting this information has been a very useful activity for helping
me understand the various options for saving and reading data in R. I
tried to pick somewhat complex queries to test out the capabilities, but
I acknowledge I’m not an expert on especially the database formats. I
would be very happy to hear more efficient ways to perform these
analyses.

Disclaimer
==========

Any use of trade, firm, or product names is for descriptive purposes
only and does not imply endorsement by the U.S. Government.
