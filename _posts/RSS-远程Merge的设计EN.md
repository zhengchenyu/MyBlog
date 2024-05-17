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

# 2 Plans
In order to solve the problem that Merge on the Reduce side may spill to disk, there are two main solutions:
* (1) Merge on Shuffle Server
* (2) Reduce side Merge on demand
## 2.1 Option 1: Merge on ShuffleServer
Move the merge process of reduce to the ShuffleServer side. ShuffleServer will merge the locally sorted Records sent from the map side into a globally sorted records sequence. The reduce side reads directly in the order of the records sequence.
* Advantages: Does not require too much memory and network RPC.
* Disadvantages: Shuffle Server needs to parse Key, Value and Comparator. The Shuffle side cannot combine.

## 2.2 Option 2: On-demand Merge on the Reduce side
<img src="/images/rss/on_demand_merge.png" width=50% height=50% text-align=center/>
Since the memory on the reduce side is limited, in order to avoid spilling data to disk when merging on the reduce side. When reduce obtains segment, it can only read part of the buffer of each segment, and then merge all the buffers. Then when the partial buffer reading of a certain segment is completed, the next buffer of this segment will continue to be read, and this buffer will continue to be added to the merge process.
There is a problem with this. The number of times the Reduce side reads data from ShuffleServer is approximately segments_num * (segment_size / buffer_size), which is a large value for large tasks. Too many RPCs means decreased performance.

> The segment here refers to the sorted record collection, which can be understood as the block in which the records have been sorted according to key.

* Advantages: Shuffle Server does not need to do anything extra.
* Disadvantages: Too many RPCs.

**This article chooses option 1, and the following content mainly discusses option 1. **

# 3 Demand analysis

## 3.1 What types of tasks require remote merge?

Currently, uniffle's map-side no longer spill disk. This article mainly considers the situation on the reduce. Mainly divided into the following situations:
* (1) For spark's non-sorted, non-aggregated, tez unordered io. It does not require any global aggregation and sorting operations, and only requires very little memory. The current version of uniffle will not use disk if related settings are reasonable. Just use the current uniffle solution. This article will not discuss this aspect.
* (2) For spark sorting or aggregation tasks, tez ordered io, mapreduce, due to the need for global sorting or aggregation, the memory may not be enough, and the record may be spilled to the disk. This article mainly discusses this situation.
**In summary, it can be seen that remote merge is only used for shuffles that require sorting or aggregation. **
## 3.2 How does ShuffleServer sort?

For sorting, map is generally sorted to obtain a set of partially sorted records, which is called segment here. Then reduce will obtain all segments and merge them. Spark, MR, and Tez all use minimum heap K-way merge sorting. This method can still be used for remote sorting.

BufferManager and FlushManager maintain block information in memory and disk. We only need to add MergeManager to ShuffleServer and merge the blocks under the same Shuffle to obtain globally sorted Records.

Introducing sorting on the ShuffleServer produces a side effect: the Shuffle's KeyClass, ValueClass and KeyComparator need to be passed to ShuffleServer.

## 3.3 How does ShuffleServer combine?

Combine is generally a user-defined operation, so ShuffleServer is prohibited from performing combine operations. If combine is performed on the Reduce side, wouldn't it violate our theme of avoiding spill to disk on the task side? In fact we don't have to use ExternalAppendOnlyMap for combine. If the Records obtained from ShuffleServer are sorted by key, it means that the same keys have been organized together, and only a small amount of memory is needed to combine.

## 3.4 How does Writer write?

Just write it the way we have it.

## 3.5 How does Reader read?

Currently, Uniffle's shuffle reader uses blockid as the read mark, which makes it easy to verify whether an accurate and complete records are obtained. For remote merge, MergeManager has merged the original Block collection into a new sequence sorted records by key. Therefore, the blockid generated by the map segment cannot be used:
We will use a new way to read Records. When MergerManager performs global Merge, an index will be generated. Reader will read according to this index.

> Note: In principle, using key as a read index is more semantic, and the first version of the demo program was also implemented by this way. However, this proposal was not friendly enough to deal with the problem of data skew, so  gave up the plan.

# 4 Scheme Design

## 4.1 Basic procedures for RemoteMerge

<img src="/images/rss/remote_merge_structure.png" width=50% height=50% text-align=center/>

The following introduces the process of Remote Merge:
* (1) Register
When AM/Driver calls the registerShuffle method, it will additionally register keyClass, valueClass and keyComparator. This information is mainly used by ShuffleServer to parse and sort the Record during merging.
* (2) sendShuffleData
The sendShuffleData logic is basically consistent with existing RSS tasks. The only difference is the use of unified serializers and deserializers, which ensures that ShuffleServer can parse the Record normally no matter which framework it is.
* (3) buffer and flush
The shuffle server will store the data in the cache, or cache it to the local file system or remote file system through the flush manager. The logic of the original ShuffleServer is still reused here.
* (4) reportUniqueBlocks
A new API is provided, reportUniqueBlocks. The Reduce end will deduplicate the blocks generated by the map, and then send the valid block set to ShuffleServer through reportUniqueBlocks. After ShuffleServer receives a valid blocks collection, it will trigger Remote Merge. The results of Remote Merge will be written to the bufferPool like a normal block, and may be flushed to disk when necessary. The result generated by RemoteMerge is an ordinary block, but for convenience of explanation, it is called merged block here. The merged block records the results sorted by key, so when reading the merged block, you need to read it in ascending order in the order of blockid.
* (5) getSortedShuffleData
Reduce will read merged blocks in the order of block serial numbers, and then choose when to use them for reduce calculations based on certain conditions.

