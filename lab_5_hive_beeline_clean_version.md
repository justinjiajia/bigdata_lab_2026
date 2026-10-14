
# EMR settings

- EMR release: 7.14.0 

- ![#f03c15](https://placehold.co/15x15/f03c15/f03c15.png) IMPORTANT: Application: Hadoop, Hive, and Tez
  
- Primary instance: type: `m4.large`, quantity: 1

- Core instance: type: `m4.large`, quantity: 3
  
- Software configurations
    ```json
    [
        {
            "classification":"core-site",
            "properties": {
                "hadoop.http.staticuser.user": "hadoop"
            }
        }
    ]
    ```


- Make sure the primary node's EC2 security group has a rule allowing for "SSH" from "Anywhere".


<br>

 
# Data preparation

Next, we'll get some real-world data to analyze.  

Data source: [Inside Airbnb](https://insideairbnb.com/get-the-data/)

<img width="800"   src="https://github.com/user-attachments/assets/ee78f2d9-e618-4ca7-9a8f-8457221221d9" />

 

```shell
nano script.sh
```

Copy and paste the code snippet below into the *script.sh* file. 


```shell
#!/bin/bash

mkdir data
cd data

echo -e "Downloading listings.csv\n"
wget https://github.com/justinjiajia/datafiles/raw/refs/heads/main/listings_clean.csv.gz
gunzip listings_clean.csv.gz
mv listings_clean.csv listings.csv
head -n2 listings.csv

echo -e "Downloading reviews.csv\n"
wget https://github.com/justinjiajia/datafiles/raw/refs/heads/main/reviews_clean.csv.gz
gunzip reviews_clean.csv.gz
mv reviews_clean.csv reviews.csv
head -n2 reviews.csv
cd ..

echo -e "Display the downloaded files\n"
ls -lh data
```


Save the change and get back to the shell. Then run:

```shell
bash script.sh
```
or 

```shell
sh script.sh
```
 



<br>

# Use SQL-like Queries with Hive

<br>

## Step 1: Open the Beeline CLI  

To launch Beeline with a custom configuration:

```shell
beeline -u jdbc:hive2://localhost:10000 -n hadoop
```





<br>

## Step 2: Create external tables

Check out [this page](https://github.com/justinjiajia/bigdata_lab_2026/blob/main/airbnb_metadata.md) to know the meaning of each field for records in the two data files.

Create the listings table:

```sql
CREATE EXTERNAL TABLE listings (
    `id` STRING, `listing_url` STRING, `scrape_id` STRING, `last_scraped` STRING, `source` STRING, `name` STRING, `description` STRING, `neighborhood_overview` STRING, `picture_url` STRING, `host_id` STRING, `host_url` STRING, `host_profile_id` STRING, `host_profile_url` STRING, `host_name` STRING, `host_since` STRING, `hosts_time_as_user_years` STRING, `hosts_time_as_user_months` STRING, `hosts_time_as_host_years` STRING, `hosts_time_as_host_months` STRING, `host_location` STRING, `host_about` STRING, `host_response_time` STRING, `host_response_rate` STRING, `host_acceptance_rate` STRING, `host_is_superhost` STRING, `host_thumbnail_url` STRING, `host_picture_url` STRING, `host_neighbourhood` STRING, `host_listings_count` STRING, `host_total_listings_count` STRING, `host_verifications` STRING, `host_has_profile_pic` STRING, `host_identity_verified` STRING, `neighbourhood` STRING, `neighbourhood_cleansed` STRING, `neighbourhood_group_cleansed` STRING, `latitude` STRING, `longitude` STRING, `property_type` STRING, `room_type` STRING, `accommodates` STRING, `bathrooms` STRING, `bathrooms_text` STRING, `bedrooms` STRING, `beds` STRING, `amenities` STRING, `price` STRING, `price_quote_checkin_date` STRING, `price_quote_checkout_date` STRING, `price_quote_total_price` STRING, `price_quote_price_per_night` STRING, `price_quote_raw` STRING, `minimum_nights` STRING, `maximum_nights` STRING, `minimum_minimum_nights` STRING, `maximum_minimum_nights` STRING, `minimum_maximum_nights` STRING, `maximum_maximum_nights` STRING, `minimum_nights_avg_ntm` STRING, `maximum_nights_avg_ntm` STRING, `calendar_updated` STRING, `has_availability` STRING, `availability_30` STRING, `availability_60` STRING, `availability_90` STRING, `availability_365` STRING, `calendar_last_scraped` STRING, `number_of_reviews` INT, `number_of_reviews_ltm` INT, `number_of_reviews_l30d` INT, `availability_eoy` STRING, `number_of_reviews_ly` STRING, `estimated_occupancy_l365d` STRING, `estimated_revenue_l365d` STRING, `first_review` STRING, `last_review` STRING, `review_scores_rating` DOUBLE, `review_scores_accuracy` STRING, `review_scores_cleanliness` STRING, `review_scores_checkin` STRING, `review_scores_communication` STRING, `review_scores_location` STRING, `review_scores_value` STRING, `license` STRING, `instant_bookable` STRING, `calculated_host_listings_count` STRING, `calculated_host_listings_count_entire_homes` STRING, `calculated_host_listings_count_private_rooms` STRING, `calculated_host_listings_count_shared_rooms` STRING, `reviews_per_month` STRING
)
ROW FORMAT SERDE 'org.apache.hadoop.hive.serde2.OpenCSVSerde'
WITH SERDEPROPERTIES ("separatorChar" = ",", "quoteChar" = "\"", "escapeChar" = "\\")
LOCATION '/bigdata/airbnb/listings'
TBLPROPERTIES ("skip.header.line.count"="1");
```


Create the reviews table:

```sql
CREATE EXTERNAL TABLE reviews (`listing_id` STRING, `id` STRING, `date` STRING, `reviewer_id` STRING, `reviewer_name` STRING, `comments` STRING)
ROW FORMAT SERDE 'org.apache.hadoop.hive.serde2.OpenCSVSerde'
WITH SERDEPROPERTIES ("separatorChar" = ",", "quoteChar" = "\"", "escapeChar" = "\\")
LOCATION '/bigdata/airbnb/reviews'
TBLPROPERTIES ("skip.header.line.count"="1");
```


```shell
!sh hadoop fs -ls /bigdata/airbnb
```

You should see:

```shell
Found 2 items
drwxr-xr-x   - hadoop hdfsadmingroup          0 202x-xx-xx 15:04 /bigdata/airbnb/listings
drwxr-xr-x   - hadoop hdfsadmingroup          0 202x-xx-xx 14:56 /bigdata/airbnb/reviews
```

This indicates that the directories that corresponds to the two tables have been created automatically.


```sql
SHOW TABLES;
```

Note:

- If you don't use the `EXTERNAL` keyword, Hive creates the table in **internal mode**, meaning:

  - Hive manages the data entirely, and stores the data in the default location in HDFS, e.g., */user/hive/warehouse/listings*.
  - Dropping the table will delete the underlying data stored in HDFS, e.g., */user/hive/warehouse/listings*.

   

  

  <br>

## Step 3: Load data onto HDFS

```sql
LOAD DATA LOCAL INPATH '/home/hadoop/data/reviews.csv' OVERWRITE INTO TABLE reviews;
LOAD DATA LOCAL INPATH '/home/hadoop/data/listings.csv' OVERWRITE INTO TABLE listings;
```

You can verify that the data files have been sucessfully loaded as follows:
  
```shell
!sh hadoop fs -ls /bigdata/airbnb/listings
```

You should see:

```shell
Found 1 items
-rw-r--r--   1 hadoop hdfsadmingroup   16566447 202x-xx-xx 16:49 /bigdata/airbnb/listings/listings.csv
```


## Step 4: Explore table schema and data



Show a brief table schema:

```sql
DESCRIBE reviews;
```

Show a detailed table schema (including storage info):

```sql
DESCRIBE FORMATTED reviews;
```

Display the first several records to have a preview:

```sql
SELECT * FROM reviews LIMIT 5;
SELECT * FROM listings LIMIT 2;
```



## Step 5: Answer analytical queries


### Question 1: "Which 10 neighborhoods have the highest concentration of top-tier Airbnb hosts?"

Specifically, we want to count the actual number of unique people who have earned *Superhost* status in each neighborhood, rank those neighborhoods from highest to lowest, and select the top 10 neighborhoods for display.

```sql
SELECT neighbourhood_cleansed, COUNT(DISTINCT host_id) AS number_of_superhosts
FROM listings WHERE host_is_superhost = 't'
GROUP BY neighbourhood_cleansed
ORDER BY number_of_superhosts DESC
LIMIT 10;
```


The meanings of the involved fields:

- `host_is_superhost`: A string flag indicating whether Airbnb has awarded the host "Superhost" status (a designation for highly rated, reliable hosts). We filter for 't' to ensure we are only looking at Superhosts.	't' (true)'f' (false)'' (empty/null)
- `neighbourhood_cleansed`:	The standardized, official name of the district or neighborhood where the listing is located.
- `host_id`: A unique ID assigned by Airbnb to every user who hosts a property.
 


#### Question 2: "Does being a "Superhost" actually correlate with better ratings and more reviews? "

```sql
SELECT  host_is_superhost, COUNT(id) AS total_listings, AVG(review_scores_rating) AS avg_rating, AVG(number_of_reviews) AS avg_reviews
FROM listings WHERE host_is_superhost IN ('t', 'f')
GROUP BY host_is_superhost;
```



#### Question 3: "Which 10 neighborhoods have the highest overall guest engagement and visitor activity?"

Since Airbnb does not explicitly tell us exactly how many people booked a stay in each area, we use reviews as a proxy for visitor foot traffic.

```sql
SELECT  l.neighbourhood_cleansed, COUNT(r.listing_id) as total_reviews 
FROM listings l JOIN reviews r ON l.id = r.listing_id 
GROUP BY l.neighbourhood_cleansed
ORDER BY total_reviews DESC 
LIMIT 10;
```


