# Bike Share Data Study

## Table of Contents
- [Introduction](#introduction)
- [Business Task](#business-task)
- [Data Sources](#data-sources)
- [Analyzing the Data](#analyzing-the-data)
- [Conclusion and Recommendations](#conclusion-and-recommendations)

## Introduction
This study is motivated by the capstone project as suggested by the Google Data Analytics Professional Certificate course, wherein analysts are asked to imagine a real-world scenario, and use the skills learned in the course to complete a business task.

In this scenario, a Chicago based bike-share company named “Cyclistic” wants to understand how casual riders and annual members use bikes differently. This information will be used to help inform the business goal of converting casual riders to annual members, since annual memberships are more profitable long-term.

The main question is: **How do annual members and casual riders use Cyclistic bikes differently?** We hope that by identifying trends in the way users utilize the bike-share service, we will be provided insights about the needs of the customers, which will then inform the marketing team’s strategy to gain more annual members. This analysis and information will be presented in a report-style document, including documentation of the analysis process as well as visualizations of key findings.

## Business Task
Compare the habits and trends of bike use between casual users and annual members. Provide the marketing team of the bike-share company with data-based recommendations to increase the number of annual memberships.

## Data Sources
For the purposes of this case study, the datasets to be used for this scenario are based on the data made available by Motivate International Inc. under the following license:
[click here](https://www.divvybikes.com/data-license-agreement). All of the available data from this company is accessible [here](https://divvy-tripdata.s3.amazonaws.com/index.html)

It is based on data from the City of Chicago’s bikeshare service called Divvy, operated by Lyft Bikes and Scooters, LLC. The license outlines the terms and conditions of this publicly available data. It should be noted that any personal information about its users are not included, maintaining the anonymity of its users. However, the data is reliable, current, and complete enough for the purposes of this project.

The data is stored in csv files, the most recent files containing a month’s worth of data while the oldest file which dates back to 2013 has an entire year’s worth of information. The data itself is organized in a wide format where every row is an independent entry about a singular bike ride, with a corresponding unique ride identification string. The data includes information such as:
- a unique trip id
- the date and time each bike trip began and ended
- the stations that the bike trip left from and arrived at (including station name, id, and GPS coordinates)
- the type of bike used (electric, classic, or docked)
- the type of user (member or casual user)

For this analysis, I will be using data from the past year, downloading 12 separate csv files, corresponding to one for each month. Altogether, there is a total of 5,677,610 rows and over a GB of data.

## Data Cleaning and Preparation

For this project, I decided to use data from the past year: December 2022 to November 2023. This should give us an expansive and current overview of user habits. I chose to use SQL for most of the analysis because it was suitable for the large size of the collective data. I began by making sure that each month’s data had the same number of columns with the same datatype. I also wanted to check that the data type matched the information in each column. I noticed that the columns with the start and end time were saved as variable character data type, but were in the date-time format: `YYYY-MM-DD HH:MI:SS`. I therefore converted this column into date-time data type for each month which is more appropriate and would more accurately reflect the information.
```SQL
ALTER TABLE 202311_tripdata
ALTER COLUMN started_at DATETIME;
```
The above SQL command was completed for the `ended_at` column as well, and for the rest of the months.

I decided to join all of the months together into one large table using union joins so that I could look at the data throughout the entire year. I store this information as a temporary table to use throughout the analysis.
```SQL
CREATE TABLE #year_tripdata (
ride_id varchar(50),
rideable_type varchar(50),
started_at datetime,
ended_at datetime,
start_station_name varchar(1024),
start_station_id varchar(50),
end_station_name varchar(1024),
end_station_id varchar(50),
start_lat varchar(50),
start_lng varchar(50),
end_lat varchar(50),
end_lng varchar(50),
member_casual varchar(50)
)

--- I insert the data from each month into the temporary table created above

INSERT INTO #year_tripdata
SELECT *
FROM "202311_tripdata"
UNION
SELECT *
FROM "202310_tripdata"
UNION
SELECT *
FROM "202309_tripdata"
UNION 
SELECT *
FROM "202308_tripdata"
UNION
SELECT *
FROM "202307_tripdata"
UNION
SELECT * 
FROM "202306_tripdata"
UNION
SELECT *
FROM "202305_tripdata"
UNION
SELECT *
FROM "202304_tripdata"
UNION
SELECT *
FROM "202303_tripdata"
UNION
SELECT *
FROM "202302_tripdata"
UNION
SELECT * 
FROM "202301_tripdata"
UNION
SELECT *
FROM "202212_tripdata"
```
Next, the `ride_id` column should be unique values, so I verified that there were no duplicate entries by counting the distinct ride ids and matching it with the number of entries.
```SQL
SELECT DISTINCT(COUNT(ride_id))
FROM #year_tripdata
```
After ensuring there were duplicates, I checked for missing data and found that there were many blanks in the latitude and longitude columns for both the start and end location. The following query is an example used for the `start_lat` column, but was repeated for the remaining columns.
```SQL
SELECT *
FROM #year_tripdata
WHERE start_lat IS NULL
```
After doing the same with several more columns, I discovered that there are also missing values in the station names and ids. However, I think the most significant differences that may help guide the stakeholders’ decisions are about the frequency and duration of trips. Therefore, continuing to use these data entries should not pose a problem since the station details are not necessary to complete this analysis. Additionally, the rest of the columns are complete, and the duration of the trips can be analyzed as a measure of time rather than distance using the start and end times of the bike ride.

If I want to use the columns with the start and end time, I need to ensure that there aren’t values that don’t make logical sense, such as negative time or unreasonably long trips. We take the difference between the end time and start time with the following query:
```SQL
SELECT *, DATEDIFF(minute, started_at, ended_at) AS trip_duration
FROM #year_tripdata
WHERE trip_duration < 0
```
The data is filtered to see only those that result in a negative value. Looking at the other columns, there is no real indication as to why these specific entries have negative time. My best guess is that the start time and end time could have been flipped upon entry of the data, possibly due to human error. I can choose to keep this data in the analysis without skewing our calculations by making sure to take the absolute value of the difference.

Next I want to see how realistic the highest and lowest ride durations are by sorting them in order.
```SQL
--- Looking at the length of trip in ascending order
SELECT *, ABS(DATEDIFF(minute, started_at, ended_at)) AS trip_duration
FROM #year_tripdata
ORDER BY trip_duration

--- Looking at length of trip in descending order
SELECT *, ABS(DATEDIFF(minute, started_at, ended_at)) AS trip_duration
FROM #year_tripdata
ORDER BY trip_duration
```
After looking at the data in ascending order, I thought it natural to exclude rides that were approximately 0 minutes, which was probably collected due to user error. These will be excluded from future analyses. By looking at the data in descending order, I noticed that a majority of the longest bike rides are associated with the ‘docked’ bike type and I realized that these are not actual rides taken by users, but information about how long a bike has been sitting unused. Going forward, docked bikes should be excluded from the analysis.

Then I check for any outliers in this information, so I want to create a bar histogram to see the distribution of ride duration.
```SQL
SELECT trip_duration, COUNT(trip_duration) AS number_of_trips
FROM(
	SELECT ABS(DATEDIFF(minute, started_at, ended_at)) AS trip_duration
	FROM #year_tripdata
	WHERE rideable_type <> 'docked_bike' AND trip_duration > 0
	) AS trip_difference
GROUP BY trip_duration
ORDER BY trip_duration
```
The resulting query gives me the number of trips that are approximately each minute long, which I can export into its own csv file and import into a spreadsheet to create my histogram.

![Total Number of Trips Made by Each Ride Duration Graph](https://github.com/aalemanp/BikeShareProject/assets/154280707/f6d2a9a2-d45b-4e0d-8603-f5201d4d9e4e)

Please note that the continued graph below is of a different scale along the y-axis.

![Total Number of Trips Made by Each Ride Duration Graph(continued)](https://github.com/aalemanp/BikeShareProject/assets/154280707/25b39bda-15f3-4e99-a300-b3f84ae25aaa)

With this visualization, one can easily spot a jump in the number of rides that last 25 hours long. After filtering the data to see what might be the cause with the following query:
```SQL
SELECT *, ABS(DATEDIFF(minute, started_at, ended_at)) AS trip_duration
FROM #year_tripdata
WHERE trip_duration = 1500
```
It only seems to happen with the classic bike type, and is only associated with casual users, probably signifying some kind of malfunction in a bike’s data collection. To me, this seems like a major error which could significantly skew the results of our analysis. To keep our analysis focused on the most relevant data, I decided to continue the analysis using data with a ride duration from 1 to 500 minutes, a generous cut-off point. This leaves us with 5,500,997 rows of data to use for our analysis.

## Analyzing the Data

Firstly, to recap, the business task was to identify the differences in bike use between casual riders and annual members. We can do this by solving for the following questions:
- What is the average yearly trip length of each user?
- What is the average monthly trip length of each user?
- How many trips do members and casual riders take every month?

I started by calculating the overall average ride duration for the year for each user type.
```SQL
SELECT member_casual, AVG(trip_duration) AS average_trip_duration
FROM (
	SELECT *, ABS(DATEDIFF(minute, started_at, ended_at)) AS trip_duration
	FROM #year_tripdata
	WHERE rideable_type <> 'docked_bike'
	) AS trip_difference
WHERE trip_duration BETWEEN 1 AND 500
GROUP BY member_casual
```
From this I found that casual users take longer bike rides on average (approximately 18 minutes) than members (approximately 12 minutes). We can calculate the averages for every month of each user type to see how it changes throughout the year. An example of this query calculated for members can be seen here, but is repeated for 'casual' users as well:
```SQL
SELECT month, AVG(trip_duration) AS average_trip_duration
FROM (
	SELECT *, ABS(DATEDIFF(minute, started_at, ended_at)) AS trip_duration, MONTH(started_at) AS month
	FROM #year_tripdata
	WHERE rideable_type <> 'docked_bike'
	) AS trip_difference
WHERE trip_duration BETWEEN 1 AND 500 AND member_casual = 'member'
GROUP BY month
ORDER BY month
```
I store this information onto a spreadsheet for reference and use it to create a bar graph of our results.

![Monthly Average Bike Trip Duration Graph](https://github.com/aalemanp/BikeShareProject/assets/154280707/4a680243-b742-4c41-ae8e-54ab61708006)

Here we see that casual users are consistently taking longer bike rides on average than annual members every month. There is also a significant difference between their minimum and maximum monthly averages. Using the table made in our spreadsheet, we can use simple functions to calculate the percent increase between the minimum and maximum averages.

For members, there is a 30% increase from their lowest ride time, an average of 10 minutes during November-March, to their highest ride time, averaging 13 minutes in July and August. On the other hand, casual users who average the lowest ride time of 11 minutes in December and January increase their ride duration by 82% during their maximum average of 20 minutes in July and September. This tells us that members have more consistent behaviour in regards to the length of their bike ride whereas casual users fluctuate more throughout the year.

In regards to quantity, there are over 3.5 million trips taken by members throughout the year compared to the 1.9 million trips taken by casual users, which we can extract from the following query:
```SQL
SELECT member_casual, COUNT(member_casual) AS number_of_trips
FROM (
	SELECT *, ABS(DATEDIFF(minute, started_at, ended_at)) AS trip_duration
	FROM #year_tripdata
	WHERE rideable_type <> 'docked_bike'
	) AS trip_difference
WHERE trip_duration BETWEEN 1 AND 500
GROUP BY member_casual
```
Once again, looking at the number of trips taken every month, we run the following query for both members and casual users.
```SQL
SELECT month, COUNT(member_casual) AS number_of_trips
FROM (
	SELECT *, ABS(DATEDIFF(minute, started_at, ended_at)) AS trip_duration, MONTH(started_at) AS month
	FROM #year_tripdata
	WHERE rideable_type <> 'docked_bike'
	) AS trip_difference
WHERE trip_duration BETWEEN 1 AND 500 AND member_casual = 'member'
GROUP BY month
ORDER BY month
```
Again, this data is stored onto the spreadsheet to create the graph belowe and to perform some more simple calculations.

![Number of Trips per Month Graph](https://github.com/aalemanp/BikeShareProject/assets/154280707/944b7e36-4da6-4735-b699-352ebd1ee49c)

It can easily be seen that, as expected, the summer months are the most popular time for the bikeshare service among both user types. More specifically, if we look at the months with the most and least number of bike rides, members took over 3.34 times more trips in August than they did in December. Even more significantly, casual users took about 8.2 times more bike rides in July than they did in January. This tells us that casual users are much more interested in using the bike share program specifically during the summer months than annual members.

Finally, I decided to check if there was any differences in the type of bike used between members and casual users. 
```SQL
SELECT rideable_type, COUNT(rideable_type) AS bike_count
FROM(
	SELECT *, ABS(DATEDIFF(minute, started_at, ended_at)) AS trip_duration
	FROM #year_tripdata
	WHERE rideable_type <> 'docked_bike'
	) AS trip_difference
WHERE trip_duration BETWEEN 1 AND 500 AND member_casual = 'casual'
GROUP BY rideable_type
```
Annual members appeared to use either the classic and electric bike equally. However, the casual users tended towards using the electric bike by just over 10% as seen in the pie charts below:

![Member User Bike Preference](https://github.com/aalemanp/BikeShareProject/assets/154280707/2925a10c-5a05-4c9e-8844-dc2148b7076a) ![Casual User Bike Preference](https://github.com/aalemanp/BikeShareProject/assets/154280707/e097c1ab-f6ec-495c-842a-7f8da86a479f)

## Conclusion and Recommendations

Overall, we can conclude that, on average, annual members take more bike rides and shorter trips than casual members. We also know that during summer months, there is an increase in both ride duration and number of rides compared to the rest of the year, especially with casual users. From this data, we can hypothesize that casual users are more interested in taking long, recreational bike rides during months with good weather and ease of use. Members appear to be more consistent in their bike use, most likely using the bikes for practical reasons such as their main form of transportation.

My suggestion to Cyclistic would be to offer a summer membership for casual users, who may be initially interested in biking more often as a hobby during these popular months. I believe that having unlimited access to the bike-share service will incentivize them to use the bikes for practical reasons beyond recreational use. This would be akin to having a “trial” of the membership that is normally only offered year-round. Then, at the end of their summer membership, they can be encouraged to continue their membership for the rest of the year, as a kind of “continuation” of the service they’ve already started and have been using. I believe that this offer will be more tempting since they will have just had a positive experience using the bikes. Furthermore, continuing the membership for the remaining eight months will feel like less of a commitment than the initial proposal of a twelve month membership.

Additionally, we also know that casual riders have a slight preference for electric bikes. It may be worth increasing the availability of electric bikes to ensure user satisfaction, therefore increasing the likelihood of continued use of the bike share program.
