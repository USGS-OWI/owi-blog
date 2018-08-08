---
author: Laura DeCicco
date: 2018-08-06
slug: boxplots
draft: True
title: Exploring ggplot2 boxplots - Defining limits and adjusting style
type: post
categories: Data Science
image: static/boxplots/visualizeBox-1.png
author_twitter: DeCiccoDonk
author_github: ldecicco-usgs
author_gs: jXd0feEAAAAJ
 
author_staff: laura-decicco
author_email: <ldecicco@usgs.gov>

tags: 
  - R
 
 
description: Identifying boxplot limits in ggplot2.
keywords:
  - R
 
 
  - boxplot
  - ggplot2
---
Boxplots are often used to show data distributions, and `ggplot2` is often used to visualize data. A question that comes up is what exactly do the box plots represent? The `ggplot2` box plots follow standard Tukey representations, and there are many references of this online and in standard statistical text books. The base R function to calculate the box plot limits is `boxplot.stats`. The help file for this function is very informative, but it's often non-R users asking what exactly the plot means. Therefore, this blog post breaks down the calculations into (hopefully!) easy-to-follow chunks of code for you to make your own box plot legend if necessary. Some additional goals here are to create boxplots that come *close* to USGS style. Features in this blog post take advantage of enhancements to `ggplot2` in version 3.0.0 or later.

First, let's get some data that might be typically plotted in a USGS report using a boxplot. Here we'll use chloride data (parameter code "00940") measured at a USGS station on the Fox River in Green Bay, WI (station ID "04085139"). We'll use the package `dataRetrieval` to get the data (see [this tutorial](https://owi.usgs.gov/R/dataRetrieval.html) for more information on `dataRetrieval`), and plot a simple boxplot by month using `ggplot2`:

``` r
library(dataRetrieval)
library(ggplot2)

chloride <- readNWISqw("04085139", "00940")
chloride$month <- month.abb[as.numeric(format(chloride$sample_dt, "%m"))]
chloride$month <- factor(chloride$month, labels = month.abb)

cl_name <- attr(chloride, "variableInfo")[["parameter_nm"]]
site_name <- attr(chloride, "siteInfo")[["station_nm"]]

ggplot(data = chloride, 
       aes(x = month, y = result_va)) +
  geom_boxplot() +
  xlab("Month") +
  ylab(cl_name) +
  labs(title = site_name)
```

<img src='/static/boxplots/getChoride-1.png'/ title='TODO' alt='TODO' />

Is that graph great? YES! And for presentations and/or journal publications, that graph might be appropriate. However, for an official USGS report, USGS employees need to get the graphics approved to assure they follow specific style guidelines. The approving officer would probably come back from the review with the following comments:

