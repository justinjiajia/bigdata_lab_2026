
# EMR settings

- EMR release: 7.14.0

- Application: Hadoop
  
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
        },
        {
            "classification": "hdfs-site",
            "properties": {
                "dfs.block.size": "16777216",
                "dfs.replication": "3"
            }
        },
        {
            "classification": "mapred-site",
            "properties": {
                "mapreduce.job.reduces": "3"
            }
        }
    ]
    ```

- Make sure the primary node's EC2 security group has a rule allowing for "ALL TCP" from "My IP" and a rule allowing for "SSH" from "Anywhere".


<br>


# Data preparation

```shell
nano data_prep.sh
```

Copy and paste the code snippet below into the *data_prep.sh* file, and change all occurrences of `<Your ITSC Account>` to your ITSC account string. 


```shell
#!/bin/bash

rm -r data
mkdir data
cd data
wget https://archive.org/download/encyclopaediabri31156gut/pg31156.txt
wget https://archive.org/download/encyclopaediabri34751gut/pg34751.txt
wget https://archive.org/download/encyclopaediabri35236gut/pg35236.txt
wget -O nytimes.txt https://raw.githubusercontent.com/justinjiajia/datafiles/main/nytimes_news_articles.txt
cd ..
hadoop fs -mkdir /<Your ITSC Account>
hadoop fs -put data /<Your ITSC Account>
hadoop fs -df -h /<Your ITSC Account>/data
```


Save the change and get back to the shell. Then run:

```shell
bash data_prep.sh
```
or 

```shell
sh data_prep.sh
```

This allows you to do all the local and HDFS file system operations in one go.

Then, you can use a HDFS filesystem checking utility to get a file's block report, e.g.,

```shell
hdfs fsck /<Your ITSC Account>/data/nytimes.txt -files -blocks -locations
```

 

<br>

# MapReduce job submission


Note that `hadoop jar` and `yarn jar` can be interchangeably used below.

```shell
hadoop jar /usr/lib/hadoop-mapreduce/hadoop-mapreduce-examples.jar wordcount /<Your ITSC Account>/data /<Your ITSC Account>/output
```

We can use the `-D` flag to define a value for a property in the format of `property=value`.
E.g., we can specify the number of reducers to use as follows:

```shell
yarn jar /usr/lib/hadoop-mapreduce/hadoop-mapreduce-examples.jar wordcount -D mapreduce.job.reduces=2  /<Your ITSC Account>/data /<Your ITSC Account>/output_2
```

<br>

# Get the output

```shell
hadoop fs -cat /<Your ITSC Account>/output/part-r-* > combined_result.txt
```

```shell
head -n 20 combined_result.txt
```
 

For an `m4.large` instance, this mapping is a 2:1 multiplier, meaning:

YARN vCores = EC2 vCPUs × 2
4 = 2 × 2

As a result, you get the `yarn.nodemanager.resource.cpu-vcores=4` you're seeing in your configuration files.
