# 面试
### Spark Shuffle
一、什么是Shuffle

如果在重分区的过程中，如果数据发生了跨节点移动，就被称为 Shuffle，在 Spark 中Shuffle 负责将 Map 端的处理的中间结果传输到 Reduce 端供 Reduce 端聚合，Spark 对 Shuffle 的实现方式有两种：Hash based Shuffle 与 Sort based Shuffle。

二、Shuffle的特点：

只有Key-value型的RDD才会有shuffle操作
早期版本的shuffle是HashShuffle，后来改为SortShuffle更适合大吞吐量的场景。
shuffle分为两端：Mapper端把发给Reducer的数据放在文件中，Reducer端通过拉取文件获取数据

三、HashShuffle
数据取Hash放在不同的文件中，1000个mapper，1000个reducer，产生文件数量1000*1000，不适合大规模数据的处理。

1、HashShuffle存在的问题

shuffle产生海量的小文件在磁盘上，即生成的中间结果文件数太多。理论上，每个 Shuffle 任务输出会产生 R 个文件（R为Reducer 的个数），而 Shuffle 任务的个数往往由 Map 任务个数 M 决定，所以总共会生成 M * R 个中间结果文件，而往往在一个作业中 M 和 R 都是很大的数字，在大型作业中，经常会出现文件句柄数突破操作系统限制。
容易导致内存不够用，由于内存需要保存海量的文件操作句柄和临时缓存信息，如果数据处理规模比较大，容易出现OOM；
容易出现数据倾斜，导致OOM。

四、SortShuffle
SortShuffle在map端有三种实现，分别是UnsafeShuffleWriter、BypassMergeSortShuffleWriter、SortShuffleWriter，三种ShuffleWriter实现均由SortShuffleManager管理。

1、三种ShuffleWriter使用时机：

UnsafeShuffleWriter：map端没有聚合操作，RDD的Partition数小于 16777216，Serializer支持relocation

BypassMergeSortShuffleWriter：map端没有聚合操作，RDD的Partition数小于200

SortShuffleWriter：map端支持聚合操作，也支持排序操作。

2、UnsafeShuffleWriter

内存管理（申请、释放）工作，由ShuffleExternalSorter来完成。ShuffleExternalSorter还有一个作用就是当内存中数据太多的时候，会先spill到磁盘，防止内存溢出。

3、BypassMergeSortShuffleWriter

和Hash Shuffle中的HashShuffleWriter实现基本一致，唯一的区别在于，map端的多个输出文件会被汇总为一个文件

4、SortShuffleWriter

Map侧将数据放入一个集合中,根据是否有aggregation操作，选择PartitionedAppendOnlyMap或PartitionedPairBuffer
然后通过一个类似于MergeSort的排序算法TimSort对AppendOnlyMap 集合底层的 Array 排序
排序的逻辑是:先按照PartitionId 排序, 后按照Key的HashCode 排序
最终每个 Map Task 生成一个输出文件, Reduce Task来拉取自己对应的数据
总结：每个Map任务最后只会输出两个文件（一个是索引文件会记录每个分区的偏移量）中间过程采用归并排序,输出完成后，Reducer会根据索引文件得到属于自己的分区。

五、Shuffle的调优点：
1、Shuffle的选择

spark.shuffle.manager：有三个可选项：hash、sort和tungsten-sort。

2、缓冲区的大小

spark.shuffle.file.buffer：该参数用于设置shuffle write task的BufferedOutputStream的buffer缓冲大小。将数据写到磁盘文件之前，会先写入buffer缓冲中，待缓冲写满之后，才会溢写到磁盘。

spark.reducer.maxSizeInFlight：该参数用于设置shuffle read task的buffer缓冲大小，而这个buffer缓冲决定了每次能够拉取多少数据。

3、间隔时间重试次数

spark.shuffle.io.maxRetries：shuffle read task从shuffle write task所在节点拉取属于自己的数据时，如果因为网络异常导致拉取失败，是会自动进行重试的。该参数就代表了可以重试的最大次数。如果在指定次数之内拉取还是没有成功，就可能会导致作业执行失败。

spark.shuffle.io.retryWait：该参数代表了每次重试拉取数据的等待间隔，默认是5s。

4、Shuffle内存配置

spark.shuffle.memoryFraction：该参数代表了Executor内存中，分配给shuffle read task进行聚合操作的内存比例，默认是20%。

