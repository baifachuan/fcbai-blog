---
title: Spark – Initial job has not accepted any resources
tags: 编程基础
categories: 编程基础
abbrlink: cf1277b5
date: 2022-02-28 18:14:44
---

After setting up a new Spark cluster running on Yarn, I’ve come across an error Initial job has not accepted any resources; check your cluster UI to ensure that workers are registered and have sufficient resources while running Spark application via spark-submit and also tried with PySpark but got the same error.

![image](spark-001.png)

As you noticed when you get this warning, your application keeps running and wait for the resources indefinitely. You need to kill a job in order to terminate.

This issue is mainly caused due to the resource unavailable to run your application on the cluster, I have resolved this issue and able to run the Spark application on the cluster successfully and though it would be helpful sharing here.

### Disable dynamic resource allocation

If you have enough nodes, cores and memory resources available to yarn, Spark uses dynamic allocation to create spark workers and by default this dynamic allocation is enabled.

When you have a small cluster with limited resources, you can disable this option and allocate resources as per your need. Make sure to allocate resources less than the actual resources available.

```
spark-submit --conf spark.dynamicAllocation.enabled=false
```

When you turn off the dynamic allocation, you need to explicitly allocate the resources. Below is Python (PySpark) spark-submit command with minimum config.

```
spark-submit --deploy-mode cluster --master yarn \
        --driver-memory 3g --executor-memory 3g \
        --num-executors 2 --executor-cores 2 \
        --conf spark.dynamicAllocation.enabled=false \
        readcsv.py
```

Below is spark-submit for scala with minimum config.

```
spark-submit --deploy-mode cluster --master yarn \
        --driver-memory 3g --executor-memory 3g \
        --num-executors 2 --executor-cores 2 \
        --conf spark.dynamicAllocation.enabled=false \
        --jar application.jar \
        --class com.example.ReadCSV
```

### By Pragmatically

You can also turn-off the dynamic resource allocation by pragmatically.

```
conf = SparkConf().setAppName("SparkByExamples.com")
        .set("spark.shuffle.service.enabled", "false")
        .set("spark.dynamicAllocation.enabled", "false")
```

### Starting Slave server

If you are running the Spark standalone cluster, you can also get this error when you are not running your slaves.

Start your slave servers by passing master URL

```
start-slave.sh <spark://master-hostname:7077>
```

Hope you able to resolve this error and able to run your application.

