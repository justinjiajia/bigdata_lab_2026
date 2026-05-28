
# EMR settings

- EMR release: 7.13.0 

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



# Services

Once you are connected to the Master node of your launched EMR cluster, run the following command to check the status of the Hive services:

```shell
$ systemctl --type=service | grep hive
```
The output should display

```shell
  hive-hcatalog-server.service                          loaded active running HCatalog server
  hive-server2.service                                  loaded active running Hive Server2
```
This indicates that these two Hive-related services are currently healthy and running in the background.

You can also try `systemctl status hive-server2` (later, to exit the display mode, type `q`).

#### Understanding the Architecture:
 

`hive-server2`: Takes Hive queries from clients (like Beeline) and does the heavy lifting to execute those queries on the distributed cluster.

`hive-hcatalog-server` (Metastore Service): Acts as a middleman. `hive-server2` talks to it to validate tables and schemas. It translates requests and securely reads from or writes to the underlying relational database where the metadata is actually stored.
 

#### Exploring the Metastore Database (MariaDB)

On Amazon EMR, the RDBMS that Hive uses for storing its metadata is MariaDB (an open-source fork of MySQL). The actual Hive Metastore DB files are located in `/var/lib/mysql/hive` on the Master node.

To explore this, we will log directly into the MariaDB database. 

First, retrieve the database password by searching the Hive configuration file:

```shell
$ grep -A 1 "javax.jdo.option.ConnectionPassword" /etc/hive/conf/hive-site.xml
```

Copy the password found inside the `<value>` tags from the output.

Next, log into the MariaDB shell:

```shell
$ mysql -u hive -p
```

Paste the password when prompted.

Once you are inside the MariaDB shell, you can explore the metadata tables that Hive uses behind the scenes:

```SQL
MariaDB [(none)]> USE hive;
MariaDB [hive]> SHOW TABLES;
```

 


# Data preparation

```shell
nano script.sh
```

Copy and paste the code snippet below into the *script.sh* file. 


```shell
#!/bin/bash

mkdir data
cd data

echo -e "Downloading listings.csv"
wget https://data.insideairbnb.com/united-states/ny/new-york-city/2026-02-13/visualisations/listings.csv
head -n2 listings.csv

echo -e "\nDownloading reviews.csv"

wget https://data.insideairbnb.com/united-states/ny/new-york-city/2026-02-13/visualisations/reviews.csv
head -n2 reviews.csv
cd ..

echo -e "\nShow the downloaded files"

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
    `id` BIGINT, `name` STRING, `host_id` BIGINT, `host_profile_id` STRING, `host_name` STRING, `neighbourhood_group` STRING, `neighbourhood` STRING,
    `latitude` STRING, `longitude` STRING, `room_type` STRING, `price` DOUBLE, `minimum_nights` INT, `number_of_reviews` INT, `last_review` STRING,
    `reviews_per_month` DOUBLE, `calculated_host_listings_count` INT, `availability_365` INT, `number_of_reviews_ltm` INT, `license` STRING
)
ROW FORMAT SERDE 'org.apache.hadoop.hive.serde2.OpenCSVSerde'
WITH SERDEPROPERTIES ("separatorChar" = ",", "quoteChar" = "\"", "escapeChar" = "\\")
LOCATION '/bigdata/airbnb/listings'
TBLPROPERTIES ("skip.header.line.count"="1");
```


Create the reviews table:

