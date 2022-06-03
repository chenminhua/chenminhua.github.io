---
title: "RocksDB"
date: 2020-11-10T20:00:08+08:00
---

## 认识 RocksDB

### RocksDB 是什么?

- 一个嵌入式的 lsm style 的 kv 存储引擎，一个 c++的库。
- key 和 value 都是 byte streams，key 按序存储。
- 高度可定制，能通过组装不同模块的方式适应不同类型的磁盘，负载。
- 能够通过增加 cpu core 以及 IOPs 进行线性 scale。
- 为 flash 和内存做优化。

### RocksDB 不是什么？

- rocksdb 没有 server code，用户需要自己实现 server 的部分来得到 c-s 架构的数据库。
- RocksDB 不是分布式数据库，没有 fault-tolerance 和 replication 机制，也没有实现 data-sharding。
- Not distributed，No failover.No HA, if machine dies you lose your data.

### 为什么需要 RocksDB

1. 基于 flash 存储的 ssd 普及，网络 latency 在 query workload latency 中占据的比例越来越高。embeded 数据库变得受欢迎。
2. dhruba 尝试比较 HBase/HDFS 和 mysql 在 query serving workload 上的表现。经过多次优化后，在机械硬盘上，几 pb 的数据集下，hbase 可以达到比 Mysql 慢两倍的查询速度。但是在 ssd 上，hdfs/hbase 等 hadoop 生态存储在当时还不能高效利用 flash，因此 dhruba 开始试图探索新的存储引擎。（dhruba 开始试图扩展 hdfs/hbase 的能力，使其能 serve query workload。但是随着 flash 的普及，他发现 hdfs 对 flash 的使用效率不高。并且将 hdfs/hbase 改成嵌入式的难度太高，因此他决定开发新的数据库存储引擎。）
3. 当时市面上有一些嵌入式数据库，leveldb 是其中的佼佼者。http://leveldb.googlecode.com/svn/trunk/doc/benchmark.html benchmark 显示，leveldb 是其中最快的一个。但是，leveldb 不是为 server workload 设计的。虽然乍一看 leveldb 的 benchmark 很棒，但是 dhruba 很快意识到，leveldb 是为那些小数据集（小于内存）设计的。

但是 leveldb 存在的问题：

- 单线程 compaction + flush，这导致写入速度不够快，并且还有 stall 的问题，latency p99 太高
- leveldb 不能用到 flash 的所有 IO 能力。

闪存的发展可能会导致我们拥有非常有限的擦除周期的存储，对于这些类型的存储，迫切需要一个可以在读放大，写放大和空间放大之间进行权衡取舍的数据库。 Leveldb 并非旨在实现这些目标。 最好的方法是派生 leveldb 代码并更改其体系结构以满足这些需求。

RocksDB 应运而生。

此时，一些变化正在发生。

Flash storage 来了，随之而来的问题是，Hbase 是否能够高效使用 Flash storage。dhruba 做了一些 benchmark http://hadoopblog.blogspot.com/2012/05/hadoop-and-solid-state-drives.html 并发现 hadoop 在使用 ssd 的时候并不高效；此外，hdfs/hbase 不是嵌入式数据库。于是 dhruba 开始设计新一代的 kv 存储。

## 深入 RockDB 内部

数据按顺序存储。RocksDB 的主要组件有 memtable，sstfile，logfile。当新的写来时，先插入 memtable 和 logfile(WAL，顺序写)。当 memtable 写满了，就把 memtable flush 到 sstfile 中，而相应的 logfile 就可以删除了。

### LSM-Tree

- 1996 年 Patrick O'Neil 发布了论文 The Log-Structured Merge-Tree (LSM-Tree) 。
- leveldb, rocksdb, cassandra, hbase 都是 lsm-tree style 的数据库。
- 数据的写入先写入内存中的某种有序结构（平衡树，或者跳表），然后内存中数据量超过一定阈值后，将内存中数据优化成易于查询的结构并写入文件（当然也是按顺序写入的）。数据的读取则是先读内存中，如果内存中不存在，则去 sst 文件中读取。
- 磁盘上文件的格式： https://github.com/facebook/rocksdb/wiki/Rocksdb-BlockBasedTable-Format

