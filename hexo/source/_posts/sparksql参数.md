---
title: sparksql参数
abbrlink: 7c6a5d83
date: 2021-12-28 11:20:52
urlname: zhanyy
categories: spark
tags: [spark,调优]
---
# [sparksql参数](https://www.cnblogs.com/yin-fei/p/10985683.html)


##### 全局参数：

1. --master yarn-cluster (or yarn-client)
    ```
    参数说明：
    制定yarn的执行模式，分集群模式和客户端模式，一般使用集群模式
    ```
2.  --num-executors 50
    ```
    参数说明：　　该参数用于设置Spark作业总共要用多少个Executor进程来执行。Driver在向YARN集群管理器申请资源时，YARN集群管理器会尽可能按照你的设置来在集群的各个工作节点上，启动相应数量的Executor进程。这个参数非常之重要，如果不设置的话，默认只会给你启动少量的Executor进程，此时你的Spark作业的运行速度是非常慢的。
    参数调优建议：　　每个Spark作业的运行一般设置20~50个左右的Executor进程比较合适，设置太少或太多的Executor进程都不好。设置的太少，无法充分利用集群资源；设置的太多的话，大部分队列可能无法给予充分的资源。
    ```

3. --executor-memory 6G
    ```
    参数说明：
    　　该参数用于设置每个Executor进程的内存。Executor内存的大小，很多时候直接决定了Spark作业的性能，而且跟常见的JVM OOM异常，也有直接的关联。
    参数调优建议：
    　　每个Executor进程的内存设置4G~8G较为合适,最大不超过 20G，否则会导致 GC 代价过高，或资源浪费严重。但是这只是一个参考值，具体的设置还是得根据不同部门的资源队列来定。可以看看自己团队的资源队列的最大内存限制是多少，num-executors乘以executor-memory，是不能超过队列的最大内存量的。此外，如果你是跟团队里其他人共享这个资源队列，那么申请的内存量最好不要超过资源队列最大总内存的1/3~1/2，避免你自己的Spark作业占用了队列所有的资源，导致别的同学的作业无法运行
    ```

4. --conf spark.executor.cores=4

    ```
    参数说明：
        该参数用于设置每个Executor进程的CPU core数量。这个参数决定了每个Executor进程并行执行task线程的能力。因为每个CPU core同一时间只能执行一个task线程，因此每个Executor进程的CPU core数量越多，越能够快速地执行完分配给自己的所有task线程。
    参数调优建议：
        Executor的CPU executor_cores 不宜为1！否则 work 进程中线程数过少，一般 2~4 为宜。。同样得根据不同部门的资源队列来定，可以看看自己的资源队列的最大CPU core限制是多少，再依据设置的Executor数量，来决定每个Executor进程可以分配到几个CPU core。同样建议，如果是跟他人共享这个队列，那么num-executors * executor-cores不要超过队列总CPU core的1/3~1/2左右比较合适，也是避免影响其他同学的作业运行。
    ```

5. --conf spark.yarn.executor.memoryOverhead=2048

    ```
    Ececutor堆外内存 
    当Spark处理超大数据量时(数十亿,百亿级别),executor的堆外内存可能会不够用,出现shuffle file can’t find, task lost,OOM等情况 
    默认情况下,这个堆外内存是300M,当运行超大数据量时,通常会出现问题,因此需要调节到1G,2G,4G等大小 
    调节方法必须在spark-submit提交脚本中设置而不能在程序中设置
    ```

6. --driver-memory 2G

    ```
    参数说明：
        该参数用于设置Driver进程的内存。
    参数调优建议：
        Driver的内存通常来说不设置，或者设置2G左右应该就够了。唯一需要注意的一点是，如果需要使用collect算子将RDD的数据全部拉取到Driver上进行处理，那么必须确保Driver的内存足够大，否则会出现OOM内存溢出、GC FULL等问题。
    ```

7. --conf spark.default.parallelism=150

    ```
    参数说明：
    　　spark_parallelism一般为executor_cores*num_executors 的 1~4 倍，系统默认值 64，不设置的话会导致 task 很多的时候被分批串行执行，或大量 cores 空闲，资源浪费严重
    ```

8. 动态executor  --避免使用

    ```
    --conf spark.dynamicAllocation.enable=true //打开动态executor模式
    --conf spark.shuffle.service.enabled=true //动态executor需要的服务，需要和上面的spark.dynamicAllocation.enable同时打开或关闭
    ```

9. --conf spark.storage.memoryFraction=0.2

    ```
    参数说明：
    　　该参数用于设置RDD持久化数据在Executor内存中能占的比例，默认是0.6。也就是说，默认Executor 60%的内存，可以用来保存持久化的RDD数据。
    ```

