
This assignment aims to give you hands-on practice with PySpark programming in EMR.

Launch an EMR cluster as instructed by lab 5's manual. Choose to install both Hadoop and Spark on the cluster.

Then connect to the master node, and run the following script to prepare the data.



```shell
#!/bin/bash

wget https://raw.githubusercontent.com/justinjiajia/datafiles/main/soc-LiveJournal1Adj.txt
wget https://raw.githubusercontent.com/justinjiajia/datafiles/main/tiny_social_data.txt

hadoop fs -mkdir /input
hadoop fs -put soc-LiveJournal1Adj.txt /input
hadoop fs -put tiny_social_data.txt /input
hadoop fs -df -h /input
hadoop fs -ls /input
```

Betweem the 2 datasets, the [*soc-LiveJournal1Adj.txt*](https://snap.stanford.edu/data/soc-LiveJournal1.html) dataset represents the social network structure of 50,000 LiveJournal users. Unlike mutual friendships you've seen before in the friend recommendation example, LiveJournal allows one-way friendship declarations - user A can declare user B as a friend without requiring B to reciprocate.

 

Despite this directional nature, the co-occurrence-based recommendation approach still applies: if two users appear together in the declaration lists of many other users, they are likely to be friends and we should recommend them to declare each other as friends. **However, we must not recommend a user someone they have already declared as a friend** — such a recommendation would be redundant.

### Implementation Note:
 

For a target user `U`, a candidate `V` must be filtered out if `U` has declared `V` as a friend — that is, if `V` appears in `U`'s own friend list. This check is one-directional: it considers only outgoing declarations from `U`.

Consequently, a candidate `V` may still be recommended to `U` even if `V` has declared `U` as a friend but `U` has not declared `V` — this is precisely the one-way connection the recommendation aims to close. In other words, the filter removes only pairs already covered by `U`'s own declarations, not pairs covered solely by the other direction.

[*tiny_social_data.txt*](https://github.com/justinjiajia/datafiles/blob/main/tiny_social_data.txt) stores a tiny portion of *soc-LiveJournal1Adj.txt*:

```
46133
46134    46069,46079,46105
46136
46148    46046,46096
46149    46046
49949    46058,49896,49936,49952,49955,49957,49960,49963,49979
46163    46062,46126
46160    46069
46157    46073,46093,46070
46156    46077
46152    46112
46140    2915
46141
46145
46150
46151    46068
46154
46161    46075
46162
46164
46168    46165,46167,46176,46178,46175
46169    46165,46170,46221
46173    46165,46170,46171,46172,46174,46176,46177,46179,46180,46194,46195,46207
46174    46165,46173,46177,46179,46180
46175    46165,46168,46176,46178,46221
```


So, the format of a record is as follows:

```
user_id <tab> friend1_id,friend2_id,friend3_id, ...
```

The record

```
46134    46069,46079,46105
```
means user `46134` has declared 3 friends. 


Note:

- User IDs in each list are not in sorted order.

- There are users who haven't declared any friends. They should be dropped from subsequent processing.

- For each target user `U`, a candidate `V` must be removed from `U`'s recommendation list if `U` has declared `V` (i.e., `V` is in `U`'s own friend list). The reverse direction alone does not trigger removal.

### Output Specification

The output must contain one line per user in the following format:

```
(<User_ID>, [(Recommended_User1, Shared_Friends_Count1), (Recommended_User2, Shared_Friends_Count2), ...])
```

Each line starts with a user ID, followed by **up to 5** recommendation tuples (fewer if fewer than 5 candidates remain **after filtering out users already declared by `U`**).
 
Each tuple contains: 

- `Recommended_User`: The suggested friend's ID;
  
- `Shared_Friends_Count`: Number of users who declared both this user and the recommended user as friends;

 
Tuples must be sorted in descending order by `Shared_Friends_Count`; only include the top 5 recommendations (or all available if fewer than 5 remain after filtering).

If, after filtering, a user has no candidates left, output the user with an empty list: `(<User_ID>, [])`

**Note**: Make sure your code works on *tiny_social_data.txt* before you run it on the entire dataset.


#### Hint

The RDD API provides a transformation called `subtractByKey` that is well suited to removing already-declared relationships from your candidate list. If you're unsure how to use it for this purpose, read the documentation for [pyspark.RDD.subtractByKey](https://spark.apache.org/docs/latest/api/python/reference/api/pyspark.RDD.subtractByKey.html) — pay attention to how keys are matched between the two RDDs and what happens to the values of keys that survive.
 


### Start the PySpark shell

```shell
pyspark --master yarn
```

### Show the output

If your output is saved in */output* on HDFS, you can use the command below to show the first 10 recommendation lists.

```shell
hadoop fs -cat /output/* | head -n10
```


### Your submission should include both the code and the screenshot of the output.

![#f03c15](https://placehold.co/15x15/f03c15/f03c15.png) IMPORTANT: After using EMR, remember to terminate your EMR cluster.

