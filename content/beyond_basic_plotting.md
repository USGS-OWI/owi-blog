---
title: "Beyond Basic R - Plotting with ggplot2 and Multiple Plots in One Figure"
author: "Lindsay R Carr"
date: '2018-08-09'
author_staff: lindsay-r-carr
author_twitter: LindsayRCarr
categories: Data Science
author_github: lindsaycarr
description: Resources for plotting, plus short examples for using ggplot2 for common
  use-cases and adding USGS style.
draft: yes
image: static/beyond-basic-plotting/cowplotmulti-1.png
keywords:
- R
- Beyond Basic R
- ggplot2
slug: beyond-basic-plotting
tags:
- R
- Beyond Basic R
author_email: <lcarr@usgs.gov>
type: post
---
R can create almost any plot imaginable and as with most things in R if you don’t know where to start, try Google. The Introduction to R curriculum summarizes some of the most used plots, but cannot begin to expose people to the breadth of plot options that exist.There are existing resources that are great references for plotting in R:

In base R:

-   [Breakdown of how to create a plot](https://www.r-bloggers.com/how-to-plot-a-graph-in-r/) from R-bloggers
-   [Another blog breaking down basic plotting](https://flowingdata.com/2012/12/17/getting-started-with-charts-in-r/) from FlowingData
-   [Basic plots](https://www.cyclismo.org/tutorial/R/plotting.html) (histograms, boxplots, scatter plots, QQ plots) from University of Georgia
-   [Intermediate plots](https://www.cyclismo.org/tutorial/R/intermediatePlotting.html) (error bars, density plots, bar charts, multiple windows, saving to a file, etc) from University of Georgia

In ggplot2:

-   [ggplot2 homepage](http://ggplot2.tidyverse.org/)
-   [ggplot2 video tutorial](https://www.youtube.com/watch?v=rsG-GgR0aEY)
-   [Website with everything you want to know about ggplot2](http://r-statistics.co/Complete-Ggplot2-Tutorial-Part1-With-R-Code.html) by Selva Prabhakaran
-   [R graphics cookbook site](http://www.cookbook-r.com/Graphs/)
-   [ggplot2 cheatsheet](https://www.rstudio.com/wp-content/uploads/2015/03/ggplot2-cheatsheet.pdf)
-   [ggplot2 reference guide](http://ggplot2.tidyverse.org/reference/)

In the [Introduction to R](https://owi.usgs.gov/R/training-curriculum/intro-curriculum) class, we have switched to teaching ggplot2 because it works nicely with other tidyverse packages (dplyr, tidyr), and can create interesting and powerful graphics with little code. While `ggplot2` has many useful features, this blog post will explore how to create figures with multiple `ggplot2` plots.

You may have already heard of ways to put multiple R plots into a single figure - specifying `mfrow` or `mfcol` arguments to `par`, `split.screen`, and `layout` are all ways to do this. However, there are other methods to do this that are optimized for `ggplot2` plots.

Multiple plots in one figure using ggplot2 and facets
-----------------------------------------------------

When you are creating multiple plots and they share axes, you should consider using facet functions from ggplot2 (`facet_grid`, `facet_wrap`). You write your `ggplot2` code as if you were putting all of the data onto one plot, and then you use one of the faceting functions to specify how to slice up the graph.

Let's start by considering a set of graphs with a common x axis. You have a data.frame with four columns: Date, site\_no, parameter, and value. You want three different plots in the same figure - a timeseries for each of the parameters with different colored symbols for the different sites. Sounds like a lot, but facets can make this very simple. First, setup your ggplot code as if you aren't faceting.

``` r
library(dataRetrieval)
library(dplyr) # for `rename` & `select`
library(tidyr) # for `gather`
library(ggplot2)

# Get the data
wi_daily_wq <- readNWISdv(siteNumbers = c("05430175", "05427880", "05427927"),
                          parameterCd = c("00060", "00530", "00631"),
                          startDate = "2017-08-01", endDate = "2017-08-31")

# Clean up data to have human-readable names + move data into long format
wi_daily_wq <- renameNWISColumns(wi_daily_wq) %>% 
  rename(TSS = `X_00530`, InorganicN = `X_00631`) %>% 
  select(-ends_with("_cd")) %>% 
  gather(key = "parameter", value = "value", -site_no, -Date)

# Setup plot without facets
p <- ggplot(data = wi_daily_wq, aes(x = Date, y = value)) + 
  geom_point(aes(color = site_no)) + 
  theme_bw()

# Now, we can look at the plot and see how it looks before we facet
# Obviously, the scales are off because we are plotting flow with concentrations
p
```

<img src='/static/beyond-basic-plotting/nofacetplot-1.png'/ title='ggplot2 setup before faceting' alt='Basic ggplot2 timeseries with 3 parameters represented in one: inorganic N, TSS, and flow.' class=''/>

Now, we know that we can't keep these different parameters on the same plot. We could have written code to filter the data frame to the appropriate values and make a plot for each of them, but we can also take advantage of `facet_grid`. Since the resulting three plots that we want will all share an x axis (Date), we can imagine slicing up the figure in the vertical direction so that the x axis remains in-tact but we end up with three different y axes. We can do this using `facet_grid` and a formula syntax, `y ~ x`. So, if you want to divide the figure along the y axis, you put variable in the data that you want to use to decide which plot data goes into as the first entry in the formula. You can use a `.` if you do not want to divide the plot in the other direction.

``` r
# Add vertical facets, aka divide the plot up vertically since they share an x axis
p + facet_grid(parameter ~ .)
```

<img src='/static/beyond-basic-plotting/verticalfacetplot-1.png'/ title='ggplot2 basic vertical facet' alt='Basic ggplot2 timeseries with inorganic N, TSS, and flow represented in three different facets along the y axis.' class=''/>

The result is a figure divided along the y axis based on the unique values of the `parameter` column in the data.frame. So, we have three plots in one figure. They still all share the same axes, which works for the x axis but not for the y axes. We can change that by letting the y axes scale freely to the data that appears just on that facet. Add the argument `scales` to `facet_grid` and specify that they should be "free" rather than the default "fixed".

``` r
# Add vertical facets, but scale only the y axes freely
p + facet_grid(parameter ~ ., scales = "free_y")
```

<img src='/static/beyond-basic-plotting/verticalfacetplotfreescale-1.png'/ title='ggplot2 with freely scaled, vertical facets' alt='Basic ggplot2 timeseries with inorganic N, TSS, and flow represented in three individually scaled facets along the y axis.' class=''/>

From here, there might be a few things you want to change about how it's labelling the facets. We would probably want the y axis labels to say the parameter and units on the left side. So, we can adjust how the facets are labeled and styled to become our y axis labels.

``` r
p + facet_grid(parameter ~ ., scales = "free_y",
               switch = "y", # flip the facet labels along the y axis from the right side to the left
               labeller = as_labeller( # redefine the text that shows up for the facets
                 c(Flow = "Flow, cfs", InorganicN = "Inorganic N, mg/L", TSS = "TSS, mg/L"))) +
  ylab(NULL) + # remove the word "values"
  theme(strip.background = element_blank(), # remove the background
        strip.placement = "outside") # put labels to the left of the axis text
```

<img src='/static/beyond-basic-plotting/verticalfacetplotfixedlabels-1.png'/ title='ggplot2 with facet labels as the y axis labels' alt='Basic ggplot2 timeseries with inorganic N, TSS, and flow represented in three individually scaled facets along the y axis, and appropriately labeled axes.' class=''/>

There are still other things you can do with facets, such as using `space = "free"`. The [Cookbook for R facet examples](http://www.cookbook-r.com/Graphs/Facets_(ggplot2)/) have even more to explore!

Using `cowplot` to create multiple plots in one figure
------------------------------------------------------

When you are creating multiple plots and they do not share axes or do not fit into the facet framework, you could use the packages `cowplot` or `patchwork` (very new!), or the `grid.arrange` function from `gridExtra`. In this blog post, we will show how to use `cowplot`, but you can explore the features of `patchwork` [here](https://github.com/thomasp85/patchwork).

The package called `cowplot` has nice wrapper functions for ggplot2 plots to have shared legends, put plots into a grid, annotate plots, and more. Below is some code that shows how to use some of these helpful `cowplot` functions to create a figure that has three plots and a shared title.

``` r
library(dataRetrieval)
library(dplyr) # for `rename`
library(tidyr) # for `gather`
library(ggplot2)
library(cowplot)

# Get the data
yahara_daily_wq <- readNWISdv(siteNumbers = "05430175", 
                          parameterCd = c("00060", "00530", "00631"),
                          startDate = "2017-08-01", endDate = "2017-08-31")

# Clean up data to have human-readable names
yahara_daily_wq <- renameNWISColumns(yahara_daily_wq)
yahara_daily_wq <- rename(yahara_daily_wq, TSS = `X_00530`, InorganicN = `X_00631`)

# Create the three different plots
flow_timeseries <- ggplot(yahara_daily_wq, aes(x=Date, y=Flow)) + 
  geom_point() + theme_bw()

yahara_daily_wq_long <- gather(yahara_daily_wq, Nutrient, Nutrient_va, TSS, InorganicN)
nutrient_boxplot <- ggplot(yahara_daily_wq_long, aes(x=Nutrient, y=Nutrient_va)) +
  geom_boxplot() + theme_bw()

tss_flow_plot <- ggplot(yahara_daily_wq, aes(x=Flow, y=TSS)) + 
  geom_point() + theme_bw()

# Create Flow timeseries plot that spans the grid by making one plot_grid
#   and then nest it inside of a second. Also, include a title at the top 
#   for the whole figure. 
title <- ggdraw() + draw_label("Conditions for site 05430175", fontface='bold')
bottom_row <- plot_grid(nutrient_boxplot, tss_flow_plot, ncol = 2, labels = "AUTO")
plot_grid(title, bottom_row, flow_timeseries, nrow = 3, labels = c("", "", "C"),
          rel_heights = c(0.2, 1, 1))
```

<img src='/static/beyond-basic-plotting/cowplotmulti-1.png'/ title='Multi-plot figure generated using cowplot.' alt='Three plots in one figure: boxplot of inorganic N & TSS, TSS vs flow, and hydrograph.' class=''/>
