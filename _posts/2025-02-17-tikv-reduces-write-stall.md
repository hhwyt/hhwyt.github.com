---
layout: single 
title: How TiKV Reduces Write Stalls Caused by RocksDB SST Ingestion
categories: [Database]
toc: true
toc_label: "Content"
toc_icon: "cog"
---

TiKV adopts RocksDB as its storage backend, utilizing RocksDB's 
methods like [`Write()`](https://github.com/facebook/rocksdb/wiki/RocksDB-Overview#updates) for regular foreground key-value writes and [`IngestExternalFile()`](https://github.com/facebook/rocksdb/wiki/creating-and-ingesting-sst-files) method for bulk data loading. The latter directly ingests SST files into the levels of the RocksDB LSM-tree as low as possible, bypassing the expensive MemTable writes and reducing the number of compactions.


![TiKV architecture]({{ site.url }}{{ site.baseurl }}/assets/images//2025-02-17-tikv-reduces-write-stall/img.png){: .align-center .width-half}
To ensure sequence number consistency, `IngestExternalFile()` triggers a write stall, temporarily pausing foreground writes during data ingestion. This can lead to latency jitter in writes, especially in cluster scaling scenarios where many Regions must be migrated(i.e., their data needs to be ingested to new nodes). In TiKV's architecture, multiple 
Regions on the same node share the same RocksDB instance. When ingestion triggers a write stall, it halts all Regions' write on that node, significantly increasing long-tail write latency. This becomes a challenge in delivering stable performance for our customers.

In this post, we'll explore this challenge in detail and discuss how TiKV addresses it.
## Sequence Number Consistency in RocksDB
RocksDB's sequence number consistency requires that, **for the same key in the LSM-tree, the sequence number of a higher-level key must be greater than that of a lower-level key**. This rule allows reads to return immediately if a key is found at a higher level, without needing to access lower levels. This reduces the cost of accessing lower levels, improving performance and ensuring that reads are up-to-date, thus preventing stale reads.
![RocksDB Ingestion]({{ site.url }}{{ site.baseurl }}/assets/images//2025-02-17-tikv-reduces-write-stall/img_1.png){: .align-center .width-half}

Keys in SST files ingested into RocksDB are assigned sequence numbers also have sequence numbers, typically assigned a global sequence number, which is recorded in the SST file's metadata. To prevent stale reads, SST files can only be ingested into a level of the LSM-tree if there are no overlapping keys at higher levels with smaller sequence numbers.

Another reason for maintaining sequence number consistency is **atomic reads**. Without consistency, a scan that should read only data from the ingested SST files may instead read a mix of stale data (from the MemTable or higher levels) and fresh data (from the ingested files), leading to inconsistencies.
## How IngestExternFile() maintains the Sequence Number Consistency
According to the RocksDB [wiki](https://github.com/facebook/rocksdb/wiki/creating-and-ingesting-sst-files#what-happens-when-you-ingest-a-file), we can see that:
>**When you call DB::IngestExternalFile() We will**
>- ...
>- **Block (not skip) writes to the DB** because we have to keep a consistent db state so we have to make sure we can safely assign the right sequence number to all the keys in the file we are going to ingest
>- If file key range overlap with memtable key range, **flush memtable**
>- Assign the file to the best level possible in the LSM-tree
>- Assign the file a global sequence number
>- **Resume writes to the DB**

This “block-and-resume” process refers to the write stall mentioned earlier. The guarantees provided by this process include:
1. The overlapping key **before ingestion** must have a lower sequence number. If the key is in the MemTable, it will be flushed to L0, and the ingested data will also be placed in L0 but with a greater sequence number. A file with a greater sequence number at the same level is treated as "higher".
2. No concurrent writes occur during this process, as writes are temporarily paused.
3. The overlapping key **after ingestion** must have a greater sequence number, because sequence number increase sequentially.

As mentioned earlier, this approach has its drawbacks and negatively impacts stable performance. Optimizing it could involve reducing the duration of the write stall or eliminating it altogether.
## First Attempt: MemTable Flush Removed from Ingestion Path
The first attempt([Pull Request #3775](https://github.com/tikv/tikv/pull/3775)) aimed to reduce the write stall duration by removing the Memtable flush from ingestion path. Flushing the MemTable is an expensive I/O operation, and eliminating it significantly reduces the duration of the write stall.

This approach utilizing an option provided by RocksDB: `allow_blocking_flush`. When set to false, and if ingestion requires a memtable flush, the `IngestExternalFile()` will fail.

The updated ingestion process works as follows:
- The user first calls IngestExternalFile() with allow_blocking_flush = false.
	- If no MemTable flush is needed, ingestion continues as usual.
	- Else:
		1. IngestExternalFile() fails, and the user must manually call Flush() to flush the MemTable.
		2. After flushing, the user calls IngestExternalFile() again with allow_blocking_flush = true.

In most cases, the flush is handled manually, outside the write stall, effectively removing the MemTable flush from the ingestion path. 

This optimization reduces TiKVs maximum write duration by approximately **100x**, as shown in the figure below:
![(Left part: without optimization, Right part: with optimization)]({{ site.url }}{{ site.baseurl }}/assets/images//2025-02-17-tikv-reduces-write-stall/img_2.png){: .align-center .width-half}
## Second Attempt: No Write Stall Ingestion
While the first optimization significantly reduced write stall duration, there is still room for further performance improvements, as write stalls still occur.

**Can we eliminate the write stall entirely?** What if user can ensure that no concurrent overlapping writes occur?

TiKV is based on Raft, a sequential commit consensus protocol, unlike Paxos. Therefore in TiKV, at any given time, for a specific Region, only one thread can write data at the **Raft layer**. This sequentiality holds true during:
• **Region creation**, where a single thread ingests SST files to load the Region's data.
• **Normal operations**, where a single thread handles foreground writes to the Region.
• **Region destruction**, where a single thread processes the cleanup.

However, there is still a scenario where concurrent writes can occur—**compaction-filter GC**(garbage collection). compaction-filter GC is a low-level operation in the **RocksDB layer**, bypassing Raft's layer.

Compaction-filter GC is a more efficient garbage collection technique integrated into RocksDB's compaction process. It cleans up expired versions of keys during compaction, reducing read and write amplification compared to traditional GC methods, which involve scanning and deleting expired versions using RocksDB's `Write()` API.

Since TiKV uses multiple column families, only keys in the **Write CF** contain version information (commit-ts) that determines the validity of a key. The Write CF is responsible for checking whether a key is expired during LSM-tree compaction. To prevent the **Default CF** from being unaware of key expiration after Write CF compaction-filter GC, GC must also be triggered for the Default CF during this process. This requires calling RocksDB's `Write()` API to perform GC on the Default CF, during Write CF compaction-filter GC which results in **foreground writes** in RocksDB.

To eliminate the write stall while maintaining the safety of compaction-filter GC, the second attempt proposed the following:
1. **Allowing concurrent writes during ingestion**: Introduce an `allow_write` option in RocksDB's IngestExternalFile() function was key. By setting allow_write = true, ingestion could proceed without triggering the write stall. RocksDB users must ensure that no concurrent overlapping writes occur during ingestion.
2. **Using a range latch for mutual exclusion**: Introduce a range latch to prevent conflicts between compaction-filter GC and data ingestion. The latch requires that both processes acquire the lock on the key range before proceeding. This guarantees that:
	- Writes triggered by compaction-filter GC won't interfere with ingestion.
	- Ingestion can safely set allow_write = true after acquiring the latch, avoiding the write stall.

With this optimization, in the TPCC scenario, the P9999 write thread wait, which used to measured write stall, has been reduced by over **90% ~ 96%:
![]({{ site.url }}{{ site.baseurl }}/assets/images//2025-02-17-tikv-reduces-write-stall/img_3.png){: .align-center .width-half}


TiKV's P99 Write Wait Latency (referred to as Apply Wait Latency in TiKV) has been reduced by **~50%**:
![]({{ site.url }}{{ site.baseurl }}/assets/images//2025-02-17-tikv-reduces-write-stall/img_4.png){: .align-center .width-half}

## Summary
This post describes TiKV's efforts to reduce write stalls during data ingestion. The first attempt removed the MemTable flush from the ingestion path, significantly cutting the write stall duration and reducing the maximum write duration by 100x. The second optimization introduced the allow_write option in RocksDB and implemented a range latch to prevent concurrent writes during compaction-filter GC and ingestion, effectively eliminating write stalls and improving performance. This optimization reduced the P9999 write thread wait latency by over 90% and P99 Write Wait Latency by ~50%.