```sql
CREATE EXTERNAL TABLE reviews (`listing_id` STRING, `date` STRING)
ROW FORMAT DELIMITED FIELDS TERMINATED BY ','
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
LOAD DATA LOCAL INPATH '/home/hadoop/data/listings.csv' OVERWRITE INTO TABLE listings;
LOAD DATA LOCAL INPATH '/home/hadoop/data/reviews.csv' OVERWRITE INTO TABLE reviews;
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


### Question 1

#### What is the total market share of each room type?
 

```sql
SELECT room_type, COUNT(id) AS total_listings
FROM listings WHERE room_type IS NOT NULL AND room_type != ''
GROUP BY room_type
ORDER BY total_listings DESC;
```


The meanings of the involved fields:
- `room_type`: Represents the category of the Airbnb listing (e.g., "Entire home/apt", "Private room", or "Shared room"). 
- `id`: The unique identifer assigned to every individual property listing on Airbnb.

 

The output should look like the following:

```
INFO  : Compiling command(queryId=hive_20260528170018_18d96420-8218-4f57-a0a1-86fb8fd2ec57): SELECT  room_type, COUNT(id) AS total_listings
FROM listings WHERE room_type IS NOT NULL AND room_type != ''
GROUP BY room_type
ORDER BY total_listings DESC
INFO  : Concurrency mode is disabled, not creating a lock manager
INFO  : Semantic Analysis Completed (retrial = false)
INFO  : Returning Hive schema: Schema(fieldSchemas:[FieldSchema(name:room_type, type:string, comment:null), FieldSchema(name:total_listings, type:bigint, comment:null)], properties:null)
INFO  : Completed compiling command(queryId=hive_20260528170018_18d96420-8218-4f57-a0a1-86fb8fd2ec57); Time taken: 0.242 seconds
INFO  : Concurrency mode is disabled, not creating a lock manager
INFO  : Executing command(queryId=hive_20260528170018_18d96420-8218-4f57-a0a1-86fb8fd2ec57): SELECT  room_type, COUNT(id) AS total_listings
FROM listings WHERE room_type IS NOT NULL AND room_type != ''
GROUP BY room_type
ORDER BY total_listings DESC
INFO  : Query ID = hive_20260528170018_18d96420-8218-4f57-a0a1-86fb8fd2ec57
INFO  : Total jobs = 1
INFO  : Launching Job 1 out of 1
INFO  : Starting task [Stage-1:MAPRED] in serial mode
INFO  : Subscribed to counters: [] for queryId: hive_20260528170018_18d96420-8218-4f57-a0a1-86fb8fd2ec57
INFO  : Session is already open
INFO  : Dag name: SELECT  room_type, COUNT(id) AS total...DESC (Stage-1)
INFO  : Status: Running (Executing on YARN cluster with App id application_1779976941000_0004)

INFO  : Map 1: -/-      Reducer 2: 0/2  Reducer 3: 0/1
INFO  : Map 1: 0/1      Reducer 2: 0/2  Reducer 3: 0/1
INFO  : Map 1: 0(+1)/1  Reducer 2: 0/2  Reducer 3: 0/1
INFO  : Map 1: 0(+1)/1  Reducer 2: 0/2  Reducer 3: 0/1
INFO  : Map 1: 1/1      Reducer 2: 2/2  Reducer 3: 0(+1)/1
INFO  : Map 1: 1/1      Reducer 2: 2/2  Reducer 3: 1/1
INFO  : Completed executing command(queryId=hive_20260528170018_18d96420-8218-4f57-a0a1-86fb8fd2ec57); Time taken: 7.769 seconds
INFO  : OK
INFO  : Concurrency mode is disabled, not creating a lock manager
+------------------+-----------------+
|    room_type     | total_listings  |
+------------------+-----------------+
| Entire home/apt  | 19343           |
| Private room     | 16365           |
| Hotel room       | 334             |
| Shared room      | 249             |
+------------------+-----------------+
```

#### Question 2

##### Who are the top 10 busiest "Mega-Hosts" in the city (hosts who manage multiple properties), and how many reviews do their properties have combined? 

```sql
SELECT host_name, host_id, COUNT(id) AS total_properties_managed, SUM(number_of_reviews) AS total_host_reviews
FROM listings WHERE host_name IS NOT NULL
GROUP BY host_name, host_id
ORDER BY total_properties_managed DESC
LIMIT 10;
```


The meanings of the involved fields:

- `host_name`: The first name or profile name of the person managing the Airbnb listing.
- `host_id`: The unique identifier for the host.
- `number_of_reviews`: The total number of reviews a single property has received over its lifetime.

The output should look like the following:

```
INFO  : Compiling command(queryId=hive_20260528170329_6195c5b8-fb25-4448-aecf-d75f060f6152): SELECT host_name, host_id, COUNT(id) AS total_properties_managed, SUM(number_of_reviews) AS total_host_reviews
FROM listings WHERE host_name IS NOT NULL
GROUP BY host_name, host_id
ORDER BY total_properties_managed DESC
LIMIT 10
INFO  : Concurrency mode is disabled, not creating a lock manager
INFO  : Semantic Analysis Completed (retrial = false)
INFO  : Returning Hive schema: Schema(fieldSchemas:[FieldSchema(name:host_name, type:string, comment:null), FieldSchema(name:host_id, type:string, comment:null), FieldSchema(name:total_properties_managed, type:bigint, comment:null), FieldSchema(name:total_host_reviews, type:double, comment:null)], properties:null)
INFO  : Completed compiling command(queryId=hive_20260528170329_6195c5b8-fb25-4448-aecf-d75f060f6152); Time taken: 0.157 seconds
INFO  : Concurrency mode is disabled, not creating a lock manager
INFO  : Executing command(queryId=hive_20260528170329_6195c5b8-fb25-4448-aecf-d75f060f6152): SELECT host_name, host_id, COUNT(id) AS total_properties_managed, SUM(number_of_reviews) AS total_host_reviews
FROM listings WHERE host_name IS NOT NULL
GROUP BY host_name, host_id
ORDER BY total_properties_managed DESC
LIMIT 10
INFO  : Query ID = hive_20260528170329_6195c5b8-fb25-4448-aecf-d75f060f6152
INFO  : Total jobs = 1
INFO  : Launching Job 1 out of 1
INFO  : Starting task [Stage-1:MAPRED] in serial mode
INFO  : Subscribed to counters: [] for queryId: hive_20260528170329_6195c5b8-fb25-4448-aecf-d75f060f6152
INFO  : Session is already open
INFO  : Dag name: SELECT host_name, host_id, COUNT(id) AS...10 (Stage-1)
INFO  : Status: Running (Executing on YARN cluster with App id application_1779976941000_0004)

