---
title: RSS-远程Merge的设计
date: 2023-12-25 18:58:15
tags:
---

# 1 默认的shuffle
> 注: 第一节简单介绍了主流框架默认的shuffle的原理，目的为了那里会使用本地磁盘，从而设计远程Merge。如果足够了解，可以忽略这部分内容。

我们依次对MapReduce, Tez, Spark的shuffle进行分析。

## 1.1 MapReduce

Map将Record写入到内存，当内存超过阈值的时候，会将内存的数据spill到磁盘文件中，按照partitionid+key的顺序依次将Record写入到磁盘文件。当Map处理完所有Record后, 会将当前内存中的数据spill到磁盘文件。然后读取所有的spill到磁盘的文件，并按照partitionid+key的顺序进行merge，得到排序的Records。

> 注: 按照partitionid排序的目的是Reduce端从Map端获取的数据的时候，尽可能顺序读。对于MR、Tez、Spark, 无论是否排序，只要有分区，都需要按照partitionid进行排序。

Reduce端会从Map端以远程或本地的方式拉取对应分区的Records，称之为MapOutput。正常情况下会直接使用内存，如果内存超过阈值会将这些Records写入到磁盘。然后Reduce端会对所有MapOutput使用最小堆K路归并排序进行一些Merge操作，得到全局排序的Reccords。Merge的过程中，可能会因为内存超过阈值，会将临时的结果spill到磁盘。另外如果spill到磁盘的文件过多，也会触发额外的merge。

## 1.2 Tez

Tez的情况略微复杂。Tez分为ordered io和unordered io。

ordered io与MapReduce相同，不再展开分析。

unordered io一般用于hashjoin等不需要key进行排序的情况。非排序的io采用来之即用方案。Map直接将Record写入文件或者通过缓存在写入文件。Reduce端也是读数据的时候也是读之即用。

## 1.3 Spark

spark的shuffle更复杂，也更合理。部分任务是不需要sort和combine的，因此spark用户可以需求决定shuffle的逻辑。

### 1.3.1 Shuffle写操作
写shuffle数据的时候，支持三种writer:
* (1) BypassMergeSortShuffleWriter

为每个partition都生成一个临时文件。在写Record的时候，找到对应的分区直接写入对应的临时文件。然后当所有数据都处理完成后，将这些临时的文件按照分区的顺序依次写入到一个最终的文件，并删除临时文件。

* (2) UnsafeShuffleWriter

UnsafeShuffleWriter主要通过ShuffleExternalSorter来实现具体的逻辑。当写Record的时候，直接序列化操作，并将序列化的字节拷贝到申请的内存中。同时也会将Record的地址和分区记录到内存中(inMemSorter)。

当内存的Record达到阈值的时候，会进行spill操作。根据内存(inMemSorter)中的信息，我们很容易得到一个按照分区排序的Record，并写到文件中。

当处理完所有Record，我们会把当前内存中的Records spill到文件中。最后对所有spilled文件进行一次聚合。由于之前spilled的文件已经是按照分区排序的，所以我们可以按照分区的顺序依次将所有spill的文件的对应自己拷贝到最终文件。这样得到的最终文件即为分区排序的文件。

* (3) SortShuffleWriter

SortShuffleWriter主要通过ExternalSorter来实现具体逻辑。ExternalSorter根据用户的需求决定是否combine和sort。

当写Record的时候，会直接插入到内存中。如果需要combine，内存架构是map，否则是buffer。

如果当前评估内存大于阈值会触发spill操作。spill操作的时候，会将Record，然后spill到磁盘。这个过程是需要进行排序的。而具体的比较器会根据用户需求的不同使用不同的值。如果设置了keyordering会按照key进行排序。如果没有设置keyordering，但是设置了aggregator(即combine)，则按照key的hashcode进行排序，这样保证相同的key组织一起，便于combine操作。如果keyordering和aggregator都没有设置，则会按照partiton进行排序。

所有Record都写完时，需要读取spill的文件，并合并成一个全局有序的文件。

三种writer的比较

| writer | 优点                                                      | 缺点                                                     | 场景                                                                                         |
| --- |---------------------------------------------------------|--------------------------------------------------------|--------------------------------------------------------------------------------------------|
| BypassMergeSortShuffleWriter | (1) 只经过一次序列化。<br>(2) 采用类hashmap的数据结构，插入数据快。             | (1) 不支持combine和sort <br>(2) 每个分区都要对应生成一个临时文件，会产生过多的临时文件。 | 适合分区数较少(默认小于等于200)且没有combine的的情况.                                                      |
| UnsafeShuffleWriter | (1) 只经过一次序列化。<br>(2) spill到磁盘的文件数目有限，不再基于分区数，可以支持更大的分区。 | (1) 不支持combine, sort <br>(2) 写入顺序Record顺序会打乱，要求supportsRelocationOfSerializedObjects。| 适用于没有combine的情况，且支持supportsRelocationOfSerializedObjects，并且支持最大支持分区数为16777216。 |
| SortShuffleWriter | (1) 支持combine, sort <br> (2) 适合于所有场景 <br> (3)  spill到磁盘的文件数目有限 |     (1) 需要进行多次序列化 |  适用于所有场景。                                                                                          | 

### 1.3.2 shuffle读