```sh
思考：内存配置多大比较合适？
	Trade off: latency vs stall
思考：为什么要保证顺序？
	scan，point lookup
思考：b+树 vs LSM-Tree。
	顺序读，顺序写，随机读，随机写，scan，point lookup，search。
思考：innodb不是也有redolog么？不也先写内存再flush么（顺序写）？
	即使不flush，innodb的写也是会有随机IO的：先将页加载到内存再写，然后等内存不足或者redolog写满时，flush回文件。当随机写负载上升时，内存页的淘汰会非常频繁。

参考
https://tech.bytedance.net/articles/6877028924220669959?from=email
https://dataintensive.net/   chapter 3-1
```

### Memtable

- RocksDB 自带了 skiplist, vector, prefix-hash 等结构的 memtable。
- vector memtable 适合于 bulk-loading data into db，每个写都是在 vector 末尾追加写，只当要 flush 到存储的时候，才进行排序并写入文件。
- Prefix-hash memtable 支持高效的 get, put 和 scans-within-a-key-prefix。
- 虽然没有开放 api 支持用户写自己的 memtable 并 pulgin 进去，但是也可以在一个 private fork 上写。
- Memtable pipelining： RocksDB 支持配置一组 memtable。当其中一个写满后，就会变成 immutable memtable 并由后台线程将其 flush 到存储上。同时，会分配一个新的 memtable 来 serve 写请求，如果这个 memtable 也很快被写满，而之前的 memtable 还没有完成 flush（写内存太快了，而写存储可能会很慢），这时候 memtable 就会组成流水线，后台线程会一个接一个将这些 memtable 们 flush 到磁盘。
- GC during Flush: flush 过程中，同时会有一个 compaction 的过程，比如对一个 key 存在两次写，则只保留后一个写，或者对同一个 key 做 put 之后接了个 delete，则这个 put 也会被删除。这能缓解空间放大和写放大

### Sstfile

- RocksDB 在磁盘存储的文件就是一系列 sst(Static Sorted Table)格式的文件。
- Table format: plain table, block based table。
- 默认 table type 是 block-based table，默认块大小是 4k。对 In-memory db 来说，plain table 更好。
- https://github.com/facebook/rocksdb/wiki/Rocksdb-BlockBasedTable-Format

#### BlockBasedTable

```sh
<beginning_of_file>
[data block 1]                               （这里都是存有序数据的）
[data block 2]
...
[data block N]
[meta block 1: filter block]                  (see section: "filter" Meta Block)
[meta block 2: index block]
[meta block 3: compression dictionary block]  (see section: "compression dictionary" Meta Block)
[meta block 4: range deletion block]          (see section: "range deletion" Meta Block)
[meta block 5: stats block]                   (see section: "properties" Meta Block)
...
[meta block K: future extended block]  (we may add more meta blocks in the future)
[metaindex block]
[Footer]                               (fixed size; starts at file_size - sizeof(Footer))
<end_of_file>
```

- kv 对就是按序存在 sst 的 data block 部分。每个数据块都按照 block_builder.cc 文件里面定义的格式来存储，并可以选择压缩。
- Data blocks 之后就是 meta blocks。meta block 的格式也在 block_builder.cc 中定义，并可选择压缩。
- Meta index 为每个 meta block 存一个 entry，用于对这些 meta block 的位置进行寻址。
- footer 则存着 metaindex 的寻址位置。

### Logfile

todo

### bloom filter

https://github.com/facebook/rocksdb/wiki/RocksDB-Bloom-Filter

- Bloom filter 是一个 bit array 结构。写入 key 时，对每个 key 用一组固定的 hash func 计算出一个 bit array 的 index，并将该 index 的 bit 置为 1。
- Bloom filter 支持的操作：add(key), check(key)，不能删除。
- False Positive: check(key) == false 一定不存在，check(key) == true 不一定存在。
- 如果设置了 filter policy，则每个新增的 sst 文件都会包含一个 bloom filter，用于确定一个文件是否包含某个我们查询的 key。

```sh
rocksdb::BlockBasedTableOptions table_options;
table_options.filter_policy.reset(rocksdb::NewBloomFilterPolicy(10, false));
my_cf_options.table_factory.reset(
      rocksdb::NewBlockBasedTableFactory(table_options));
```

### Multi-Threaded Compactions

压缩（Compactions）是为了移除 key 的多份数据，以及移除被删除的数据。

在从 memtable flush 到 L0 sst 的时候，rocksdb 进行 compaction。在 compaction 过程中，一些文件也会被周期性的读出并 Merge 成更大的文件进入下一层。

**LSM style 的数据库的写吞吐，直接由压缩决定**。当使用 SSD 时，测试发现多线程 compaction 时，写入速率能打到单线程的 10 倍。

#### Compaction Styles

