data preparation


```

#!/bin/bash
 
mkdir -p retail_data_lab
cd retail_data_lab


# Format: receipt_id, item_sku, sale_price, sale_date, store_id
 
cat << EOF > sales_1.csv
98401,SKU-A111,29.99,2023-10-14,NY-01
98402,SKU-A892,14.99,2023-10-15,NY-01
98403,SKU-B222,9.99,2023-10-16,CA-45
98404,SKU-C333,49.50,2023-11-01,NY-01
EOF
  
cat << EOF > sales_2.csv
98405,SKU-A892,14.99,2023-11-02,CA-45
98406,SKU-D444,19.99,2023-11-05,CA-45
98407,SKU-E555,5.99,2024-01-10,NY-01
98408,SKU-A111,29.99,2024-01-12,TX-99
EOF

hdfs dfs -mkdir -p /data/retail_raw/

 
hdfs dfs -put -f sales_1.csv /data/retail_raw/
hdfs dfs -put -f sales_2.csv /data/retail_raw/

 
 
hdfs dfs -ls /data/retail_raw/ 
```


launch beeline:

```
$ beeline -u jdbc:hive2://localhost:10000 -n hadoop
Connecting to jdbc:hive2://localhost:10000
Connected to: Apache Hive (version 3.1.3-amzn-22)
Driver: Hive JDBC (version 3.1.3-amzn-22)
Transaction isolation: TRANSACTION_REPEATABLE_READ
Beeline version 3.1.3-amzn-22 by Apache Hive
0: jdbc:hive2://localhost:10000> CREATE EXTERNAL TABLE retail_staging_raw (
. . . . . . . . . . . . . . . .>     receipt_id INT,
. . . . . . . . . . . . . . . .>     item_sku STRING,
. . . . . . . . . . . . . . . .>     sale_price DOUBLE,
. . . . . . . . . . . . . . . .>     sale_date STRING,  -- The raw data contains the exact full date
. . . . . . . . . . . . . . . .>     store_id STRING
. . . . . . . . . . . . . . . .> )
. . . . . . . . . . . . . . . .> ROW FORMAT DELIMITED FIELDS TERMINATED BY ','
. . . . . . . . . . . . . . . .> LOCATION '/data/retail_raw/';
INFO  : Compiling command(queryId=hive_20260601164907_37532375-4596-4f8a-abba-eaad7060d70f): CREATE EXTERNAL TABLE retail_staging_raw (
receipt_id INT,
item_sku STRING,
sale_price DOUBLE,
sale_date STRING,
store_id STRING
)
ROW FORMAT DELIMITED FIELDS TERMINATED BY ','
LOCATION '/data/retail_raw/'
INFO  : Concurrency mode is disabled, not creating a lock manager
INFO  : Semantic Analysis Completed (retrial = false)
INFO  : Returning Hive schema: Schema(fieldSchemas:null, properties:null)
INFO  : Completed compiling command(queryId=hive_20260601164907_37532375-4596-4f8a-abba-eaad7060d70f); Time taken: 0.747 seconds
INFO  : Concurrency mode is disabled, not creating a lock manager
INFO  : Executing command(queryId=hive_20260601164907_37532375-4596-4f8a-abba-eaad7060d70f): CREATE EXTERNAL TABLE retail_staging_raw (
receipt_id INT,
item_sku STRING,
sale_price DOUBLE,
sale_date STRING,
store_id STRING
)
ROW FORMAT DELIMITED FIELDS TERMINATED BY ','
LOCATION '/data/retail_raw/'
INFO  : Starting task [Stage-0:DDL] in serial mode
INFO  : Completed executing command(queryId=hive_20260601164907_37532375-4596-4f8a-abba-eaad7060d70f); Time taken: 0.433 seconds
INFO  : OK
INFO  : Concurrency mode is disabled, not creating a lock manager
No rows affected (1.583 seconds)



0: jdbc:hive2://localhost:10000> CREATE External TABLE retail_partitioned_clean (
. . . . . . . . . . . . . . . .>     receipt_id INT,
. . . . . . . . . . . . . . . .>     item_sku STRING,
. . . . . . . . . . . . . . . .>     sale_price DOUBLE,
. . . . . . . . . . . . . . . .>     sale_date STRING   -- Keep the exact date inside the file!
. . . . . . . . . . . . . . . .> )
. . . . . . . . . . . . . . . .> PARTITIONED BY (store_id STRING, sale_month STRING)
. . . . . . . . . . . . . . . .> ROW FORMAT DELIMITED FIELDS TERMINATED BY ','
. . . . . . . . . . . . . . . .> LOCATION '/data/retail_partitioned/';
INFO  : Compiling command(queryId=hive_20260601165341_56ab0cd3-ab73-4222-8dd9-4338a268df6e): CREATE External TABLE retail_partitioned_clean (
receipt_id INT,
item_sku STRING,
sale_price DOUBLE,
sale_date STRING
)
PARTITIONED BY (store_id STRING, sale_month STRING)
ROW FORMAT DELIMITED FIELDS TERMINATED BY ','
LOCATION '/data/retail_partitioned/'
INFO  : Concurrency mode is disabled, not creating a lock manager
INFO  : Semantic Analysis Completed (retrial = false)
INFO  : Returning Hive schema: Schema(fieldSchemas:null, properties:null)
INFO  : Completed compiling command(queryId=hive_20260601165341_56ab0cd3-ab73-4222-8dd9-4338a268df6e); Time taken: 0.029 seconds
INFO  : Concurrency mode is disabled, not creating a lock manager
INFO  : Executing command(queryId=hive_20260601165341_56ab0cd3-ab73-4222-8dd9-4338a268df6e): CREATE External TABLE retail_partitioned_clean (
receipt_id INT,
item_sku STRING,
sale_price DOUBLE,
sale_date STRING
)
PARTITIONED BY (store_id STRING, sale_month STRING)
ROW FORMAT DELIMITED FIELDS TERMINATED BY ','
LOCATION '/data/retail_partitioned/'
INFO  : Starting task [Stage-0:DDL] in serial mode
INFO  : Completed executing command(queryId=hive_20260601165341_56ab0cd3-ab73-4222-8dd9-4338a268df6e); Time taken: 0.106 seconds
INFO  : OK
INFO  : Concurrency mode is disabled, not creating a lock manager
No rows affected (0.156 seconds)

0: jdbc:hive2://localhost:10000> !sh hadoop fs -ls /data
Found 2 items
drwxr-xr-x   - hadoop hdfsadmingroup          0 2026-06-01 16:53 /data/retail_partitioned
drwxr-xr-x   - hadoop hdfsadmingroup          0 2026-06-01 16:46 /data/retail_raw

0: jdbc:hive2://localhost:10000> !sh hadoop fs -ls /data/retail_partitioned

0: jdbc:hive2://localhost:10000> SET hive.exec.dynamic.partition = true;
No rows affected (0.011 seconds)

0: jdbc:hive2://localhost:10000> SET hive.exec.dynamic.partition.mode = nonstrict;
No rows affected (0.008 seconds)

0: jdbc:hive2://localhost:10000> INSERT OVERWRITE TABLE retail_partitioned_clean
. . . . . . . . . . . . . . . .> PARTITION (store_id, sale_month)
. . . . . . . . . . . . . . . .> SELECT 
. . . . . . . . . . . . . . . .>     receipt_id, 
. . . . . . . . . . . . . . . .>     item_sku, 
. . . . . . . . . . . . . . . .>     sale_price, 
. . . . . . . . . . . . . . . .>     sale_date, 
. . . . . . . . . . . . . . . .>     store_id,                               -- The Top-Level Folder
. . . . . . . . . . . . . . . .>     substr(sale_date, 1, 7) AS sale_month   -- The Sub-Folder (Extracts '2023-10')
. . . . . . . . . . . . . . . .> FROM retail_staging_raw;
INFO  : Compiling command(queryId=hive_20260601165613_cd821a61-65db-4580-b22d-dbdd3bf8ba3b): INSERT OVERWRITE TABLE retail_partitioned_clean
PARTITION (store_id, sale_month)
SELECT
receipt_id,
item_sku,
sale_price,
sale_date,
store_id,
substr(sale_date, 1, 7) AS sale_month
FROM retail_staging_raw
INFO  : Concurrency mode is disabled, not creating a lock manager
INFO  : Semantic Analysis Completed (retrial = false)
INFO  : Returning Hive schema: Schema(fieldSchemas:[FieldSchema(name:receipt_id, type:int, comment:null), FieldSchema(name:item_sku, type:string, comment:null), FieldSchema(name:sale_price, type:double, comment:null), FieldSchema(name:sale_date, type:string, comment:null), FieldSchema(name:store_id, type:string, comment:null), FieldSchema(name:sale_month, type:string, comment:null)], properties:null)
INFO  : Completed compiling command(queryId=hive_20260601165613_cd821a61-65db-4580-b22d-dbdd3bf8ba3b); Time taken: 2.92 seconds
INFO  : Concurrency mode is disabled, not creating a lock manager
INFO  : Executing command(queryId=hive_20260601165613_cd821a61-65db-4580-b22d-dbdd3bf8ba3b): INSERT OVERWRITE TABLE retail_partitioned_clean
PARTITION (store_id, sale_month)
SELECT
receipt_id,
item_sku,
sale_price,
sale_date,
store_id,
substr(sale_date, 1, 7) AS sale_month
FROM retail_staging_raw
INFO  : Query ID = hive_20260601165613_cd821a61-65db-4580-b22d-dbdd3bf8ba3b
INFO  : Total jobs = 1
INFO  : Launching Job 1 out of 1
INFO  : Starting task [Stage-1:MAPRED] in serial mode
INFO  : Subscribed to counters: [] for queryId: hive_20260601165613_cd821a61-65db-4580-b22d-dbdd3bf8ba3b
INFO  : Tez session hasn't been created yet. Opening session
INFO  : Dag name: INSERT OVERWRITE TABLE ...retail_staging_raw (Stage-1)
INFO  : Status: Running (Executing on YARN cluster with App id application_1780331932332_0001)

INFO  : Map 1: -/-      Reducer 2: 0/2  Reducer 3: 0/2
INFO  : Map 1: 0/2      Reducer 2: 0/2  Reducer 3: 0/2
INFO  : Map 1: 0/2      Reducer 2: 0/2  Reducer 3: 0/2
INFO  : Map 1: 0(+1)/2  Reducer 2: 0/2  Reducer 3: 0/2
INFO  : Map 1: 0(+2)/2  Reducer 2: 0/2  Reducer 3: 0/2
INFO  : Map 1: 1(+1)/2  Reducer 2: 0/2  Reducer 3: 0(+1)/2
INFO  : Map 1: 2/2      Reducer 2: 0(+1)/2      Reducer 3: 0(+1)/2
INFO  : Map 1: 2/2      Reducer 2: 1(+0)/2      Reducer 3: 1(+0)/2
INFO  : Map 1: 2/2      Reducer 2: 1(+0)/2      Reducer 3: 1(+1)/2
INFO  : Map 1: 2/2      Reducer 2: 1(+1)/2      Reducer 3: 1(+1)/2
INFO  : Map 1: 2/2      Reducer 2: 1(+1)/2      Reducer 3: 2/2
INFO  : Map 1: 2/2      Reducer 2: 2/2  Reducer 3: 2/2
INFO  : Starting task [Stage-2:DEPENDENCY_COLLECTION] in serial mode
INFO  : Starting task [Stage-0:MOVE] in serial mode
INFO  : Loading data to table default.retail_partitioned_clean partition (store_id=null, sale_month=null) from hdfs://ip-172-31-85-235.ec2.internal:8020/data/retail_partitioned/.hive-staging_hive_2026-06-01_16-56-13_106_4058393959094558622-1/-ext-10000
INFO  : 

INFO  :          Time taken to load dynamic partitions: 0.905 seconds
INFO  :          Time taken for adding to write entity : 0.002 seconds
INFO  : Starting task [Stage-3:STATS] in serial mode
INFO  : Executing stats task
INFO  : Partition {store_id=TX-99, sale_month=2024-01} stats: [numFiles=1, numRows=1, totalSize=32, rawDataSize=31]
INFO  : Partition {store_id=NY-01, sale_month=2023-10} stats: [numFiles=1, numRows=2, totalSize=64, rawDataSize=62]
INFO  : Partition {store_id=CA-45, sale_month=2023-10} stats: [numFiles=1, numRows=1, totalSize=31, rawDataSize=30]
INFO  : Partition {store_id=NY-01, sale_month=2024-01} stats: [numFiles=1, numRows=1, totalSize=31, rawDataSize=30]
INFO  : Partition {store_id=NY-01, sale_month=2023-11} stats: [numFiles=1, numRows=1, totalSize=31, rawDataSize=30]
INFO  : Partition {store_id=CA-45, sale_month=2023-11} stats: [numFiles=1, numRows=2, totalSize=64, rawDataSize=62]
INFO  : Completed executing command(queryId=hive_20260601165613_cd821a61-65db-4580-b22d-dbdd3bf8ba3b); Time taken: 35.38 seconds
INFO  : OK
INFO  : Concurrency mode is disabled, not creating a lock manager
No rows affected (38.333 seconds)

0: jdbc:hive2://localhost:10000> !sh hadoop fs -ls -R /data/retail_partitioned
drwxr-xr-x   - hadoop hdfsadmingroup          0 2026-06-01 16:56 /data/retail_partitioned/store_id=CA-45
drwxr-xr-x   - hadoop hdfsadmingroup          0 2026-06-01 16:56 /data/retail_partitioned/store_id=CA-45/sale_month=2023-10
-rw-r--r--   1 hadoop hdfsadmingroup         31 2026-06-01 16:56 /data/retail_partitioned/store_id=CA-45/sale_month=2023-10/000001_0
drwxr-xr-x   - hadoop hdfsadmingroup          0 2026-06-01 16:56 /data/retail_partitioned/store_id=CA-45/sale_month=2023-11
-rw-r--r--   1 hadoop hdfsadmingroup         64 2026-06-01 16:56 /data/retail_partitioned/store_id=CA-45/sale_month=2023-11/000001_0
drwxr-xr-x   - hadoop hdfsadmingroup          0 2026-06-01 16:56 /data/retail_partitioned/store_id=NY-01
drwxr-xr-x   - hadoop hdfsadmingroup          0 2026-06-01 16:56 /data/retail_partitioned/store_id=NY-01/sale_month=2023-10
-rw-r--r--   1 hadoop hdfsadmingroup         64 2026-06-01 16:56 /data/retail_partitioned/store_id=NY-01/sale_month=2023-10/000000_0
drwxr-xr-x   - hadoop hdfsadmingroup          0 2026-06-01 16:56 /data/retail_partitioned/store_id=NY-01/sale_month=2023-11
-rw-r--r--   1 hadoop hdfsadmingroup         31 2026-06-01 16:56 /data/retail_partitioned/store_id=NY-01/sale_month=2023-11/000001_0
drwxr-xr-x   - hadoop hdfsadmingroup          0 2026-06-01 16:56 /data/retail_partitioned/store_id=NY-01/sale_month=2024-01
-rw-r--r--   1 hadoop hdfsadmingroup         31 2026-06-01 16:56 /data/retail_partitioned/store_id=NY-01/sale_month=2024-01/000001_0
drwxr-xr-x   - hadoop hdfsadmingroup          0 2026-06-01 16:56 /data/retail_partitioned/store_id=TX-99
drwxr-xr-x   - hadoop hdfsadmingroup          0 2026-06-01 16:56 /data/retail_partitioned/store_id=TX-99/sale_month=2024-01
-rw-r--r--   1 hadoop hdfsadmingroup         32 2026-06-01 16:56 /data/retail_partitioned/store_id=TX-99/sale_month=2024-01/000001_0


0: jdbc:hive2://localhost:10000> select * from retail_partitioned_clean where store_id='NY-01';
INFO  : Compiling command(queryId=hive_20260601170022_e4eceae7-4182-4f06-9c87-9276586d9f61): select * from retail_partitioned_clean where store_id='NY-01'
INFO  : Concurrency mode is disabled, not creating a lock manager
INFO  : Semantic Analysis Completed (retrial = false)
INFO  : Returning Hive schema: Schema(fieldSchemas:[FieldSchema(name:retail_partitioned_clean.receipt_id, type:int, comment:null), FieldSchema(name:retail_partitioned_clean.item_sku, type:string, comment:null), FieldSchema(name:retail_partitioned_clean.sale_price, type:double, comment:null), FieldSchema(name:retail_partitioned_clean.sale_date, type:string, comment:null), FieldSchema(name:retail_partitioned_clean.store_id, type:string, comment:null), FieldSchema(name:retail_partitioned_clean.sale_month, type:string, comment:null)], properties:null)
INFO  : Completed compiling command(queryId=hive_20260601170022_e4eceae7-4182-4f06-9c87-9276586d9f61); Time taken: 0.634 seconds
INFO  : Concurrency mode is disabled, not creating a lock manager
INFO  : Executing command(queryId=hive_20260601170022_e4eceae7-4182-4f06-9c87-9276586d9f61): select * from retail_partitioned_clean where store_id='NY-01'
INFO  : Completed executing command(queryId=hive_20260601170022_e4eceae7-4182-4f06-9c87-9276586d9f61); Time taken: 0.003 seconds
INFO  : OK
INFO  : Concurrency mode is disabled, not creating a lock manager
+--------------------------------------+------------------------------------+--------------------------------------+-------------------------------------+------------------------------------+--------------------------------------+
| retail_partitioned_clean.receipt_id  | retail_partitioned_clean.item_sku  | retail_partitioned_clean.sale_price  | retail_partitioned_clean.sale_date  | retail_partitioned_clean.store_id  | retail_partitioned_clean.sale_month  |
+--------------------------------------+------------------------------------+--------------------------------------+-------------------------------------+------------------------------------+--------------------------------------+
| 98401                                | SKU-A111                           | 29.99                                | 2023-10-14                          | NY-01                              | 2023-10                              |
| 98402                                | SKU-A892                           | 14.99                                | 2023-10-15                          | NY-01                              | 2023-10                              |
| 98404                                | SKU-C333                           | 49.5                                 | 2023-11-01                          | NY-01                              | 2023-11                              |
| 98407                                | SKU-E555                           | 5.99                                 | 2024-01-10                          | NY-01                              | 2024-01                              |
+--------------------------------------+------------------------------------+--------------------------------------+-------------------------------------+------------------------------------+--------------------------------------+
4 rows selected (0.843 seconds)


0: jdbc:hive2://localhost:10000> EXPLAIN FORMATTED
. . . . . . . . . . . . . . . .> INSERT OVERWRITE TABLE retail_partitioned_clean
. . . . . . . . . . . . . . . .> PARTITION (store_id, sale_month)
. . . . . . . . . . . . . . . .> SELECT
. . . . . . . . . . . . . . . .>     receipt_id,
. . . . . . . . . . . . . . . .>     item_sku,
. . . . . . . . . . . . . . . .>     sale_price,
. . . . . . . . . . . . . . . .>     sale_date,
. . . . . . . . . . . . . . . .>     store_id,
. . . . . . . . . . . . . . . .>     substr(sale_date, 1, 7) AS sale_month
. . . . . . . . . . . . . . . .> FROM retail_staging_raw;
INFO  : Compiling command(queryId=hive_20260601173738_7f57a171-c7cd-4a53-bb86-c28466e662be): EXPLAIN FORMATTED
INSERT OVERWRITE TABLE retail_partitioned_clean
PARTITION (store_id, sale_month)
SELECT
receipt_id,
item_sku,
sale_price,
sale_date,
store_id,
substr(sale_date, 1, 7) AS sale_month
FROM retail_staging_raw
INFO  : Concurrency mode is disabled, not creating a lock manager
INFO  : Semantic Analysis Completed (retrial = false)
INFO  : Returning Hive schema: Schema(fieldSchemas:[FieldSchema(name:Explain, type:string, comment:null)], properties:null)
INFO  : Completed compiling command(queryId=hive_20260601173738_7f57a171-c7cd-4a53-bb86-c28466e662be); Time taken: 0.481 seconds
INFO  : Concurrency mode is disabled, not creating a lock manager
INFO  : Executing command(queryId=hive_20260601173738_7f57a171-c7cd-4a53-bb86-c28466e662be): EXPLAIN FORMATTED
INSERT OVERWRITE TABLE retail_partitioned_clean
PARTITION (store_id, sale_month)
SELECT
receipt_id,
item_sku,
sale_price,
sale_date,
store_id,
substr(sale_date, 1, 7) AS sale_month
FROM retail_staging_raw
INFO  : Starting task [Stage-4:EXPLAIN] in serial mode
INFO  : Completed executing command(queryId=hive_20260601173738_7f57a171-c7cd-4a53-bb86-c28466e662be); Time taken: 0.061 seconds
INFO  : OK
INFO  : Concurrency mode is disabled, not creating a lock manager
+----------------------------------------------------+
|                      Explain                       |
+----------------------------------------------------+
| {"optimizedSQL":"SELECT `receipt_id`, `item_sku`, `sale_price`, `sale_date`, `store_id`, SUBSTR(`sale_date`, 1, 7) AS `sale_month`\nFROM `default`.`retail_staging_raw`","cboInfo":"Plan optimized by CBO.","STAGE DEPENDENCIES":{"Stage-1":{"ROOT STAGE":"TRUE"},"Stage-2":{"DEPENDENT STAGES":"Stage-1"},"Stage-0":{"DEPENDENT STAGES":"Stage-2"},"Stage-3":{"DEPENDENT STAGES":"Stage-0"}},"STAGE PLANS":{"Stage-1":{"Tez":{"DagId:":"hive_20260601173738_7f57a171-c7cd-4a53-bb86-c28466e662be:2","Edges:":{"Reducer 2":{"parent":"Map 1","type":"SIMPLE_EDGE"},"Reducer 3":{"parent":"Map 1","type":"SIMPLE_EDGE"}},"DagName:":"hive_20260601173738_7f57a171-c7cd-4a53-bb86-c28466e662be:2","Vertices:":{"Map 1":{"Map Operator Tree:":[{"TableScan":{"alias:":"retail_staging_raw","columns:":["receipt_id","item_sku","sale_price","sale_date","store_id"],"database:":"default","Statistics:":"Num rows: 6 Data size: 3384 Basic stats: COMPLETE Column stats: NONE","table:":"retail_staging_raw","isTempTable:":"false","OperatorId:":"TS_0","children":{"Select Operator":{"expressions:":"receipt_id (type: int), item_sku (type: string), sale_price (type: double), sale_date (type: string), store_id (type: string), substr(sale_date, 1, 7) (type: string)","columnExprMap:":{"_col0":"receipt_id","_col1":"item_sku","_col2":"sale_price","_col3":"sale_date","_col4":"store_id","_col5":"substr(sale_date, 1, 7)"},"outputColumnNames:":["_col0","_col1","_col2","_col3","_col4","_col5"],"Statistics:":"Num rows: 6 Data size: 3384 Basic stats: COMPLETE Column stats: NONE","OperatorId:":"SEL_1","children":[{"Select Operator":{"expressions:":"_col0 (type: int), _col1 (type: string), _col2 (type: double), _col3 (type: string), _col4 (type: string), _col5 (type: string)","columnExprMap:":{"item_sku":"_col1","receipt_id":"_col0","sale_date":"_col3","sale_month":"_col5","sale_price":"_col2","store_id":"_col4"},"outputColumnNames:":["receipt_id","item_sku","sale_price","sale_date","store_id","sale_month"],"Statistics:":"Num rows: 6 Data size: 3384 Basic stats: COMPLETE Column stats: NONE","OperatorId:":"SEL_4","children":{"Group By Operator":{"aggregations:":["compute_stats(receipt_id, 'hll')","compute_stats(item_sku, 'hll')","compute_stats(sale_price, 'hll')","compute_stats(sale_date, 'hll')"],"columnExprMap:":{"_col0":"store_id","_col1":"sale_month"},"keys:":"store_id (type: string), sale_month (type: string)","mode:":"hash","outputColumnNames:":["_col0","_col1","_col2","_col3","_col4","_col5"],"Statistics:":"Num rows: 6 Data size: 3384 Basic stats: COMPLETE Column stats: NONE","OperatorId:":"GBY_5","children":{"Reduce Output Operator":{"columnExprMap:":{"KEY._col0":"_col0","KEY._col1":"_col1","VALUE._col0":"_col2","VALUE._col1":"_col3","VALUE._col2":"_col4","VALUE._col3":"_col5"},"key expressions:":"_col0 (type: string), _col1 (type: string)","sort order:":"++","Map-reduce partition columns:":"_col0 (type: string), _col1 (type: string)","Statistics:":"Num rows: 6 Data size: 3384 Basic stats: COMPLETE Column stats: NONE","value expressions:":"_col2 (type: struct<columntype:string,min:bigint,max:bigint,countnulls:bigint,bitvector:binary>), _col3 (type: struct<columntype:string,maxlength:bigint,sumlength:bigint,count:bigint,countnulls:bigint,bitvector:binary>), _col4 (type: struct<columntype:string,min:double,max:double,countnulls:bigint,bitvector:binary>), _col5 (type: struct<columntype:string,maxlength:bigint,sumlength:bigint,count:bigint,countnulls:bigint,bitvector:binary>)","OperatorId:":"RS_6","outputname:":"Reducer 2","outputOperator:":["GBY_7"]}}}}}},{"Reduce Output Operator":{"columnExprMap:":{"KEY._col4":"_col4","KEY._col5":"_col5","VALUE._col0":"_col0","VALUE._col1":"_col1","VALUE._col2":"_col2","VALUE._col3":"_col3"},"key expressions:":"_col4 (type: string), _col5 (type: string)","sort order:":"++","Map-reduce partition columns:":"_col4 (type: string), _col5 (type: string)","Statistics:":"Num rows: 6 Data size: 3384 Basic stats: COMPLETE Column stats: NONE","value expressions:":"_col0 (type: int), _col1 (type: string), _col2 (type: double), _col3 (type: string)","OperatorId:":"RS_10","outputname:":"Reducer 3","outputOperator:":["SEL_15"]}}]}}}}]},"Reducer 2":{"Reduce Operator Tree:":{"Group By Operator":{"aggregations:":["compute_stats(VALUE._col0)","compute_stats(VALUE._col1)","compute_stats(VALUE._col2)","compute_stats(VALUE._col3)"],"columnExprMap:":{"_col0":"KEY._col0","_col1":"KEY._col1"},"keys:":"KEY._col0 (type: string), KEY._col1 (type: string)","mode:":"mergepartial","outputColumnNames:":["_col0","_col1","_col2","_col3","_col4","_col5"],"Statistics:":"Num rows: 3 Data size: 1692 Basic stats: COMPLETE Column stats: NONE","OperatorId:":"GBY_7","children":{"Select Operator":{"expressions:":"_col2 (type: struct<columntype:string,min:bigint,max:bigint,countnulls:bigint,numdistinctvalues:bigint,ndvbitvector:binary>), _col3 (type: struct<columntype:string,maxlength:bigint,avglength:double,countnulls:bigint,numdistinctvalues:bigint,ndvbitvector:binary>), _col4 (type: struct<columntype:string,min:double,max:double,countnulls:bigint,numdistinctvalues:bigint,ndvbitvector:binary>), _col5 (type: struct<columntype:string,maxlength:bigint,avglength:double,countnulls:bigint,numdistinctvalues:bigint,ndvbitvector:binary>), _col0 (type: string), _col1 (type: string)","columnExprMap:":{"_col0":"_col2","_col1":"_col3","_col2":"_col4","_col3":"_col5","_col4":"_col0","_col5":"_col1"},"outputColumnNames:":["_col0","_col1","_col2","_col3","_col4","_col5"],"Statistics:":"Num rows: 3 Data size: 1692 Basic stats: COMPLETE Column stats: NONE","OperatorId:":"SEL_8","children":{"File Output Operator":{"compressed:":"false","Statistics:":"Num rows: 3 Data size: 1692 Basic stats: COMPLETE Column stats: NONE","table:":{"input format:":"org.apache.hadoop.mapred.SequenceFileInputFormat","output format:":"org.apache.hadoop.hive.ql.io.HiveSequenceFileOutputFormat","serde:":"org.apache.hadoop.hive.serde2.lazy.LazySimpleSerDe"},"OperatorId:":"FS_9"}}}}}}},"Reducer 3":{"Execution mode:":"vectorized","Reduce Operator Tree:":{"Select Operator":{"expressions:":"VALUE._col0 (type: int), VALUE._col1 (type: string), VALUE._col2 (type: double), VALUE._col3 (type: string), KEY._col4 (type: string), KEY._col5 (type: string)","outputColumnNames:":["_col0","_col1","_col2","_col3","_col4","_col5"],"OperatorId:":"SEL_15","children":{"File Output Operator":{"compressed:":"false","Dp Sort State:":"PARTITION_SORTED","Statistics:":"Num rows: 6 Data size: 3384 Basic stats: COMPLETE Column stats: NONE","table:":{"input format:":"org.apache.hadoop.mapred.TextInputFormat","output format:":"org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat","serde:":"org.apache.hadoop.hive.serde2.lazy.LazySimpleSerDe","name:":"default.retail_partitioned_clean"},"OperatorId:":"FS_16"}}}}}}}},"Stage-2":{"Dependency Collection":{}},"Stage-0":{"Move Operator":{"tables:":{"partition:":{},"replace:":"true","table:":{"input format:":"org.apache.hadoop.mapred.TextInputFormat","output format:":"org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat","serde:":"org.apache.hadoop.hive.serde2.lazy.LazySimpleSerDe","name:":"default.retail_partitioned_clean"}}}},"Stage-3":{"Stats Work":{"Column Stats Desc:":{"Columns:":["receipt_id","item_sku","sale_price","sale_date"],"Column Types:":["int","string","double","string"],"Table:":"default.retail_partitioned_clean"}}}}} |
+----------------------------------------------------+
1 row selected (0.563 seconds)

```


