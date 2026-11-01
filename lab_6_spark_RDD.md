# EMR settings

- EMR release: 7.14.0 

-  ![#f03c15](https://placehold.co/15x15/f03c15/f03c15.png) IMPORTANT: Application: Hadoop, Spark, and Hive
  
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
    
- Make sure the primary node's EC2 security group has a rule allowing for "ALL TCP" from "My IP" and a rule allowing for "SSH" from "Anywhere".

<br>

# 1 Data Preparation

```shell
nano script.sh
```

Copy and paste the code snippet below into the *script.sh* file and replace all placeholders with your ITSC account string. 

```shell
#!/bin/bash
wget -O transactions.txt  https://raw.githubusercontent.com/justinjiajia/datafiles/main/browsing.csv
hadoop fs -mkdir -p /<Your ITSC Account>/input
hadoop fs -put transactions.txt /<Your ITSC Account>/input
hadoop fs -df -h /<Your ITSC Account>
```

```shell
bash script.sh
```



# 2 Run PySpark on Yarn via shell

## Start the shell

```shell
pyspark --master yarn --deploy-mode client
```
> Note: Spark shell can only be started in client mode.

## PySpark code to run sequentially

```python
transactions = sc.textFile("hdfs:///<Your ITSC Account>/input/transactions.txt")  # absolute path of the input file on HDFS

from itertools import combinations

pairs = transactions.map(lambda x: x.strip().split(" ")).flatMap(lambda x: combinations(x, 2)).map(lambda x: (x[0], x[1]) if x[0] <= x[1] else (x[1], x[0]))

pairs_count = pairs.map(lambda x: (x, 1)).reduceByKey(lambda x, y: x+y)

pairs_count.cache()

combined_pairs = pairs_count.flatMap(lambda x: [(x[0][0], [(x[0][1], x[1])]), (x[0][1], [(x[0][0], x[1])])])

rec_pairs_ordered = combined_pairs.reduceByKey(lambda x, y: sorted(x+y, key=lambda val: val[1], reverse=True)[:5])

rec_pairs_ordered.saveAsTextFile("hdfs:///<Your ITSC Account>/output")  # absolute path of the output directory file on HDFS

quit()
```


## Print the output

```shell
hadoop fs -cat /<Your ITSC Account>/output/part-* | grep "ELE96863"
```

<br>

# 3 Launch PySpark applications with `spark-submit`

## Create a PySpark script file

```shell
nano recommendation.py
```

## Code to paste into the file

```python
from pyspark.sql import SparkSession
from itertools import combinations

spark = SparkSession.builder.appName("recommendation").getOrCreate()
sc = spark.sparkContext

transactions = sc.textFile("hdfs:///<Your ITSC Account>/input/transactions.txt")
transactions.cache()
pairs = transactions.map(lambda x: x.strip().split(" ")).flatMap(lambda x: combinations(x, 2)).map(lambda x: (x[0], x[1]) if x[0] <= x[1] else (x[1], x[0]))
pairs_count = pairs.map(lambda x: (x, 1)).reduceByKey(lambda x, y: x+y)
combined_pairs = pairs_count.flatMap(lambda x: [(x[0][0], [(x[0][1], x[1])]), (x[0][1], [(x[0][0], x[1])])])
rec_pairs_ordered = combined_pairs.reduceByKey(lambda x, y: sorted(x+y, key=lambda val: val[1], reverse=True)[:5])

# --- save to S3 instead of HDFS ---
rec_pairs_ordered.saveAsTextFile("s3://hkust-isom3370-<Your ITSC Account>/recommendation_output")
```

## Submit the job


Before submitting the Spark job, create the S3 bucket that will store the processing results by using the `aws s3 mb` command:




```shell
aws s3 mb s3://hkust-isom3370-<Your ITSC Account>
```

Spark's `saveAsTextFile` operation does not create buckets automatically. 

If the target bucket does not exist, the Spark job will fail. Therefore, make sure the bucket is created before you proceed.


Now you are ready to submit the Spark program and run it on YARN:


```shell
spark-submit --master yarn recommendation.py
```

> Check out [this page](spark-submit_options.md) for more launch options 



## Download the output

Open the [Amazon S3 console](https://us-east-1.console.aws.amazon.com/s3/buckets?region=us-east-1) to browse and download the objects stored in the bucket.
<br>

# Reference

https://spark.apache.org/docs/latest/submitting-applications