## 4.2 Analyze the process from the perspective of Record
We can use WordCount as an example to explain the flow of record in the entire process. In this example, there are two partitions and one reduce, that is, one reduce processes the data of two partitions.

<img src="/images/rss/remote_merge_from_record_perspective.jpg" width=50% height=50% text-align=center/>

* (1) MAP side
After the document data is processed on the map side, it will be sorted. For Map1, since there are two partitions, records with odd numbers as keys will be written to block101, and records with even numbers as keys will be written to block102. The same goes for Map2. Note that the Records in the block here are all sorted.
* (2) Map side sends data
The map side sends the block to ShuffleServer through sendShuffleData, and ShuffleServer stores it in the bufferPool.
What I mean here is that when registering, the app named APP1 will also be registered, and the app named APP1@RemoteMerge will also be registered, which will be introduced later.
* (3) ShuffleServer side Merge
After reduce is started, reportUniqueBlocks will be called to report the available block set, and the corresponding partition in ShuffleServer will be triggered to merge. The result of Merge is a globally sorted record collection under this partition.
Then the question is where are the results of merge stored? The merge process occurs in memory. Whenever a certain number of records are merged, the results are written to a new block. In order to distinguish it from the original appid, this group of blocks will be managed in an appid ending with "@RemoteMerge". The blockid of this new group of blocks increases from 1 and is sorted globally. That is, the records inside each block are sorted, and the records with blockid=1 must be less than or equal to the records with blockid=2.
* (4) Reduce end reading
According to the previous analysis, the reduce side only needs to read the block managed by the appid ending with "@RemoteMerge". When reduce reads a block, it starts from the block with blockid=1 and reads in the order of blockid. We know that when reduce performs calculations, it is calculated in order. Since the data we obtain on the ShuffleServer side is already sorted, we only need to obtain a small amount of data from the ShuffleServer side each time. This enables on-demand reading from the ShuffleServer side, which greatly reduces memory usage.
There are two special situations here, detailed in 5.5.​
# 5 Plan

## 5.1 Unified serializer
Since Merge needs to be performed on the ShuffleServer side, a unified serializer that is independent of the computing framework needs to be extracted. Two types of serializers are extracted here: (1) Writable (2) Kryo. Writable serialization is used for classes that handle the org.apache.hadoop.io.Writable interface, used in the MR and TEZ frameworks. Kryo can serialize most classes and is generally used in the Spark framework.
## 5.2 RecordsFileWriter/RecordFileReader
Provides abstract methods for processing Records
## 5.3 Merger
Provides basic Merger service to merge multiple data streams according to key. Minimum heap K-way merge sorting is used to merge and sort the data streams that have been partially sorted.
## 5.4 MergeManager
Used to merge Records on the server side.
## 5.5 Tools for reading sorted data
Generally speaking, when the Reduce side reads data, it can be sent directly to downstream calculations. But there are two special situations:
(1) For situations where it is necessary to combine in Merge, we need to wait for all the same keys to arrive before combining, and then send them to the downstream.
(2) For spark and tez, the reduce end can read data from multiple partitions. Therefore, we need to merge the data of multiple partitions again on the reduce side, and then send it to the downstream.
RMRecordsReader is a tool for reading sorted data. The general structure is as follows:
<img src="/images/rss/rm_records_reader.png" width=50% height=50% text-align=center/>
The figure depicts a situation where a single Reduce processes two partitions. The RecordsFetcher thread will read the block of the corresponding partition and then parse it into Records. Then send it to combineBuffers. RecordCombiner reads Records from combineBuffer. When all records of a certain key are collected, a combine operation is performed. The result will be sent to mergedBuffer. RecordMerge will obtain all mergedBuffers and then merge and sort them again in memory. Finally, the global sorting results are obtained for downstream.
## 5.6 Framework adaptation
Compatible with MR, Tez, and Spark architectures.

> I has used online application to conduct large-scale stress testing on MR and Tez. Spark has currently only tested some basic examples and still needs a lot of testing.

## 5.7 Isolated classloader
For different versions of keyclass, valueclass and comparatorclass, use isolated classloaders to load them.

> Chinese version document: https://zhengchenyu.github.io/2023/12/25/RSS-%E8%BF%9C%E7%A8%8BMerge%E7%9A%84%E8%AE%BE%E8%AE%A1/