INFO  : Map 1: -/-      Reducer 2: 0/2  Reducer 3: 0/1
INFO  : Map 1: 0/1      Reducer 2: 0/2  Reducer 3: 0/1
INFO  : Map 1: 0/1      Reducer 2: 0/2  Reducer 3: 0/1
INFO  : Map 1: 0(+1)/1  Reducer 2: 0/2  Reducer 3: 0/1
INFO  : Map 1: 0(+1)/1  Reducer 2: 0/2  Reducer 3: 0/1
INFO  : Map 1: 1/1      Reducer 2: 0(+1)/2      Reducer 3: 0/1
INFO  : Map 1: 1/1      Reducer 2: 2/2  Reducer 3: 0(+1)/1
INFO  : Map 1: 1/1      Reducer 2: 2/2  Reducer 3: 1/1
INFO  : Completed executing command(queryId=hive_20260528170329_6195c5b8-fb25-4448-aecf-d75f060f6152); Time taken: 8.416 seconds
INFO  : OK
INFO  : Concurrency mode is disabled, not creating a lock manager
+----------------------+------------+---------------------------+---------------------+
|      host_name       |  host_id   | total_properties_managed  | total_host_reviews  |
+----------------------+------------+---------------------------+---------------------+
| Blueground           | 107434423  | 1210                      | 211.0               |
| Eugene               | 3223938    | 560                       | 132.0               |
| Luxury Bookings Fze  | 446820235  | 330                       | 0.0                 |
| Jeniffer             | 51501835   | 260                       | 1375.0              |
| Hiroki               | 19303369   | 251                       | 494.0               |
| Urban Furnished      | 162280872  | 246                       | 545.0               |
| Shogo                | 200239515  | 225                       | 363.0               |
| Momoyo               | 204704622  | 216                       | 328.0               |
| Nat                  | 35491667   | 214                       | 832.0               |
| Furnished Quarters   | 22541573   | 183                       | 149.0               |
+----------------------+------------+---------------------------+---------------------+
10 rows selected (8.627 seconds)
```


#### Question 3

##### Which 10 neighborhoods received the highest volume of tourist traffic (based on reviews) specifically in the year 2023?


The listings table only shows the all-time total number of reviews.  To find out what happened in a specific year, we must join the reviews table. 

```sql
SELECT l.neighbourhood, COUNT(r.listing_id) AS reviews_in_2023
FROM listings l JOIN reviews r ON l.id = r.listing_id
WHERE r.`date` LIKE '2023-%'
GROUP BY l.neighbourhood
ORDER BY reviews_in_2023 DESC
LIMIT 10;
```

The word `date` is a reserved keyword in Hive (it is an actual data type, like `INT` or `STRING`).
Wrap the word date in backticks to tell Hive that we are specifically referring to a column name and not the reserved keyword, 


The output should look like the following:

```shell
INFO  : Compiling command(queryId=hive_20260528171046_0b962aca-1c0c-447d-b4b5-a9cfaf1bf194): SELECT l.neighbourhood, COUNT(r.listing_id) AS reviews_in_2023
FROM listings l JOIN reviews r ON l.id = r.listing_id
WHERE r.`date` LIKE '2023-%'
GROUP BY l.neighbourhood
ORDER BY reviews_in_2023 DESC
LIMIT 10
INFO  : Concurrency mode is disabled, not creating a lock manager
INFO  : Semantic Analysis Completed (retrial = false)
INFO  : Returning Hive schema: Schema(fieldSchemas:[FieldSchema(name:l.neighbourhood, type:string, comment:null), FieldSchema(name:reviews_in_2023, type:bigint, comment:null)], properties:null)
INFO  : Completed compiling command(queryId=hive_20260528171046_0b962aca-1c0c-447d-b4b5-a9cfaf1bf194); Time taken: 0.305 seconds
INFO  : Concurrency mode is disabled, not creating a lock manager
INFO  : Executing command(queryId=hive_20260528171046_0b962aca-1c0c-447d-b4b5-a9cfaf1bf194): SELECT l.neighbourhood, COUNT(r.listing_id) AS reviews_in_2023
FROM listings l JOIN reviews r ON l.id = r.listing_id
WHERE r.`date` LIKE '2023-%'
GROUP BY l.neighbourhood
ORDER BY reviews_in_2023 DESC
LIMIT 10
INFO  : Query ID = hive_20260528171046_0b962aca-1c0c-447d-b4b5-a9cfaf1bf194
INFO  : Total jobs = 1
INFO  : Launching Job 1 out of 1
INFO  : Starting task [Stage-1:MAPRED] in serial mode
INFO  : Subscribed to counters: [] for queryId: hive_20260528171046_0b962aca-1c0c-447d-b4b5-a9cfaf1bf194
INFO  : Session is already open
INFO  : Dag name: SELECT l.neighbourhood, COUNT(r.listing...10 (Stage-1)
INFO  : Setting tez.task.scale.memory.reserve-fraction to 0.30000001192092896
INFO  : Tez session was closed. Reopening...
INFO  : Session re-established.
INFO  : Session re-established.
INFO  : Status: Running (Executing on YARN cluster with App id application_1779976941000_0005)

INFO  : Map 1: -/-      Map 2: -/-      Reducer 3: 0/2  Reducer 4: 0/1
INFO  : Map 1: 0/1      Map 2: 0/2      Reducer 3: 0/2  Reducer 4: 0/1
INFO  : Map 1: 0/1      Map 2: 0/2      Reducer 3: 0/2  Reducer 4: 0/1
INFO  : Map 1: 0(+1)/1  Map 2: 0/2      Reducer 3: 0/2  Reducer 4: 0/1
INFO  : Map 1: 0(+1)/1  Map 2: 0(+2)/2  Reducer 3: 0/2  Reducer 4: 0/1
INFO  : Map 1: 1/1      Map 2: 0(+2)/2  Reducer 3: 0/2  Reducer 4: 0/1
INFO  : Map 1: 1/1      Map 2: 1(+1)/2  Reducer 3: 0(+2)/2      Reducer 4: 0/1
INFO  : Map 1: 1/1      Map 2: 2/2      Reducer 3: 2/2  Reducer 4: 0(+1)/1
INFO  : Map 1: 1/1      Map 2: 2/2      Reducer 3: 2/2  Reducer 4: 1/1
INFO  : Completed executing command(queryId=hive_20260528171046_0b962aca-1c0c-447d-b4b5-a9cfaf1bf194); Time taken: 17.419 seconds
INFO  : OK
INFO  : Concurrency mode is disabled, not creating a lock manager
+---------------------+------------------+
|   l.neighbourhood   | reviews_in_2023  |
+---------------------+------------------+
| Bedford-Stuyvesant  | 14393            |
| Harlem              | 8008             |
| Midtown             | 6080             |
| Williamsburg        | 5783             |
| Crown Heights       | 5538             |
| Lower East Side     | 4915             |
| Bushwick            | 4647             |
| Hell's Kitchen      | 4395             |
| Chelsea             | 3773             |
| East Flatbush       | 3047             |
+---------------------+------------------+
10 rows selected (17.747 seconds)
```



 