A quick interpretation of one line:

Map 1: 1(+1)/2 = 2 map tasks total, 1 finished, 1 running.

Reducer 2: 0/2 = 2 reducer tasks total, none finished yet.

Reducer 3: 0(+1)/2 = 2 reducer tasks total, 1 running, none finished.

```
"Edges:":{"Reducer 2":{"parent":"Map 1","type":"SIMPLE_EDGE"},"Reducer 3":{"parent":"Map 1","type":"SIMPLE_EDGE"}}
```

In Hadoop, output filenames from a distributed job follow a specific pattern:

- The first part (000000 or 000001) is the Task ID (specifically, the ID of the Reducer or Tez task that wrote the file).
- The second part (_0) is the attempt number (e.g., if the task failed and had to be retried, it would be _1).

From your HDFS listing, only store_id=NY-01/sale_month=2023-10 has file 000000_0, while all the other partition directories contain 000001_0, 
so in this run NY-01/2023-10 appears to have been assigned to reducer task 0 and the other partition keys to reducer task 1 (the partitioning function hashed those five partition keys to reducer 1 in this tiny dataset)

Map 1 has two outgoing SIMPLE_EDGEs, one to Reducer 2 and one to Reducer 3.

So this job is not a single linear map -> reduce flow; 
it is a small DAG with one upstream vertex feeding two downstream branches.

