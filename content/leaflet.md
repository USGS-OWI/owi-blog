---
author: Laura DeCicco
date: 2016-08-23
slug: leaflet
type: post
title: Using Leaflet and htmlwidgets in a Hugo blog post
categories: Data Science
image: static/leaflet/screenshot.png
tags: 
  - R
  - dataRetrieval
 
---

<a href="mailto:ldecicco@usgs.gov"><i class="fa fa-envelope-square fa-2x" aria-hidden="true"></i></a>
<a href="https://twitter.com/DeCiccoDonk"><i class="fa fa-twitter-square fa-2x" aria-hidden="true"></i></a>
<a href="https://github.com/ldecicco-usgs"><i class="fa fa-github-square fa-2x" aria-hidden="true"></i></a>
<a href="https://scholar.google.com/citations?hl=en&user=jXd0feEAAAAJ"><i class="ai ai-google-scholar-square ai-2x" aria-hidden="true"></i></a>
<a href="https://www.researchgate.net/profile/Laura_De_Cicco"><i class="ai ai-researchgate-square ai-2x" aria-hidden="true"></i></a>

We are excited to use many of the JavaScript data visualizations in R using the [`htmlwidgets`](http://www.htmlwidgets.org/) package in future blog posts. Having decided on using [Hugo](https://gohugo.io/), one of our first tasks was to figure out a fairly straightforward way to incorporate these widgets. This post describes the basic process to get a basic [`leaflet`](https://cran.r-project.org/package=leaflet) map in our Hugo-generated blog post.

In this example, we are looking for phosphorus measured throughout Wisconsin with the [`dataRetrieval`](https://cran.r-project.org/package=dataRetrieval) package. Using [`dplyr`](https://cran.r-project.org/package=dplyr), we filter the data to sites that have records longer than 15 years, and more than 50 measurements.

``` r
library(dataRetrieval)
pCode <- c("00665")
phos.wi <- readNWISdata(stateCd="WI", parameterCd=pCode,
                     service="site", seriesCatalogOutput=TRUE)

library(dplyr)
phos.wi <- filter(phos.wi, parm_cd %in% pCode) %>%
            filter(count_nu > 50) %>%
            mutate(period = as.Date(end_date) - as.Date(begin_date)) %>%
            filter(period > 15*365)
```

Plot the sites on a map:

``` r
library(leaflet)
leafMap <- leaflet(data=phos.wi) %>% 
  addProviderTiles("CartoDB.Positron") %>%
  addCircleMarkers(~dec_long_va,~dec_lat_va,
                   color = "red", radius=3, stroke=FALSE,
                   fillOpacity = 0.8, opacity = 0.8,
                   popup=~station_nm)
```

Next, we use the `htmlwidgets` package to save a self-contained html file. The following code could be generally hid from the reader using `echo=FALSE`.

``` r
library(htmlwidgets)
library(htmltools)

currentWD <- getwd()
dir.create("static/leaflet", showWarnings = FALSE)
setwd("static/leaflet")
saveWidget(leafMap, "leafMap.html")
setwd(currentWD)
```

The html that was saved with the `saveWidget` function can be called with the `iframe` html tag.

``` r
<iframe seamless src="/static/leaflet/leafMap/index.html" width="100%" height="500"></iframe>
```

<iframe seamless src="/static/leaflet/leafMap/index.html" width="100%" height="500">
</iframe>
When building the site, Hugo converts the "leafMap.html" to "leafMap/index.html".

One issue for our Hugo theme was that the created widget page is included in the overall blog index. The was fixed by adding a line to the overall index.html layout page to only lists pages with dates above Jan. 1, 1970 (so really, any legitimate date):

``` r
{{ if ge $value.Date.Unix 0 }}
  <div class="col-sm-4">
    {{ .Render "grid" }}
  </div>
{{ end }}
```

A screenshot of the map was taken to use for the thumbnail in the blog index.

Questions
=========

Please direct any questions or comments on `dataRetrieval` to: <https://github.com/USGS-R/dataRetrieval/issues>
