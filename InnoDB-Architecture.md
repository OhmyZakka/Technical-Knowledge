# MySQL 8.0 版本 InnoDB 内存架构

## 一、InnoDB 内存架构的整体视图

从 InnoDB 角度看，关键内存组件可以这样分层理解：

1. **Buffer Pool（缓冲池）**
   - 占用内存最大的一块，一般建议配置为物理内存的 60–80%。
   - 缓存表数据页、索引页、Undo 页，以及 Change Buffer、Adaptive Hash Index 等内部结构。[Google Cloud+1](https://cloud.google.com/mysql/memory-usage?utm_source=chatgpt.com)
2. **Change Buffer（变更缓冲）**
   - 专门缓存“**二级索引叶子页**”的变更，如果对应页不在 Buffer Pool，就先写到 Change Buffer，后续再合并。[dev.mysql.com](https://dev.mysql.com/doc/refman/8.2/en/innodb-change-buffer.html?utm_source=chatgpt.com)
   - 目的是减少随机 I/O，对大批量 INSERT/UPDATE/DELETE 的场景很有用。
3. **Adaptive Hash Index（自适应哈希索引，AHI）**
   - 观察 B+Tree 索引访问模式，在热点范围上自动建立**内存哈希索引**，加速等值查找。[dev.mysql.com](https://dev.mysql.com/doc/refman/8.3/en/innodb-adaptive-hash.html?utm_source=chatgpt.com)
   - 放在 Buffer Pool 里，占用 buffer 的一部分空间。
4. **Log Buffer（重做日志缓冲区）**
   - 存放 redo log 记录的内存区域，在刷到 redo log 文件前，修改先进入 log buffer。[Google Cloud](https://cloud.google.com/mysql/memory-usage?utm_source=chatgpt.com)
5. **其他 InnoDB 内部内存结构（通常不在“官方四件套”图里单独画）**：
   - **Data Dictionary / Table Cache**：缓存 InnoDB 数据字典、表和索引的元数据（8.0 之后字典完全持久化，但仍有大块内存 cache）。
   - **事务与锁系统**：trx_t、锁对象（lock_t）、MVCC read view 等结构。
   - **后台线程相关队列、IO 控制结构**：如 flush 队列、IO slot 等。



## 二、Buffer Pool：InnoDB 的“内存主战场”

### 2.1 主要作用

**Buffer Pool 是 InnoDB 最核心的缓存层**，主要功能：[Google Cloud+1](https://cloud.google.com/mysql/memory-usage?utm_source=chatgpt.com)

- 缓存各类 **页（Page）**：
  - 数据页（聚簇索引页）
  - 二级索引页
  - Undo 页
  - 自适应哈希索引、变更缓冲等内部使用的页
- 所有对表数据的读写，都是先落在 Buffer Pool 页上进行：
  - 读：如果命中缓冲页，直接返回；未命中再从磁盘读入。
  - 写：先改缓冲页，标记为 dirty，再写 redo log，最后通过后台 Checkpoint 刷回磁盘。

Buffer Pool 大小由 `innodb_buffer_pool_size` 控制，并且 8.0 可以在线调整；实际 InnoDB 会额外为管理结构多申请约 10% 的空间。[Google Cloud+1](https://cloud.google.com/mysql/memory-usage?utm_source=chatgpt.com)

### 2.2 Buffer Pool 的内部组织

1. **按 Instance/Chunk 切分**
   - 参数：
     - `innodb_buffer_pool_instances`：把 buffer pool 切成多个实例，每个实例维护独立的 LRU/Free/Flush 链表，降低高并发下的 latch 竞争。
     - `innodb_buffer_pool_chunk_size`：每个 Chunk 的大小，整个 buffer pool = 实例数 × chunk_size 的整数倍。[dev.mysql.com](https://dev.mysql.com/doc/refman/8.1/en/innodb-parameters.html?utm_source=chatgpt.com)
2. **三大链表：Free / LRU / Flush**（典型 InnoDB 架构）
   - **Free List**：空闲页链表，启动时所有缓存页都在这里；读到新页时，从 Free 拿一个页使用。
   - **LRU List**：最近最少使用链表，核心用于淘汰冷数据；InnoDB 用的是“**中点插入的变体 LRU**”：
     - 把 LRU 分成 Young（新子链表）和 Old（老子链表），典型比例是 5/8 : 3/8；
     - 新读入的页先插到 Old 区中部，访问命中多次才进入 Young 区，避免一次大表扫描把热数据全刷出去。
   - **Flush List**：所有 Dirty 页都会挂在 Flush List 上，按照 LSN 顺序组织，供后台 Checkpoint/刷盘线程按策略刷新到磁盘。
3. **Page 内部 + 控制块**
   - 每个 Page 有一块“控制块”记录元信息（space_id、page_no、LSN、链表指针、锁信息等），控制块 + page 数据本身都在 Buffer Pool 内，控制块大约占 5% 左右的额外空间。

### 2.3 Buffer Pool 里到底放了什么？

典型来说，Buffer Pool 的内容可以粗分为：[Google Cloud+1](https://cloud.google.com/mysql/memory-usage?utm_source=chatgpt.com)

- 表/索引的 **数据页、索引页**（绝大多数）
- **Undo 页**
- **Change Buffer 页**
- **Adaptive Hash Index 的结构（依附在页上）**
- 部分 **锁信息、控制块** 等

这一点很重要：**Change Buffer 和 AHI 都“寄生”在 Buffer Pool 上**，不是额外分出一块独立大内存。



## 三、Change Buffer：二级索引更新的“缓冲区”

### 3.1 定义与位置

官方定义：**Change Buffer 是缓存在 Buffer Pool 内的一种特殊数据结构，用来缓存对“二级索引页” 的修改，当对应页不在 Buffer Pool 时使用**。[dev.mysql.com+1](https://dev.mysql.com/doc/refman/8.2/en/innodb-change-buffer.html?utm_source=chatgpt.com)

- **内存中**：Change Buffer 占用 Buffer Pool 的一部分页；
- **磁盘上**：Change Buffer 的持久化内容是 **System Tablespace（ibdata）的一部分**。

### 3.2 解决什么问题？

二级索引页的访问特征往往是“**随机 + 不一定常用**”：

- 对很多大表来说，写入/更新会频繁命中聚簇索引的相邻页（已缓存在 buffer pool 里），但二级索引的叶子页散得比较开。
- 如果每次修改二级索引都强制把那个页从磁盘读入：会制造大量随机 I/O。

Change Buffer 的思路就是：

> 当目标二级索引页不在 Buffer Pool 时，先把“要对它做的变更”缓存起来，等将来这个页因为读操作被读入 buffer pool 时，再把这些变更一起 merge 进去。

合并操作发生的时机：[docs.oracle.com](https://docs.oracle.com/cd/E17952_01/mysql-8.0-en/faqs-innodb-change-buffer.html?utm_source=chatgpt.com)

- 页被真正读入 Buffer Pool 时；
- 后台空闲时的 purge/merge 线程；
- 慢关机或崩溃恢复阶段。

### 3.3 使用限制与配置

- 只对 **二级索引的叶子页** 起作用；
- 对 **唯一索引**、含 descending 列的索引可能不启用 change buffering。[dev.mysql.com](https://dev.mysql.com/doc/refman/8.2/en/innodb-change-buffer.html?utm_source=chatgpt.com)
- 通过变量控制：
  - `innodb_change_buffering`：控制是否 buffer insert / delete-mark / purge 等，8.0 默认值是 `none`（即默认关闭变更缓冲，更多交给 cache 本身处理）。[dev.mysql.com+1](https://dev.mysql.com/doc/refman/8.2/en/innodb-change-buffer.html?utm_source=chatgpt.com)
  - `innodb_change_buffer_max_size`：限定 change buffer 最多占 Buffer Pool 的百分比（默认 25%，最大 50%）。[dev.mysql.com](https://dev.mysql.com/doc/refman/8.2/en/innodb-change-buffer.html?utm_source=chatgpt.com)

**权衡**：

- 写入很多、二级索引多且工作集大于 Buffer Pool：开启 change buffering 往往能显著减 I/O；
- 读多写少、或数据基本能放进 Buffer Pool：开了收益不大，反而占 buffer，甚至拖慢查询。



## 四、Adaptive Hash Index：在热点上“叠一层哈希”

### 4.1 目的和原理

AHI 解决的是：**已经在内存里的 B+Tree 查找还能不能再快一点**。

- InnoDB 观察索引访问模式，当发现某个范围被频繁按相同前缀查找时，会在对应页上构建一个 **哈希表项**：
  - Key：索引键值或其前缀；
  - Value：指向 Buffer Pool 中对应页 / 行的指针。[dev.mysql.com](https://dev.mysql.com/doc/refman/8.3/en/innodb-adaptive-hash.html?utm_source=chatgpt.com)
- 下次再查相同 key 时，可以走哈希直接命中，而不是 B+Tree 多层查找。

它本质上是一个 **运行期自学习的加速层**：

- 构建和维护 AHI 需要开销，所以 InnoDB 动态监控命中率，不合算就削弱使用，必要时可以关闭。[dev.mysql.com](https://dev.mysql.com/doc/refman/8.3/en/innodb-adaptive-hash.html?utm_source=chatgpt.com)

### 4.2 内存结构 & 并发

- AHI 存储在 Buffer Pool 占用的一些内存结构中；
- 为减少大哈希表上的 latch 竞争，引入了分区：
  - `innodb_adaptive_hash_index_parts`：把 AHI 分成若干 partition，每个 partition 有独立 latch，默认 8、最大 512。[dev.mysql.com](https://dev.mysql.com/doc/refman/8.3/en/innodb-adaptive-hash.html?utm_source=chatgpt.com)
- 开关：`innodb_adaptive_hash_index` ON/OFF，关闭时会清空哈希表。[dev.mysql.com](https://dev.mysql.com/doc/refman/8.3/en/innodb-adaptive-hash.html?utm_source=chatgpt.com)

**适用场景**：

- 点查 OLTP（`WHERE pk = ?` 或二级索引精确查）多、工作集能大致放进 Buffer Pool 的系统，常常能有收益。
- 复杂 JOIN、大量 Range Scan 或超级高并发写入场景，有时 AHI 反而会变成锁竞争点，需要评估是否关闭。



## 五、Log Buffer：Redo Log 的“写前缓冲”

### 5.1 作用

**Log Buffer 是一块连续内存区域，用来暂存 redo log 记录**：[Google Cloud+1](https://cloud.google.com/mysql/memory-usage?utm_source=chatgpt.com)

- 事务在修改 Buffer Pool 中的页的同时，会生成 redo log 记录写入 log buffer；
- 根据 `innodb_flush_log_at_trx_commit` 策略决定在事务提交时是否立刻把 log buffer 刷到 redo log 文件：
  - `1`：每次提交都刷盘（最安全，写多时 I/O 压力大）；
  - `2`：提交写 OS cache，周期性 fsync；
  - `0`：周期性写 & fsync，允许一定量数据在崩溃时丢失。

### 5.2 大小和调优

- 默认大小一般为 16MB（社区版本），由 `innodb_log_buffer_size` 控制。[Google Cloud+1](https://cloud.google.com/mysql/memory-usage?utm_source=chatgpt.com)
- 过小会导致长事务频繁刷 log buffer，引发 `Innodb_log_waits` 增长；
- 写入非常频繁的场景，常见做法是把 `innodb_log_buffer_size` 调到 32MB 甚至 64MB 以减少等待。[Medium+1](https://duthaho.medium.com/understanding-the-mysql-innodb-buffer-pool-a-deep-dive-for-performance-optimization-0fa6b52ffb21?utm_source=chatgpt.com)

Log Buffer 与 Buffer Pool 的协作是 InnoDB WAL 机制的核心：

> 先写 log buffer → 刷到 redo log → 再异步刷脏页到数据文件，保障 `crash-safe`。



## 六、其他关键 InnoDB 内存结构

除了上面四块，InnoDB 在 8.0 里还有一些不太显眼、但非常重要的内存结构：

1. **Data Dictionary / Table Cache**
   - InnoDB 持久化数据字典（表、索引、列、外键等定义）存在系统表空间 / 专门字典表中；
   - 内存中维护一套 dictionary cache 和 table cache，供执行层快速获取元数据，而不用频繁访问磁盘。[DataBase Administration+1](https://deepakmysqldba.wordpress.com/2021/05/09/innodb-mysql-8-architecture/?utm_source=chatgpt.com)
2. **事务与锁系统内存**
   - 每个事务一个 trx_t 结构，记录隔离级别、read view、undo 链接等；
   - 每个锁一个 lock_t，挂在全局锁表上（也在内存中），配合行记录上的 lock bit 位实现行级锁、gap 锁等。
3. **后台线程队列 / I/O 管理结构**
   - 包括 flush 任务队列、IO slot、异步 IO 完成队列等，用于协调多线程刷盘和读写。

这些结构通常不通过简单的“大小参数”直接配置，但会随连接数、并发事务数、表/索引个数线性增长，所以在做容量规划时也要预留 10–20% 的额外内存空间。



## 七、从一次事务视角看 InnoDB 内存的配合

以一条 `UPDATE` 为例，从内存角度看看各组件怎么配合工作：

1. SQL 到达 InnoDB，定位到目标表和索引：
   - 使用 **dictionary cache / table cache** 拿到表、索引定义。
2. 访问目标行：
   - 先查 **Buffer Pool** 是否已有对应聚簇索引页；没有则从磁盘读入，挂入 LRU。
3. 修改：
   - 在 Buffer Pool 的页中修改行数据，页变成 dirty 页，进入 Flush List；
   - 生成 redo log 记录写入 **Log Buffer**。
4. 需要修改二级索引：
   - 如果对应二级索引页在 Buffer Pool：直接更新页；
   - 不在缓冲池：根据 `innodb_change_buffering` 设置，把操作写入 **Change Buffer**，等待后续合并。
5. 提交事务：
   - 根据刷新策略把 **Log Buffer** 的内容刷到 redo log 文件；
   - 只要 redo 在磁盘上，就认为事务“持久化”了；实际数据页刷盘可后移，由后台 Checkpoint 线程根据脏页比例、redo 使用量等策略进行。[Medium+1](https://duthaho.medium.com/understanding-the-mysql-innodb-buffer-pool-a-deep-dive-for-performance-optimization-0fa6b52ffb21?utm_source=chatgpt.com)
6. 读热点 key 越来越多时：
   - InnoDB 观察到某些 B+Tree 范围被频繁访问，会自动在 Buffer Pool 中构建 **Adaptive Hash Index** 来加速后续访问。[dev.mysql.com](https://dev.mysql.com/doc/refman/8.3/en/innodb-adaptive-hash.html?utm_source=chatgpt.com)

这样你就可以把 InnoDB 内存架构想象成：

> **Buffer Pool 是大仓库，Change Buffer 和 AHI 是在仓库里开的两个“专用区/加速区”，Log Buffer 是出门口的“记账台”，其余事务/锁/字典结构则是仓库的管理系统。**

如果你后面想深入到 **“Buffer Pool 具体数据结构（控制块、hash table、LRU 算法细节）”** 或者 **“8.0 中 redo/flush 线程模型与内存结构的关系”**，我也可以单独拆一节来画更底层的图。