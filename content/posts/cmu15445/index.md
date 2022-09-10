---
title: "cmu15-445 数据库系统 Project"
subtitle: ""
date: 2022-06-07T17:20:22+08:00
draft: false
author: "fitenne"
keywords: ""
license: ""
comment: false
weight: 0

tags:
- cmu15-445
categories:
- cmu15-445

hiddenFromHomePage: false
hiddenFromSearch: false

summary: "cmu15-445 fall2021 project"

toc:
  enable: true
math:
  enable: true
lightgallery: false
---

## 引言

[CMU 15-445](https://15445.courses.cs.cmu.edu/) 主要讲授数据库管理系统的设计与实现。课程 project 要求基于 [bustub](https://github.com/cmu-db/bustub) 实现一个关系型数据库。

本文参考的 project 要求来自 2021fall 学期，相应的测试用例在 [gradescope](https://15445.courses.cs.cmu.edu/fall2021/faq.html#q7)。

课程的 faq 页面给出了非 cmu 学生也可以使用的一个 Entry Code，任何人都可以在 gradescope 免费注册账号，进行提交验证自己的代码是否正确。

本文主要是辅助我完成 project 的材料，主要是 project 需求的（不完整）翻译，也包括代码过程中遇到的一些问题。

## project 1

project 1 要求实现一个线程安全的 buffer pool，负责管理页（Page） 在内存和硬盘之间的换入换出。buffer pool 的操作对系统的其他部分是透明的。比如，系统以 `page_id_t` 向 buffer pool 要求一个页，此时系统不知道这个页是在内存还是在硬盘中。

buffer pool 具体可以分解为 3 个类：

- `LRUReplacer`

  实现 LRU 策略。

- `BufferPoolManagerInstance`

  管理若干个页。
  
- `ParallelBufferPoolManager`

  为了保证线程安全，`BufferPoolManagerInstance` 经常需要上锁，这会导致性能问题。`ParallelBufferPoolManager` 管理多个 `BufferPoolManagerInstance`，每个 `ParallelBufferPoolManager` 有自己的锁，从而整体提高性能。

### `LRUReplacer`

这一部分负责追踪页在 buffer pool 中的使用情况。

BufferPoolManager 中的帧（frame）指 buffer pool 中的一个占位符，因此 LRUReplacer 中页的最大数量和 buffer pool 一致。任意时刻，并不是全部的帧都在 LRUReplacer 中。一开始没有帧在
LRUReplacer 中，之后只有刚刚被 unpinned 的那些在 LRUReplacer。

我们需要实现下面的方法：

- `Victim(frame_id_t*)`
  
  移除最久没有被访问过的帧，将其内容保存在出参中，返回 `true`，如果 LRUReplacer 为空，返回 `false`。

- `Pin(frame_id_t)`

  一个页在 BufferPoolManager 中被 pin 后应调用这个方法，此时这个方法应当从 LRUReplacer 移除包含对应页的帧。

- `Unpin(frame_id_t)`

  一个页的 `pin_count` 变为 0 后应当调用此方法，此方法应当将包含对应页的帧加入 LRUReplacer。

- `Size()`

  返回 LRUReplacer 中的帧的数量。

`LRUReplacer` 的实现主要涉及 `std::list` 和 `std::unordered_map<frame_id_t,std::list<frame_id_t>::iterator>`，`list` 用来维护按时间顺序被 `Unpin` 的 frame，`unordered_map` 维护 frame 到 `list` 节点的映射，用来加速 `Pin` 操作。

> `Unpin` 不被认为更新了 frame 最近一次被使用的时间。

### `BufferPoolManagerInstance`

BufferPoolManagerInstance 负责从 DiskManager 抓取页，并将页存入内存。BufferPoolManagerInstance 也可以将脏页写入硬盘如果它被要求这么做或者需要换出一个页。

所有的内存中的页都以 `Page` 对象代表。DiskManager 使用`Page` 对象中的特定块来存储物理页的副本。每个 `Page` 对象不同时刻可能包含不同物理页。如果一个 `Page` 对象不包含任何物理页，它的 `page_id` 应为 `INVALID_PAGE_ID`。

每个 `Page` 对象维护一个计数器计算有多少线程 pinned 那一页。BufferPoolManagerInstance 不可以释放被 pinned 的页。每个页需要知道自己是否是脏页。一个脏页的 `Page` 对象被复用前必须写入硬盘。

需要实现的方法：
- `FetchPgImp(page_id)`
- `UnpinPgImp(page_id, is_dirty)`
- `FlushPgImp(page_id)`
- `NewPgImp(page_id)`
- `DeletePgImp(page_id)`
- `FlushAllPagesImpl()`

`FetchPgImp` 应当返回 NULL 如果没有空闲页并页其他页都被 pinned。`FlushPgImp` 不管 pin 的状态都应当执行刷盘。

`is_dirty` 追踪一个页在被 pinned 的过程中是否被修改。

> `UnpinPgImp` 的 `is_dirty` 参数不应被直接赋给对应页的 `is_dirty_` 标志，而是与原值取或。
> 
> `NewPgImp` 应当立刻 `Pin` 申请的新页。

### `ParallelBufferPoolManager`

这个类控制多个 `BufferPoolManagerInstance`，每次操作，这个类选择一个 `BufferPoolManagerInstance`，代表它进行操作。

注意将 `num_instances` 和 `instance_index` 传给 `BufferPoolManagerInstance` 的构造函数。

要实现的方法：
- `ParallelBufferPoolManager(num_instances, pool_size, disk_manager, log_manager)`
- `~ParallelBufferPoolManager()`
- `GetPoolSize()`
- `GetBufferPoolManager(page_id)`
- `FetchPgImp(page_id)`
- `UnpinPgImp(page_id, is_dirty)`
- `FlushPgImp(page_id)`
- `NewPgImp(page_id)`
- `DeletePgImp(page_id)`
- `FlushAllPagesImpl()`

### tips

gradescope 给出的时间限制十分紧张，即使使用 Hash 优化所有查表的操作(ie，避免了所有 $O(n)$ 查表)，在程序实现比较差的情况下，
依然可能超出 gradescope 的时间限制，尤其是打开了 valgrind 的测试点。这种情况下，可以主要考虑优化程序中锁的使用。

关于 valgrind 输出 "ASan runtime does not come first in initial library list;"，见 
[issue135](https://github.com/cmu-db/bustub/issues/135)，此 issue 提及的 pr 修复了这一问题，不过值得注意的是，
pr 中修改之后的 flag `-static-libasan` 只有 gcc 支持，而静态检查使用的 clang 会认为修改后的 flag 是一个错误的参数。

## project 2

project 2 需要使用可扩展 hash 方案(extendible hashing)实现一个硬盘支持的哈希表。

哈希表索引由目录页和其中包含的桶页组成。表通过 buffer pool 访问页。桶需要支持分裂合并，目录需要支持扩展收缩。

这个 project 分为 3 个部分：

- 页布局

- 可扩展哈希实现

- 并发控制

### 页布局

哈希表要通过 `BufferPoolManager` 访问。所有东西都需要存储到硬盘页中，这样你可以通过 `DiskManager` 读写这些页，
并且系统重启后数据不会丢失。

这里需要实现 `HashTableDirectoryPage` 和 `HashTableBucketPage`。

### hash 实现

我们需要实现使用可扩展哈希方案的哈希表，这个表需要支持 `Insert`，`GetValue` 和 `Remove`。下面是一些相关的要求。

- 目录索引

  使用最低有效位索引目录项。

- 桶分裂

  推荐当一次插入操作会引起溢出的时候，分裂桶。

- 合并桶

  当一个桶变空的时候必须尝试桶合并。为了简单，遵循下面的规则：

  - 只有空桶可以被合并。

  - 只有分裂镜像桶(split image)与桶有相同 local depth时，才允许合并。

  - 只有 local depth 大于 0 的桶才能合并。

- 目录收缩(directory shrinking)

  只有每个桶的 local depth 小于 global depth 时允许收缩。

### tips

project 2 对各部分实现细节描述十分详细，不过实现起来的依然很容易忽略一些细节。

注意在一个空桶上执行 Remove 是合法的，此时虽然 Remove 会返回 false，但依然需要尝试 Merge 操作。

## project 3

在 project 3 中， 我们需要添加对查询执行器的支持，我们需要实现负责接受询问计划结点并执行他们的 **executors**。
我们的 executors 支持下面的操作：

访问方法：顺序访问。

修改：插入、更新、删除。

杂项：嵌套循环连接(nested loop join)、哈希连接(hash join)、聚集、限制、去重。

DBMS 目前还不支持 SQL，我们的实现会直接操作手写的执行计划。

### tips

对于 2021fall semester, 注意 [issue195](https://github.com/cmu-db/bustub/issues/195)。
虽然 issus 中说明那一段代码不会有问题，但是在我的开发环境下，没有转换过的 tuple 会导致崩溃。

对于 2021fall semester，注意 [issue195](https://github.com/cmu-db/bustub/issues/227) 解决 gradescope 方面的一些问题。

project 3 中对我来说的难点在于，从子执行器 `Next()` 的返回结果中获取当前需要的 tuple 的方法。一般情况下，直接从 
`Column.GetExpr` 类型转换为 `ColumnValueExpression` 获取生成列的表达式，然后执行 `Evaluate` 获取某一特定的列。
不过，对于 `DistinctExecutor`，观察其对应的测试用例，有这么一段代码：

```c++
auto *table_info = GetExecutorContext()->GetCatalog()->GetTable("test_7");
auto &schema = table_info->schema_;

auto *col_c = MakeColumnValueExpression(schema, 0, "colC");
auto *out_schema = MakeOutputSchema({{"colC", col_c}});

// Construct sequential scan
auto seq_scan_plan = std::make_unique<SeqScanPlanNode>(out_schema, nullptr, table_info->oid_);

// Construct the distinct plan
auto distinct_plan = std::make_unique<DistinctPlanNode>(out_schema, seq_scan_plan.get());
```

最终输出模式 中的 "colC" 是通过 `MakeColumnValueExpression(schema, 0, "colC");` 得到的，其中的 `schema` 
并不是 `seq_scan_plan` 的输出模式，而是底层表的模式，这意味者 col_c 的索引为值 2。而子执行器的输出模式只有一列，
因此直接在子执行器的输出 tuple 上执行 `Evaluate`，会发生越界。实际上这里可以通过列名获取正确的索引值，
不确定这是不是测试用例的问题。

## project 4

### 术语与概念

本次 project 涉及到的陌生概念比较多，此处给出其中一些概念的快速索引。

DBMS 执行操作的顺序称为**执行调度(execution schedule)**。

**串行调度(Serial Schedule)**：不同事务的操作不相交叉。

**等价调度(Equivalent Schedules)**：两个调度执行的效果相同，则他们是等价的。

**可串行调度(Serializable Schedule)**：可串行调度至少等价于一个串行调度。等效的多个串行调度可能会产生不同的
结果，但是所有结果都被认为是「正确」的。

两个连续的指令 I 和 J 是 **冲突(conflict)** 的，如果
- 他们来自不同的事务。
- 操作同一个对象。
- 至少一个是 write 指令。

三种类型的冲突：
- 读-写冲突("Unrepeatable Reads")：事务中多次读同一对象结果不同。
- 写-读冲突("Dirty Reads")：事务提交之前，看到了另一个事务的影响。
- 写-写冲突("Lost Updates")：一个事务覆盖了另一个事务未提交的修改。

两个调度是 **冲突等价(conflict equivalent)** 的，如果他们涉及到相同的一些事务的相同一些操作，并且每一对冲突的操作
在两个调度中的顺序相同。一个调度 S 是 **冲突可串行化(conflict serializable)** 的，如果 S 和某个串行调度冲突等价。

二段锁的级联撤销 **(cascading aborts)** 问题：一个事务的撤销，导致另一个事务也必须撤销。（例如，第二个事务读到了
第一个事务写入的值，由于第一个事务的撤销，读到的值逻辑上没有存在过，显然是一个不合法的值，所以第二个事务必须撤销。）

一个调度是 **严格(strict)** 的，如果事务写入的任何值直到事务提交之前都不会被其他事务读或写。

project 4 的内容主要是实现一个 Lock Manager，然后修改 project 3 中实现的 executor 使用 Lock Manager 实现三种
事务的隔离级别：repeatable read、read committed、read uncommitted。

Lock Manager 采取 Wound-Wait 策略避免死锁。该策略下，每个事务都有一个时间戳，只有「年轻」事务等待
「年老」事务是被允许。同时「年老」事务可以抢占式的获取「年轻」事务的锁(wound)。

project 4 不要求对索引上锁，仅对元组上锁，依照 strict 2PL 协议可以保证 repeatable read 隔离级别。read 
committed 允许读-写冲突，实际上是「读事务」获取共享锁完成读操作后立刻释放锁，从而使得另一「写事务」可以对进行同一
元组进行写操作，最终「读事务」第二次进行读操作和前一次读操作的结果矛盾。在 read committed 的基础上，read 
uncommitted 还允许写-读冲突，不论哪种隔离级别，写操作获取排他锁后，都只会在事务结束(提交或终止)时释放排他锁，
写-读冲突实际上是读操作不获取共享锁引起的。

### tips

注意等待队列对不同请求的公平性。

注意 commit 7cb323ba8e1f324efca47ba629af5449063de39c 修改了 `IndexWriteRecord` 的构造函数为 7 个参数，
而 gradescope 使用的代码版本是该提交前的版本。为了通过编译，我将这个 commit revert 掉了。

疑问：我的实现中 Wound-Wait 通过将事务状态设置为 Aborted 隐式终止一个事务。但是对于已经获取到锁的事务，这并不能
组织事务继续对锁锁定对象做出修改，这似乎可能最终使得主动进行 Wound 的事务得到错误结果。