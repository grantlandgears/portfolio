# CYCLISTIC DATA STUDY

## This markdown contains the detailed steps I took in my analysis of the Cyclistic dataset.  The company is fictional as part of a Coursera / Google DA cert, but data is real - unsure from what company.

## The ask
Cyclisitic is looking to increase revenue by converting casual riders to subscribed members locking in regular monthly subscriptions.  The questions posed are:
1. How do annual members and casual riders use Cyclistic bikes differently?
2. Why would casual riders buy Cyclistic annual memberships?

## The Dataset(s)
The data came in 12 .csvs, one for each month from Oct 2020 to Sept 2021, so the data is quite up to date.  The .csvs ranged in size from 9MB to 150MB.  Total number of rows was 5 million.  I definitely had some challenges just being able to view all the data!

## EDA
The first place I went to was the fastest - Gsheets.  I pulled in the smallest monthly file because anything bigger choked up Gsheets.  The .csvs had up to hundreds of thousands of rows, but they ranged in sized.  I feel confident this is due to the seasons with fewer riders in the winter than summer.  Feb .csv was the smallest and July was the biggest.  That's neither here nor there, but just an observation.

I pulled in February data to Gsheets and started to get used to the data.  Normally I would prefer to go to SQL, but popping over to Gsheets was fastest.  I started to get familiar with the data.  There were 
