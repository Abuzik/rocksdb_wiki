## 1. Introduction
  
The RocksDB project started at [Facebook](https://www.facebook.com/Engineering) as an experiment to develop an efficient database software that can realize the full potential of storing data on fast storage ( especially Flash storage) for server workloads. It is a C++ library and may be used to store keys and values, which are arbitrarily-sized byte streams. It supports atomic reads and writes. 

RocksDB features highly flexible configuration settings that may be tuned to run on a variety of production environments, including pure memory, Flash, hard disks or HDFS. It supports various compression algorithms and good tools for production support and debugging. 
  
RocksDB borrows significant code from the open source [leveldb](https://code.google.com/p/leveldb/) project as well as significant ideas from [Apache HBase](http://hbase.apache.org/). The initial code was forked from open source leveldb 1.5. It also builds upon code and ideas that were developed at Facebook before RocksDB.

## 2. Assumptions and Goals

### Performance:
The primary design point for RocksDB is that it should be performant for fast storage and for server workloads. It should exploit the full potential of high read/write rates offered by Flash or RAM subsystems. It should support efficient point lookups as well as range scans. It should be configurable to support high random-read workloads, high update workloads or a combination of both. Its architecture should support easy tuning of Read Amplification, Write Amplification and Space Amplification.

### Production Support:
RocksDB should be designed in such a way that it has built-in support for tools and utilities that help deployment and debugging in production environments. Most major parameters should be fully tunable so that it can be used by different applications on different hardware.

### Backward Compatibility:
Newer versions of this software should be backward compatible, so that existing applications do not need to change when upgrading to newer releases of RocksDB. 

## 3. High Level Architecture

RocksDB is an embedded key-value store where keys and values are arbitrary byte streams. RocksDB organizes all data in sorted order and the common operations are `Get(key)`, `Put(key)`, `Delete(key)` and `Scan(key)`.

The three basic constructs of RocksDB are _memtable_, _sstfile_ and _logfile_. The _memtable_ is an in-memory data structure - new writes are inserted into the _memtable_ and are optionally written to the _logfile_. The _logfile_ is a sequentially-written file on storage. When the _memtable_ fills up, it is flushed to a _sstfile_ on storage and the corresponding _logfile_ can be safely deleted.  The data in an _sstfile_ is kept sorted to facilitate easy lookup of keys.

The format of a default _sstfile_ is described in more details [here](https://github.com/facebook/rocksdb/wiki/Rocksdb-BlockBasedTable-Format).

## 4. Features

### Gets, Iterators and Snapshots
Keys and values are treated as pure byte streams. There is no limit to the size of a key or a value. The `Get` API allows an application to fetch a single key-value from the database. The `MultiGet` API allows an application to retrieve a bunch of keys from the database. All the keys-values returned via a `MultiGet` call are consistent with one-another.

All data in the database is logically arranged in sorted order. An application can specify a key comparison method that specifies a total ordering of keys. An `Iterator` API allows an application to do a `RangeScan` on the database. The `Iterator` can seek to a specified key and then the application can start scanning one key at a time from that point. The `Iterator` API can also be used to do a reverse iteration of the keys in the database. A consistent-point-in-time view of the database is created when the Iterator is created. Thus, all keys returned via the Iterator are from a consistent view of the database.

A `Snapshot` API allows an application to create a point-in-time view of a database. The `Get` and `Iterator` APIs can be used to read data from a specified snapshot. In some sense, a `Snapshot` and an `Iterator` both provide a point-in-time view of the database. But their implementations are different. Short lived scans are best done via an iterator while long-running scans are better used via a snapshot. An iterator keeps a reference count on all underlying files that correspond to that point-in-time-view of the database, these files will not get deleted till the Iterator is released. On the other hand, a snapshot does not prevent files deletions; instead the compaction process understands the existence of snapshots and promises to never delete a key that is visible in any existing snapshot.

Snapshots are not persisted across database restarts: a reload of the RocksDB library (via a server restart) would release all pre-existing snapshots.

### Prefix Iterators
Most LSM engines cannot support an efficient `RangeScan` API because it needs to look into every data file. But most applications do not do pure-random scans of key ranges in the database; instead applications typically scan within a key-prefix. RocksDB uses this to its advantage. Applications can configure a `prefix_extractor` to specify a key-prefix. RocksDB uses this to store blooms for every key-prefix. An iterator that specifies a prefix (via ReadOptions) will use these bloom bits to avoid looking into data files that do not contain keys with the specified key-prefix.

### Updates
There is a `Put` API that can insert a single key-value to the database. A `Write` API allows multiple keys-values to be inserted atomically into the database. The database guarantees that the keys-values in a single `Write` call will either all be inserted into the database or none of them will be inserted into the database. 

### Persistency
RocksDB has a transaction log. All Puts are stored in a in-memory buffer called the memtable as well as optionally inserted into a transaction log. Each `Put` has a set of flags specified via `WriteOptions`. The `WriteOptions` can specify whether the `Put` should be inserted into the transaction log. Also, the `WriteOptions` can specify whether a sync call is issued to the transaction log before a `Put` is declared to be committed. 

Internally, RocksDB uses a batch-commit mechanism to batch transactions into the transaction log so that it can potentially commit multiple transactions using a single sync call.

### Fault Tolerance
RocksDB uses a checksum to detect corruptions in the storage. These checksums are for each block; a block is typically anywhere between `4K` to `128K` in size. A block, once written to storage, is never modified. RocksDB will dynamically detect hardware support for checksum computations and avail that support wherever available. 

### Multi-Threaded Compactions
Compactions are needed to remove multiple copies of the same key that can occur if an application overwrites an existing key. Compactions also process deletions of keys. Compactions can occur in multiple threads if configured appropriately. The overall write throughput of an LSM database is directly dependent on the speed at which compactions can occur, especially when the data is stored in fast storage like SSD or RAM. RocksDB can be configured to issue compaction requests from multiple threads concurrently. It is observed that sustained write rates can increase upto a factor of 10 with multi-threaded compaction when the database is on SSDs compared to single-threaded compactions. 

The entire database is stored in a set of _sstfiles_. When a _memtable_ is full, its content is written out to a file in Level-0 (L0). RocksDB removes duplicates and overwritten keys in the memtable when it is flushed to a file in L0. Periodically, some files are read in and merged to form larger files. This is called Compaction. 

RocksDB supports two different styles of compaction. The Universal Style Compaction stores all files in L0 and all files are arranged in time order. A compaction picks up a few files that are chronologically adjacent to one another and merges them back into a new file L0. All files can have overlapping keys.

The Level Style Compaction stores data in multiple levels in the database. The more recent data is in L0 and the earliest data is in Lmax. Files in L0 can have overlapping keys but files in other layers do not have overlapping keys. A compaction process picks one file in Ln and all its overlapping files in Ln+1 and replaces them with new files in Ln+1. The Universal Style Compaction typically results in lower write amplification but higher space amplification than Level Style Compaction.

There is a `MANIFEST` file in the database that records the state of the database. The Compaction process adds new files and deletes existing files from the database, and it makes these operations persistent by recording them in the `MANIFEST` file. Transactions to be recorded in the `MANIFEST` file uses a batch-commit algorithm to amortize the cost of repeated syncs to the `MANIFEST` file.

### Avoiding Stalls
Background compaction threads are also used to flush _memtable_ contents to a file on storage. If all background compaction threads are busy doing long-running compactions, then a sudden burst of writes can fill up the _memtable_(s) quickly, thus stalling new writes. This situation can be avoided by configuring RocksDB to keep a small set of threads explicitly reserved for the sole purpose of flushing _memtable_ to storage. 

### Compaction Filter
There are times when an application would like to process keys at compaction time. For example, a database that has inherent support for Time-To-Live (ttl) can optionally remove keys that are expired. This can be done via an application defined Compaction Filter. If the application wants to continuously delete data older than a specific time, then it can use the compaction filter to drop records that have expired. The RocksDB Compaction Filter gives control to the application to modify the value of a key or to entirely drop a key as part of the compaction process. For example, an application can continuously run a data sanitizer as part of the compaction.

### ReadOnly Mode
There are times when an application wants to open a database for reading only. It can open the database in ReadOnly mode, the database guarantees that the application won't be able to modify anything in the database. This results in much higher read performance because oft-traversed code paths can avoid locks completely.

### Database Debug Logs
The RocksDB database software writes detailed logs to a file named LOG*. These are mostly used for debugging and analyzing a running system. There is a configuration to allow rolling this LOG at a specified periodicity.

### Data Compression
RocksDB supports snappy, zlib and bzip2 compression. RocksDB can be configured to support different compression algorithms at data in different levels. Typically, `90%` of data in the L-max level. A typical installation might configure no-compression for levels L0-L2, snappy compression for the mid levels and zlip compression for Lmax.

### Transaction Logs ###
RocksDB stores transactions into _logfile_ to protect against system crashes. On restart, it re-processes all the transactions that were recorded in the _logfile_. The _logfile_ can be configured to be stored in a directory different from the directory where the _sstfile_s are stored. This is necessary for those cases where you might want to store all data files in non-persistent fast storage, while at the same time, you can ensure no data loss by putting all transaction logs on slower but persistent storage.

### Full Backups, Incremental Backups and Replication
Rocksdb has support for full backups and incremental backups. Rocksdb is an LSM database engine, so data files once created are never overwritten and this makes it easy to extract a list of file-names that correspond to a point-in-time snapshot of the database contents. The API `DisableFileDeletions` instructs RocksDB to not delete data files. Compactions will continue to occur, but files that are not needed by the database will not be deleted. An backup application can then invoke the API `GetLiveFiles`/`GetSortedWalFiles` to retrieve the list of live files in the database and copy them to a backup location. Once the backup is complete, the application can invoke `EnableFileDeletions`; the database is now free to reclaim all the files that are not needed any more. 

Incremental Backups and Replication need to be able to find and _tail_ all the recent changes to the database. The API `GetUpdatesSince` allows an application to _tail_ the RocksDB transaction log. It can continuously fetch transactions from the RocksDB transaction log and apply them to a remote replica or a remote backup. 

A replication system typically wants to annotate each Put with some arbitrary metadata. This metadata may be used to detect loops in the replication pipeline. It can also be used to timestamp and sequence transactions. For this purpose, RocksDB supports an API called `PutLogData` that an application can use to annotate each Put with metadata. This metadata is stored only in the transaction log and is not stored in the data files. The metadata inserted via `PutLogData` can be retrieved via the `GetUpdatesSince` API.

RocksDB transaction logs are created in the database directory. When a log file is no longer needed, it is moved to the archive directory. The reason for the existence of the archive directory is because a replication stream that is falling behind might need to retrieve transactions from a log file that is way in the past. The API `GetSortedWalFiles` returns a list of all transaction log files.

### Support for Multiple Embedded Databases in the same process
A common use-case for applications that use RocksDB is that they inherently partition their data set into multiple logical partitions or shards. This technique is helpful for application load balancing and fast recovery from faults. This means that a single server process should be able to operate multiple RocksDB databases simultaneously. This is done via an environment object named `Env`. Among other things, a thread pool is associated with an `Env`. If applications want to share a common thread pool (for background compactions) among multiple database instances, then it should use the same `Env` object for opening those databases.

Similarly, there is support for multiple database instances to share the same block cache.

### Block Cache -- Compressed and Uncompressed Data
RocksDB uses a LRU cache for blocks to serve reads. The block cache is partitioned into two individual caches: the first one caches uncompressed blocks and the second one caches compressed blocks in RAM. If a compressed block cache is configured, then the database intelligently avoids caching data in the OS buffers.

### Table Cache
The Table Cache is a construct that caches open file descriptors. These file descriptors are for __sstfiles__. An application can specify the maximum size of the Table Cache.

### External Compaction Algorithms
The performance of an LSM database has a significant dependency on the compaction algorithm and implementation. RocksDB has two supported compaction algorithms: LevelStyle and UniversalStyle. But we would like to enable the large community of developers to develop and experiment with other compaction policies. For this reason, RocksDB has appropriate hooks to switch off the inbuilt compaction algorithm and has other APIs to allow an application to operate their own compaction algorithms. Options.disable_auto_compaction, if set, disables the inbuilt compaction algorithm. The `GetLiveFilesMetaData` API allows an external component to look at every data file in the database, decide on which data files to merge and compact, and the `DeleteFile` API allows it to delete data files that are deemed obsolete.

### Non-Blocking Database Access
There are certain applications that are architected in such a way that they would like to retrieve data from the database only if that data retrieval call is non-blocking, i.e. the data retrieval call does not have to read in data from storage. RocksDB caches a portion of the database in the block cache and these applications would like to retrieve the data only if it is found in this block cache. If this call does not find the data in the block cache then RocksDB returns an appropriate error code to the application. The application can then schedule a normal Get/Next operation understanding that fact that this data retrieval call could potentially block for IO from the storage (maybe in a different thread context).

### Stackable DB
RocksDB has a built-in wrapper-mechanism to layer additional functionality as a layer above the code database kernel.  This functionality is encapsulated by an API called `StackableDB`. For example, the time-to-live functionality is implemented by a `StackableDB` and is not part of the core RocksDB API. This approach keeps the code modularized and clean.

### Backupable DB
One of the features implemented using `StackableDB` interface is `BackupableDB`, which makes backing up RocksDB simple. You can read more about it here: [How to backup RocksDB?](https://github.com/facebook/rocksdb/wiki/How-to-backup-RocksDB%3F)
### Memtables:
#### Pluggable Memtables:
The default implementation of the memtable for RocksDB is a skiplist. The skiplist is an sorted set. The sorted set is a necessary construct when the workload interleaves writes with range-scans. But there are some applications that do not interleave their writes and scans and there are other applications that do not do range-scans altogether. For these applications, a sorted set might not provide the optimal performance. For this reason, RocksDB supports a pluggable API that allows an application to provide their own implementation of a memtable. There are three memtables that are part of the library: a skiplist memtable, a vector memtable and a prefix-hash memtable. A vector memtable is appropriate for bulk-loading data into the database. Every write inserts a new element at the end of the vector; when it is time to flush the memtable to storage the elements in the vector are sorted and written out to a file in L0. A prefix-hash memtable allows efficient processing of gets, puts and scans-within-a-key-prefix.

#### Memtable Pipelining
RocksDB supports configuring an arbitrary number of memtables for a database. When a memtable is full it becomes an immutable memtable and a background thread starts flushing its contents to storage. Meanwhile, new writes continue to accumulate to a newly allocated memtable. If the newly allocated memtable is filled up to its limit, it too is converted to an immutable memtable and inserted into the flush pipeline. The background thread continues to flush all the pipelined immutable memtables to storage. This pipelining increases write throughput of RocksDB especially when it is operating on slow storage devices.

#### Memtable Compaction:
When a memtable is being flushed to storage, an inline-compaction process removes duplicate records from the output steam. Similarly, if an earlier put is hidden by a later delete, then the put is not written to the output file at all. This feature reduces the size of data on storage and write amplification greatly. This is an essential feature when RocksDB is used as a producer-consumer-queue, especially when the lifetime of an element in the queue is very short-lived.

### Merge Operator
RocksDB natively supports three types of records, a `Put` record, a `Delete` record and a `Merge` record. When a compaction process encounters a Merge record, it invokes an application-specified method called the Merge Operator. The Merge can combine multiple Put and Merge records into a single one. This is a powerful feature and allows applications that typically do read-modify-writes to completely avoid the reads. It allows an application to record the intent-of-the-operation as a Merge Record and the RocksDB compaction process lazily applies that intent to the original value. This feature is described in detail in [Merge Operator](https://github.com/facebook/rocksdb/wiki/Merge-Operator)

## 5. Tools
There are a number of interesting tools that are used to support a database in production. The `sst_dump` utility dumps all the keys-values in a sst file.  The `ldb` tool can put, get, scan the contents of a database. `ldb` can also dump contents of the `MANIFEST`, it can also be used to change the number of configured levels of the database. It can be used to manually compact a database.

## 6. Tests
There are a bunch of unit tests that test specific features of the database. A `make check` command runs all unit tests. The unit tests trigger specific features of RocksDB and are not designed to test data correctness at scale. The `db_stress` test is used to validate data correctness at scale.

## 7. Performance
RocksDB performance is benchmarked via a utility called `db_bench`. `db_bench` is part of the RocksDB source code. Performance results of a few typical workloads using Flash storage are described [here](https://github.com/facebook/rocksdb/wiki/Performance-Benchmarks). You can also find RocksDB performance results for in-memory workload [here](https://github.com/facebook/rocksdb/wiki/RocksDB-In-Memory-Workload-Performance-Benchmarks).

##### Author: Dhruba Borthakur