10. exector、storage内存分配
    ```
    
    当Spark一个JOB被提交，就会开辟内存空间来存储和计算
    Spark中执行计算和数据存储都是共享同一个内存区域（M），spark.memory.fraction 表示M的大小，其值为相对于JVM堆内存的比例（默认0.6）。剩余的40%是为其他用户数据结构、Spark内部元数据以及避免OOM错误的安全预留空间（大量稀疏数据和异常大的数据记录）。
    
    spark.memory.storageFraction 表示数据存储比例（R）的大小，其值为相对于M的一个比例（默认0.5）。R是M中专门用于缓存数据块，且这部分数据块永远不会因执行计算任务而逐出内存。
    所以当发生FULL GC之后，有两种办法：
    第一就是增大M区域，也就是增加--executor-memory 10G
    这样相当于增大了new generation区和old generation区，能放得下大数据块
    
    
    第二就是减小R区域，也就是减小-- spark.memory.storageFraction
    这样相当于增大了内存中用于计算的区域，从而避免FULL GC的问题
    
    spark.memory.fraction这个参数建议保持默认值，非特殊情况不要修改。
    ```
    
    ![img](https://s3.bmp.ovh/imgs/2021/12/d90ad01acd9c1c03.png)


##### shuffle参数：

1. --conf spark.shuffle.memoryFraction=0.5

    ```
    参数说明：
        该参数用于设置shuffle过程中一个task拉取到上个stage的task的输出后，进行聚合操作时能够使用的Executor内存的比例，默认是0.2。
    
    
    也就是说，Executor默认只有20%的内存用来进行该操作。shuffle操作在进行聚合时，如果发现使用的内存超出了这个20%的限制，那么多余的数据就会溢写到磁盘文件中去，此时就会极大地降低性能。
    参数调优建议：
    
    如果Spark作业中的RDD持久化操作较少，shuffle操作较多时，建议降低持久化操作的内存占比，提高shuffle操作的内存占比比例，避免shuffle过程中数据过多时内存不够用，必须溢写到磁盘上，降低了性能。此外，如果发现作业由于频繁的gc导致运行缓慢，意味着task执行用户代码的内存不够用，那么同样建议调低这个参数的值。
    
    按实际情况设置这个参数，但是不推荐超过0.6
    ```
2. --conf spark.sql.shuffle.partitions=20

    ```
    默认值：300
    spark中有partition的概念，每个partition都会对应一个task，task越多，在处理大规模数据的时候，就会越有效率。不过task并不是越多越好，如果发现数据量没有那么大，则没有必要task数量太多。
    其实这个参数相当于Hive参数mapred.reduce.tasks，那种大促期间数据量翻好几倍的任务不推荐写死这个参数，否则会造成单个task处理的数据量激增导致任务失败或者阻塞。
    ```

3. --conf spark.shuffle.compress=true  
    ```
     //shuffle过程是否压缩
    ```
4. --conf spark.shuffle.file.buffer=512

    ```
    默认值：32k
    参数说明：　　该参数用于设置shuffle write task的BufferedOutputStream的buffer缓冲大小。将数据写到磁盘文件之前，会先写入buffer缓冲中，待缓冲写满之后，才会溢写到磁盘。
    调优建议：　　如果作业可用的内存资源较为充足的话，可以适当增加这个参数的大小（比如64k），从而减少shuffle write过程中溢写磁盘文件的次数，也就可以减少磁盘IO次数，进而提升性能。在实践中发现，合理调节该参数，性能会有1%~5%的提升。
    ```

5. --conf spark.reducer.maxSizeInFlight=256m

    ```
    默认值：48m
    参数说明：　　该参数用于设置shuffle read task的buffer缓冲大小，而这个buffer缓冲决定了每次能够拉取多少数据。
    调优建议：　　如果作业可用的内存资源较为充足的话，可以适当增加这个参数的大小（比如96m），从而减少拉取数据的次数，也就可以减少网络传输的次数，进而提升性能。在实践中发现，合理调节该参数，性能会有1%~5%的提升。
    ```



6. --conf spark.shuffle.io.maxRetries=20

    ```
    默认值：3
    参数说明：　　shuffle read task从shuffle write task所在节点拉取属于自己的数据时，如果因为网络异常导致拉取失败，是会自动进行重试的。该参数就代表了可以重试的最大次数。如果在指定次数之内拉取还是没有成功，就可能会导致作业执行失败。
    调优建议：　　对于那些包含了特别耗时的shuffle操作的作业，建议增加重试最大次数（比如60次），以避免由于JVM的full gc或者网络不稳定等因素导致的数据拉取失败。在实践中发现，对于针对超大数据量（数十亿~上百亿）的shuffle过程，调节该参数可以大幅度提升稳定性。
    ```

7. --spark.shuffle.io.retryWait=5s

    ```
    默认值：5s
    参数说明：　　具体解释同上，该参数代表了每次重试拉取数据的等待间隔，默认是5s。调优建议：建议加大间隔时长（比如60s），以增加shuffle操作的稳定性。
    ```

##### 动态分配

1. reducer的可伸缩化

    ```
    --spark.sql.adaptive.enabled=true
    --spark.sql.adaptive.shuffle.targetPostShuffleInputSize=102400000
    ```

2. JOIN过程动态广播

    ```
    --spark.sql.autoBroadcastJoinThreshold=10485760  //(10 MB)
    
    类似Hive中的mapjoin，在join的过程中把小于10M的小表广播到所有节点，从而进行Hashjoin，提升join的效率。
    
    目前动态分配在处理几十亿以上的数据量时还是有很多未知bug缺陷，使用需谨慎
    ```