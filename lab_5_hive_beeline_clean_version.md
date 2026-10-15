
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



In the SSH terminal, start from `/home/hadoop` and create a download script:

```shell
nano script.sh
```


Copy and paste the code below into *script.sh*. It downloads and decompresses both files, renames them, and previews the header and first data row. 

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


Save the file (`Ctrl+O`, then `Enter`) and exit nano (`Ctrl+X`). Run the script:

```shell
bash script.sh
```
or 

```shell
sh script.sh
```
 

<br>

# Use SQL-like Queries with Hive

With the CSV files downloaded, define Hive tables, load the data into HDFS, and run SQL queries.

<br>

## Step 1: Open the Beeline CLI  

Open Beeline and connect to HiveServer2 on the primary node as user `hadoop`:

```shell
beeline -u jdbc:hive2://localhost:10000 -n hadoop
```


Run the following SQL statements inside Beeline. Commands beginning with `!sh` run shell commands without leaving Beeline.



<br>

## Step 2: Create external tables

Before loading the data, define the columns and CSV format for each table. Creating a table defines its schema; it does not yet load the downloaded records.


Check out [this page](https://github.com/justinjiajia/bigdata_lab_2026/blob/main/airbnb_metadata.md) to know the meaning of each field for records in the two data files.

Create the `listings` table:

```sql
CREATE EXTERNAL TABLE listings (
    `id` STRING, `listing_url` STRING, `scrape_id` STRING, `last_scraped` STRING, `source` STRING, `name` STRING, `description` STRING, `neighborhood_overview` STRING, `picture_url` STRING, `host_id` STRING, `host_url` STRING, `host_profile_id` STRING, `host_profile_url` STRING, `host_name` STRING, `host_since` STRING, `hosts_time_as_user_years` STRING, `hosts_time_as_user_months` STRING, `hosts_time_as_host_years` STRING, `hosts_time_as_host_months` STRING, `host_location` STRING, `host_about` STRING, `host_response_time` STRING, `host_response_rate` STRING, `host_acceptance_rate` STRING, `host_is_superhost` STRING, `host_thumbnail_url` STRING, `host_picture_url` STRING, `host_neighbourhood` STRING, `host_listings_count` STRING, `host_total_listings_count` STRING, `host_verifications` STRING, `host_has_profile_pic` STRING, `host_identity_verified` STRING, `neighbourhood` STRING, `neighbourhood_cleansed` STRING, `neighbourhood_group_cleansed` STRING, `latitude` STRING, `longitude` STRING, `property_type` STRING, `room_type` STRING, `accommodates` STRING, `bathrooms` STRING, `bathrooms_text` STRING, `bedrooms` STRING, `beds` STRING, `amenities` STRING, `price` STRING, `price_quote_checkin_date` STRING, `price_quote_checkout_date` STRING, `price_quote_total_price` STRING, `price_quote_price_per_night` STRING, `price_quote_raw` STRING, `minimum_nights` STRING, `maximum_nights` STRING, `minimum_minimum_nights` STRING, `maximum_minimum_nights` STRING, `minimum_maximum_nights` STRING, `maximum_maximum_nights` STRING, `minimum_nights_avg_ntm` STRING, `maximum_nights_avg_ntm` STRING, `calendar_updated` STRING, `has_availability` STRING, `availability_30` STRING, `availability_60` STRING, `availability_90` STRING, `availability_365` STRING, `calendar_last_scraped` STRING, `number_of_reviews` INT, `number_of_reviews_ltm` INT, `number_of_reviews_l30d` INT, `availability_eoy` STRING, `number_of_reviews_ly` STRING, `estimated_occupancy_l365d` STRING, `estimated_revenue_l365d` STRING, `first_review` STRING, `last_review` STRING, `review_scores_rating` DOUBLE, `review_scores_accuracy` STRING, `review_scores_cleanliness` STRING, `review_scores_checkin` STRING, `review_scores_communication` STRING, `review_scores_location` STRING, `review_scores_value` STRING, `license` STRING, `instant_bookable` STRING, `calculated_host_listings_count` STRING, `calculated_host_listings_count_entire_homes` STRING, `calculated_host_listings_count_private_rooms` STRING, `calculated_host_listings_count_shared_rooms` STRING, `reviews_per_month` STRING
)
ROW FORMAT SERDE 'org.apache.hadoop.hive.serde2.OpenCSVSerde'
WITH SERDEPROPERTIES ("separatorChar" = ",", "quoteChar" = "\"", "escapeChar" = "\\")
LOCATION '/bigdata/airbnb/listings'
TBLPROPERTIES ("skip.header.line.count"="1");
```


Next, create the `reviews` table using the same CSV settings:

```sql
CREATE EXTERNAL TABLE reviews (`listing_id` STRING, `id` STRING, `date` STRING, `reviewer_id` STRING, `reviewer_name` STRING, `comments` STRING)
ROW FORMAT SERDE 'org.apache.hadoop.hive.serde2.OpenCSVSerde'
WITH SERDEPROPERTIES ("separatorChar" = ",", "quoteChar" = "\"", "escapeChar" = "\\")
LOCATION '/bigdata/airbnb/reviews'
TBLPROPERTIES ("skip.header.line.count"="1");
```

The table definitions use:

- `OpenCSVSerde`: reads CSV fields using the specified separator, quote, and escape characters.
- `LOCATION`: specifies the HDFS directory for each table.
- `skip.header.line.count`: skips the CSV header row.

Check the HDFS directories associated with the tables:


```shell
!sh hadoop fs -ls /bigdata/airbnb
```

You should see:

```shell
Found 2 items
drwxr-xr-x   - hadoop hdfsadmingroup          0 202x-xx-xx 15:04 /bigdata/airbnb/listings
drwxr-xr-x   - hadoop hdfsadmingroup          0 202x-xx-xx 14:56 /bigdata/airbnb/reviews
```

This output confirms that the two table directories exist in HDFS.

Also check that both tables are registered in Hive:

```sql
SHOW TABLES;
```

Note: External tables normally retain their data when dropped. Without `EXTERNAL`, Hive creates a **managed** table, whose data is normally deleted when the table is dropped.
 

  <br>

## Step 3: Load data onto HDFS

Now load the downloaded CSV files into the table directories. `LOCAL` refers to the local filesystem; `OVERWRITE` replaces any existing data in the target table.

```sql
LOAD DATA LOCAL INPATH '/home/hadoop/data/reviews.csv' OVERWRITE INTO TABLE reviews;
LOAD DATA LOCAL INPATH '/home/hadoop/data/listings.csv' OVERWRITE INTO TABLE listings;
```

Verify that the *listings* file has been loaded into HDFS:
  
```shell
!sh hadoop fs -ls /bigdata/airbnb/listings
```

You should see:

```shell
Found 1 items
-rw-r--r--   1 hadoop hdfsadmingroup   16566447 202x-xx-xx 16:49 /bigdata/airbnb/listings/listings.csv
```


## Step 4: Explore table schema and data


With the data loaded, inspect the table definitions and preview a few rows before running analytical queries.

Show the column names and data types:

```sql
DESCRIBE reviews;
```

Show a detailed table schema (including storage info):

```sql
DESCRIBE FORMATTED reviews;
```

Preview a few records. `LIMIT` restricts the output size:


```sql
SELECT * FROM reviews LIMIT 5;
SELECT id, name, neighbourhood, neighbourhood_cleansed FROM listings LIMIT 10;
```



## Step 5: Answer analytical queries


Now use filtering, aggregation, and joins to explore hosts, neighborhoods, and reviews.


### Question 1: "Which 10 neighborhoods have the most unique Superhosts?"

Count unique Superhosts in each neighborhood and display the top 10. `COUNT(DISTINCT host_id)` avoids counting a host multiple times within the same neighborhood.

```sql
SELECT neighbourhood_cleansed, COUNT(DISTINCT host_id) AS number_of_superhosts
FROM listings WHERE host_is_superhost = 't'
GROUP BY neighbourhood_cleansed
ORDER BY number_of_superhosts DESC
LIMIT 10;
```

The meanings of the involved fields:

- `host_is_superhost`: Superhost status; `'t'` means true and `'f'` means false. The query keeps only `'t'` records.
- `neighbourhood_cleansed`:	The standardized, official name of the district or neighborhood where the listing is located.
- `host_id`: A unique ID assigned by Airbnb to every user who hosts a property.
 
 


#### Question 2: "Do Superhost listings have higher average ratings and more reviews?"

Next, compare listings with and without Superhost status. Grouping by `host_is_superhost` gives the listing count, average rating, and average review count for each group.

```sql
SELECT host_is_superhost, COUNT(id) AS total_listings, AVG(review_scores_rating) AS avg_rating, AVG(number_of_reviews) AS avg_reviews
FROM listings WHERE host_is_superhost IN ('t', 'f')
GROUP BY host_is_superhost;
```



#### Question 3: "Which 10 neighborhoods have the most reviews?"

Join `listings` with `reviews` to count reviews by neighborhood. The join matches each review to its listing using `l.id = r.listing_id`. Review counts are a proxy for activity, not a direct count of guests or bookings.

```sql
SELECT  l.neighbourhood_cleansed, COUNT(r.listing_id) as total_reviews 
FROM listings l JOIN reviews r ON l.id = r.listing_id 
GROUP BY l.neighbourhood_cleansed
ORDER BY total_reviews DESC 
LIMIT 10;
```

#### Question 4: "Which 10 hosts have the most listings, and what room types do they offer?"



Finally, each city's exploration page includes a table showing the Airbnb hosts with the most listings and a breakdown of their listings by room type. 

<img width="400" src="https://github.com/user-attachments/assets/8087f294-7b54-4ea9-ab91-82e6e58ce70d" />

Let's try to reproduce this table.

Summarize each host's listings by room type. Each `CASE` expression returns 1 for a matching room type and 0 otherwise; `SUM` counts the matches. 

```sql
SELECT 
    host_id,
    host_name AS `Host Name`,
    SUM(CASE WHEN room_type = 'Entire home/apt' THEN 1 ELSE 0 END) AS `#Entire homes/apts`,
    SUM(CASE WHEN room_type = 'Private room' THEN 1 ELSE 0 END) AS `#private rooms`,
    SUM(CASE WHEN room_type = 'Shared room' THEN 1 ELSE 0 END) AS `#shared rooms`,
    SUM(CASE WHEN room_type = 'Hotel room' THEN 1 ELSE 0 END) AS `#hotel rooms`,
    COUNT(*) AS `#listings`
FROM listings
WHERE host_id IS NOT NULL AND host_id != ''
GROUP BY host_id, host_name
ORDER BY `#listings` DESC
LIMIT 10;
```

# Practice

Apply the same grouping and aggregation techniques to reproduce the output shown below: find the top 10 hosts who have the most listings.


<img width="400"   src="https://github.com/user-attachments/assets/c11e137f-5e96-41bc-8f06-acaeff3f268a" />

