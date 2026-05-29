
 


# Data preparation

```shell
nano script.sh
```

Copy and paste the code snippet below into the *script.sh* file. 


```shell
#!/bin/bash

mkdir data
cd data

echo -e "Downloading listings.csv\n"

wget https://github.com/justinjiajia/datafiles/raw/refs/heads/main/HK_listings.csv.gz  -O listings.csv.gz
gunzip listings.csv.gz
head -n2 listings.csv

echo -e "Downloading reviews.csv\n"

wget https://github.com/justinjiajia/datafiles/raw/refs/heads/main/HK_reviews.csv.gz -O reviews.csv.gz
gunzip reviews.csv.gz
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

Note: 
- `beeline`: Launches the Beeline command-line tool.
- `-u`:	The flag that tells Beeline the URL for connection.
- `jdbc:hive2://`:	The protocol. It tells Beeline to use the standard JDBC driver specifically designed for HiveServer2.
- `localhost`:	This tells Beeline that the Hive server is running on the same machine where you are typing the command. (In a production environment, this would be the IP address or domain name of a remote master node, like *10.0.5.24*).
- `:10000`: The port number. Port 10000 is the universal default port that HiveServer2 listens on for incoming traffic.
- `-hiveconf` lets us set configuration properties at startup.
- Here, we're telling Hive where to store temporary files when it chooses to run MapReduce jobs locally.




```shell
beeline -u jdbc:hive2://localhost:10000 -hiveconf mapreduce.cluster.local.dir=/home/hadoop/tmp
```




<br>

## Step 2: Creating a Table (External vs Internal)


Create the listings table:

```sql
CREATE EXTERNAL TABLE listings (
    `id` STRING, `listing_url` STRING, `scrape_id` STRING, `last_scraped` STRING, `source` STRING, `name` STRING, `description` STRING, `neighborhood_overview` STRING, `picture_url` STRING, `host_id` STRING, `host_url` STRING, `host_name` STRING, `host_since` STRING, `host_location` STRING, `host_about` STRING, `host_response_time` STRING, `host_response_rate` STRING, `host_acceptance_rate` STRING, `host_is_superhost` STRING, `host_thumbnail_url` STRING, `host_picture_url` STRING, `host_neighbourhood` STRING, `host_listings_count` STRING, `host_total_listings_count` STRING, `host_verifications` STRING, `host_has_profile_pic` STRING, `host_identity_verified` STRING, `neighbourhood` STRING, `neighbourhood_cleansed` STRING, `neighbourhood_group_cleansed` STRING, `latitude` STRING, `longitude` STRING, `property_type` STRING, `room_type` STRING, `accommodates` STRING, `bathrooms` STRING, `bathrooms_text` STRING, `bedrooms` STRING, `beds` STRING, `amenities` STRING, `price` STRING, `minimum_nights` STRING, `maximum_nights` STRING, `minimum_minimum_nights` STRING, `maximum_minimum_nights` STRING, `minimum_maximum_nights` STRING, `maximum_maximum_nights` STRING, `minimum_nights_avg_ntm` STRING, `maximum_nights_avg_ntm` STRING, `calendar_updated` STRING, `has_availability` STRING, `availability_30` STRING, `availability_60` STRING, `availability_90` STRING, `availability_365` STRING, `calendar_last_scraped` STRING, `number_of_reviews` INT, `number_of_reviews_ltm` INT, `number_of_reviews_l30d` INT, `availability_eoy` STRING, `number_of_reviews_ly` STRING, `estimated_occupancy_l365d` STRING, `estimated_revenue_l365d` STRING, `first_review` STRING, `last_review` STRING, `review_scores_rating` DOUBLE, `review_scores_accuracy` STRING, `review_scores_cleanliness` STRING, `review_scores_checkin` STRING, `review_scores_communication` STRING, `review_scores_location` STRING, `review_scores_value` STRING, `license` STRING, `instant_bookable` STRING, `calculated_host_listings_count` STRING, `calculated_host_listings_count_entire_homes` STRING, `calculated_host_listings_count_private_rooms` STRING, `calculated_host_listings_count_shared_rooms` STRING, `reviews_per_month` STRING
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
drwxr-xr-x   - hadoop hdfsadmingroup          0 2026-05-28 15:04 /bigdata/airbnb/listings
drwxr-xr-x   - hadoop hdfsadmingroup          0 2026-05-28 14:56 /bigdata/airbnb/reviews
```

This indicates that the directories that corresponds to the two tables have benn created automatically.


```sql
SHOW TABLES;
```

Note:

