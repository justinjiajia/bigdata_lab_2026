
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

Note: You can also check the detailed status by running `systemctl status hive-server2`. To exit the display mode and return to your terminal prompt, simply type `q`.

#### Understanding the Architecture:
 

`hive-server2`: Takes Hive queries from clients (like Beeline) and does the heavy lifting to execute those queries on the distributed cluster.

`hive-hcatalog-server` (Metastore Service): Acts as a middleman. `hive-server2` talks to it to validate tables and schemas. It translates requests and securely reads from or writes to the underlying relational database where the metadata is actually stored.
 

#### Exploring the Metastore Database (MariaDB)

On Amazon EMR, the relational database that Hive uses for storing its metadata is MariaDB (an open-source fork of MySQL). The actual Hive Metastore DB files are located in `/var/lib/mysql/hive` on the Master node.

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

Next, we'll get some real-world data to analyze.  

Data source (provided by Inside Airbnb): https://insideairbnb.com/get-the-data/ 

 <img width="800"   src="https://github.com/user-attachments/assets/d8d303d1-51ff-4c77-abd6-03b0dab6956c" />


```shell
nano script.sh
```

Copy and paste the code snippet below into the *script.sh* file. 

This script creates a data folder, downloads the listings and reviews datasets for New York City, and peeks at the first few lines.


```shell
#!/bin/bash

mkdir data
cd data

echo -e "Downloading listings.csv"
wget https://data.insideairbnb.com/united-states/ny/new-york-city/2026-02-13/visualisations/listings.csv

echo -e "\nPeeking into listings.csv"

head -n2 listings.csv

echo -e "\nDownloading reviews.csv"

wget https://data.insideairbnb.com/united-states/ny/new-york-city/2026-02-13/visualisations/reviews.csv

echo -e "\nPeeking into reviews.csv"
head -n2 reviews.csv
cd ..

echo -e "\nShow the downloaded files"

ls -lh data
```

Save the change and exit the editor. Then run the script:

```shell
bash script.sh
```
or 

```shell
sh script.sh
```
 



<br>

# Use SQL-like Queries with Hive


We will use Beeline, which is a JDBC client that connects to our HiveServer2 daemon, to query downloaded data. 



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
- `-n hadoop`: Specifies that we are connecting as the hadoop user.
- (Optional) `-hiveconf`: Lets us set Hive configuration properties at startup.
 




<br>

## Step 2: Creating External Tables


Before we can analyze our CSV files, we need to define their schemas in Hive.

### Create the listings table

Because Airbnb listing names often contain commas (e.g., "Cozy Apartment, Great View"), we use the OpenCSVSerde to parse the file correctly and ignore commas wrapped in quotes

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


### Create the reviews table

This file is simple, so we can use a standard delimiter format.

```sql
CREATE EXTERNAL TABLE reviews (`listing_id` STRING, `date` STRING)
ROW FORMAT DELIMITED FIELDS TERMINATED BY ','
LOCATION '/bigdata/airbnb/reviews'
TBLPROPERTIES ("skip.header.line.count"="1");
```

Notice that we pointed the tables to specific LOCATIONs in HDFS. 

Hive automatically creates these directories for us. 

Let's verify this using an HDFS shell command directly from Beeline:

```shell
!sh hadoop fs -ls /bigdata/airbnb
```

You should see that the directories corresponding to the two tables have been created:

```shell
Found 2 items
drwxr-xr-x   - hadoop hdfsadmingroup          0 2026-05-28 15:04 /bigdata/airbnb/listings
drwxr-xr-x   - hadoop hdfsadmingroup          0 2026-05-28 14:56 /bigdata/airbnb/reviews
```
 
Confirm the metadata about the 2 tables have been inserted into the Metastore:

```sql
SHOW TABLES;
```
  

## Step 3: Loading Data into the Tables

Now that the schemas and HDFS directories are ready, 
we need to push our downloaded CSV files from the local Linux filesystem into HDFS so Hive can process them.

```sql
LOAD DATA LOCAL INPATH '/home/hadoop/data/listings.csv' OVERWRITE INTO TABLE listings;
LOAD DATA LOCAL INPATH '/home/hadoop/data/reviews.csv' OVERWRITE INTO TABLE reviews;
```

You can verify that the data files have been successfully loaded into the distributed file system:
  
```shell
!sh hadoop fs -ls /bigdata/airbnb/listings
Found 1 items
-rw-r--r--   1 hadoop hdfsadmingroup    6425805 2026-05-28 16:53 /bigdata/airbnb/listings/listings.csv
```

```shell
!sh hadoop fs -ls /bigdata/airbnb/reviews;
Found 1 items
-rw-r--r--   1 hadoop hdfsadmingroup   22516069 2026-05-28 16:53 /bigdata/airbnb/reviews/reviews.csv
```


## Step 4: Exploring Schema and Data

Before running heavy analytical queries, a best practice is to inspect the schema and preview the data.

Show a brief schema of the reviews table:



Show a brief schema:

```sql
DESCRIBE reviews;
```

Show a detailed schema:

```sql
DESCRIBE FORMATTED reviews;
```

Preview the first 5 records to ensure the columns aligned correctly:

```sql
SELECT * FROM reviews LIMIT 5;
SELECT * FROM listings LIMIT 5;
```



## Step 5: Exploring Analytical Queries


Now comes the fun part! Let's run queries to answer to answer several analytics questions.


