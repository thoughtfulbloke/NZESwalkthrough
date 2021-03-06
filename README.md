---
title: "New Zealand Election Study (NZES) data in R"
author: "David Hood"
date: "12/8/2018"
output:
  html_document:
    keep_md: yes
  word_document: default
---



## New Zealand Election Study example R code

This is some example code for working with the publicly available New Zealand Election Study data, available at http://www.nzes.org/exec/show/data

For this tutorial, I am assuming you are comfortable enough downloading data and setting file paths for reading the data in, and you just want a nudge on understanding the structure of the data and how to put it to use.

## Tutorial packages


```r
library(foreign)
# all the rest of the libraries are for analysis after reading in the data
library(dplyr)
library(ggplot2)
library(knitr)
```

Note: The recent (as of writing) foreign package had some enhancements in the underlying libraries that make it much easier to read in these SPSS files compared to older versions. I am using version 0.8.69.

I am also loading dplyr and ggplot2, for use in showing some example analysis and knitr for making a table. These packages do not need to be loaded for reading in the data.

## Reading in the data

For each data file, I am using read.spss() from the "foreign" package, wrapped up in a suppressWarnings() function. This is because some of the entries in the SPSS file are labelled, and it will generate a warning message for each variable that does not have a full set of labels. Using suppressWarnings() stops many, many, many, many lines informing your of this, while the variables read in fine and are stored as factors with a mix of textual labels (either based on label of the entry in SPSS or the numeric value in the case of unlabeled entries).

As well as data file, from the data I am creating a meta file, containing the variable names and descriptions of the data. This is to make it easier to check the subject of each column in the data.



```r
sav14 <- suppressWarnings(read.spss("NZES2014GeneralReleaseApril16.sav",
                    to.data.frame = TRUE, add.undeclared.levels = "sort"))
meta14 <- data.frame(varnames = attributes(sav14)$names,
                     descriptions = attributes(sav14)$variable.labels,
                     stringsAsFactors = FALSE)
sav11 <- suppressWarnings(read.spss("NZES2011_Jun_18_rel.sav",
                    to.data.frame = TRUE, add.undeclared.levels = "sort"))
meta11 <- data.frame(varnames = attributes(sav11)$names,
                     descriptions = attributes(sav11)$variable.labels,
                     stringsAsFactors = FALSE)
sav08 <- suppressWarnings(read.spss("2008NZES_Release.sav",
                    to.data.frame = TRUE, add.undeclared.levels = "sort"))
meta08 <- data.frame(varnames = attributes(sav08)$names,
                     descriptions = attributes(sav08)$variable.labels,
                     stringsAsFactors = FALSE)
sav05 <- suppressWarnings(read.spss("NZES_Release_05.sav",
                    to.data.frame = TRUE, add.undeclared.levels = "sort"))
meta05 <- data.frame(varnames = attributes(sav05)$names,
                     descriptions = attributes(sav05)$variable.labels,
                     stringsAsFactors = FALSE)
```

Clearly doing representative statistics for the country would mean understanding the survey weightings etc by reading the documentation at the website, but as a short example of using the data:

## Example One

For those in the survey in 2014 who gave age and which party (of the four main parties) they voted for, what is the age distribution by party.

* check the metadata to find the appropriate columns
* reduce the data to the relevant entries


```r
sav14 %>%
  # entries with age that voted for a party of interest
  filter(!is.na(dage),
         dpartyvote %in% c("National", "Labour", "Green", "NZ First")) %>%
#make a graph
  ggplot(aes(x=dage, colour=dpartyvote)) + geom_density() +
  theme_minimal() + ggtitle("Age distribution of voters by party, 2014 NZES") +
  scale_colour_manual(values= c("darkred", "blue", "green", "black")) +
  xlab("Age") + ylab("proportion of voters")
```

![](README_files/figure-html/unnamed-chunk-1-1.png)<!-- -->

## Example Two

Doing some exploratory graph making people are placing National at 0, the most left possible on the left right political spectrum. It would be highly unusual to truly consider National to the left of Labour, so it might be worth considering if this is a kind of answer that is specific to a group, such as an age group. To tackle this:

* check the metadata to find the appropriate columns
* change the left/right scale factor (ordinal categorical) columns from a factor where the entries include "Don't Know" to the original 0-10 point scale.
* figure out if the entry for National was lower (leftwards) of the entry for Labour

Examining one of the left/right variables to get a sense of the factor levels:


```r
sav14 %>%
  mutate(label=dlablr) %>%
  group_by(label) %>%
  summarise(ordinal=as.numeric(dlablr)[1]) %>% kable()
```



label         ordinal
-----------  --------
Left                1
1                   2
2                   3
3                   4
4                   5
Centre              6
6                   7
7                   8
8                   9
9                  10
Right              11
Don't know         12
NA                 NA

In theory it would be possible to make a conversion on the basis of the numeric ordinal component of the factor, however if a party lacks responses for a particular option the numeric levels would not know about the missing ordinal values, so it is best to do the conversion based on the text


```r
sav14 %>%
  # relevant columns to select known from reading metadata
  # convert to numeric
  mutate(
    numeric_lablr = case_when(
      dlablr == "Left" ~ 0,
      dlablr == "Centre" ~ 5,
      dlablr == "Right" ~ 10,
      TRUE ~ as.numeric(as.character(dlablr))),
    numeric_natlr = case_when(
      dnatlr == "Left" ~ 0,
      dnatlr == "Centre" ~ 5,
      dnatlr == "Right" ~ 10,
      TRUE ~ as.numeric(as.character(dnatlr))),
    placement = ifelse(numeric_natlr < numeric_lablr,
                       "N left of L", "not N left of L")) %>%
  # for those who placed both National and Labour on Left/Right
  filter(!is.na(placement)) %>%
  # make a graph
  ggplot(aes(x=dage, fill=placement)) + geom_histogram(bins=20) +
  ggtitle("NZES 2014: number of responses placing National left of Labour") + xlab("Age of respondent")
```

![](README_files/figure-html/unnamed-chunk-3-1.png)<!-- -->

## Example Three

To show working with more than one of the data sets at once, I decide to examine how the Green voter age distribution has changed at each election. This involves:

* Pick out the Green entries with age data for each study data set
* Standardize column names
* Combine the individual data sets into one


```r
# make one dataset of data prepared on the fly
bind_rows(sav14 %>%
  filter(!is.na(dage), dpartyvote == "Green") %>% select(age = dage) %>%
  mutate(election=2014),
  sav11%>%
  filter(!is.na(jage), jpartyvote == "Green") %>% select(age = jage) %>%
  mutate(election=2011),
  sav08%>%
  filter(!is.na(ZAGE), zvt08p == "Green") %>% select(age = ZAGE) %>%
  mutate(election=2008),
  sav05%>%
  filter(!is.na(YAGE), yvot05p == "Green") %>% select(age = YAGE) %>%
  mutate(election=2005)) %>% 
  # graph it
  ggplot(aes(x=age, colour=as.factor(election), group=election)) + geom_density() +
  theme_minimal() + ggtitle("Green voters by age, 2005- 2014 NZES") +
  scale_colour_viridis_d() +
  xlab("Age") + ylab("proportion of voters")
```

![](README_files/figure-html/unnamed-chunk-4-1.png)<!-- -->