- If you **don't** use the `EXTERNAL` keyword, Hive creates the table in **internal mode**, meaning:

  - Hive manages the data entirely.
  - Dropping the table will delete the underlying data stored in HDFS, e.g., `/user/hive/warehouse/books`.

   

  

  <br>

## Step 3

```sql
LOAD DATA LOCAL INPATH '/home/hadoop/data/reviews.csv' OVERWRITE INTO TABLE reviews;
LOAD DATA LOCAL INPATH '/home/hadoop/data/listings.csv' OVERWRITE INTO TABLE listings;
```

You can verify that the data files have been sucessfully loaded  as follows:
  
```shell
!sh hadoop fs -ls /bigdata/airbnb/listings
Found 1 items
-rw-r--r--   1 hadoop hdfsadmingroup   12877434 2026-05-28 15:14 /bigdata/airbnb/listings/listings.csv
```


## Step 3: Exploring Schema and Data



Show a brief schema:

```sql
DESCRIBE reviews;
```

Show a detailed schema (including storage info):

```sql
DESCRIBE FORMATTED reviews;
```

Display the first 5 records to have a preview:

```sql
SELECT * FROM reviews LIMIT 5;
SELECT * FROM listings LIMIT 5;
```



## Step 4: Exploring analytical queries


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
 

The output should look like the following:

```
INFO  : Compiling command(queryId=hive_20260528153109_8606a3a3-ab81-434c-87fd-2eef0ec5c88d): SELECT neighbourhood_cleansed, COUNT(DISTINCT host_id) AS number_of_superhosts
FROM listings WHERE host_is_superhost = 't'
GROUP BY neighbourhood_cleansed
ORDER BY number_of_superhosts DESC
LIMIT 10
INFO  : Concurrency mode is disabled, not creating a lock manager
INFO  : Semantic Analysis Completed (retrial = false)
INFO  : Returning Hive schema: Schema(fieldSchemas:[FieldSchema(name:neighbourhood_cleansed, type:string, comment:null), FieldSchema(name:number_of_superhosts, type:bigint, comment:null)], properties:null)
INFO  : Completed compiling command(queryId=hive_20260528153109_8606a3a3-ab81-434c-87fd-2eef0ec5c88d); Time taken: 1.749 seconds
INFO  : Concurrency mode is disabled, not creating a lock manager
INFO  : Executing command(queryId=hive_20260528153109_8606a3a3-ab81-434c-87fd-2eef0ec5c88d): SELECT neighbourhood_cleansed, COUNT(DISTINCT host_id) AS number_of_superhosts
FROM listings WHERE host_is_superhost = 't'
GROUP BY neighbourhood_cleansed
ORDER BY number_of_superhosts DESC
LIMIT 10
INFO  : Query ID = hive_20260528153109_8606a3a3-ab81-434c-87fd-2eef0ec5c88d
INFO  : Total jobs = 1
INFO  : Launching Job 1 out of 1
INFO  : Starting task [Stage-1:MAPRED] in serial mode
INFO  : Subscribed to counters: [] for queryId: hive_20260528153109_8606a3a3-ab81-434c-87fd-2eef0ec5c88d
INFO  : Tez session hasn't been created yet. Opening session
INFO  : Dag name: SELECT neighbourhood_cleansed, COUNT(DI...10 (Stage-1)
INFO  : Status: Running (Executing on YARN cluster with App id application_1779976941000_0001)

INFO  : Map 1: -/-      Reducer 2: 0/2  Reducer 3: 0/1
INFO  : Map 1: 0/1      Reducer 2: 0/2  Reducer 3: 0/1
INFO  : Map 1: 0/1      Reducer 2: 0/2  Reducer 3: 0/1
INFO  : Map 1: 0(+1)/1  Reducer 2: 0/2  Reducer 3: 0/1
INFO  : Map 1: 0(+1)/1  Reducer 2: 0/2  Reducer 3: 0/1
INFO  : Map 1: 1/1      Reducer 2: 0/2  Reducer 3: 0/1
INFO  : Map 1: 1/1      Reducer 2: 2/2  Reducer 3: 0(+1)/1
INFO  : Map 1: 1/1      Reducer 2: 2/2  Reducer 3: 1/1
INFO  : Completed executing command(queryId=hive_20260528153109_8606a3a3-ab81-434c-87fd-2eef0ec5c88d); Time taken: 29.99 seconds
INFO  : OK
INFO  : Concurrency mode is disabled, not creating a lock manager
+-------------------------+-----------------------+
| neighbourhood_cleansed  | number_of_superhosts  |
+-------------------------+-----------------------+
| Central & Western       | 71                    |
| Yau Tsim Mong           | 57                    |
| Islands                 | 36                    |
| Wan Chai                | 31                    |
| Eastern                 | 14                    |
| Sai Kung                | 7                     |
| Sham Shui Po            | 4                     |
| North                   | 4                     |
| Kowloon City            | 4                     |
| Kwun Tong               | 3                     |
+-------------------------+-----------------------+
10 rows selected (31.907 seconds)
```