Map 1 role
Map 1 is scanning retail_staging_raw, projecting the original columns, and computing substr(sale_date, 1, 7) as sale_month.
That matches Hive’s dynamic partition rule that partition columns must appear last in the SELECT list and in the same order as the PARTITION(...) clause.

Reducer 2 role
Reducer 2 is the stats branch.
You can tell because the map side sends rows into a Group By Operator with compute_stats(...), keyed by store_id and sale_month, and Reducer 2 merges those partial statistics before writing an intermediate file.
So Reducer 2 is not writing your final table rows; it is supporting the later statistics work for the partitioned insert.

Reducer 3 role
Reducer 3 is the data-writing branch.
Its tree takes the row values plus the partition keys, and its File Output Operator targets default.retail_partitioned_clean with Dp Sort State: PARTITION_SORTED, which is exactly what you would expect for writing dynamic partitions.

So the practical mapping is:

Map 1 = read source rows and derive sale_month.

Reducer 2 = compute per-partition column statistics.

Reducer 3 = write the actual partitioned data files.

Why Reducer 3 could start first
This plan shows that both reducers are siblings under Map 1, not parent-child stages.

So Tez is free to schedule one child branch before the other based on readiness and available containers, 
which is why Reducer 3 could begin before Reducer 2 even though its number is higher.

A compact picture of your plan is:

Map 1 -> Reducer 2 for stats.

Map 1 -> Reducer 3 for final table output.

Then MOVE moves staged files into the final partition directories, and STATS records partition statistics.
