### 1. Code for reading in the dataset and/or processing the data

Lets get the data in here and load some packages.

    # 1. Code for reading in the dataset and/or processing the data
    if (!file.exists("activity.csv")) {
            unzip("activity.zip")
    }
    ped_data <- read.csv("activity.csv")
    library(tidyverse)

    ## -- Attaching packages ------------------------------------------------------------------------------------------ tidyverse 1.3.0 --

    ## v ggplot2 3.3.2     v purrr   0.3.4
    ## v tibble  3.0.3     v dplyr   1.0.0
    ## v tidyr   1.1.0     v stringr 1.4.0
    ## v readr   1.3.1     v forcats 0.5.0

    ## -- Conflicts --------------------------------------------------------------------------------------------- tidyverse_conflicts() --
    ## x dplyr::filter() masks stats::filter()
    ## x dplyr::lag()    masks stats::lag()

    library(gridExtra)

    ## 
    ## Attaching package: 'gridExtra'

    ## The following object is masked from 'package:dplyr':
    ## 
    ##     combine

### 2. Histogram of the total number of steps taken each day

Here I use the Dplyr package to summarize the total steps taken by date
and then plot the distribution on a histogram using ggplot.

    daily_steps <- ped_data %>% group_by(date) %>% summarize(daily_steps_sum = sum(steps, na.rm = T))

    ## `summarise()` ungrouping output (override with `.groups` argument)

    total_step_distribution <- ggplot(daily_steps, aes(daily_steps_sum)) + geom_histogram(bins = 20) 
    total_step_distribution

![](Course-Project_files/figure-markdown_strict/unnamed-chunk-2-1.png)

### 3. Mean and median number of steps taken each day

Here I calculate the mean and median of steps taken when grouped by day.
Though the second graph is completely flat, this constant median of 0 is
testament to truly just how many values of “steps” are 0. Which is also
why I suspect the mean values are so low. This reflects the data in 2
which has an incredible amount of days that have a total of 0 steps for
all 288 observations.

    meanmed <- ped_data %>% group_by(date) %>% summarize(mean_steps = mean(steps, na.rm = T), 
                                                            median_steps = median(steps, na.rm = T))

    ## `summarise()` ungrouping output (override with `.groups` argument)

    mean_plot <- ggplot(meanmed, aes(mean_steps)) + geom_histogram(bins = 20)
    med_plot <- ggplot(meanmed, aes(median_steps)) + geom_histogram()
    grid.arrange(mean_plot, med_plot)

    ## Warning: Removed 8 rows containing non-finite values (stat_bin).

    ## `stat_bin()` using `bins = 30`. Pick better value with `binwidth`.

    ## Warning: Removed 8 rows containing non-finite values (stat_bin).

![](Course-Project_files/figure-markdown_strict/unnamed-chunk-3-1.png)

### 4. Time series plot of the average number of steps taken

Here I group the 288 different types of interval data available in the
dataset(24 hours \* (60/5 minutes)) and find the sum of steps taken for
each interval.

    time_activity <- ped_data %>% group_by(interval) %>% summarize(step_minute = sum(steps, na.rm = T)) 

    ## `summarise()` ungrouping output (override with `.groups` argument)

    time_activity_plot <- ggplot(time_activity, aes(interval, step_minute)) + geom_line()
    time_activity_plot

![](Course-Project_files/figure-markdown_strict/unnamed-chunk-4-1.png)

### 5. The 5-minute interval that, on average, contains the maximum number of steps

Here I subset the grouped interval data made in 4 with a logical index.
Highest\_time finds the row that corresponds to the max of that row and
then returns the first column for that row, yielding the time with the
highest total steps per day, 8:35.

    highest_time <- time_activity[time_activity$step_minute == max(time_activity$step_minute), 1]
    highest_time

    ## # A tibble: 1 x 1
    ##   interval
    ##      <int>
    ## 1      835