#### Question 2: "Does being a "Superhost" actually correlate with better ratings and more reviews? "

```sql
SELECT  host_is_superhost, COUNT(id) AS total_listings, AVG(review_scores_rating) AS avg_rating, AVG(number_of_reviews) AS avg_reviews
FROM listings WHERE host_is_superhost IN ('t', 'f')
GROUP BY host_is_superhost;
```

The output should look like the following:
```
INFO  : Compiling command(queryId=hive_20260528154437_5b3fd264-3f9c-450e-aa16-4bdc13642b71): SELECT
host_is_superhost,
COUNT(id) AS total_listings,
AVG(review_scores_rating) AS avg_rating,
AVG(number_of_reviews) AS avg_reviews
FROM listings
WHERE host_is_superhost IN ('t', 'f')
GROUP BY host_is_superhost
INFO  : Concurrency mode is disabled, not creating a lock manager
INFO  : Semantic Analysis Completed (retrial = false)
INFO  : Returning Hive schema: Schema(fieldSchemas:[FieldSchema(name:host_is_superhost, type:string, comment:null), FieldSchema(name:total_listings, type:bigint, comment:null), FieldSchema(name:avg_rating, type:double, comment:null), FieldSchema(name:avg_reviews, type:double, comment:null)], properties:null)
INFO  : Completed compiling command(queryId=hive_20260528154437_5b3fd264-3f9c-450e-aa16-4bdc13642b71); Time taken: 0.304 seconds
INFO  : Concurrency mode is disabled, not creating a lock manager
INFO  : Executing command(queryId=hive_20260528154437_5b3fd264-3f9c-450e-aa16-4bdc13642b71): SELECT
host_is_superhost,
COUNT(id) AS total_listings,
AVG(review_scores_rating) AS avg_rating,
AVG(number_of_reviews) AS avg_reviews
FROM listings
WHERE host_is_superhost IN ('t', 'f')
GROUP BY host_is_superhost
INFO  : Query ID = hive_20260528154437_5b3fd264-3f9c-450e-aa16-4bdc13642b71
INFO  : Total jobs = 1
INFO  : Launching Job 1 out of 1
INFO  : Starting task [Stage-1:MAPRED] in serial mode
INFO  : Subscribed to counters: [] for queryId: hive_20260528154437_5b3fd264-3f9c-450e-aa16-4bdc13642b71
INFO  : Tez session hasn't been created yet. Opening session
INFO  : Dag name: SELECT
host_is_superhost...host_is_superhost (Stage-1)
INFO  : Status: Running (Executing on YARN cluster with App id application_1779976941000_0002)

INFO  : Map 1: -/-      Reducer 2: 0/2
INFO  : Map 1: 0/1      Reducer 2: 0/2
INFO  : Map 1: 0/1      Reducer 2: 0/2
INFO  : Map 1: 0(+1)/1  Reducer 2: 0/2
INFO  : Map 1: 0(+1)/1  Reducer 2: 0/2
INFO  : Map 1: 1/1      Reducer 2: 0(+1)/2
INFO  : Map 1: 1/1      Reducer 2: 2/2
INFO  : Completed executing command(queryId=hive_20260528154437_5b3fd264-3f9c-450e-aa16-4bdc13642b71); Time taken: 19.877 seconds
INFO  : OK
INFO  : Concurrency mode is disabled, not creating a lock manager
+--------------------+-----------------+--------------------+---------------------+
| host_is_superhost  | total_listings  |     avg_rating     |     avg_reviews     |
+--------------------+-----------------+--------------------+---------------------+
| t                  | 572             | 4.08372377622378   | 45.09440559440559   |
| f                  | 3138            | 2.685755258126196  | 12.281708094327596  |
+--------------------+-----------------+--------------------+---------------------+
2 rows selected (20.256 seconds)
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



The output should look like the following:

```shell
INFO  : Compiling command(queryId=hive_20260528160241_43f24398-ead4-4e70-b985-0eb885e42024): SELECT  l.neighbourhood_cleansed, COUNT(r.listing_id) as total_reviews
FROM listings l JOIN reviews r ON l.id = r.listing_id
GROUP BY l.neighbourhood_cleansed
ORDER BY total_reviews DESC
LIMIT 10
INFO  : Concurrency mode is disabled, not creating a lock manager
INFO  : Semantic Analysis Completed (retrial = false)
INFO  : Returning Hive schema: Schema(fieldSchemas:[FieldSchema(name:l.neighbourhood_cleansed, type:string, comment:null), FieldSchema(name:total_reviews, type:bigint, comment:null)], properties:null)
INFO  : Completed compiling command(queryId=hive_20260528160241_43f24398-ead4-4e70-b985-0eb885e42024); Time taken: 0.352 seconds
INFO  : Concurrency mode is disabled, not creating a lock manager
INFO  : Executing command(queryId=hive_20260528160241_43f24398-ead4-4e70-b985-0eb885e42024): SELECT  l.neighbourhood_cleansed, COUNT(r.listing_id) as total_reviews
FROM listings l JOIN reviews r ON l.id = r.listing_id
GROUP BY l.neighbourhood_cleansed
ORDER BY total_reviews DESC
LIMIT 10
INFO  : Query ID = hive_20260528160241_43f24398-ead4-4e70-b985-0eb885e42024
INFO  : Total jobs = 1
INFO  : Launching Job 1 out of 1
INFO  : Starting task [Stage-1:MAPRED] in serial mode
INFO  : Subscribed to counters: [] for queryId: hive_20260528160241_43f24398-ead4-4e70-b985-0eb885e42024
INFO  : Session is already open
INFO  : Dag name: SELECT  l.neighbourhood_cleansed, COUNT...10 (Stage-1)
INFO  : Setting tez.task.scale.memory.reserve-fraction to 0.30000001192092896
INFO  : Status: Running (Executing on YARN cluster with App id application_1779976941000_0003)

