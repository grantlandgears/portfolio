# CYCLISTIC DATA STUDY
<img src="https://drive.google.com/uc?export=view&id=1vaicvK3W1eCDOc7PG_tpRctlsRmLV5vZ" width="250" height="200">

## This markdown contains the detailed steps I took in my analysis of the Cyclistic dataset.  The company is fictional as part of a Coursera / Google DA cert, but data is real - unsure from what company.  Here is my [deck outlining my findings](https://docs.google.com/presentation/d/1aay_YG4JIxWKMNDpuxqPFX2U4PCMtRjTa-zuxy40RRA/edit?usp=sharing)

## The ask
Cyclisitic is looking to increase revenue by converting casual riders to subscribed members locking in regular monthly subscriptions.  The questions posed are:
1. How do annual members and casual riders use Cyclistic bikes differently?
2. Why would casual riders buy Cyclistic annual memberships?

## The Dataset(s)
The data came in 12 .csvs, one for each month from Oct 2020 to Sept 2021, so the data is quite up to date.  The .csvs ranged in size from 9MB to 150MB.  Total number of rows was 5 million.  I definitely had some challenges just being able to view all the data!

## EDA
The first place I went to was the fastest - Gsheets.  I pulled in the smallest monthly file because anything bigger choked up Gsheets.  The .csvs had up to hundreds of thousands of rows, but they ranged in sized.  I feel confident this is due to the seasons with fewer riders in the winter than summer.  Feb .csv was the smallest and July was the biggest.  That's neither here nor there, but just an observation.

### Gsheets
I pulled in February data to Gsheets and started to get used to the data.  Normally I would prefer to go to SQL, but popping over to Gsheets was fastest in this case.  Although at a price as I only imported in 1400 rows out of 5 million.  There were 13 columns and here's a sample line of data:

ride_id	|rideable_type	|started_at	|ended_at	|start_station_name	|start_station_id	|end_station_name	|end_station_id	|start_lat	|start_lng	|end_lat	|end_lng	|member_casual
------------ |	------------ |	------------ |	------------ |	------------ |	------------ |	------------ |	------------ |	------------ |	------------ |	------------ |	------------ |	------------ |
89E7AA6C29227EFF	|classic_bike	|2021-02-12 16:14:56	|2021-02-12 16:21:43	|Glenwood Ave & Touhy Ave	|525	|Sheridan Rd & Columbia Ave	|660	|42.012701	|-87.666058	|42.004583	|-87.661406	|member

The ones that popped to me first were member, rideable type, started_at and ended_at.  I suspected they would be the most use to the questions I was looking to answer.  I checked through the data to see how clean it was.  I looked for nulls, data type discrepencies, garbage data (eg for an address 123456789 Main Ave).  All in all the data looked pretty clean.  I would need to more thoroughly clean it of course, but it looked pretty good which makes sense as this data is clearly automatically captured by Cyclistic software and not by humans.

The started_at and ended_at were a date timestamp.  I knew I'd want to look at time of day and day of week separate, so I started to pull those apart.  I used the started_at time as I was more interested in when a bike ride was initiated for this.  This was done with a split on space:
```
=split(B2, " ")
```
I then created a day_phase column which put the started_at time into two buckets; morning (before noon) and afternoon.  Using a pivot and a quick calculation I found both riders rode more often in the afternoon, but members rode a little more evenly throughout the day whereas casual riders were more heavily in the afternoon.
```
casual percentage afternoon 78.67%  member percentage afternoon 69.22%
casual percentage morning 21.33%  member percentage morning 30.78%
```
I was also curious to see average ride duration which I got to by first subtracting the started_at time from the ended_at time and a pivot.  The average casual rider time was 2x a member!  This was a substantial delta I would dig into further as it was a clear difference.

Next I would curious about bike type.  This data needed no modifying, just a pivot.  Members never rode a docked bike which jumped out.  I thought this was an ah-ha moment - altough this would prove wrong later as I pulled in more data.

Lastly I wanted to know what days of the week casual and members rode the most.  I first pulled the day of week from the date timestamp:
```
=TEXT(B2,"dddd")
```

