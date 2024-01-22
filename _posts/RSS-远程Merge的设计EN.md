---
title: RSS-Remote Merge Design
date: 2023-12-25 18:58:15
tags:
---

# 1 Default shuffle

> Note: The first chapter briefly introduces the principle of default shuffle, with the purpose of find where local disks are used, then design remote merge. If you know enough, you can ignore this part.

We will analyze the shuffle of MapReduce, Tez, and Spark in turn.

## 1.1 MapReduce

Map writes the record to the memory. When the memory exceeds the threshold, the memory data is spilled to the disk file, and the Record is written to the disk file in order of partitionid+key. After Map has processed all records, it will spill the data currently in memory to a disk file. Then read all the files spilled to the disk and merge them in the order of partitionid+key to get the sorted Records.

> Note: The purpose of sorting according to partitionid is that when the Reduce side obtains the data from the Map side, it should be read as sequentially as possible. For MR, Tez, and Spark, regardless of whether they are sorted or not, as long as there are partitioned, they need to be sorted according to partitionid.

The reduce will pull the records of the corresponding partition remotely or locally from the Map, which is called MapOutput. Under normal circumstances, the memory will be used directly. If the memory exceeds the threshold, these records will be written to the disk. Then the reduce will perform merge operations on MapOutputs using minimum heap K-way merge sorting to obtain globally sorted records. During the Merge process, temporary results may be spilled to disk because the memory exceeds the threshold. In addition, if there are too many files spilled to disk, additional merges will be triggered.

## 1.2 Tez

There are two cases of tez: (1) ordered io  (2) unordered io.

Ordered io is the same as MapReduce and so ignore it here.

Unordered io is generally used in hashjoin and other situations where keys are not required for sorting. Non-sorted io adopts a ready-to-use solution. Map writes the Record directly to the file or writes it to the file through cache. The Reduce side can also read and use it when reading data.

## 1.3 Spark

Spark's shuffle is more complex and more reasonable. Some tasks do not require sort and combine, so spark users can determine the shuffle logic according to their needs.

### 1.3.1 Shuffle write operation
When writing shuffle data, three writers are supported:

* (1) BypassMergeSortShuffleWriter

A temporary file is generated for each partition. When writing record, find the corresponding partition and write it directly to the corresponding temporary file. Then when all data is processed, these temporary files are written to a final file in order of the partitions, and the temporary files are deleted.

* (2) UnsafeShuffleWriter

UnsafeShuffleWriter mainly implements specific logic through ShuffleExternalSorter. When writing a Record, the serialization operation is performed directly and the serialized bytes are copied to the requested memory. At the same time, the address and partition of the record will also be recorded into the memory (inMemSorter).

When the memory reaches the threshold, spill operation will be performed. Based on the information in memory (inMemSorter), we can easily get a Record sorted by partition and write it to a file.

When all Records are processed, we will spill the records currently in memory into the file. Finally, all spilled files are aggregated once. Since the previously spilled files have been sorted according to the partition, we can copy the corresponding copies of all the spilled files to the final file in the order of the partitions. The final file obtained in this way is the partition-sorted file.

* (3) SortShuffleWriter

SortShuffleWriter mainly implements specific logic through ExternalSorter. ExternalSorter decides whether to combine and sort based on the user's needs.

When writing record, it will be inserted directly into memory. If combine is required, the memory architecture is map, otherwise it is buffer.

If the current evaluation memory is greater than the threshold, the spill operation will be triggered. During the spill operation, the Record will be spilled to the disk. This process requires sorting. The specific comparator will use different values according to different user needs. If keyordering is set, it will be sorted by key. If keyordering is not set, but aggregator (i.e. combine) is set, the keys are sorted according to  the hashcode of key, thus ensuring that the same keys are organized together to facilitate combine operations. If neither keyordering nor aggregator is set, it will be sorted according to partition.

When all Records are written, the spill files need to be read and merged into a globally ordered file.
​
Comparison of three writers

| writer | advantages | disadvantages | scene |
| --- |---|---|---|
| BypassMergeSortShuffleWriter | (1) Only serialized once. <br>(2) Using hashmap-like data structure, inserting data is fast. | (1) Combine and sort are not supported <br>(2) Each partition must generate a temporary file, which will generate too many temporary files. | Suitable for situations where the number of partitions is small (default is less than or equal to 200) and there is no combine. |
| UnsafeShuffleWriter | (1) Only serialized once. <br>(2) The number of files spilled to disk is limited and is no longer based on the number of partitions, and can support larger partitions. | (1) Combine, sort is not supported <br>(2) The writing order Record order will be disrupted, and supportsRelocationOfSerializedObjects is required.  | Applicable to situations where combine does not exist, and supportsRelocationOfSerializedObjects is true, and the maximum number of supported partitions is 16777216. |
| SortShuffleWriter | (1) Supports combine, sort <br> (2) Suitable for all scenarios <br> (3) The number of files spilled to disk is limited | (1) Multiple serializations are required | Suitable for all scenarios. |

### 1.3.2 shuffle read

