
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
 
With the cluster up and running, the next step is to get the data onto the primary node. 

We'll use a real-world dataset from Airbnb so that the queries later on produce meaningful results.

## About the data


<img width="800" src="https://github.com/user-attachments/assets/29e62040-a2ee-4a47-ba65-4162fbdb380a" />

&nbsp;

[Inside Airbnb](https://insideairbnb.com/get-the-data/) provides publicly available datasets for exploring Airbnb listings and activity in cities around the world. In this lab, we will use the Hong Kong listings and reviews datasets dated September 20. The listings data describes properties and their hosts, while the reviews data contains guest reviews linked to individual listings. Together, these datasets allow us to analyze hosts, room types, neighborhoods, and review activity using Hive.

## Download data


We'll create a shell script called *script.sh* to download, decompress, and preview both CSV files in one go.

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


Save the change (`Ctrl+O`, then `Enter`) and exit nano (`Ctrl+X`). 

Run the script once getting back to the SSH shell:

```shell
bash script.sh
```
or 

```shell
sh script.sh
```
 

<br>

# Use SQL-like queries with Hive

Now that the data is on the primary node, we need a way to query it. 

Hive provides a SQL-like interface over HDFS, and Beeline is the command-line client we'll use to talk to HiveServer2.


<br>

## Step 1: Open the Beeline CLI  

To launch Beeline with a custom configuration:

```shell
beeline -u jdbc:hive2://localhost:10000 -n hadoop
```

What the flags mean:

- `-u jdbc:hive2://localhost:10000`: connects to HiveServer2 running on the local machine (the primary node) on its default port `10000`.

- `-n hadoop`: authenticates as user `hadoop`, which is the default user on EMR and has permissions to create tables in HDFS.

Once Beeline starts, you'll see a `0: jdbc:hive2://localhost:10000>` prompt. All the SQL below is typed there.

 

<br>

## Step 2: Create external tables

Now we tell Hive how to interpret the CSV files. We define a schema and point it at a location in HDFS. Note that it does not yet load the downloaded records.


Check out [this page](https://github.com/justinjiajia/bigdata_lab_2026/blob/main/airbnb_metadata.md) to know the meaning of each field for records in the two data files.

Create the `listings` table:

```sql
CREATE EXTERNAL TABLE listings (
    `id` STRING, `listing_url` STRING, `scrape_id` STRING, `last_scraped` STRING, `source` STRING, `name` STRING, `description` STRING, `neighborhood_overview` STRING, `picture_url` STRING, `host_id` STRING, `host_url` STRING, `host_profile_id` STRING, `host_profile_url` STRING, `host_name` STRING, `host_since` STRING, `hosts_time_as_user_years` STRING, `hosts_time_as_user_months` STRING, `hosts_time_as_host_years` STRING, `hosts_time_as_host_months` STRING, `host_location` STRING, `host_about` STRING, `host_response_time` STRING, `host_response_rate` STRING, `host_acceptance_rate` STRING, `host_is_superhost` STRING, `host_thumbnail_url` STRING, `host_picture_url` STRING, `host_neighbourhood` STRING, `host_listings_count` STRING, `host_total_listings_count` STRING, `host_verifications` STRING, `host_has_profile_pic` STRING, `host_identity_verified` STRING, `neighbourhood` STRING, `neighbourhood_cleansed` STRING, `neighbourhood_group_cleansed` STRING, `latitude` STRING, `longitude` STRING, `property_type` STRING, `room_type` STRING, `accommodates` STRING, `bathrooms` STRING, `bathrooms_text` STRING, `bedrooms` STRING, `beds` STRING, `amenities` STRING, `price` STRING, `price_quote_checkin_date` STRING, `price_quote_checkout_date` STRING, `price_quote_total_price` STRING, `price_quote_price_per_night` STRING, `price_quote_raw` STRING, `minimum_nights` STRING, `maximum_nights` STRING, `minimum_minimum_nights` STRING, `maximum_minimum_nights` STRING, `minimum_maximum_nights` STRING, `maximum_maximum_nights` STRING, `minimum_nights_avg_ntm` STRING, `maximum_nights_avg_ntm` STRING, `calendar_updated` STRING, `has_availability` STRING, `availability_30` STRING, `availability_60` STRING, `availability_90` STRING, `availability_365` STRING, `calendar_last_scraped` STRING, `number_of_reviews` INT, `number_of_reviews_ltm` INT, `number_of_reviews_l30d` INT, `availability_eoy` STRING, `number_of_reviews_ly` STRING, `estimated_occupancy_l365d` STRING, `estimated_revenue_l365d` STRING, `first_review` STRING, `last_review` STRING, `review_scores_rating` DOUBLE, `review_scores_accuracy` STRING, `review_scores_cleanliness` STRING, `review_scores_checkin` STRING, `review_scores_communication` STRING, `review_scores_location` STRING, `review_scores_value` STRING, `license` STRING, `instant_bookable` STRING, `calculated_host_listings_count` STRING, `calculated_host_listings_count_entire_homes` STRING, `calculated_host_listings_count_private_rooms` STRING, `calculated_host_listings_count_shared_rooms` STRING, `reviews_per_month` STRING
)
ROW FORMAT SERDE 'org.apache.hadoop.hive.serde2.OpenCSVSerde'
WITH SERDEPROPERTIES ("separatorChar" = ",", "quoteChar" = "\"", "escapeChar" = "\\")
LOCATION '/<your ITSC account>/airbnb/listings'
TBLPROPERTIES ("skip.header.line.count"="1");
```


Next, create the `reviews` table using the same CSV settings:

```sql
CREATE EXTERNAL TABLE reviews (`listing_id` STRING, `id` STRING, `date` STRING, `reviewer_id` STRING, `reviewer_name` STRING, `comments` STRING)
ROW FORMAT SERDE 'org.apache.hadoop.hive.serde2.OpenCSVSerde'
WITH SERDEPROPERTIES ("separatorChar" = ",", "quoteChar" = "\"", "escapeChar" = "\\")
LOCATION '/<your ITSC account>/airbnb/reviews'
TBLPROPERTIES ("skip.header.line.count"="1");
```

The table definitions use:

- `OpenCSVSerde`: reads CSV fields using the specified separator, quote, and escape characters.
- `LOCATION`: specifies the HDFS directory for each table.
- `skip.header.line.count`: skips the CSV header row.


Let's confirm the tables were created and that their target directories exist in HDFS:




```shell
!sh hadoop fs -ls /<your ITSC account>/airbnb
```

> Tip: in Beeline, prefix a shell command with `!sh` to run it without leaving the client.


You should see:

```shell
Found 2 items
drwxr-xr-x   - hadoop hdfsadmingroup          0 202x-xx-xx 15:04 /<your ITSC account>/airbnb/listings
drwxr-xr-x   - hadoop hdfsadmingroup          0 202x-xx-xx 14:56 /<your ITSC account>/airbnb/reviews
```

This output confirms that the directories that corresponds to the two tables have been created automatically.

Also check that both tables are registered in Hive:

```sql
SHOW TABLES;
```

Note: External tables normally retain their data when dropped. Without `EXTERNAL`, Hive creates a **managed** table, whose data is normally deleted when the table is dropped.
 

  <br>

## Step 3: Load data onto HDFS

The tables exist and know where to look, but the data is still sitting on the primary node's local disk. 

This step copies it into HDFS so Hive can later read it in parallel across all the core nodes.



```sql
LOAD DATA LOCAL INPATH '/home/hadoop/data/reviews.csv' OVERWRITE INTO TABLE reviews;
LOAD DATA LOCAL INPATH '/home/hadoop/data/listings.csv' OVERWRITE INTO TABLE listings;
```

What the keywords mean:

- Why `LOCAL`? Without `LOCAL`, Hive would look for the file in HDFS. With `LOCAL`, it looks on the local filesystem of the node where Beeline is running.   

- `OVERWRITE` replaces any existing data in the target table. If you omit it, Hive will simply add the file alongside any existing ones, causing duplicates.

You can verify that the data files have been sucessfully loaded as follows:
 
  
```shell
!sh hadoop fs -ls /<your ITSC account>/airbnb/listings
!sh hadoop fs -ls /<your ITSC account>/airbnb/reviews
```



## Step 4: Explore table schema and data

Before running any analysis, it's good practice to confirm the schema and eyeball a few rows.  If something is misaligned, you'll catch it here rather than after writing a complicated query.
 

Show the column names and data types:

```sql
DESCRIBE reviews;
```

Show a detailed table schema (including storage info):

```sql
DESCRIBE FORMATTED reviews;
```

The `FORMATTED` variant prints extra metadata like the table location, SerDe class, and table type. It is useful when you want to double-check that Hive is reading the right directory with the right CSV settings.

Preview a few records. `LIMIT` restricts the output size:


```sql
SELECT * FROM reviews LIMIT 5;
SELECT id, name, neighbourhood, neighbourhood_cleansed FROM listings LIMIT 10;
```

> ⚠️ If you see odd rows where most columns are `NULL` and text from descriptions appears in unexpected columns, that is a sign that embedded newlines were not stripped from the CSV before loading. The cleaned files from the lab's data source have already been preprocessed, so this shouldn't happen here.


## Step 5: Answer analytical queries


With the data loaded and verified, we can now turn to the analytical questions. 

Each question below uses a different SQL technique — aggregation, conditional counting, and joins — to showcase Hive's capabilities.
 


### Question 1: "Which 10 neighborhoods have the most unique Superhosts?"

Count unique Superhosts in each neighborhood and display the top 10. `COUNT(DISTINCT host_id)` avoids counting a host multiple times within the same neighborhood. This matters because a host may run several listings in the same area.

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



Next, compare listings with and without Superhost status. Grouping by `host_is_superhost` gives the listing count, average rating, and average review count for each group. By comparing the two rows side-by-side, we can see whether the Superhost badge actually reflects better guest experiences.


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

Summarize each host's listings by room type. Each `CASE` expression returns 1 for a matching room type and 0 otherwise; `SUM` counts the matches. This is called conditional aggregation — it lets us compute several category counts in a single pass over the data, without needing a pivot operation.

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