The pivoted that for totals by day and a calculation to average per total percentage.  Again was another ah-ha moment where you saw a very consistent ridership in members throughout the week, particularly M-F whereas the casual riders were more heavily on the weekends:
![image](https://user-images.githubusercontent.com/70623337/140694682-cbef2b28-9ef7-4ff8-a6ed-23f6510319a0.png)

At this point i was ready to get more data, so on to rstudio!

### Rstudio
My big challenge with Rstudio was getting the data in there, all 5 million rows.  I would run out ram trying to merge 3 of the 10 csv's in memory.  I had to import each csv one at a time then save out a new csv with only the columns I wanted to work with then import those back into Rstudio and merge them togther.  Phew, it was exhausting.

Before that though I found a more efficient way to import and load packages (thank interwebs):
```
IMPORT PACKAGES
#Package names
packages <- c("tidyverse", "readr", "lubridate", "dplyr", "skimr", "DataExplorer", "ggplot2", "sqldf","data.table")

#Install packages not yet installed
installed_packages <- packages %in% rownames(installed.packages())
if (any(installed_packages == FALSE)) {
  install.packages(packages[!installed_packages])
}

#Packages loading
invisible(lapply(packages, library, character.only = TRUE))
```
Since I was consistently running out of Ram and needed to restart Rstudio it was nice to find the above shortcut for loading packages.

I discovered 'fread' which was great because I could import select columns at a time and reduce memory:
```
df <- read.csv("/cloud/project/bike_data/source_sets/202010-divvy-tripdata.csv")
df1 <- fread("/cloud/project/bike_data/source_sets/202010-divvy-tripdata.csv", select = c("rideable_type","started_at","member_casual"))
```
There was a discrepancy in the csvs where two of them had their columns in a different format, so I needed to convert those:
```
glimpse(df)

#convert one column from int to character
df1$start_station_id <- as.character(as.numeric(df1$start_station_id))  

#convert multiple columns from int to character
cols.num <- c("start_station_id","end_station_id")

df1[cols.num] <- sapply(df[cols.num],as.character)
```
I was able to import a couple csvs at a time and would merge them after the field type change then export:
```
# merge datasets together
merged <- rbind(df, df1)

#exported csv with converted columns
write.csv(df_dropped,"/cloud/project/bike_data/trimmed-day_phase-202010-divvy-tripdata.csv", row.names = FALSE)
```
I had to be very careful in my naming to remember what was done to the exported set and for how many months.

I started my EDA phase of investigating the data:
```
#view rows, columns, type, and sample
glimpse(df)

#count number of rows
count(df)
```
Then it was time to clean the nulls
```
##Remove nulls
is.na(df)

#Total number of nulls
sum(is.na(df))

#find the nulls
row.has.na <- apply(df, 1, function(x){any(is.na(x))})

#See how many rows will be removed
sum(row.has.na)

#remove the nulls
df <- df[!row.has.na,]

#Confirm rows are removed
count(df)
```

I knew I'd want a ride duration column, so whipped that up:
```
#Find duration of ride time
df$ride_duration <- difftime(df$ended_at,df$started_at)
```
I realized there were some rides lasting 8, 12, 24+ hours.  
```
#There are hundreds greater than 8 hours (28800 seconds)
sort(df$ride_duration,decreasing=TRUE)
```
I would exclude longer rides from my analysis because I felt anything over 8 hours was an accident and bad data.

Seeing the data in chart form is obviously a benefit, so I threw some plots at it;
```
#generates a beautiful web page with multiple reports and graphs of the data
DataExplorer::create_report(df)

#compare rider type to bike type
barplot(df, member_casual = "X-axis", rideable_type = "Y-axis", main ="Bar-Chart")

#boxplot graph showing that members take shorter rides than casual
#See aggregate above for exact average number
#My theory here is members know exactly where they want to go, eg work or
#the store, so they're rides are short and direct.  Casual are taking a 
#more leisurely ride.
#average ride time
df %>%
filter(ride_duration < 4000) %>%
  ggplot(aes(member_casual,ride_duration)) +
  geom_boxplot()
```
The charts confirmed my thoughts from Gsheets, but with more data.

I wanted to get some precise numbers, so did some calculations:
```
#Determine average ride duration
aggregate(df$ride_duration, list(df$member_casual), FUN=mean)

RESULTS
1  casual 1912.2909 secs
2  member 664.4842 secs

#Find total number of aggregate and as percentage of the whole
##Example is elements for members & casual as a percentage

#first install this package
install.packages("formattable")

#then run
g <- df %>%
  group_by(member_casual, rideable_type) %>%
  summarise(cnt = n()) %>%
  mutate(freq = formattable::percent(cnt / sum(cnt))) %>% 
  arrange(desc(freq))

View(g)

#Group by member type and bike type to see counts using SQL for fun
sqldf('select member_casual, rideable_type, count(*) as UNIQUE_COUNT from df group by member_casual, rideable_type')

RESULTS
  member_casual rideable_type UNIQUE_COUNT
1        casual  classic_bike      1120715
2        casual   docked_bike       407780
3        casual electric_bike       829792
4        member  classic_bike      1630116
5        member   docked_bike       270200
6        member electric_bike       877658
'

#Group by member type and bike type to see counts
member_bike_type_count <- df %>%
  count(member_casual,rideable_type)

#combine member and bike type into single column to graph against their counts  
member_bike_type_count <- unite(member_bike_type_count, combo, c(member_casual,rideable_type))
```
I wanted to see when departure times were:
```
#Generate day phase counts in two steps
#no need creating a empty day phase column and adding 1 for morning and 2 for afternoon

# Step 1: Split 'started_at' into two columns to do analysis on time of day
df <- separate(df, started_at, into = c("started_at_date", "started_at_time"), sep = " ")

# count values extracted in step above
# FALSE is morning and TRUE is afternoon
df %>%
  count(member_casual,started_at_time >= "12:00:00")

phase <- df %>%
  group_by(member_casual, day_phase) %>%
  summarise(cnt = n()) %>%
  mutate(freq = formattable::percent(cnt / sum(cnt))) %>% 
  arrange(desc(freq))

#Group by member_casual column with day_phase
day_phase_df <- df %>%
  count(member_casual,day_phase)

day_phase_df

#Create df for members
day_phase_df_member <- filter(day_phase_df, member_casual == "member")
#Create df for casuals
day_phase_df_casual <- filter(day_phase_df, member_casual == "casual")

day_phase_df_casual
day_phase_df_member

#Find percent of whole per for members
day_phase_df_member %>%
  group_by(member_casual) %>%
  mutate(percent = day_phase/sum(day_phase))

#Find percent of whole per for casuals
day_phase_df_casual %>%
  group_by(member_casual) %>%
  mutate(percent = day_phase/sum(day_phase))
```
I previously did some work in Jupyter notebooks and preferred that over Rstudio.

### Last it was over to Tableau!
Tableau was great for big data sets.  I was able to join all of them without removing any columns.

I created seven viz's; Average Ride Time, Day of Week for Casual and Members, Day Phase Heat Graph, Day Phase Line Chart Difference and Percentile, and Bike Type.
[Here is the public view of these](https://public.tableau.com/app/profile/grantland.gears/viz/CyclistBikeSharev2/AllVizs?publish=yes).  Below are specific breakdowns. 


## Average Ride Time
For this one I needed to create a ride duration measure by subtracking started_on from ended_on:

<img src="https://user-images.githubusercontent.com/70623337/141251180-d27f3667-06d7-4bd7-8cf3-b6ba00632b51.png" width="400">

Then I filtered out rides over 8 hours (480 minutes) by creating another measure:

<img src="https://user-images.githubusercontent.com/70623337/141251341-6f9aed74-d6dd-4219-ad01-019017d66c62.png" width="400">

Then I tossed that into a simple bar chart:

<img src="https://user-images.githubusercontent.com/70623337/141251409-7a44af9f-96f9-465c-ae50-557cb4319ffa.png" width="500">

## Day of Week for Casual and Members
For these vizs I couldn't figure out to display casual and members as their own line since they're in the same field, so I created two graphs and filtered for just one at a time then created a dashboard with both graphs:

<img src="https://user-images.githubusercontent.com/70623337/141253018-d48b4563-cd88-4241-ac10-3d860af1c9b7.png" width="600">


## Day Phase Heat Graph
I liked this graph a lot.  I knew members rode more around commute times and this graph made that very visible.  This is where the ease of Tableau is great.  I have a timestamp field with when a ride started.  I didn't have to do any fancy converting of timestamp to hours.  I just selected Date Part > Hours in the Custom Date selector and voila, you can see the clear commute trend for member riders:

<img src="https://user-images.githubusercontent.com/70623337/141253420-080d4ba8-4632-4e8b-ac30-d0c4789732da.png" width="400">


## Day Phase Line Chart Difference and Percentile
This was a continuation of the heat graph chart.  I wanted to look at the data in another way just for fun.  I was never satisfied with these and ultimately the heat graph provided the best visual.  For posterity here are the two line graphs I created of casual and member rides by hour of the day.  First is a difference graph with each hour being the difference from the previous hour:
<img src="https://user-images.githubusercontent.com/70623337/141254842-6e79f000-4c08-4a0a-bc67-32256eb6739e.png" width="500">


Second is a Percentile relative to the max:

<img src="https://user-images.githubusercontent.com/70623337/141254900-2942d25a-15c7-427f-857a-567dfea67825.png" width="500">

## Bike Type 
This one didn't reveal too much aside from members ride the classic_bike more than the other two.  So do casual riders, but members at a greater percentage:

<img src="https://user-images.githubusercontent.com/70623337/141255762-5f012ff1-687f-4760-b7c2-24638abdb524.png" width="500">

That's my story and I'm sticking to it ;).   Seriously though, 