Currently there is only one implementation of BlockStoreShuffleReader. The implementation is similar to MapReduce.
The reduce will pull the records of the corresponding partition remotely or locally from the map. Under normal circumstances, it will be written directly to the memory, but if the block size to be obtained exceeds the threshold, will use disk.
Then it will be decided according to the user's needs whether to combine or sort, and finally form a record iterator required by the user.
Combine and sort use ExternalAppendOnlyMap and ExternalSorter respectively. When the memory exceeds the threshold, the data will be spilled to the local disk.

## 1.4 Summary

(1) About the semantics of each framework

For MapReduce and the ordered io of Tez, it is a special case of spark sorting. For Tez's unordered io, it is essentially a special case of spark's non-sorting. In essence, the semantics of each framework are the same, and spark is more general.

(2) Where will generate local disk files?

After analyzing the three computing frameworks, we learned that the following processes will use disks:
* (1) Map may generate intermediate temporary files because the memory exceeds the threshold.
* (2) The map will eventually generate disk files to provide shuffle services.
* (3) When reduce pulls records, disk files may be generated because the threshold is exceeded.
* (4) When merging on the reduce side, temporary disk files may be generated for global sorting.

In fact, uniffle has solved (1), (2). For (3), if the parameters are adjusted effectively, it is difficult to generate disk files. In fact, only (4) needs to be discussed in this article.

# 2 Demand analysis

## 2.1 What types of tasks require remote merge?

Currently, uniffle's map-side no longer spill disk. This article mainly considers the situation on the reduce. Mainly divided into the following situations:
* (1) For spark's non-sorted, non-aggregated, tez unordered io. It does not require any global aggregation and sorting operations, and only requires very little memory. The current version of uniffle will not use disk if related settings are reasonable. Just use the current uniffle solution. This article will not discuss this aspect.
* (2) For spark sorting or aggregation tasks, tez ordered io, mapreduce, due to the need for global sorting or aggregation, the memory may not be enough, and the record may be spilled to the disk. This article mainly discusses this situation.
**In summary, it can be seen that remote merge is only used for shuffles that require sorting or aggregation. **
## 2.2 How does ShuffleServer sort?

For sorting, map is generally sorted to obtain a set of partially sorted records, which is called segment here. Then reduce will obtain all segments and merge them. Spark, MR, and Tez all use minimum heap K-way merge sorting. This method can still be used for remote sorting.

BufferManager and FlushManager maintain block information in memory and disk. We only need to add MergeManager to ShuffleServer and merge the blocks under the same Shuffle to obtain globally sorted Records.

Introducing sorting on the ShuffleServer produces a side effect: the Shuffle's KeyClass, ValueClass and KeyComparator need to be passed to ShuffleServer.

## 2.3 How does ShuffleServer combine?

Combine is generally a user-defined operation, so ShuffleServer is prohibited from performing combine operations. If combine is performed on the Reduce side, wouldn't it violate our theme of avoiding spill to disk on the task side? In fact we don't have to use ExternalAppendOnlyMap for combine. If the Records obtained from ShuffleServer are sorted by key, it means that the same keys have been organized together, and only a small amount of memory is needed to combine.

## 2.4 How does Writer write?

Just write it the way we have it.

## 2.5 How does Reader read?

Currently, Uniffle's shuffle reader uses blockid as the read mark, which makes it easy to verify whether an accurate and complete records are obtained. For remote merge, MergeManager has merged the original Block collection into a new sequence sorted records by key. Therefore, the blockid generated by the map segment cannot be used:
We will use a new way to read Records. When MergerManager performs global Merge, an index will be generated. Reader will read according to this index.

> Note: In principle, using key as a read index is more semantic, and the first version of the demo program was also implemented by this way. However, this proposal was not friendly enough to deal with the problem of data skew, so  gave up the plan.

# 3 Scheme Design

<img src="/images/rss/remote_merge_structure.png" width=50% height=50% text-align=center/>

The process is as follows:
* (1) AM/Driver calls the registerShuffle method, which will additionally register keyClass, valueClass and keyComparator.
* (2) The Map side produces records, sorts and combines them in RMWriteBufferManager, and sends the block to ShuffleServer. When the sending is completed, calling reportShuffleResult will additionally send the taskattemptId, mainly to avoid data duplication caused by speculative execution.
* (3) The shuffle server will store the data in the cache, or flush it to the local file system through the flush manager.
* (4) Add a new MergeManager to the shuffle server to manage each shuffle. segment information will be recorded under each partition, and the data of each segment comes from the memory or file mentioned in (3). When all data of a shuffle is reported, all Segments will be merged into one MergedSegment. Got a Records sorted by Key. During the process of merging the last MergedSegment, an index will be added to facilitate Reduce reading.
* (5) Reducer reads MergedSegment in index order, and combines on the Reducer side during the reading process. Finally, the results are handed over to the downstream.
​
# 4 Plan

Main plans:

* Unified serialization
* ufile, shuffle writer and reader
* Development of Merger and Merger Manager
* Architecture adaptation
