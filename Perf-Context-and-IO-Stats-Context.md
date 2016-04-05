Perf Context and IO Stats Context can help us understand performance bottleneck of your queries. Compared to `options.statistics`, which stores accumulated statistics across all the operations, Perf Context and IO Stat Context can help us look inside a query.

This is the header file for perf context: https://github.com/facebook/rocksdb/blob/master/include/rocksdb/perf_context.h
This is the header file for IO Stats Context: https://github.com/facebook/rocksdb/blob/master/include/rocksdb/iostats_context.h
Level of profiling for the two is controlled by the same function in this header file: https://github.com/facebook/rocksdb/blob/master/include/rocksdb/perf_level.h

Perf Context and IO Stats Context use the same mechanism. The only difference is Perf Context measures functions of RocksDB, while IO Stats Context measures I/O related calls. They need be enabled in the thread where query to profile is executed. After the profile level is higher than disable, RocksDB will update some counters into a thread-local data structure. After the query, we can read the counters from the thread-local data structure.

## How to Use Them
Here is a typical example of using using Perf Context and IO Stats Context:

``` 
#include “rocksdb/iostat_context.h”
#include “rocksdb/perf_context.h”

rocksdb::SetPerfLevel(rocksdb::PerfLevel::kEnableTimeExceptForMutex);

rocksdb::perf_context.Reset();
rocksdb::iostats_context.Reset();

... // run your query

rocksdb::SetPerfLevel(rocksdb::PerfLevel::kDisable);

... // evaluate or report variables of rocksdb::perf_context and/or rocksdb:iostats_context
```
Note that the same perf level is applied to both of Perf Context and IO Stats Context.

You can also call rocksdb::perf_context.ToString() and rocksdb::iostat_context.ToString() for a human-readable report.

## Profile Levels And Costs

There are several profiling levels to help us trade off the costs:

`kEnableCount` will only enable counters. Stats of time duration are disabled with this level. This is cheap. In many use cases, we measure in this level for all queries, and report them when specific counters are abnormal.

`kEnableTimeExceptForMutex` enables counter stats and most time duration stats, except the cases where timing function needs to be called inside a shared mutex. With this level, RocksDB may call the timing function tons of times for a query. Our common practice is to turn on this log level by sampling, as well as by user request. Users may still need to be careful when choosing sample rates, since costs of timing functions vary across different platforms. Compared to the next level `kEnableTime`, we avoid timing inside shared mutex in this case, so that the extra costs will mostly be introduced to queries being profiled. The impacts on global performance will be limited.

`kEnableTime` enables counter stats and all time duration stats. With this level, we will measure all the stats measured in `kEnableTimeExceptForMutex`, as well as some counters to measure mutex acquiring and waiting time. When we suspect mutex contention is be the bottleneck, we can use this level to figure it out.

Stats will not be updated, if they are not included in the profile level being used. It's the users' responsibility to only use stats being updated. 

## Stats
We are giving some typical examples of  how to use those stats to solve your problem. We are not introducing all the stats here. To get description of all the stats, read the code comments in the header files.

### Perf Context
#### Binary Searchable Costs
`user_key_comparison_count` helps us figure out whether too many comparisons in binary search can be a problem, especially when a more expensive comparator is used. Moreover, since number of comparisons is usually uniform based on the size of memtable size, the SST file size for Level 0 and level size for Level 1+, an significant increase of the counter can indicate unexpected LSM-tree shape. For example, you may want to check whether flush/compaction can keep up with the write speed.

#### Block Cache and OS Page Cache Efficiency
`block_cache_hit_count` tells us how many times we read data blocks from block cache, and `block_read_count` tells us how many times we have to read blocks from the file system (either block cache is disabled or it is a cache miss). We can evaluate the block cache efficiency by looking at the two counters over time.

`block_read_byte` tells us how many total bytes we read from the file system. It can tell us whether a slow query can be caused by reading large blocks from the file system. Index and bloom filter blocks are usually large blocks. A large block can also be the result of a very large value of a key.

In many setting of RocksDB, we rely on OS page cache to reduce I/O from the device. In fact, in the most common hardware setting, we suggest users tune the OS page cache size to be large enough to hold all the data except the last level so that we can limit a ready query to issue no more than one I/O. With this setting, we will issue multiple file system calls but at most one of them end up with reading from devices. To verify whether, it is the case, we can use counter `block_read_time` to see whether total time spent on reading the blocks from file system is expected.

#### Tombstones
When deleting a key, RocksDB simply put a marker, called tombstone to memtable. The original value of the key will not be removed until we compact to the files containing the keys with the tombstone. The tombstone may even live longer even after the original value is removed. So if we have lots of consecutive keys deleted, a user may experience slowness when iterating across these tombstones. Counters `internal_delete_skipped_count` tells us how many tombstones we skipped. `internal_key_skipped_count` covers some other keys we skip. From the counter, we can infer how many tombstones could have been dropped because the original keys are already removed.

#### Get() Break-Down
We can use "get_*" stats to break down time inside one Get() query. The most important two are `get_from_memtable_time` and `get_from_output_files_time`. The counters tell us whether the slowness is caused by memtable, SST tables, or both. `seek_on_memtable_time` can tell us how much of the time is spent on seeking memtables.

#### Write Break-Down
"write_*" stats break down write operations. `write_wal_time`, `write_memtable_time` and `write_delay_time` tell us the time is spent on writing WAL, memtable, or active slow-down. `write_pre_and_post_process_time` mostly means time spent on waiting in the writer queue. It includes the time waiting for another thread to group commits this thread's data together with other data.

#### Iterator Operations Break-Down
"seek_*" and "find_next_user_entry_time" break down iterator operations. The most interesting one is `seek_child_seek_count`. It tells us how many sub-iterators we have, which mostly means number of sorted runs in the LSM tree.

### IO Stats Context
We have counters for time spent on major file system calls. Write related counters are more interesting to look if you see stalls in write paths. It tells us we are slow because of which file system operations, or it's not caused by file system calls.