当前只有BlockStoreShuffleReader一个实现。实现与MapReduce类似。
Reduce端会从Map端以远程或本地的方式拉取对应分区的Records。正常情况下会直接写到内存中，但如果要获取的block大小超过阈值则会使用磁盘。
然后会根据用户的需求决定是否进行combine或sort，最终形成一个用户要求的record iterator。
combine和sort分别使用了ExternalAppendOnlyMap和ExternalSorter，当内存超过阈值后，会将数据spill到本地磁盘中。

## 1.4 总结

(1) 关于各个框架语义

对于MapReduce和Tez的ordered io, 实质就是spark的排序的特例。对于Tez的unordered io，实质上就是spark的非排序的特例。实质各个框架上语义是相同的, spark更加泛化。

(2) 哪里会产生本地磁盘文件? 

分析三种计算框架后，我们得知如下过程会使用磁盘:
* (1) Map由于内存超越阈值，可能会产生中间的临时文件。
* (2) Map端最终必然会产生磁盘文件，用于提供shuffle服务。
* (3) Reduce拉取records时候，可能因为超越阈值，产生磁盘文件。
* (4) Reduce端Merge的时候，可能会产生临时的磁盘文件，用于全局排序。

而事实上, uniffle已经解决(1), (2)。对于(3), 如果有效的调整参数，是很难产生磁盘文件的。事实上只有(4)是本文需要讨论的。

# 2 需求分析

## 2.1 哪类任务需要远程merge?

当前uniffle的map端操作已经不再需要磁盘操作。本文主要考虑reduce端的情况。主要分如下几种情况:
* (1) 对于spark的非排序且非聚集、tez unordered io，Record是来之即用的，不需要有任何的全局的聚合和排序操作，只需要非常少的内存。当前版本的uniffle在内存设置合理的情况下是不会使用磁盘的。使用当前uniffle的方案即可。本文不会讨论这方面的内容。
* (2) 对于spark的排序或聚集任务、tez ordered io、mapreduce，由于需要全局排序或聚集，内存可能不够用，可能会将Record spill到磁盘。本文主要讨论这种情况。
**综上可知，remote merge仅用于需要排序或聚集的shuffle。**

## 2.2 ShuffleServer如何进行排序?

对于排序类的操作，一般会在Map进行排序得到一组局部排序的记录，这里称之为segment。然后Reduce会获取所有的segment, 并进行归并，Spark, MR, Tez都是用了最小堆K路归并排序。远程排序依旧可以使用这种方式。

BufferManager和FlushManager维护着block在内存和磁盘中的信息。我们只需要在需要在ShuffleServer中新增MergeManager，并将同一个Shuffle下的block进行Merge，得到全局排序的Records即可。

在ShuffleServer端引入排序后产生一个副作用: 即需要将该Shuffle的KeyClass和ValueClass以及KeyComparator传递给ShuffleServer。

## 2.3 ShuffleServer如何进行combine?

Combine一般都是用户自定义的操作，因此禁止ShuffleServer端进行Combine操作。如果在Reduce端进行Combine，岂不是违背了我们避免在任务端进行磁盘操作的主题？事实上我们不必使用ExternalAppendOnlyMap进行combine。如果从ShuffleServer获取的Record是按照key排序的，那么意味着相同的key已经组织在一起了，只需要很少的内存就可以combine。

## 2.4 Writer如何写?

按照现有方式写即可。

## 2.5 Reader如何进行读？

当前uniffle的shuffle reader使用blockid作为读的标记，很容易校验是否得到准确完整的Record。对于远程merge，MergeManager已经将原有的Block集合合并为一个新的按key进行排序的序列。因此不能在使用map段生成的blockid了。
我们将使用一种新的方式读取Records。在MergerManager进行全局Merge的时候，会生成索引。Reader会按照该索引进行读取。

> 注: 原则上使用key作为读索引更符合语义，第一版demo程序也是这么设计的。但是他对于处理数据倾斜的问题不够友好，所以放弃该方案。

# 3 方案设计

<img src="/images/rss/remote_merge_structure.png" width=50% height=50% text-align=center/>

流程如下:
* (1) AM/Driver调用registerShuffle方法，会额外注册keyClass, valueClass和keyComparator.
* (2) Map端生产Records，在RMWriteBufferManager进行排序和conbine，并将block发送给ShuffleServer。当发送完成的时候，调用reportShuffleResult，会额外的发送taskattemptId，主要为了避免推测执行造成数据丛洪福。
* (3) shuffle server端会将数据存在缓存中，或者通过flush manager缓存到本地文件系统或远程文件系统。
* (4) shuffle server端增加一个新的MergeManager，对每个shuffle进行管理。每个partition下会记录Segment信息，每个Segment的数据来自于(3)中提到的内存或文件。当某个Shuffle的所有数据都汇报完成后，会将所有的Segment合并为一个MergedSegment。得到了一个按Key进行排序的Records。在合并最后的MergedSegment的过程中，会添加索引，便于Reduce读取。
* (5) Reducer按照索引顺序地读取MergedSegment，在读取的过程中顺便进行Reducer端的combine。最后将结果交给下游是有。

## 4 计划
主要计划:
* 统一序列化
* ufile, shuffle writer和reader 
* Merger与 Merger Manager开发
* 架构适配