INFO  : Map 1: -/-      Map 2: -/-      Reducer 3: 0/2  Reducer 4: 0/1
INFO  : Map 1: 0/1      Map 2: 0/2      Reducer 3: 0/2  Reducer 4: 0/1
INFO  : Map 1: 0/1      Map 2: 0/2      Reducer 3: 0/2  Reducer 4: 0/1
INFO  : Map 1: 0/1      Map 2: 0(+1)/2  Reducer 3: 0/2  Reducer 4: 0/1
INFO  : Map 1: 0(+1)/1  Map 2: 0(+2)/2  Reducer 3: 0/2  Reducer 4: 0/1
INFO  : Map 1: 0(+1)/1  Map 2: 0(+2)/2  Reducer 3: 0/2  Reducer 4: 0/1
INFO  : Map 1: 0(+1)/1  Map 2: 0(+2)/2  Reducer 3: 0/2  Reducer 4: 0/1
INFO  : Map 1: 1/1      Map 2: 0(+2)/2  Reducer 3: 0/2  Reducer 4: 0/1
INFO  : Map 1: 1/1      Map 2: 1(+1)/2  Reducer 3: 0(+2)/2      Reducer 4: 0/1
INFO  : Map 1: 1/1      Map 2: 2/2      Reducer 3: 2/2  Reducer 4: 0(+1)/1
INFO  : Map 1: 1/1      Map 2: 2/2      Reducer 3: 2/2  Reducer 4: 1/1
INFO  : Completed executing command(queryId=hive_20260528160241_43f24398-ead4-4e70-b985-0eb885e42024); Time taken: 17.225 seconds
INFO  : OK
INFO  : Concurrency mode is disabled, not creating a lock manager
+---------------------------+----------------+
| l.neighbourhood_cleansed  | total_reviews  |
+---------------------------+----------------+
| NULL                      | 43987          |
| Yau Tsim Mong             | 38863          |
| Islands                   | 7292           |
| Central & Western         | 6574           |
| Wan Chai                  | 6389           |
| Kowloon City              | 1440           |
| Sai Kung                  | 1346           |
| Eastern                   | 1179           |
| Sham Shui Po              | 1071           |
| Sha Tin                   | 366            |
+---------------------------+----------------+
10 rows selected (17.62 seconds)
```

### Diagnosis: Why so many NULL entries:

```sql
SELECT id, name, neighbourhood, neighbourhood_cleansed FROM listings LIMIT 10;
```


```
INFO  : Compiling command(queryId=hive_20260528160619_924eb9eb-e200-4b91-a89b-31a4278b3105): SELECT id, name, neighbourhood, neighbourhood_cleansed
FROM listings
LIMIT 10
INFO  : Concurrency mode is disabled, not creating a lock manager
INFO  : Semantic Analysis Completed (retrial = false)
INFO  : Returning Hive schema: Schema(fieldSchemas:[FieldSchema(name:id, type:string, comment:null), FieldSchema(name:name, type:string, comment:null), FieldSchema(name:neighbourhood, type:string, comment:null), FieldSchema(name:neighbourhood_cleansed, type:string, comment:null)], properties:null)
INFO  : Completed compiling command(queryId=hive_20260528160619_924eb9eb-e200-4b91-a89b-31a4278b3105); Time taken: 0.12 seconds
INFO  : Concurrency mode is disabled, not creating a lock manager
INFO  : Executing command(queryId=hive_20260528160619_924eb9eb-e200-4b91-a89b-31a4278b3105): SELECT id, name, neighbourhood, neighbourhood_cleansed
FROM listings
LIMIT 10
INFO  : Completed executing command(queryId=hive_20260528160619_924eb9eb-e200-4b91-a89b-31a4278b3105); Time taken: 0.003 seconds
INFO  : OK
INFO  : Concurrency mode is disabled, not creating a lock manager
+----------------------------------------------------+------------------------------------------------+----------------+-------------------------+
|                         id                         |                      name                      | neighbourhood  | neighbourhood_cleansed  |
+----------------------------------------------------+------------------------------------------------+----------------+-------------------------+
| 103760                                             | Central Centre 5 min walk to/from Central MTR  |                | Central & Western       |
| 248140                                             | Bright Studio - Soho - Central HK              | NULL           | NULL                    |
| NULL                                               | NULL                                           | NULL           | NULL                    |
| my Wife and I have this little apartment for friends and family which we let to travelers to Hong Kong when its free.   | NULL                                           | NULL           | NULL                    |
| NULL                                               | NULL                                           | NULL           | NULL                    |
| We like to look after people and ensure that they enjoy their stay in Hong Kong.  We have someone who will visit every other day to keep the place clean and tidy.   | NULL                                           | NULL           | NULL                    |
| NULL                                               | NULL                                           | NULL           | NULL                    |
| We can arrange travel cards                        | NULL                                           | NULL           | NULL                    |
| NULL                                               | NULL                                           | NULL           | NULL                    |
| We will be available on the phone during your stay to help with whatever you need and make sure that you enjoy your stay.  If you need an unlocked phone for your stay we are happy to lend you one. | NULL                                           | NULL           | NULL                    |
+----------------------------------------------------+------------------------------------------------+----------------+-------------------------+
10 rows selected (0.163 seconds)
```

 

An Airbnb host wrote a long description for their apartment (Listing ID: 248140) and pressed the Enter key a few times to create paragraphs.

In the raw CSV file, those "Enters" are saved as newline characters (\n).

When Hive reads a file from HDFS, its base reader (TextInputFormat) strictly reads line-by-line. It completely ignores the CSV quote marks (") and violently chops the row in half every time it sees a newline.

The remaining paragraphs get pushed to the next row. 

Because they are the first thing on that new line, Hive dumps them straight into the first column (id), and all the other columns become NULL.

---

In Hadoop, there is a strict two-step process when reading data:

The `InputFormat`: Hadoop uses a built-in tool called `TextInputFormat` to fetch the file from HDFS.
Its only job is to chop the file into individual rows. It does this blindly by looking for the Enter key (\n).

The SerDe: Once chopped, the row is handed to Hive's OpenCSVSerde, which looks at the commas and quotes to assign columns.

Because the TextInputFormat chops the file before the SerDe even sees the quotes, the multiline paragraphs are already destroyed.
Hive simply cannot handle embedded newlines in CSV files natively.


 ### The PySpark Fix (The Modern Data Engineer Route)
If you want to keep all 75 columns, you have to use a tool that is smart enough to respect quotes before splitting the lines. Apache Spark can do this.

You can launch PySpark on your EMR master node and convert the messy CSV into a highly-optimized Parquet file in just three lines of code:

python
# Launch 'pyspark' in your terminal, then run:

```
# 1. Read the messy CSV, explicitly telling Spark it has multi-line quotes
df = spark.read.csv("hdfs:///bigdata/airbnb/listings/listings.csv", header=True, multiLine=True, escape='"')

# 2. Save it back to HDFS as a perfectly clean Parquet file
df.write.mode("overwrite").parquet("hdfs:///bigdata/airbnb/listings_clean_parquet/")

```

Once that is done, you create a Hive table pointed at that Parquet folder, and the data will be flawlessly structured with zero shifted columns!