<table>
<colgroup>
<col width="51%" />
<col width="48%" />
</colgroup>
<thead>
<tr class="header">
<th>Reviewer's Comments</th>
<th>Adjustment in <code>ggplot2</code></th>
</tr>
</thead>
<tbody>
<tr class="odd">
<td>Remove background color, grid lines</td>
<td>Adjust theme</td>
</tr>
<tr class="even">
<td>Add horizontal bars to the upper and lower whiskers</td>
<td>Add <code>stat_boxplot</code></td>
</tr>
<tr class="odd">
<td>Have tick marks go inside the plot</td>
<td>Adjust theme</td>
</tr>
<tr class="even">
<td>Tick marks should be on both sides of the y axis</td>
<td>Add <code>sec.axis</code> to <code>scale_y_continuous</code></td>
</tr>
<tr class="odd">
<td>Remove tick marks from discrete data</td>
<td>Adjust theme</td>
</tr>
<tr class="even">
<td>y-axis needs to start exactly at 0</td>
<td>Add <code>expand_limits</code></td>
</tr>
<tr class="odd">
<td>y-axis labels need to be shown at 0 and at the upper scale</td>
<td>Add <code>breaks</code> and <code>limits</code> to <code>scale_y_continuous</code></td>
</tr>
<tr class="even">
<td>Add very specific legend</td>
<td>Create function <code>ggplot_box_legend</code></td>
</tr>
<tr class="odd">
<td>Add the number of observations above each boxplot</td>
<td>Add custom <code>stat_summary</code></td>
</tr>
<tr class="even">
<td>Change text size</td>
<td>Adjust <code>geom_text</code> defaults</td>
</tr>
<tr class="odd">
<td>Change font (we'll use &quot;serif&quot; in this blog, although that is not the official USGS font)</td>
<td>Adjust <code>geom_text</code> defaults</td>
</tr>
</tbody>
</table>

As you can see, it will not be as simple as creating a single custom ggplot theme to comply with the requirements. However, we can string together ggplot commands in a list for easy re-use. This blog is *not* going to get you perfect compliance with the USGS standards, but it will get much closer. Also, while these style adjustments are tailored to USGS requirements, the process described here may be useful for other graphic guidelines as well.

So, let's skip to the exciting conclusion and use some code that will be described later (`boxplot_framework` and `ggplot_box_legend`) to create the same plot, now closer to those USGS style requirements:

``` r
library(cowplot)

legend_plot <- ggplot_box_legend()

chloride_plot <- ggplot(data = chloride, 
       aes(x = month, y = result_va)) +
  boxplot_framework(upper_limit = 70) + 
  xlab("Month") +
  ylab(cl_name) +
  labs(title = site_name)

plot_grid(chloride_plot, 
          legend_plot,
          nrow = 1, rel_widths = c(.6,.4))
```

<img src='/static/boxplots/chlorideWithLegend-1.png'/ title='Chloride by month styled.' alt='TODO' />

As can be seen in the code chunk, we are now using a function `ggplot_box_legend` to make a legend, `boxplot_framework` to accommodate all of the style requirements, and the `cowplot` package to plot them together.

`boxplot_framework`: Styling the boxplot
========================================

Now, let's get our style requirements figured out. First, we can set some basic plot elements for a theme. We can start with the `theme_bw` and add to that. Here we remove the grid, set the size of the title, bring the y ticks inside the plotting area, and remove the x ticks:

``` r
theme_USGS_box <- function(base_family = "serif", ...){
  theme_bw(base_family = base_family, ...) +
  theme(
    panel.grid = element_blank(),
    plot.title = element_text(size = 8),
    axis.ticks.length = unit(-0.05, "in"),
    axis.text.y = element_text(margin=unit(c(0.3,0.3,0.3,0.3), "cm")), 
    axis.text.x = element_text(margin=unit(c(0.3,0.3,0.3,0.3), "cm")),
    axis.ticks.x = element_blank(),
    aspect.ratio = 1,
    legend.background = element_rect(color = "black", fill = "white")
  )
}
```

Next, we can change the defaults of the geom\_text to a smaller size and font.

``` r
update_geom_defaults("text", 
                   list(size = 3, 
                        family = "serif"))
```

We also need to figure out what other `ggplot2` elements need to be added. The basic ggplot code for the chloride plot would be:

``` r
n_fun <- function(x){
  return(data.frame(y = 0.95*70,
                    label = length(x)))
}

ggplot(data = chloride, 
       aes(x = month, y = result_va)) +
  stat_boxplot(geom ='errorbar', width = 0.6) +
  geom_boxplot(width = 0.6, fill = "lightgrey") +
  stat_summary(fun.data = n_fun, geom = "text", hjust = 0.5) +
  expand_limits(y = 0) +
  theme_USGS_box() +
  scale_y_continuous(sec.axis = dup_axis(label = NULL, 
                                         name = NULL),
                     expand = expand_scale(mult = c(0, 0)),
                     breaks = pretty(c(0,70), n = 5), 
                     limits = c(0,70))
```

Let's look at a few other common boxplots to see if there are other ggplot2 elements that would be useful in a common `boxplot_framework` function.

Logrithmic boxplot
------------------

For another example, we might need to make a boxplot with a logarithm scale. This data is for phosphorus measurements on the Pheasant Branch Creek in Middleton, WI.

``` r
site <- "05427948"
pCode <- "00665"

phos_data <- readNWISqw(site, pCode)
phos_data$month <- month.abb[as.numeric(format(phos_data$sample_dt, "%m"))]
phos_data$month <- factor(phos_data$month, labels = month.abb)

phos_name <- attr(phos_data, "variableInfo")[["parameter_nm"]]
site_name <- attr(phos_data, "siteInfo")[["station_nm"]]

n_fun <- function(x){
    return(data.frame(y = 0.95*log10(50),
                      label = length(x)))
}

phos_plot <- ggplot(data = phos_data, 
       aes(x = month, y = result_va)) +
  stat_boxplot(geom ='errorbar', width = 0.6) +
  geom_boxplot(width = 0.6, fill = "lightgrey") +
  stat_summary(fun.data = n_fun, geom = "text", hjust = 0.5) +
  theme_USGS_box() +
  scale_y_log10(limits = c(0.01, 50),
                  expand = expand_scale(mult = c(0, 0))) +
  annotation_logticks(sides = c("rl")) +
  xlab("Month") +
  ylab(phos_name) +
  labs(title = site_name) 

phos_plot
```

<img src='/static/boxplots/phosDistribution-1.png'/ title='Phosphorus distribution by month.' alt='TODO' />

Grouped boxplots
----------------

We might also want to make grouped boxplots. In ggplot, it's pretty easy to add a "fill" to the `aes` argument. Here we'll plot temperature distributions at 4 USGS stations. We'll group the measurements by a "daytime" and "nighttime" factor. Temperature might be a parameter that would not be required to start at 0.

``` r
library(dplyr)

temp_q_data <- readNWISuv(siteNumbers = c("04026561", "04063700",
                                          "04082400", "05427927"),
                          parameterCd = '00010', 
                          startDate = "2018-06-01", 
                          endDate = "2018-06-03")

temperature_name <- attr(temp_q_data, "variableInfo") %>%
  filter(variableCode == "00010") %>% pull(variableName)

station_nms <- attr(temp_q_data, "siteInfo")

temp_q_data <- renameNWISColumns(temp_q_data)

# add an hour of day to create groups (daytime or nighttime)
temp_q_data <- temp_q_data %>% 
  mutate(site_no = factor(site_no)) %>% # grouping var should be a factor
  mutate(hourOfDay = as.numeric(format(dateTime, "%H"))) %>% 
  mutate(timeOfDay = case_when(
    hourOfDay < 20 & hourOfDay > 6 ~ "daytime",
    TRUE ~ "nighttime" # catchall for anything that doesn't fit above
  ))

n_fun <- function(x){
  return(data.frame(y = 0.95*30,
                    label = length(x)))
}

temperature_plot <- ggplot(data = temp_q_data, 
       aes(x=site_no, y=Wtemp_Inst, fill=timeOfDay)) +
  stat_boxplot(geom ='errorbar', width = 0.6) +
  geom_boxplot(width = 0.6) +
  stat_summary(fun.data = n_fun, geom = "text", 
               aes(group=timeOfDay),
               hjust = 0.5, position = position_dodge(0.6)) +
  expand_limits(y = 0) +
  scale_y_continuous(sec.axis = dup_axis(label = NULL, 
                                         name = NULL),
                     expand = expand_scale(mult = c(0, 0)),
                     breaks = pretty(c(10,30), n = 5), 
                     limits = c(10,30)) +
  theme_USGS_box() +
  xlab("Station ID") +
  ylab(temperature_name) +
  scale_fill_discrete(name = "EXPLANATION") +
  theme(legend.position = c(0.175, 0.78))

temperature_plot
```

<img src='/static/boxplots/groupPlot1-1.png'/ title='TODO' alt='TODO' />

The parameter name that comes back from `dataRetrieval` could use some formatting. The following function can fix that for both `ggplot2` and base R graphics:

``` r
unescape_html <- function(str){
  fancy_chars <- regmatches(str, gregexpr("&#\\d{3};",str)) 

  unescaped <- xml2::xml_text(xml2::read_html(paste0("<x>", fancy_chars, "</x>")))

  fancy_chars <- gsub(pattern = "&#\\d{3};", 
                      replacement = unescaped, x = str)

  fancy_chars <- gsub("Â","", fancy_chars)
  return(fancy_chars)
}
```

We'll use this function in the next section.

Framework function
------------------

Finally, we can bring all of those elements together into a single list for `ggplot2` to use. While we're at it, we can create a function that is flexible for both linear and logarithmic scales, as well as grouped boxplots. It's a bit clunky because you need to specify the upper and lower limits of the plot. There might be a slicker way to do that, but for now, this works:

``` r
boxplot_framework <- function(upper_limit, 
                              family_font = "serif",
                              lower_limit = 0, 
                              logY = FALSE, 
                              fill_var = NA,
                              fill = "lightgrey", width = 0.6){
  
  update_geom_defaults("text", 
                     list(size = 3, 
                          family = family_font))
  
  n_fun <- function(x, lY = logY){
    return(data.frame(y = ifelse(logY, 0.95*log10(upper_limit), 0.95*upper_limit),
                      label = length(x)))
  }
  
  if(!is.na(fill_var)){
      basic_elements <- list(stat_boxplot(geom ='errorbar', width = width),
                            geom_boxplot(width = width),
                            stat_summary(fun.data = n_fun, 
                                         geom = "text", 
                                         position = position_dodge(width),
                                         hjust =0.5,
                                         aes_string(group=fill_var)),
                            expand_limits(y = lower_limit),
                            theme_USGS_box())
  } else {
      basic_elements <- list(stat_boxplot(geom ='errorbar', width = width),
                            geom_boxplot(width = width, fill = fill),
                            stat_summary(fun.data = n_fun, 
                                         geom = "text", hjust =0.5),
                            expand_limits(y = lower_limit),
                            theme_USGS_box())
  }
  
  if(logY){
    return(c(basic_elements,
              scale_y_log10(limits = c(lower_limit, upper_limit),
                  expand = expand_scale(mult = c(0, 0))),
              annotation_logticks(sides = c("rl"))))      
  } else {
    return(c(basic_elements,
              scale_y_continuous(sec.axis = dup_axis(label = NULL, 
                                                     name = NULL),
                                 expand = expand_scale(mult = c(0, 0)),
                                 breaks = pretty(c(lower_limit,upper_limit), n = 5), 
                                 limits = c(lower_limit,upper_limit))))    
  }
}
```

Examples with our framework
===========================

Let's see if it works! Let's take the last set of examples using the framework. I'm also going to use the 'cowplot' package to print them all together. I'll also include the `ggplot_box_legend` which will be described in the next section.

``` r
legend_plot <- ggplot_box_legend()

chloride_plot <- ggplot(data = chloride, 
       aes(x = month, y = result_va)) +
  boxplot_framework(upper_limit = 70) + 
  xlab("Month") +
  ylab(cl_name) +
  labs(title = site_name)

phos_plot <- ggplot(data = phos_data, 
       aes(x = month, y = result_va)) +
  boxplot_framework(upper_limit = 50,
                    lower_limit = 0.01,
                    logY = TRUE) +
  xlab("Month") +
  ylab("Phosphorus in milligraphs per liter") +
  labs(title = site_name) 

temperature_plot <- ggplot(data = temp_q_data, 
       aes(x=site_no, y=Wtemp_Inst, fill=timeOfDay)) +
  boxplot_framework(upper_limit = 30, 
                    lower_limit = 10, 
                    fill_var = "timeOfDay") + 
  xlab("Station ID") +
  ylab(unescape_html(temperature_name)) +
  labs(title = "Daytime vs Nighttime Temperature Distribution") +
  scale_fill_discrete(name = "EXPLANATION") +
  theme(legend.position = c(0.225, 0.78))

plot_grid(chloride_plot, 
          phos_plot,
          temperature_plot,
          legend_plot,
          nrow = 2)
```

<img src='/static/boxplots/comboGraph-1.png'/ title='TODO' alt='TODO' />

`ggplot_box_legend`: What is a boxplot?
=======================================

A non-trivial requirement to the USGS boxplot style guidelines is to make a detailed, prescribed legend. In this section we'll first verify that `ggplot2` boxplots use the same definitions for the lines and dots, and then we'll make a function that creates the prescribed legend. To start, let's set up random data using the R function `sample` and then create a function to calculate each value.

``` r
set.seed(100)

sample_df <- data.frame(parameter = "test",
                        values = sample(500))

# Extend the top whisker a bit:
sample_df$values[1:100] <- 701:800
# Make sure there's only 1 lower outlier:
sample_df$values[1] <- -350
```

Next, we'll create a function that calculates the necessary values for the boxplots:

``` r
ggplot2_boxplot <- function(x){
  
  quartiles <- as.numeric(quantile(x, 
                                   probs = c(0.25, 0.5, 0.75)))
  
  names(quartiles) <- c("25th percentile", 
                        "50th percentile\n(median)",
                        "75th percentile")
  
  IQR <- diff(quartiles[c(1,3)])

  upper_whisker <- max(x[x < (quartiles[3] + 1.5 * IQR)])
  lower_whisker <- min(x[x > (quartiles[1] - 1.5 * IQR)])
    
  upper_dots <- x[x > (quartiles[3] + 1.5*IQR)]
  lower_dots <- x[x < (quartiles[1] - 1.5*IQR)]

  return(list("quartiles" = quartiles,
              "25th percentile" = as.numeric(quartiles[1]),
              "50th percentile\n(median)" = as.numeric(quartiles[2]),
              "75th percentile" = as.numeric(quartiles[3]),
              "IQR" = IQR,
              "upper_whisker" = upper_whisker,
              "lower_whisker" = lower_whisker,
              "upper_dots" = upper_dots,
              "lower_dots" = lower_dots))
}

ggplot_output <- ggplot2_boxplot(sample_df$values)
```

What are those calculations?

-   Quartiles (25, 50, 75 percentiles), 50% is the median
-   Interquartile range is the difference between the 75th and 25th percentiles
-   The upper whisker is the maximum value of the data that is within 1.5 times the interquartile range over the 75th percentile.
-   The lower whisker is the minimum value of the data that is within 1.5 times the interquartile range under the 25th percentile.
-   Outlier values are considered any values over 1.5 times the interquartile range over the 75th percentile or any values under 1.5 times the interquartile range under the 25th percentile.

Let's check that the output matches `boxplot.stats`:

``` r
# Using base R:
base_R_output <- boxplot.stats(sample_df$values)

# Some checks:

# Outliers:
all(c(ggplot_output[["upper_dots"]], 
      ggplot_output[["lowerdots"]]) %in%
    c(base_R_output[["out"]]))
```

    ## [1] TRUE

``` r
# whiskers:
ggplot_output[["upper_whisker"]] == base_R_output[["stats"]][5]
```

    ## [1] TRUE

``` r
ggplot_output[["lower_whisker"]] == base_R_output[["stats"]][1]
```

    ## [1] TRUE

Boxplot Legend
--------------

Let's plot that information, and while we're at it, we can make the function used in the first plot. There is a *lot* of `ggplot2` code to digest here. Most of it is style adjustments to approximate the USGS style guidelines for a boxplot legend.

``` r
ggplot_box_legend <- function(family = "serif"){
  set.seed(100)

  sample_df <- data.frame(parameter = "test",
                        values = sample(500))

  # Extend the top whisker a bit:
  sample_df$values[1:100] <- 701:800
  # Make sure there's only 1 lower outlier:
  sample_df$values[1] <- -350
  
  ggplot_output <- ggplot2_boxplot(sample_df$values)
  
  update_geom_defaults("text", 
                     list(size = 3, 
                          hjust = 0,
                          family = family))
  
  update_geom_defaults("label", 
                     list(size = 3, 
                          hjust = 0,
                          family = family))
  
  explain_plot <- ggplot() +
    stat_boxplot(data = sample_df,
                 aes(x = parameter, y=values),
                 geom ='errorbar', width = 0.3) +
    geom_boxplot(data = sample_df,
                 aes(x = parameter, y=values), 
                 width = 0.3, fill = "lightgrey") +
    geom_text(aes(x = 1, y = 950, label = "500"), hjust = 0.5) +
    geom_text(aes(x = 1.17, y = 950,
                  label = "Number of values"),
              fontface = "bold", vjust = 0.4) +
    theme_minimal(base_size = 5, base_family = family) +
    geom_segment(aes(x = 2.3, xend = 2.3, 
                     y = ggplot_output[["25th percentile"]], 
                     yend = ggplot_output[["75th percentile"]])) +
    geom_segment(aes(x = 1.2, xend = 2.3, 
                     y = ggplot_output[["25th percentile"]], 
                     yend = ggplot_output[["25th percentile"]])) +
    geom_segment(aes(x = 1.2, xend = 2.3, 
                     y = ggplot_output[["75th percentile"]], 
                     yend = ggplot_output[["75th percentile"]])) +
    geom_text(aes(x = 2.4, y = ggplot_output[["50th percentile\n(median)"]]), 
              label = "Interquartile\nrange", fontface = "bold",
              vjust = 0.4) +
    geom_text(aes(x = c(1.17,1.17), 
                  y = c(ggplot_output[["upper_whisker"]],
                        ggplot_output[["lower_whisker"]]), 
                  label = c("Largest value within 1.5 times\ninterquartile range above\n75th percentile",
                            "Smallest value within 1.5 times\ninterquartile range below\n25th percentile")),
                  fontface = "bold", vjust = 0.9) +
    geom_text(aes(x = c(1.17), 
                  y =  ggplot_output[["lower_dots"]], 
                  label = "Outside value"), 
              vjust = 0.5, fontface = "bold") +
    geom_text(aes(x = c(1.9), 
                  y =  ggplot_output[["lower_dots"]], 
                  label = "-Value is >1.5 times and"), 
              vjust = 0.5) +
    geom_text(aes(x = 1.17, 
                  y = ggplot_output[["lower_dots"]], 
                  label = "<3 times the interquartile range\nbeyond either end of the box"), 
              vjust = 1.5) +
    geom_label(aes(x = 1.17, y = ggplot_output[["quartiles"]], 
                  label = names(ggplot_output[["quartiles"]])),
              vjust = c(0.4,0.85,0.4), 
              fill = "white", label.size = 0) +
    ylab("") + xlab("") +
    theme(axis.text = element_blank(),
          axis.ticks = element_blank(),
          panel.grid = element_blank(),
          plot.title = element_text(hjust = 0.5, size = 10)) +
    coord_cartesian(xlim = c(1.4,3.1), ylim = c(-600, 900)) +
    labs(title = "EXPLANATION")

  return(explain_plot) 
  
}

ggplot_box_legend()
```

<img src='/static/boxplots/visualizeBox-1.png'/ title='ggplot2 box plot with explanation.' alt='ggplot2 box plot with explanation.' />

Bring it together
=================

What's nice about leaving this in the world of `ggplot2` is that it is still possible to use other `ggplot2` elements on the plot. For example, let's add the detection limits as horizontal lines to the phosphorous graph:

``` r
DLs <- unique(as.numeric(phos_data$rpt_lev_va))
DLs <- DLs[!is.na(DLs)]

phos_plot_with_DL <- phos_plot +
  geom_hline(linetype = "dashed",
             yintercept = DLs)

explain_plot_DL <- ggplot_box_legend() +
  geom_segment(aes(y = -650, yend = -650,
               x = 0.6, xend = 1.6),
           linetype="dashed") +
  geom_text(aes(y = -650, x = 1.8, label = "Detection Limit"))

plot_grid(phos_plot_with_DL, 
          explain_plot_DL,
          nrow = 1, rel_widths = c(.6,.4))
```

<img src='/static/boxplots/picsWithDL-1.png'/ title='TODO' alt='TODO' />

I hoped you like my "deep dive" into `ggplot2` boxplots. Many of the techniques here can be used to modify other `ggplot2` plots.