### 6. Code to describe and show a strategy for imputing missing data

In most cases the median is a more accurate statistical summary of a
dataset that has a high amount of outliers (in this case 0), however in
this dataset the ratio of 0 values to non-zero values is actually
&gt;50% for most time intervals & days. Because of this I have decided
to replace NA values in the “steps” column with the mean of the steps
taken for that interval across all days.  
Here I use a logical vector to subset the data that has an NA value for
the number of steps taken for that interval. Then I merge this NA
dataset with the summary dataset “time\_activity\_means” along the
matched “interval” column, replacing the average steps taken for each
interval with the corresponding NA value for that row.

    ## I have decided instead to substitute the interval instead.
    na_steps <- ped_data[is.na(ped_data$steps), ]
    time_activity_means <- ped_data %>% group_by(interval) %>% summarize(step_minute = mean(steps, na.rm = T))

    ## `summarise()` ungrouping output (override with `.groups` argument)

    na_means <- merge(na_steps, time_activity_means)
    na_means$steps <- na_means$step_minute
    na_means <- select(na_means, steps, date, interval)
    ped_merge <- rbind(na_means, ped_data[!is.na(ped_data$steps), ])

### 7. Histogram of the total number of steps taken each day after missing values are imputed

Here I take the merged NA replacement data above and plot a histogram
showing the distribution of the total daily steps for each day. This
graph is interesting because it kinda looks similar to the distribution
of the non-NA values shown in 2, yet it has the glaring difference of
much fewer days with 0 steps. This makes sense when comparing it to the
graph in 3, which shows the mean values for each day. Because we
replaced NA values with their mean across that time interval, many
intervals that would otherwise be 0 for that day are non-0, lending the
distribution to be most similar to 3 rather than 2.

    ped_merge_steps <- ped_merge %>% group_by(date) %>% summarize(daily_steps_sum = sum(steps))

    ## `summarise()` ungrouping output (override with `.groups` argument)

    ped_merge_total_plot <- ggplot(ped_merge_steps, aes(daily_steps_sum)) + geom_histogram(bins = 20)
    ped_merge_total_plot

![](Course-Project_files/figure-markdown_strict/unnamed-chunk-7-1.png)

### 8. Panel plot comparing the average number of steps taken per 5-minute interval across weekdays and weekends

Here I use the weekdays() function along with mutate() to add a column
for the corresponding weekday of a specific date. Then I subset the days
with filter and then summarize their average steps taken by interval.
Finally, I plot the two corresponding line graphs onto a multi-panel
output with ggplot2 and gridExtra.

    # 8. Panel plot comparing the average number of steps taken per 5-minute interval across weekdays and weekends
    ped_days <- mutate(ped_merge, day = weekdays(as.Date(ped_merge$date)))
    ped_weekend <- filter(ped_days, day == "Saturday"| day == "Sunday")
    ped_workweek <- filter(ped_days, day != "Saturday" & day != "Sunday")
    ped_weekend_avg <- ped_weekend %>% group_by(interval) %>% summarize(average_steps = mean(steps))

    ## `summarise()` ungrouping output (override with `.groups` argument)

    ped_workweek_avg <- ped_workweek %>% group_by(interval) %>% summarize(average_steps = mean(steps))

    ## `summarise()` ungrouping output (override with `.groups` argument)

    weekend_plot <- ggplot(ped_weekend_avg, aes(interval, average_steps)) + geom_line() +
            labs(x = "Interval", y = "Average Number of Steps", title = "Weekend Average (by 5 minute Interval)")
    workweek_plot <- ggplot(ped_workweek_avg, aes(interval, average_steps)) + geom_line() +
            labs(x = "Interval", y = "Average Number of Steps", title = "Workweek Average (by 5 minute Interval)")
    grid.arrange(weekend_plot, workweek_plot)

![](Course-Project_files/figure-markdown_strict/unnamed-chunk-8-1.png)