- 越新的数据存在 L0 上，越老的数据 level 越大。L0 的文件可能会有 overlapping keys，而其他层的文件则不会。
- Level Style Compaction(default)：优化空间放大，每次只将一个文件同下一层的所有和它有 overlapping key 的文件一起 merge。
- Universal Style Compaction：优化写放大。一次 merge 足够多的文件。
- FIFO Style Compaction 会直接丢弃老的文件（cache-like）。

参考：https://www.slideshare.net/meeeejin/rocksdb-compaction

#### Avoiding Stalls

压缩线程也用于 flush memtable。如果所有压缩线程都在忙，这时候大量的写把 memtables 都写满了，会 stall 新的写。可以通过配置一些预留线程专门做 flush 来解决这个问题。

#### Compaction Filter

The RocksDB Compaction Filter gives control to the application to modify the value of a key or to drop a key entirely as part of the compaction process.比如我们想实现 key TTL for expiration.

### RocksDB vs LevelDB

- leveldb 的写速度不够快，rocksdb 通过多线程 compaction 来提高了写入速率。
- Leveldb has stalls，Latency p99 可达到 10s（not designed for server workload）。rocksdb 多线程 flush，并且可预留 flush 线程。
- leveldb 有写放大问题。RocksDB 支持 Universal style compaction 来解决写放大。
- leveldb 在解决 read modify write 问题的时候，用了 2 倍的 IO。RocksDB 支持 mergeRecord，在 compaction 的时候 merge，Only 1x IOs。 https://github.com/facebook/rocksdb/wiki/Merge-Operator

## Potential Use-cases of RocksDB

- 需要低延迟数据库访问的应用程序可以使用 RocksDB。
- 存储网站的查看历史记录和用户状态的面向用户的应用程序可以将该内容存储在 RocksDB 上。
- 需要快速访问大数据集的垃圾邮件检测应用程序可以使用 RocksDB。
- 需要实时扫描数据集的图形搜索查询可以使用 RocksDB。
- RocksDB 可用于从 Hadoop 缓存数据，从而允许应用程序实时查询 Hadoop 数据。
- 支持大量插入和删除的消息队列可以使用 RocksDB。

## Demo

https://github.com/chenminhua/darkbase
rocksdb 性能可随 cpu 数量线性扩展。下载 rocksdb 源码后可以将其编译成静态链接库，让 go 程序链接。
性能测试：当 value 为 1000byte 时，rocksdb 写入 10 万个 kv 只要 800ms，平均每个 8 微秒。读每个只要 3 微秒。不过这 10 万个数据写入一共也才 100 兆数据，磁盘操作非常少。我们还需要大容量测试。

随着写入容量的提升，写入吞吐也会不断下降。

- 在写大容量数据的时候，磁盘本身的性能就会下降。
- 写放大(write amplification)：写入操作还有额外开销，每写入一个单位的数据需要实际读写磁盘的数据大于一个单位。主要体现在 merge 操作。
- 要想提高 rocksdb 写入性能，可以从提高磁盘写入速度和降低写放大系数角度考虑。

## Benchmark

Redis 单机版，单客户端，1000byte value，31000 write qps, 32010 read qps

RocksDB，单机版，写 10 万个 1000byte value，
100000 records put in 689938 usec, 6.89938 usec average
100000 records get in 168648 usec, 1.68648 usec average

如果改成批量写，每 100 个 value 写一次。
100000 records batch put in 149552 usec, 1.49552 usec average, throughput is 668.664 MB/s, rps is 668664

## Reference

- https://www.cs.umb.edu/~poneil/lsmtree.pdf
- https://github.com/facebook/rocksdb/wiki/A-Tutorial-of-RocksDB-SST-formats
- https://github.com/facebook/rocksdb/wiki/Performance-Benchmarks
- http://highscalability.com/blog/2011/8/10/leveldb-fast-and-lightweight-keyvalue-database-from-the-auth.html#:~:text=LevelDB%20writes%20keys%20and%20values,number%20of%20larger%20sorted%20files).
- https://github.com/google/leveldb
- On [Hacker News](https://news.ycombinator.com/item?id=2526032)
- The Log-Structured Merge-Tree (LSM-Tree) by Patrick O'Neil
- Cache Conscious Indexing for Decision-Support in Main Memory
- Bigtable: A Distributed Storage System for Structured Data
- [Hot Trend: Move Behavior To Data For A New Interactive Application Architecture](http://highscalability.com/blog/2010/11/1/hot-trend-move-behavior-to-data-for-a-new-interactive-applic.html)
- https://github.com/facebook/rocksdb/wiki
