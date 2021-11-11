# CYCLISTIC DATA STUDY
<img src="https://drive.google.com/uc?export=view&id=1vaicvK3W1eCDOc7PG_tpRctlsRmLV5vZ" width="250" height="200">

## This markdown contains the detailed steps I took in my analysis of the Cyclistic dataset.  The company is fictional as part of a Coursera / Google DA cert, but data is real - unsure from what company.

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
``` th
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

I started my EDA phase of investigating the data.  I used a variety of commands to view/summarise data:
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

I then began cleaning and prepping the data.  I used both 'read' and 'fread' commands:
```
df <- read.csv("/cloud/project/bike_data/source_sets/202010-divvy-tripdata.csv")
df1 <- fread("/cloud/project/bike_data/source_sets/202010-divvy-tripdata.csv", select = c("rideable_type","started_at","member_casual"))
```
fread was great for loading a subset of columns without hogging all my memory.

There was a discrepancy in the csvs where two of them had their columns in a different format, so I needed to convert those:
```
glimpse(df)

#convert one column from int to character
df1$start_station_id <- as.character(as.numeric(df1$start_station_id))  

#convert multiple columns from int to character
cols.num <- c("start_station_id","end_station_id")

df1[cols.num] <- sapply(df[cols.num],as.character)
```
I was able to do a couple csvs at a time, so I would merge them after the change then export:
```
# merge datasets together
merged <- rbind(df, df1)

#exported csv with converted columns
write.csv(df_dropped,"/cloud/project/bike_data/trimmed-day_phase-202010-divvy-tripdata.csv", row.names = FALSE)
```

I previously did some work in Jupyter notebooks and preferred that over Rstudio.
