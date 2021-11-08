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
I pulled in February data to Gsheets and started to get used to the data.  Normally I would prefer to go to SQL, but popping over to Gsheets was fastest.  I started to get familiar with the data.  There were X columns: 

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
casual percentage afternoon		78.67%	  member percentage afternoon		69.22%
casual percentage morning		  21.33%	  member percentage morning		  30.78%
```