###  Question 1: What is the total market share of each room type?

This query helps us understand how the Airbnb market is divided between entire homes, private rooms, and shared rooms.
  
```sql
SELECT room_type, COUNT(id) AS total_listings
FROM listings WHERE room_type IS NOT NULL AND room_type != ''
GROUP BY room_type
ORDER BY total_listings DESC;
```


The meanings of the involved fields:
- `room_type`: Represents the category of the Airbnb listing (e.g., "Entire home/apt", "Private room", or "Shared room"). 
- `id`: The unique identifer assigned to every individual property listing on Airbnb.

(When you run this, you will see the execution logs showing Map and Reduce stages progressing, followed by the final output table). 

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

###  Question 2: Who are the top 10 busiest "Mega-Hosts" in the city?

Let's identify hosts who manage multiple properties and see how much guest feedback their properties have combined.


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
- `number_of_reviews`: The total number of reviews a single property has received over its lifetime. We `SUM()` this to get the host's grand total.

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


#### Question 3: Which 10 neighborhoods received the highest volume of tourist traffic specifically in the year 2023?

The listings table only shows the all-time total number of reviews. 

To find out what happened in a specific year, we must join the listings table with the reviews table.
 

```sql
SELECT l.neighbourhood, COUNT(r.listing_id) AS reviews_in_2023
FROM listings l JOIN reviews r ON l.id = r.listing_id
WHERE r.`date` LIKE '2023-%'
GROUP BY l.neighbourhood
ORDER BY reviews_in_2023 DESC
LIMIT 10;
```

Note: The word `date` is a reserved keyword in Hive (it is an actual data type, like `INT` or `STRING`). Wrap the word date in backticks to tell Hive that we are specifically referring to a column name and not the reserved keyword.


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

## Step 6: Saving Output to HDFS or Local Filesystem


We will often need to save the results of our analytical queries for reporting or visualization.


### Option 1: Save to a Custom HDFS Directory (Recommended)

This method securely writes the output back into the distributed file system.

```sql
INSERT OVERWRITE DIRECTORY "/bigdata/airbnb/output" ROW FORMAT DELIMITED FIELDS TERMINATED BY ','
SELECT l.neighbourhood, COUNT(r.listing_id) AS reviews_in_2023
FROM listings l JOIN reviews r ON l.id = r.listing_id
WHERE r.`date` LIKE '2023-%'
GROUP BY l.neighbourhood
ORDER BY reviews_in_2023 DESC;
```

You can confirm the output files were generated using:

```shell
!sh hadoop fs -ls /bigdata/airbnb/output
```

 

### Option 2: Save Output to a Local Filesystem Directory
 
If you want to extract the data directly to the local master node, you can use `LOCAL DIRECTORY`.

First, create the folder and grant everyone write permissions (`777`).
 

```shell
!sh mkdir -p /home/hadoop/output
!sh chmod 777 /home/hadoop/output
```

#### Why is this necessary?

The Client (Beeline): We logged in as the hadoop user.

The Coordinator (HiveServer2): This daemon permanently runs in the background as the hive user.

The Processing (Tez/YARN): Because `hive.server2.enable.doAs=true` is configured by default, HiveServer2 tells YARN, "Run this Tez job impersonating the hadoop user." The actual heavy lifting (reading the CSVs, joining them, writing to the temporary HDFS folder /tmp/hive/hadoop/...) is done securely on behalf of `hadoop`.

The Failure Point (The MoveTask): Once the job finishes, HiveServer2 attempts to copy the data from HDFS down to the local Linux filesystem. Crucially, this local copy is executed by the HiveServer2 JVM process, which is running as the `hive` user.

The hive user attempts to write into `/home/hadoop/output`. Linux blocks it with a "Permission Denied" error because hive does not have access to hadoop's private directory. Setting permissions to 777 bypasses this crash!


---
 
Now, run the query:

```sql
INSERT OVERWRITE LOCAL DIRECTORY "/home/hadoop/output" ROW FORMAT DELIMITED FIELDS TERMINATED BY ','
SELECT l.neighbourhood, COUNT(r.listing_id) AS reviews_in_2023
FROM listings l JOIN reviews r ON l.id = r.listing_id
WHERE r.`date` LIKE '2023-%'
GROUP BY l.neighbourhood
ORDER BY reviews_in_2023 DESC;
```

You can confirm the output is saved on your local machine using:

```shell
!sh ls /tmp/output
```



## Step 7: Exit Beeline

When you are finished with your session, exit Beeline by simply typing:

```shell
!quit
```



# Run Hive Queries on Tez (Optional)

**Apache Tez** is a high-performance, DAG-based execution engine that is often used as a drop-in replacement for MapReduce in Hive.

Tez supports a rich set of operators — including filters, joins, unions, group-bys, and sorts — and can pass intermediate results in-memory between stages. 

By skipping the heavy disk-writing bottlenecks of MapReduce, it can run complex Hive queries much faster.
 
 
If you are experimenting and want to force Hive to fall back to legacy MapReduce, you can pass the engine configuration when launching Beeline:

```shell
beeline -u jdbc:hive2://localhost:10000 -n hadoop -hiveconf hive.execution.engine=mr
```

To ensure that your jobs are submitted to the YARN cluster (rather than executing in a single local JVM process for small datasets), disable the local auto mode in your Beeline session:

```shell
set hive.exec.mode.local.auto=false;
```

 

