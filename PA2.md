# Reproducible Research: Peer Assessment 2

## Sypnopsis
The goal of this data analysis exercise is to answer the following questions:

1. Across the United States, which types of weather events are most harmful with respect to population health?

2. Across the United States, which types of weather events have the greatest economic consequences?

## Data
The data used in this analysis is available below:

* [Storm Data](https://d396qusza40orc.cloudfront.net/repdata%2Fdata%2FStormData.csv.bz2)

There is also some documentation of the database available. Here you will find how some of the variables are constructed/defined:

* National Weather Service [Storm Data Documentation](https://d396qusza40orc.cloudfront.net/repdata%2Fpeer2_doc%2Fpd01016005curr.pdf)

* National Climatic Data Center Storm Events [FAQ](https://d396qusza40orc.cloudfront.net/repdata%2Fpeer2_doc%2FNCDC%20Storm%20Events-FAQ%20Page.pdf)

The events in the database start in the year 1950 and end in November 2011. In the earlier years of the database there are generally fewer events recorded, most likely due to a lack of good records. More recent years should be considered more complete.  

(TODO) In the Data Procession section, I will discuss the strategy and implementation for dealing with the relatively lacking of good records in the earlier years.

In addition, the documentations cited above leave much to be desired in terms of clear semantics and interpretation of the contents of the data file.  In the Data Processing section, I'll go into details of what those gaps are and how I deal with them.

## Data Processing
This section describes the strategy, rationale, and implemention of the data processing and analysis.


### Load the Data into R
Let's load the data into a dataframe first:

```r
# -- # install.packages("R.utils")  # you should install R.utils before running this markdown file.
library(R.utils)

url="https://d396qusza40orc.cloudfront.net/repdata%2Fdata%2FStormData.csv.bz2"
dest="repdata-data-StormData.csv.bz2"
# download.file(url, destfile=dest, method="curl")
# bunzip2(dest, overwrite=TRUE, remove=FALSE)
df1 <- read.csv("repdata-data-StormData.csv", stringsAsFactors=F)

# df1 <- read.csv("tmp.csv", stringsAsFactors=F)  # smaller data set for debugging
```

### Data Cleansing and Preprocessing
This section performs various preprocessing on the dataset.

#### NA Verification

The columns FATALITIES, INJURIES and PROPDMG contain the key quantities used in the analysis.
Code below verifies there are no missing/NA values in these columns:


```r
sum(is.na(df1$FATALITIES))
```

```
## [1] 0
```

```r
sum(is.na(df1$INJURIES))
```

```
## [1] 0
```

```r
sum(is.na(df1$PROPDMG))
```

```
## [1] 0
```

#### Append the year column
BGN_DATE column contains the beginning date of each of the storm events in the dataframe.  It is a string in the format of "MM/DD/yyyy hh:mm:ss".   Let's extract the year portion from the string and append that info as a new column.  We'll be doing some comparison/analysis/verification based on a year-to-year basis.


```r
df1["year"] <- as.integer(sub(".*/.*/(.*) .*", "\\1", df1$BGN_DATE) )
```

#### Normalize Property Damange Quantity
Below is an excerpt from Section 2.7, Damange in [Storm Data Documentation](https://d396qusza40orc.cloudfront.net/repdata%2Fpeer2_doc%2Fpd01016005curr.pdf):

*Estimates should be rounded to three significant digits, followed by an alphabetical character signifying the magnitude of the number, i.e., 1.55B for $1,550,000,000. Alphabetical characters used to signify magnitude include “K” for thousands, “M” for millions, and “B” for billions. *

However, in PROPDMGEXP column, in addition to characters such as M/m, K, H/h, B, you can also find other characters as shown below:


```r
table(df1["PROPDMGEXP"])
```

```
## 
##             -      ?      +      0      1      2      3      4      5 
## 465934      1      8      5    216     25     13      4      4     28 
##      6      7      8      B      h      H      K      m      M 
##      4      5      1     40      1      6 424665      7  11330
```

(BTW, I cannot find any documentation indicating PROPDMGEXP is the indicator of "the magnitude of the number" -- it is an educated guess as to what that column represents.)

The ratio below (ie, n_subset/n_total) indicates about half of the data points have a defined indicator (ie, B, M, K, or H - case-insensitive).


```r
n_total <- nrow(df1)
n_total
```

```
## [1] 902297
```

```r
n_subset <- nrow(df1[toupper(df1$PROPDMGEXP) %in% c('B','M','K','H'), ])
n_subset
```

```
## [1] 436049
```

```r
n_subset / n_total
```

```
## [1] 0.4832655
```

Turning our attention to the other rows (that is, those rows without any indicator or with an indicator that is NOT one of the above defined ones):

First, for those rows with an empty PROPDMGEXP, we see that most of them have 0 as the PROPDMG:

```r
n_total2 <- nrow(df1[df1$PROPDMGEXP=='', ])
n_total2
```

```
## [1] 465934
```

```r
n_subset2 <- nrow(df1[df1$PROPDMGEXP=='' & df1$PROPDMG == 0.0, ])
n_subset2
```

```
## [1] 465858
```

```r
n_subset2 / n_total2
```

```
## [1] 0.9998369
```

Given the stats above, for this analysis, we will treat rows with an empty string for PROPDMGEXP as having an undefined property damage amount.

Second, regarding those rows with an imprecise indicator (e.g, -, +, ?, 0-8): after reviewing some of those rows (including their REMARKS column values), there does not appear to be any strong indication what those indicators mean.  Therefore, for this analysis, we will treat the property damage amount for those rows as undefined as well.   This should not skew our analysis too greatly as the number of these rows is insignificant.

Below we will add a column called propdmg_norm (in dollars) to the data set based on the decisions we made above:


```r
compute_propdmg_norm <- function(val, indicator) {
  if (toupper(indicator) == 'B') {
    retval <- val * 10^9
  } else if (toupper(indicator) == 'M' ) {
    retval <- val * 10^6
  } else if (toupper(indicator) == 'K' ) {
    retval <- val * 10^3
  } else if (toupper(indicator) == 'H' ) {
    retval <- val * 10^2
  } else {
    retval <- NA
  }
  retval
}

df1["propdmg_norm"] <- mapply(compute_propdmg_norm, df1$PROPDMG, df1$PROPDMGEXP)

# head(df1[,c("EVTYPE", "PROPDMG", "PROPDMGEXP", "propdmg_norm")])
```

#### Normalize Event Type
The code below counts the total number of distinct EVTYPE values (case-insensitive) in the entire dataset:

```r
 length(sort(unique(toupper(df1$EVTYPE))))
```

```
## [1] 898
```

The resulting count is much higher than the number of Storm Data Event type defined in section 2.1.1, Storm Data Event Table in [Storm Data Documentation](https://d396qusza40orc.cloudfront.net/repdata%2Fpeer2_doc%2Fpd01016005curr.pdf).

One thing we could do is map all the distinct EVTYPEs in the input dataset to the ones defined in Storm Data Event Table.  However, that is a lot of work.   Instead, the approach this analysis will take is to filter the data set by the health damage (and later by property damange).   The result of filtering will be a lot less rows than the original dataset.  Hopefully the number of distinct EVTYPEs in those subsets would also be much less, which would reduce the work of mapping.

#### Effect of number of data samples from different years

The code below produces a plot showing the number of data points from each year:

```r
library(dplyr)
```

```
## 
## Attaching package: 'dplyr'
## 
## The following object is masked from 'package:stats':
## 
##     filter
## 
## The following objects are masked from 'package:base':
## 
##     intersect, setdiff, setequal, union
```

```r
library(ggplot2)

summary1 <- df1 %>% group_by(year) %>%
  summarise(cnt = n())
ggplot(summary1, aes(x=year,y=cnt)) + geom_point() + geom_line()
```

![](PA2_files/figure-html/unnamed-chunk-9-1.png) 

Specifically, prior to year 1989, the number of data points per year is less than 10,000.

The code below produces a plot showing the number of health damage (for the purpose of this analysis, this is equal to the number of FATALITIES plus INJURIES from the dataset), from each year:


```r
summary_hdmg <- df1 %>% group_by(year) %>%
  mutate(hdmg = FATALITIES + INJURIES) %>%
  summarise(sum_hdmg = sum(hdmg) )

ggplot(summary_hdmg, aes(x=year,y=sum_hdmg)) + geom_point() + geom_line()
```

![](PA2_files/figure-html/unnamed-chunk-10-1.png) 

The code below produces a plot showing the total property damage amount (in millions of dollars) from each year:


```r
summary_pdmg <- df1 %>% 
  filter(! is.na(propdmg_norm)) %>%
  group_by(year) %>%
  summarise(sum_pdmg = sum(propdmg_norm) / 10^6)

ggplot(summary_pdmg, aes(x=year,y=sum_pdmg)) + geom_point() + geom_line()
```

![](PA2_files/figure-html/unnamed-chunk-11-1.png) 

Same data, except that the total property damage amount is plotted on the log10 scale:

```r
ggplot(summary_pdmg, aes(x=year,y=sum_pdmg)) + geom_point() + geom_line() + scale_y_log10()
```

![](PA2_files/figure-html/unnamed-chunk-12-1.png) 

What observations regarding the dataset can we make from the plots above:

1. The number of data points per year is higher in recent years than earlier years.
2. The health damage shows only a slight increasing trend in the time span covered by the dataset, albeit certain years show significantly worse health damage (using 5000 as the threshold).
3. The property damange mount increases in an exponential mannger.  The document implies the dollar amount are 'actual dollar amounts' (Section 2.7, Damange), so it's reasonable to assume the amounts are not inflation adjusted.

Given these observations, one might raise the following questions:

* What factors contribute to the fact that there are more data points in recent years?  Possible factors:
  + Better record keeping
  + Worsening climate conditions
* What can explain the fairly steady health damange amount, given that there is a general, steady increase in data points from year to year?
  + Better protection against inclimate weather conditions?
  + Several weather conditions cause a similar amount of health damange from year to year that technology has not been able to prevent more effectively yet.
* What can explain the exponential increase in property damage amount?
  + Inflation?
  + More data points per year?

To reduce the influence of the factors not directly related to the weather conditions, this analysis will use data from 1985 and onward.  This should reduce the effect of the apparent worse record keeping in prior years.   This should also reduce the inflation effect (assuming that is a factor).

### Data Analysis

#### Subsetting the Dataset
Since we are interested in data points that cause property damange and/or health damage, the code below subset the dataset based as follows:

1. The row is from year is 1985 or later
2. The row contains one or more of the following: (1) PROPDMB > 0, (2) FATALITIES > 0, (3) INJURIES > 0


```r
df2 <- df1[df1$year >= 1985 & (df1$PROPDMG+df1$FATALITIES+df1$INJURIES >0),]
```

#### Normalize event types in the subset
The code below shows the number of unique EVTYPE in the subset.  

```r
evtype_list <- sort(unique(df2$EVTYPE))
length(evtype_list)
```

```
## [1] 467
```

```r
head(evtype_list)
```

```
## [1] "   HIGH SURF ADVISORY" " FLASH FLOOD"          " TSTM WIND"           
## [4] " TSTM WIND (G45)"      "?"                     "APACHE COUNTY"
```

After visually inspecting the unique EVTYPE values in the subset, the mapping of those values to the event types defined in the documentation Storm Data is accomplished by the code below:


```r
df_mapping <- as.data.frame(rbind(
c("Astronomical Low Tide", '^ASTRONOMICAL LOW TIDE' ),
c("Avalanche", '^AVALANC' ),
c("Blizzard", '^BLIZZARD|^GROUND BLIZZARD' ),
c("Coastal Flood", '^COASTAL [ ]*FLOOD|^.*Cstl Flood'),
c("Cold/Wind Chill", '^COLD|^Ext.* Cold|^EXTREME WIND[ ]*CHILL|^RECORD COLD|^UNSEASONABLY COLD|^LOW TEMPERATURE' ),
c("Debris Flow", '^XXXXunknownXXXX'),
c("Dense Fog", '^DENSE FOG|^FOG'),
c("Dense Smoke", '^XXXXunknownXXXX' ),
c("Drought", '^DROUGHT'),
c("Dust Devil", '^DUST DEVIL'),
c("Dust Storm", '^DUST STORM'),
c("Excessive Heat", '^EXCESSIVE HEAT|EXTREME HEAT|^RECORD HEAT'),
c("Extreme Cold/Wind Chill", '^EXTREME COLD|^EXTREAM WINDCHILL'),
c("Flash Flood", '^[ ]*FLASH FLOOD.*|^BREAKUP FLOODING'),
c("Flood", '^FLOOD|^MAJOR FLOOD|^MINOR FLOOD|^river.*flood|^RURAL FLOOD|^URBAN FLOOD|^TIDAL FLOODING|^SNOWMELT FLOODING'  ),
c("Frost/Freeze", '^FROST|^FREEZE|^DAMAGING FREEZE'),
c("Funnel Cloud", '^FUNNEL CLOUD'),
c("Freezing Fog", '^FREEZING FOG'),
c("Hail", '^HAIL|^WIND/HAIL'),
c("Heat", '^HEAT'),
c("Heavy Rain", '^HEAVY RAIN|^EXCESSIVE RAINFALL|^HEAVY PRECIPITATION|^RAIN$|^RAINSTORM$|^Torrential Rainfall'),
c("Heavy Snow", '^HEAVY SNOW|^EXCESSIVE SNOW|^HEAVY LAKE SNOW|^snow$|^SNOW.*HEAVY SNOW|^SNOW SQUALL|^BLOWING SNOW|^FALLING SNOW/ICE|^RECORD SNOW|^SNOW/.*'),
c("High Surf", '.*HIGH SURF|^HIGH SEAS|^HIGH SWELLS|^HIGH WATER|^HAZARDOUS SURF|^HEAVY SURF|^HEAVY SWELL|^HIGH WAVE|^ROGUE WAVE|^ROUGH SEAS|^ROUGH SURF'),
c("High Wind", '^HIGH [ ]*WIND[S]*.*$|^WIND[S]*$|^GUSTY.*WIND|^NON.*TSTM WIND|^NON-SEVERE WIND DAMAGE|^STORM FORCE WINDS|^WIND DAMAGE|^WIND STORM'  ),
c("Hurricane (Typhoon)", '^HURRICANE|^TYPHOON'),
c("Ice Storm", '^ICE STORM|^ICE/STRONG WINDS|^SNOW AND ICE'),
c("Lake-Effect Snow", '^LAKE.EFFECT SNOW'),
c("Lakeshore Flood", '^XXXXunknownXXXX'),
c("Lightning", '^LIGHTNING|^LIGNTNING|^LIGHTING'),
c("Marine Hail", '^MARINE HAIL'),
c("Marine High Wind", '^MARINE HIGH WIND'),
c("Marine Strong Wind", '^MARINE STRONG WIND'),
c("Marine Thunderstorm Wind", '^MARINE T.*ST.*M WIND'),
c("Rip Current", '^RIP CURRENT'),
c("Seiche", '^SEICHE'),
c("Sleet", '^SLEET|^SNOW/SLEET|^FREEZING RAIN/SLEET'),
c("Storm Surge/Tide", '^STORM SURGE|^COASTAL SURGE'),
c("Strong Wind", '^STRONG WIND'),
c("Thunderstorm Wind", '^THU[N]*DE[E]*R[E]*STORM [ ]*WIN|^[ ]*TSTM|^SEVERE THUNDERSTORM|^TUNDERSTORM WIND' ),
c("Tornado", '^TORNADO|^TORNDAO'),
c("Tropical Depression", '^TROPICAL DEPRESSION'),
c("Tropical Storm", '^TROPICAL STORM'),
c("Tsunami", '^TSUNAMI'),
c("Volcanic Ash", '^VOLCANIC ASH'),
c("Waterspout", '^WATERSPOUT'),
c("Wildfire", '^WILD.*FIRE'),
c("Winter Storm", '^WINTER STORM'),
c("Winter Weather", '^WINTER WEATHER|^Wintry Mix')
))
names(df_mapping) <- c("typname", "pattern")

for (i in 1:nrow(df_mapping)) {
  pttrn <- as.character(df_mapping[i,"pattern"])
  typnm <- as.character(df_mapping[i,"typname"])
  df2[grep(pttrn, df2$EVTYPE, ignore.case=T), "evtype_norm"] <- typnm
}

dim(df2)
```

```
## [1] 227578     40
```

```r
dim(df2[ is.na(df2$evtype_norm),])
```

```
## [1] 1473   40
```
Notes:

1. The total number of unmapped EVTYPE (ie, where df2$evtype_norm is NA) is very small compared to the total number of data points in df2.
2. The mapping between EVTYPE and evtype_norm requires a lot of arbitraty decisions.  

At this point we have cleaned up / normalized the data to the point we can proceed to the next step of analyzing the data (stored in df2).

## Results

### Events That Cause the Most Population Health Damage

Following code computes the health damage grouped by evtype_norm, ordered by total health damange, with top 15 evtype_norm displayed.  The data used starts from 1985 to the latest data available in the dataset:

```r
df2 %>% group_by(evtype_norm) %>%
  mutate(hdmg = FATALITIES + INJURIES) %>%
  summarise(sum_hdmg = sum(hdmg), cnt = n(), avg_hdmg = sum(hdmg) / n() ) %>%
  select(evtype_norm, sum_hdmg, cnt, avg_hdmg) %>%
  arrange(desc(sum_hdmg)) %>%
  head(15)
```

```
## Source: local data frame [15 x 4]
## 
##            evtype_norm sum_hdmg    cnt    avg_hdmg
## 1              Tornado    33530  18559  1.80667062
## 2    Thunderstorm Wind     9793 118157  0.08288125
## 3       Excessive Heat     8731    715 12.21118881
## 4                Flood     7310  10534  0.69394342
## 5            Lightning     6049  13233  0.45711479
## 6                 Heat     3612    246 14.68292683
## 7          Flash Flood     2803  21230  0.13203015
## 8            Ice Storm     2071    714  2.90056022
## 9            High Wind     1901   6337  0.29998422
## 10            Wildfire     1696   1200  1.41333333
## 11        Winter Storm     1570   1508  1.04111406
## 12 Hurricane (Typhoon)     1468    230  6.38260870
## 13                Hail     1300  23217  0.05599345
## 14          Heavy Snow     1294   1515  0.85412541
## 15           Dense Fog     1158    182  6.36263736
```

In order to evaluate wether skewness was introduced by the EVTYPE-to-evtype_norm mapping, the same logic is used below to compute the health damange -- but this time grouped by the original EVTYPE:

```r
df2 %>% group_by(EVTYPE) %>%
  mutate(hdmg = FATALITIES + INJURIES) %>%
  summarise(sum_hdmg = sum(hdmg), cnt = n(), avg_hdmg = sum(hdmg) / n() ) %>%
  select(EVTYPE, sum_hdmg, cnt, avg_hdmg) %>%
  arrange(desc(sum_hdmg)) %>%
  head(15)
```

```
## Source: local data frame [15 x 4]
## 
##               EVTYPE sum_hdmg   cnt    avg_hdmg
## 1            TORNADO    33487 18544  1.80581320
## 2     EXCESSIVE HEAT     8428   697 12.09182209
## 3              FLOOD     7259  9829  0.73852884
## 4          TSTM WIND     7077 62106  0.11395034
## 5          LIGHTNING     6046 13223  0.45723361
## 6               HEAT     3037   212 14.32547170
## 7        FLASH FLOOD     2755 20879  0.13195076
## 8          ICE STORM     2064   708  2.91525424
## 9  THUNDERSTORM WIND     1621 43459  0.03729952
## 10      WINTER STORM     1527  1506  1.01394422
## 11         HIGH WIND     1385  5504  0.25163517
## 12 HURRICANE/TYPHOON     1339    71 18.85915493
## 13              HAIL     1300 23201  0.05603207
## 14        HEAVY SNOW     1148  1335  0.85992509
## 15          WILDFIRE      986   816  1.20833333
```

### Events That Cause the Most Property Damage
Following code computes the property damange amounts grouped by evtype_norm, ordered by total health damange, with top 15 evtype_norm displayed.  The data used starts from 1985 to the latest data available in the dataset:

```r
df2 %>% group_by(evtype_norm) %>%
  summarise(sum_pdmg = sum(propdmg_norm, na.rm = T), cnt = n(), avg_pdmg = sum(propdmg_norm, na.rm =T) / n() ) %>%
  select(evtype_norm, sum_pdmg, cnt, avg_pdmg) %>%
  arrange(desc(sum_pdmg)) %>%
  head(15)
```

```
## Source: local data frame [15 x 4]
## 
##            evtype_norm     sum_pdmg    cnt     avg_pdmg
## 1                Flood 150327927600  10534  14270735.49
## 2  Hurricane (Typhoon)  85356410010    230 371114826.13
## 3     Storm Surge/Tide  47965224000    225 213178773.33
## 4              Tornado  37998400210  18559   2047437.91
## 5          Flash Flood  16732868610  21230    788170.92
## 6                 Hail  15974470220  23217    688050.58
## 7    Thunderstorm Wind  10968253230 118157     92827.79
## 8             Wildfire   8491563500   1200   7076302.92
## 9       Tropical Storm   7714390550    410  18815586.71
## 10        Winter Storm   6748997250   1508   4475462.37
## 11           High Wind   6014993410   6337    949186.27
## 12           Ice Storm   3948927860    714   5530711.29
## 13          Heavy Rain   3223348190   1073   3004052.37
## 14             Drought   1046106000     55  19020109.09
## 15          Heavy Snow    975754690   1515    644062.50
```

In order to evaluate wether skewness was introduced by the EVTYPE-to-evtype_norm mapping, the same logic is used below to compute the health damange -- but this time grouped by the original EVTYPE:
```{R}
df2 %>% group_by(EVTYPE) %>%
  summarise(sum_pdmg = sum(propdmg_norm, na.rm = T), cnt = n(), avg_pdmg = sum(propdmg_norm, na.rm =T) / n() ) %>%
  select(EVTYPE, sum_pdmg, cnt, avg_pdmg) %>%
  arrange(desc(sum_pdmg)) %>%
  head(15)
```

