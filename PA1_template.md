# Reproducible Research: Peer Assessment 1

For this analysis we use the following libraries as dependencies and hence they have to be loaded into the environment.


```r
library(dplyr)
library(ggplot2)
```

## Loading and preprocessing the data

Load the data from the provided zip file:


```r
conn <- unz("activity.zip", filename = "activity.csv")
activity_data <- read.csv(conn, colClasses = c("integer", "Date", "integer"))
```

## What is mean total number of steps taken per day?

First, we want to summarize the data and compute the total number of steps made in each day of the measurement period. 


```r
daily_steps <- activity_data %>%
    group_by(date) %>% 
    summarise(steps = sum(steps, na.rm = T))
```

Next, let's take a look at the distribution of the daily number of steps:


```r
with(daily_steps, hist(steps, breaks=max(steps) / 1000))
```

![](PA1_template_files/figure-html/distribution_of_total_daily_steps-1.png) 

Because we ignored the missing values when we did our summaries, we ended up having a lot of days with the total number of steps equal to zero. We can see that in the histogram above.

And finally, let's see what are the mean and median total number of steps taken each day.


```r
with(daily_steps,
     list(mean = mean(steps),
          median = median(steps)))
```

```
## $mean
## [1] 9354.23
## 
## $median
## [1] 10395
```

## What is the average daily activity pattern?

In order to visualize the average daily activity pattern, we need to summarise the data and compute the average of steps taken within each time interval across all the days.


```r
activity_pattern <- activity_data %>%
    group_by(interval) %>% 
    summarise(avg_steps = mean(steps, na.rm = T))
```

Next, we need to plot the results.


```r
ggplot(mapping = aes(x = interval, y = avg_steps), data=activity_pattern) + 
    geom_line() +
    labs(title="Average steps taken per time interval", y = "average steps taken")
```

![](PA1_template_files/figure-html/daily_activity_pattern-1.png) 

Here's an interesting question: which 5-minute interval, on average across all the days in the dataset, contains the maximum number of steps?


```r
filter(activity_pattern, avg_steps == max(avg_steps))$interval
```

```
## [1] 835
```

The result tells us that the most walking on average is done from 8:35 am to 8:40 am, which is consistent with the pattern of a person going to work in the morning, probably to start the work day around 9:00 am.

## Imputing missing values

Note that there are a number of days/intervals where there are missing values (coded as ```NA```). The presence of missing days may introduce bias into some calculations or summaries of the data.

Let's see how many observations with missing values for the step measurement are.


```r
summarise(activity_data, NAs = sum(is.na(steps)), NAs.percentile = NAs / n() * 100)
```

```
##    NAs NAs.percentile
## 1 2304       13.11475
```

We can see that about 13% of the mesaurements are missing values. Since the daily activity patterns for a person are similar throughout the day (sleep, waking up, going to work, working, e.t.c.), we could fill in the missing values by using the means of the corresponding 5-minutes intervals.

First, we `merge` the data by interval, so we have all the steps and their corresponding mean values together. Next we use `mutate` to update the missing steps values with the mean values. Because the merge operation reordered both the columns and the rows, the last two steps are to select and reorder back the columns to match the initial dataset and then to sort the dataframe by date.


```r
n_activity_data <- merge(activity_data, activity_pattern, by="interval") %>%
    mutate(steps = ifelse(is.na(steps), avg_steps, steps)) %>%
    select(steps, date, interval) %>% 
    arrange(date)
```

Let's now do the same analysis for the imputed data set, as we did for the initial one.


```r
n_daily_steps <- n_activity_data %>%
    group_by(date) %>% 
    summarise(steps = sum(steps))
```


```r
with(n_daily_steps, hist(steps, breaks=max(steps) / 1000))
```

![](PA1_template_files/figure-html/n_distribution_of_total_daily_steps-1.png) 


```r
with(n_daily_steps,
     list(mean = mean(steps),
          median = median(steps)))
```

```
## $mean
## [1] 10766.19
## 
## $median
## [1] 10766.19
```

We can see that the distribution of total number of steps represents more accurately a normal distribution that the original in data set. Also the mean and median values are equal now, since the total number of daily steps is no more skewed towards zero, for the days when we didn't have any measurements.

## Are there differences in activity patterns between weekdays and weekends?

In order to answer this question we need first to tag each measurement with the corresponding type of week day: weekday or weekend.


```r
n_activity_data <- n_activity_data %>%
    mutate(day_type = as.factor(
        ifelse(weekdays(date) %in% c("Saturday", "Sunday"), 
               "weekend",
               "weekday")))
```

Next, let summarise the data and compute the average number of steps taken in each 5-minutes interval based on the day of week type.


```r
n_activity_pattern <- n_activity_data %>%
    group_by(interval, day_type) %>% 
    summarise(avg_steps = mean(steps))
```

And finally, let's vizualize the results.


```r
ggplot(mapping = aes(x = interval, y = avg_steps), data=n_activity_pattern) + 
    geom_line() +
    facet_wrap(~ day_type) +
    labs(title="Average steps taken per time interval", y = "average steps taken")
```

![](PA1_template_files/figure-html/weekday_activity_pattern-1.png) 

We can see in the plot above slightly different patterns of activities:

1. On weekdays, we have a significant spike in number of steps in the morning (probably when walking to work) and a lower number of steps taken during the work hours.
2. On weekends, on the other hand, we don't see a morning spike anymore, though the number of steps taken throughout the day is bigger that on weekdays. This is probably due to outdoor activities, like walking or hicking.
