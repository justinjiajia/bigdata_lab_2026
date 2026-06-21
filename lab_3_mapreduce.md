
# EMR settings

- EMR release: 7.13.0

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

# Local file system operations for data preparation


```shell
$ mkdir data

$ cd data

$ wget https://archive.org/download/encyclopaediabri31156gut/pg31156.txt

$ wget https://archive.org/download/encyclopaediabri34751gut/pg34751.txt

$ wget https://archive.org/download/encyclopaediabri35236gut/pg35236.txt

$ wget -O nytimes.txt https://raw.githubusercontent.com/justinjiajia/datafiles/main/nytimes_news_articles.txt

$ cd ..

$ du -sh data
```

Instead of running them one after another, we can also put them into a `.script` file and run them in batch.

```shell
$ nano data_prep.sh
```


Copy and paste the code snippet below into the file:

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
du -sh data
```

Save the change and get back to the shell. Then run:

```shell
bash data_prep.sh
```
or 

```shell
sh data_prep.sh
```




<br>

# HDFS operations for data preparation

You can change all `<Your ITSC Account>` placeholders below to your ITSC account string first. 
Later, you can just copy and paste the commands to the terminal for execution

Note that `hadoop fs` and `hdfs dfs` can be interchangeably used below.

```shell
hadoop fs -df -h
```

```shell
hadoop fs -ls /
```

```shell
hadoop fs -mkdir -p /<Your ITSC Account>
```

```shell
hadoop fs -put data /<Your ITSC Account>
```

```shell
hadoop fs -ls /<Your ITSC Account>/data
```

```shell
hdfs dfs -df -h
```

You can use a HDFS filesystem checking utility to get a file's block report, e.g.,

```shell
hdfs fsck /<Your ITSC Account>/data/nytimes.txt -files -blocks -locations
```

<br>

# MapReduce job submission


Note that `hadoop jar` and `yarn jar` can be interchangeably used below.

```shell
hadoop jar /usr/lib/hadoop-mapreduce/hadoop-mapreduce-examples.jar wordcount /<Your ITSC Account>/data /<Your ITSC Account>/wordcount_output
```

We can use the `-D` flag to define a value for a property in the format of `property=value`.
E.g., we can specify the number of reducers to use as follows:

```shell
yarn jar /usr/lib/hadoop-mapreduce/hadoop-mapreduce-examples.jar wordcount -D mapreduce.job.reduces=2  /<Your ITSC Account>/data /<Your ITSC Account>/wordcount_output_1
```

<br>

# Get the output

```shell
hadoop fs -cat /<Your ITSC Account>/wordcount_output/part-r-* > combinedresult.txt
```

```shell
head -n20 combinedresult.txt
```

```shell
tail -n20 combinedresult.txt
```


---

add the following things:



```bash
[hadoop@ip-172-31-88-24 ~]$ cat /etc/hadoop/conf/yarn-site.xml | grep -A 2 'yarn.nodemanager.resource.cpu-vcores'
    <name>yarn.nodemanager.resource.cpu-vcores</name>
    <value>4</value>
  </property>
[hadoop@ip-172-31-88-24 ~]$ yarn node -list
2026-06-06 14:44:57,625 INFO client.DefaultNoHARMFailoverProxyProvider: Connecting to ResourceManager at ip-172-31-88-24.ec2.internal/172.31.88.24:8032
2026-06-06 14:44:58,048 INFO client.AHSProxy: Connecting to Application History server at ip-172-31-88-24.ec2.internal/172.31.88.24:10200
Total Nodes:3
         Node-Id             Node-State Node-Http-Address       Number-of-Running-Containers
ip-172-31-88-31.ec2.internal:8041               RUNNING ip-172-31-88-31.ec2.internal:8042                                  0
ip-172-31-82-214.ec2.internal:8041              RUNNING ip-172-31-82-214.ec2.internal:8042                                 0
ip-172-31-91-160.ec2.internal:8041              RUNNING ip-172-31-91-160.ec2.internal:8042                                 0
[hadoop@ip-172-31-88-24 ~]$ yarn node -list -showDetails
2026-06-06 14:46:59,801 INFO client.DefaultNoHARMFailoverProxyProvider: Connecting to ResourceManager at ip-172-31-88-24.ec2.internal/172.31.88.24:8032
2026-06-06 14:47:00,370 INFO client.AHSProxy: Connecting to Application History server at ip-172-31-88-24.ec2.internal/172.31.88.24:10200
2026-06-06 14:47:00,649 INFO conf.Configuration: resource-types.xml not found
2026-06-06 14:47:00,650 INFO resource.ResourceUtils: Unable to find 'resource-types.xml'.
Total Nodes:3
         Node-Id             Node-State Node-Http-Address       Number-of-Running-Containers
ip-172-31-88-31.ec2.internal:8041               RUNNING ip-172-31-88-31.ec2.internal:8042                                  0
Detailed Node Information :
        Configured Resources : <memory:6144, vCores:4>
        Allocated Resources : <memory:0, vCores:0>
        Resource Utilization by Node : PMem:2690 MB, VMem:2690 MB, VCores:0.1466178
        Resource Utilization by Containers : PMem:0 MB, VMem:0 MB, VCores:0.0
        Node-Labels : 
ip-172-31-82-214.ec2.internal:8041              RUNNING ip-172-31-82-214.ec2.internal:8042                                 0
Detailed Node Information :
        Configured Resources : <memory:6144, vCores:4>
        Allocated Resources : <memory:0, vCores:0>
        Resource Utilization by Node : PMem:2680 MB, VMem:2680 MB, VCores:0.033322226
        Resource Utilization by Containers : PMem:0 MB, VMem:0 MB, VCores:0.0
        Node-Labels : 
ip-172-31-91-160.ec2.internal:8041              RUNNING ip-172-31-91-160.ec2.internal:8042                                 0
Detailed Node Information :
        Configured Resources : <memory:6144, vCores:4>
        Allocated Resources : <memory:0, vCores:0>
        Resource Utilization by Node : PMem:2751 MB, VMem:2751 MB, VCores:0.15
        Resource Utilization by Containers : PMem:0 MB, VMem:0 MB, VCores:0.0
        Node-Labels :
[hadoop@ip-172-31-88-24 ~]$ cat /etc/spark/conf/spark-defaults.conf | grep spark.default.parallelism
[hadoop@ip-172-31-88-24 ~]$ cat /etc/spark/conf/spark-defaults.conf | grep spark.emr
spark.emr.default.executor.memory 4269M
spark.emr.default.executor.cores 4
```

- EC2 vCPU: The actual physical threads on the CPU hardware (for `m4.large`, this is 2).
- YARN vCore: A "virtual" scheduling unit YARN uses to decide how many tasks to run. This is set by `yarn.nodemanager.resource.cpu-vcores`. This creates the "ceiling" that YARN uses for resource allocation.

The mapping between these two is not always 1:1. To optimize CPU utilization for different types of workloads, EMR uses a predefined mapping for every instance type/family.

For an `m4.large` instance, this mapping is a 2:1 multiplier, meaning:

YARN vCores = EC2 vCPUs × 2
4 = 2 × 2

As a result, you get the `yarn.nodemanager.resource.cpu-vcores=4` you're seeing in your configuration files.
