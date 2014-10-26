---
layout: post
title: Spark使用总结(更新中)
category: 技术
tags: [Spark]
keywords: Spark
description: 
---
### 1、集群内存分配


- SPARK_WORKER_MEMORY

	 SPARK_WORKER_MEMORY是计算节点worker所能支配的内存，各个节点可以根据实际物理内存的大小，通过配置conf/spark-env.sh来分配内存给该节点的worker进程使用。在spark standalone集群中，如果各节点的物理配置不一样，conf/spark-env.sh中的配置也可以不一样。

- SPARK_MEM

	SPARK_MEM是SparkContext提交给worker运行的executor进程所需要的内存，如果worker本身能支配的内存小于这个内存，那么在该worker上就不会分配到executor。Spark Application在SparkContext里中没有配置内存需求的情况下，如果设置了环境变量SPARK_MEM，Spark Application则取设置的数值，不然取缺省值512m。运行前设置：

		export SPARK_MEM=900m			
	执行spark-shell，每个worker分配900M内存

###2、作业提交参数及方法

- spark-submit

		./bin/spark-submit \
		  --class org.apache.spark.examples.SparkPi \
		  --master spark://slave1:7077 \
		  --executor-memory 1G \
		  --total-executor-cores 4 \
		  /path/to/examples.jar \

- spark-shell

	> spark-shell 是一个spark application，运行时需要向资源管理器申请资源，如standalone spark、YARN、Mesos。本例向standalone spark申请资源，所以在运行spark-shell时需要指向申请资源的standalone spark集群信息，其参数为MASTER。
	- 如果未在spark-env.sh中申明MASTER，则使用命令:

			MASTER=spark://slave1:7077 bin/spark-shell
	- 如果已经在spark-env.sh中申明MASTER，格式如下：

			export SPARK_MASTER_IP=192.168.1.171
			export SPARK_MASTER_PORT=7077
			export MASTER=spark://${SPARK_MASTER_IP}:${SPARK_MASTER_PORT}

			则可以直接用: 
	
			bin/spark-shell
			
	- 缺省的情况下，会申请所有的CPU资源，分配2个核
	
			bin/spark-shell -c 2
	
	`（注：如果是远程客户端来连接到spark Standalone集群的话，部署目录要和集群的部署目录一致。）`
	
###参考资料
1. [mmicky 的博客](http://mmicky.blog.163.com/)