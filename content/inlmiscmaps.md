---
author: Jason C Fisher
date: 2018-07-17
slug: inlmiscmaps
title: Maps with inlmisc
type: post
categories: Data Science
image: static/inlmiscmaps/inlmiscmaps.png
author_github: jfisher-usgs
author_staff: jason-c-fisher
author_email: <jfisher@usgs.gov>
tags:
  - R
  - inlmisc
  - leaflet
keywords:
  - R
  - inlmisc
  - leaflet
description: Using the R-package inlmisc to create static and dynamic maps.
---



## Introduction

This document gives a brief introduction to making static and dynamic maps using
[inlmisc](https://CRAN.R-project.org/package=inlmisc),
an R package developed by researchers at the United States Geological Survey (USGS)
Idaho National Laboratory (INL)
[Project Office](https://www.usgs.gov/centers/id-water/science/idaho-national-laboratory-project-office).
Included with **inlmisc** is a collection of functions for creating high-level graphics,
such as graphs, maps, and cross sections.
All graphics attempt to adhere to the formatting standards for illustrations in USGS publications.
You can install the package from [CRAN](https://CRAN.R-project.org/) using the command:


```r
if (system.file(package = "inlmisc", lib.loc = .libPaths()) == "")
  utils::install.packages("inlmisc", dependencies = TRUE)
```

## Static Maps

Let's begin by transforming the now famous
[meuse](https://CRAN.R-project.org/web/packages/gstat/vignettes/gstat.pdf) data set,
introduced by Burrough and McDonnell (1998), into a static map.
First define a georeferenced raster layer object from
the point data of top soil zinc concentrations.


```r
data(meuse, meuse.grid, package = "sp")
sp::coordinates(meuse.grid) <- ~x+y
sp::proj4string(meuse.grid) <- sp::CRS("+init=epsg:28992")
sp::gridded(meuse.grid) <- TRUE
meuse.grid <- raster::raster(meuse.grid, layer = "soil")
model <- gstat::gstat(id = "zinc", formula = zinc~1, locations = ~x+y, data = meuse)
r <- raster::interpolate(meuse.grid, model)
r <- raster::mask(r, meuse.grid)
```

Next, plot a map from the gridded data and include a scale bar and vertical legend.


```r
Pal <- function(n) inlmisc::GetColors(n, stops = c(0.3, 0.9))  # color palette
breaks <- seq(0, 2000, by = 200)  # break points used to partition colors
credit <- paste("Data collected in a flood plain of the river Meuse,",
                "near the village of Stein (Netherlands),",
                "\nand iterpolated on a grid with 40-meter by 40-meter spacing",
                "using inverse distance weighting.")
inlmisc::PlotMap(r, breaks = breaks, pal = Pal, dms.tick = TRUE, bg.lines = TRUE,
                 contour.lines = list(col = "#1F1F1F"), credit = credit,
                 draw.key = FALSE, simplify = 0)
inlmisc::AddScaleBar(unit = c("KILOMETER", "MILES"), conv.fact = c(0.001, 0.0006214),
                     loc = "bottomright", inset = c(0.12, 0.05))
inlmisc::AddGradientLegend(breaks, Pal, at = breaks,
                           title = "Topsoil zinc\nconcentration\n(ppm)",
                           loc = "topleft", inset = c(0.05, 0.1),
                           strip.dim = c(2, 20))
```

<img src='/static/inlmiscmaps/plot_meuse-1.png'/ title='Static map of meuse data set.' alt='Static map of meuse data set.' class=''/>

For the next example, transform Auckland's Maunga Whau volcano data set into a static map.
First define a georeferenced raster layer object for the volcano's topographic information.


```r
m <- t(datasets::volcano)[61:1, ]
x <- seq(from = 6478705, length.out = 87, by = 10)
y <- seq(from = 2667405, length.out = 61, by = 10)
r <- raster::raster(m, xmn = min(x), xmx = max(x), ymn = min(y), ymx = max(y),
                    crs = "+init=epsg:27200")
```

Next, plot a map from the gridded data and include a color key beneath the plot region.


```r
credit <- paste("Digitized from a topographic map by Ross Ihaka",
                "on a grid with 10-meter by 10-meter spacing.")
explanation <- "Elevation on Auckland's Maunga Whau volcano, in meters."
inlmisc::PlotMap(r, xlim = range(x), ylim = range(y), extend.z = TRUE,
                 pal = terrain.colors, explanation = explanation, credit = credit,
                 shade = list(alpha = 0.3), contour.lines = list(col = "#1F1F1F"),
                 useRaster = TRUE)
```

<img src='/static/inlmiscmaps/plot_volcano-1.png'/ title='Static map of valcano data set.' alt='Static map of valcano data set.' class=''/>

One thing you may have noticed is the white space drawn above and below the raster image.
White space that results from plotting to a graphics device
using the default device dimensions for the canvas of the plotting window (typically 7 inches by 7 inches).
Because margin sizes are fixed, the width and height of the plotting region are
dependent on the device dimensions---not the data.

If a publication-quality figure is what you're after,
never use the default values for the device dimensions.
Instead, have the `PlotMap` function return the dimensions that are optimized for the data.
To do so, specify an output file using the `file` argument.
The file's extension determines the format type (only PDF and PNG are supported).
The maximum device dimensions are constrained using the `max.dev.dim` argument---a
vector of length 2 giving the maximum width and height for the graphics device in picas.
Where 1 pica is equal to 1/6 of an inch, 4.23 millimeters, or 12 points.
Suggested dimensions for single-column, double-column, and sidetitle figures are
`c(21, 56)`, `c(43, 56)` (default), and `c(56, 43)`, respectively.


```r
out <- inlmisc::PlotMap(r, xlim = range(x), ylim = range(y), extend.z = TRUE,
                        pal = terrain.colors, explanation = explanation,
                        credit = credit, shade = list(alpha = 0.3),
                        contour.lines = list(col = "#1F1F1F"),
                        useRaster = TRUE, file = tempfile(fileext = ".png"))
din <- round(out$din, digits = 2)
cat(sprintf("width = %s, height = %s", din[1], din[2]))
```

```
## width = 7.16, height = 5.52
```

Replotting the map using the returned device dimensions results in
a figure that is void of extraneous white space.

<img src='/static/inlmiscmaps/plot_volcano_din-1.png'/ title='Static map of valcano data set with improved device dimensions.' alt='Static map of valcano data set with improved device dimensions.' class=''/>

## Dynamic Maps

A dynamic map is an interactive display of geographic information that is powered by the web.
Interactive panning and zooming allows for an explorative view of a map area.
Use the `CreateWebMap` function to make a dynamic map object.


```r
map <- inlmisc::CreateWebMap()
```

This function is based on [Leaflet for R](https://rstudio.github.io/leaflet/) with
base maps provided by [The National Map](https://viewer.nationalmap.gov/) (TNM) services
and displayed in a WGS 84 / Pseudo-Mercator (EPSG:3857) coordinate reference system.
Data from TNM is free and in the public domain,
and available from the USGS, National Geospatial Program.

As an example, transform U.S. city location data into a dynamic map.
First define a georeferenced spatial points object for U.S. cities.


```r
city <- rgdal::readOGR(system.file("extdata/city.geojson", package = "inlmisc")[1])
```

The city data was originally extracted from the Census Bureau's
[MAF/TIGER](https://www.census.gov/geo/maps-data/data/tiger.html) database.
Next, add a layer of markers to call out cities on the map,
and buttons that may be used to zoom to the initial map extent,
and toggle marker clusters on and off.
Also add a search element to locate, and move to, a marker.


```r
opt <- leaflet::markerClusterOptions(showCoverageOnHover = FALSE)
map <- leaflet::addMarkers(map, label = ~name, popup = ~name, clusterOptions = opt,
                           clusterId = "cluster", group = "marker", data = city)
map <- inlmisc::AddHomeButton(map)
map <- inlmisc::AddClusterButton(map, clusterId = "cluster")
map <- inlmisc::AddSearchButton(map, group = "marker", zoom = 15,
                                textPlaceholder = "Search city names...")
```

Print the dynamic map object to display it in your web browser.


```r
print(map)
```



<iframe seamless src="/static/inlmiscmaps/map/index.html" width="100%" height="500" frameborder="0"></iframe>

Some users have reported that base maps do not render correctly in the
[RStudio](https://www.rstudio.com/) viewer.
Until RStudio can address this issue, the following workaround is provided.


```r
options(viewer = NULL); print(map)
```

Let's take this example a step further and embed the dynamic map within a standalone HTML document.
You can share this HTML document just like you would a PDF document.
Making it suitable for an appendix in an USGS Scientific Investigation Report.

Before getting started, a few more pieces of software are required.
If not already installed, download and install the universal document converter
[pandoc](https://pandoc.org/).
The pandoc installer is robust and does not require administrative privileges.
[R Markdown](https://rmarkdown.rstudio.com/) is also required and used to
render a [R Markdown document](https://bookdown.org/yihui/rmarkdown/html-document.html)
to a HTML document.
The R-markdown document is a text file that has the extension '.Rmd'
and contains R-code chunks and
[markdown](https://pandoc.org/MANUAL.html#pandocs-markdown) text.
You can install the R-package [rmarkdown](https://CRAN.R-project.org/package=rmarkdown)
using the command:


```r
if (system("pandoc -v") == "") warning("pandoc not available")
if (!inlmisc::IsPackageInstalled("rmarkdown"))
  utils::install.packages("rmarkdown")
```

Let's also install the R-package
[leaflet.extras](https://CRAN.R-project.org/package=leaflet.extras)
to provide extra functionality for the
[leaflet](https://CRAN.R-project.org/package=leaflet) R package.


```r
if (!inlmisc::IsPackageInstalled("leaflet.extras"))
  utils::install.packages("leaflet.extras")
```

Next, create a R Markdown document in your working directory.
The document contains a block of R-code with instructions for
creating the dynamic map and adding a fullscreen control button.


```r
file <- file.path(getwd(), "example.Rmd")
cat("# Example", "", "```{r out.width = '100%', fig.height = 6}",
    "map <- inlmisc::CreateWebMap(options = leaflet::leafletOptions(minZoom = 2))",
    "map <- leaflet.extras::addFullscreenControl(map, pseudoFullscreen = TRUE)",
    "map", "```", file = file, sep = "\n")
file.show(file)
```

Note that the R Markdown document is typically created in a text editor.
Finally, render the document and view the results in a web browser.


```r
rmarkdown::render(file, "html_document", quiet = TRUE)
utils::browseURL(sprintf("file://%s", file.path(getwd(), "example.html")))
```

## References Cited

Burrough, P.A., and McDonnell, R.A., 1998,
Principles of Geographical Information Systems (2d ed.):
Oxford, N.Y., Oxford University Press, 35 p.

## Reproducibility

R-session information for content in this document is as follows:


```
## - Session info ----------------------------------------------------------
##  setting  value                       
##  version  R version 3.5.1 (2018-07-02)
##  os       Windows 10 x64              
##  system   x86_64, mingw32             
##  ui       Rgui                        
##  language (EN)                        
##  collate  English_United States.1252  
##  ctype    English_United States.1252  
##  tz       America/Los_Angeles         
##  date     2018-11-08                  
## 
## - Packages --------------------------------------------------------------
##  package        * version date       lib source        
##  assertthat       0.2.0   2017-04-11 [1] CRAN (R 3.5.1)
##  backports        1.1.2   2017-12-13 [1] CRAN (R 3.5.0)
##  base64enc        0.1-3   2015-07-28 [1] CRAN (R 3.5.0)
##  callr            3.0.0   2018-08-24 [1] CRAN (R 3.5.1)
##  checkmate        1.8.5   2017-10-24 [1] CRAN (R 3.5.1)
##  cli              1.0.1   2018-09-25 [1] CRAN (R 3.5.1)
##  codetools        0.2-15  2016-10-05 [1] CRAN (R 3.5.1)
##  colorspace       1.3-2   2016-12-14 [1] CRAN (R 3.5.1)
##  crayon           1.3.4   2017-09-16 [1] CRAN (R 3.5.1)
##  crosstalk        1.0.0   2016-12-21 [1] CRAN (R 3.5.1)
##  data.table       1.11.8  2018-09-30 [1] CRAN (R 3.5.1)
##  debugme          1.1.0   2017-10-22 [1] CRAN (R 3.5.1)
##  desc             1.2.0   2018-05-01 [1] CRAN (R 3.5.1)
##  devtools         2.0.1   2018-10-26 [1] CRAN (R 3.5.1)
##  digest           0.6.18  2018-10-10 [1] CRAN (R 3.5.1)
##  evaluate         0.12    2018-10-09 [1] CRAN (R 3.5.1)
##  FNN              1.1.2.1 2018-08-10 [1] CRAN (R 3.5.1)
##  fs               1.2.6   2018-08-23 [1] CRAN (R 3.5.1)
##  glue             1.3.0   2018-07-17 [1] CRAN (R 3.5.1)
##  gstat            1.1-6   2018-04-02 [1] CRAN (R 3.5.1)
##  htmltools        0.3.6   2017-04-28 [1] CRAN (R 3.5.1)
##  htmlwidgets      1.3     2018-09-30 [1] CRAN (R 3.5.1)
##  httpuv           1.4.5   2018-07-19 [1] CRAN (R 3.5.1)
##  igraph           1.2.2   2018-07-27 [1] CRAN (R 3.5.1)
##  inlmisc          0.4.4   2018-11-08 [1] CRAN (R 3.5.1)
##  intervals        0.15.1  2015-08-27 [1] CRAN (R 3.5.0)
##  jsonlite         1.5     2017-06-01 [1] CRAN (R 3.5.1)
##  knitr            1.20    2018-02-20 [1] CRAN (R 3.5.1)
##  later            0.7.5   2018-09-18 [1] CRAN (R 3.5.1)
##  lattice          0.20-38 2018-11-04 [1] CRAN (R 3.5.1)
##  leaflet          2.0.2   2018-08-27 [1] CRAN (R 3.5.1)
##  leaflet.extras   1.0.0   2018-04-21 [1] CRAN (R 3.5.1)
##  magrittr         1.5     2014-11-22 [1] CRAN (R 3.5.1)
##  memoise          1.1.0   2017-04-21 [1] CRAN (R 3.5.1)
##  mime             0.6     2018-10-05 [1] CRAN (R 3.5.1)
##  munsell          0.5.0   2018-06-12 [1] CRAN (R 3.5.1)
##  pkgbuild         1.0.2   2018-10-16 [1] CRAN (R 3.5.1)
##  pkgconfig        2.0.2   2018-08-16 [1] CRAN (R 3.5.1)
##  pkgload          1.0.2   2018-10-29 [1] CRAN (R 3.5.1)
##  prettyunits      1.0.2   2015-07-13 [1] CRAN (R 3.5.1)
##  processx         3.2.0   2018-08-16 [1] CRAN (R 3.5.1)
##  promises         1.0.1   2018-04-13 [1] CRAN (R 3.5.1)
##  ps               1.2.1   2018-11-06 [1] CRAN (R 3.5.1)
##  R6               2.3.0   2018-10-04 [1] CRAN (R 3.5.1)
##  raster           2.8-4   2018-11-03 [1] CRAN (R 3.5.1)
##  Rcpp             1.0.0   2018-11-07 [1] CRAN (R 3.5.1)
##  remotes          2.0.2   2018-10-30 [1] CRAN (R 3.5.1)
##  rgdal            1.3-6   2018-10-16 [1] CRAN (R 3.5.1)
##  rgeos            0.4-1   2018-10-19 [1] CRAN (R 3.5.1)
##  rlang            0.3.0.1 2018-10-25 [1] CRAN (R 3.5.1)
##  rmarkdown        1.10    2018-06-11 [1] CRAN (R 3.5.1)
##  rprojroot        1.3-2   2018-01-03 [1] CRAN (R 3.5.1)
##  scales           1.0.0   2018-08-09 [1] CRAN (R 3.5.1)
##  sessioninfo      1.1.1   2018-11-05 [1] CRAN (R 3.5.1)
##  shiny            1.2.0   2018-11-02 [1] CRAN (R 3.5.1)
##  sp               1.3-1   2018-06-05 [1] CRAN (R 3.5.1)
##  spacetime        1.2-2   2018-07-17 [1] CRAN (R 3.5.1)
##  stringi          1.2.4   2018-07-20 [1] CRAN (R 3.5.1)
##  stringr          1.3.1   2018-05-10 [1] CRAN (R 3.5.1)
##  testthat         2.0.1   2018-10-13 [1] CRAN (R 3.5.1)
##  usethis          1.4.0   2018-08-14 [1] CRAN (R 3.5.1)
##  withr            2.1.2   2018-03-15 [1] CRAN (R 3.5.1)
##  xtable           1.8-3   2018-08-29 [1] CRAN (R 3.5.1)
##  xts              0.11-2  2018-11-05 [1] CRAN (R 3.5.1)
##  yaml             2.2.0   2018-07-25 [1] CRAN (R 3.5.1)
##  zoo              1.8-4   2018-09-19 [1] CRAN (R 3.5.1)
## 
## [1] C:/Users/jfisher/Tools/R/R-3.5.1/library
```
